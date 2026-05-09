# Fireblocks 资金归集手续费管理方案

## 背景

系统为每个用户绑定 Fireblocks vault account。用户在 Arbitrum、Solana 等链上向自己的 vault wallet 入金后，系统需要把资金归集到指定 treasury/omnibus vault account。归集交易本身也是链上交易，因此需要在源 vault account 中准备对应链的手续费资产：

- Arbitrum token 归集需要源 vault 有 Arbitrum ETH 支付 gas。
- Solana SPL token 归集需要源 vault 有 SOL 支付交易费，部分场景还可能涉及 token account/rent 成本。
- Arbitrum 原生 ETH 归集可以从入账 ETH 中扣除手续费。
- Solana 原生 SOL 归集可以从入账 SOL 中扣除手续费。

生产环境可使用 Fireblocks Gas Station 自动补充 EVM gas，但当前 sandbox 环境无法使用 Gas Station。因此，本方案设计为：

1. 生产环境：优先使用 Fireblocks Gas Station 管理 EVM 手续费。
2. Sandbox、Solana、或任何无法使用 Gas Station 的环境：使用内部 Fee Treasury Vault 自动给用户 vault 补充手续费资产。
3. 归集 worker 在发起 sweep 前必须先检查手续费余额；余额不足时先创建 fee top-up 任务，等待 top-up 完成和余额更新后再归集。

参考文档：

- Fireblocks Sweep Funds: https://developers.fireblocks.com/docs/sweep-funds
- Fireblocks Gas Station: https://developers.fireblocks.com/docs/work-with-gas-station
- Enable Gas Station: https://developers.fireblocks.com/docs/enabling-the-gas-station-1
- Configure Gas Station Values: https://developers.fireblocks.com/docs/configure-gas-station-values
- Set Auto Fueling Property: https://developers.fireblocks.com/docs/set-auto-fueling-property
- Estimate Transaction Fee: https://developers.fireblocks.com/api-reference/transactions/estimate-transaction-fee
- Estimate Network Fee: https://developers.fireblocks.com/api-reference/transactions/estimate-the-required-fee-for-an-asset
- Create Transactions: https://developers.fireblocks.com/reference/create-transactions

## 核心原则

1. 归集前必须检查源 vault 的手续费资产余额。
2. token 归集不能假设 token 本身可支付手续费；手续费由所在链 base asset 支付。
3. sandbox 不依赖 Gas Station，使用 Fee Treasury Vault 手动补 gas/SOL。
4. fee top-up 与 sweep 是两个独立链上交易，分别落库、幂等、监听状态。
5. 用户入账记账不依赖 sweep 完成，也不依赖 fee top-up 完成。
6. 平台承担的手续费要单独记账，不能混入用户资产余额。
7. 小额入金要设置最小归集阈值，避免手续费大于归集价值。

## 资产手续费映射

建立资产配置表，明确每个可入金资产对应的手续费资产。

```sql
CREATE TABLE asset_fee_configs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  fee_asset_id VARCHAR(64) NOT NULL,
  is_base_asset TINYINT NOT NULL DEFAULT 0,
  sweep_enabled TINYINT NOT NULL DEFAULT 1,
  gas_station_supported TINYINT NOT NULL DEFAULT 0,
  manual_topup_enabled TINYINT NOT NULL DEFAULT 1,
  min_sweep_amount DECIMAL(38, 18) NOT NULL DEFAULT 0,
  min_fee_balance DECIMAL(38, 18) NOT NULL DEFAULT 0,
  target_fee_balance DECIMAL(38, 18) NOT NULL DEFAULT 0,
  max_topup_amount DECIMAL(38, 18) NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_asset_fee (chain, asset_id)
);
```

示例：

```text
chain     asset_id       fee_asset_id        is_base_asset
arbitrum  ETH_ARB        ETH_ARB             1
arbitrum  USDC_ARB       ETH_ARB             0
arbitrum  USDT_ARB       ETH_ARB             0
solana    SOL            SOL                 1
solana    USDC_SOL       SOL                 0
```

说明：

- `asset_id` 必须使用 Fireblocks workspace 中实际资产 ID。
- `min_fee_balance` 是允许发起 sweep 的最低手续费资产余额。
- `target_fee_balance` 是补款后希望源 vault 达到的手续费资产余额。
- `max_topup_amount` 防止异常配置导致大额补款。
- `min_sweep_amount` 用于过滤 dust deposit。

## Fee Treasury Vault

为每个环境配置一个或多个手续费 treasury vault account。

```sql
CREATE TABLE fee_treasury_vaults (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  env VARCHAR(32) NOT NULL,
  chain VARCHAR(32) NOT NULL,
  fee_asset_id VARCHAR(64) NOT NULL,
  fireblocks_vault_account_id VARCHAR(64) NOT NULL,
  enabled TINYINT NOT NULL DEFAULT 1,
  min_inventory_balance DECIMAL(38, 18) NOT NULL DEFAULT 0,
  alert_balance DECIMAL(38, 18) NOT NULL DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_fee_treasury (env, chain, fee_asset_id)
);
```

建议：

- `sandbox` 使用独立 Fee Treasury Vault，通过测试网 faucet 或预置 testnet 资产补充。
- `production` 即使启用 Gas Station，也保留 Fee Treasury Vault 作为 Solana 和异常补偿路径。
- 不要用用户 vault 之间互相拆借手续费资产，避免资金归属和审计混乱。

## 总体流程

```text
Deposit confirmed
  -> create sweep task
  -> Sweep Worker loads asset_fee_config
  -> check whether deposit amount reaches min_sweep_amount
  -> estimate sweep fee
  -> read source vault fee asset balance
  -> if fee balance sufficient:
       create sweep transaction
     else:
       if production + EVM + Gas Station enabled:
         wait for Gas Station auto-fuel or trigger retry later
       else:
         create fee top-up task from Fee Treasury Vault to user vault
         wait for top-up COMPLETED and balance update
         retry sweep
```

## Sweep task 状态扩展

在已有 `sweep_tasks` 上增加手续费相关状态。

```text
PENDING
  -> FEE_CHECKING
  -> WAITING_FEE_TOPUP
  -> FEE_TOPUP_SUBMITTED
  -> FEE_READY
  -> SUBMITTING_SWEEP
  -> SWEEP_SUBMITTED
  -> SWEEP_COMPLETED
```

异常状态：

```text
SKIPPED_DUST
  入金金额低于最小归集金额。

FEE_TREASURY_LOW
  Fee Treasury Vault 手续费资产库存不足。

FEE_ESTIMATE_FAILED
  Fireblocks estimate fee 或内部估算失败。

FEE_TOPUP_FAILED
  手续费补款交易失败。

FEE_TOPUP_TIMEOUT
  手续费补款长时间未完成或余额未更新。

SWEEP_FAILED_INSUFFICIENT_FEE
  sweep 发起后仍因手续费不足失败，需要重新估算和补款。
```

## Fee top-up 任务表

```sql
CREATE TABLE fee_topup_tasks (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  sweep_task_id BIGINT NOT NULL,
  deposit_id BIGINT NOT NULL,
  env VARCHAR(32) NOT NULL,
  chain VARCHAR(32) NOT NULL,
  fee_asset_id VARCHAR(64) NOT NULL,
  source_fee_treasury_vault_id VARCHAR(64) NOT NULL,
  destination_user_vault_id VARCHAR(64) NOT NULL,
  requested_amount DECIMAL(38, 18) NOT NULL,
  estimated_sweep_fee DECIMAL(38, 18) NULL,
  fee_balance_before DECIMAL(38, 18) NULL,
  target_fee_balance DECIMAL(38, 18) NULL,
  external_tx_id VARCHAR(255) NOT NULL,
  fireblocks_tx_id VARCHAR(128) NULL,
  tx_hash VARCHAR(128) NULL,
  status VARCHAR(32) NOT NULL,
  attempts INT NOT NULL DEFAULT 0,
  last_error TEXT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_sweep_task (sweep_task_id),
  UNIQUE KEY uk_external_tx_id (external_tx_id),
  KEY idx_status (status, updated_at),
  KEY idx_fireblocks_tx (fireblocks_tx_id)
);
```

`external_tx_id` 建议：

```text
fee-topup:sweep:{sweep_task_id}:asset:{fee_asset_id}
```

## 平台手续费账本

手续费补款不是用户充值，不应增加用户可用余额。建议建立平台资金事件表。

```sql
CREATE TABLE platform_fee_events (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  event_type VARCHAR(64) NOT NULL,
  ref_type VARCHAR(64) NOT NULL,
  ref_id BIGINT NOT NULL,
  env VARCHAR(32) NOT NULL,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  amount DECIMAL(38, 18) NOT NULL,
  direction VARCHAR(16) NOT NULL,
  source_vault_id VARCHAR(64) NULL,
  destination_vault_id VARCHAR(64) NULL,
  fireblocks_tx_id VARCHAR(128) NULL,
  status VARCHAR(32) NOT NULL,
  metadata JSON NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_ref_event (ref_type, ref_id, event_type),
  KEY idx_asset_time (env, chain, asset_id, created_at)
);
```

事件类型：

```text
FEE_TOPUP_CREATED
  Fee Treasury 向用户 vault 补充手续费资产。

FEE_TOPUP_COMPLETED
  手续费补款链上完成。

SWEEP_NETWORK_FEE_RESERVED
  sweep 发起前记录预计手续费。

SWEEP_NETWORK_FEE_CONFIRMED
  sweep 完成后记录实际手续费。
```

## 手续费估算策略

### Fireblocks estimate transaction fee

优先使用 `POST /v1/transactions/estimate_fee` 模拟即将创建的 sweep 交易。该接口会按真实交易规则估算费用，因此要求源 vault 有相关资产和余额。

适用：

- 归集前估算 Arbitrum token sweep 的 gas。
- 归集前估算 Arbitrum ETH native sweep 的 gas。
- 归集前估算 Solana SOL/SPL token transfer 的 fee。

注意：

- Solana `PROGRAM_CALL` 不支持 fee estimation；普通 `TRANSFER` 可按 Fireblocks 支持情况使用。
- estimate_fee 不是最终手续费承诺，实际链上费用可能变化。
- 不要对每个微小变化高频调用；缓存短时间结果。

### Fireblocks estimate network fee

`GET /v1/estimate_network_fee?assetId={feeAssetId}` 返回当前网络费用，Fireblocks 侧约 30 秒缓存一次。适合：

- 定期刷新 fee level。
- 为同一链同一资产批量 sweep 计算大致补款额度。
- 当 estimate transaction fee 失败时作为降级估算来源。

### 内部默认 gas profile

为 sandbox 和异常场景保留静态 profile：

```sql
CREATE TABLE fee_gas_profiles (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  env VARCHAR(32) NOT NULL,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  operation VARCHAR(64) NOT NULL,
  fee_asset_id VARCHAR(64) NOT NULL,
  default_fee DECIMAL(38, 18) NOT NULL,
  high_fee DECIMAL(38, 18) NOT NULL,
  safety_multiplier DECIMAL(10, 4) NOT NULL DEFAULT 1.5,
  enabled TINYINT NOT NULL DEFAULT 1,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_profile (env, chain, asset_id, operation)
);
```

用途：

- sandbox testnet 手续费波动小，使用固定 profile 可以减少对 estimate API 的依赖。
- 当 Fireblocks fee estimate 短暂不可用时，用 high fee + safety multiplier 继续测试。
- 生产环境只能作为 fallback，不能长期替代真实 fee estimate。

## 手续费补款金额计算

输入：

```text
fee_balance = source vault 当前 fee asset 可用余额
estimated_fee = 本次 sweep 预计手续费
min_fee_balance = 配置最低余额
target_fee_balance = 配置目标余额
max_topup_amount = 单次最大补款
```

计算：

```text
required_balance = max(min_fee_balance, estimated_fee * safety_multiplier)

if fee_balance >= required_balance:
  no top-up
else:
  topup_amount = target_fee_balance - fee_balance
  topup_amount = max(topup_amount, required_balance - fee_balance)
  topup_amount = min(topup_amount, max_topup_amount)
```

如果 `topup_amount <= 0` 或 `topup_amount > max_topup_amount`，进入人工检查或标记 `FEE_CONFIG_INVALID`。

## Sandbox 环境方案

由于 sandbox 环境无法使用 Gas Station，采用手动 Fee Treasury 自动补款。

### Sandbox vault 结构

```text
Sandbox Fee Treasury Vault
  - ARB testnet ETH / 对应 Arbitrum 测试网络 base asset
  - SOL testnet

User Deposit Vaults
  - 用户入金 vault
  - auto fuel 不作为依赖

Omnibus/Treasury Vault
  - 归集目标 vault
```

### Sandbox 流程

```text
1. 用户向 user vault 入金 token
2. deposit confirmed 后创建 sweep task
3. Sweep Worker 检查 user vault 的 fee asset balance
4. 如果不足：
   - 从 Sandbox Fee Treasury Vault 转少量 fee asset 到 user vault
   - 监听 fee top-up transaction COMPLETED
   - 等待 vault_account.asset.balance_updated
5. fee asset 到账后发起 sweep
6. sweep 完成后记录实际手续费
```

### Sandbox 初始化

上线测试前准备：

- 为 Fee Treasury Vault 准备足够 testnet base asset。
- 为每条测试链配置 `asset_fee_configs`。
- 设置较低 `min_sweep_amount`，方便小额测试。
- 设置保守 `target_fee_balance`，避免每笔测试都补 gas。
- 建立 faucet 或人工补充流程，避免 Fee Treasury 资产耗尽。

Sandbox 示例配置：

```text
arbitrum testnet token:
  min_fee_balance = 0.00005 ETH
  target_fee_balance = 0.0002 ETH
  max_topup_amount = 0.001 ETH

solana testnet SPL token:
  min_fee_balance = 0.005 SOL
  target_fee_balance = 0.02 SOL
  max_topup_amount = 0.05 SOL
```

实际数值必须按你的 Fireblocks sandbox asset、测试网费用和 faucet 可得性调整。

## 生产环境 EVM 方案

生产环境如果已开通 Fireblocks Gas Station：

```text
1. 每个用户 vault 启用 auto-fueling。
2. 为每条 EVM 链配置 gasThreshold、gasCap、maxGasPrice。
3. 入账 token 完成后，Gas Station 检查用户 vault 的 base asset balance。
4. 如果低于 threshold，Gas Station 从 Gas Station wallet 补至 gasCap。
5. Sweep Worker 等待 fee asset balance 达到 required_balance。
6. 发起 sweep。
```

Gas Station 配置建议：

- `gasThreshold`: 至少覆盖 1 次普通 token sweep 的 high fee。
- `gasCap`: 覆盖 2 到 5 次普通 token sweep，减少频繁补 gas。
- `maxGasPrice`: 设置为业务可接受上限，避免极端 gas 价格下自动补款成本失控。

注意：

- Gas Station 支持 EVM 网络，不解决 Solana SOL 手续费问题。
- Gas Station auto-fuel 也是链上交易，需要等待完成。
- Sweep Worker 不能因为开启 Gas Station 就跳过 fee balance check。
- 当 Gas Station 长时间未补款，fallback 到人工处理；生产是否允许 Fee Treasury 手动补 gas 需由安全策略决定。

## Solana 方案

Solana 不使用 Fireblocks Gas Station。所有 Solana 归集都走 Fee Treasury 补 SOL。

规则：

- SOL 入金归集：如果入金 SOL 足够覆盖手续费，可使用 native asset sweep；不要把用户 vault 扫到 0，保留少量 SOL 或使用可扣费参数。
- SPL token 入金归集：必须确保用户 vault 有 SOL。
- 如果 SPL token account 创建或 rent 需要额外 SOL，估算金额要包含 rent buffer。
- 小额 SPL token 入金低于 `min_sweep_amount` 时不立即归集，避免 SOL 成本过高。

Solana sweep 前检查：

```text
fee_asset_id = SOL
required_balance = max(config.min_fee_balance, estimated_fee * safety_multiplier, rent_buffer)
if SOL balance < required_balance:
  create SOL fee top-up task
```

## 原生资产归集策略

### Arbitrum ETH / EVM base asset

如果归集资产本身就是 fee asset：

- 可使用 `treatAsGrossAmount = true`，让手续费从转账金额中扣除。
- 或者保留最小 dust，不扫空源 vault。
- 推荐对用户入账记账仍按 deposit amount 入账；平台手续费单独记账为成本，不从用户账面余额扣除，除非产品规则明确由用户承担提现/归集手续费。

注意：

- 如果你把归集视为平台内部资金调拨，手续费应由平台承担。
- 如果使用 gross amount 导致实际到账 treasury 少于用户入账金额，需要平台账本记录差额。

### Solana SOL

SOL sweep 同样不能盲目扫空：

- 保留 `min_fee_balance`。
- 或按 Fireblocks 支持的 fee 参数让手续费从转账金额扣除。
- 对小额 SOL 入金设置最小归集阈值。

## Sweep Worker 伪代码

```text
processSweepTask(task):
  deposit = load deposit
  config = load asset_fee_config(deposit.chain, deposit.asset_id)

  if deposit.amount < config.min_sweep_amount:
    mark task SKIPPED_DUST
    return

  estimatedFee = estimateSweepFee(task, config)
  feeBalance = getVaultAssetBalance(task.source_vault_account_id, config.fee_asset_id)

  requiredBalance = max(config.min_fee_balance, estimatedFee * safetyMultiplier)

  if feeBalance < requiredBalance:
    if shouldUseGasStation(env, config):
      mark task WAITING_GAS_STATION
      schedule retry
      return

    topup = calculateTopupAmount(feeBalance, estimatedFee, config)
    createFeeTopupTask(task, topup)
    mark task WAITING_FEE_TOPUP
    return

  createSweepTransaction(task)
  mark task SWEEP_SUBMITTED
```

## Fee top-up Worker 伪代码

```text
processFeeTopupTask(task):
  lock task

  if task.status not in [PENDING, RETRYABLE_FAILED]:
    return

  treasuryBalance = getVaultAssetBalance(task.source_fee_treasury_vault_id, task.fee_asset_id)
  if treasuryBalance < task.requested_amount + treasuryMinInventory:
    mark FEE_TREASURY_LOW
    alert
    return

  create Fireblocks transaction:
    assetId = task.fee_asset_id
    amount = task.requested_amount
    source = Fee Treasury Vault
    destination = User Vault
    externalTxId = fee-topup:sweep:{sweep_task_id}:asset:{fee_asset_id}
    feeLevel = MEDIUM or HIGH

  save fireblocks_tx_id
  mark SUBMITTED
```

top-up 完成处理：

```text
on transaction.status.updated COMPLETED for fee top-up:
  mark fee_topup_task COMPLETED
  insert platform_fee_event FEE_TOPUP_COMPLETED
  mark sweep_task FEE_READY
  enqueue sweep_task
```

如果收到 balance update 更晚：

```text
on vault_account.asset.balance_updated:
  if related fee_topup_task completed and balance >= requiredBalance:
    mark sweep_task FEE_READY
    enqueue sweep_task
```

## Fireblocks create transaction 参数建议

Fee top-up：

```json
{
  "assetId": "ETH_ARB",
  "amount": "0.0002",
  "externalTxId": "fee-topup:sweep:123:asset:ETH_ARB",
  "source": {
    "type": "VAULT_ACCOUNT",
    "id": "FEE_TREASURY_VAULT_ID"
  },
  "destination": {
    "type": "VAULT_ACCOUNT",
    "id": "USER_VAULT_ID"
  },
  "feeLevel": "MEDIUM",
  "note": "fee top-up for sweep 123"
}
```

Token sweep：

```json
{
  "assetId": "USDC_ARB",
  "amount": "10.25",
  "externalTxId": "sweep:deposit:456:asset:USDC_ARB",
  "source": {
    "type": "VAULT_ACCOUNT",
    "id": "USER_VAULT_ID"
  },
  "destination": {
    "type": "VAULT_ACCOUNT",
    "id": "OMNIBUS_VAULT_ID"
  },
  "feeLevel": "MEDIUM",
  "failOnLowFee": true,
  "note": "sweep deposit 456"
}
```

Native asset sweep：

```json
{
  "assetId": "ETH_ARB",
  "amount": "0.01",
  "treatAsGrossAmount": true,
  "externalTxId": "sweep:deposit:456:asset:ETH_ARB",
  "source": {
    "type": "VAULT_ACCOUNT",
    "id": "USER_VAULT_ID"
  },
  "destination": {
    "type": "VAULT_ACCOUNT",
    "id": "OMNIBUS_VAULT_ID"
  },
  "feeLevel": "MEDIUM",
  "failOnLowFee": true
}
```

具体字段以 Fireblocks 对该链和资产的实际支持为准。

## 失败补偿

### Fee top-up 长时间未完成

处理：

- 查询 `GET /transactions/{txId}`。
- 如果 Fireblocks 交易不存在，使用同一个 `externalTxId` 重试创建。
- 如果状态为 `FAILED/REJECTED/CANCELLED`，标记 `FEE_TOPUP_FAILED`。
- 如果状态为 `CONFIRMING` 时间过长，进入 `FEE_TOPUP_STUCK` 并报警。

### Sweep 因手续费不足失败

处理：

- 更新 sweep task 为 `SWEEP_FAILED_INSUFFICIENT_FEE`。
- 重新调用 estimate fee。
- 重新检查 fee balance。
- 如不足，创建或更新 fee top-up task。
- 使用同一个 sweep `externalTxId` 前必须先确认 Fireblocks 是否已创建过交易；不要重复发起不同 externalTxId 的 sweep。

### Fee Treasury 库存不足

处理：

- 标记 `FEE_TREASURY_LOW`。
- 暂停该链新增 fee top-up。
- 发出告警。
- sandbox 从 faucet 或人工转入 testnet base asset。
- production 由 treasury 运维流程补充 base asset。

### Dust deposit

处理：

- deposit 仍然给用户记账。
- sweep task 标记 `SKIPPED_DUST` 或 `DEFERRED_DUST`。
- 等同一用户同一 vault 同一 asset 累计达到阈值后合并归集。

## 监控告警

关键指标：

- `fee_topup_tasks_pending_total`
- `fee_topup_failed_total`
- `fee_topup_completed_latency_seconds`
- `fee_treasury_balance`
- `sweep_waiting_fee_total`
- `sweep_failed_insufficient_fee_total`
- `gas_station_waiting_total`
- `dust_deferred_total`
- `estimated_fee_vs_actual_fee_ratio`

告警：

- Fee Treasury 余额低于 `alert_balance`。
- `WAITING_FEE_TOPUP` 超过 10 分钟。
- `FEE_TOPUP_SUBMITTED` 超过链上合理确认时间。
- `SWEEP_FAILED_INSUFFICIENT_FEE` 连续出现。
- 生产 Gas Station 已启用但 EVM sweep 长时间等待 fee。
- sandbox faucet 或 testnet base asset 库存耗尽。

## 上线检查清单

- 已为每个支持资产配置 `asset_fee_configs`。
- 已为 sandbox 配置 Fee Treasury Vault。
- 已为 Arbitrum token 配置 base asset ETH 手续费策略。
- 已为 Solana SPL token 配置 SOL 手续费策略。
- Sweep Worker 在 create transaction 前检查 fee balance。
- Fee top-up task 使用唯一 `externalTxId`。
- Sweep task 使用唯一 `externalTxId`。
- Fee top-up 完成后等待 transaction completed 和 balance update。
- 平台手续费事件不影响用户可用余额。
- 小额 deposit 有 dust/deferred 策略。
- 生产 Gas Station 配置了 threshold、cap、max gas price。
- sandbox 流程不依赖 Gas Station。

## 推荐实施顺序

1. 建立 `asset_fee_configs` 和 `fee_treasury_vaults`。
2. 为 sandbox 准备 Fee Treasury Vault 和测试网 base asset。
3. 扩展 sweep task 状态机，增加 fee check。
4. 实现 fee estimate service。
5. 实现 fee top-up task 和 worker。
6. 接入 Fireblocks webhook，跟踪 fee top-up transaction 状态。
7. 在 Arbitrum token sandbox 跑通：入金、补 gas、归集。
8. 在 Solana SPL sandbox 跑通：入金、补 SOL、归集。
9. 增加 dust defer、库存告警、失败补偿。
10. 生产环境接入 Gas Station，但保留 fee balance check 和异常补偿。
