# Bounce Webhook 系统深度分析

## 目录
1. [整体架构概述](#整体架构概述)
2. [各 Provider 认证方式对比](#各-provider-认证方式对比)
3. [认证失败处理差异](#认证失败处理差异)
4. [Bounce 处理核心流程](#bounce-处理核心流程)
5. [Source/Type/Action 决策机制](#sourcetypeaction-决策机制)
6. [SQL 查询逻辑深度分析](#sql-查询逻辑深度分析)
7. [生产问题根因分析](#生产问题根因分析)
8. [修复建议](#修复建议)

---

## 整体架构概述

### 核心组件调用链

```
Webhook Request → BounceWebhook(c) [cmd/bounce.go:116]
    ↓
    ├─→ 各 Provider ProcessBounce() [internal/bounce/webhooks/]
    │       ↓
    │       └─→ 签名验证 + 事件解析
    ↓
    ├─→ bounce.Manager.Record(b) [internal/bounce/bounce.go:146]
    │       ↓
    │       └─→ 写入 channel 队列
    ↓
    └─→ bounce.Manager.Run() [internal/bounce/bounce.go:118]
            ↓
            └─→ RecordBounceCB(b) → core.RecordBounce(b) [internal/core/bounces.go:59]
                    ↓
                    └─→ record-bounce SQL [queries/bounces.sql:1]
```

### 关键文件索引

| 组件 | 文件路径 | 关键行 |
|------|---------|--------|
| Webhook 入口 | [cmd/bounce.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/bounce.go#L116-L256) | L116-L256 |
| Bounce Manager | [internal/bounce/bounce.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/bounce.go) | L46-L150 |
| Core RecordBounce | [internal/core/bounces.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/bounces.go#L59-L87) | L59-L87 |
| Bounce SQL | [queries/bounces.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/bounces.sql#L1-L30) | L1-L30 |
| Bounce Model | [models/bounces.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/bounces.go) | L8-L34 |

---

## 各 Provider 认证方式对比

### 1. SES (Amazon Simple Email Service)

**认证机制**: X.509 证书签名验证

- **实现文件**: [internal/bounce/webhooks/ses.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/webhooks/ses.go#L198-L212)
- **验证流程**:
  1. 从 `SigningCertURL` 下载 AWS 证书（URL 格式校验: `sns.*.amazonaws.com`）
  2. 缓存证书避免重复下载
  3. 使用 SHA1WithRSA 算法验证消息签名
  4. 签名字段包含: Message, MessageId, Subject, SubscribeURL, Timestamp, Token, TopicArn, Type

**关键代码**:
```go
// verifyNotif verifies the signature on a notification payload.
func (s *SES) verifyNotif(n sesNotif) error {
    cert, err := s.getCert(n.SigningCertURL)  // 下载并缓存证书
    sign, _ := base64.StdEncoding.DecodeString(n.Signature)
    return cert.CheckSignature(x509.SHA1WithRSA, s.buildSignature(n), sign)
}
```

**配置项**: 无密钥配置，仅需启用 `bounce.ses_enabled`

---

### 2. Sendgrid

**认证机制**: ECDSA 公钥验签

- **实现文件**: [internal/bounce/webhooks/sendgrid.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/webhooks/sendgrid.go#L89-L114)
- **验证流程**:
  1. 从配置读取 Base64 编码的公钥 `bounce.sendgrid_key`
  2. 解析为 ECDSA 公钥
  3. 使用 SHA256 哈希: `timestamp + body`
  4. 验证签名: `X-Twilio-Email-Event-Webhook-Signature`

**关键代码**:
```go
func (s *Sendgrid) verifyNotif(sig, timestamp string, b []byte) error {
    sigB, _ := base64.StdEncoding.DecodeString(sig)
    h := sha256.New()
    h.Write([]byte(timestamp))
    h.Write(b)
    hash := h.Sum(nil)
    if !ecdsa.Verify(s.pubKey, hash, ecdsaSig.R, ecdsaSig.S) {
        return errors.New("invalid signature")
    }
    return nil
}
```

**配置项**:
- `bounce.sendgrid_enabled`: 启用开关
- `bounce.sendgrid_key`: Base64 编码的公钥

---

### 3. Postmark

**认证机制**: HTTP Basic Auth

- **实现文件**: [internal/bounce/webhooks/postmark.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/webhooks/postmark.go#L43-L117)
- **验证流程**:
  1. 使用 Echo 框架的 `middleware.BasicAuth`
  2. 比较用户名密码（使用 `subtle.ConstantTimeCompare` 防时序攻击）
  3. **空密码放行**: 如果配置的用户名/密码为空，则直接通过验证

**关键代码**:
```go
func makePostmarkAuthHandler(cfgUser, cfgPassword string) func(username, password string, c echo.Context) (bool, error) {
    return func(username, password string, c echo.Context) (bool, error) {
        if len(u) == 0 || len(p) == 0 {
            return true, nil  // ⚠️ 空配置则跳过验证
        }
        return subtle.ConstantTimeCompare([]byte(username), u) == 1 &&
               subtle.ConstantTimeCompare([]byte(password), p) == 1, nil
    }
}
```

**配置项**:
- `bounce.postmark.enabled`: 启用开关
- `bounce.postmark.username`: Basic Auth 用户名
- `bounce.postmark.password`: Basic Auth 密码

---

### 4. ForwardEmail

**认证机制**: HMAC-SHA256

- **实现文件**: [internal/bounce/webhooks/forwardemail.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/webhooks/forwardemail.go#L49-L64)
- **验证流程**:
  1. 从 Header `X-Webhook-Signature` 获取十六进制签名
  2. 使用配置的密钥计算 HMAC-SHA256(body)
  3. 比较签名（`hmac.Equal` 防时序攻击）

**关键代码**:
```go
func (p *Forwardemail) ProcessBounce(sigHex string, body []byte) ([]models.Bounce, error) {
    sig, _ := hex.DecodeString(sigHex)
    mac := hmac.New(sha256.New, p.hmacKey)
    mac.Write(body)
    if !hmac.Equal(mac.Sum(nil), sig) {
        return nil, errors.New("invalid signature")
    }
    // ...
}
```

**配置项**:
- `bounce.forwardemail.enabled`: 启用开关
- `bounce.forwardemail.key`: HMAC 密钥

---

### 5. Lettermint

**认证机制**: HMAC-SHA256 + 时间戳重放保护

- **实现文件**: [internal/bounce/webhooks/lettermint.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/webhooks/lettermint.go#L44-L69)
- **验证流程**:
  1. 解析 Header `X-Lettermint-Signature`: `t={timestamp},v1={hex_signature}`
  2. 时间戳校验: 与当前时间差 ≤ 300 秒（5分钟）
  3. 计算 HMAC-SHA256: `f"{timestamp}.{body}"`
  4. 比较签名

**关键代码**:
```go
func (l *Lettermint) ProcessBounce(sig string, body []byte) ([]models.Bounce, error) {
    ts, sigHex, _ := parseLettermintSignature(sig)
    // 重放保护: 时间戳容忍度 300秒
    if math.Abs(float64(time.Now().Unix()-ts)) > 300 {
        return nil, fmt.Errorf("signature timestamp expired")
    }
    mac := hmac.New(sha256.New, l.hmacKey)
    mac.Write([]byte(fmt.Sprintf("%d.%s", ts, body)))
    if !hmac.Equal(mac.Sum(nil), sigB) {
        return nil, fmt.Errorf("invalid signature")
    }
    // ...
}
```

**配置项**:
- `bounce.lettermint.enabled`: 启用开关
- `bounce.lettermint.key`: HMAC 密钥

---

## 认证失败处理差异

| Provider | 认证失败返回 | HTTP 状态码 | 错误信息 | 日志记录 |
|----------|-------------|------------|---------|---------|
| **SES** | `error` → HTTPError | 400 Bad Request | `globals.messages.invalidData` | ✅ `error processing SES notification: %v` |
| **Sendgrid** | `error` → HTTPError | 400 Bad Request | `globals.messages.invalidData` | ✅ `error processing sendgrid notification: %v` |
| **Postmark** | `echo.HTTPError` (直接返回) | 401 Unauthorized | Echo 默认 Basic Auth 错误 | ✅ `error processing postmark notification: %v` |
| **ForwardEmail** | `echo.HTTPError` (直接返回) | 由错误类型决定 | 透传错误信息 | ✅ `error processing forwardemail notification: %v` |
| **Lettermint** | `echo.HTTPError` (直接返回) | 由错误类型决定 | 透传错误信息 | ✅ `error processing lettermint notification: %v` |

### ⚠️ 关键差异点

**Postmark、ForwardEmail、Lettermint vs SES、Sendgrid**:

```go
// SES 和 Sendgrid: 所有错误都转为 400 通用错误
case service == "ses":
    b, err := a.bounce.SES.ProcessBounce(rawReq)
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("globals.messages.invalidData"))
    }

// Postmark、ForwardEmail、Lettermint: echo.HTTPError 直接透传
case service == "postmark":
    bs, err := a.bounce.Postmark.ProcessBounce(rawReq, c)
    if err != nil {
        if _, ok := err.(*echo.HTTPError); ok {
            return err  // ⚠️ 直接返回，保留状态码和消息
        }
        return echo.NewHTTPError(http.StatusBadRequest, ...)
    }
```

**影响**: Postmark Basic Auth 失败会返回 `401 Unauthorized`，而 Sendgrid 签名失败只返回 `400 Bad Request`。

---

## Bounce 处理核心流程

### 1. BounceWebhook 入口 [cmd/bounce.go:116]

```go
func (a *App) BounceWebhook(c echo.Context) error {
    // 1. 读取原始请求体 (用于保存 meta 和签名验证)
    rawReq, _ := io.ReadAll(c.Request().Body)
    
    // 2. 根据 service 参数分发到对应 provider
    switch service {
    case "": /* 原生 webhook */
    case "ses": /* SES 处理 */
    case "sendgrid": /* Sendgrid 处理 */
    case "postmark": /* Postmark 处理 */
    case "forwardemail": /* ForwardEmail 处理 */
    case "lettermint": /* Lettermint 处理 */
    }
    
    // 3. 批量记录 bounces
    for _, b := range bounces {
        if err := a.bounce.Record(b); err != nil {
            a.log.Printf("error recording bounce: %v", err)
        }
    }
    return c.JSON(http.StatusOK, okResp{true})
}
```

### 2. bounce.Manager.Record [internal/bounce/bounce.go:146]

```go
func (m *Manager) Record(b models.Bounce) error {
    m.queue <- b  // 异步写入 channel，非阻塞
    return nil
}
```

**注意**: `Record` 方法只是将 bounce 放入 channel，立即返回 `nil`，不会等待数据库写入。

### 3. bounce.Manager.Run [internal/bounce/bounce.go:118]

```go
func (m *Manager) Run() {
    for b := range m.queue {
        if b.CreatedAt.IsZero() {
            b.CreatedAt = time.Now()
        }
        if err := m.opt.RecordBounceCB(b); err != nil {
            continue  // ⚠️ 记录失败静默跳过
        }
    }
}
```

### 4. core.RecordBounce [internal/core/bounces.go:59]

```go
func (c *Core) RecordBounce(b models.Bounce) error {
    // 1. 根据 bounce type 获取配置的 action
    action, ok := c.consts.BounceActions[b.Type]
    if !ok {
        return echo.NewHTTPError(http.StatusBadRequest, ...)
    }
    
    // 2. 执行 SQL: 插入 bounce + 计数 + 可能的 blocklist/delete
    _, err := c.q.RecordBounce.Exec(
        b.SubscriberUUID, b.Email, b.CampaignUUID,
        b.Type, b.Source, b.Meta, b.CreatedAt,
        action.Count,   // 阈值: 达到多少次触发 action
        action.Action)  // 动作: none / blocklist / unsubscribe / delete
    
    // 3. 错误处理: subscriber 不存在时忽略错误
    if err != nil {
        if pqErr, ok := err.(*pq.Error); ok && pqErr.Column == "subscriber_id" {
            c.log.Printf("bounced subscriber (%s / %s) not found", ...)
            return nil  // ⚠️ 订阅者不存在时静默成功
        }
        c.log.Printf("error recording bounce: %v", err)
    }
    return err
}
```

---

## Source/Type/Action 决策机制

### 1. Bounce Type 映射

| Provider | 事件类型 | 映射到 BounceType |
|----------|---------|------------------|
| **SES** | `Permanent` | `hard` |
| | `Transient` + `Status=5.4.4` (无效域名) | `hard` |
| | `Transient` (其他) | `soft` |
| | `Complaint` | `complaint` |
| **Sendgrid** | `bounce` + `bounce_classification=technical` | `soft` |
| | `bounce` + `bounce_classification=content` | `soft` |
| | `bounce` (其他) | `hard` |
| **Postmark** | `HardBounce`, `BadEmailAddress`, `ManuallyDeactivated` | `hard` |
| | `SoftBounce`, `Transient`, `DnsError`, `SpamNotification`, `VirusNotification`, `DMARCPolicy` | `soft` |
| | `SpamComplaint` | `complaint` |
| **ForwardEmail** | `category=block/recipient/virus/spam` | `hard` |
| | 其他 category | `soft` |
| **Lettermint** | `message.hard_bounced` | `hard` |
| | `message.soft_bounced` | `soft` |
| | `message.spam_complaint` | `complaint` |

### 2. BounceActions 配置 [schema.sql:286]

**默认配置**:
```json
{
  "soft":      {"count": 2, "action": "none"},
  "hard":      {"count": 1, "action": "blocklist"},
  "complaint": {"count": 1, "action": "blocklist"}
}
```

**配置结构** [models/settings.go:118]:
```go
BounceActions map[string]struct {
    Count  int    `json:"count"`   // 达到多少次触发 action
    Action string `json:"action"`  // none / blocklist / unsubscribe / delete
}
```

### 3. Source 字段值

每个 provider 硬编码固定的 source 值:

| Provider | Source 值 |
|----------|----------|
| SES | `"ses"` |
| Sendgrid | `"sendgrid"` |
| Postmark | `"postmark"` |
| ForwardEmail | `"forwardemail"` |
| Lettermint | `"lettermint"` |
| 原生 Webhook | 请求中的 `source` 字段 |

---

## SQL 查询逻辑深度分析

### record-bounce CTE 查询 [queries/bounces.sql:1-30]

这是整个 bounce 处理的核心 SQL，使用 PostgreSQL CTE (Common Table Expression) 实现原子操作。

```sql
-- name: record-bounce
WITH sub AS (
    -- 步骤1: 查找订阅者 (优先 UUID，其次 Email)
    SELECT id, status FROM subscribers 
    WHERE CASE WHEN $1 != '' THEN uuid = $1::UUID ELSE email = $2 END
),
camp AS (
    -- 步骤2: 查找 campaign
    SELECT id FROM campaigns WHERE $3 != '' AND uuid = $3::UUID
),
num AS (
    -- 步骤3: 计算该订阅者同类型 bounce 数量 (+1 包含本次)
    SELECT COUNT(*) + 1 AS num 
    FROM bounces 
    WHERE subscriber_id = (SELECT id FROM sub) AND type = $4
),
-- 步骤4: 条件更新 blocklist
block1 AS (
    UPDATE subscribers SET status='blocklisted'
    WHERE $9 = 'blocklist' 
      AND (SELECT num FROM num) >= $8 
      AND id = (SELECT id FROM sub) 
      AND (SELECT status FROM sub) != 'blocklisted'
),
-- 步骤5: 条件更新 unsubscribe
block2 AS (
    UPDATE subscriber_lists SET status='unsubscribed'
    WHERE $9 = 'unsubscribe' 
      AND (SELECT num FROM num) >= $8 
      AND subscriber_id = (SELECT id FROM sub) 
      AND (SELECT status FROM sub) != 'blocklisted'
),
-- 步骤6: 插入 bounce 记录
bounce AS (
    INSERT INTO bounces (subscriber_id, campaign_id, type, source, meta, created_at)
    SELECT (SELECT id FROM sub), (SELECT id FROM camp), $4, $5, $6, $7
    -- ⚠️ 关键条件: 已 blocklist 或 超过阈值则不插入
    WHERE NOT EXISTS (
        SELECT 1 WHERE (SELECT status FROM sub) = 'blocklisted' 
                     OR (SELECT num FROM num) > $8
    )
)
-- 步骤7: 条件删除订阅者
DELETE FROM subscribers
WHERE $9 = 'delete' 
  AND (SELECT num FROM num) >= $8 
  AND id = (SELECT id FROM sub);
```

### 参数映射

| 参数索引 | Go 变量 | 说明 |
|---------|---------|------|
| $1 | `b.SubscriberUUID` | 订阅者 UUID |
| $2 | `b.Email` | 订阅者邮箱 |
| $3 | `b.CampaignUUID` | 活动 UUID |
| $4 | `b.Type` | bounce 类型 (hard/soft/complaint) |
| $5 | `b.Source` | 来源 (ses/sendgrid/...) |
| $6 | `b.Meta` | 原始请求元数据 |
| $7 | `b.CreatedAt` | 创建时间 |
| $8 | `action.Count` | 阈值次数 |
| $9 | `action.Action` | 执行动作 |

### ⚠️ 关键逻辑分析

#### 1. Blocklist 条件 (block1 CTE)

```sql
WHERE $9 = 'blocklist'              -- 配置的 action 是 blocklist
  AND (SELECT num FROM num) >= $8   -- bounce 次数 >= 阈值
  AND id = (SELECT id FROM sub)     -- 找到订阅者
  AND (SELECT status FROM sub) != 'blocklisted'  -- 尚未 blocklist
```

#### 2. Bounce 插入条件 (bounce CTE)

```sql
WHERE NOT EXISTS (
    SELECT 1 
    WHERE (SELECT status FROM sub) = 'blocklisted'  -- 已 blocklist 不记录
       OR (SELECT num FROM num) > $8                -- 超过阈值不记录
)
```

**⚠️ 重要发现**: 
- `(SELECT num FROM num) > $8` 是**大于**，不是**大于等于**
- 意味着: **第 N 次 bounce 会被记录（刚好达到阈值），第 N+1 次及以后不会被记录**

#### 3. Delete 条件 (最后的 DELETE)

```sql
WHERE $9 = 'delete'                 -- 配置的 action 是 delete
  AND (SELECT num FROM num) >= $8   -- bounce 次数 >= 阈值
  AND id = (SELECT id FROM sub)     -- 找到订阅者
```

---

## 生产问题根因分析

### 问题 1: 伪造请求被 Postmark 拒绝但 Sendgrid 路径记录了 soft bounce

**现象**: 同一次伪造请求，Postmark 拒绝但 Sendgrid 记录成功

**可能原因分析**:

#### A. Postmark Basic Auth 配置正确，Sendgrid 公钥配置错误

```go
// Postmark: 配置正确 → 认证失败返回 401
if err := p.authHandler(c); err != nil {
    return nil, err  // echo.HTTPError(401) 直接返回
}

// Sendgrid: 公钥配置错误（比如为空或无效）→ ???
func NewSendgrid(key string) (*Sendgrid, error) {
    sigB, err := base64.StdEncoding.DecodeString(key)
    if err != nil {
        return nil, err  // 初始化失败
    }
    // ...
}

// bounce.Manager 初始化时 Sendgrid 失败只是打日志不中断
if opt.SendgridEnabled {
    sg, err := webhooks.NewSendgrid(opt.SendgridKey)
    if err != nil {
        lo.Printf("error initializing sendgrid webhooks: %v", err)  // ⚠️ 只是打日志
    } else {
        m.Sendgrid = sg
    }
}

// 如果 Sendgrid 初始化失败，m.Sendgrid = nil
// 则 BounceWebhook 中条件不匹配:
case service == "sendgrid" && a.bounce.Sendgrid != nil:
    // 不会进入这里
    // ...
default:
    return echo.NewHTTPError(http.StatusBadRequest, a.i18n.Ts("bounces.unknownService"))
```

**但这不会导致 Sendgrid 路径记录 bounce**。

#### B. Sendgrid 签名验证存在绕过可能？

检查 Sendgrid `verifyNotif`:
```go
func (s *Sendgrid) verifyNotif(sig, timestamp string, b []byte) error {
    sigB, err := base64.StdEncoding.DecodeString(sig)
    if err != nil {
        return err  // 签名解码失败直接报错
    }
    // ...
    if !ecdsa.Verify(s.pubKey, hash, ecdsaSig.R, ecdsaSig.S) {
        return errors.New("invalid signature")  // 签名验证失败报错
    }
    return nil
}
```

签名验证逻辑看起来正确。

#### C. ⚠️ 最可能原因: 多 provider 同时处理同一 bounce

**场景**:
1. 邮件通过多个 provider 发送
2. bounce 通知同时发送到所有配置的 webhook endpoint
3. Postmark: 因为不是 Postmark 发送的邮件，签名/认证失败 → 拒绝
4. Sendgrid: 是 Sendgrid 发送的邮件，签名正确 → 处理并记录

**Sendgrid bounce 分类逻辑**:
```go
typ := models.BounceTypeHard
if n.BounceClassification == "technical" || n.BounceClassification == "content" {
    typ = models.BounceTypeSoft  // ⚠️ technical 和 content 是 soft bounce
}
```

Sendgrid 的 `bounce_classification` 字段:
- `hard`: 硬 bounc
- `technical`: 技术问题（如 DNS 失败、超时）→ soft
- `content`: 内容相关问题 → soft

如果伪造的请求触发了 `technical` 或 `content` 分类，就会被记录为 **soft bounce**。

---

### 问题 2: Hard bounce 应该 blocklist 但只插入了 bounces 记录

**现象**: Hard bounce 被记录，但订阅者没有被 blocklist

**可能原因分析**:

#### A. 配置的 action 不是 blocklist

```go
action, ok := c.consts.BounceActions[b.Type]
// action.Action 可能是 "none" 而不是 "blocklist"
```

检查默认配置 [schema.sql:286]:
```sql
('bounce.actions', '{"soft": {"count": 2, "action": "none"}, "hard": {"count": 1, "action": "blocklist"}, ...}')
```

默认 hard bounce action 是 blocklist，但可能被管理员修改为 `none`。

#### B. ⚠️ 订阅者已经是 blocklisted 状态

SQL 中的关键条件 [queries/bounces.sql:16]:
```sql
UPDATE subscribers SET status='blocklisted'
WHERE ... AND (SELECT status FROM sub) != 'blocklisted'
-- ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
-- 如果已经是 blocklisted，则不更新
```

同时，bounce 插入条件 [queries/bounces.sql:26]:
```sql
INSERT INTO bounces ...
WHERE NOT EXISTS (
    SELECT 1 WHERE (SELECT status FROM sub) = 'blocklisted'
                 OR ...
)
```

**但这样 bounce 也不会被插入**。

#### C. ⚠️ bounce 计数超过阈值

SQL 中的条件 [queries/bounces.sql:26]:
```sql
WHERE NOT EXISTS (
    SELECT 1 WHERE ... OR (SELECT num FROM num) > $8
)
```

`num` 的计算 [queries/bounces.sql:11]:
```sql
SELECT COUNT(*) + 1 AS num FROM bounces 
WHERE subscriber_id = (SELECT id FROM sub) AND type = $4
```

**场景复现**:
1. 配置: `hard: {count: 1, action: "blocklist"}`
2. 第 1 次 hard bounce:
   - `num = 0 + 1 = 1`
   - `num >= count` (1 >= 1) → **执行 blocklist**
   - `num > count` (1 > 1) → false → **插入 bounce 记录**
   - ✅ 预期行为: blocklist + 记录 bounce
3. 第 2 次 hard bounce:
   - `num = 1 + 1 = 2`
   - `num >= count` (2 >= 1) → 执行 blocklist（但已经是 blocklisted，无实际变化）
   - `num > count` (2 > 1) → true → **不插入 bounce 记录**
   - ❌ 只执行 blocklist（无变化），不记录 bounce

这与"只插入了 bounces 记录"的现象不符。

#### D. ⚠️ Action 参数传递错误

检查 `core.RecordBounce` [internal/core/bounces.go:66]:
```go
_, err := c.q.RecordBounce.Exec(
    b.SubscriberUUID, b.Email, b.CampaignUUID,
    b.Type, b.Source, b.Meta, b.CreatedAt,
    action.Count,    // $8
    action.Action)   // $9
```

参数顺序正确。

#### E. ⚠️ 最可能原因: Subquery 评估顺序和 CTE 执行顺序

PostgreSQL CTE 的执行顺序:
1. CTE 是按照定义顺序执行的
2. 但 `WHERE NOT EXISTS` 中的子查询可能在不同时间点评估

**关键问题**: `block1` (UPDATE subscribers) **发生在** `bounce` (INSERT) **之前**

```sql
WITH ...,
block1 AS (
    -- 先执行: UPDATE subscribers SET status='blocklisted' ...
),
block2 AS (...),
bounce AS (
    -- 后执行: INSERT ...
    -- 此时 (SELECT status FROM sub) 可能已经是 'blocklisted'
    WHERE NOT EXISTS (
        SELECT 1 WHERE (SELECT status FROM sub) = 'blocklisted'
    )
)
```

**⚠️ 但 `sub` CTE 是在查询开始时执行的，结果被缓存**

PostgreSQL CTE 是**物化**的（在 PG 12+ 中，除非使用 `NOT MATERIALIZED`），所以 `sub` 的结果在查询开始时就确定了，不会因为 `block1` 的 UPDATE 而改变。

#### F. ⚠️ 最可能原因: 事务隔离级别或并发问题

**场景**:
1. 两个相同的 hard bounce 请求几乎同时到达
2. 请求 A 和请求 B 同时读取 `sub` CTE，得到 status='enabled'
3. 请求 A 和请求 B 同时计算 `num = 0 + 1 = 1`
4. 请求 A 执行 block1 UPDATE: status='blocklisted' ✓
5. 请求 B 执行 block1 UPDATE: status 已经是 'blocklisted'，无变化
6. 请求 A 插入 bounce: `num > count` 为 false ✓
7. 请求 B 插入 bounce: `num > count` 为 false ✓

**结果**: 两条 bounce 记录，但只有一次 blocklist 操作。

这也不对。

#### G. ⚠️ 根本原因: Action 配置为 "none" 但 count=1

如果配置是:
```json
{
  "hard": {"count": 1, "action": "none"}
  //                    ^^^^^^^ 不是 "blocklist"
}
```

那么:
1. `$9 = 'none'`
2. block1 UPDATE 条件: `$9 = 'blocklist'` → false → **不执行 blocklist**
3. bounce INSERT 条件: `(SELECT num FROM num) > $8` → 1 > 1 → false → **插入 bounce 记录**

**完全匹配现象**: 只插入 bounces 记录，不 blocklist。

---

## 修复建议

### 针对问题 1: Sendgrid 记录伪造请求的 soft bounce

**建议 1: 加强 Sendgrid 事件类型过滤**

当前 Sendgrid 只过滤 `Event != "bounce"`，但 bounce 分类可能过于宽泛。建议添加配置项控制哪些 classification 视为 bounce。

**建议 2: 添加 IP 白名单**

Sendgrid 提供 Webhook IP 列表，可在应用层或网络层添加 IP 白名单。

**建议 3: 添加请求速率限制**

对 /webhooks/service/* 端点添加速率限制，防止滥用。

### 针对问题 2: Hard bounce 不 blocklist

**建议 1: 验证 BounceActions 配置**

确保配置正确:
```sql
SELECT value FROM settings WHERE key = 'bounce.actions';
-- 应返回: {"soft": {"count": 2, "action": "none"}, "hard": {"count": 1, "action": "blocklist"}, "complaint" : {"count": 1, "action": "blocklist"}}
```

**建议 2: 添加日志埋点**

在 `core.RecordBounce` 中添加详细日志:
```go
func (c *Core) RecordBounce(b models.Bounce) error {
    action, ok := c.consts.BounceActions[b.Type]
    c.log.Printf("recording bounce: type=%s, source=%s, email=%s, action_count=%d, action=%s",
        b.Type, b.Source, b.Email, action.Count, action.Action)
    // ...
}
```

**建议 3: 监控告警**

添加监控:
- Hard bounce 数量 vs Blocklisted 订阅者数量
- Bounce 记录插入但未触发 action 的告警

### 通用建议

**建议 1: 统一认证失败处理**

所有 provider 认证失败都应该返回明确的 HTTP 状态码，便于调试:
```go
// 改为统一处理: 所有错误都检查是否是 echo.HTTPError
if err != nil {
    a.log.Printf("error processing %s notification: %v", service, err)
    if httpErr, ok := err.(*echo.HTTPError); ok {
        return httpErr
    }
    return echo.NewHTTPError(http.StatusBadRequest, ...)
}
```

**建议 2: 添加 webhook 请求审计日志**

记录所有 webhook 请求的:
- Provider
- IP 地址
- User-Agent
- 认证结果
- 处理结果

**建议 3: 定期同步订阅者状态**

添加定时任务，重新计算所有订阅者的 bounce 计数，确保状态一致性。
