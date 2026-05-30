# Listmonk i18n 故障路径与安全审计分析报告

## 一、系统架构总览

Listmonk 的国际化体系由以下核心组件构成：

| 组件 | 文件 | 职责 |
|------|------|------|
| i18n 核心包 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go) | `New`/`Load`/`T`/`Ts`/`Tc` 翻译函数，语言包解析与参数替换 |
| HTTP handler | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go) | `getI18nLangList`/`GetI18nLang`/`getI18nLang`，语言列表与语言包获取 |
| 初始化 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go) | `initI18n`/`initConstConfig`/`initHTTPServer`，启动时加载语言与模板 |
| 前端入口 | [main.js](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/frontend/src/main.js) | VueI18n 初始化，从 `/api/lang/:lang` 拉取语言包 |
| 公共模板 | [public/templates/](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/static/public/templates/) | 使用 `L.T` 模板函数渲染 i18n 文本 |
| 语言包文件 | [i18n/*.json](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/i18n/) | 35 个 JSON 语言文件，en.json 为基准 |

---

## 二、语言包加载流程分析

### 2.1 启动时加载（后端单实例 i18n）

```
main() → initI18n(lang, fs) → getI18nLang(lang, fs)
```

[initI18n](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L551-L561) 调用 [getI18nLang](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L75-L98)，核心逻辑：

1. **先加载 en.json**（硬编码 `def = "en"`）作为基础语言包
2. **再加载目标语言**覆盖到 en 之上

```go
// 第78行：先读 en.json
b, err := fs.Read(fmt.Sprintf("/i18n/%s.json", def))
i, err := i18n.New(b)        // 用 en 初始化

// 第90行：再读目标语言覆盖
b, err = fs.Read(fmt.Sprintf("/i18n/%s.json", lang))
i.Load(b)                     // 覆盖到 en 之上
```

**关键问题**：后端 `App.i18n` 是启动时创建的**单实例**，整个进程生命周期内只有一个语言版本。配置 `app.lang` 决定了后端所有 `a.i18n.T()` 调用使用的语言。

### 2.2 前端动态加载

[main.js](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/frontend/src/main.js#L34-L41) 中：

```javascript
const [profile, cfg] = await Promise.all([api.getUserProfile(), api.getServerConfig()]);
const lang = await api.getLang(cfg.lang);   // GET /api/lang/:lang
i18n.locale = cfg.lang;
i18n.setLocaleMessage(i18n.locale, lang);
```

- 前端从 `/api/config` 获取 `lang` 字段
- 再从 `/api/lang/:lang` 获取完整语言包
- 设置到 VueI18n 实例中

### 2.3 语言列表发现

[getI18nLangList](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L43-L71) 通过 `fs.Glob("/i18n/*.json")` 扫描所有 JSON 文件，只读取 `_.code` 和 `_.name` 两个字段组成列表。

---

## 三、故障路径一：新增语言缺字段导致后端返回英文 key

### 3.1 故障根因

`i18n.New()` 只校验 `_.code` 和 `_.name`（[i18n.go#L29-L37](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L29-L37)），**不校验翻译条目完整性**：

```go
func New(b []byte) (*I18n, error) {
    var l map[string]string
    json.Unmarshal(b, &l)
    code, ok := l["_.code"]    // 只检查这两个
    name, ok := l["_.name"]    // 只检查这两个
    // ... 没有任何字段完整性校验
}
```

而 `i18n.Load()` 是简单 merge，也不做缺失检测（[i18n.go#L48-L59](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L48-L59)）：

```go
func (i *I18n) Load(b []byte) error {
    var l map[string]string
    json.Unmarshal(b, &l)
    for k, v := range l {
        i.langMap[k] = v   // 仅覆盖已有 key
    }
}
```

### 3.2 en 回退机制分析

`getI18nLang` 的 en 优先加载策略确实提供了隐式回退：新语言缺失的 key 会保留 en 的值。但这只对**后端启动时的单实例 i18n** 有效。

**问题出在前端**：`GetI18nLang` 返回的是 `i.JSON()`——即经过 en + 目标语言 merge 后的完整语言包。前端拿到的应该是不缺字段的。所以前端显示正常，但**后端直接返回 key 的场景**不同。

### 3.3 后端返回英文 key 的实际路径

当后端代码调用 `a.i18n.T("some.key")` 时，如果 `some.key` 在 en.json 中存在但在目标语言中不存在，en 回退机制**会正常工作**。真正返回英文 key 的情况是：

**路径A：i18n.T() key 不存在于 en.json**

[i18n.go#L68-L75](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L68-L75)：

```go
func (i *I18n) T(key string) string {
    s, ok := i.langMap[key]
    if !ok {
        return key   // 直接返回 key 字符串！
    }
    return i.getSingular(s)
}
```

如果后端代码中硬编码了一个 key（如 `"public.archiveSlug"`），但 en.json 中没有这个 key，则 **T() 直接返回 key 原始字符串**，不会显示任何翻译。

**路径B：Ts() 参数替换时 key 不存在**

[i18n.go#L86-L103](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L86-L103)：

```go
func (i *I18n) Ts(key string, params ...string) string {
    s, ok := i.langMap[key]
    if !ok {
        return key   // 直接返回 key，不做参数替换
    }
    // ...
}
```

如果 key 不存在，`Ts()` 返回 key 原文，所有 `{param}` 占位符都不会被替换，用户看到类似 `globals.messages.notFound` 的原始 key。

**路径C：新增 en.json 字段但旧语言文件未同步**

当 en.json 新增了字段（如 2FA 相关的 `users.twoFA`、`users.totpCode` 等），而旧语言文件尚未同步，en 回退能覆盖。但如果**新版本代码引用了 en.json 中也不存在的 key**，则任何语言都会显示 key 原文。

### 3.4 实际证据

对比 [en.json](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/i18n/en.json)（704行）和 [zh-CN.json](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/i18n/zh-CN.json)（702行），zh-CN 缺少以下 en.json 中存在的 key：

| 缺失 key | en.json 值 |
|-----------|-----------|
| `campaigns.confirmOverwriteContent` | `"This will overwrite all content. Continue?"` |
| `campaigns.customHeadersHelp` (含模板表达式版本) | 含 `{{ .Subscriber.Attribs.city }}` |
| `settings.smtp.fromAddresses` | `"From addresses / domains"` |
| `settings.smtp.fromAddressesHelp` | 域名路由说明 |
| `settings.smtp.retryDelay` | `"Retry delay"` |
| `settings.smtp.retryDelayHelp` | 重试延迟说明 |
| `settings.bounces.enableLettermint` | `"Enable Lettermint"` |
| `settings.bounces.lettermintKey` | `"Lettermint Webhook Secret"` |

这些缺失的 key 在后端通过 en 回退机制不会出问题，但**前端**从 `/api/lang/:lang` 获取的包是 merge 后的完整包，也不缺字段——所以这类缺失对用户不可见。

**真正可见的"返回英文 key"** 场景是：代码中 `T("key")` 引用的 key 在 **en.json 中也不存在**（代码与语言包不同步），此时任何语言都返回 key 原文。

---

## 四、故障路径二：非法 lang 参数

### 4.1 语言代码校验

[GetI18nLang](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L28-L40) 中有校验：

```go
var reLangCode = regexp.MustCompile(`[^a-zA-Z_0-9\\-]`)

func (a *App) GetI18nLang(c echo.Context) error {
    lang := c.Param("lang")
    if len(lang) > 6 || reLangCode.MatchString(lang) {
        return echo.NewHTTPError(http.StatusBadRequest, "Invalid language code.")
    }
    // ...
}
```

校验规则：
- 长度不超过 6 个字符
- 只允许 `[a-zA-Z0-9_-]`

### 4.2 存在的问题

**问题1：长度限制过严**

`len(lang) > 6` 的限制会**误杀合法语言代码**：

| 合法代码 | 长度 | 是否被误杀 |
|----------|------|-----------|
| `zh-CN` | 5 | ✅ 通过 |
| `pt-BR` | 5 | ✅ 通过 |
| `fr-CA` | 5 | ✅ 通过 |
| `es-419` | 6 | ✅ 通过（刚好） |
| `zh-Hans-CN` | 10 | ❌ 被拒绝 |
| `en-US-posix` | 12 | ❌ 被拒绝 |

如果有用户提交 `es-419.json`（长度6），刚好通过；但如果新增 `zh-Hans-CN.json` 就会被拒绝。

**问题2：下划线允许但文件名不使用**

正则允许 `_`，但所有 i18n 文件名使用 `-` 分隔（如 `pt-BR`、`fr-CA`），没有使用 `_` 的文件。这不影响安全性，但允许了无效的语言代码通过校验后在 `getI18nLang` 中读文件失败——此时会回退到 en 并返回警告。

**问题3：校验与文件系统路径构造脱节**

虽然 `reLangCode` 阻止了 `../` 和特殊字符，但路径构造在 `getI18nLang` 中：

```go
b, err = fs.Read(fmt.Sprintf("/i18n/%s.json", lang))
```

`lang` 由 `c.Param("lang")` 获取。Echo 的路由参数 `/api/lang/:lang` 在提取时**不会做路径清洗**——这部分由 `reLangCode` 保障。

---

## 五、故障路径三：路径遍历攻击 /api/i18n/../../secret

### 5.1 攻击场景

恶意用户请求：`GET /api/i18n/../../secret`

### 5.2 防御层分析

**第一层：Echo 路由匹配**

路由注册为 `g.GET("/api/lang/:lang", a.GetI18nLang)`（[handlers.go#L104](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L104)）。注意路由路径是 `/api/lang/:lang`，**不是** `/api/i18n/:lang`。

攻击者请求 `/api/i18n/../../secret`：
- **不会匹配** `/api/lang/:lang` 路由
- 会落到 `g.RouteNotFound("/api/*", ...)` 返回 404

> ⚠️ 注意：路由路径是 `/api/lang/:lang`，不存在 `/api/i18n/:lang` 路由。攻击路径 `/api/i18n/../../secret` 不会被该 handler 处理。

**第二层：reLangCode 正则校验**

假设攻击者修正路径为 `GET /api/lang/../../secret`：

```go
lang := c.Param("lang")  // Echo 提取 :lang 参数
// 对于 /api/lang/../../secret，Echo 可能提取 "../" 或 ".."
```

`reLangCode` 匹配 `[^a-zA-Z_0-9\\-]`，点号 `.` 不在允许字符集中：
- `".."` 包含 `.` → `reLangCode.MatchString("..")` 返回 `true` → 校验失败 → 返回 400

**第三层：stuffbin 文件系统**

即使校验被绕过，`fs.Read(fmt.Sprintf("/i18n/%s.json", lang))` 使用的是 stuffbin 虚拟文件系统，**不是 os.ReadFile**。stuffbin 的文件系统是嵌入式的或受控的本地映射，路径遍历在虚拟 FS 中不会逃逸到宿主文件系统。

### 5.3 安全评估

| 防御层 | 机制 | 有效性 |
|--------|------|--------|
| Echo 路由 | `/api/lang/:lang` 路由不匹配 `/api/i18n/...` | ✅ 完全阻断错误路径前缀 |
| 正则校验 | `reLangCode` 拒绝 `.`、`/`、空格等 | ✅ 阻止路径遍历字符 |
| 长度校验 | `len(lang) > 6` | ✅ `../../secret` 远超6字符 |
| stuffbin FS | 虚拟文件系统隔离 | ✅ 纵深防御，即使绕过也无法逃逸 |

**结论**：`/api/i18n/../../secret` 路径遍历攻击在当前代码中**不可行**。但正则中允许 `_` 且长度限制仅6字符是两个设计缺陷，建议改进。

### 5.4 潜在绕过场景

如果未来有人添加了 `/api/i18n/:lang` 路由别名且忘记了校验，则 `reLangCode` 和长度校验是唯一的防线。此时正则中的 `\\-`（双重转义）值得注意：

```go
var reLangCode = regexp.MustCompile(`[^a-zA-Z_0-9\\-]`)
```

在 Go 原始字符串中 `\\-` 匹配的是字面量 `\` 和 `-`（在字符类中 `-` 在末尾不需要转义）。实际上这等价于 `[^a-zA-Z_0-9\-]`，即不允许反斜杠和 `-` 以外的特殊字符……但 `-` 在字符类末尾是字面量，所以 `\\-` 实际允许了 `\` 字符！

**等等**，重新分析：在 Go 原始字符串 `` `[^a-zA-Z_0-9\\-]` `` 中：
- `\\` 是一个字面反斜杠 `\`
- `-` 在字符类中

这会编译为正则 `[^a-zA-Z_0-9\-]`，在字符类中 `\-` 就是转义的 `-`，即字面量 `-`。

所以实际效果是：允许 `a-zA-Z`、`0-9`、`_`、`-`，其他字符被拒绝。这是正确的。

但反斜杠 `\` 在 URL path 中经过 Echo 提取时会被 URL 解码，所以实际输入中很难注入 `\`。

---

## 六、故障路径四：嵌套参数递归替换

### 6.1 subAllParams 递归机制

[i18n.go#L149-L163](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L149-L163)：

```go
func (i *I18n) subAllParams(s string) string {
    if !strings.Contains(s, `{`) {
        return s
    }
    parts := reParam.FindAllStringSubmatch(s, -1)
    for _, p := range parts {
        s = strings.ReplaceAll(s, p[0], i.T(p[1]))
    }
    return i.subAllParams(s)   // 递归调用！
}
```

`Ts()` 在替换参数值时调用 `subAllParams`（[i18n.go#L99](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L99)）：

```go
val := i.subAllParams(params[n+1])
s = strings.ReplaceAll(s, `{`+params[n]+`}`, val)
```

### 6.2 递归终止条件

递归在 `!strings.Contains(s, `{`)` 时终止。只要替换结果不产生新的 `{...}` 模式，递归就会停止。

### 6.3 无限递归风险

**场景1：循环引用**

如果 en.json 中：
```json
"key.a": "{key.b}",
"key.b": "{key.a}"
```

则 `i.T("key.a")` → `subAllParams("{key.b}")` → `i.T("key.b")` → `"key.b"` 的值是 `"{key.a}"` → `subAllParams("{key.a}")` → `i.T("key.a")` → ... **无限递归 → 栈溢出 panic**

**场景2：自引用**

```json
"key.self": "{key.self}"
```

`i.T("key.self")` → `"key.self"` 不含 `{` → 返回 `"{key.self}"`。但在 `Ts()` 的 `subAllParams` 调用中，参数值如果是 `"{key.self}"`，会触发：`subAllParams("{key.self}")` → `i.T("key.self")` → `"{key.self}"` → `subAllParams("{key.self}")` → **无限递归**

### 6.4 Ts() 中的双重替换

`Ts()` 中存在**两层参数替换**：

1. **第一层**：`subAllParams(params[n+1])` — 对参数值中的 `{...}` 做翻译替换
2. **第二层**：`strings.ReplaceAll(s, `{`+params[n]+`}`, val)` — 将模板中的 `{param}` 替换为参数值

这意味着如果参数值中包含 `{another_key}`，它会被先翻译再插入。如果翻译结果又产生 `{yet_another}`，递归继续。

**无递归深度限制**：代码中没有递归深度计数器或超时机制，恶意构造的语言文件可以导致 DoS。

---

## 七、故障路径五：Public 模板渲染失败

### 7.1 模板渲染链

```
HTTP Request → handler → c.Render(status, templateName, data) → tplRenderer.Render()
```

[tplRenderer.Render](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L106-L119)：

```go
func (t *tplRenderer) Render(w io.Writer, name string, data any, c echo.Context) error {
    return t.templates.ExecuteTemplate(w, name, tplData{
        // ...
        Data: data,
        L:    c.Get("app").(*App).i18n,   // 从 App 获取 i18n 实例
    })
}
```

模板中通过 `L.T "key"` 调用翻译（见 [subscription.html](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/static/public/templates/subscription.html#L5)）。

### 7.2 故障场景分析

**场景A：i18n key 缺失导致原始 key 显示**

[message.html](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/static/public/templates/message.html) 是最简单的模板，只渲染 `.Data.Title` 和 `.Data.Message`。Title 和 Message 由 handler 通过 `makeMsgTpl()` 构造。

例如 [handlers.go#L296-L298](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L296-L298)：

```go
g.RouteNotFound("/*", func(c echo.Context) error {
    return c.Render(http.StatusNotFound, tplMessage,
        makeMsgTpl("404 - "+a.i18n.T("public.notFoundTitle"), "", ""))
})
```

如果 `public.notFoundTitle` 不在语言包中，`T()` 返回 key 原文，页面显示 `"404 - public.notFoundTitle"`。

**场景B：Ts() 参数替换失败导致裸 key + 占位符**

[public.go#L161](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L161)：

```go
makeMsgTpl(a.i18n.T("public.errorTitle"), "", a.i18n.Ts("public.errorFetchingCampaign"))
```

`public.errorFetchingCampaign` 的 en 值是 `"Error fetching e-mail message."`——不含参数，`Ts()` 正常。

但 [public.go#L299-L300](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L299-L300)：

```go
a.i18n.Ts("globals.messages.pFound",
    "name", a.i18n.T("globals.terms.subscriber"))
```

如果 `globals.messages.pFound` 不在语言包中，`Ts()` 返回 `"globals.messages.pFound"` 原文，参数 `"name"` 和 `"globals.terms.subscriber"` 的翻译结果被丢弃。

**场景C：模板执行 panic 导致 500**

Go 的 `template.ExecuteTemplate` 在调用 `L.T` 时，如果 `L` 为 nil 或 `L.T` 方法 panic（如无限递归栈溢出），模板执行会 panic，Echo 的 recover 中间件会捕获并返回 500。

**场景D：新语言文件 JSON 格式错误导致启动失败**

[i18n.New()](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L23-L44) 中 `json.Unmarshal` 失败会返回 error。在 [getI18nLang](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L90-L95) 中，如果目标语言 JSON 解析失败：

```go
b, err = fs.Read(fmt.Sprintf("/i18n/%s.json", lang))
if err != nil {
    return i, true, fmt.Errorf(...)   // 返回 en 回退 + 警告
}
if err := i.Load(b); err != nil {
    return i, true, fmt.Errorf(...)   // 返回 en 回退 + 警告
}
```

**注意**：当 `Load()` 失败时，返回的是 `(i, true, error)`——即返回了仅含 en 的 i18n 实例且 `ok=true`。这意味着：
- 后端：启动时 `initI18n` 只打印警告，不会 fatal
- 前端：`GetI18nLang` 返回 en 的语言包给前端，前端看到的语言代码是 en 而非用户请求的语言

**场景E：后端 i18n 单实例 vs 前端动态加载不一致**

后端 `a.i18n` 在启动时根据 `app.lang` 配置初始化一次。如果配置是 `en`，后端所有公共页面都用 en 渲染，**无论用户偏好什么语言**。前端 admin 面板可以动态切换语言，但公共页面（subscription、optin 等）始终使用后端配置的语言。

这是**设计缺陷**：公共订阅页面不支持用户语言偏好，始终使用 `app.lang` 配置值。

---

## 八、故障路径汇总

| # | 故障路径 | 触发条件 | 影响 | 严重性 |
|---|---------|---------|------|--------|
| 1 | T()/Ts() key 不存在 | 代码引用了 en.json 中也没有的 key | 页面显示原始 key 字符串 | 中 |
| 2 | Ts() 参数数量为奇数 | 调用方传了奇数个参数 | 返回 `"key: Invalid arguments"` | 低 |
| 3 | subAllParams 无限递归 | 语言文件中存在循环/自引用 `{key}` | 栈溢出 panic → 500 | 高 |
| 4 | reLangCode 长度限制 | 合但长的语言代码如 `zh-Hans-CN` | 合法语言被拒绝 | 中 |
| 5 | Load() 失败仍返回 ok=true | 新语言 JSON 格式错误 | 前端收到 en 语言包但以为是目标语言 | 中 |
| 6 | 公共页面语言固定 | app.lang 配置为 en | 非英语用户看到英文公共页面 | 低 |
| 7 | 路径遍历尝试 | `/api/i18n/../../secret` | 被路由+正则+stuffbin 三层阻断 | 无（已防御） |
| 8 | 新增 en.json 字段未同步 | 版本升级新增翻译 key | 后端 en 回退正常；前端也因 merge 正常 | 低 |

---

## 九、代码行级引用索引

| 功能 | 文件 | 行号 |
|------|------|------|
| i18n.New() 校验 _.code/_.name | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L23-L44) | L23-L44 |
| i18n.Load() merge 无完整性校验 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L48-L59) | L48-L59 |
| i18n.T() key 不存在返回 key | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L68-L75) | L68-L75 |
| i18n.Ts() 奇数参数返回错误 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L87-L89) | L87-L89 |
| i18n.Ts() key 不存在返回 key | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L91-L94) | L91-L94 |
| i18n.subAllParams() 递归无深度限制 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/i18n/i18n.go#L149-L163) | L149-L163 |
| reLangCode 正则与长度校验 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L25-L32) | L25-L32 |
| GetI18nLang handler | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L28-L40) | L28-L40 |
| getI18nLang en 回退机制 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L75-L98) | L75-L98 |
| getI18nLangList 扫描与排序 | [i18n.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/i18n.go#L43-L71) | L43-L71 |
| initI18n 启动加载 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L551-L561) | L551-L561 |
| initConstConfig 读取 app.lang | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L504) | L504 |
| tplRenderer.Render 注入 i18n | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L106-L119) | L106-L119 |
| 前端 VueI18n 初始化 | [main.js](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/frontend/src/main.js#L34-L41) | L34-L41 |
| 前端 getLang API 调用 | [api/index.js](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/frontend/src/api/index.js#L451-L454) | L451-L454 |
| 路由注册 /api/lang/:lang | [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L104) | L104 |
| 404 路由使用 i18n.T | [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L296-L298) | L296-L298 |
| initTplFuncs 注册 L 模板函数 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L1073-L1104) | L1087-L1089 |
| GetServerConfig 返回语言列表 | [admin.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/admin.go#L41-L77) | L71-L77 |

---

## 十、修复建议

### 10.1 subAllParams 递归深度限制

```go
func (i *I18n) subAllParams(s string, depth int) string {
    if depth > 5 || !strings.Contains(s, `{`) {
        return s
    }
    parts := reParam.FindAllStringSubmatch(s, -1)
    for _, p := range parts {
        s = strings.ReplaceAll(s, p[0], i.T(p[1]))
    }
    return i.subAllParams(s, depth+1)
}
```

### 10.2 reLangCode 长度限制放宽

```go
if len(lang) > 16 || reLangCode.MatchString(lang) {
```

### 10.3 New()/Load() 增加字段完整性校验

启动时对比 en.json 与目标语言的 key 差异，输出 warning 日志。

### 10.4 Load() 失败应返回 ok=false

```go
if err := i.Load(b); err != nil {
    return i, false, fmt.Errorf(...)   // ok=false 让调用方决定是否 fatal
}
```

### 10.5 公共页面支持 Accept-Language

在 `tplRenderer.Render` 中读取 `Accept-Language` header，动态选择 i18n 实例，而非使用全局单例。
