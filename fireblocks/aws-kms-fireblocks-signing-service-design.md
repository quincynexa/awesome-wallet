# Fireblocks API AWS KMS 签名服务方案

## 背景

Fireblocks API 使用 API key 和 JWT 签名认证。每个请求需要：

- `X-API-Key`: Fireblocks API User ID
- `Authorization: Bearer <JWT>`

JWT payload 需要包含 `uri`、`nonce`、`iat`、`exp`、`sub`、`bodyHash`，并且 JWT 必须使用 Fireblocks API 私钥按 RS256 签名。Fireblocks 文档说明 RS256 即 `RSASSA-PKCS1-v1_5 using SHA-256`。Fireblocks 创建 API user 时需要上传 CSR，Fireblocks 保存 CSR 中的公钥，用它验证之后的 API 请求签名。

本方案目标是：Fireblocks API 私钥不落在业务服务、容器镜像、环境变量、磁盘文件或 CI 日志中；所有 API JWT 签名都通过 AWS KMS 非对称签名能力完成，并由独立签名服务统一审计、限流和授权。

参考文档：

- Fireblocks Authenticate: https://developers.fireblocks.com/reference/signing-a-request-jwt-structure
- Fireblocks Manage API Access: https://developers.fireblocks.com/docs/manage-api-keys
- Fireblocks Create CSR: https://developers.fireblocks.com/docs/generate-a-csr-for-an-api-user
- AWS KMS Sign API: https://docs.aws.amazon.com/kms/latest/APIReference/API_Sign.html
- AWS KMS Asymmetric keys: https://docs.aws.amazon.com/kms/latest/developerguide/symmetric-asymmetric.html
- AWS KMS Imported key material: https://docs.aws.amazon.com/kms/latest/developerguide/importing-keys.html
- AWS KMS Imported key requirements: https://docs.aws.amazon.com/kms/latest/developerguide/importing-keys-conceptual.html

## 关键结论

1. 新建 Fireblocks API user 时，优先使用 AWS KMS 生成 `RSA_4096`、`SIGN_VERIFY`、`RSASSA_PKCS1_V1_5_SHA_256` 的非对称 KMS key，再用 KMS 对 CSR 进行签名。这样私钥从未以明文形式出现。
2. 已存在的 Fireblocks 私钥可以迁移到 AWS KMS，但应使用 `Origin=EXTERNAL` 的 asymmetric KMS key 导入 RSA 私钥材料。导入材料必须是 PKCS#8 BER/DER 编码，并且 key spec 与 KMS key 一致。
3. 不建议把 Fireblocks 私钥 PEM 放到 Secrets Manager 后由应用读取再本地签名。Secrets Manager 适合保存 API key、base URL、key ARN 等配置，不适合让业务服务取出 Fireblocks 私钥。
4. 业务服务不要直接调用 AWS KMS `Sign`。建议通过内部 Fireblocks Signing Service 统一生成 JWT，集中做 URI/bodyHash 计算、权限校验、审计、限流和重放防护。
5. 不能直接使用官方 Fireblocks SDK 的默认私钥文件模式，除非对 SDK 做自定义 signer 适配；否则它会期望读取 PEM 私钥并在进程内签名。

## 总体架构

```text
Business Services
  -> Fireblocks API Client
      -> request JWT from Signing Service
      -> call Fireblocks REST API

Fireblocks Signing Service
  -> validate caller identity and scope
  -> build JWT header and payload
  -> hash JWT signing input
  -> AWS KMS Sign
  -> return compact JWT
  -> write audit log

AWS KMS
  -> asymmetric RSA_4096 SIGN_VERIFY key
  -> signing algorithm RSASSA_PKCS1_V1_5_SHA_256
  -> private key never leaves KMS unencrypted

Fireblocks
  -> stores public key from CSR
  -> verifies JWT signature
```

## KMS key 设计

### 推荐方案：KMS 原生生成密钥

适用于新建 Fireblocks API user。

KMS key 参数：

```text
Key type: Asymmetric
Key usage: SIGN_VERIFY
Key spec: RSA_4096
Origin: AWS_KMS
Signing algorithm: RSASSA_PKCS1_V1_5_SHA_256
Alias: alias/fireblocks/{env}/{api-user-name}/jwt-signing
```

优点：

- 私钥不会以明文形式生成、导出或被应用读取。
- KMS key policy、IAM、CloudTrail 可以完整控制和审计签名行为。
- 避免私钥文件在离线机器、CI、容器、运维终端之间传递。

限制：

- Fireblocks 创建 API user 需要 CSR。因为私钥不能导出，不能用普通 `openssl req -new -key fireblocks_secret.key` 生成 CSR。
- 需要实现一个 CSR 生成工具：用 KMS `GetPublicKey` 获取公钥，构造 PKCS#10 `CertificationRequestInfo`，再调用 KMS `Sign` 对 CSR 内容签名。

### 兼容方案：导入已有 Fireblocks 私钥

适用于已经用 OpenSSL 创建过 Fireblocks API user，并且 Fireblocks 已保存该私钥对应公钥的场景。

KMS key 参数：

```text
Key type: Asymmetric
Key usage: SIGN_VERIFY
Key spec: RSA_4096
Origin: EXTERNAL
Signing algorithm: RSASSA_PKCS1_V1_5_SHA_256
```

导入要求：

- 只导入 RSA 私钥材料，AWS KMS 会从私钥派生公钥。
- RSA 私钥必须是 2048/3072/4096 bits 之一；Fireblocks 推荐 RSA 4096。
- RSA 必须是双素数 RSA，不支持 multi-prime RSA。
- 私钥材料必须是 PKCS#8 BER/DER 编码。

风险：

- 迁移阶段私钥仍然会在离线机器上短暂存在。
- 一旦明文私钥曾经被复制、传输或备份，历史泄露风险无法由 KMS 消除。
- 对 asymmetric 或 HMAC imported key，AWS KMS 只允许关联一份 key material，不能像 symmetric imported key 那样做 on-demand key material rotation。

迁移后要求：

- 销毁原 PEM 私钥文件和所有副本。
- 轮换长期暴露过的 API user：新建 KMS 原生密钥、生成 CSR、创建新的 Fireblocks API user、灰度切换、禁用旧 API user。

## Fireblocks API user 创建流程

### 新 API user 流程

```text
1. KMS Admin 创建 RSA_4096 SIGN_VERIFY KMS key
2. Security/Platform 运行 CSR 生成工具
3. CSR 工具调用 KMS GetPublicKey
4. CSR 工具构造 PKCS#10 CertificationRequestInfo
5. CSR 工具调用 KMS Sign:
   SigningAlgorithm = RSASSA_PKCS1_V1_5_SHA_256
   MessageType = RAW 或 DIGEST
6. CSR 工具输出 fireblocks.csr，不输出私钥
7. Admin 在 Fireblocks Console 或 Create API Key API 上传 CSR
8. Fireblocks API user 审批通过后，保存 API key 到 Secrets Manager
9. Signing Service 配置 API key、KMS key ARN、base URL、允许权限范围
10. 调用 Fireblocks whoami / users/me 做验收
```

CSR 生成工具注意事项：

- CSR signature algorithm 必须与 KMS 签名算法一致，使用 `sha256WithRSAEncryption`。
- CSR subject 只放必要字段，例如组织名、环境名、API user 名称。
- CSR 文件不敏感，但仍要走受控发布流程，避免上传错环境。
- 工具产物禁止包含私钥、临时 PEM、debug signature input。

### 已有私钥迁移流程

```text
1. 在离线安全环境确认 fireblocks_secret.key 与 Fireblocks 当前 API user 匹配
2. 将 RSA private key 转为 PKCS#8 DER
3. 创建 Origin=EXTERNAL、RSA_4096、SIGN_VERIFY 的 KMS key
4. 调用 GetParametersForImport 获取 wrapping public key 和 import token
5. 使用 AWS 指定 wrapping algorithm 加密 PKCS#8 DER 私钥材料
6. 调用 ImportKeyMaterial
7. 使用 KMS GetPublicKey 导出公钥并与原 CSR/证书公钥比对
8. Signing Service 使用新 KMS key 生成 JWT
9. 调 Fireblocks whoami / users/me 验证
10. 销毁原私钥文件和中间材料
```

示例转换命令：

```bash
openssl pkcs8 \
  -topk8 \
  -inform PEM \
  -outform DER \
  -in fireblocks_secret.key \
  -out fireblocks_secret.pk8.der \
  -nocrypt
```

该命令只应在离线安全环境执行，输出文件应在导入成功并验收后立即安全删除。

## 签名服务职责

签名服务负责生成 Fireblocks JWT，而不是直接代理所有 Fireblocks API 请求。推荐接口：

```http
POST /internal/fireblocks/sign-jwt
Content-Type: application/json

{
  "workspace": "prod-us",
  "method": "POST",
  "uri": "/v1/transactions",
  "body": "{\"assetId\":\"ETH_TEST\",...}",
  "purpose": "create_transaction",
  "idempotencyKey": "business-request-id"
}
```

响应：

```json
{
  "apiKey": "masked-or-full-api-key-for-header",
  "authorization": "Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresAt": 1778300000,
  "nonce": "01HX..."
}
```

签名服务内部步骤：

```text
1. 认证调用方身份：mTLS、IAM role、service account token 或内部网关身份
2. 授权调用方是否允许请求该 workspace、method、uri、purpose
3. 生成 nonce，保证单 workspace 单 API key 下唯一
4. 计算 bodyHash = hex(sha256(raw_http_body))
5. 构造 JWT payload:
   uri, nonce, iat, exp, sub, bodyHash
6. 构造 JWT header:
   alg = RS256
   typ = JWT
7. 计算 signingInput:
   base64url(header) + "." + base64url(payload)
8. sha256(signingInput)
9. 调 KMS Sign:
   KeyId = configured KMS key ARN
   Message = sha256(signingInput)
   MessageType = DIGEST
   SigningAlgorithm = RSASSA_PKCS1_V1_5_SHA_256
10. base64url(signature) 拼接 compact JWT
11. 写审计日志
12. 返回 JWT
```

也可以把 `signingInput` 作为 `Message` 并使用 `MessageType=RAW`。推荐使用 `DIGEST`，因为语义更明确，且 Fireblocks JWT signing input 很短但签名服务可以统一按 digest 处理。

## JWT 规范

Header：

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

Payload：

```json
{
  "uri": "/v1/transactions",
  "nonce": "01HXX...",
  "iat": 1778300000,
  "exp": 1778300025,
  "sub": "fireblocks-api-key",
  "bodyHash": "hex-encoded-sha256-of-raw-body"
}
```

约束：

- `uri` 必须是请求 URI path 和 query，不包含 scheme、host。
- `bodyHash` 必须基于最终发送给 Fireblocks 的 raw HTTP body 字节计算。
- GET/DELETE 无 body 时，body 应固定为 empty string，对应 SHA-256 空字符串 hash。
- `exp` 必须小于 `iat + 30` 秒，建议设置为 `iat + 25` 秒。
- `nonce` 全局唯一，建议使用 ULID/UUIDv7，并将最近 24 小时 nonce 写入 Redis 或数据库防重。
- 不要缓存 JWT 给多次请求复用。每次 Fireblocks API 调用生成一个新 JWT。

## 业务服务调用方式

业务服务调用 Fireblocks API 前：

```text
1. 准备 method、uri、rawBody
2. 调 Signing Service 获取 JWT
3. 使用签名服务返回的 apiKey 和 authorization header 调 Fireblocks
4. Fireblocks API 使用业务自己的幂等键，比如 Idempotency-Key
5. 将 Fireblocks response 与 signing audit id 关联落库
```

业务服务不要：

- 持有 Fireblocks 私钥 PEM。
- 调用 KMS `Sign`。
- 自己拼 Fireblocks JWT。
- 在日志中打印 JWT、raw private key、KMS signature、完整 API key。

## IAM 与 KMS key policy

角色划分：

```text
KMS Admin Role
  创建/禁用/排期删除 KMS key
  管理 alias、tag、key policy
  不能调用 kms:Sign

Key Import Role
  仅迁移已有私钥时使用
  kms:GetParametersForImport
  kms:ImportKeyMaterial
  kms:DeleteImportedKeyMaterial
  不常驻，不给业务运行时使用

Signing Service Role
  kms:Sign
  kms:GetPublicKey 可选
  只能访问指定 alias/fireblocks/{env}/*

Audit Role
  读 CloudTrail、CloudWatch Logs、KMS key metadata
  不能调用 kms:Sign
```

示例 Signing Service IAM policy：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFireblocksJwtSigningOnly",
      "Effect": "Allow",
      "Action": [
        "kms:Sign"
      ],
      "Resource": [
        "arn:aws:kms:us-east-1:123456789012:key/KEY_ID"
      ],
      "Condition": {
        "StringEquals": {
          "kms:SigningAlgorithm": "RSASSA_PKCS1_V1_5_SHA_256"
        }
      }
    }
  ]
}
```

Key policy 也应显式限制：

- 只有 Signing Service Role 可以 `kms:Sign`。
- Admin Role 不能 `kms:Sign`。
- Key Import Role 不能长期存在。
- 生产与测试环境使用不同 AWS account 或至少不同 KMS key、不同 IAM role。

## 签名服务内部授权

签名服务不能成为“任意 Fireblocks 请求签名器”。必须按调用方、环境、URI、method、purpose 做白名单。

示例授权表：

```sql
CREATE TABLE fireblocks_signing_policies (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  caller_service VARCHAR(128) NOT NULL,
  workspace VARCHAR(64) NOT NULL,
  api_key_ref VARCHAR(128) NOT NULL,
  method VARCHAR(16) NOT NULL,
  uri_pattern VARCHAR(255) NOT NULL,
  purpose VARCHAR(64) NOT NULL,
  enabled TINYINT NOT NULL DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_policy (caller_service, workspace, method, uri_pattern, purpose)
);
```

建议策略：

- deposit/sweep worker 只允许 `POST /v1/transactions`、`GET /v1/transactions/*`、必要 vault 查询接口。
- admin 工具只允许非资金转移类管理接口，生产环境默认关闭高危接口。
- 不同 Fireblocks API user 对应不同 Fireblocks 角色；签名服务策略不能放大 Fireblocks 侧权限。

## 审计日志

签名服务每次签名必须记录：

```sql
CREATE TABLE fireblocks_signing_audit_logs (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  request_id VARCHAR(128) NOT NULL,
  caller_service VARCHAR(128) NOT NULL,
  workspace VARCHAR(64) NOT NULL,
  api_key_ref VARCHAR(128) NOT NULL,
  kms_key_id VARCHAR(256) NOT NULL,
  method VARCHAR(16) NOT NULL,
  uri VARCHAR(512) NOT NULL,
  body_hash CHAR(64) NOT NULL,
  nonce VARCHAR(128) NOT NULL,
  iat BIGINT NOT NULL,
  exp BIGINT NOT NULL,
  purpose VARCHAR(64) NOT NULL,
  decision VARCHAR(32) NOT NULL,
  denial_reason VARCHAR(255) NULL,
  fireblocks_idempotency_key VARCHAR(128) NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY uk_request_id (request_id),
  UNIQUE KEY uk_nonce (workspace, api_key_ref, nonce),
  KEY idx_caller_time (caller_service, created_at),
  KEY idx_uri_time (uri, created_at)
);
```

日志禁止存储：

- JWT 完整值
- KMS signature 原文
- Fireblocks 私钥
- 完整敏感业务 body

可以存储：

- `bodyHash`
- masked API key ref
- KMS key ARN 或 alias
- URI、method、purpose
- caller identity
- request id、idempotency key

## Secrets 与配置

建议配置项：

```text
FIREBLOCKS_WORKSPACE=prod-us
FIREBLOCKS_BASE_URL=https://api.fireblocks.io/v1
FIREBLOCKS_API_KEY_SECRET_ARN=arn:aws:secretsmanager:...
FIREBLOCKS_KMS_KEY_ARN=arn:aws:kms:...
FIREBLOCKS_KMS_SIGNING_ALGORITHM=RSASSA_PKCS1_V1_5_SHA_256
JWT_TTL_SECONDS=25
```

Secrets Manager 存储：

```json
{
  "apiKey": "Fireblocks API User ID",
  "baseUrl": "https://api.fireblocks.io/v1",
  "kmsKeyArn": "arn:aws:kms:us-east-1:123456789012:key/..."
}
```

API key 本身不是签名私钥，但仍应视为敏感配置：

- 不提交代码库。
- 不打印完整值。
- 只允许签名服务运行时读取。
- 按环境隔离。

## 可用性与性能

AWS KMS `Sign` 是在线调用，会增加每个 Fireblocks API 请求的延迟。建议：

- Signing Service 部署多副本，跨 AZ。
- 使用短超时，例如 1 到 2 秒。
- 对 KMS 5xx、timeout 做有限重试，使用指数退避。
- 不缓存 JWT，但可以缓存 Secrets Manager 配置和 KMS public key。
- 高 QPS 场景提前评估 KMS TPS quota，并申请提升。
- 对 Fireblocks 创建交易类请求，业务层必须使用 Fireblocks `Idempotency-Key`。

不可用处理：

- KMS 不可用时，拒绝生成 JWT，不降级到本地私钥签名。
- 不保留热备 PEM 私钥。
- 对资金操作类请求，失败后由业务 worker 基于幂等键重试。

## 轮换方案

Fireblocks API user 的轮换本质是公私钥对轮换，通常需要新建 API user。

推荐流程：

```text
1. 创建新的 KMS RSA_4096 SIGN_VERIFY key
2. 用新 KMS key 生成 CSR
3. 在 Fireblocks 创建新 API user，授予最小权限
4. Signing Service 新增 api_key_ref 和 kms_key_ref
5. 灰度少量调用到新 API user
6. 全量切换
7. 禁用旧 API user
8. 排期删除旧 KMS key，保留审计日志
```

不建议：

- 试图在同一个 Fireblocks API user 下替换私钥。
- 长期保留旧 API user 和新 API user 并行可写权限。
- 用同一把 KMS key 服务多个环境或多个权限等级的 API user。

## 监控告警

关键指标：

- `sign_requests_total`
- `sign_denied_total`
- `kms_sign_latency_ms`
- `kms_sign_error_total`
- `fireblocks_auth_error_total`
- `jwt_expired_error_total`
- `nonce_duplicate_total`
- `policy_miss_total`

建议告警：

- 任意生产环境 `kms:Sign` 被非 Signing Service Role 调用。
- 生产 KMS key policy 被修改。
- KMS key 被 disable、schedule deletion、delete imported material。
- `fireblocks_auth_error_total` 突增。
- `kms_sign_error_total` 连续 5 分钟高于阈值。
- 同一 caller 在短时间内请求大量不同 URI 的签名。
- 出现未授权 URI 的签名请求。

CloudTrail 重点事件：

- `Sign`
- `GetPublicKey`
- `CreateKey`
- `ScheduleKeyDeletion`
- `DisableKey`
- `PutKeyPolicy`
- `ImportKeyMaterial`
- `DeleteImportedKeyMaterial`

## 安全边界

必须做到：

- Fireblocks 私钥只存在于 AWS KMS 中。
- Signing Service 是唯一拥有 `kms:Sign` 权限的运行时组件。
- 业务服务只能通过内部鉴权访问 Signing Service。
- 签名服务只签允许的 Fireblocks URI 和 method。
- 每个 JWT 只用于一次 Fireblocks 请求。
- 签名服务审计日志与 Fireblocks 业务请求落库记录能互相追溯。

严禁：

- 将 Fireblocks 私钥 PEM 写入环境变量。
- 将私钥存入普通数据库或配置中心。
- 在 CI/CD 中生成生产私钥并作为 artifact 传递。
- 将 KMS `Sign` 权限授予所有业务服务。
- 在日志中打印 JWT、signature、完整 API key 或 raw request body。

## 验收测试

### CSR 与 API user 验收

```text
1. 使用 KMS GetPublicKey 导出公钥
2. 校验 CSR 中公钥与 KMS 公钥一致
3. 上传 CSR 创建 Fireblocks API user
4. 使用 Signing Service 生成 JWT
5. 调用 GET /v1/users/me 或等价 whoami 接口
6. 确认 Fireblocks 返回 200
```

### JWT 兼容性验收

验证以下场景：

- GET 空 body，`bodyHash = sha256("")`
- POST JSON body，bodyHash 与最终发送字节完全一致
- URI 带 query string
- `exp = iat + 25`
- 重放同一个 JWT 被拒绝或至少不被业务复用
- 错误 bodyHash 会被 Fireblocks 拒绝
- 错误 KMS key 生成的 JWT 会被 Fireblocks 拒绝

### 权限验收

验证以下场景：

- 未授权业务服务不能请求签名。
- 授权业务服务不能请求未授权 URI。
- Signing Service Role 可以 `kms:Sign`。
- KMS Admin Role 不能 `kms:Sign`。
- 业务服务 Role 不能直接 `kms:Sign`。

## 实施顺序

1. 确认 Fireblocks API user 权限模型，拆分 signer、viewer、admin 等角色。
2. 创建 dev/sandbox KMS `RSA_4096 SIGN_VERIFY` key。
3. 实现 KMS CSR 生成工具。
4. 在 Fireblocks sandbox 创建 API user。
5. 实现 Signing Service JWT 生成和审计日志。
6. 实现业务 Fireblocks client 适配，不再读取 PEM 私钥。
7. 完成 sandbox whoami、vault 查询、创建交易等验收。
8. 上线生产 KMS key、CSR、API user。
9. 灰度迁移 deposit/sweep worker。
10. 禁用旧私钥文件路径和旧 API user。

## 与入金归集方案的关系

入金、记账、归集方案中的 Sweep Worker 需要调用 Fireblocks `POST /transactions` 创建归集交易。接入本签名服务后：

```text
Sweep Worker
  -> build Fireblocks create transaction request
  -> call Signing Service for JWT
  -> call Fireblocks POST /v1/transactions
  -> persist fireblocks_tx_id and signing_audit_id
```

这样归集 worker 不再持有 Fireblocks 私钥。即使归集 worker 被攻破，攻击者仍需要通过 Signing Service 的内部授权策略，不能任意构造 Fireblocks API 请求。
