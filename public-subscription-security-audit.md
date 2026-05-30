# 公开订阅页安全审计：请求流与防护分析

## 1. 概述

本文档追踪一条从机器人发出的恶意订阅请求，经过 HTML 模板/前端脚本、后端 handler、captcha 验证、列表类型过滤、域名 blocklist/allowlist、到核心写入路径的完整生命周期，逐一说明每个检查点在何处发生、请求是被接受还是被拒绝，并指出前端检查与后端检查的边界。最后给出三项回归测试方案。

---

## 2. 请求入口与路由

两个公开入口均注册在无认证的公共路由组：

| 路由 | 方法 | Handler | 认证 |
|---|---|---|---|
| `/subscription/form` | GET | `SubscriptionFormPage` | 无 |
| `/subscription/form` | POST | `SubscriptionForm` | 无 |
| `/api/public/subscription` | POST | `PublicSubscription` | 无 |

路由定义见 [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L260-L261) 第 260–261 行：

```go
g.GET("/api/public/lists", a.GetPublicLists)
g.POST("/api/public/subscription", a.PublicSubscription)
```

以及 [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L269-L270) 第 269–270 行：

```go
g.GET("/subscription/form", a.SubscriptionFormPage)
g.POST("/subscription/form", a.SubscriptionForm)
```

**关键区别**：`SubscriptionForm`（HTML 表单提交）包含 nonce 蜜罐和 captcha 校验，而 `PublicSubscription`（JSON API）**两者都不包含**。

---

## 3. 前端检查（可被绕过）

### 3.1 SubscriptionFormPage — HTML 模板渲染

入口：[public.go#SubscriptionFormPage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L413-L448)

模板文件：[subscription-form.html](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/static/public/templates/subscription-form.html)

前端层面的检查包括：

| 检查项 | 实现位置 | 可被绕过？ |
|---|---|---|
| `<input required="true" type="email">` — HTML5 邮箱校验 | 模板第 10 行 | ✅ 直接 POST 可绕过 |
| `<input name="nonce" class="nonce" value="" />` — CSS 隐藏蜜罐字段 | 模板第 12 行 | ✅ 机器人不填即可绕过前端，但**后端有对应检查**（见 §4.1） |
| 列表 checkbox `checked="true"` 仅渲染 `ListTypePublic` 列表 | 模板第 21–29 行 | ✅ 机器人可自行构造 `l` 参数提交任意 UUID |
| hCaptcha / Altcha widget 渲染 | 模板第 32–41 行 | ✅ 机器人可不渲染 widget，直接构造 POST |
| CSS `.nonce { display: none }` 隐藏蜜罐 | [style.css](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/static/public/static/style.css#L143-L145) 第 143–145 行 | ✅ CSS 对机器人无意义 |

**结论**：前端所有检查都可被绕过。安全性**必须**依赖后端验证。

---

## 4. 后端检查详解

### 4.1 EnablePublicSubPage 开关

两个入口均首先检查此开关：

```go
// SubscriptionForm — 第 453–455 行
if !a.cfg.EnablePublicSubPage {
    return echo.NewHTTPError(http.StatusNotFound, a.i18n.T("public.invalidFeature"))
}

// PublicSubscription — 第 517–519 行
if !a.cfg.EnablePublicSubPage {
    return echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.invalidFeature"))
}
```

✅ **后端检查，无法绕过**（配置级）。

### 4.2 Nonce 蜜罐（仅 SubscriptionForm）

```go
// SubscriptionForm — 第 459–461 行
if c.FormValue("nonce") != "" {
    return echo.NewHTTPError(http.StatusBadGateway, a.i18n.T("public.invalidFeature"))
}
```

✅ **后端检查**。若机器人盲目填写所有字段，nonce 非空则请求被拒。

⚠️ **注意**：此检查**仅在 `SubscriptionForm` 中执行**，`PublicSubscription`（JSON API 端点）不检查 nonce。因此机器人若直接 POST 到 `/api/public/subscription`，蜜罐完全不生效。

### 4.3 Captcha 验证（仅 SubscriptionForm）

```go
// SubscriptionForm — 第 464–492 行
if a.captcha.IsEnabled() {
    var val string
    switch a.captcha.GetProvider() {
    case captcha.ProviderHCaptcha:
        val = c.FormValue("h-captcha-response")
    case captcha.ProviderAltcha:
        val = c.FormValue("altcha")
    default:
        return c.Render(http.StatusBadRequest, ...)
    }
    if val == "" {
        return c.Render(http.StatusBadRequest, ...)
    }
    err, ok := a.captcha.Verify(val)
    if err != nil { a.log.Printf("captcha request failed: %v", err) }
    if !ok {
        return c.Render(http.StatusBadRequest, ...)
    }
}
```

✅ **后端检查**。验证流程：

1. 读取对应 provider 的表单字段值
2. 空值直接拒绝
3. 调用 `a.captcha.Verify(val)` 服务端验证

⚠️ **关键安全缺口**：Captcha 检查**仅在 `SubscriptionForm` 中执行**，`PublicSubscription`（`/api/public/subscription`）**完全跳过 captcha**。机器人可直接 POST 到 JSON API 绕过所有 captcha 防护。

### 4.3.1 Captcha Verify 实现

[captcha.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/captcha/captcha.go#L147-L204)

**hCaptcha**（第 159–184 行）：
- 向 `https://hcaptcha.com/siteverify` POST 验证 token
- 使用服务端 secret key，机器人无法伪造

**Altcha**（第 187–204 行）：
- 使用 `altcha.VerifySolution()` 校验签名
- **防重放**：使用 `tmptokens` 包记录已使用的 payload，5 分钟 TTL 内拒绝重复 token
- HMAC key 在服务端启动时随机生成

```go
// captcha.go 第 198–201 行
if _, err := tmptokens.Check(payload); err == nil {
    return fmt.Errorf("captcha token already used"), false
}
tmptokens.Set(payload, 5*time.Minute, nil)
```

✅ Altcha 有防重放机制。hCaptcha 由第三方服务端验证，同样安全。

### 4.4 processSubForm — 核心业务处理

[public.go#processSubForm](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L706-L788)

此函数被 `SubscriptionForm` 和 `PublicSubscription` **共用**。

#### 4.4.1 表单绑定与字段校验

```go
var req struct {
    Name          string   `form:"name" json:"name"`
    Email         string   `form:"email" json:"email"`
    FormListUUIDs []string `form:"l" json:"list_uuids"`
}
if err := c.Bind(&req); err != nil { return false, err }
```

- 支持 `form` 和 `json` 两种绑定方式，因此机器人可用 JSON 或 form-data 提交

```go
if len(req.FormListUUIDs) == 0 {
    return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.noListsSelected"))
}
if len(req.Email) > 1000 {
    return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("subscribers.invalidEmail"))
}
```

✅ **后端检查**：必须至少选择一个列表，邮箱长度不超过 1000。

#### 4.4.2 Email 清洗与域名 Blocklist/Allowlist

```go
em, err := a.importer.SanitizeEmail(req.Email)
if err != nil {
    return false, echo.NewHTTPError(http.StatusBadRequest, err.Error())
}
req.Email = em
```

[importer.go#SanitizeEmail](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L607-L636) 执行两层验证：

**第一层**：[utils.SanitizeEmail](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/utils/utils.go#L25-L32)
- Trim + Lowercase
- `mail.ParseAddress` RFC 校验
- 拒绝包含 display name 的地址

**第二层**：域名 blocklist/allowlist
```go
if im.hasAllowlist || im.hasBlocklist {
    d := strings.Split(addr, "@")
    domain := d[1]

    if im.hasAllowlist {
        if !im.checkInList(domain, im.hasAllowlistWildcards, im.domainAllowlist) {
            return "", errors.New(im.i18n.T("subscribers.domainBlocklisted"))
        }
    } else if im.hasBlocklist {
        if im.checkInList(domain, im.hasBlocklistWildcards, im.domainBlocklist) {
            return "", errors.New(im.i18n.T("subscribers.domainBlocklisted"))
        }
    }
}
```

✅ **后端检查**。allowlist 优先于 blocklist；支持 `*.example.com` 通配符匹配。

#### 4.4.3 Name 校验

```go
req.Name = strings.TrimSpace(req.Name)
if len(req.Name) == 0 {
    req.Name = strings.Split(req.Email, "@")[0]
} else if len(req.Name) > stdInputMaxLen {
    return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("subscribers.invalidName"))
}
```

✅ **后端检查**。name 为空则取邮箱前缀，超过 2000 字符则拒绝。

#### 4.4.4 列表类型过滤（Private List 防护）

```go
listTypes, err := a.core.GetListTypes(nil, req.FormListUUIDs)
if err != nil {
    return false, echo.NewHTTPError(http.StatusInternalServerError, ...)
}

for _, t := range listTypes {
    if t == models.ListTypePrivate {
        return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("globals.messages.invalidUUID"))
    }
}
```

[GetListTypes](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/lists.go#L126-L146) 通过 SQL 查询 `lists` 表获取每个 UUID 对应的 `type`：

```sql
-- get-list-types
SELECT id, uuid, type FROM lists WHERE
    (CASE WHEN $1::INT[] IS NOT NULL THEN id = ANY($1::INT[])
          WHEN $2::UUID[] IS NOT NULL THEN uuid = ANY($2::UUID[])
    END);
```

✅ **后端检查**。若提交的 UUID 中包含 `type = 'private'` 的列表，请求被拒。

⚠️ **注意**：若机器人提交了**不存在的 UUID**，`GetListTypes` 返回的 map 中不会包含该 UUID，循环也不会命中 `ListTypePrivate`。但后续 `InsertSubscriber` SQL 中 `listIDs` CTE 只匹配存在的列表：

```sql
listIDs AS (
    SELECT id FROM lists WHERE
        (CASE WHEN CARDINALITY($6::INT[]) > 0 THEN id=ANY($6)
              ELSE uuid=ANY($7::UUID[]) END)
)
```

不存在的 UUID 被静默忽略，不会报错。这意味着**机器人提交一个混合了 public + 不存在 UUID 的请求时，public 列表仍然会被订阅**。

#### 4.4.5 核心写入路径 — InsertSubscriber

```go
_, hasOptin, err := a.core.InsertSubscriber(models.Subscriber{
    Name:   req.Name,
    Email:  req.Email,
    Status: models.SubscriberStatusEnabled,
}, nil, listUUIDs, false, true)
```

[InsertSubscriber](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L286-L349) 执行：

1. 生成新 UUID
2. 设置 `subStatus = 'unconfirmed'`（`preconfirm = false`）
3. 执行 `insert-subscriber` SQL：
   - INSERT subscriber，若 email 冲突（`subscribers_email_key` 唯一约束 → `idx_subs_email ON subscribers(LOWER(email))`）返回 `http.StatusConflict`
   - INSERT `subscriber_lists`，`ON CONFLICT (subscriber_id, list_id) DO UPDATE` 设置 status
4. 发送 optin 确认邮件（若有 double-optin 列表）

#### 4.4.6 重复邮箱处理

```go
if e, ok := err.(*echo.HTTPError); ok && e.Code == http.StatusConflict {
    sub, err := a.core.GetSubscriber(0, "", req.Email)
    if err != nil { return false, err }

    _, hasOptin, err := a.core.UpdateSubscriberWithLists(sub.ID, sub, nil, listUUIDs, false, false, true, nil, true)
    if err == nil { return hasOptin, nil }
}
```

当 email 已存在时，走 `UpdateSubscriberWithLists` 路径。关键参数 `allowResubscribe = true`：

[update-subscriber-with-lists SQL](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L160-L201)：

```sql
ON CONFLICT (subscriber_id, list_id) DO UPDATE
SET status = (
    CASE
        WHEN $4='blocklisted' THEN 'unsubscribed'::subscription_status
        WHEN subscriber_lists.status = 'confirmed' THEN 'confirmed'
        WHEN $11 = TRUE THEN $8::subscription_status
        WHEN subscriber_lists.status = 'unsubscribed' THEN 'unsubscribed'::subscription_status
        ELSE $8::subscription_status
    END
)
```

`$11 = allowResubscribe = TRUE`，`$8 = subStatus = 'unconfirmed'`。因此：
- **已 confirmed 的订阅**：保持 confirmed（不会被降级）✅
- **已 unsubscribed 的订阅**：保持 unsubscribed（**不会自动重新订阅**）✅
- **其他状态**：设为 unconfirmed

⚠️ **注意**：已 unsubscribed 的订阅不会通过公开表单重新订阅，这是设计如此。但机器人反复提交同一邮箱不会导致错误，只是不会改变已确认/已退订的状态。

---

## 5. 检查点汇总表

| # | 检查项 | 前端/后端 | SubscriptionForm | PublicSubscription | 机器人能否绕过 |
|---|---|---|---|---|---|
| 1 | EnablePublicSubPage 开关 | 后端 | ✅ 检查 | ✅ 检查 | 否（配置级） |
| 2 | HTML5 required/type 校验 | 前端 | ✅ | N/A | 是 |
| 3 | Nonce 蜜罐字段 | 后端 | ✅ 检查 | ❌ 不检查 | 是（用 JSON API） |
| 4 | Captcha 验证 | 后端 | ✅ 检查 | ❌ 不检查 | 是（用 JSON API） |
| 5 | 列表不能为空 | 后端 | ✅ | ✅ | 否 |
| 6 | Email 长度 ≤ 1000 | 后端 | ✅ | ✅ | 否 |
| 7 | Email RFC 校验 + 清洗 | 后端 | ✅ | ✅ | 否 |
| 8 | 域名 Allowlist/Blocklist | 后端 | ✅ | ✅ | 否 |
| 9 | Name 长度 ≤ 2000 | 后端 | ✅ | ✅ | 否 |
| 10 | Private list 过滤 | 后端 | ✅ | ✅ | 否 |
| 11 | Email 唯一约束 | 后端（DB） | ✅ | ✅ | 否 |
| 12 | 重复订阅状态保护 | 后端（DB） | ✅ | ✅ | 否 |

---

## 6. 安全风险分析

### 6.1 PublicSubscription 端点绕过 Captcha 和 Nonce

**严重程度：高**

`/api/public/subscription` 端点直接调用 `processSubForm`，跳过了：
- nonce 蜜罐检查
- captcha 验证

机器人可直接向此端点 POST JSON：

```bash
curl -X POST http://host/api/public/subscription \
  -H "Content-Type: application/json" \
  -d '{"email":"bot@example.com","name":"Bot","list_uuids":["<public-list-uuid>"]}'
```

无论 captcha 是否启用，此请求都会被处理。

### 6.2 不存在的 UUID 静默忽略

`GetListTypes` 只查询存在的列表。若机器人提交包含不存在 UUID 的请求，这些 UUID 被静默忽略，不会产生错误，也不会被订阅到任何列表。虽然不是安全漏洞，但可能让攻击者探测哪些 UUID 有效。

### 6.3 重复邮箱请求无速率限制

`processSubForm` 没有内置速率限制。对于已存在的邮箱，每次请求都会：
1. 尝试 INSERT → 冲突
2. 查询已有 subscriber
3. 执行 UPDATE（但不会改变 confirmed/unsubscribed 状态）

这意味着机器人可以高频提交相同邮箱，虽然数据不会损坏，但会产生不必要的数据库负载和潜在的 optin 邮件洪泛。

---

## 7. 回归测试方案

### 7.1 测试：绕过 Private List

**目的**：验证机器人无法通过公开端点订阅 private 列表。

```go
// TestBypassPrivateList 验证提交 private list UUID 会被拒绝
func TestBypassPrivateList(t *testing.T) {
    // 前置条件：数据库中存在一个 type='private' 的列表
    privateListUUID := "<private-list-uuid>"
    publicListUUID := "<public-list-uuid>"

    tests := []struct {
        name       string
        listUUIDs  []string
        wantStatus int
        wantErr    bool
    }{
        {
            name:       "仅提交 private list UUID",
            listUUIDs:  []string{privateListUUID},
            wantStatus: http.StatusBadRequest,
            wantErr:    true,
        },
        {
            name:       "混合提交 public + private UUID",
            listUUIDs:  []string{publicListUUID, privateListUUID},
            wantStatus: http.StatusBadRequest,
            wantErr:    true,
        },
        {
            name:       "仅提交 public list UUID",
            listUUIDs:  []string{publicListUUID},
            wantStatus: http.StatusOK,
            wantErr:    false,
        },
        {
            name:       "提交不存在的 UUID",
            listUUIDs:  []string{"00000000-0000-0000-0000-000000000000"},
            wantStatus: http.StatusBadRequest, // 空列表（不存在的UUID被忽略后FormListUUIDs为空）
            wantErr:    true,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            // 构造请求
            body := map[string]any{
                "email":      "test@example.com",
                "name":       "Test",
                "list_uuids": tt.listUUIDs,
            }
            jsonBody, _ := json.Marshal(body)

            req := httptest.NewRequest(http.MethodPost, "/api/public/subscription", bytes.NewReader(jsonBody))
            req.Header.Set("Content-Type", "application/json")
            rec := httptest.NewRecorder()

            // 执行 handler（需要注入有 private list 的 core）
            // ... app.PublicSubscription(echo.NewContext(req, rec))

            if tt.wantErr {
                assert.Equal(t, tt.wantStatus, rec.Code)
            } else {
                assert.Equal(t, tt.wantStatus, rec.Code)
            }

            // 额外验证：数据库中 subscriber 不应关联到 private list
            if !tt.wantErr {
                var count int
                db.Get(&count, `
                    SELECT COUNT(*) FROM subscriber_lists sl
                    JOIN lists l ON sl.list_id = l.id
                    JOIN subscribers s ON sl.subscriber_id = s.id
                    WHERE s.email = 'test@example.com' AND l.type = 'private'
                `)
                assert.Equal(t, 0, count, "subscriber should NOT be subscribed to any private list")
            }
        })
    }
}
```

**验证逻辑**：
- 提交 private list UUID → 400 Bad Request
- 混合提交 → 400 Bad Request（只要有一个 private 就拒绝整个请求）
- 仅提交 public list UUID → 200 OK
- 数据库断言：subscriber 不关联任何 private list

### 7.2 测试：绕过 Captcha

**目的**：验证 `/api/public/subscription` 端点在 captcha 启用时仍接受无 captcha 的请求（当前行为），以及验证 `/subscription/form` 端点在 captcha 启用时拒绝无 captcha 的请求。

```go
// TestBypassCaptcha 验证 captcha 在不同端点的执行情况
func TestBypassCaptcha(t *testing.T) {
    // 前置条件：启用 captcha（例如 altcha）
    // app.cfg.Security.Captcha.Altcha.Enabled = true

    t.Run("SubscriptionForm_无captcha_应被拒绝", func(t *testing.T) {
        form := url.Values{
            "email": {"bot@example.com"},
            "name":  {"Bot"},
            "l":     {"<public-list-uuid>"},
            // 注意：没有 h-captcha-response 或 altcha 字段
        }
        req := httptest.NewRequest(http.MethodPost, "/subscription/form", strings.NewReader(form.Encode()))
        req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
        rec := httptest.NewRecorder()

        // ... app.SubscriptionForm(echo.NewContext(req, rec))
        assert.Equal(t, http.StatusBadRequest, rec.Code)
    })

    t.Run("SubscriptionForm_伪造captcha_应被拒绝", func(t *testing.T) {
        form := url.Values{
            "email":  {"bot@example.com"},
            "name":   {"Bot"},
            "l":      {"<public-list-uuid>"},
            "altcha": {"fake-altcha-payload"},
        }
        req := httptest.NewRequest(http.MethodPost, "/subscription/form", strings.NewReader(form.Encode()))
        req.Header.Set("Content-Type", "application/x-www-form-urlencoded")
        rec := httptest.NewRecorder()

        // ... app.SubscriptionForm(echo.NewContext(req, rec))
        assert.Equal(t, http.StatusBadRequest, rec.Code)
    })

    t.Run("SubscriptionForm_重放captcha_应被拒绝", func(t *testing.T) {
        // 第一步：通过正常流程获取一个有效的 altcha payload
        validPayload := solveAltchaChallenge(t, app)

        // 第二步：首次使用 — 应成功
        form1 := url.Values{
            "email":  {"first@example.com"},
            "name":   {"First"},
            "l":      {"<public-list-uuid>"},
            "altcha": {validPayload},
        }
        req1 := httptest.NewRequest(http.MethodPost, "/subscription/form", strings.NewReader(form1.Encode()))
        req1.Header.Set("Content-Type", "application/x-www-form-urlencoded")
        rec1 := httptest.NewRecorder()
        // ... app.SubscriptionForm(echo.NewContext(req1, rec1))
        assert.Equal(t, http.StatusOK, rec1.Code)

        // 第三步：重放同一 payload — 应被拒绝
        form2 := url.Values{
            "email":  {"second@example.com"},
            "name":   {"Second"},
            "l":      {"<public-list-uuid>"},
            "altcha": {validPayload},
        }
        req2 := httptest.NewRequest(http.MethodPost, "/subscription/form", strings.NewReader(form2.Encode()))
        req2.Header.Set("Content-Type", "application/x-www-form-urlencoded")
        rec2 := httptest.NewRecorder()
        // ... app.SubscriptionForm(echo.NewContext(req2, rec2))
        assert.Equal(t, http.StatusBadRequest, rec2.Code, "replayed captcha token should be rejected")
    })

    t.Run("PublicSubscription_无captcha_当前行为可接受请求_安全缺口", func(t *testing.T) {
        // 此测试记录当前行为：JSON API 不检查 captcha
        body := map[string]any{
            "email":      "bot@example.com",
            "name":       "Bot",
            "list_uuids": []string{"<public-list-uuid>"},
        }
        jsonBody, _ := json.Marshal(body)
        req := httptest.NewRequest(http.MethodPost, "/api/public/subscription", bytes.NewReader(jsonBody))
        req.Header.Set("Content-Type", "application/json")
        rec := httptest.NewRecorder()

        // ... app.PublicSubscription(echo.NewContext(req, rec))
        // 当前行为：200 OK — 这正是安全缺口
        assert.Equal(t, http.StatusOK, rec.Code, "SECURITY GAP: PublicSubscription accepts requests without captcha")

        // 修复后应改为：
        // assert.Equal(t, http.StatusBadRequest, rec.Code, "PublicSubscription should reject requests without captcha")
    })
}
```

### 7.3 测试：重复订阅

**目的**：验证重复邮箱提交的行为——已 confirmed 的不会被降级，已 unsubscribed 的不会被重新订阅。

```go
// TestDuplicateSubscription 验证重复订阅的状态保护
func TestDuplicateSubscription(t *testing.T) {
    email := "duplicate@example.com"
    publicListUUID := "<public-list-uuid>"

    // 第一步：首次订阅（single-optin public list）
    sub1 := map[string]any{
        "email":      email,
        "name":       "First",
        "list_uuids": []string{publicListUUID},
    }
    jsonBody1, _ := json.Marshal(sub1)
    req1 := httptest.NewRequest(http.MethodPost, "/api/public/subscription", bytes.NewReader(jsonBody1))
    req1.Header.Set("Content-Type", "application/json")
    rec1 := httptest.NewRecorder()
    // ... app.PublicSubscription(echo.NewContext(req1, rec1))
    assert.Equal(t, http.StatusOK, rec1.Code)

    // 验证：subscriber 存在且状态为 confirmed（single optin + preconfirm=false 但状态由SQL ON CONFLICT处理）
    var subStatus string
    db.Get(&subStatus, `
        SELECT sl.status FROM subscriber_lists sl
        JOIN subscribers s ON sl.subscriber_id = s.id
        JOIN lists l ON sl.list_id = l.id
        WHERE s.email = $1 AND l.uuid = $2
    `, email, publicListUUID)
    // 首次订阅：status 应为 unconfirmed（preconfirm=false, single optin list）
    assert.Equal(t, "unconfirmed", subStatus)

    // 第二步：手动将 subscription 确认为 confirmed
    db.Exec(`UPDATE subscriber_lists SET status = 'confirmed' WHERE subscriber_id = (SELECT id FROM subscribers WHERE email = $1)`, email)

    // 第三步：再次提交相同邮箱 + 相同列表
    sub2 := map[string]any{
        "email":      email,
        "name":       "Second Attempt",
        "list_uuids": []string{publicListUUID},
    }
    jsonBody2, _ := json.Marshal(sub2)
    req2 := httptest.NewRequest(http.MethodPost, "/api/public/subscription", bytes.NewReader(jsonBody2))
    req2.Header.Set("Content-Type", "application/json")
    rec2 := httptest.NewRecorder()
    // ... app.PublicSubscription(echo.NewContext(req2, rec2))
    assert.Equal(t, http.StatusOK, rec2.Code)

    // 验证：confirmed 状态不应被降级
    db.Get(&subStatus, `
        SELECT sl.status FROM subscriber_lists sl
        JOIN subscribers s ON sl.subscriber_id = s.id
        JOIN lists l ON sl.list_id = l.id
        WHERE s.email = $1 AND l.uuid = $2
    `, email, publicListUUID)
    assert.Equal(t, "confirmed", subStatus, "confirmed subscription should NOT be downgraded")

    // 第四步：手动将 subscription 设为 unsubscribed
    db.Exec(`UPDATE subscriber_lists SET status = 'unsubscribed' WHERE subscriber_id = (SELECT id FROM subscribers WHERE email = $1)`, email)

    // 第五步：再次提交
    sub3 := map[string]any{
        "email":      email,
        "name":       "Third Attempt",
        "list_uuids": []string{publicListUUID},
    }
    jsonBody3, _ := json.Marshal(sub3)
    req3 := httptest.NewRequest(http.MethodPost, "/api/public/subscription", bytes.NewReader(jsonBody3))
    req3.Header.Set("Content-Type", "application/json")
    rec3 := httptest.NewRecorder()
    // ... app.PublicSubscription(echo.NewContext(req3, rec3))
    assert.Equal(t, http.StatusOK, rec3.Code)

    // 验证：unsubscribed 状态不应被重新订阅
    db.Get(&subStatus, `
        SELECT sl.status FROM subscriber_lists sl
        JOIN subscribers s ON sl.subscriber_id = s.id
        JOIN lists l ON sl.list_id = l.id
        WHERE s.email = $1 AND l.uuid = $2
    `, email, publicListUUID)
    assert.Equal(t, "unsubscribed", subStatus, "unsubscribed should NOT be resubscribed via public form")
}

// TestDuplicateSubscription_BlocklistedEmail 验证 blocklisted 域名始终被拒绝
func TestDuplicateSubscription_BlocklistedEmail(t *testing.T) {
    // 前置条件：域名 "spam.com" 在 blocklist 中
    email := "spammer@spam.com"

    sub := map[string]any{
        "email":      email,
        "name":       "Spammer",
        "list_uuids": []string{"<public-list-uuid>"},
    }
    jsonBody, _ := json.Marshal(sub)
    req := httptest.NewRequest(http.MethodPost, "/api/public/subscription", bytes.NewReader(jsonBody))
    req.Header.Set("Content-Type", "application/json")
    rec := httptest.NewRecorder()

    // ... app.PublicSubscription(echo.NewContext(req, rec))
    assert.Equal(t, http.StatusBadRequest, rec.Code, "blocklisted domain should be rejected")

    // 验证数据库中无此 subscriber
    var count int
    db.Get(&count, `SELECT COUNT(*) FROM subscribers WHERE email = $1`, email)
    assert.Equal(t, 0, count, "no subscriber should be created for blocklisted domain")
}
```

---

## 8. 建议修复

### 8.1 为 PublicSubscription 添加 Captcha 验证

[public.go#PublicSubscription](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L516-L529) 应在调用 `processSubForm` 之前添加与 `SubscriptionForm` 相同的 captcha 验证逻辑。建议提取为公共方法：

```go
func (a *App) verifyCaptcha(c echo.Context) error {
    if a.captcha.IsEnabled() {
        var val string
        switch a.captcha.GetProvider() {
        case captcha.ProviderHCaptcha:
            val = c.FormValue("h-captcha-response")
        case captcha.ProviderAltcha:
            val = c.FormValue("altcha")
        default:
            return echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.invalidCaptcha"))
        }
        if val == "" {
            return echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.invalidCaptcha"))
        }
        err, ok := a.captcha.Verify(val)
        if err != nil {
            a.log.Printf("captcha request failed: %v", err)
        }
        if !ok {
            return echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("public.invalidCaptcha"))
        }
    }
    return nil
}
```

### 8.2 为 PublicSubscription 添加速率限制

在 `/api/public/subscription` 路由上添加基于 IP 的速率限制中间件，防止暴力订阅攻击。

### 8.3 对不存在的 UUID 返回错误

当前 `GetListTypes` 查询到的结果数量可能少于提交的 UUID 数量，但代码没有检查这一差异。建议添加：

```go
if len(listTypes) != len(req.FormListUUIDs) {
    return false, echo.NewHTTPError(http.StatusBadRequest, a.i18n.T("globals.messages.invalidUUID"))
}
```

---

## 9. 完整请求流转图

```
机器人 POST /api/public/subscription
│
├─ 1. EnablePublicSubPage 检查 ──── 关闭 → 400
│
├─ [缺失] Captcha 验证 ──── 安全缺口！此端点无 captcha
│
├─ [缺失] Nonce 蜜罐 ──── 此端点无蜜罐检查
│
└─ processSubForm()
   │
   ├─ 2. 绑定 name/email/list_uuids ──── 格式错误 → 400
   │
   ├─ 3. list_uuids 不能为空 ──── 空 → 400
   │
   ├─ 4. Email 长度 ≤ 1000 ──── 超长 → 400
   │
   ├─ 5. SanitizeEmail()
   │   ├─ RFC 校验 ──── 不合法 → 400
   │   └─ 域名 allowlist/blocklist ──── 命中 → 400
   │
   ├─ 6. Name 长度 ≤ 2000 ──── 超长 → 400
   │
   ├─ 7. GetListTypes() ──── 查询 DB
   │   └─ 含 private list → 400
   │
   └─ 8. InsertSubscriber() ──── DB 写入
       ├─ Email 不存在 → INSERT 成功 → 200
       └─ Email 已存在 → UPDATE（状态保护）
           ├─ confirmed → 保持 confirmed → 200
           ├─ unsubscribed → 保持 unsubscribed → 200
           └─ 其他 → 设为 unconfirmed → 200
```

```
机器人 POST /subscription/form (HTML 表单)
│
├─ 1. EnablePublicSubPage 检查 ──── 关闭 → 404
│
├─ 2. Nonce 蜜罐 ──── 非空 → 502
│
├─ 3. Captcha 验证 ──── 
│   ├─ 未启用 → 跳过
│   └─ 已启用
│       ├─ 空值 → 400
│       ├─ 伪造值 → 400
│       ├─ 重放值 → 400
│       └─ 有效值 → 通过
│
└─ processSubForm() ──── 同上
```
