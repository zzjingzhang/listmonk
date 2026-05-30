# SMTP 邮件发送全链路推演：Envelope / MIME / Headers / Attachments / Pool 选择

> 升级后部分 SMTP 服务商拒收邮件，日志显示 Return-Path、Message-Id、Bcc/Cc、From 域名池和附件大小处理异常。
> 本文档从 `initSMTPMessengers` → `manager.attachMedia` → `message.render` → `email.Push` → `campaign header` 模板推演一封邮件的完整生命周期。

---

## 1. SMTP Pool 初始化：`initSMTPMessengers`

**入口**：[init.go#L665-L712](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L665-L712)

### 1.1 执行流程

```
initSMTPMessengers()
  ├── 遍历 ko.Slices("smtp") 配置项
  │     ├── 跳过 enabled=false 的服务器
  │     ├── Unmarshal 到 email.Server 结构体
  │     ├── 若 s.Name != ""，创建独立 messenger：email.New(s.Name, s)
  │     └── 收集所有 server 到 servers 列表
  └── 创建默认 "email" messenger：email.New("email", servers...)
```

### 1.2 `email.New()` 内部构建 Pool 映射

**源码**：[email.go#L64-L124](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/messenger/email/email.go#L64-L124)

```
New(name, servers...)
  ├── 为每个 server：
  │     ├── 选择 Auth（cram / plain / login / none）
  │     ├── 配置 TLS（SSLTLS / SSLSTARTTLS / none）
  │     ├── 创建 smtppool.Pool（连接池）
  │     └── 填充 pools 映射：
  │           ├── pools[""]     ← 全部 server（兜底 round-robin）
  │           └── pools[from_addr] ← 按 from_addresses 中的地址/域名索引
  └── 返回 Emailer{pools: map[string][]*Server}
```

### 1.3 Pool 路由机制：From 域名池

| 映射键 | 含义 | 使用场景 |
|--------|------|---------|
| `""` (空字符串) | 所有 SMTP server 的合集 | 兜底 round-robin |
| `user@domain.com` | 完整 email 地址匹配 | campaign.FromEmail 精确路由 |
| `domain.com` | 域名匹配 | From 域名级别路由 |

**路由逻辑**（[email.go#L241-L256](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/messenger/email/email.go#L241-L256)）：

```
getPool(from)
  ├── ParseEmailAddress(from) → 得到小写裸地址 user@domain.com
  ├── 精确匹配 pools["user@domain.com"]
  └── 若未命中，取 @ 后的域名匹配 pools["domain.com"]
```

### 1.4 字段来源清单

| 字段 | 来源 | 配置路径 |
|------|------|---------|
| `Server.Host` | settings DB | `smtp[].host` |
| `Server.Port` | settings DB | `smtp[].port` |
| `Server.Username` | settings DB | `smtp[].username` |
| `Server.Password` | settings DB | `smtp[].password` |
| `Server.AuthProtocol` | settings DB | `smtp[].auth_protocol` |
| `Server.TLSType` | settings DB | `smtp[].tls_type` |
| `Server.TLSSkipVerify` | settings DB | `smtp[].tls_skip_verify` |
| `Server.EmailHeaders` | settings DB | `smtp[].email_headers` |
| `Server.FromAddresses` | settings DB | `smtp[].from_addresses` |
| `Server.MaxConns` | settings DB | `smtp[].max_conns` |
| `Server.MaxMsgRetries` | settings DB | `smtp[].max_msg_retries` |
| `Server.IdleTimeout` | settings DB | `smtp[].idle_timeout` |
| `Server.WaitTimeout` | settings DB | `smtp[].wait_timeout` |
| `Server.HelloHostname` | settings DB | `smtp[].hello_hostname` |

---

## 2. 附件加载：`manager.attachMedia`

**源码**：[manager.go#L663-L679](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L663-L679)

### 2.1 调用时机

```
newPipe(campaign)                    ← pipe.go#L27
  ├── c.CompileTemplate(funcMap)     ← 编译模板
  ├── m.attachMedia(c)               ← 加载附件 ★
  └── 创建 pipe 对象

PushCampaignMessage(msg)             ← manager.go#L213
  └── m.attachMedia(msg.Campaign)    ← 再次调用（幂等）
```

### 2.2 执行逻辑

```go
func (m *Manager) attachMedia(c *models.Campaign) error {
    if len(c.Attachments) > 0 {
        return nil                    // 已加载，跳过（幂等保护）
    }
    for _, mid := range []int64(c.MediaIDs) {
        a, err := m.store.GetAttachment(int(mid))
        // GetAttachment 内部：
        //   1. core.GetMedia(id) → 获取 meta 信息
        //   2. media.GetBlob(url) → 读取文件字节
        //   3. MakeAttachmentHeader(filename, "base64", contentType)
        c.Attachments = append(c.Attachments, a)
    }
}
```

### 2.3 `GetAttachment` 详情

**源码**：[manager_store.go#L89-L105](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/manager_store.go#L89-L105)

```
GetAttachment(mediaID)
  ├── core.GetMedia(mediaID) → 获取 media.Meta（filename, content_type, url）
  ├── media.GetBlob(url) → 读取文件内容字节
  └── 组装 Attachment{
        Name:    m.Filename,
        Content: b,                   // 原始字节
        Header:  MakeAttachmentHeader(filename, "base64", contentType)
      }
```

### 2.4 `MakeAttachmentHeader` 生成 MIME 头

**源码**：[manager.go#L684-L697](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L684-L697)

```go
func MakeAttachmentHeader(filename, encoding, contentType string) textproto.MIMEHeader {
    // encoding 默认 "base64"
    // contentType 默认 "application/octet-stream"
    h.Set("Content-Disposition", "attachment; filename="+filename)
    h.Set("Content-Type", fmt.Sprintf("%s; name=\"%s\"", contentType, filename))
    h.Set("Content-Transfer-Encoding", encoding)
    return h
}
```

### 2.5 附件大小处理异常点

| 问题 | 说明 |
|------|------|
| **无大小限制检查** | `attachMedia` 从媒体存储加载完整字节到内存，无上限校验 |
| **base64 膨胀** | Content-Transfer-Encoding 为 base64，实际传输体积 ≈ 原始 × 1.37 |
| **多附件累加** | 所有 media_id 的附件全部加载到 `c.Attachments` 切片，无总量限制 |
| **smtppool 无分块** | smtppool.Send() 将整个附件内容作为 `[]byte` 一次性写入 MIME，不流式分块 |

---

## 3. 消息渲染：`message.render`

**源码**：[message.go#L33-L88](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/message.go#L33-L88)

### 3.1 调用链

```
pipe.newMessage(subscriber)               ← pipe.go#L172
  └── manager.NewCampaignMessage(camp, sub)  ← message.go#L13
        ├── 初始化 CampaignMessage{
        │     from:     camp.FromEmail,
        │     to:       sub.Email,
        │     subject:  camp.Subject,
        │     unsubURL: fmt.Sprintf(cfg.UnsubURL, camp.UUID, sub.UUID)
        │   }
        └── msg.render()                      ← ★ 渲染
```

### 3.2 render() 内部流程

```
render()
  ├── 1. 渲染 Subject 模板（如有 {{ }} 表达式）
  │     └── camp.SubjectTpl.ExecuteTemplate → m.subject
  │
  ├── 2. 渲染 Body 模板
  │     └── camp.Tpl.ExecuteTemplate(BaseTpl) → m.body
  │
  ├── 3. 渲染 AltBody（纯文本版本）
  │     └── camp.AltBodyTpl.ExecuteTemplate → m.altBody
  │
  └── 4. 渲染 Header 模板
        ├── 若 camp.HeaderTpls == nil → 直接使用 camp.Headers（无需渲染）
        └── 若 camp.HeaderTpls != nil → 逐个渲染含模板表达式的 header 值
              for i, set := range camp.Headers
                for hdr, val := range set
                  tpl = camp.HeaderTpls[i][hdr]
                  if tpl != nil → tpl.ExecuteTemplate → m.headers[i][hdr]
                  else → m.headers[i][hdr] = val（原始值）
```

### 3.3 Campaign Headers 数据结构

**DB schema**：`headers JSONB NOT NULL DEFAULT '[]'`

存储格式为 `[]map[string]string`，即：
```json
[
  {"Return-Path": "bounce@example.com"},
  {"X-Custom": "{{ .Subscriber.Email }}"}
]
```

### 3.4 字段来源清单

| 字段 | 来源 | 说明 |
|------|------|------|
| `m.from` | `campaign.FromEmail` | DB campaigns 表 |
| `m.to` | `subscriber.Email` | DB subscribers 表 |
| `m.subject` | `campaign.Subject` → 可能经模板渲染 | DB campaigns 表 |
| `m.body` | `campaign.Tpl` 渲染结果 | 模板 + campaign.body |
| `m.altBody` | `campaign.AltBodyTpl` 渲染结果 | 模板 + campaign.altbody |
| `m.headers` | `campaign.Headers` → 可能经模板渲染 | DB campaigns.headers JSONB |
| `m.unsubURL` | `cfg.UnsubURL` + campaign.UUID + subscriber.UUID | settings 配置 |

---

## 4. SMTP 发送：`email.Push`

**源码**：[email.go#L132-L223](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/messenger/email/email.go#L132-L223)

### 4.1 完整流程

```
Push(m models.Message)
  │
  ├── 1. 选择 SMTP Pool ★
  │     ├── pool = e.pools[""]（默认全量池）
  │     ├── if len(pools) > 1 → getPool(m.From)
  │     │     ├── 精确匹配 from 地址 → pools["user@domain.com"]
  │     │     └── 域名匹配 → pools["domain.com"]
  │     └── srv = pool[rand.Intn(len(pool))]（随机 round-robin）
  │
  ├── 2. 转换附件
  │     └── models.Attachment → smtppool.Attachment（copy Content）
  │
  ├── 3. 构建 smtppool.Email
  │     ├── From:    m.From
  │     ├── To:      m.To（始终 []string{subscriber.Email}）
  │     ├── Subject: m.Subject
  │     ├── Attachments: files
  │     └── Headers: textproto.MIMEHeader{}
  │
  ├── 4. 合并 SMTP 级 headers ★
  │     └── for k, v := range srv.EmailHeaders → em.Headers.Set(k, v)
  │
  ├── 5. 合并消息级 headers ★
  │     └── for k, v := range m.Headers → em.Headers.Set(k, v[0])
  │
  ├── 6. Message-Id 生成 ★
  │     ├── if em.Headers.Get("Message-Id") == ""
  │     │     ├── 解析 m.From 提取 @ 后域名
  │     │     ├── GenerateRandomString(24) → 随机部分
  │     │     └── Set("Message-Id", "<{random}@{domain}>")
  │     └── 若已有 Message-Id（来自 headers）→ 保留不覆盖
  │
  ├── 7. Return-Path 处理 ★
  │     ├── sender = em.Headers.Get("Return-Path")
  │     ├── if sender != ""
  │     │     ├── em.Sender = sender          ← 设置 SMTP envelope MAIL FROM
  │     │     └── em.Headers.Del("Return-Path") ← 从 MIME 头删除
  │     └── 若无 Return-Path → em.Sender 为空
  │           → smtppool 使用 From 作为 MAIL FROM
  │
  ├── 8. Bcc 处理 ★★
  │     ├── bcc = em.Headers.Get("Bcc")
  │     ├── if bcc != ""
  │     │     ├── 解析逗号分隔 → em.Bcc = [...]
  │     │     └── em.Headers.Del("Bcc")      ← 从 MIME 头删除 ★
  │     └── 无 Bcc → em.Bcc 为空
  │
  ├── 9. Cc 处理 ★
  │     ├── cc = em.Headers.Get("Cc")
  │     ├── if cc != ""
  │     │     ├── 解析逗号分隔 → em.Cc = [...]
  │     │     └── em.Headers.Del("Cc")       ← 从 MIME 头删除 ★
  │     └── 无 Cc → em.Cc 为空
  │
  ├── 10. 正文设置
  │     ├── contentType == "plain" → em.Text = body
  │     └── default → em.HTML = body, em.Text = altBody（如有）
  │
  └── 11. srv.pool.Send(em)
        ├── smtppool 内部：
        │     ├── SMTP MAIL FROM: <em.Sender>（若非空）或 <em.From>
        │     ├── SMTP RCPT TO: <em.To[0]>, <em.Cc[...]>, <em.Bcc[...]>
        │     ├── 构造 MIME 信件：
        │     │     ├── From: em.From
        │     │     ├── To: em.To[0]
        │     │     ├── Cc: em.Cc（如有，但已在 headers 中删除，smtppool 会重建？）
        │     │     ├── Subject: em.Subject
        │     │     ├── em.Headers 中所有字段
        │     │     ├── multipart/mixed + multipart/alternative
        │     │     └── 附件 part（带 Header）
        │     └── DATA 发送
```

### 4.2 关键字段覆盖优先级

Headers 的 Set 语义是"最后写入者胜出"：

```
优先级从低到高：
1. srv.EmailHeaders    （SMTP 服务器级配置）   ← 最先 Set
2. m.Headers           （消息级/Campaign 级）   ← 后 Set，覆盖同名
3. 代码逻辑覆盖（Message-Id / Return-Path / Bcc / Cc） ← 特殊处理
```

---

## 5. Campaign Header 模板推演

### 5.1 headers 编译流程

**源码**：[campaigns.go#L212-L239](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L212-L239)

```
CompileTemplate()
  └── 检测 Headers 中是否包含 {{ }} 模板表达式
        ├── 若包含 → 创建 HeaderTpls[i][hdr] = txttpl.Template
        └── 若不含 → HeaderTpls = nil（直接使用原始值）
```

### 5.2 可在 Campaign Headers 中设置的关键字段

| Header 名 | 用途 | 是否在 Push 中被特殊处理 | 风险点 |
|-----------|------|-------------------------|--------|
| `Return-Path` | 退信地址 | ✅ → 移至 envelope Sender | 域名必须与 SMTP 服务商授权域名匹配 |
| `Message-Id` | 消息唯一标识 | ✅ → 若为空则自动生成 | 自定义时域名部分必须与 From 域名一致 |
| `Bcc` | 密送 | ✅ → 移至 envelope Bcc，删除 MIME 头 | ⚠️ 见下方 Bcc 泄露分析 |
| `Cc` | 抄送 | ✅ → 移至 envelope Cc，删除 MIME 头 | ⚠️ 见下方 Cc 泄露分析 |
| `From` | 发件人 | ❌ 不处理，由 em.From 字段控制 | ⚠️ Header 中的 From 与 em.From 可能不一致 |
| `Reply-To` | 回复地址 | ❌ 直接写入 MIME 头 | 安全 |
| `X-Custom-*` | 自定义头 | ❌ 直接写入 MIME 头 | 安全 |

### 5.3 From 域名池机制

```
campaign.FromEmail = "marketing@example.com"
     ↓
Push() 中：
  em.From = m.From  → "marketing@example.com"
  getPool(em.From) → pools["marketing@example.com"] 或 pools["example.com"]
     ↓
选择匹配的 SMTP server 发送
```

**关键问题**：如果 campaign.FromEmail 的域名与实际 SMTP server 授权发送的域名不一致（如 From 用 `@company.com` 但 SMTP 只授权 `@mail.company.com`），SMTP 服务商会拒收（DMARC/S PF 校验失败）。

---

## 6. 一封邮件的完整数据流图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SETTINGS (DB + config.toml)                         │
│  ┌──────────────────────┐  ┌────────────────────────────────────────────┐  │
│  │ app.from_email       │  │ smtp[].host / port / auth / tls           │  │
│  │ app.root_url         │  │ smtp[].email_headers                      │  │
│  │ app.notify_emails    │  │ smtp[].from_addresses                     │  │
│  │ privacy.unsub_header │  │ smtp[].max_conns / max_msg_retries        │  │
│  └──────────┬───────────┘  └──────────────────┬─────────────────────────┘  │
└─────────────┼─────────────────────────────────┼───────────────────────────┘
              │                                 │
              ▼                                 ▼
┌──────────────────────────┐     ┌──────────────────────────────────────────┐
│     initSMTPMessengers   │     │             initCampaignManager          │
│  ┌────────────────────┐  │     │  ┌────────────────────────────────────┐  │
│  │ Emailer.pools      │  │     │  │ Manager.cfg.FromEmail             │  │
│  │ Emailer.pools[""]  │  │     │  │ Manager.cfg.UnsubURL              │  │
│  │ Emailer.pools[addr]│  │     │  │ Manager.cfg.UnsubHeader           │  │
│  └────────────────────┘  │     │  └────────────────────────────────────┘  │
└──────────────────────────┘     └──────────────────────────────────────────┘
                                            │
              ┌─────────────────────────────┘
              ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     CAMPAIGN (DB campaigns 表)                          │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────────────┐│
│  │ from_email      │  │ subject      │  │ headers (JSONB)            ││
│  │ body / altbody  │  │ content_type │  │   [{"Return-Path": ...}]  ││
│  │ messenger       │  │ template_id  │  │   [{"Bcc": "..."}]        ││
│  │ media_ids       │  │ tags         │  │   [{"Cc": "..."}]         ││
│  └────────┬────────┘  └──────┬───────┘  └─────────────┬──────────────┘│
└───────────┼──────────────────┼────────────────────────┼───────────────┘
            │                  │                        │
            ▼                  ▼                        ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     SUBSCRIBER (DB subscribers 表)                      │
│  ┌─────────────────┐  ┌──────────────┐  ┌────────────────────────────┐│
│  │ email           │  │ uuid         │  │ attribs                    ││
│  └────────┬────────┘  └──────┬───────┘  └────────────────────────────┘│
└───────────┼──────────────────┼                                            │
            │                  │                                            │
            ▼                  ▼                                            │
┌─────────────────────────────────────────────────────────────────────────┐
│  newPipe() → CompileTemplate() → attachMedia()                          │
│                                                                          │
│  NewCampaignMessage(camp, sub) → render()                               │
│    m.from     = camp.FromEmail                                          │
│    m.to       = sub.Email                                               │
│    m.subject  = rendered(camp.Subject)                                  │
│    m.body     = rendered(camp.Tpl)                                      │
│    m.headers  = rendered(camp.Headers)  ← 含模板表达式的 header 被渲染  │
│    m.unsubURL = cfg.UnsubURL + camp.UUID + sub.UUID                     │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     worker() → 组装 models.Message                       │
│                                                                          │
│  out.From        = msg.from                                             │
│  out.To          = []string{msg.to}                                     │
│  out.Subject     = msg.subject                                          │
│  out.ContentType = msg.Campaign.ContentType                             │
│  out.Body        = msg.body                                             │
│  out.AltBody     = msg.altBody                                          │
│  out.Attachments = msg.Campaign.Attachments                             │
│  out.Headers     = textproto.MIMEHeader{                               │
│                      "X-Listmonk-Campaign":    camp.UUID                │
│                      "X-Listmonk-Subscriber":  sub.UUID                 │
│                      "List-Unsubscribe-Post":  "List-Unsubscribe=One-Click" │
│                      "List-Unsubscribe":       "<unsubURL>"            │
│                      ...msg.headers (campaign headers)...               │
│                    }                                                     │
│                                                                          │
│  messenger.Push(out)                                                    │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                     email.Push() — 关键处理                              │
│                                                                          │
│  ┌─ SMTP Pool 选择 ─────────────────────────────────────────────────┐  │
│  │ pool = getPool(m.From)                                           │  │
│  │ srv  = pool[rand.Intn]                                           │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─ Headers 合并（优先级：低→高）───────────────────────────────────┐  │
│  │ 1. srv.EmailHeaders → em.Headers.Set(k, v)                       │  │
│  │ 2. m.Headers        → em.Headers.Set(k, v[0])  ← 可覆盖上方     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─ 特殊字段处理 ──────────────────────────────────────────────────┐   │
│  │ Message-Id:  若为空 → 生成 <random@from-domain>                  │  │
│  │ Return-Path: → em.Sender (envelope), 删除 MIME 头               │  │
│  │ Bcc:         → em.Bcc (envelope RCPT TO), 删除 MIME 头 ★        │  │
│  │ Cc:          → em.Cc (envelope RCPT TO), 删除 MIME 头 ★         │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  ┌─ 最终 SMTP 信封 ────────────────────────────────────────────────┐  │
│  │ MAIL FROM: <em.Sender || em.From>                                │  │
│  │ RCPT TO:   <em.To[0]>, <em.Cc[...]>, <em.Bcc[...]>             │  │
│  │ DATA:      MIME 信件（不含 Bcc/Cc/Return-Path 头）              │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 7. 字段来源矩阵

| 字段 | Settings (DB/Config) | Campaign (DB) | Manager (运行时) | Messenger (发送时覆盖) |
|------|:--------------------:|:-------------:|:----------------:|:---------------------:|
| **Envelope MAIL FROM** | `smtp[].from_addresses` 间接 | `campaign.from_email` | — | `Return-Path` header → `em.Sender` |
| **Envelope RCPT TO** | — | — | `subscriber.Email` → `To` | `Bcc`/`Cc` header → `em.Bcc`/`em.Cc` |
| **MIME From** | `app.from_email`（兜底） | `campaign.from_email` | — | — |
| **MIME To** | — | — | `subscriber.Email` | — |
| **MIME Subject** | — | `campaign.subject` | 模板渲染 | — |
| **MIME Message-Id** | — | `campaign.headers` 中可设置 | — | 若为空则自动生成 |
| **MIME Return-Path** | `smtp[].email_headers` 可设 | `campaign.headers` 可设 | — | 移至 envelope，**删除 MIME 头** |
| **MIME Bcc** | — | `campaign.headers` 可设 | — | 移至 envelope，**删除 MIME 头** |
| **MIME Cc** | — | `campaign.headers` 可设 | — | 移至 envelope，**删除 MIME 头** |
| **MIME List-Unsubscribe** | `privacy.unsubscribe_header` | — | 构造 URL | — |
| **MIME X-Listmonk-Campaign** | — | — | `camp.UUID` | — |
| **MIME X-Listmonk-Subscriber** | — | — | `sub.UUID` | — |
| **MIME 自定义头** | `smtp[].email_headers` | `campaign.headers` | 模板渲染 | — |
| **MIME Content-Type** | — | `campaign.content_type` | — | — |
| **MIME 正文** | — | `campaign.body` | 模板渲染 | — |
| **MIME AltBody** | — | `campaign.altbody` | 模板渲染 | — |
| **Attachments** | `upload.*` 配置 | `campaign.media_ids` | `attachMedia` 加载 | `copy(Content)` |
| **Attachment Headers** | — | — | `MakeAttachmentHeader()` | — |
| **SMTP Pool 选择** | `smtp[].from_addresses` | `campaign.from_email` | — | `getPool(m.From)` |

---

## 8. Bcc 泄露验证

### 8.1 防泄露机制

**源码**：[email.go#L196-L202](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/messenger/email/email.go#L196-L202)

```go
// 如果 Bcc header 被设置，将其移至 Envelope
if bcc := em.Headers.Get(hdrBcc); bcc != "" {
    for _, part := range strings.Split(bcc, ",") {
        em.Bcc = append(em.Bcc, strings.TrimSpace(part))
    }
    em.Headers.Del(hdrBcc)  // ★ 从 MIME 头删除
}
```

### 8.2 防泄露保证

| 检查项 | 结果 | 说明 |
|--------|------|------|
| Bcc 是否出现在 MIME 头中 | ✅ 已删除 | `em.Headers.Del(hdrBcc)` |
| Bcc 收件人是否收到邮件 | ✅ 能收到 | `em.Bcc` 加入 RCPT TO |
| To 收件人是否看到 Bcc 地址 | ✅ 看不到 | MIME 头中无 Bcc 字段 |
| Cc 是否出现在 MIME 头中 | ✅ 已删除 | `em.Headers.Del(hdrCc)` |
| Cc 收件人是否收到邮件 | ✅ 能收到 | `em.Cc` 加入 RCPT TO |
| To 收件人是否看到 Cc 地址 | ⚠️ 取决于 smtppool | smtppool 需重建 Cc 头 |

### 8.3 Cc 泄露风险分析

Cc 的处理与 Bcc 不同：
- Bcc：删除 MIME 头 + 加入 envelope RCPT TO → **收件人不可见** ✅
- Cc：删除 MIME 头 + 加入 envelope RCPT TO → **但 smtppool 可能重建 Cc 头**

如果 smtppool 库在构造 MIME 信件时，会将 `em.Cc` 字段重新写入 `Cc:` MIME 头（标准 SMTP 行为），则：
- To 收件人 **能看到** Cc 地址（这是 Cc 的正常行为）
- 但如果本意是"隐藏抄送"却用了 Cc header，则会泄露

**结论**：若要隐藏收件人，必须使用 `Bcc` header 而非 `Cc` header。

### 8.4 验证方法

```bash
# 方法 1：发送测试邮件，检查原始 MIME 源
# 在邮件客户端查看"原始邮件"/"View Source"
# 搜索 "Bcc:" 头 → 不应存在

# 方法 2：在代码中添加断言验证
# 在 email.Push() 的 srv.pool.Send(em) 之前：
if em.Headers.Get("Bcc") != "" {
    panic("Bcc header not stripped!")
}

# 方法 3：开启 SMTP 调试日志
# 在 config.toml 中设置日志级别，观察 SMTP DATA 内容
```

---

## 9. 升级后拒收问题根因分析

### 9.1 Return-Path 异常

| 场景 | 问题 | 根因 |
|------|------|------|
| 自定义 Return-Path | 域名与 SPF/DKIM 不匹配 | Return-Path 域名必须与 SMTP 服务商授权的发送域名一致 |
| 无 Return-Path | MAIL FROM 使用 From 地址 | 部分服务商要求 MAIL FROM ≠ From（如 SES） |
| `smtp[].email_headers` 设置 Return-Path | 与 campaign.headers 中的 Return-Path 冲突 | campaign headers 覆盖 SMTP headers（Set 语义） |

**修复建议**：
- 确保 Return-Path 域名在 SMTP 服务商处已授权
- 在 `smtp[].email_headers` 中统一设置 Return-Path，避免 campaign 级覆盖

### 9.2 Message-Id 异常

| 场景 | 问题 | 根因 |
|------|------|------|
| 自动生成 | 域名部分取自 From 地址的 @ 后部分 | 若 From 格式异常（如 `Name <email>`），解析可能失败回退 `localhost` |
| 自定义 Message-Id | 域名与 From 域名不一致 | 部分 SMTP 服务商校验 Message-Id 域名与 From 域名一致性 |

**根因代码**：[email.go#L179-L187](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/messenger/email/email.go#L179-L187)

```go
d := "localhost"  // ← 回退值！
if a, err := mail.ParseAddress(m.From); err == nil {
    d = a.Address[strings.LastIndex(a.Address, "@")+1:]
}
```

**修复建议**：
- 确保 campaign.FromEmail 格式为标准 RFC 5322 地址
- 在 campaign headers 中显式设置 Message-Id

### 9.3 From 域名池异常

| 场景 | 问题 | 根因 |
|------|------|------|
| From 地址不在任何 from_addresses 中 | 回退到全量池 round-robin | 可能选到不匹配的 SMTP server |
| 多个 SMTP 共享域名 | SPF 记录不包含所有 server IP | DMARC 校验失败 |

**修复建议**：
- 每个 SMTP server 的 `from_addresses` 必须覆盖所有可能的 campaign.FromEmail
- 确保 DNS SPF 记录包含所有 SMTP server 的发送 IP

### 9.4 附件大小异常

| 场景 | 问题 | 根因 |
|------|------|------|
| 大附件 | base64 编码后超 SMTP 大小限制 | 无预检查，发送时才失败 |
| 多附件累加 | 总大小超限 | 无总量限制 |

**修复建议**：
- 在 `attachMedia` 中添加大小检查
- 在 campaign 创建时校验 media 总大小

---

## 10. 完整字段流转表

```
字段                     配置层              DB 层                运行时层              发送时层
─────────────────────────────────────────────────────────────────────────────────────────────────
MAIL FROM (envelope)    smtp 配置        →  campaign.from_email  →  msg.from         →  em.Sender (from Return-Path)
                                                                     或 em.From (无 Return-Path 时)

RCPT TO (envelope)      —                →  subscriber.email      →  msg.to           →  em.To[0] + em.Cc + em.Bcc

MIME From               app.from_email   →  campaign.from_email   →  msg.from         →  em.From (不变)

MIME To                 —                →  subscriber.email      →  msg.to           →  em.To[0]

MIME Subject            —                →  campaign.subject      →  msg.subject      →  em.Subject (不变)
                                                                    (可能经模板渲染)

MIME Message-Id         —                →  campaign.headers      →  out.Headers      →  若为空自动生成
                                                                                          <random@from-domain>

MIME Return-Path        smtp.email_headers→  campaign.headers     →  out.Headers      →  移至 em.Sender
                                                                                         删除 MIME 头

MIME Bcc                —                →  campaign.headers      →  out.Headers      →  移至 em.Bcc
                                                                                         删除 MIME 头

MIME Cc                 —                →  campaign.headers      →  out.Headers      →  移至 em.Cc
                                                                                         删除 MIME 头

MIME List-Unsubscribe   privacy.unsub    →  —                     →  构造 URL          →  em.Headers
                        _header

MIME X-Listmonk-*       —                →  camp.UUID/sub.UUID    →  out.Headers      →  em.Headers

MIME 自定义头           smtp.email_headers→  campaign.headers     →  out.Headers      →  em.Headers
                                                                                       (campaign 覆盖 smtp)

Attachments             upload 配置      →  campaign.media_ids    →  attachMedia 加载 →  copy(Content)
                                                                                       + AttachmentHeader
```

---

## 11. 验证清单

### 11.1 Return-Path 验证

- [ ] 确认 `smtp[].email_headers` 中 Return-Path 域名与 SMTP 服务商授权域名一致
- [ ] 确认 campaign.headers 中 Return-Path 不会覆盖 SMTP 级设置
- [ ] 确认 MAIL FROM（envelope sender）通过 SPF 验证
- [ ] 确认 Return-Path 域名有有效的 DKIM 签名

### 11.2 Message-Id 验证

- [ ] 确认 campaign.FromEmail 格式正确（`user@domain.com` 或 `Name <user@domain.com>`）
- [ ] 确认自动生成的 Message-Id 域名部分与 From 域名一致
- [ ] 确认自定义 Message-Id 符合 RFC 5322 格式（`<unique@domain>`）

### 11.3 Bcc 泄露验证

- [ ] 发送含 Bcc header 的测试邮件，检查接收方原始邮件源中无 `Bcc:` 字段
- [ ] 确认 Bcc 收件人能正常收到邮件
- [ ] 确认 To 收件人无法看到 Bcc 地址
- [ ] 确认 `em.Headers.Get("Bcc")` 在 `Send()` 调用前为空字符串

### 11.4 From 域名池验证

- [ ] 确认每个 SMTP server 的 `from_addresses` 覆盖所有可能的 campaign.FromEmail
- [ ] 确认 `getPool()` 能正确路由到匹配的 SMTP server
- [ ] 确认 fallback 到全量池时，所选 SMTP server 仍能通过 SPF/DKIM 验证

### 11.5 附件验证

- [ ] 确认单个附件大小不超过 SMTP 服务商限制
- [ ] 确认所有附件 base64 编码后总大小不超过 SMTP 限制
- [ ] 确认 `Content-Type` 正确设置（非一律 `application/octet-stream`）
- [ ] 确认 `Content-Transfer-Encoding` 为 `base64`
