# 订阅者SQL查询安全机制分析

## 概述

本文档分析 listmonk 系统中高级用户在订阅者页面输入 SQL 表达式进行筛选时的安全机制。系统通过多层防护机制防止 SQL 注入和权限绕过。

---

## 1. 核心调用路径分析

### 1.1 QuerySubscribers 调用路径

**入口点**: [cmd/subscribers.go:98-146](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L98-L146)

```
HTTP 请求
    ↓
App.QuerySubscribers (Handler)
    ├─ 用户权限检查: subscribers:sql_query
    ├─ filterListQueryByPerm() → 过滤列表权限
    ├─ formatSQLExp() → 基础SQL清理
    └─ Core.QuerySubscribers
        ├─ 拼接SQL模板: strings.ReplaceAll(%query%)
        ├─ validateQueryTables() → 表名白名单验证
        ├─ getSubscriberCount() → 只读事务预执行验证
        └─ 只读事务执行最终查询
```

### 1.2 ExportSubscribers 调用路径

**入口点**: [cmd/subscribers.go:148-222](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L148-L222)

```
HTTP 请求
    ↓
App.ExportSubscribers (Handler)
    ├─ 用户权限检查: subscribers:sql_query
    ├─ filterListQueryByPerm() → 过滤列表权限
    ├─ formatSQLExp() → 基础SQL清理
    └─ Core.ExportSubscribers
        ├─ 拼接SQL模板: strings.ReplaceAll(%query%)
        ├─ getSubscriberCount() → 只读事务预执行验证
        └─ 批量导出迭代器执行查询
```

---

## 2. 关键安全函数分析

### 2.1 formatSQLExp - 基础SQL表达式清理

**位置**: [cmd/subscribers.go:825-838](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L825-L838)

**功能**:
- 去除首尾空白字符
- 移除末尾的分号 `;` 防止多语句执行

**代码逻辑**:
```go
func formatSQLExp(q string) string {
    q = strings.TrimSpace(q)
    if len(q) == 0 {
        return ""
    }
    // 移除分号后缀，防止多语句执行
    if q[len(q)-1] == ';' {
        q = q[:len(q)-1]
    }
    return q
}
```

**防护边界**:
- ✅ 防止 `; DROP TABLE subscribers;` 形式的多语句注入
- ❌ 不验证SQL语法
- ❌ 不检查表名或列名

### 2.2 validateQueryTables - 查询表验证

**位置**: [internal/core/subscribers.go:595-623](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L595-L623)

**核心机制**:
1. 创建只读事务
2. 执行 `EXPLAIN (FORMAT JSON) + 查询语句`
3. 解析查询计划JSON提取所有访问的表名
4. 与白名单 `allowedSubQueryTables` 对比

**代码逻辑**:
```go
func validateQueryTables(db *sqlx.DB, query string, allowedTables map[string]struct{}) error {
    // 获取 EXPLAIN (FORMAT JSON) 输出
    tx, _ := db.BeginTxx(context.Background(), &sql.TxOptions{ReadOnly: true})
    defer tx.Rollback()
    
    var plan string
    tx.QueryRow("EXPLAIN (FORMAT JSON) "+query, ...).Scan(&plan)
    
    // 从JSON计划中提取所有关系名
    tables, _ := getTablesFromQueryPlan(plan)
    
    // 验证是否在白名单内
    for _, table := range tables {
        if _, ok := allowedTables[table]; !ok {
            return fmt.Errorf("table '%s' is not allowed", table)
        }
    }
    return nil
}
```

**防护边界**:
- ✅ 检测子查询中访问的表
- ✅ 检测JOIN操作中访问的表
- ✅ 检测CTE(WITH语句)中访问的表
- ❌ 仅在 QuerySubscribers 中调用，**ExportSubscribers 未调用此验证**

### 2.3 getTablesFromQueryPlan - 查询计划表提取

**位置**: [internal/core/subscribers.go:625-663](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L625-L663)

**工作原理**:
- 递归遍历 Postgres EXPLAIN JSON 输出
- 查找所有 "Relation Name" 字段
- 收集所有被访问的表（包括子查询、JOIN、CTE）

**递归遍历逻辑**:
```go
func traverseQueryPlan(node map[string]any, tables map[string]struct{}) {
    if relName, ok := node["Relation Name"].(string); ok {
        tables[relName] = struct{}{}
    }
    // 递归检查嵌套计划（子查询、CTE等）
    for _, v := range node {
        // ... 递归处理 map 和 slice
    }
}
```

### 2.4 allowedSubQueryTables - 允许的表白名单

**位置**: [internal/core/subscribers.go:19-31](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L19-L31)

```go
var allowedSubQueryTables = map[string]struct{}{
    "subscribers":       {},  // 订阅者主表
    "lists":             {},  // 邮件列表表
    "subscribers_lists": {},  // 订阅者-列表关联表
    "campaigns":         {},  // 活动表
    "campaign_lists":    {},  // 活动-列表关联表
    "campaign_views":    {},  // 活动浏览表
    "links":             {},  // 链接表
    "link_clicks":       {},  // 链接点击表
    "bounces":           {},  // 退信表
}
```

**边界说明**:
- ✅ 允许访问与订阅者营销相关的所有业务表
- ❌ 禁止访问系统表（如 `pg_class`, `pg_user`）
- ❌ 禁止访问用户表 `users`、角色表 `roles`
- ❌ 禁止访问设置表、模板表等敏感配置表

---

## 3. 权限控制机制

### 3.1 filterListQueryByPerm - 列表权限过滤

**位置**: [cmd/subscribers.go:808-823](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L808-L823)

**功能**:
- 从查询参数中提取 list_id
- 根据当前用户权限过滤列表ID
- 只返回用户有权访问的列表ID

**代码逻辑**:
```go
func (a *App) filterListQueryByPerm(param string, qp url.Values, user auth.User) ([]int, error) {
    var listIDs []int
    if qp.Has(param) {
        ids, _ := getQueryInts(param, qp)
        listIDs = ids
    }
    // 过滤用户无权访问的列表
    return user.GetPermittedListIDs(listIDs), nil
}
```

**边界说明**:
- ✅ 确保用户只能查询自己有权限的列表中的订阅者
- ✅ 即使SQL查询尝试访问其他列表，`subscriber_lists.list_id = ANY($1)` 会限制结果
- ❌ **注意**: 此权限仅在查询参数 `list_id` 层面生效，如果SQL表达式直接访问其他表，需要依赖表白名单

### 3.2 权限检查层级

| 层级 | 检查点 | 位置 | 说明 |
|------|--------|------|------|
| 1 | 接口权限 | cmd/subscribers.go:111-116 | 检查 `subscribers:sql_query` 权限 |
| 2 | 列表权限 | cmd/subscribers.go:104-107 | 过滤查询参数中的 list_id |
| 3 | 表白名单 | internal/core/subscribers.go:131 | 验证查询访问的表 |
| 4 | 只读事务 | internal/core/subscribers.go:150 | 强制只读执行 |

---

## 4. SQL模板拼接与执行流程

### 4.1 QuerySubscribers SQL模板

**模板位置**: [queries/subscribers.sql:280-298](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L280-L298)

```sql
SELECT subscribers.* FROM subscribers
    LEFT JOIN subscriber_lists
    ON (
        (CASE WHEN CARDINALITY($1::INT[]) > 0 THEN true ELSE false END)
        AND subscriber_lists.subscriber_id = subscribers.id
        AND ($2 = '' OR subscriber_lists.status = $2::subscription_status)
    )
    WHERE (CARDINALITY($1) = 0 OR subscriber_lists.list_id = ANY($1::INT[]))
    AND (CASE WHEN $3 != '' THEN name ~* $3 OR email ~* $3 ELSE TRUE END)
    AND %query%  -- 用户输入的SQL表达式插入此处
    ORDER BY %order% OFFSET $4 LIMIT (CASE WHEN $5 < 1 THEN NULL ELSE $5 END);
```

### 4.2 ExportSubscribers SQL模板

**模板位置**: [queries/subscribers.sql:320-344](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L320-L344)

```sql
SELECT subscribers.id, subscribers.uuid, ... FROM subscribers
    LEFT JOIN subscriber_lists
    ON (
        (CASE WHEN CARDINALITY($1::INT[]) > 0 THEN true ELSE false END)
        AND subscriber_lists.subscriber_id = subscribers.id
        AND ($4 = '' OR subscriber_lists.status = $4::subscription_status)
    )
    WHERE subscriber_lists.list_id = ALL($1::INT[]) AND id > $2
    AND (CASE WHEN CARDINALITY($3::INT[]) > 0 THEN id=ANY($3) ELSE true END)
    AND (CASE WHEN $5 != '' THEN name ~* $5 OR email ~* $5 ELSE TRUE END)
    AND %query%  -- 用户输入的SQL表达式插入此处
    ORDER BY subscribers.id ASC LIMIT (CASE WHEN $6 < 1 THEN NULL ELSE $6 END);
```

### 4.3 拼接执行流程

```
用户输入: "subscribers.email LIKE '%@example.com'"
    ↓
formatSQLExp() 处理
    ↓
字符串替换: strings.ReplaceAll(template, "%query%", cond)
    ↓
[QuerySubscribers 独有]
validateQueryTables() 表白名单验证
    ↓
getSubscriberCount() 只读事务计数验证
    ↓
只读事务执行查询 (QuerySubscribers)
或
Preparex + 执行查询 (ExportSubscribers)
```

---

## 5. 应允许的表达式示例（3个）

### 示例1: 邮箱域名过滤

**表达式**:
```sql
subscribers.email LIKE '%@company.com'
```

**最终拼接SQL**:
```sql
SELECT subscribers.* FROM subscribers
    LEFT JOIN subscriber_lists
    ON (...)
    WHERE ...
    AND subscribers.email LIKE '%@company.com'
    ORDER BY subscribers.id DESC OFFSET 0 LIMIT 10;
```

**允许原因**:
- 只访问 `subscribers` 表（在白名单内）
- 标准的 WHERE 条件，无 JOIN 或子查询
- 不尝试访问敏感表或绕过权限

---

### 示例2: 带属性查询的多条件筛选

**表达式**:
```sql
subscribers.status = 'enabled' AND
subscribers.attribs->>'city' = 'Beijing' AND
(subscribers.attribs->>'projects')::INT > 5
```

**最终拼接SQL**:
```sql
SELECT subscribers.* FROM subscribers
    LEFT JOIN subscriber_lists
    ON (...)
    WHERE ...
    AND subscribers.status = 'enabled' AND
        subscribers.attribs->>'city' = 'Beijing' AND
        (subscribers.attribs->>'projects')::INT > 5
    ORDER BY subscribers.id DESC OFFSET 0 LIMIT 10;
```

**允许原因**:
- 只访问 `subscribers` 表
- 使用 Postgres JSONB 操作符查询属性
- 无额外表访问，符合白名单规则

---

### 示例3: 使用子查询关联活动浏览表

**表达式**:
```sql
EXISTS(SELECT 1 FROM campaign_views 
       WHERE campaign_views.subscriber_id = subscribers.id 
       AND campaign_views.campaign_id = 123)
```

**最终拼接SQL**:
```sql
SELECT subscribers.* FROM subscribers
    LEFT JOIN subscriber_lists
    ON (...)
    WHERE ...
    AND EXISTS(SELECT 1 FROM campaign_views 
               WHERE campaign_views.subscriber_id = subscribers.id 
               AND campaign_views.campaign_id = 123)
    ORDER BY subscribers.id DESC OFFSET 0 LIMIT 10;
```

**允许原因**:
- 访问的 `campaign_views` 表在白名单内
- 子查询用于关联活动浏览数据，属业务正常查询
- EXPLAIN 会检测到 `campaign_views` 表并通过验证

---

## 6. 应拒绝的表达式示例（3个）

### 示例1: 尝试访问用户表获取凭据

**表达式**:
```sql
EXISTS(SELECT 1 FROM users WHERE users.email = subscribers.email)
```

**拒绝原因**:
- `users` 表不在 `allowedSubQueryTables` 白名单中
- EXPLAIN 会检测到 "Relation Name": "users"
- 验证失败返回错误: `table 'users' is not allowed`

**风险**: 如果未被拦截，可能获取系统用户的邮箱和密码哈希等敏感信息

---

### 示例2: 使用JOIN绕过列表权限

**表达式**:
```sql
subscribers.id IN (
    SELECT sl.subscriber_id FROM subscriber_lists sl
    JOIN lists l ON l.id = sl.list_id
    WHERE l.name = 'Restricted List'
)
```

**拒绝原因**:
- 虽然 `subscriber_lists` 和 `lists` 都在白名单内
- **但**: `filterListQueryByPerm` 确保查询参数中的 list_id 已被过滤
- **且**: SQL模板中的 `subscriber_lists.list_id = ANY($1::INT[])` 会强制限制列表
- **注意**: 从表白名单角度这会被**允许**，但列表权限会在参数层面拦截

**实际行为分析**:
- 表验证通过（所有表都在白名单）
- 但 `$1` 只包含用户有权限的列表ID
- 即使表达式尝试访问其他列表，主查询的 JOIN 条件会限制结果

---

### 示例3: 使用ORDER BY进行SQL注入

**恶意表达式**:
```sql
TRUE; SELECT * FROM users;
```

**拒绝原因**:
- `formatSQLExp()` 会移除末尾的分号
- 处理后变为: `TRUE; SELECT * FROM users`
- 但 Postgres 不允许在单个 prepared statement 中执行多语句
- 会导致 SQL 语法错误

**更隐蔽的注入尝试**:
```sql
TRUE ORDER BY (SELECT password FROM users LIMIT 1)
```

**拒绝原因**:
- `users` 表不在白名单中
- EXPLAIN 会检测到子查询访问了 `users` 表
- 验证失败

---

## 7. 安全边界总结

### 7.1 现有防护机制的强项

| 机制 | 防护能力 |
|------|----------|
| formatSQLExp | 防止分号结尾的多语句执行 |
| validateQueryTables | 防止访问白名单外的表（QuerySubscribers） |
| 只读事务 | 防止数据修改操作 |
| filterListQueryByPerm | 限制只能查询有权限的列表 |
| 参数化查询 | 防止经典的SQL注入 |

### 7.2 潜在风险点

⚠️ **重要发现**: `ExportSubscribers` 函数 **未调用** `validateQueryTables()` 进行表验证！

**对比**:
| 函数 | validateQueryTables 调用 |
|------|---------------------------|
| QuerySubscribers | ✅ 有调用 (internal/core/subscribers.go:131) |
| ExportSubscribers | ❌ 无调用 |

**风险影响**:
- ExportSubscribers 仅依赖 `getSubscriberCount()` 的只读事务预执行
- 恶意用户可能通过导出功能访问白名单外的表
- 建议在 ExportSubscribers 中也添加表验证

### 7.3 关于 subscriber_lists 别名的问题

**用户问题**: "使用 subscriber_lists 别名尝试绕过列表权限"

**分析**:
- SQL模板中使用的是 `subscriber_lists` 表（无别名）
- 即使在查询表达式中使用别名如 `sl`，EXPLAIN 仍会检测到真实表名
- 别名不影响表验证机制

**示例**:
```sql
EXISTS(SELECT 1 FROM subscriber_lists sl WHERE sl.subscriber_id = subscribers.id)
```
- EXPLAIN JSON 中 "Relation Name" 仍是 `subscriber_lists`
- 别名不影响检测结果

---

## 8. 代码引用

- [QuerySubscribers Handler](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L98-L146)
- [ExportSubscribers Handler](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L148-L222)
- [Core.QuerySubscribers](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L105-L171)
- [Core.ExportSubscribers](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L231-L281)
- [validateQueryTables](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L595-L623)
- [formatSQLExp](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L825-L838)
- [SQL查询模板](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L280-L344)

---

## 9. 建议改进

1. **在 ExportSubscribers 中添加 validateQueryTables 调用**
   - 当前 QuerySubscribers 有表验证但 ExportSubscribers 没有
   - 保持两个接口的安全一致性

2. **考虑添加列级白名单验证**
   - 当前只验证表名，不验证列名
   - 敏感列如 `password_hash` 仍可能被访问（如果表在白名单内）

3. **增加 SQL 复杂度限制**
   - 限制子查询嵌套深度
   - 限制 JOIN 数量

4. **审计日志**
   - 记录所有使用自定义SQL表达式的查询
   - 便于事后审计和异常检测
