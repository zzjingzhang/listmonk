# 用户管理模块安全与竞态分析

## 场景描述

管理员执行以下操作：
1. 批量删除多个用户
2. 修改另一个用户的角色（例如从 SuperAdmin 降级为普通用户）

与此同时，被修改角色的用户正在使用：
- 旧的 Session Cookie
- 旧的 API Token

持续调用 `/api/settings` 接口（需要 `settings:get` 或 `settings:manage` 权限）。

---

## 一、核心函数执行效果分析

### 1. CreateUser - 创建用户

**调用链路**：
[cmd/users.go:53-104](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L53-L104) → [core/users.go:44-74](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/users.go#L44-L74) → [queries/users.sql:1-13](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L1-L13)

**执行流程**：
1. 参数校验（用户名格式、邮箱格式、密码强度等）
2. 对于 API 用户，自动生成 32 位随机 Token
3. SQL 插入：通过 `CRYPT($3, GEN_SALT('bf'))` 对普通用户密码进行 bcrypt 哈希
4. 创建成功后调用 `cacheUsers()` 更新 API 用户缓存

**关键特性**：
- 普通用户密码自动 bcrypt 哈希存储
- API 用户 Token 明文存储（用于后续快速校验）
- 创建后立即刷新缓存，新 API Token 立即可用

---

### 2. UpdateUser - 更新用户（管理员操作）

**调用链路**：
[cmd/users.go:107-189](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L107-L189) → [core/users.go:77-98](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/users.go#L77-L98) → [queries/users.sql:15-42](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L15-L42)

**执行流程**：
1. 参数校验
2. SQL 更新（包含关键保护逻辑）
3. 若修改了密码，调用 `DeleteUserSessions(id, "")` 销毁该用户所有会话
4. 调用 `cacheUsers()` 更新 API 用户缓存

**SQL 保护分支（关键）**：
```sql
WITH u AS (
    SELECT
        CASE
            WHEN (SELECT COUNT(*) FROM users WHERE id != $1 AND status = 'enabled' AND type = 'user' AND user_role_id = 1) = 0  
                AND ($8 != 1 OR $10 != 'enabled')
            THEN FALSE
            ELSE TRUE
        END AS canEdit
)
UPDATE users SET ... WHERE id=$1 AND (SELECT canEdit FROM u) = TRUE;
```

**保护逻辑解读**：
- 统计除当前用户外，还有多少个启用状态的 SuperAdmin（`user_role_id = 1`）
- 如果数量为 0（即当前用户是最后一个 SuperAdmin），则不允许：
  - 修改角色（`$8 != 1`，即 user_role_id 不再是 1）
  - 禁用用户（`$10 != 'enabled'`）
- 若 `RowsAffected() == 0`，返回错误：`users.needSuper`

---

### 3. UpdateUserProfile - 用户自我更新

**调用链路**：
[cmd/users.go:240-284](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L240-L284) → [core/users.go:101-114](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/users.go#L101-L114) → [queries/users.sql:143-146](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L143-L146)

**执行流程**：
1. 从 `auth.GetUser(c)` 获取当前登录用户（从 Session 或 API Token 中提取）
2. 只允许更新 `name`, `email`, `password` 字段
3. 若修改了密码，调用 `DeleteUserSessions(user.ID, auth.GetSessionID(c))` 销毁其他会话（保留当前会话）
4. **注意**：不会调用 `cacheUsers()` 更新缓存

**关键差异**：
- 与 `UpdateUser` 不同，`UpdateUserProfile` 没有保护最后一个 SuperAdmin 的 SQL 逻辑
- 但它不允许修改 `user_role_id` 和 `status`，只能修改基本信息
- 修改密码时只销毁其他会话，保留当前会话

---

### 4. DeleteUsers - 批量删除用户

**调用链路**：
[cmd/users.go:208-225](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L208-L225) → [core/users.go:146-157](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/users.go#L146-L157) → [queries/users.sql:44-48](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L44-L48)

**执行流程**：
1. 接收待删除 ID 列表
2. SQL 删除（包含关键保护逻辑）
3. 调用 `cacheUsers()` 更新 API 用户缓存

**SQL 保护分支（关键）**：
```sql
WITH u AS (
    SELECT COUNT(*) AS num FROM users 
    WHERE NOT(id = ANY($1)) AND user_role_id=1 AND type='user' AND status='enabled'
)
DELETE FROM users WHERE id = ALL($1) AND (SELECT num FROM u) > 0;
```

**保护逻辑解读**：
- 统计删除操作后，剩余的启用状态 SuperAdmin 数量
- 如果剩余数量为 0（即删除了最后一个 SuperAdmin），则不执行删除
- 若 `RowsAffected() == 0`，返回错误：`users.needSuper`

---

### 5. DeleteUserSessions - 删除用户会话

**调用链路**：
[core/users.go:138-143](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/users.go#L138-L143) → [queries/users.sql:154-155](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L154-L155)

**SQL 逻辑**：
```sql
DELETE FROM sessions 
WHERE data->>'user_id' = $1 AND ($2 = '' OR id != $2);
```

**执行效果**：
- 删除 `sessions` 表中该用户的所有会话记录
- 若 `excludeID` 不为空，则保留指定会话（用于用户改密后保留当前登录状态）
- 直接操作数据库，不涉及缓存

---

### 6. cacheUsers - 缓存 API 用户

**调用链路**：
[cmd/users.go:355-375](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L355-L375) → [auth/auth.go:118-126](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L118-L126)

**执行流程**：
1. 从数据库查询所有用户
2. 筛选出类型为 `api` 且状态为 `enabled` 的用户
3. 调用 `a.CacheAPIUsers(apiUsers)` 更新内存缓存
4. 使用 `sync.RWMutex` 保证并发安全

**缓存结构**：
```go
type Auth struct {
    apiUsers map[string]User  // key: username, value: User (包含 Password token)
    sync.RWMutex
}
```

**API Token 校验流程**（[auth/auth.go:136-146](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L136-L146)）：
1. 从内存缓存 `apiUsers` 中查找用户
2. 使用 `subtle.ConstantTimeCompare` 安全比较 Token
3. 不查询数据库，纯内存操作，性能高

---

## 二、权限检查流程（以 /api/settings 为例）

### Session 认证流程

[auth/auth.go:286-333](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L286-L333) → [auth/auth.go:396-423](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L396-L423)

1. 解析 Cookie 中的 `session_id`
2. 从 `sessions` 表查询会话，获取 `user_id`
3. 调用 `cb.GetUser(user_id)` 从数据库查询用户最新信息（**每次请求都查**）
4. 将用户信息存入 echo context

### API Token 认证流程

[auth/auth.go:302-319](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L302-L319) → [auth/auth.go:136-146](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L136-L146)

1. 解析 `Authorization`  header
2. 从内存缓存 `apiUsers` 中查找用户（**不查数据库**）
3. 安全比较 Token
4. 将用户信息存入 echo context

### 权限中间件检查

[auth/auth.go:336-367](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L336-L367)

```go
func (o *Auth) Perm(next echo.HandlerFunc, perms ...string) echo.HandlerFunc {
    return func(c echo.Context) error {
        u, ok := c.Get(UserHTTPCtxKey).(User)
        // SuperAdmin 直接放行
        if u.UserRole.ID == SuperAdminRoleID {
            return next(c)
        }
        // 检查具体权限
        for _, perm := range perms {
            if _, ok := u.PermissionsMap[perm]; ok {
                return next(c)
            }
        }
        return echo.NewHTTPError(http.StatusForbidden, ...)
    }
}
```

**注意**：`PermissionsMap` 是在 `setupUserFields` 中根据 `user_role_permissions` 构建的，来源于每次 Session 认证时的数据库查询，或 API Token 缓存中的数据。

---

## 三、所有保护分支汇总

### 1. SQL 层面保护（数据库级）

| 保护点 | 位置 | 触发条件 | 保护效果 |
|--------|------|----------|----------|
| 最后一个 SuperAdmin 角色修改保护 | [queries/users.sql:21-22](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L21-L22) | 当前用户是最后一个启用的 SuperAdmin，且尝试修改角色或禁用 | 更新不执行，返回 `users.needSuper` |
| 最后一个 SuperAdmin 删除保护 | [queries/users.sql:44-48](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L44-L48) | 删除后将没有启用的 SuperAdmin | 删除不执行，返回 `users.needSuper` |
| 密码自动 bcrypt 哈希 | [queries/users.sql:3-12](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L3-L12) | 普通用户且启用密码登录 | 密码自动哈希存储 |

### 2. 应用层面保护

| 保护点 | 位置 | 触发条件 | 保护效果 |
|--------|------|----------|----------|
| 用户名格式校验 | [cmd/users.go:64-69](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L64-L69) | 用户名不符合正则或长度要求 | 返回错误 |
| 邮箱格式校验 | [cmd/users.go:71-73](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L71-L73) | 邮箱格式非法 | 返回错误 |
| 密码强度校验 | [cmd/users.go:75-77](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L75-L77) | 密码小于 8 位 | 返回错误 |
| 密码修改后会话失效 | [cmd/users.go:177-181](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L177-L181) | 管理员修改用户密码 | 销毁该用户所有会话 |
| 用户自我改密保留当前会话 | [cmd/users.go:274-278](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L274-L278) | 用户修改自己的密码 | 销毁其他会话，保留当前 |
| API Token 安全比较 | [auth/auth.go:141](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L141) | API Token 校验 | 使用 `ConstantTimeCompare` 防时序攻击 |
| SuperAdmin 权限短路 | [auth/auth.go:345-347](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L345-L347) | 用户角色 ID 为 1 | 跳过所有权限检查 |

### 3. 并发安全保护

| 保护点 | 位置 | 保护效果 |
|--------|------|----------|
| API 缓存读写锁 | [auth/auth.go:119-120, 137, 139](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L119-L120) | 防止并发读写 map 导致 panic |

---

## 四、竞态窗口分析

### 竞态场景时序图

```
管理员操作                         用户请求
   |                                |
   | 1. UPDATE users SET role=2     |
   |    (SQL执行成功)               |
   |                                | 2. 请求 /api/settings (旧Session)
   |                                |    → 从DB查询用户（已更新）
   |                                |    → 权限检查失败 ✗
   |                                |
   | 3. DeleteUserSessions()        |
   |    (删除会话记录)              |
   |                                | 4. 请求 /api/settings (旧Session)
   |                                |    → Session不存在，认证失败 ✗
   |                                |
   | 5. cacheUsers()                |
   |    (更新内存缓存)              |
   |                                | 6. 请求 /api/settings (旧API Token)
   |                                |    → 从缓存查询（已更新）
   |                                |    → 权限检查失败 ✗
```

### 竞态窗口 1：SQL 更新后 → Session 认证用户权限失效

**时间窗口**：[UpdateUser SQL 执行完成] → [DeleteUserSessions 执行完成]

**风险分析**：
- 对于 Session 认证用户：**无风险**。因为 `validateSession` 每次都调用 `cb.GetUser(user_id)` 从数据库查询最新用户信息（[auth/auth.go:417](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L417)）。
- SQL 更新成功后，下一次 Session 认证请求就会拿到新的角色信息，权限检查立即生效。

**竞态窗口大小**：0（Session 认证无竞态）

### 竞态窗口 2：SQL 更新后 → DeleteUserSessions 执行前

**时间窗口**：[UpdateUser SQL 执行完成] → [DeleteUserSessions 执行完成]

**风险分析**：
- 虽然 Session 认证用户的权限已经正确降级（因为每次查 DB），但旧 Session 在数据库中仍然存在
- 用户仍然可以通过认证中间件，只是权限检查会失败
- 这不是权限提升风险，只是会话没有及时清理

**竞态窗口大小**：约几毫秒（HTTP 请求处理时间）

### 竞态窗口 3：SQL 更新后 → cacheUsers 执行前（API Token 认证）

**时间窗口**：[UpdateUser SQL 执行完成] → [cacheUsers 执行完成]

**风险分析**：
- **高风险**：API Token 认证使用内存缓存，不查询数据库
- 在 `cacheUsers()` 执行完成前，旧的用户信息（包括旧角色、旧权限）仍然在缓存中
- 被降级的用户仍然可以使用旧 API Token 通过认证，并且旧的 `PermissionsMap` 仍然包含 `settings:get` 等权限
- 可能导致**权限提升漏洞**：用户被降级后，在缓存刷新前仍能执行需要更高权限的操作

**竞态窗口大小**：约几毫秒到几十毫秒（HTTP 请求处理时间 + cacheUsers 查询时间）

### 竞态窗口 4：批量删除 + 角色修改并发执行

**时间窗口**：两个管理员操作并发执行

**风险分析**：
- 假设管理员 A 删除用户 X、Y、Z
- 管理员 B 同时将最后一个 SuperAdmin 降级
- SQL 保护是每个语句独立检查的
  - DeleteUsers 检查：删除后剩余 SuperAdmin 数量 > 0
  - UpdateUser 检查：如果是最后一个 SuperAdmin，不允许修改角色
- 如果两个操作并发：
  - 情况 1：DeleteUsers 先执行，删除了一些非 SuperAdmin 用户，不影响 UpdateUser 的保护逻辑
  - 情况 2：UpdateUser 先执行，如果是最后一个 SuperAdmin，会被保护阻止
  - 情况 3：两个操作都涉及 SuperAdmin，SQL 的 WITH 子句是在同一个事务内统计的，应该能正确保护

**但存在一个边缘情况**：
如果系统中只有 2 个 SuperAdmin，管理员 A 删除其中一个，管理员 B 同时降级另一个。两个 SQL 语句的 WITH 子句统计时，可能都认为"还有另一个 SuperAdmin"，导致两个操作都成功，最终系统没有 SuperAdmin。

**根本原因**：两个独立的 SQL 语句各自统计时，对方的修改尚未提交。

### 竞态窗口 5：UpdateUserProfile 不刷新缓存

**风险分析**：
- `UpdateUserProfile` 不会调用 `cacheUsers()`（[cmd/users.go:240-284](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L240-L284)）
- 但 `UpdateUserProfile` 只能修改 name、email、password，不能修改角色或状态
- 密码修改会触发 `DeleteUserSessions`，但 API Token 缓存不会更新
- 对于 API 用户，密码就是 Token，如果 API 用户通过 `UpdateUserProfile` 修改密码（Token），缓存不会刷新，导致新 Token 无法使用，旧 Token 仍然有效

**注意**：实际上 API 用户的 `PasswordLogin` 是 false，而 `UpdateUserProfile` 中密码修改的条件是 `u.PasswordLogin && u.Password.String != ""`，所以 API 用户无法通过此接口修改 Token。

---

## 五、场景具体分析：降级 + 旧 Session/Token 调用 settings

### 情况 1：使用旧 Session 调用

**时序**：
1. 管理员执行 `UpdateUser` 将用户 A 从 SuperAdmin 降级为普通用户
2. SQL 执行成功，`user_role_id` 从 1 变为 2
3. 用户 A 使用旧 Session 调用 `GET /api/settings`
4. `validateSession` → `cb.GetUser(user_id)` 从 DB 查询，得到新角色（普通用户）
5. `Perm` 中间件检查：`u.UserRole.ID == 1` 为 false，检查 `PermissionsMap["settings:get"]`
6. 如果新角色没有该权限，返回 403 Forbidden ✗

**结论**：Session 认证**无竞态风险**，权限立即失效。

### 情况 2：使用旧 API Token 调用

**时序**：
1. 管理员执行 `UpdateUser` 将用户 A 从 SuperAdmin 降级为普通用户
2. SQL 执行成功，`user_role_id` 从 1 变为 2
3. 执行 `DeleteUserSessions`（不影响 API Token）
4. **在 cacheUsers 执行前**，用户 A 使用旧 API Token 调用 `GET /api/settings`
5. `GetAPIToken` 从内存缓存查询，得到旧的用户信息（`UserRole.ID = 1`）
6. `Perm` 中间件检查：`u.UserRole.ID == 1` 为 true，直接放行 ✓
7. **权限提升成功**

**结论**：API Token 认证**存在竞态风险**，在 `cacheUsers` 执行前，旧权限仍然有效。

### 情况 3：self profile 更新

**分析**：
- 用户调用 `UpdateUserProfile` 更新自己的信息
- 只能修改 name、email、password
- 不能修改角色或状态，所以不存在自我降级的问题
- 但如果管理员在用户调用 `UpdateUserProfile` 的同时修改了该用户的角色：
  - `UpdateUserProfile` 使用 `auth.GetUser(c)` 获取的用户 ID 进行更新
  - 即使角色已被修改，`UpdateUserProfile` 仍然可以成功更新 name/email
  - 因为 `update-user-profile` SQL 不涉及角色字段

---

## 六、问题总结与改进建议

### 已有的保护措施 ✅

1. ✅ SQL 层面保护最后一个 SuperAdmin 不被删除或降级
2. ✅ Session 认证每次从 DB 查询最新用户信息，权限变更立即生效
3. ✅ 密码修改后及时销毁会话
4. ✅ API Token 使用 `ConstantTimeCompare` 安全比较
5. ✅ 缓存使用 `sync.RWMutex` 保证并发安全

### 存在的风险 ❌

1. **API Token 缓存竞态**：角色变更后到 `cacheUsers` 执行前，旧 API Token 仍然具有旧权限
2. **并发删除+降级风险**：两个并发操作可能绕过"至少一个 SuperAdmin"的保护
3. **DeleteUserSessions 只清会话不清 Token**：角色变更后 API Token 不会被吊销

### 改进建议

#### 1. 缩小 API Token 竞态窗口

**问题代码**（[cmd/users.go:168-186](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L168-L186)）：
```go
// 1. SQL 更新
user, err := a.core.UpdateUser(id, u)
// 2. 清会话（如果改密）
if u.Password.String != "" {
    if err := a.core.DeleteUserSessions(id, ""); err != nil { ... }
}
// 3. 刷新缓存
if _, err := cacheUsers(a.core, a.auth); err != nil { ... }
```

**建议**：将 `cacheUsers` 提到 `DeleteUserSessions` 之前执行，或者在 SQL 更新后立即执行：

```go
user, err := a.core.UpdateUser(id, u)
// 先刷新缓存，再清会话，缩小竞态窗口
if _, err := cacheUsers(a.core, a.auth); err != nil { ... }
if u.Password.String != "" {
    if err := a.core.DeleteUserSessions(id, ""); err != nil { ... }
}
```

#### 2. 添加 API Token 吊销机制

在 `UpdateUser` 和 `DeleteUsers` 时，除了刷新缓存，还应该考虑：
- 如果用户被禁用或角色变更，记录一个 Token 版本号或失效时间
- 或者简单地为 API 用户生成新的 Token（使旧 Token 失效）

#### 3. 加强"最后一个 SuperAdmin"保护的事务性

对于并发删除+降级的场景，应该使用数据库事务或行级锁：
```sql
-- 使用 FOR UPDATE 锁定相关行，防止并发修改
WITH u AS (
    SELECT COUNT(*) AS num FROM users 
    WHERE NOT(id = ANY($1)) AND user_role_id=1 AND type='user' AND status='enabled'
    FOR UPDATE  -- 锁定统计的行
)
DELETE FROM users WHERE id = ALL($1) AND (SELECT num FROM u) > 0;
```

#### 4. API Token 认证增加 DB 回查

对于敏感操作（如 settings 修改），可以考虑偶尔回查数据库确认用户状态，而不是完全信任缓存。

---

## 七、关键代码索引

| 功能 | 文件位置 |
|------|---------|
| 用户创建 | [cmd/users.go:53-104](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L53-L104) |
| 用户更新（管理员） | [cmd/users.go:107-189](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L107-L189) |
| 用户更新（自我） | [cmd/users.go:240-284](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L240-L284) |
| 用户删除 | [cmd/users.go:208-225](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L208-L225) |
| 删除会话 | [core/users.go:138-143](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/users.go#L138-L143) |
| 缓存 API 用户 | [cmd/users.go:355-375](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/users.go#L355-L375) |
| 最后 SuperAdmin 更新保护 | [queries/users.sql:15-42](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L15-L42) |
| 最后 SuperAdmin 删除保护 | [queries/users.sql:44-48](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/users.sql#L44-L48) |
| 认证中间件 | [auth/auth.go:286-333](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L286-L333) |
| 权限中间件 | [auth/auth.go:336-367](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L336-L367) |
| API Token 校验 | [auth/auth.go:136-146](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L136-L146) |
| Session 校验 | [auth/auth.go:396-423](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/auth.go#L396-L423) |
