# Transactional API 消息处理流程深度分析

## 1. 概述

本文档深度分析 listmonk 中 transactional (tx) 消息从接收、解析、模板选择、渲染、附件校验到最终投递的完整处理流程。重点分析 `SendTxMessage`、`validateTxMessage`、`initTxTemplates`、`models.TxMessage.Render`、`Template.Compile` 和 `manager.PushMessage` 六个核心函数，并比较 `default`/`fallback`/`external` 三种收件人模式的行为差异，最后指出可能造成模板缓存陈旧的操作。

---

## 2. 整体处理流程

```
外部请求
    ↓
SendTxMessage (API入口)
    ├─ 解析请求 (JSON / multipart/form-data)
    ├─ 解析附件 (multipart时)
    ↓
validateTxMessage (参数校验)
    ├─ 校验 subscriber_mode 合法性
    ├─ 校验收件人格式
    └─ 设置默认值 (from_email, messenger)
    ↓
manager.GetTpl (获取缓存模板)
    ↓
按 subscriber_mode 处理收件人
    ├─ default: 查DB，不存在则报错
    ├─ fallback: 查DB，不存在则创建临时subscriber
    └─ external: 不查DB，直接创建临时subscriber
    ↓
TxMessage.Render (渲染消息)
    ├─ 渲染 body (使用预编译模板)
    ├─ 渲染 altbody (按需动态编译)
    └─ 渲染 subject (优先使用请求中的，否则使用模板中的)
    ↓
manager.PushMessage (投递消息)
    └─ 推入消息队列 msgQ
        └─ worker 异步调用 messenger.Push 发送
```

---

## 3. 核心函数详细分析

### 3.1 SendTxMessage

**位置**: [tx.go#L17-L176](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L17-L176)

#### 功能：事务消息API的入口处理函数。

#### 处理流程：

1.  **请求解析** ([tx.go#L21-L63](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L21-L63))：
    - 支持两种请求格式：
      - `multipart/form-data`：用于带附件的请求，从 `form.Value["data"]` 解析JSON，从 `form.File["file"]` 读取附件
      - `application/json`：直接使用 `c.Bind(&m)` 绑定

2.  **附件处理** ([tx.go#L40-L59](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L40-L59))：
    - 遍历所有上传的文件
    - 读取文件内容到内存
    - 构造 `models.Attachment`，设置文件名、MIME头和内容
    - **注意**：此处没有附件大小限制检查，完全依赖Go HTTP服务器和echo框架的默认限制

3.  **参数校验**：调用 `validateTxMessage` 校验消息字段

4.  **获取模板** ([tx.go#L73-L77](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L73-L77))：
    - 从manager缓存中获取指定 `template_id` 对应的模板
    - 模板不存在返回400错误

5.  **收件人处理循环** ([tx.go#L88-L169](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L88-L169))：
    - 遍历所有收件人，逐个处理
    - 根据 `subscriber_mode` 采用不同策略获取或创建subscriber
    - 调用 `Render` 渲染消息
    - 构造最终 `models.Message`
    - 调用 `PushMessage` 投递

6.  **错误返回** ([tx.go#L171-L173](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L171-L173))：
    - 所有收件人处理完成后，若有 `notFound` 列表非空，返回400错误
    - **注意**：此处存在部分成功部分失败的不一致性问题：部分收件人可能已成功发送，但因后续收件人渲染失败或投递失败，前面已发送的消息无法撤回

---

### 3.2 validateTxMessage

**位置**: [tx.go#L179-L244](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L179-L244)

#### 功能：校验tx消息字段的合法性并标准化。

#### 校验规则：

1.  **字段互斥检查** ([tx.go#L180-L195](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L180-L195))：
    - `subscriber_email` 和 `subscriber_emails` 不能同时存在
    - `subscriber_id` 和 `subscriber_ids` 不能同时存在
    - 兼容旧字段：自动将 `subscriber_email` 转为 `subscriber_emails`，`subscriber_id` 转为 `subscriber_ids`

2.  **subscriber_mode 校验** ([tx.go#L197-L222](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L197-L222))：
    - 默认值：`default`
    - `default` 模式：需要 `subscriber_emails` 或 `subscriber_ids`，但不能同时有
    - `fallback`/`external` 模式：只能使用 `subscriber_emails`，不允许 `subscriber_ids`

3.  **邮箱格式校验** ([tx.go#L224-L232](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L224-L232))：
    - 调用 `a.importer.SanitizeEmail` 标准化邮箱格式

4.  **默认值设置**：
    - `from_email` 默认为配置中的 `a.cfg.FromEmail`
    - `messenger` 默认为 `email`，并校验是否已注册

---

### 3.3 initTxTemplates

**位置**: [init.go#L625-L639](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L625-L639)

#### 功能：系统启动时初始化并缓存所有transactional模板。

#### 处理流程：

1.  从数据库加载所有类型为 `TemplateTypeTx` 的模板 ([init.go#L626](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L626))
2.  遍历每个模板，调用 `tpl.Compile` 编译模板 ([init.go#L633](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L633))
3.  调用 `m.CacheTpl` 将编译后的模板缓存到manager内存中 ([init.go#L637](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L637))

**重要**：此函数仅在系统启动时调用一次，后续模板变更需要通过API更新才能刷新缓存。

---

### 3.4 models.TxMessage.Render

**位置**: [messages.go#L73-L133](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L73-L133)

#### 功能：渲染事务消息的body、altbody和subject。

#### 渲染数据结构：

```go
data := struct {
    Subscriber Subscriber
    Tx         *TxMessage
}{sub, m}
```

模板中可通过 `{{ .Subscriber }}` 访问订阅者信息，`{{ .Tx.Data }}` 访问请求传入的 `data` 对象。

#### 渲染流程：

1.  **Body渲染** ([messages.go#L79-L86](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L79-L86))：
    - 使用模板中预编译的 `tpl.Tpl`（`*template.Template`）
    - 调用 `ExecuteTemplate` 执行渲染
    - 结果存入 `m.Body`

2.  **AltBody渲染** ([messages.go#L88-L99](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L88-L99))：
    - 仅当 `m.AltBody` 非空且包含模板表达式 `{{ ... }}` 时才渲染
    - 动态编译 `m.AltBody` 字符串为模板
    - 执行渲染，结果覆盖 `m.AltBody`
    - **注意**：每次渲染都会重新编译，无缓存

3.  **Subject渲染** ([messages.go#L101-L130](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L101-L130))：
    - **优先级**：请求中的 `subject` > 模板中的 `subject`
    - 若请求指定了 `subject`：
      - 包含模板表达式：动态编译并渲染
      - 不包含模板表达式：直接使用
    - 若请求未指定 `subject`：
      - 使用模板中预编译的 `tpl.SubjectTpl` 渲染
      - 若模板subject不含模板表达式：直接使用 `tpl.Subject` 字符串

**关键问题**：当请求中指定的 `subject` 包含模板表达式但语法错误时，会在第一个收件人处立即返回错误，导致后续收件人无法处理，但之前已成功渲染的收件人可能已发送。

---

### 3.5 Template.Compile

**位置**: [templates.go#L39-L58](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/templates.go#L39-L58)

#### 功能：编译模板的body和subject（仅tx模板）并缓存编译结果。

#### 编译流程：

1.  **Body编译** ([templates.go#L40-L44](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/templates.go#L40-L44))：
    - 使用 `html/template` 编译 `t.Body`
    - 注入模板函数 `f`
    - 编译结果缓存到 `t.Tpl`

2.  **Subject编译** ([templates.go#L46-L55](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/templates.go#L46-L55))：
    - 调用 `hasTplExpr` 检查 `t.Subject` 是否包含 `{{ ... }}` 模板表达式
    - 包含则使用 `text/template` 编译subject
    - 编译结果缓存到 `t.SubjectTpl`
    - 不包含则 `t.SubjectTpl` 为 `nil`，渲染时直接使用字符串

**模板表达式检测** ([campaigns.go#L244-L248](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L244-L248))：
```go
func hasTplExpr(s string) bool {
    _, after, ok := strings.Cut(s, "{{")
    return ok && strings.Contains(after, "}}")
}
```

---

### 3.6 manager.PushMessage

**位置**: [manager.go#L197-L209](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L197-L209)

#### 功能：将非活动邮件推入消息队列，由worker异步发送。

#### 处理流程：

1.  创建3秒超时的ticker ([manager.go#L198](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L198))
2.  尝试将消息推入 `m.msgQ` 通道
3.  3秒内推入成功则返回 `nil`
4.  超时则返回错误 "message push timed out"

**队列配置** ([manager.go#L176](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L176))：
```go
msgQ: make(chan models.Message, cfg.Concurrency*cfg.MessageRate*2)
```
队列大小为 `并发数 * 每秒发送速率 * 2`。

**Worker处理** ([manager.go#L546-L556](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L546-L556))：
- worker从 `msgQ` 接收消息
- 直接调用 `m.messengers[msg.Messenger].Push(msg)` 发送
- 发送失败仅打日志，无重试机制

---

## 4. default / fallback / external 模式比较

### 4.1 模式定义

**常量定义** ([messages.go#L39-L44](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L39-L44))：
```go
const (
    TxSubModeDefault  = "default"
    TxSubModeFallback = "fallback"
    TxSubModeExternal = "external"
)
```

### 4.2 行为对比表

| 维度 | default | fallback | external |
|------|---------|----------|----------|
| **DB查询** | 必须查询DB | 优先查询DB | 不查询DB |
| **收件人标识** | 支持 subscriber_emails 或 subscriber_ids | 仅支持 subscriber_emails | 仅支持 subscriber_emails |
| **订阅者不存在时** | 加入 notFound 列表，跳过发送 | 创建临时subscriber（仅含email） | 创建临时subscriber（仅含email） |
| **subscriber信息** | 完整DB信息（属性、列表等） | 完整DB信息 或 仅email | 仅email |
| **模板可访问数据** | `.Subscriber` 含完整信息 | `.Subscriber` 可能完整可能仅email | `.Subscriber` 仅email |
| **适用场景** | 已注册订阅者 | 可能注册也可能未注册 | 完全外部收件人 |
| **错误行为** | 不存在则报错，不发送 | 不存在也发送 | 始终发送 |

### 4.3 源码级差异

**default模式** ([tx.go#L100-L124](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L100-L124))：
```go
sub, err = a.core.GetSubscriber(subID, "", subEmail)
if err != nil {
    if er, ok := err.(*echo.HTTPError); ok && er.Code == http.StatusBadRequest {
        // default: log error and continue
        notFound = append(notFound, fmt.Sprintf("%v", er.Message))
        continue
    }
}
```

**fallback模式** ([tx.go#L115-L119](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L115-L119))：
```go
// fallback: Create an ephemeral "subscriber" if the subscriber wasn't found.
if m.SubscriberMode == models.TxSubModeFallback {
    sub = models.Subscriber{
        Email: subEmail,
    }
}
```

**external模式** ([tx.go#L92-L97](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L92-L97))：
```go
// external: Always create an ephemeral "subscriber" and don't lookup in the DB.
sub = models.Subscriber{
    Email: m.SubscriberEmails[n],
}
```

### 4.4 潜在问题

当混合使用 `fallback` 模式时，模板中如果访问 `{{ .Subscriber.Name }}` 等非 `Email` 字段，对于不存在的订阅者会得到空值，可能导致模板渲染结果不符合预期。

---

## 5. 模板缓存机制与陈旧问题

### 5.1 缓存结构

**Manager中的模板缓存** ([manager.go#L75-L76](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L75-L76))：
```go
tpls    map[int]*models.Template
tplsMut sync.RWMutex
```

使用 `map[int]*models.Template` 按模板ID缓存，`sync.RWMutex` 保护并发访问。

### 5.2 缓存操作函数

- **CacheTpl** ([manager.go#L319-L323](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L319-L323))：写入/更新缓存
- **DeleteTpl** ([manager.go#L326-L330](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L326-L330))：删除缓存
- **GetTpl** ([manager.go#L333-L343](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L333-L343))：读取缓存

### 5.3 缓存更新时机

1.  **系统启动**：`initTxTemplates` 加载并缓存所有tx模板
2.  **创建模板**：`CreateTemplate` ([templates.go#L140-L142](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L140-L142))：创建tx模板时调用 `CacheTpl`
3.  **更新模板**：`UpdateTemplate` ([templates.go#L179-L182](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L179-L182))：更新tx模板时调用 `CacheTpl` 覆盖旧缓存
4.  **删除模板**：`DeleteTemplate` ([templates.go#L207-L209](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L207-L209))：删除模板时调用 `DeleteTpl`

### 5.4 可能造成模板缓存陈旧的操作

| 操作 | 是否刷新缓存 | 风险 |
|------|------------|------|
| 通过API创建模板 | ✅ 自动 CacheTpl | 无 |
| 通过API更新模板 | ✅ 自动 CacheTpl 覆盖 | 无 |
| 通过API删除模板 | ✅ 自动 DeleteTpl | 无 |
| **直接修改数据库模板** | ❌ 不刷新 | **高风险** - 缓存与DB不一致 |
| **多实例部署，一个实例更新模板** | ❌ 仅更新当前实例缓存 | **高风险** - 其他实例缓存陈旧 |
| **修改模板后未重启** | ❌ 需重启或调用更新API | 缓存陈旧 |
| **数据库迁移/批量更新模板** | ❌ 不触发缓存刷新 | **高风险** |
| 系统重启 | ✅ 重新加载所有模板 | 无 |

### 5.5 缓存不一致场景示例

场景：多实例部署，实例A通过API更新了模板ID=1的内容：
1.  实例A调用 `UpdateTemplate` → 实例A的缓存已更新
2.  实例B的缓存仍为旧版本
3.  请求路由到实例B → 使用旧模板发送邮件
4.  请求路由到实例A → 使用新模板发送邮件
5.  导致同一模板ID发送出不同内容的邮件

---

## 6. 附件处理与大小限制问题

### 6.1 附件处理流程

```
multipart/form-data 请求
    ↓
c.MultipartForm()  (echo框架解析)
    ↓
遍历 form.File["file"]
    ↓
f.Open() 打开文件
    ↓
io.ReadAll(file) 读取全部内容到内存
    ↓
构造 models.Attachment
    ├─ Name: 文件名
    ├─ Header: MIME头 (Content-Disposition, Content-Type, Content-Transfer-Encoding)
    └─ Content: 文件内容字节数组
    ↓
追加到 m.Attachments
    ↓
Render 后复制到 msg.Attachments
    ↓
PushMessage 推入队列
    ↓
messenger.Push 发送
```

### 6.2 附件大小限制不一致问题

**当前实现问题**：

1.  **tx API 无显式附件大小限制**：
    - `tx.go` 中读取附件时没有大小检查
    - `schema.sql` 中 `upload.max_file_size` = 5000KB 仅用于 media 上传，不适用tx API

2.  **依赖框架默认限制**：
    - Go标准库 `http.Request.ParseMultipartForm` 默认限制为 32MB
    - echo框架 `BodyLimit` 中间件未启用
    - 不同部署环境（反向代理、WAF）可能有不同的大小限制

3.  **不一致表现**：
    - 小附件：正常处理
    - 超过Go默认32MB：`http: request body too large` 错误
    - 超过反向代理限制：反向代理返回413或连接断开
    - 内存不足：大附件导致OOM

4.  **错误时机不一致**：
    - 部分收件人渲染失败可能导致已发送的邮件无法撤回
    - 附件大小超限在请求解析阶段就失败，不会发送任何邮件

### 6.3 MakeAttachmentHeader

**位置**: [manager.go#L684-L697](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L684-L697)

构造附件的MIME头：
```go
h.Set("Content-Disposition", "attachment; filename="+filename)
h.Set("Content-Type", fmt.Sprintf("%s; name=\""+filename+"\"", contentType))
h.Set("Content-Transfer-Encoding", encoding)
```

---

## 7. 总结

### 7.1 核心流程回顾

1.  **解析**：`SendTxMessage` 解析请求和附件
2.  **校验**：`validateTxMessage` 校验参数合法性和模式
3.  **选模板**：从manager缓存获取预编译模板
4.  **选收件人**：根据 `subscriber_mode` 采用不同策略
5.  **渲染**：`TxMessage.Render` 渲染body、altbody、subject
6.  **投递**：`PushMessage` 推入队列异步发送

### 7.2 三种模式核心差异

- **default**：严格模式，必须DB存在，信息完整
- **fallback**：兼容模式，优先DB，不存在降级
- **external**：外部模式，不查DB，纯外部邮箱

### 7.3 模板缓存陈旧风险点

1.  直接修改DB不通过API
2.  多实例部署不同步
3.  数据库批量迁移
4.  无定时刷新机制

### 7.4 附件处理问题

1.  无显式大小限制，依赖框架默认值
2.  不同环境限制不一致
3.  大附件可能导致OOM
4.  部分失败导致发送状态不一致

---

## 8. 代码引用索引

| 函数/常量 | 文件位置 |
|----------|---------|
| SendTxMessage | [tx.go#L17-L176](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L17-L176) |
| validateTxMessage | [tx.go#L179-L244](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/tx.go#L179-L244) |
| initTxTemplates | [init.go#L625-L639](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L625-L639) |
| TxMessage.Render | [messages.go#L73-L133](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L73-L133) |
| Template.Compile | [templates.go#L39-L58](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/templates.go#L39-L58) |
| manager.PushMessage | [manager.go#L197-L209](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L197-L209) |
| TxSubMode* 常量 | [messages.go#L39-L44](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/messages.go#L39-L44) |
| hasTplExpr | [campaigns.go#L244-L248](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L244-L248) |
| CacheTpl/DeleteTpl/GetTpl | [manager.go#L318-L343](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L318-L343) |
| MakeAttachmentHeader | [manager.go#L684-L697](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L684-L697) |
