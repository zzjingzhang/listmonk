# Listmonk 数据新鲜度分析报告

## 问题描述

大批量导入和删除订阅者后，`dashboard counts`、`list subscriber stats` 和 `campaign analytics` 在几分钟内显示不一致。管理员运行 maintenance 的 GC 和 vacuum 后仍未立即恢复。

---

## 1. 物化视图架构概览

Listmonk 使用 PostgreSQL 物化视图（Materialized Views）缓存昂贵的聚合查询。共有三个核心物化视图：

| 物化视图名称 | 用途 | 定义位置 |
|-------------|------|---------|
| `mat_dashboard_counts` | Dashboard 统计数据（订阅者总数、黑名单、孤儿、列表数、活动数等） | [schema.sql#L365-L396](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L365-L396) |
| `mat_dashboard_charts` | Dashboard 图表数据（近30天链接点击、邮件查看趋势） | [schema.sql#L400-L433](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L400-L433) |
| `mat_list_subscriber_stats` | 列表订阅者统计（每个列表各状态的订阅者数量） | [schema.sql#L436-L443](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L436-L443) |

### 核心常量定义

在 [core.go#L26-L29](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go#L26-L29) 中定义：

```go
const (
    matDashboardCharts = "mat_dashboard_charts"
    matDashboardCounts = "mat_dashboard_counts"
    matListSubStats    = "mat_list_subscriber_stats"
)
```

---

## 2. 刷新机制核心函数分析

### 2.1 `RefreshMatView()` - 强制刷新单个视图

[core.go#L98-L113](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go#L98-L113)

```go
func (c *Core) RefreshMatView(name string, concurrent bool) error {
    q := "REFRESH MATERIALIZED VIEW %s %s"
    if concurrent {
        q = fmt.Sprintf(q, "CONCURRENTLY", name)
    } else {
        q = fmt.Sprintf(q, "", name)
    }
    if _, err := c.db.Exec(q); err != nil {
        c.log.Printf("error refreshing materialized view: %s: %v", name, err)
        return err
    }
    return nil
}
```

**行为**：无条件执行 `REFRESH MATERIALIZED VIEW`，支持 `CONCURRENTLY` 选项（不锁表但需要唯一索引）。

### 2.2 `RefreshMatViews()` - 强制刷新所有视图

[core.go#L90-L96](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go#L90-L96)

```go
func (c *Core) RefreshMatViews(concurrent bool) error {
    for _, v := range []string{matDashboardCharts, matDashboardCounts, matListSubStats} {
        _ = c.RefreshMatView(v, true)
    }
    return nil
}
```

**行为**：遍历刷新所有三个物化视图，**总是使用 `CONCURRENTLY=true`**，忽略单个视图刷新错误。

### 2.3 `refreshCache()` - 条件刷新（核心逻辑）

[core.go#L115-L122](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go#L115-L122)

```go
func (c *Core) refreshCache(name string, concurrent bool) error {
    if c.consts.CacheSlowQueries {
        return nil
    }
    return c.RefreshMatView(name, concurrent)
}
```

**关键行为**：
- 当 `CacheSlowQueries = false`（默认值）：每次查询前主动刷新视图 → **实时模式**
- 当 `CacheSlowQueries = true`：直接返回 nil，不执行刷新 → **缓存模式**

---

## 3. 主动刷新路径分析

### 3.1 查询时即时刷新（CacheSlowQueries = false 时）

以下路径在每次查询前调用 `refreshCache()`，当缓存未启用时主动刷新：

| 函数 | 调用位置 | 刷新的视图 |
|------|---------|-----------|
| `GetDashboardCharts()` | [dashboard.go#L12](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/dashboard.go#L12) | `mat_dashboard_charts` |
| `GetDashboardCounts()` | [dashboard.go#L25](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/dashboard.go#L25) | `mat_dashboard_counts` |
| `QueryLists()` | [lists.go#L46](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/lists.go#L46) | `mat_list_subscriber_stats` |
| `getSubscriberCount()` | [subscribers.go#L564](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L564) | `mat_list_subscriber_stats` |

### 3.2 导入完成后强制刷新

在 [init.go#L653-L660](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L653-L660) 中，导入完成后的 PostCB 回调：

```go
PostCB: func(subject string, data any) error {
    // Refresh cached subscriber counts and stats.
    core.RefreshMatViews(true)  // 强制刷新所有视图，使用 CONCURRENTLY
    // Send admin notification.
    notifs.NotifySystem(subject, notifs.TplImport, data, nil)
    return nil
},
```

**调用时机**：导入成功完成后，在 [importer.go#L362](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L362) 和 [importer.go#L342](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L342) 中调用 `sendNotif()` 触发。

**重要**：`RefreshMatViews(true)` 是**无条件强制刷新**，不受 `CacheSlowQueries` 配置影响。

### 3.3 Cron 定时刷新（缓存模式下）

在 [init.go#L998-L1013](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L998-L1013) 中：

```go
if ko.Bool("app.cache_slow_queries") {
    intval := ko.String("app.cache_slow_queries_interval")
    _, err := c.Add(intval, func() {
        lo.Println("refreshing slow query cache")
        _ = co.RefreshMatViews(true)  // 定时强制刷新所有视图
        lo.Println("done refreshing slow query cache")
    })
    // 默认 cron: "0 3 * * *"（每天凌晨3点）
}
```

**配置位置**：[schema.sql#L242-L243](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L242-L243)

```sql
('app.cache_slow_queries', 'false'),
('app.cache_slow_queries_interval', '"0 3 * * *"'),
```

---

## 4. 依赖缓存 TTL 或手动维护的路径

### 4.1 批量删除操作（**无刷新**）

以下批量操作完成后**不会触发任何物化视图刷新**：

| 操作 | 函数位置 | 影响的视图 |
|------|---------|-----------|
| 批量删除订阅者 | `DeleteSubscribers()` [subscribers.go#L464-L479](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L464-L479) | `mat_dashboard_counts`, `mat_list_subscriber_stats` |
| 按查询删除订阅者 | `DeleteSubscribersByQuery()` [subscribers.go#L482-L491](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L482-L491) | `mat_dashboard_counts`, `mat_list_subscriber_stats` |
| 批量加入黑名单 | `BlocklistSubscribers()` [subscribers.go#L442-L450](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L442-L450) | `mat_dashboard_counts`, `mat_list_subscriber_stats` |
| 按查询加入黑名单 | `BlocklistSubscribersByQuery()` [subscribers.go#L453-L461](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L453-L461) | `mat_dashboard_counts`, `mat_list_subscriber_stats` |
| 批量管理列表（添加/移除/取消订阅） | `AddSubscriptions()` / `DeleteSubscriptions()` / `UnsubscribeLists()` | `mat_list_subscriber_stats` |
| 按查询管理列表 | `AddSubscriptionsByQuery()` / `DeleteSubscriptionsByQuery()` / `UnsubscribeListsByQuery()` | `mat_list_subscriber_stats` |

**HTTP Handler 位置**：[subscribers.go#L525-L678](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L525-L678) 中的批量操作 handler 均未调用刷新。

### 4.2 Maintenance GC 操作（**无刷新**）

| 操作 | 函数位置 | 影响的视图 |
|------|---------|-----------|
| 删除黑名单订阅者 | `DeleteBlocklistedSubscribers()` [subscribers.go#L549-L559](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L549-L559) | `mat_dashboard_counts` |
| 删除孤儿订阅者 | `DeleteOrphanSubscribers()` [subscribers.go#L536-L546](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L536-L546) | `mat_dashboard_counts` |
| 删除未确认订阅 | `DeleteUnconfirmedSubscriptions()` [maintenance.go#L42-L58](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go#L42-L58) | `mat_list_subscriber_stats` |
| 删除活动分析数据 | `DeleteCampaignViews()` / `DeleteCampaignLinkClicks()` [campaigns.go#L494-L511](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L494-L511) | `mat_dashboard_charts`, `mat_dashboard_counts` |

**HTTP Handler 位置**：[maintenance.go#L14-L87](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go#L14-L87) 中的所有 GC handler 均未调用刷新。

### 4.3 VACUUM 操作（**无刷新**）

[maintenance.go#L163-L171](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go#L163-L171)

```go
func RunDBVacuum(db *sqlx.DB, lo *log.Logger) {
    lo.Println("running database VACUUM ANALYZE")
    if _, err := db.Exec("VACUUM ANALYZE"); err != nil {
        lo.Printf("error running VACUUM ANALYZE: %v", err)
        return
    }
    lo.Println("finished database VACUUM ANALYZE")
}
```

**注意**：`VACUUM ANALYZE` 只回收死元组和更新查询规划器统计信息，**不会刷新物化视图**。

### 4.4 Cron VACUUM（**无刷新**）

[init.go#L1016-L1031](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L1016-L1031)

```go
if ko.Bool("maintenance.db.vacuum") {
    intval := ko.String("maintenance.db.vacuum_cron_interval")
    _, err := c.Add(intval, func() {
        RunDBVacuum(db, lo)  // 只执行 VACUUM，不刷新视图
    })
}
```

---

## 5. 数据不一致根因分析

### 场景复现路径

```
大批量导入 → 导入完成 → PostCB 调用 RefreshMatViews(true) → 数据短暂一致
    ↓
大批量删除/黑名单/GC → 无刷新 → 物化视图过期
    ↓
管理员打开 Dashboard
    ├─ CacheSlowQueries = false → refreshCache() 尝试刷新
    │   └─ 使用 CONCURRENTLY 刷新 → 大数据量下可能需要数分钟
    │       └─ 刷新完成前显示旧数据
    └─ CacheSlowQueries = true → refreshCache() 直接返回
        └─ 等待 cron 刷新（默认每天凌晨3点）
            ↓
管理员运行 GC + VACUUM
    ├─ GC 删除数据 → 仍无刷新
    └─ VACUUM ANALYZE → 只回收空间，不刷新视图
        ↓
数据持续不一致，直到下一次 cron 刷新或下一次导入完成
```

### 关键问题点

1. **CONCURRENTLY 刷新性能问题**：`RefreshMatViews(true)` 和导入后 PostCB 都使用 `CONCURRENTLY` 选项。对于大数据量表，`REFRESH MATERIALIZED VIEW CONCURRENTLY` 可能需要**数分钟**完成，期间查询返回旧数据。

2. **批量删除操作无刷新**：所有批量删除、黑名单、GC 操作完成后未调用任何刷新函数。

3. **VACUUM 不刷新视图**：VACUUM 与物化视图刷新是完全独立的操作。

4. **refreshCache 的静默失败**：`refreshCache()` 忽略错误返回值（使用 `_ = c.refreshCache(...)`），刷新失败时无感知。

5. **CacheSlowQueries 配置的双重影响**：
   - `false`（默认）：每次查询刷新，但使用 `CONCURRENTLY` 可能很慢
   - `true`：依赖每日 cron，最长可能延迟 24 小时

---

## 6. SQL 查询使用物化视图的位置

### 6.1 Dashboard 查询

[misc.sql#L1-L5](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/misc.sql#L1-L5)

```sql
-- name: get-dashboard-charts
SELECT data FROM mat_dashboard_charts;

-- name: get-dashboard-counts
SELECT data FROM mat_dashboard_counts;
```

### 6.2 列表统计查询

[lists.sql#L30-L39](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/lists.sql#L30-L39)

```sql
statuses AS (
    SELECT
        list_id,
        COALESCE(JSONB_OBJECT_AGG(status, subscriber_count) FILTER (WHERE status IS NOT NULL), '{}') AS subscriber_statuses,
        SUM(subscriber_count) AS subscriber_count
    FROM mat_list_subscriber_stats
    GROUP BY list_id
)
SELECT ls.*, COALESCE(ss.subscriber_statuses, '{}') AS subscriber_statuses, ...
```

### 6.3 订阅者总数查询

[subscribers.sql#L314-L318](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L314-L318)

```sql
-- name: query-subscribers-count-all
SELECT COALESCE(SUM(subscriber_count), 0) AS total FROM mat_list_subscriber_stats
    WHERE list_id = ANY(CASE WHEN CARDINALITY($1::INT[]) > 0 THEN $1 ELSE '{0}' END)
    AND ($2 = '' OR status = $2::subscription_status);
```

### 6.4 Campaign Analytics 查询

Campaign analytics 不使用物化视图，直接查询原始表：
- `GetCampaignAnalyticsCounts()` → 查询 `campaign_views` / `link_clicks` 表
- `GetCampaignAnalyticsLinks()` → 查询 `link_clicks` 表

**注意**：用户提到的 "campaign analytics" 不一致可能是因为 `mat_dashboard_counts` 中的 `messages` 字段（来源于 `campaigns.sent` 字段总和）过期，而非 campaign 详情页的实时分析。

---

## 7. 一致性验证方案

### 7.1 数据一致性校验 SQL

```sql
-- 验证 mat_dashboard_counts 与实际数据的差异
SELECT
    'subscribers_total' AS metric,
    (SELECT SUM(num) FROM (SELECT COUNT(*) AS num, status FROM subscribers GROUP BY status) s) AS actual,
    (data->'subscribers'->>'total')::bigint AS cached,
    (SELECT SUM(num) FROM (SELECT COUNT(*) AS num, status FROM subscribers GROUP BY status) s) 
    - (data->'subscribers'->>'total')::bigint AS diff
FROM mat_dashboard_counts
UNION ALL
SELECT
    'subscribers_blocklisted' AS metric,
    (SELECT COUNT(*) FROM subscribers WHERE status='blocklisted') AS actual,
    (data->'subscribers'->>'blocklisted')::bigint AS cached,
    (SELECT COUNT(*) FROM subscribers WHERE status='blocklisted') 
    - (data->'subscribers'->>'blocklisted')::bigint AS diff
FROM mat_dashboard_counts
UNION ALL
SELECT
    'subscribers_orphans' AS metric,
    (SELECT COUNT(id) FROM subscribers
     LEFT JOIN subscriber_lists ON (subscribers.id = subscriber_lists.subscriber_id)
     WHERE subscriber_lists.subscriber_id IS NULL) AS actual,
    (data->'subscribers'->>'orphans')::bigint AS cached,
    (SELECT COUNT(id) FROM subscribers
     LEFT JOIN subscriber_lists ON (subscribers.id = subscriber_lists.subscriber_id)
     WHERE subscriber_lists.subscriber_id IS NULL)
    - (data->'subscribers'->>'orphans')::bigint AS diff
FROM mat_dashboard_counts;

-- 验证 mat_list_subscriber_stats 与实际数据的差异
SELECT
    list_id, status,
    actual_count, cached_count,
    actual_count - cached_count AS diff
FROM (
    SELECT
        list_id, status, COUNT(*) AS actual_count
    FROM subscriber_lists
    GROUP BY list_id, status
) actual
FULL OUTER JOIN (
    SELECT list_id, status, subscriber_count AS cached_count
    FROM mat_list_subscriber_stats
    WHERE list_id != 0
) cached USING (list_id, status)
WHERE actual_count != cached_count OR actual_count IS NULL OR cached_count IS NULL;
```

### 7.2 视图最后刷新时间查询

```sql
-- 查看每个物化视图的最后刷新时间
SELECT 'mat_dashboard_counts' AS view_name, updated_at FROM mat_dashboard_counts
UNION ALL
SELECT 'mat_dashboard_charts' AS view_name, updated_at FROM mat_dashboard_charts
UNION ALL
SELECT 'mat_list_subscriber_stats' AS view_name, MAX(updated_at) AS updated_at FROM mat_list_subscriber_stats;
```

### 7.3 刷新性能基准测试

```sql
-- 测试非并发刷新时间（注意：会锁表！）
EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW mat_list_subscriber_stats;

-- 测试并发刷新时间
EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW CONCURRENTLY mat_list_subscriber_stats;
```

---

## 8. 修复方案建议

### 8.1 短期方案（立即生效）

1. **批量操作后主动刷新**：在批量删除、黑名单、GC 操作后调用 `core.RefreshMatViews(false)`（不使用 CONCURRENTLY，速度更快但会短暂锁表）。

2. **Maintenance 页面添加手动刷新按钮**：提供 "刷新统计数据" 按钮，管理员可手动触发刷新。

3. **调整 cron 频率**：如果 `CacheSlowQueries=true`，将刷新间隔从每日改为每小时：
   ```sql
   UPDATE settings SET value = '"0 * * * *"' WHERE key = 'app.cache_slow_queries_interval';
   ```

### 8.2 中期方案（代码修改）

1. **批量操作后异步刷新**：在 [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go) 的批量操作 handler 中，操作完成后启动 goroutine 异步刷新：
   ```go
   // DeleteSubscribers handler 末尾添加
   go func() {
       _ = a.core.RefreshMatViews(true)
   }()
   ```

2. **GC 操作后刷新**：在 [maintenance.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go) 的 GC handler 中添加刷新。

3. **改进 refreshCache 错误处理**：不忽略刷新错误，记录日志以便排查。

4. **智能选择刷新策略**：
   - 小数据量：使用非并发刷新（更快）
   - 大数据量：使用并发刷新（不锁表）
   - 或者根据操作类型选择：导入后并发刷新，删除后非并发刷新

### 8.3 长期方案（架构优化）

1. **增量刷新替代全量刷新**：使用 PostgreSQL 触发器 + 队列表，记录变更，只刷新受影响的部分。

2. **缓存失效标记**：批量操作后设置一个失效标记，下一次查询时检测到标记则触发刷新。

3. **物化视图转换为常规表 + 实时更新**：将统计数据存储在常规表中，通过触发器实时更新，适用于中等数据量场景。

4. **添加数据新鲜度指示器**：在 Dashboard 上显示 "数据最后更新于 X 分钟前"，让用户了解数据时效。

---

## 9. 刷新路径总结表

| 操作路径 | 主动 REFRESH？ | 受 `CacheSlowQueries` 影响？ | 使用 CONCURRENTLY？ | 影响的视图 |
|---------|---------------|----------------------------|--------------------|-----------|
| **打开 Dashboard** | ✅ 每次查询前 | ✅（true 时不刷新） | ❌（`concurrent=false`） | `mat_dashboard_counts`, `mat_dashboard_charts` |
| **打开列表页** | ✅ 每次查询前 | ✅（true 时不刷新） | ❌（`concurrent=false`） | `mat_list_subscriber_stats` |
| **查询订阅者列表** | ✅ 无查询条件时 | ✅（true 时不刷新） | ❌（`concurrent=false`） | `mat_list_subscriber_stats` |
| **导入完成后（PostCB）** | ✅ 强制刷新 | ❌（不受影响） | ✅（总是 true） | 所有三个视图 |
| **Cron 定时任务** | ✅ 定时刷新 | ❌（仅 true 时启用） | ✅（总是 true） | 所有三个视图 |
| **批量删除订阅者** | ❌ 无刷新 | - | - | 所有三个视图 |
| **批量加入黑名单** | ❌ 无刷新 | - | - | 所有三个视图 |
| **GC 删除黑名单** | ❌ 无刷新 | - | - | `mat_dashboard_counts` |
| **GC 删除孤儿** | ❌ 无刷新 | - | - | `mat_dashboard_counts` |
| **GC 删除未确认订阅** | ❌ 无刷新 | - | - | `mat_list_subscriber_stats` |
| **GC 删除活动分析** | ❌ 无刷新 | - | - | `mat_dashboard_charts`, `mat_dashboard_counts` |
| **VACUUM ANALYZE** | ❌ 无刷新 | - | - | 无（不影响物化视图） |
| **单个添加/更新订阅者** | ❌ 无刷新 | - | - | 所有三个视图 |
| **列表管理（添加/移除）** | ❌ 无刷新 | - | - | `mat_list_subscriber_stats` |

---

## 10. 关键代码索引

| 功能 | 文件 | 行号 |
|------|------|------|
| 物化视图定义 | [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql) | L362-L443 |
| `RefreshMatViews()` | [core.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go) | L90-L96 |
| `RefreshMatView()` | [core.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go) | L98-L113 |
| `refreshCache()` | [core.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go) | L115-L122 |
| 导入 PostCB 刷新 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go) | L653-L660 |
| Cron 刷新任务 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go) | L993-L1013 |
| Cron VACUUM 任务 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go) | L1016-L1031 |
| Dashboard 查询刷新 | [dashboard.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/dashboard.go) | L12, L25 |
| 列表查询刷新 | [lists.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/lists.go) | L46 |
| 订阅者计数刷新 | [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go) | L564 |
| 批量删除 Handler | [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go) | L525-L590 |
| GC Handler | [maintenance.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go) | L14-L87 |
| RunDBVacuum | [maintenance.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go) | L163-L171 |
