# Fireblocks 用户入金、余额校验、归集与记账方案

## 背景

系统为每个用户绑定一个 Fireblocks vault account。用户在 Arbitrum、Solana 等链上向自己的 vault wallet 地址转账。系统需要通过 Fireblocks Webhooks 监听入账事件，在确认资金真实入账后，为用户生成 MySQL 记账事件，并将资金归集到指定 treasury/omnibus vault account。

本方案基于以下 Fireblocks 文档约束：

- Validate Balances: https://developers.fireblocks.com/docs/validate-balances
- Webhook best practices: https://developers.fireblocks.com/reference/webhooks-best-practices
- Transaction events: https://developers.fireblocks.com/reference/webhooks-structures-eventtypes-transaction
- Sweep Funds: https://developers.fireblocks.com/docs/sweep-funds
- Sweep to Omnibus: https://developers.fireblocks.com/docs/sweep-to-omnibus
- Create Transactions: https://developers.fireblocks.com/reference/create-transactions
- Validating webhooks: https://developers.fireblocks.com/reference/validating-webhooks

## 核心原则

1. Webhook 入口只做验签、落库、入队，立即返回 2xx。
2. 不依赖 webhook 到达顺序，所有业务处理都必须幂等。
3. 用户最终入账必须等待两个条件：
   - `transaction.status.updated` 显示入账交易为 `COMPLETED`
   - `vault_account.asset.balance_updated` 显示对应 vault asset 余额已更新到同一 `blockHeight`，并尽量校验同一 `blockHash`
4. 用户可见的“入账中”与真正增加可用余额分离。
5. 归集必须异步执行，并使用 `externalTxId` 防止 Fireblocks 侧重复创建交易。
6. MySQL 是业务事实源；Fireblocks 是链上资金与交易状态源。

## 总体架构

```text
Fireblocks Webhooks
  -> Webhook Ingress API
      -> verify signature
      -> insert raw webhook event
      -> enqueue event id
      -> return 2xx

Queue
  -> Deposit Reconciliation Worker
      -> handle transaction.status.updated
      -> handle vault_account.asset.balance_updated
      -> confirm deposit with block height/hash
      -> create ledger event
      -> create sweep task

Queue
  -> Sweep Worker
      -> create Fireblocks transfer
      -> source: user vault account
      -> destination: treasury/omnibus vault account
      -> save sweep transaction id

Fireblocks Webhooks
  -> Sweep Status Worker
      -> handle sweep transaction.status.updated
      -> update sweep task status

Scheduled Jobs
  -> pending deposit reconciliation
  -> pending sweep retry
  -> webhook gap recovery
  -> Fireblocks/API consistency audit
```

## Fireblocks Webhook 配置

至少订阅：

```text
transaction.status.updated
vault_account.asset.balance_updated
```

建议额外订阅：

```text
transaction.created
transaction.alert.stuck_confirming
transaction.network_records.processing_completed
```

用途：

- `transaction.status.updated`: 判断入账交易和归集交易的状态变化。
- `vault_account.asset.balance_updated`: 判断 vault asset 余额是否已经更新到交易所在区块。
- `transaction.alert.stuck_confirming`: 监控 EVM 链上卡住的交易，尤其是归集交易。
- `transaction.created`: 可用于更早展示“检测到入账”，但不能作为最终入账依据。

## 用户 vault 绑定模型

每个用户在每条链、每个资产维度绑定一个 Fireblocks vault account asset。

```sql
CREATE TABLE user_vault_accounts (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  fireblocks_vault_account_id VARCHAR(64) NOT NULL,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  address VARCHAR(128) NOT NULL,
  address_tag VARCHAR(128) NULL,
  enabled TINYINT NOT NULL DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_vault_asset (fireblocks_vault_account_id, asset_id),
  UNIQUE KEY uk_user_chain_asset (user_id, chain, asset_id),
  KEY idx_address (address)
);
```

说明：

- Arbitrum 常见资产包括原生 ETH、USDC、USDT 等，Fireblocks `assetId` 需要以实际工作区支持的 ID 为准。
- Solana 常见资产包括 SOL 和 SPL token。SPL token 转账、归集时要关注 token account、rent、SOL 手续费。
- 如果一个用户同一条链下多个资产共用同一 vault account，可以通过 `(fireblocks_vault_account_id, asset_id)` 定位用户资产。

## Webhook 原始事件表

```sql
CREATE TABLE fireblocks_webhook_events (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  event_id VARCHAR(128) NOT NULL,
  event_type VARCHAR(128) NOT NULL,
  resource_id VARCHAR(128) NULL,
  workspace_id VARCHAR(128) NULL,
  created_at_ms BIGINT NULL,
  payload JSON NOT NULL,
  processed TINYINT NOT NULL DEFAULT 0,
  process_attempts INT NOT NULL DEFAULT 0,
  last_error TEXT NULL,
  received_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  processed_at TIMESTAMP NULL,
  UNIQUE KEY uk_event_id (event_id),
  KEY idx_resource_type (resource_id, event_type),
  KEY idx_processed (processed, received_at)
);
```

处理要求：

- `event_id` 使用 Fireblocks webhook notification 的 `id`。
- 重复事件直接忽略或更新 `received_at`，不得重复执行业务逻辑。
- 保留完整 `payload`，便于审计、补偿、重放。

## 入账表

```sql
CREATE TABLE deposits (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  deposit_key VARCHAR(255) NOT NULL,
  fireblocks_tx_id VARCHAR(128) NOT NULL,
  tx_hash VARCHAR(128) NULL,
  blockchain_index VARCHAR(64) NULL,
  source_type VARCHAR(64) NULL,
  source_address VARCHAR(128) NULL,
  destination_vault_account_id VARCHAR(64) NOT NULL,
  destination_address VARCHAR(128) NULL,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  amount DECIMAL(38, 18) NOT NULL,
  block_height BIGINT NULL,
  block_hash VARCHAR(128) NULL,
  transaction_status VARCHAR(32) NOT NULL,
  balance_confirmed TINYINT NOT NULL DEFAULT 0,
  status VARCHAR(32) NOT NULL,
  raw_tx_event JSON NULL,
  raw_balance_event JSON NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_deposit_key (deposit_key),
  KEY idx_fireblocks_tx (fireblocks_tx_id),
  KEY idx_user_asset_status (user_id, asset_id, status),
  KEY idx_vault_asset_status (destination_vault_account_id, asset_id, status)
);
```

`deposit_key` 建议：

```text
fireblocks_tx_id + ":" + coalesce(blockchainIndex, logIndex, index, "0")
```

或者：

```text
chain + ":" + txHash + ":" + assetId + ":" + destinationVaultAccountId + ":" + coalesce(blockchainIndex, logIndex, index, "0")
```

不要只用 `txHash`。一个链上交易可能包含多个 transfer，尤其是 EVM token transfer、Solana nested instruction 或聚合交易场景。

## Vault asset 余额水位表

`vault_account.asset.balance_updated` 是 vault asset 维度的余额水位事件。建议维护独立水位表，用于判断 Fireblocks 余额视图已经同步到哪个链上区块，并为余额 delta 聚合校验提供基准。

```sql
CREATE TABLE vault_asset_balance_watermarks (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  vault_account_id VARCHAR(64) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  last_block_height BIGINT NOT NULL,
  last_block_hash VARCHAR(128) NULL,
  last_total_balance DECIMAL(38, 18) NULL,
  last_available_balance DECIMAL(38, 18) NULL,
  last_balance_event_id VARCHAR(128) NULL,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_vault_asset (vault_account_id, asset_id),
  KEY idx_block_height (vault_account_id, asset_id, last_block_height)
);
```

说明：

- `last_block_height` 表示该 vault asset 在 Fireblocks 余额视图中已经同步到的最高区块高度。
- `last_block_hash` 用于在同高度场景下校验链上区块是否一致。
- `last_total_balance` 和 `last_available_balance` 用于聚合余额 delta 校验，不建议作为单笔入账的唯一依据。
- 如果首次收到 balance update 时没有历史水位，可以先初始化水位，并通过交易状态和区块水位确认后续 deposit。

## 记账事件表

建议使用事件账本，而不是直接覆盖用户余额。用户余额可由账本汇总，或由余额表在同一个事务内更新。

```sql
CREATE TABLE ledger_events (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT NOT NULL,
  event_type VARCHAR(64) NOT NULL,
  ref_type VARCHAR(64) NOT NULL,
  ref_id BIGINT NOT NULL,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  amount DECIMAL(38, 18) NOT NULL,
  direction VARCHAR(16) NOT NULL,
  balance_effect VARCHAR(32) NOT NULL,
  status VARCHAR(32) NOT NULL,
  metadata JSON NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_ref_event (ref_type, ref_id, event_type),
  KEY idx_user_asset (user_id, asset_id, created_at)
);
```

建议的记账事件：

```text
DEPOSIT_DETECTED
  status = PENDING
  balance_effect = NONE
  含义：Fireblocks 发现入账交易，用户可见为充值处理中。

DEPOSIT_CONFIRMED
  status = POSTED
  balance_effect = AVAILABLE_CREDIT
  含义：交易已完成且 vault 余额已按同一区块更新，可以增加用户可用余额。

SWEEP_CREATED
  status = PENDING
  balance_effect = NONE
  含义：归集任务已提交 Fireblocks，不影响用户余额。

SWEEP_COMPLETED
  status = POSTED
  balance_effect = NONE
  含义：归集已完成，只影响资金所在 vault，不影响用户账面余额。
```

## 归集任务表

```sql
CREATE TABLE sweep_tasks (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  deposit_id BIGINT NOT NULL,
  user_id BIGINT NOT NULL,
  source_vault_account_id VARCHAR(64) NOT NULL,
  destination_vault_account_id VARCHAR(64) NOT NULL,
  chain VARCHAR(32) NOT NULL,
  asset_id VARCHAR(64) NOT NULL,
  amount DECIMAL(38, 18) NOT NULL,
  external_tx_id VARCHAR(255) NOT NULL,
  fireblocks_tx_id VARCHAR(128) NULL,
  tx_hash VARCHAR(128) NULL,
  status VARCHAR(32) NOT NULL,
  attempts INT NOT NULL DEFAULT 0,
  last_error TEXT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  UNIQUE KEY uk_deposit_id (deposit_id),
  UNIQUE KEY uk_external_tx_id (external_tx_id),
  KEY idx_status (status, updated_at),
  KEY idx_fireblocks_tx_id (fireblocks_tx_id)
);
```

`external_tx_id` 建议：

```text
sweep:deposit:{deposit_id}:asset:{asset_id}
```

Fireblocks 文档建议创建交易时传 `externalTxId`，同一个 `externalTxId` 的重复请求不会重复处理。

## 入账状态机

```text
NEW
  -> TX_COMPLETED_WAIT_BALANCE
  -> CONFIRMED
  -> LEDGER_POSTED
  -> SWEEP_PENDING
  -> SWEEP_SUBMITTED
  -> SWEEP_COMPLETED
```

异常状态：

```text
IGNORED
  非用户 vault、非支持资产、非入账方向。

TX_FAILED
  Fireblocks 入账交易最终失败。

BALANCE_MISMATCH
  收到 balance update，但 blockHeight/blockHash 与交易不一致。

RECONCILE_TIMEOUT
  长时间未收到对应 balance update，需要补偿查询。

SWEEP_FAILED
  归集交易失败，可重试或人工处理。

SWEEP_STUCK
  归集交易长时间 CONFIRMING 或收到 stuck alert。
```

## Webhook 入口处理

伪代码：

```text
handleWebhook(rawBody, headers):
  verify Fireblocks-Webhook-Signature with JWKS
  parse payload

  insert fireblocks_webhook_events(event_id, event_type, resource_id, payload)
    on duplicate key ignore

  if inserted:
    enqueue(event_id)

  return 200
```

要求：

- 必须使用 raw body 验签，不能用已 JSON 解析再序列化后的 body。
- JWKS 按 Fireblocks 文档缓存，按环境选择 US/EU/EU2/Sandbox 的 key endpoint。
- 返回 2xx 前不调用 Fireblocks API，不做归集，不写用户余额。

## 处理入账交易事件

处理 `transaction.status.updated`：

```text
handleTransactionStatusUpdated(event):
  tx = event.data

  if tx.operation != TRANSFER:
    return

  if tx.destination.type != VAULT_ACCOUNT:
    return

  userVault = find user_vault_accounts by tx.destination.id + tx.assetId
  if not found:
    return

  if tx.status not in [COMPLETED, FAILED, REJECTED, CANCELLED]:
    upsert deposit status only if already tracked
    return

  depositKey = buildDepositKey(tx)

  if tx.status == COMPLETED:
    upsert deposits:
      status = TX_COMPLETED_WAIT_BALANCE
      transaction_status = COMPLETED
      block_height = tx.blockInfo.blockHeight
      block_hash = tx.blockInfo.blockHash
      amount = tx.amountInfo.netAmount or tx.amountInfo.amount
      raw_tx_event = event.payload

    insert ledger_events DEPOSIT_DETECTED
      on duplicate key ignore

  if tx.status in [FAILED, REJECTED, CANCELLED]:
    mark deposit TX_FAILED when deposit exists
```

金额选择：

- 对外部用户向 vault 入账，通常使用 `amountInfo.amount` 或 `amountInfo.netAmount`。
- 具体应以 Fireblocks 对该链/资产返回字段为准，并在测试网和小额主网验证。
- 不要使用浮点数；所有金额用字符串解析为定点数或最小单位整数。

## 处理余额更新事件

处理 `vault_account.asset.balance_updated`：

```text
handleVaultAssetBalanceUpdated(event):
  balance = event.data
  vaultAccountId = balance.vaultAccountId
  assetId = balance.assetId
  balanceBlockHeight = balance.blockHeight
  balanceBlockHash = balance.blockHash

  oldWatermark = load vault_asset_balance_watermarks for update
  vaultAsset = GET /vault/accounts/{vaultAccountId}/{assetId}

  if vaultAsset.blockHeight < balanceBlockHeight:
    defer event or retry
    return

  if balanceBlockHash exists
     and vaultAsset.blockHash exists
     and vaultAsset.blockHeight = balanceBlockHeight
     and vaultAsset.blockHash != balanceBlockHash:
    mark BALANCE_MISMATCH
    alert
    return

  deposits = find deposits
    where destination_vault_account_id = vaultAccountId
      and asset_id = assetId
      and status = TX_COMPLETED_WAIT_BALANCE
      and block_height <= balanceBlockHeight

  run aggregateBalanceDeltaCheck(oldWatermark, vaultAsset, deposits)

  for each deposit:
    if deposit.block_hash is not null
       and balanceBlockHash is not null
       and deposit.block_height = balanceBlockHeight
       and deposit.block_hash != balanceBlockHash:
      mark BALANCE_MISMATCH
      alert
      continue

    if vaultAsset.blockHeight < deposit.block_height:
      continue

    if both block hashes exist and mismatch:
      mark BALANCE_MISMATCH
      alert
      continue

    confirmDepositAndCreateSweep(deposit, event, vaultAsset)

  update vault_asset_balance_watermarks
    set last_block_height = vaultAsset.blockHeight,
        last_block_hash = vaultAsset.blockHash,
        last_total_balance = vaultAsset.total,
        last_available_balance = vaultAsset.available,
        last_balance_event_id = event.id
```

### Balance update 校验规则

`vault_account.asset.balance_updated` 需要做两类校验：强校验和软校验。

强校验用于决定是否可以最终入账：

```text
transaction.status = COMPLETED
+ transaction.destination vaultAccountId 与 balance_updated.vaultAccountId 一致
+ transaction.assetId 与 balance_updated.assetId 一致
+ balance_updated.blockHeight >= deposit.block_height
+ blockHash 一致，如果 transaction 和 balance update 都提供 blockHash
+ GET /vault/accounts/{vaultAccountId}/{assetId} 返回的 blockHeight 也 >= deposit.block_height
+ GET 返回的 blockHash 与 balance update/transaction 一致，如果这些字段都存在
```

软校验用于风控、对账和异常报警：

```text
当前 Fireblocks vault asset 余额
与上一次 balance watermark 的余额
以及这段区块范围内的已知入账、归集、手续费、补 gas 事件
能够基本对上
```

不要把单笔入账校验写成：

```text
newBalance - oldBalance == deposit.amount
```

这个判断在资金系统里不可靠，因为同一余额水位区间内可能同时发生：

- 同一区块多笔入金。
- 多个区块的 balance update 被合并通知。
- 已经发起 sweep 归集。
- 已经发生 fee top-up。
- 原生资产归集扣除 network fee。
- Solana SOL 余额受 SPL token account、rent、手续费影响。

推荐按 `vaultAccountId + assetId + blockHeight range` 做聚合 delta 校验：

```text
oldHeight = oldWatermark.last_block_height
newHeight = vaultAsset.blockHeight
oldBalance = oldWatermark.last_total_balance
newBalance = vaultAsset.total

incomingSum =
  sum(deposits.amount)
  where vault_account_id = vaultAccountId
    and asset_id = assetId
    and block_height > oldHeight
    and block_height <= newHeight

knownOutgoingSum =
  sum(sweep / outbound transfer amount)
  where source_vault_account_id = vaultAccountId
    and asset_id = assetId
    and block_height > oldHeight
    and block_height <= newHeight

knownFeeSum =
  sum(network fees charged from this vault asset)
  where vault_account_id = vaultAccountId
    and fee_asset_id = assetId
    and block_height > oldHeight
    and block_height <= newHeight

expectedDelta = incomingSum - knownOutgoingSum - knownFeeSum
actualDelta = newBalance - oldBalance
```

如果系统完整掌握这个区间内所有出入账和手续费，可以使用严格校验：

```text
actualDelta == expectedDelta
```

如果系统不能完整掌握所有余额变动，则使用容忍校验和报警：

```text
actualDelta >= incomingSum - knownOutgoingSum - knownFeeSum - tolerance
```

实时入账不应因为余额 delta 无法严格相等而直接丢弃 deposit；更稳妥的处理是：

1. 强校验通过，允许生成 `DEPOSIT_CONFIRMED`。
2. 软校验失败，将 vault asset 标记为 `BALANCE_REVIEW_REQUIRED` 并报警。
3. 对该 vault asset 后续 sweep 可选择降速、暂停或进入人工复核。

### 同一区块多笔入金

`vault_account.asset.balance_updated` 表示某个 `vaultAccountId + assetId` 的余额已经更新到某个链上区块水位，它不是单笔入金的唯一凭证。因此，同一个用户 vault 在同一区块收到两笔或多笔转账时，处理方式是：

1. 每个 `transaction.status.updated` 都按自己的 Fireblocks transaction id、`txHash`、`blockchainIndex/logIndex/index` 生成一条独立 `deposits` 记录。
2. 这些 deposit 可以拥有相同的 `destination_vault_account_id`、`asset_id`、`block_height`、`block_hash`，但 `deposit_key` 必须不同。
3. 收到该 vault asset 的 `balance_updated` 后，把所有 `status = TX_COMPLETED_WAIT_BALANCE` 且 `block_height <= balanceBlockHeight` 的 deposit 都作为候选。
4. 对候选 deposit 逐笔执行幂等确认：每笔 deposit 单独插入一条 `DEPOSIT_CONFIRMED`，并单独创建对应 sweep task。
5. 如果两笔入金在同一区块，通常会被同一次 `balance_updated` 批量释放；如果其中某笔 `transaction.status.updated` 晚到，后续补偿任务也会用当前 vault asset 区块水位把它补确认。

示例：

```text
tx A: deposit_key = txA:0, amount = 100 USDC, block_height = 1000
tx B: deposit_key = txB:0, amount = 50 USDC,  block_height = 1000

balance_updated:
  vaultAccountId = userVault1
  assetId = USDC_ARB
  blockHeight = 1000

处理结果：
  deposit A -> DEPOSIT_CONFIRMED -> sweep task A
  deposit B -> DEPOSIT_CONFIRMED -> sweep task B
```

如果一次链上交易中包含多笔转入同一个 vault asset，也不能合并成一笔 deposit。仍然要用 Fireblocks 返回的 `blockchainIndex/logIndex/index` 区分，生成多条 `deposit_key` 不同的记录。

确认入账与创建归集任务必须在同一个 MySQL 事务内：

```text
begin transaction
  select deposit for update

  if deposit.status already confirmed:
    commit
    return

  update deposits
    set status = CONFIRMED,
        balance_confirmed = 1,
        raw_balance_event = event.payload

  insert ledger_events DEPOSIT_CONFIRMED
    on duplicate key ignore

  insert sweep_tasks
    status = PENDING,
    external_tx_id = sweep:deposit:{deposit_id}:asset:{asset_id}
    on duplicate key ignore

commit
```

## 归集执行

归集交易使用 Fireblocks `POST /transactions`。

请求结构：

```json
{
  "assetId": "ASSET_ID",
  "amount": "AMOUNT",
  "externalTxId": "sweep:deposit:123:asset:ASSET_ID",
  "source": {
    "type": "VAULT_ACCOUNT",
    "id": "USER_VAULT_ACCOUNT_ID"
  },
  "destination": {
    "type": "VAULT_ACCOUNT",
    "id": "TREASURY_VAULT_ACCOUNT_ID"
  },
  "note": "sweep deposit 123"
}
```

执行流程：

```text
processSweepTask(task):
  select task for update
  if status not in [PENDING, RETRYABLE_FAILED]:
    return

  call Fireblocks create transaction with externalTxId

  update sweep_tasks:
    fireblocks_tx_id = response.id
    status = SUBMITTED
    attempts = attempts + 1

  insert ledger_events SWEEP_CREATED
    on duplicate key ignore
```

注意：

- 对 account-based 链，归集一般按用户 vault、按资产逐笔或按可用余额执行。
- 如果你要求“每笔转账后立即归集”，则每个 deposit 创建一个 sweep task。
- 如果手续费敏感，可以改为按 vault+asset 聚合归集，但记账仍按 deposit 独立生成。
- 用户账面余额不应因为 sweep 成功或失败改变；sweep 只是资金库存位置变化。

## Arbitrum 与 Solana 特殊点

Arbitrum：

- ERC-20 归集需要源 vault 有 Arbitrum ETH 支付 gas。
- 建议启用 Fireblocks Gas Station / auto fueling。
- 监听 `transaction.alert.stuck_confirming`，对卡住的 EVM 交易进行 boost 或人工处理。
- token deposit 可能有 EVM `logIndex`，应纳入 `deposit_key`。

Solana：

- SPL token 归集需要源 vault 有 SOL 支付手续费。
- 部分 token 账户创建、rent、ATA 相关行为可能影响实际可归集时间。
- Solana 的 `blockchainIndex` 可能对应 nested instruction 位置，应纳入 `deposit_key`。
- 小额 SPL token 归集前要确认可用余额和手续费策略。

## 补偿与对账任务

### Pending deposit reconciliation

周期：每 1 到 5 分钟。

目标：

- 找出 `TX_COMPLETED_WAIT_BALANCE` 超过阈值的 deposits。
- 调 `GET /transactions/{txId}` 获取最新 `blockInfo`。
- 调 `GET /vault/accounts/{vaultAccountId}/{assetId}` 获取最新余额区块。
- 如果满足区块校验，补做 `DEPOSIT_CONFIRMED` 和 sweep task。
- 如果长时间不满足，标记 `RECONCILE_TIMEOUT` 并报警。

### Webhook gap recovery

目标：

- 当 webhook 服务停机或 Fireblocks circuit breaker 生效后，使用 Fireblocks webhook resend 能力或交易查询 API 补齐事件。
- 按 `fireblocks_webhook_events` 的缺口时间窗口重放。

### Sweep retry

周期：每 1 到 5 分钟。

目标：

- 找出 `SWEEP_FAILED`、`RETRYABLE_FAILED`、长时间 `SUBMITTED/CONFIRMING` 的任务。
- 如果未创建 Fireblocks tx，使用同一个 `externalTxId` 重试创建。
- 如果已创建 Fireblocks tx，先查询交易状态，不要创建新 sweep。
- 对 EVM stuck 交易使用 Fireblocks boost 机制或进入人工处理。

### Daily reconciliation

目标：

- 按用户 vault account 和 asset 汇总：
  - Fireblocks vault balance
  - MySQL deposits
  - MySQL ledger posted amount
  - sweep completed amount
- 检查用户账面余额与链上资金库存是否一致。
- 输出差异报表，所有差异必须可追踪到 deposit、ledger event、sweep task。

## 幂等策略

| 场景 | 幂等键 |
| --- | --- |
| Webhook 原始事件 | `event.id` |
| Deposit | `deposit_key` |
| Ledger event | `(ref_type, ref_id, event_type)` |
| Sweep task | `deposit_id` |
| Fireblocks sweep transaction | `externalTxId` |

所有 worker 都应允许重复执行。重复执行时，只能得到同一个最终状态，不能生成第二笔用户账或第二笔归集交易。

## 事务边界

推荐事务：

1. `transaction.status.updated` 入账事件：
   - upsert `deposits`
   - insert `DEPOSIT_DETECTED`

2. 余额确认：
   - lock `deposits`
   - update `deposits.status = CONFIRMED`
   - insert `DEPOSIT_CONFIRMED`
   - insert `sweep_tasks`

3. 归集提交：
   - lock `sweep_tasks`
   - call Fireblocks 前先把 task 标记为 `SUBMITTING` 可选
   - call Fireblocks 后保存 `fireblocks_tx_id`
   - insert `SWEEP_CREATED`

对第 3 点，如果不希望事务持有期间进行外部 API 调用，可以使用两阶段：

```text
PENDING -> SUBMITTING
commit
call Fireblocks
SUBMITTING -> SUBMITTED or RETRYABLE_FAILED
```

之后补偿任务负责处理卡在 `SUBMITTING` 的任务。

## 安全要求

- Webhook endpoint 必须是 HTTPS。
- 必须校验 `Fireblocks-Webhook-Signature`。
- 原始 body 必须保留用于验签和审计。
- Fireblocks API key 权限最小化；归集服务只授予必要交易创建权限。
- API private key 放入 KMS/Secrets Manager，不能落盘到镜像。
- Webhook endpoint 做 IP allowlist 时，仍然保留签名校验。
- 所有入账、记账、归集状态变更写审计日志。

## 监控告警

关键指标：

- webhook 接收 QPS、验签失败数、重复事件数
- queue lag
- `TX_COMPLETED_WAIT_BALANCE` 数量和最长等待时间
- `BALANCE_MISMATCH` 数量
- `RECONCILE_TIMEOUT` 数量
- sweep 创建成功率、失败率、平均完成时间
- `SWEEP_STUCK` 数量
- Fireblocks API 错误率、限流次数

建议告警：

- 任意 `BALANCE_MISMATCH`
- `TX_COMPLETED_WAIT_BALANCE` 超过 10 到 30 分钟
- `SUBMITTING` 超过 5 分钟
- `SWEEP_SUBMITTED/CONFIRMING` 超过链上合理确认时间
- Fireblocks webhook endpoint 被暂停或错误率升高

## 用户侧展示建议

用户入金状态：

```text
DETECTED
  文案：充值已检测，确认中

CONFIRMED
  文案：充值成功

RECONCILE_TIMEOUT / BALANCE_MISMATCH
  文案：充值确认中，请联系客服或等待处理
```

不要把 Fireblocks sweep 状态暴露为用户充值成功的前置条件。用户充值成功只依赖 deposit confirmed 和 ledger posted；sweep 是平台资金管理动作。

## 上线检查清单

- Fireblocks Webhooks v2 已订阅 `transaction.status.updated` 和 `vault_account.asset.balance_updated`。
- Webhook 签名校验已接入，并覆盖 US/EU/EU2/Sandbox 对应 JWKS。
- `fireblocks_webhook_events.event_id` 唯一约束已生效。
- `deposits.deposit_key` 唯一约束已生效。
- `ledger_events(ref_type, ref_id, event_type)` 唯一约束已生效。
- `sweep_tasks.external_tx_id` 唯一约束已生效。
- Fireblocks `POST /transactions` 归集请求已传 `externalTxId`。
- Arbitrum 用户 vault 有 gas 来源，Gas Station / auto fueling 策略已验证。
- Solana 用户 vault 有 SOL 手续费来源，SPL token 归集已在测试资产验证。
- Worker 重试不会生成重复 ledger event 或重复 sweep transaction。
- 补偿任务可以处理 webhook 乱序、重复、延迟和短暂停机。
- 每日对账报表可以按 vault account、asset、user、deposit 追溯差异。

## 推荐实施顺序

1. 建立 user vault 绑定表和资产白名单。
2. 上线 webhook ingress：验签、原始事件落库、入队、快速 2xx。
3. 实现 `transaction.status.updated` 入账检测，生成 `DEPOSIT_DETECTED`。
4. 实现 `vault_account.asset.balance_updated` 校验，生成 `DEPOSIT_CONFIRMED`。
5. 实现 ledger posted 后的 sweep task 创建。
6. 实现 Fireblocks sweep worker，使用 `externalTxId`。
7. 实现 sweep 状态回写。
8. 增加 pending deposit reconciliation、sweep retry、daily reconciliation。
9. 小额测试 Arbitrum 原生资产、Arbitrum ERC-20、Solana SOL、Solana SPL token。
10. 灰度开放真实用户入金。
