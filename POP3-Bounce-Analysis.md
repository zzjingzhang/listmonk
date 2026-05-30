# Listmonk POP3 Bounce 扫描机制分析与不可丢信改造方案

## 1. 扫描任务全链路工作流程

### 1.1 初始化：initBounceManager

入口位于 [initBounceManager](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L817-L875)，在 `main.go` 中当 `bounce.enabled=true` 时调用：

```
bounce = initBounceManager(core.RecordBounce, queries.RecordBounce, lo, ko)
go bounce.Run()
```

核心逻辑：
1. 构建 `bounce.Opt`，填入 webhook / SES / SendGrid / Postmark / ForwardEmail / Lettermint 配置
2. 遍历 `bounce.mailboxes` 配置切片，取**第一个** `enabled=true` 的邮箱（只支持一个），填充 `opt.MailboxType` 和 `opt.Mailbox`
3. 调用 `bounce.New(opt, queries, lo)` 创建 `bounce.Manager`

### 1.2 bounce.Manager 结构与队列

定义于 [bounce.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/bounce.go#L47-L58)：

| 字段 | 类型 | 说明 |
|------|------|------|
| `queue` | `chan models.Bounce` (容量 1000) | 所有 bounce 事件的汇聚通道 |
| `mailbox` | `Mailbox` 接口 | POP3 扫描器实例 |
| `SES/Sendgrid/Postmark/...` | webhook 处理器 | 各平台 webhook |
| `queries.RecordQuery` | `*sqlx.Stmt` | 预编译的 INSERT 语句 |

`Run()` 方法（[L118-L132](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/bounce.go#L118-L132)）做了两件事：
- 启动 `go m.runMailboxScanner()` 异步扫描
- 主循环 `for b := range m.queue` 消费队列，调用 `m.opt.RecordBounceCB(b)`

`Record()` 方法（[L147-L150](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/bounce.go#L147-L150)）将 bounce 推入 `queue` 通道。

### 1.3 POP3 邮箱扫描流程

定义于 [pop.go Scan()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L79-L208)：

```
连接 POP3 → 认证 → Stat() 获取消息数 → 逐条 RetrRaw(id) → 解析/分类 → 推入 channel → 批量 Dele(id)
```

**关键步骤详解：**

1. **连接与认证**（L80-L91）：使用 `pop3.Client` 建立 TCP 连接，支持 TLS
2. **获取消息数**（L94-L106）：`c.Stat()` 返回消息总数，若 `count > limit` 则截断
3. **逐条下载**（L109-L198）：
   - `c.RetrRaw(id)` 获取原始字节
   - `message.Read(b)` 解析邮件
   - 对 multipart 消息，遍历所有 part，取**最后一个 part** 的 header
   - 重新 `c.RetrRaw(id)` 获取原始字节（用于正则 fallback）
   - 查找 7 个 header + Received header
   - `classifyBounce()` 分类
   - 通过 `select/case ch <- / default` 非阻塞推入 channel
4. **批量删除**（L200-L205）：**无论前面步骤是否成功，对所有 `1..count` 执行 `c.Dele(id)`**

### 1.4 Header 正则 Fallback 机制

定义于 [pop.go L41-L49](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L41-L49)：

```go
headerLookups = []bounceHeaders{
    {X-Listmonk-Campaign,   regexp},
    {X-Listmonk-Subscriber, regexp},
    {Date,        regexp},
    {From,        regexp},
    {Subject,     regexp},
    {Message-Id,  regexp},
    {Delivered-To,regexp},
}
```

查找策略（[L146-L156](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L146-L156)）：
1. 先从解析后的 `h.Header.Get(headerName)` 获取
2. 若为空，则在**原始字节**上用正则 `FindAllSubmatch` 搜索，取**最后一个匹配**

**重要缺陷：没有 Return-Path header 的查找！** Return-Path 是 bounce 邮件中携带原始收件人信息的关键 header。

### 1.5 SMTP 状态码 / 关键词分类

定义于 [classifyBounce()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L214-L239)：

**分类优先级：**
1. **SMTP 状态码** `reSMTPStatus` → 匹配 `5.x.x` 返回 hard，`4.x.x` 返回 soft
2. **硬 bounce 关键词** `reHardBounce` → 匹配 NXDOMAIN/user unknown/... 等返回 hard
3. **默认 soft** → 以上都不匹配，返回 `BounceTypeSoft, "default"`

正则定义（[L53-L59](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L53-L59)）：

```go
reSMTPStatus = `(?m)(?i)^(?:Status:\s*)?(?:\d{3}\s+)?([45]\.\d+\.\d+)`

reHardBounce = `(?i)(NXDOMAIN|user unknown|address not found|mailbox not found|
    address.*reject|does not exist|invalid recipient|no such user|
    recipient.*invalid|undeliverable|permanent.*failure|permanent.*error|
    bad.*address|unknown.*user|account.*disabled|address.*disabled)`
```

### 1.6 core.RecordBounce 写入逻辑

定义于 [RecordBounce()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/bounces.go#L60-L87)：

```go
func (c *Core) RecordBounce(b models.Bounce) error {
    action, ok := c.consts.BounceActions[b.Type]  // 查找该类型对应的动作
    if !ok {
        return error  // 无效类型直接报错
    }
    _, err := c.q.RecordBounce.Exec(
        b.SubscriberUUID, b.Email, b.CampaignUUID,
        b.Type, b.Source, b.Meta, b.CreatedAt,
        action.Count, action.Action)
    // 如果 subscriber 不存在，吞掉错误返回 nil
    if pqErr.Column == "subscriber_id" { return nil }
    return err
}
```

对应 SQL（[bounces.sql record-bounce](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/bounces.sql#L1-L30)）：
- 用 `SubscriberUUID` 或 `Email` 查找 subscriber
- 用 `CampaignUUID` 查找 campaign
- 若 bounce 次数 ≥ 阈值，执行 `blocklist` / `unsubscribe` / `delete` 动作
- **已 blocklisted 的 subscriber 不再记录新 bounce**

---

## 2. 问题根因分析

### 2.1 问题一：真实 hard bounce 被识别为 soft

| 根因 | 详细说明 |
|------|---------|
| **默认分类为 soft** | `classifyBounce()` 末尾 `return BounceTypeSoft, "default"`，任何未匹配的 bounce 一律归为 soft |
| **SMTP 状态码正则局限** | `reSMTPStatus` 仅匹配 `Status: 5.x.x` 或 `ddd 5.x.x` 格式。许多 MTA 的 bounce 报告格式各异（如 `550 5.1.1` 不带 `Status:` header、或 `status=5.1.1`、或 `#5.1.1`），导致漏匹配 |
| **关键词列表不全** | `reHardBounce` 仅覆盖 16 种英文模式。大量合法 hard bounce 不在列表中，例如：`552 mailbox full`（某些 MTA 标记为 permanent）、`relay denied`、`spam rejected`、`domain not found`、`550 5.7.1` 等 |
| **4.x.x 可能是 hard** | 某些 `4.x.x` 状态码在特定上下文中实际是永久性错误（如 `4.4.7` message expired after long retry），但一律被判为 soft |
| **multipart 丢失诊断信息** | 取最后一个 part 的 header，但 SMTP 诊断码通常在第一个 text/plain part 的 body 中，multipart 处理后可能丢失 |

### 2.2 问题二：邮件下载后删除但未写入 bounces 表

| 根因 | 代码位置 | 详细说明 |
|------|---------|---------|
| **channel 满时静默丢弃** | [pop.go L187-L197](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L187-L197) | `select { case ch <- bounce: default: }` — 当 queue channel (容量 1000) 满时，bounce 被静默丢弃，无日志、无重试 |
| **删除与处理解耦** | [pop.go L200-L205](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L200-L205) | `Dele(id)` 循环覆盖 `1..count`，不管该消息是否成功解析和入队 |
| **解析失败仍被删除** | [pop.go L118-L122](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L118-L122) | `message.Read(b)` 失败后 `continue`，但消息仍然在后续 delete 循环中被删除 |
| **SubscriberUUID 缺失** | [pop.go L191](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L191) | 若 MTA 剥离了 `X-Listmonk-Subscriber` header，`SubscriberUUID` 为空，SQL 中 `WHERE uuid = $1::UUID` 对空字符串会失败（非法 UUID），导致 DB 写入失败 |
| **CampaignUUID 缺失** | [pop.go L190](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L190) | 同理，`X-Listmonk-Campaign` 被剥离时 `CampaignUUID` 为空，虽然 SQL 用了 `$3 != ''` 保护，但 campaign_id 为 NULL 时丧失了与活动的关联 |
| **Email 字段缺失** | [pop.go L188-L195](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L188-L195) | Bounce 结构体中没有从邮件内容提取收件人 Email。`Return-Path` 不在 headerLookups 中，bounce 邮件的 `Delivered-To` / `Return-Path` / `To` 中的原始收件人地址未被用于填充 `Email` 字段 |
| **RecordBounce 错误被吞** | [bounce.go L128-L129](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/bounce.go#L128-L129) | `if err := m.opt.RecordBounceCB(b); err != nil { continue }` — DB 写入失败后仅跳过，不重试，消息已从 POP 服务器删除 |

### 2.3 数据流完整追踪图

```
POP3 服务器
  │
  ├─ RetrRaw(1..N)     下载消息
  ├─ message.Read()     解析 ─── 失败 → continue（消息仍被删除！）
  ├─ headerLookup       查找7个header + 正则fallback
  │   └─ 缺失 Return-Path / To（原始收件人地址丢失！）
  ├─ classifyBounce()   分类 ─── 默认soft（hard被误分类！）
  ├─ select/case ch     入队 ─── channel满 → 静默丢弃（消息仍被删除！）
  │
  ├─ Dele(1..N)         无条件删除所有消息
  │
  ▼
queue channel (cap=1000)
  │
  ▼
Manager.Run() 消费
  │
  ├─ RecordBounceCB → core.RecordBounce()
  │   ├─ BounceActions[type] 不存在 → 返回 error → continue（不重试）
  │   ├─ SubscriberUUID 为空/非法 → SQL 失败 → continue
  │   ├─ Subscriber 不存在 → 吞掉错误 → return nil（写入失败但无记录）
  │   └─ 成功 → INSERT bounces + 触发 action
  │
  ▼
bounces 表
```

---

## 3. 关键影响因素分析

### 3.1 Scan Interval 的影响

`ScanInterval` 定义于 [opt.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/opt.go#L28)，在 `runMailboxScanner` 中用作 `time.Sleep` 的间隔：

```go
func (m *Manager) runMailboxScanner() {
    for {
        m.mailbox.Scan(1000, m.queue)
        time.Sleep(m.opt.Mailbox.ScanInterval)
    }
}
```

| 影响 | 说明 |
|------|------|
| **间隔过长** | bounce 邮件在 POP3 邮箱中积压，延迟处理。如果邮箱有容量限制，可能弹回新邮件 |
| **间隔过短** | 频繁 POP3 连接增加服务器负载；若上一次 Scan 还在处理中（无并发控制），不会触发新的 Scan（因为 `Run()` 中是串行的），但频繁的无效连接消耗资源 |
| **无并发保护** | 如果 `Scan()` 执行时间 > `ScanInterval`，不会有并发问题（因为是串行循环），但也没有跳过机制 |

### 3.2 Delete Downloaded Messages 的影响

当前行为：**下载完成后无条件删除所有消息**（[pop.go L200-L205](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/bounce/mailbox/pop.go#L200-L205)）

```go
for id := 1; id <= count; id++ {
    if err := c.Dele(id); err != nil {
        return err
    }
}
```

**风险分析：**
- POP3 协议中 `DELE` 命令在 `QUIT` 时才真正执行（标记删除），但如果 `Dele()` 本身返回错误，`Scan()` 直接 return，之前成功的 `Dele()` 标记仍然生效（因为 `defer c.Quit()` 会执行）
- **没有"先记录成功再删除"的语义**：即使 bounce 入队失败或 DB 写入失败，消息已经从服务器删除
- **这是丢信的最根本原因**：删除与写入没有事务性保证

### 3.3 Return-Path / Message-Id 依赖

**Return-Path 的角色：**

在发送侧（[email.go L189-L194](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/messenger/email/email.go#L189-L194)），`Return-Path` header 被用作 SMTP envelope sender：

```go
if sender := em.Headers.Get(hdrReturnPath); sender != "" {
    em.Sender = sender
    em.Headers.Del(hdrReturnPath)  // 从 header 中删除！
}
```

配置中 `return_path` 字段（[settings.go L145](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/settings.go#L145)）可以设置 bounce 接收地址。

**关键问题：**
1. **发送时 Return-Path 被从 header 中删除**，只在 SMTP envelope 层面使用
2. **bounce 邮件到达时**，MTA 会自动在邮件顶部添加 `Return-Path` header（值为 envelope sender），但 POP3 扫描器**没有解析 Return-Path**
3. 如果 `X-Listmonk-Subscriber` header 被 MTA 剥离（某些 MTA 会清除 `X-*` header），扫描器只能依赖正则 fallback
4. **Message-Id** 虽然被提取并存入 meta，但**没有被用来反查原始邮件关联**。如果能在发送时记录 `Message-Id → (subscriber_uuid, campaign_uuid)` 的映射，即使自定义 header 被剥离也能通过 Message-Id 恢复关联

### 3.4 分类优先级的影响

当前优先级：**SMTP 状态码 > 关键词 > 默认 soft**

| 场景 | 实际应为 | 当前结果 | 原因 |
|------|---------|---------|------|
| `Status: 5.1.1` | hard | ✅ hard | SMTP 状态码正确匹配 |
| `550 5.1.1 User unknown` | hard | ✅ hard | SMTP 状态码匹配 |
| `#5.1.1` (Postfix DSN) | hard | ❌ soft | 正则不匹配 `#5.x.x` 格式 |
| `550 User not found` (无 Status header) | hard | ❌ soft | 无 SMTP 状态码，"User not found" 不在关键词列表 |
| `554 spam content rejected` | hard | ❌ soft | 无匹配关键词 |
| `4.4.7 Message expired` | hard | ❌ soft | 4.x.x 一律为 soft |
| `Status: 4.2.2` (mailbox full, temporary) | soft | ✅ soft | 正确 |
| 未知 NDR 格式 | soft? | soft | 默认 soft，但可能是 hard |

**默认 soft 的危险性：**
- soft bounce 不触发 blocklist/delete 动作，仅计数
- 真实的 hard bounce 被误判为 soft 后，系统会继续向无效地址发信
- 这不仅浪费资源，还会损害发送信誉

---

## 4. 不可丢信改造方案

### 4.1 设计原则

1. **先写入再删除（Write-Before-Delete）**：只有确认 bounce 成功持久化后才删除 POP3 消息
2. **零静默丢弃**：所有丢弃/失败必须有日志和可追踪的指标
3. **多路径 subscriber 关联**：不只依赖 X-Listmonk-* header，增加 Return-Path / Message-Id / To 地址等多重回退
4. **分类保守偏向 hard**：无法确定时宁可误判 hard（阻止继续发送），而非误判 soft（继续浪费发送）

### 4.2 改造项一：Write-Before-Delete 机制

**目标：** 消息只有成功写入 bounces 表后才从 POP3 服务器删除

**方案：** 将 `Scan()` 拆分为"下载"和"确认删除"两阶段

```go
// 新增 ScanResult 结构
type ScanResult struct {
    ID     int            // POP3 消息 ID
    Bounce models.Bounce  // 解析后的 bounce
    Raw    []byte         // 原始字节（用于重试/审计）
    Err    error          // 解析错误
}

// 修改 Mailbox 接口
type Mailbox interface {
    Scan(limit int) ([]ScanResult, error)
    Delete(ids []int) error
}
```

**pop.go Scan() 改造：**

```go
func (p *POP) Scan(limit int) ([]ScanResult, error) {
    // ... 连接、认证、Stat 同原逻辑 ...

    var results []ScanResult
    for id := 1; id <= count; id++ {
        result := ScanResult{ID: id}

        b, err := c.RetrRaw(id)
        if err != nil {
            result.Err = err
            results = append(results, result)
            continue
        }

        // 保存原始字节用于审计
        result.Raw = make([]byte, len(b.Bytes()))
        copy(result.Raw, b.Bytes())

        m, err := message.Read(b)
        if err != nil {
            result.Err = err
            results = append(results, result)
            continue
        }

        // 解析 header + 分类（同原逻辑）
        result.Bounce, result.Err = p.parseBounce(m, result.Raw)
        results = append(results, result)
    }

    // 不在此处删除！返回结果让调用方决定
    // 注意：不调用 c.Quit()，保持连接以便后续 Delete
    // 需要将连接对象返回或使用连接池
    return results, nil
}
```

**Manager 层改造：**

```go
func (m *Manager) runMailboxScanner() {
    for {
        results, conn, err := m.mailbox.Scan(1000)
        if err != nil {
            m.log.Printf("error scanning bounce mailbox: %v", err)
            time.Sleep(m.opt.Mailbox.ScanInterval)
            continue
        }

        var processedIDs []int
        var failedIDs []int

        for _, r := range results {
            if r.Err != nil {
                m.log.Printf("error parsing bounce message %d: %v", r.ID, r.Err)
                // 解析失败的不删除，留待人工处理或下次重试
                continue
            }

            // 同步写入 DB（而非入 channel），确保写入成功
            if err := m.opt.RecordBounceCB(r.Bounce); err != nil {
                m.log.Printf("error recording bounce for message %d: %v", r.ID, err)
                failedIDs = append(failedIDs, r.ID)
                // 写入失败的也不删除
                continue
            }

            processedIDs = append(processedIDs, r.ID)
        }

        // 只删除成功写入的
        if len(processedIDs) > 0 {
            if err := conn.Delete(processedIDs); err != nil {
                m.log.Printf("error deleting processed messages: %v", err)
            }
        }

        if len(failedIDs) > 0 {
            m.log.Printf("WARNING: %d messages failed processing and were NOT deleted", len(failedIDs))
        }

        conn.Quit()
        time.Sleep(m.opt.Mailbox.ScanInterval)
    }
}
```

### 4.3 改造项二：消除 channel 静默丢弃

**目标：** 确保 bounce 入队不丢失

**方案 A（推荐）：** 如上改造，直接同步调用 `RecordBounceCB`，绕过 channel

**方案 B：** 如果需要保留 channel 解耦架构，改用阻塞发送 + 超时：

```go
select {
case ch <- bounce:
    // 成功入队
default:
    // channel 满，记录到本地文件作为持久化缓冲
    m.log.Printf("CRITICAL: bounce queue full, writing to fallback store")
    if err := m.fallbackStore.Write(bounce); err != nil {
        m.log.Printf("CRITICAL: fallback store also failed: %v", err)
    }
}
```

Fallback store 可以是：
- 本地 JSON 文件（简单可靠）
- SQLite 嵌入式数据库
- Redis / 文件系统队列

### 4.4 改造项三：增强 Header 提取与 Subscriber 关联

**3a. 增加 Return-Path 解析**

```go
headerLookups = []bounceHeaders{
    // 现有 7 个 ...
    {models.EmailHeaderReturnPath, regexp.MustCompile(`(?m)(?:^Return-Path:\s+?)(.*)`)},
}
```

从 `Return-Path` 中提取原始收件人地址，用于填充 `Bounce.Email` 字段。

**3b. 增加 To / Original-To / Final-Recipient 解析**

```go
{models.EmailHeaderTo,              regexp.MustCompile(`(?m)(?:^To:\s+?)(.*)`)},
{"Original-Recipient",              regexp.MustCompile(`(?m)(?:^Original-Recipient:\s+?)(.*)`)},
{"Final-Recipient",                 regexp.MustCompile(`(?m)(?:^Final-Recipient:\s+?)(.*)`)},
}
```

**3c. 基于 Message-Id 的反查机制**

在发送邮件时，将 `Message-Id` 与 `(subscriber_uuid, campaign_uuid)` 的映射持久化到数据库：

```sql
CREATE TABLE message_tracking (
    message_id     TEXT PRIMARY KEY,
    subscriber_uuid UUID NOT NULL,
    campaign_uuid   UUID NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW()
);
```

当 bounce 邮件的 `X-Listmonk-*` header 缺失时，用 `Message-Id` 反查：

```go
// 在 parseBounce 中，如果 SubscriberUUID 为空但 MessageID 不为空
if bounce.SubscriberUUID == "" && bounce.MessageID != "" {
    subUUID, campUUID := lookupByMessageID(bounce.MessageID)
    bounce.SubscriberUUID = subUUID
    bounce.CampaignUUID = campUUID
}
```

**3d. 基于 Email 地址的回退查找**

如果 header 全部缺失但能从 `Return-Path` / `To` / `Final-Recipient` 中提取到收件人邮箱：

```go
if bounce.SubscriberUUID == "" && bounce.Email != "" {
    subUUID := lookupSubscriberByEmail(bounce.Email)
    bounce.SubscriberUUID = subUUID
}
```

### 4.5 改造项四：增强 Bounce 分类准确性

**4a. 扩展 SMTP 状态码正则**

```go
reSMTPStatus = regexp.MustCompile(
    `(?m)(?i)(?:Status:\s*)?(?:\d{3}\s+)?(?:#)?([45])\.(?:\d{1,3})\.(?:\d{1,3})`)
```

增加对 `#5.x.x`、`5.x.x` 无 Status 行首缀、`diagnostic-code: smtp; 550 5.x.x` 等格式的支持。

**4b. 扩展 hard bounce 关键词**

```go
reHardBounce = regexp.MustCompile(`(?i)(` +
    // 原有
    `NXDOMAIN|user unknown|address not found|mailbox not found|` +
    `address.*reject|does not exist|invalid recipient|no such user|` +
    `recipient.*invalid|undeliverable|permanent.*failure|permanent.*error|` +
    `bad.*address|unknown.*user|account.*disabled|address.*disabled|` +
    // 新增
    `user not found|no such address|no mailbox here|mailbox unavailable|` +
    `relay.*denied|access denied|rejected.*address|delivery failed|` +
    `host not found|domain not found|domain.*invalid|` +
    `spam.*reject|content.*reject|policy.*reject|` +
    `550\s|551\s|552\s|553\s|554\s|` +
    `not.*deliverable|delivery.*permanent|` +
    `recipient.*reject|sender.*reject)`)
```

**4c. 增加显式 soft bounce 关键词**

```go
reSoftBounce = regexp.MustCompile(`(?i)(` +
    `mailbox full|quota exceeded|over quota|temporarily rejected|` +
    `try again later|temporary.*failure|deferred|greylist|` +
    `421\s|450\s|451\s|452\s)`)
```

**4d. 修改分类逻辑为三段式**

```go
func classifyBounce(b []byte) (string, string) {
    // 1. SMTP 状态码优先
    if matches := reSMTPStatus.FindAllSubmatch(b, -1); matches != nil {
        for _, m := range matches {
            status := m[1]
            if status[0] == '5' {
                return BounceTypeHard, fmt.Sprintf("smtp_status=%s", status)
            }
            if status[0] == '4' {
                return BounceTypeSoft, fmt.Sprintf("smtp_status=%s", status)
            }
        }
    }

    // 2. 显式 hard bounce 关键词
    if match := reHardBounce.FindSubmatch(b); match != nil {
        return BounceTypeHard, fmt.Sprintf("hard_keyword=%s", match[1])
    }

    // 3. 显式 soft bounce 关键词
    if match := reSoftBounce.FindSubmatch(b); match != nil {
        return BounceTypeSoft, fmt.Sprintf("soft_keyword=%s", match[1])
    }

    // 4. 默认 hard（保守策略）
    return BounceTypeHard, "default_unknown"
}
```

**为何默认改 hard？**
- 默认 soft 意味着系统继续向可能无效的地址发信
- 默认 hard 意味着可能误阻有效地址，但可通过管理员手动解除 blocklist
- 误判 hard 的代价（需人工恢复）远小于误判 soft 的代价（持续发送到无效地址、损害域名信誉）

### 4.6 改造项五：持久化审计日志

**目标：** 即使 DB 写入失败，也能追溯丢失的 bounce

```go
type BounceAuditLog struct {
    ID         int       `db:"id"`
    POP3MsgID  int       `db:"pop3_msg_id"`
    RawMessage []byte    `db:"raw_message"`
    Status     string    `db:"status"`      // "downloaded" / "parsed" / "recorded" / "failed"
    Error      string    `db:"error"`
    CreatedAt  time.Time `db:"created_at"`
}
```

```sql
CREATE TABLE bounce_audit_log (
    id           SERIAL PRIMARY KEY,
    pop3_msg_id  INTEGER,
    raw_message  BYTEA,
    status       TEXT NOT NULL DEFAULT 'downloaded',
    error        TEXT,
    created_at   TIMESTAMP DEFAULT NOW()
);

-- 定期清理已成功处理的审计记录
-- DELETE FROM bounce_audit_log WHERE status = 'recorded' AND created_at < NOW() - INTERVAL '7 days'
```

在 `Scan()` 每个关键节点写入审计记录：

```go
// 下载后
auditLog.Write(msgID, raw, "downloaded", "")

// 解析后
auditLog.Write(msgID, nil, "parsed", "")

// DB 写入成功后
auditLog.Write(msgID, nil, "recorded", "")

// 失败时
auditLog.Write(msgID, raw, "failed", err.Error())
```

### 4.7 改造项六：POP3 连接容错与重试

**问题：** 当前 `Scan()` 如果中途连接断开，已下载但未删除的消息下次不会再处理（因为 POP3 的消息 ID 在会话间可能变化）

**方案：**

```go
func (p *POP) Scan(limit int) ([]ScanResult, *POPConn, error) {
    c, err := p.client.NewConn()
    if err != nil {
        return nil, nil, err
    }
    conn := &POPConn{client: c, opt: p.opt}

    // ... 下载逻辑 ...

    // 返回连接对象，让调用方决定何时删除和关闭
    return results, conn, nil
}

type POPConn struct {
    client      *pop3.Conn
    opt         Opt
    deleteIDs   []int
}

func (c *POPConn) MarkDelete(ids []int) {
    c.deleteIDs = append(c.deleteIDs, ids...)
}

func (c *POPConn) CommitDelete() error {
    for _, id := range c.deleteIDs {
        if err := c.client.Dele(id); err != nil {
            return err
        }
    }
    return c.client.Quit()
}

func (c *POPConn) Rollback() error {
    // 不调用 Dele，直接 Quit，POP3 会自动回滚未提交的删除标记
    return c.client.Quit()
}
```

---

## 5. 改造优先级与实施路线

| 优先级 | 改造项 | 影响范围 | 实施难度 | 预期效果 |
|--------|--------|---------|---------|---------|
| **P0** | 4.2 Write-Before-Delete | pop.go, bounce.go | 中 | **彻底解决丢信问题** |
| **P0** | 4.3 消除 channel 静默丢弃 | bounce.go | 低 | 消除队列满时的静默丢信 |
| **P1** | 4.4 增强 Header 提取 | pop.go, 新增 DB 表 | 中 | 解决 header 被剥离后的关联丢失 |
| **P1** | 4.5 增强分类准确性 | pop.go | 低 | 减少 hard→soft 误判 |
| **P2** | 4.6 持久化审计日志 | 新增 DB 表 + 逻辑 | 中 | 提供可追溯性 |
| **P2** | 4.7 POP3 连接容错 | pop.go | 中 | 提升扫描可靠性 |

### 最小可行改造（P0 only）

如果时间有限，至少完成：

1. **修改 `Scan()` 为下载+返回结果，不自动删除**
2. **修改 `runMailboxScanner()` 为同步写入 DB，成功后才删除**
3. **将 `select/default` 改为阻塞发送或同步调用**
4. **默认分类改为 hard**

这四项改动可以用约 150 行代码修改完成，立即消除丢信风险。

---

## 6. 总结

Listmonk 当前的 POP3 bounce 扫描存在两个严重问题：

1. **硬弹回误判为软弹回**：默认 soft + 有限的正则匹配，导致大量真实 hard bounce 被当作 soft 处理，系统持续向无效地址发信
2. **下载即删除，无写入保证**：POP3 消息在下载阶段就被标记删除，但后续的 channel 入队、DB 写入都可能失败，且没有回滚机制。这是"邮件被下载后删除但未写入 bounces 表"的根本原因

核心改造思路是**将"先删后写"反转为"先写后删"**，同时增强 header 提取的多路径回退和分类准确性，最终实现零丢信的 bounce 处理管道。
