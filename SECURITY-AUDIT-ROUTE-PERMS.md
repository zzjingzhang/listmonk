# Listmonk 路由权限安全审计报告

> 审计日期：2026-05-31  
> 审计范围：`initHTTPServer` → `initHTTPHandlers` 注册的全部路由，涵盖 admin / api / public / webhook / static 五大分组  
> 审计目标：判断低权限用户是否能通过管理端路由访问不该看的资源

---

## 一、路由分组与中间件架构总览

### 1.1 入口函数

[initHTTPServer](cmd/init.go#L915) 创建 Echo 实例后，注册了全局 App 注入中间件，然后调用 [initHTTPHandlers](cmd/handlers.go#L33) 注册所有业务路由。

### 1.2 四大路由分组

| 分组 | 中间件链 | 认证方式 | 未认证行为 |
|------|---------|---------|-----------|
| **Admin 页面组** | `auth.Middleware` → 登录重定向 | Session Cookie | 302 重定向到 `/admin/login?next=...` |
| **API 组** (`/api/*`) | `auth.Middleware` → JSON 403 | Session Cookie / BasicAuth / token | 返回 JSON 403 |
| **Public 组** | **无** | 无 | 完全开放 |
| **Static 组** | **无** | 无 | 完全开放 |

### 1.3 中间件层级

```
请求 → CORS中间件(可选) → App注入中间件
     → auth.Middleware          ← 解析 Authorization 头或 Session Cookie，将 User/HTTPError 写入 echo context
     → auth-redirect/JSON中间件  ← 若 context 中是 HTTPError，Admin组则302重定向，API组则返回JSON
     → hasID/hasUUID/hasSub     ← 参数校验包装器（非权限校验）
     → auth.Perm(perms...)      ← 细粒度权限检查
     → Handler 函数              ← 可能包含额外 list 级权限检查
     → Core 层 SQL              ← 通过参数传递 permittedListIDs 做 SQL 级过滤
```

---

## 二、权限系统组件详解

### 2.1 `auth.Middleware`（[auth.go#L286](internal/auth/auth.go#L286)）

- 优先检查 `Authorization` 头（支持 `token key:secret` 和 `Basic base64`）
- 若 Cookie 中含 `session=` 则忽略 Authorization 头（向后兼容 v3→v4 升级）
- 无 Authorization 头时验证 Session Cookie
- **成功**：将 `auth.User` 写入 `UserHTTPCtxKey`
- **失败**：将 `echo.HTTPError(403)` 写入 `UserHTTPCtxKey`，**不中断请求**

> ⚠️ 关键设计：`auth.Middleware` 永远调用 `next(c)`，不主动拒绝请求。拒绝逻辑由后续中间件/包装器完成。

### 2.2 `auth.Perm`（[auth.go#L336](internal/auth/auth.go#L336)）

- 从 context 取出 `User` 对象
- 若 `UserRoleID == SuperAdminRoleID(1)` 直接放行
- 检查用户 `PermissionsMap` 是否包含传入的任一权限字符串
- 不满足则返回 `403 permission denied`

### 2.3 `hasID` / `hasUUID` / `hasSub`（[handlers.go#L359-L403](cmd/handlers.go#L359)）

- `hasID`：解析 `:id` 参数为 int，若 < 1 返回 400
- `hasUUID`：校验 UUID 格式，不匹配返回 400
- `hasSub`：查库验证 subscriber 是否存在
- **这些仅为参数校验，不涉及权限判断**

### 2.4 `permissions.json`（[permissions.json](permissions.json)）

定义了 8 个权限组共 23 个权限字符串，是 `auth.Perm` 检查的依据。前端 UI 通过 `/api/config` 获取此文件来控制菜单/按钮的可见性。

### 2.5 List 级权限（[models.go#L161-L312](internal/auth/models.go#L161)）

| 方法 | 用途 |
|------|------|
| `User.HasPerm(perm)` | 检查用户是否有某权限（SuperAdmin 放行） |
| `User.HasListPerm(types, listIDs...)` | 检查用户对指定 list 的 get/manage 权限 |
| `User.GetPermittedLists(types)` | 返回 `(hasAllPerm, permittedListIDs)` |
| `User.FilterListsByPerm(types, listIDs)` | 从给定 listIDs 中过滤出用户有权限的 |
| `User.GetPermittedListIDs(listIDs)` | 先过滤，若结果为空则回退到用户全部允许 list |

---

## 三、五类代表接口的完整请求流追踪

### 3.1 Campaign — `GET /api/campaigns/:id`

```
请求 GET /api/campaigns/42
  │
  ├─ auth.Middleware: 解析 token/session → 设置 User 到 context
  ├─ auth-redirect 中间件: User 有效 → 放行
  ├─ auth.Perm("campaigns:get_all", "campaigns:get"): 检查用户是否有其中任一权限
  ├─ hasID: 解析 :id=42, 写入 context
  │
  ▼ 进入 GetCampaign handler (campaigns.go#L112)
  │
  ├─ getID(c) → 42
  ├─ checkCampaignPerm(PermTypeGet, 42, c)
  │   ├─ user.HasPerm(PermCampaignsGetAll)? → YES → return nil
  │   └─ NO → user.HasPerm(PermCampaignsManageAll)? → YES → return nil
  │       └─ NO → user.GetPermittedLists(Get|Manage)
  │           └─ hasAllPerm? → YES → return nil
  │               └─ NO → core.CampaignHasLists(42, permittedListIDs)
  │                   └─ SQL: SELECT EXISTS(SELECT 1 FROM campaign_lists 
  │                           WHERE campaign_id=42 AND list_id = ANY($permittedIDs))
  │
  ├─ core.GetCampaign(42, "", "") → 查库获取 campaign
  └─ 返回 JSON
```

**关键点**：`checkCampaignPerm` 在 handler 内部做了二次 list 级权限校验，即使 `auth.Perm` 放行了 `campaigns:get`，如果 campaign 关联的 list 不在用户允许列表中，仍会被拒绝。

### 3.2 Subscriber — `GET /api/subscribers/:id`

```
请求 GET /api/subscribers/7
  │
  ├─ auth.Middleware → 设置 User
  ├─ auth-redirect → 放行
  ├─ auth.Perm("subscribers:get_all", "subscribers:get")
  ├─ hasID → :id=7
  │
  ▼ 进入 GetSubscriber handler (subscribers.go#L59)
  │
  ├─ hasSubPerm(user, [7])
  │   ├─ user.GetPermittedLists(Get|Manage)
  │   │   → hasAllPerm=false, listIDs=[1,3,5]
  │   ├─ core.HasSubscriberLists([7], [1,3,5])
  │   │   → SQL: 检查 subscriber 7 是否关联 list 1/3/5 中的任意一个
  │   └─ 若不关联 → 403 permission denied
  │
  ├─ core.GetSubscriber(7, "", "")
  ├─ maskRestrictedSubLists(user, &out)
  │   └─ 将用户无权访问的 list 名称替换为 "*Unknown"
  └─ 返回 JSON
```

### 3.3 List — `GET /api/lists/:id`

```
请求 GET /api/lists/5
  │
  ├─ auth.Middleware → 设置 User
  ├─ auth-redirect → 放行
  ├─ ⚠️ 无 auth.Perm 包装！仅 hasID
  │
  ▼ 进入 GetList handler (lists.go#L75)
  │
  ├─ user.HasListPerm(PermTypeGet, 5)
  │   ├─ UserRoleID == 1? → 放行
  │   ├─ HasPerm("lists:get_all")? → 放行
  │   └─ ListPermissionsMap[5]["list:get"] 存在? → 放行
  │       └─ 否 → 403
  │
  ├─ core.GetList(5, "")
  └─ 返回 JSON
```

**关键点**：`GET /api/lists/:id` 路由层未绑定 `auth.Perm`，但 handler 内部调用了 `user.HasListPerm` 做了 list 级权限检查。**权限校验不在中间件层而在 handler 层**。

### 3.4 Media — `GET /api/media/:id`

```
请求 GET /api/media/10
  │
  ├─ auth.Middleware → 设置 User
  ├─ auth-redirect → 放行
  ├─ auth.Perm("media:get")
  ├─ hasID → :id=10
  │
  ▼ 进入 GetMedia handler (media.go#L166)
  │
  ├─ core.GetMedia(10, "", "", mediaStore)
  │   → SQL: SELECT ... FROM media WHERE id=10
  └─ 返回 JSON
```

**关键点**：Media 无 list 级权限概念，仅有 `media:get` / `media:manage` 两个粗粒度权限。任何有 `media:get` 的用户可查看任意 media 记录。

### 3.5 Settings — `GET /api/settings`

```
请求 GET /api/settings
  │
  ├─ auth.Middleware → 设置 User
  ├─ auth-redirect → 放行
  ├─ auth.Perm("settings:get")
  │
  ▼ 进入 GetSettings handler (settings.go#L58)
  │
  ├─ core.GetSettings()
  │   → SQL: SELECT settings FROM settings
  ├─ 脱敏处理：密码字段用 "•" 替换
  └─ 返回 JSON（已脱敏）
```

**关键点**：Settings 无 list 级权限，是全局性的。拥有 `settings:get` 的用户可看到所有配置（含脱敏后的敏感字段）。

---

## 四、权限漏洞与风险分析

### 🔴 漏洞 1：List 路由缺少 `auth.Perm` 中间件绑定

**位置**：[handlers.go#L154-L159](cmd/handlers.go#L154)

```go
g.GET("/api/lists", a.GetLists)                    // ❌ 无 pm 包装
g.GET("/api/lists/:id", hasID(a.GetList))          // ❌ 无 pm 包装
g.PUT("/api/lists/:id", hasID(a.UpdateList))       // ❌ 无 pm 包装
g.DELETE("/api/lists", a.DeleteLists)              // ❌ 无 pm 包装
g.DELETE("/api/lists/:id", hasID(a.DeleteList))    // ❌ 无 pm 包装
```

对比 `POST /api/lists`：
```go
g.POST("/api/lists", pm(a.CreateList, "lists:manage_all"))  // ✅ 有 pm 包装
```

**风险分析**：

- 虽然各 handler 内部调用了 `user.HasListPerm()` / `user.GetPermittedLists()` 进行了 list 级权限检查，但**路由层缺少 `auth.Perm` 意味着任何已认证用户（包括没有任何 list 相关权限的用户）都能进入 handler**。
- `GET /api/lists` 的 handler 调用 `user.GetPermittedLists(PermTypeGet)`，若用户既没有 `lists:get_all` 也没有任何 list 级权限，`permittedIDs` 为空，SQL `id = ANY('{}')` 会返回空集。**行为上不会泄露数据，但绕过了 `auth.Perm` 的统一权限入口，违反了纵深防御原则**。
- `GET /api/lists/:id` 的 handler 调用 `user.HasListPerm(PermTypeGet, id)`，如果用户无权限会返回 403。**功能上安全**，但错误响应格式与 `auth.Perm` 不一致（`auth.Perm` 返回 `permission denied: xxx`，`HasListPerm` 返回 `ErrPermDenied`）。
- 最严重的场景：若未来有人修改 handler 忘记加 list 级权限检查，由于路由层无 `auth.Perm` 兜底，将直接暴露数据。

### 🟡 漏洞 2：2FA TOTP 端点缺少 `auth.Perm` 绑定

**位置**：[handlers.go#L210-L212](cmd/handlers.go#L210)

```go
g.GET("/api/users/:id/twofa/totp", hasID(a.GenerateTOTPQR))   // ❌ 无 pm
g.PUT("/api/users/:id/twofa", hasID(a.EnableTOTP))            // ❌ 无 pm
g.DELETE("/api/users/:id/twofa", hasID(a.DisableTOTP))        // ❌ 无 pm
```

**风险分析**：

- [GenerateTOTPQR](cmd/auth.go#L755) 直接 `c.Get(auth.UserHTTPCtxKey).(auth.User)` 取当前用户，但**未验证 URL 中的 `:id` 是否与当前登录用户 ID 一致**。低权限用户 A 可请求 `/api/users/1/twofa/totp`（假设 1 是管理员 ID），触发为管理员生成 TOTP 密钥。
- `EnableTOTP` / `DisableTOTP` 同理，可能被用于为其他用户启用/禁用 2FA。
- 不过 `GenerateTOTPQR` 内部检查了 `u.TwofaType`，且 TOTP key 保存需要与用户绑定，实际利用难度中等。

### 🟡 漏洞 3：Dashboard / Config / About / Health 端点无权限保护

**位置**：[handlers.go#L102-L116](cmd/handlers.go#L102)

```go
g.GET("/api/health", a.HealthCheck)            // ❌ 无 pm
g.GET("/api/config", a.GetServerConfig)        // ❌ 无 pm — 泄露 permissions.json 全文
g.GET("/api/lang/:lang", a.GetI18nLang)        // ❌ 无 pm
g.GET("/api/dashboard/charts", a.GetDashboardCharts)  // ❌ 无 pm — 全局统计数据
g.GET("/api/dashboard/counts", a.GetDashboardCounts)  // ❌ 无 pm — 全局计数
g.GET("/api/about", a.GetAboutInfo)            // ❌ 无 pm — 系统信息
```

**风险分析**：

- `GetServerConfig` 返回了 `PermissionsRaw`（permissions.json 全文）、`HasLegacyUser`、`Version` 等信息。虽然不直接暴露业务数据，但为攻击者提供了权限结构和版本信息。
- `GetDashboardCharts` / `GetDashboardCounts` 返回**全局**统计，没有按用户 list 权限过滤。低权限用户可以看到全平台的订阅者趋势、活动总数等。
- `GetAboutInfo` 返回数据库版本、操作系统、主机名等系统指纹。

### 🟡 漏洞 4：Profile 端点无权限校验

**位置**：[handlers.go#L199-L200](cmd/handlers.go#L199)

```go
g.GET("/api/profile", a.GetUserProfile)      // ❌ 无 pm
g.PUT("/api/profile", a.UpdateUserProfile)   // ❌ 无 pm
```

**风险分析**：这两个端点仅要求认证（在 auth.Middleware 组内），不检查任何权限。这是合理设计（用户应能查看/修改自己的 profile），但需确保 handler 内部做了"只能操作自己"的校验。

### 🟢 已正确保护的路由模式

以下路由采用了 **`auth.Perm` + handler 内 list 级权限** 的双重保护：

| 模块 | 路由 | Perm | Handler 内 list 校验 |
|------|------|------|---------------------|
| Campaigns | `GET /api/campaigns` | `campaigns:get_all` / `campaigns:get` | ✅ `GetPermittedLists` → SQL 过滤 |
| Campaigns | `GET /api/campaigns/:id` | 同上 | ✅ `checkCampaignPerm` |
| Subscribers | `GET /api/subscribers` | `subscribers:get_all` / `subscribers:get` | ✅ `filterListQueryByPerm` |
| Subscribers | `GET /api/subscribers/:id` | 同上 | ✅ `hasSubPerm` + `maskRestrictedSubLists` |
| Media | `GET /api/media` | `media:get` | N/A（无 list 概念） |
| Settings | `GET /api/settings` | `settings:get` | N/A（全局配置） |

---

## 五、Core 层 SQL 权限过滤机制

### 5.1 List 过滤参数传递模式

Core 层 SQL 查询通过两个参数实现 list 级权限过滤：

- `getAll bool`：若为 `true`，SQL 中 `WHEN $n = TRUE THEN TRUE` 跳过 list 过滤
- `permittedIDs []int`：若 `getAll=false`，SQL 中 `id = ANY($m::INT[])` 限制只返回允许的 list

### 5.2 SQL 示例（[lists.sql](queries/lists.sql)）

```sql
-- get-lists
SELECT * FROM lists WHERE ...
    AND CASE
        WHEN $4 = TRUE THEN TRUE    -- getAll=true: 不过滤
        ELSE id = ANY($5::INT[])    -- getAll=false: 只返回 permittedIDs
    END
```

```sql
-- query-campaigns
AND (
    $5 OR EXISTS (                  -- $5=getAll
        SELECT 1 FROM campaign_lists 
        WHERE campaign_id = c.id AND list_id = ANY($6::INT[])
    )
)
```

### 5.3 Subscriber 查询的 list 过滤

[QuerySubscribers](internal/core/subscribers.go#L106) 接收 `listIDs []int`，通过 `pq.Array(listIDs)` 传入 SQL，在 `WHERE` 条件中限制只查询属于这些 list 的 subscriber。

---

## 六、Public 路由误暴露分析

### 6.1 Public 分组中的敏感端点

| 路由 | 认证 | 风险 |
|------|------|------|
| `POST /webhooks/service/:service` | ❌ 无 | 公网可触发 bounce webhook，依赖 service 自身验证 |
| `GET /api/public/lists` | ❌ 无 | 仅返回 public 类型 list，安全 |
| `POST /api/public/subscription` | ❌ 无 | 有 captcha 保护 + 仅允许非 private list，安全 |
| `GET /api/public/archive` | ❌ 无 | 受 `EnablePublicArchive` 配置控制 |
| `GET /subscription/:campUUID/:subUUID` | ❌ 无 | UUID 猜测难度高，但 URL 模式可枚举 |
| `GET /health` | ❌ 无 | 仅返回 `{"data":true}`，安全 |

### 6.2 Static 文件暴露

```go
srv.GET("/public/static/*", echo.WrapHandler(fSrv))   // 无认证
srv.GET("/admin/static/*", echo.WrapHandler(fSrv))    // 无认证
```

Admin 前端静态资源（JS/CSS）无需认证即可访问，这是 SPA 架构的正常设计——数据由 API 返回而非静态文件。

### 6.3 S3/Filesystem Media 暴露

```go
// filesystem 模式
srv.Static(uploadFsURI, ko.String("upload.filesystem.upload_path"))

// S3 模式
srv.GET(path.Join(publicURL, "/:filepath"), app.ServeS3Media)
```

**两种模式的媒体文件 URL 均可通过 API 的 `media:get` 权限获得，但直接访问文件 URL 不受权限保护**。知道文件路径的任何人都可以直接访问上传的媒体文件。

---

## 七、最小修复策略

### 修复 1：为 List 路由补全 `auth.Perm` 中间件（🔴 高优先级）

**文件**：[handlers.go](cmd/handlers.go)

```go
// 修复前
g.GET("/api/lists", a.GetLists)
g.GET("/api/lists/:id", hasID(a.GetList))
g.PUT("/api/lists/:id", hasID(a.UpdateList))
g.DELETE("/api/lists", a.DeleteLists)
g.DELETE("/api/lists/:id", hasID(a.DeleteList))

// 修复后
g.GET("/api/lists", pm(a.GetLists, "lists:get_all", "list:get"))
g.GET("/api/lists/:id", pm(hasID(a.GetList), "lists:get_all", "list:get"))
g.PUT("/api/lists/:id", pm(hasID(a.UpdateList), "lists:manage_all", "list:manage"))
g.DELETE("/api/lists", pm(a.DeleteLists, "lists:manage_all", "list:manage"))
g.DELETE("/api/lists/:id", pm(hasID(a.DeleteList), "lists:manage_all", "list:manage"))
```

**理由**：即使 handler 内已有 list 级检查，路由层的 `auth.Perm` 提供纵深防御。注意 `list:get` / `list:manage` 是 list 级权限（非 `lists:get_all`），`auth.Perm` 会检查用户是否拥有 `lists:get_all` 或 `list:get` 中的任一权限，与现有 handler 内逻辑一致。

### 修复 2：2FA 端点增加用户身份校验（🟡 中优先级）

**文件**：[handlers.go](cmd/handlers.go) + [auth.go](cmd/auth.go)

方案 A：在路由层增加 `auth.Perm` 保护：

```go
g.GET("/api/users/:id/twofa/totp", pm(hasID(a.GenerateTOTPQR), "users:manage"))
g.PUT("/api/users/:id/twofa", pm(hasID(a.EnableTOTP), "users:manage"))
g.DELETE("/api/users/:id/twofa", pm(hasID(a.DisableTOTP), "users:manage"))
```

方案 B（推荐）：在 handler 内校验 `:id` 是否为当前用户：

```go
func (a *App) GenerateTOTPQR(c echo.Context) error {
    u := c.Get(auth.UserHTTPCtxKey).(auth.User)
    id := getID(c)
    if u.ID != id && u.UserRoleID != auth.SuperAdminRoleID {
        return echo.NewHTTPError(http.StatusForbidden, "cannot modify another user's 2FA")
    }
    // ... 原有逻辑
}
```

**理由**：方案 B 允许用户管理自己的 2FA（无需 `users:manage` 权限），同时阻止操作他人账户。

### 修复 3：Dashboard / Config 端点增加最低权限检查（🟡 中优先级）

**文件**：[handlers.go](cmd/handlers.go)

```go
// 修复前
g.GET("/api/dashboard/charts", a.GetDashboardCharts)
g.GET("/api/dashboard/counts", a.GetDashboardCounts)
g.GET("/api/config", a.GetServerConfig)
g.GET("/api/about", a.GetAboutInfo)

// 修复后 — 至少要求用户拥有某项权限
g.GET("/api/dashboard/charts", pm(a.GetDashboardCharts, "subscribers:get", "campaigns:get", "lists:get_all"))
g.GET("/api/dashboard/counts", pm(a.GetDashboardCounts, "subscribers:get", "campaigns:get", "lists:get_all"))
g.GET("/api/config", a.GetServerConfig)       // 保留无权限（前端登录页需要）
g.GET("/api/about", pm(a.GetAboutInfo, "settings:get"))
```

**理由**：`/api/config` 是前端初始化必需（登录页需要获取 OIDC 配置等），不建议加权限。Dashboard 和 About 应限制为有实际业务权限的用户。

### 修复 4：Dashboard 数据按 list 权限过滤（🟢 低优先级）

**文件**：[admin.go](cmd/admin.go) / [core/dashboard.go](internal/core/dashboard.go)

当前 `GetDashboardCharts` / `GetDashboardCounts` 返回全局统计数据，未按用户 list 权限过滤。建议：

1. 在 handler 中获取用户的 `permittedListIDs`
2. 传递给 core 层查询
3. SQL 中增加 `WHERE list_id = ANY($permittedIDs)` 过滤

**影响**：需要修改 SQL 查询和 Core 接口签名，改动范围较大，建议作为后续迭代。

---

## 八、权限体系关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    permissions.json (23个权限)                     │
│  lists:get_all | lists:manage_all | list:get | list:manage      │
│  subscribers:get | subscribers:get_all | subscribers:manage ... │
│  campaigns:get | campaigns:get_all | campaigns:manage_all ...   │
│  media:get | media:manage | settings:get | settings:manage ...  │
└──────────────┬──────────────────────────────────────────────────┘
               │ 加载到 Config.Permissions map
               │ 通过 /api/config 返回给前端控制UI
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    用户角色 (user_roles 表)                       │
│  SuperAdmin (ID=1) → 全部权限自动放行                            │
│  自定义角色 → 包含 permissions.json 中的权限子集                   │
└──────────────┬──────────────────────────────────────────────────┘
               │ 用户登录后 PermissionsMap 从角色继承
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                    List 角色 (list_roles 表)                      │
│  定义用户对特定 list 的 get/manage 权限                           │
│  User.ListPermissionsMap: map[listID] → {list:get, list:manage} │
│  User.GetListIDs / ManageListIDs: 缓存的允许 list ID 列表        │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│               auth.Perm(perms...) 中间件                         │
│  检查 User.PermissionsMap 中是否有 perms 中任一权限              │
│  SuperAdmin 自动放行                                            │
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│              Handler 内 list 级权限检查                           │
│  user.HasListPerm(types, listIDs)                               │
│  user.GetPermittedLists(types) → (hasAll, permittedIDs)         │
│  user.FilterListsByPerm(types, listIDs) → filteredIDs           │
│  a.hasSubPerm(user, subIDs) → 检查 subscriber 是否关联允许 list │
│  a.checkCampaignPerm(types, campID, c) → 检查 campaign 关联 list│
└──────────────┬──────────────────────────────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────────────────────────────┐
│              Core 层 SQL 过滤                                    │
│  getAll bool + permittedIDs []int → 传入 SQL 参数                 │
│  SQL: CASE WHEN $getAll THEN TRUE ELSE id = ANY($permittedIDs)  │
│  Campaigns: EXISTS(SELECT 1 FROM campaign_lists WHERE ...)      │
│  Subscribers: list_id = ANY($permittedIDs)                      │
│  Lists: id = ANY($permittedIDs)                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、总结

| 类别 | 问题 | 严重程度 | 修复难度 |
|------|------|---------|---------|
| List 路由缺 `auth.Perm` | 纵深防御缺失，依赖 handler 内检查 | 🔴 高 | 低（加 pm 包装） |
| 2FA 端点无用户身份校验 | 可操作他人 2FA 设置 | 🟡 中 | 低（加 ID 比对） |
| Dashboard 无权限保护 | 泄露全局统计数据 | 🟡 中 | 低（加 pm 包装） |
| About 无权限保护 | 泄露系统指纹 | 🟡 低 | 低（加 pm 包装） |
| Media 文件 URL 直接访问 | 知路径即可访问媒体 | 🟢 低 | 中（需签名 URL） |
| Dashboard 数据未按 list 过滤 | 低权限用户看到全平台数据 | 🟢 低 | 高（改 SQL） |

**核心结论**：当前系统的权限体系设计是合理的——`auth.Perm` 做粗粒度权限过滤，handler 内做 list 级细粒度校验，SQL 层做最终兜底。**主要风险不在于权限模型本身，而在于部分路由未接入 `auth.Perm` 中间件，导致纵深防御缺失**。低权限用户目前已认证即可进入 List/2FA/Dashboard 等端点的 handler，虽然 handler 内部有 list 级检查阻止了大部分数据泄露，但违反了最小权限原则。

**优先修复建议**：先完成修复 1（List 路由加 `auth.Perm`），再完成修复 2（2FA 身份校验），最后视安全策略决定是否修复 3 和 4。
