# Campaign 浏览、点击、导出与隐私擦除关系分析

## 一、问题场景描述

管理员执行以下操作后观察到的现象：

1. **关闭 `individual_tracking` 但未关闭 `disable_tracking`**
   - Campaign 报表**总点击数仍继续增长**
   - 导出的 clicks CSV **缺少订阅者身份信息**

2. **擦除订阅者数据后**
   - 历史报表**仍保留总数**（浏览/点击计数不变）

---

## 二、核心组件分析

### 2.1 RegisterCampaignView — 浏览追踪

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L564-L592)

**功能**: 处理追踪像素（`{{ TrackView }}`）的 HTTP 请求，记录 campaign 浏览。

```go
func (a *App) RegisterCampaignView(c echo.Context) error {
    // [1] 全局追踪禁用 → 只返回像素，不记录
    if a.cfg.Privacy.DisableTracking {
        return c.Blob(http.StatusOK, "image/png", pixelPNG)
    }

    // [2] 个人追踪禁用 → 清空 subUUID
    subUUID := c.Param("subUUID")
    if !a.cfg.Privacy.IndividualTracking {
        subUUID = ""
    }

    // [3] 记录浏览（排除预览 dummy UUID）
    if campUUID != dummyUUID && subUUID != dummyUUID {
        a.core.RegisterCampaignView(campUUID, subUUID)
    }

    return c.Blob(http.StatusOK, "image/png", pixelPNG)
}
```

**SQL 写入** ([campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L483-L489)):
```sql
WITH view AS (
    SELECT campaigns.id as campaign_id, subscribers.id AS subscriber_id FROM campaigns
    LEFT JOIN subscribers ON (CASE WHEN $2::TEXT != '' THEN subscribers.uuid = $2::UUID ELSE FALSE END)
    WHERE campaigns.uuid = $1
)
INSERT INTO campaign_views (campaign_id, subscriber_id)
    VALUES((SELECT campaign_id FROM view), (SELECT subscriber_id FROM view));
```

**关键逻辑**:
- 当 `$2::TEXT = ''`（`IndividualTracking=false` 或 `subUUID` 无效）时，`subscriber_id` 子查询返回 `NULL`
- 记录仍然写入 `campaign_views` 表，但 `subscriber_id = NULL`

---

### 2.2 LinkRedirect — 点击追踪

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L530-L563)

**功能**: 处理追踪链接（`{{ TrackLink }}`）的点击请求，记录点击并重定向。

```go
func (a *App) LinkRedirect(c echo.Context) error {
    // [1] 全局追踪禁用 → 只重定向，不记录点击
    if a.cfg.Privacy.DisableTracking {
        url, _ := a.core.GetLinkURL(linkUUID)
        return c.Redirect(http.StatusTemporaryRedirect, url)
    }

    // [2] 个人追踪禁用 → 清空 subUUID
    subUUID := c.Param("subUUID")
    if !a.cfg.Privacy.IndividualTracking {
        subUUID = ""
    }

    // [3] 记录点击 + 获取原始 URL
    url, _ := a.core.RegisterCampaignLinkClick(linkUUID, campUUID, subUUID)
    return c.Redirect(http.StatusTemporaryRedirect, url)
}
```

**SQL 写入** ([links.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L8-L18)):
```sql
WITH link AS(
    SELECT id, url FROM links WHERE uuid = $1
)
INSERT INTO link_clicks (campaign_id, subscriber_id, link_id) VALUES(
    (SELECT id FROM campaigns WHERE uuid = $2),
    (SELECT id FROM subscribers WHERE
        (CASE WHEN $3::TEXT != '' THEN subscribers.uuid = $3::UUID ELSE FALSE END)
    ),
    (SELECT id FROM link)
) RETURNING (SELECT url FROM link);
```

**关键逻辑**:
- 当 `$3::TEXT = ''` 时，`subscriber_id` 子查询返回 `NULL`
- 记录仍然写入 `link_clicks` 表，但 `subscriber_id = NULL`
- **总点击数继续增长**（因为记录本身被插入）

---

### 2.3 Core Analytics 查询

**文件位置**: [campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L384-L422)

**查询选择逻辑** ([init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L400-L429)):

```go
func prepareQueries(qMap goyesql.Queries, db *sqlx.DB, ko *koanf.Koanf) *models.Queries {
    countQuery := "get-campaign-analytics-counts"
    linkSel := "*"
    
    if ko.Bool("privacy.individual_tracking") {
        countQuery = "get-campaign-analytics-unique-counts"
        linkSel = "DISTINCT subscriber_id"
    }
    
    // 动态插值表名和选择器
    qMap["get-campaign-view-counts"].Query = fmt.Sprintf(qMap[countQuery].Query, "campaign_views")
    qMap["get-campaign-click-counts"].Query = fmt.Sprintf(qMap[countQuery].Query, "link_clicks")
    qMap["get-campaign-link-counts"].Query = fmt.Sprintf(qMap["get-campaign-link-counts"].Query, linkSel)
}
```

#### 查询类型对比

| individual_tracking | 浏览/点击统计查询 | 链接统计查询 |
|---------------------|-------------------|-------------|
| **true** (启用) | `get-campaign-analytics-unique-counts` | `COUNT(DISTINCT subscriber_id)` |
| **false** (禁用) | `get-campaign-analytics-counts` | `COUNT(*)` |

##### **通用计数查询** (`individual_tracking=false`)
```sql
-- name: get-campaign-analytics-counts
SELECT campaign_id, COUNT(*) AS "count", DATE_TRUNC(...) AS "timestamp"
    FROM %s
    WHERE campaign_id=ANY($1) AND created_at >= $2 AND created_at <= $3
    GROUP BY campaign_id, "timestamp" ORDER BY "timestamp" ASC;
```

##### **去重计数查询** (`individual_tracking=true`)
```sql
-- name: get-campaign-analytics-unique-counts
WITH uniqIDs AS (
    SELECT DISTINCT ON(subscriber_id, campaign_id) subscriber_id, campaign_id, DATE_TRUNC(...) AS "timestamp"
    FROM %s
    WHERE ...
    ORDER BY subscriber_id, campaign_id, "timestamp"
)
SELECT COUNT(*) AS "count", campaign_id, "timestamp"
    FROM uniqIDs GROUP BY campaign_id, "timestamp";
```

**关键差异**:
- `COUNT(*)`: 统计**所有记录**（包括 `subscriber_id=NULL` 的记录），同一用户多次点击**每次都计数**
- `COUNT(DISTINCT subscriber_id)`: 统计**独立用户**，`subscriber_id=NULL` 的记录可能被排除或只计 1 次

---

### 2.4 Export Endpoints — 数据导出

**文件位置**: 
- HTTP Handler: [maintenance.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go#L89-L160)
- Core 逻辑: [campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L462-L491)

**导出 SQL** ([campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L278-L307)):

```sql
-- name: export-campaign-views
SELECT campaign_views.campaign_id,
       COALESCE(campaigns.uuid::TEXT, '') AS campaign_uuid,
       COALESCE(campaigns.name, '') AS campaign_name,
       COALESCE(campaign_views.subscriber_id, 0) AS subscriber_id,  -- NULL → 0
       COALESCE(subscribers.uuid::TEXT, '') AS subscriber_uuid,     -- NULL → ''
       COALESCE(subscribers.email, '') AS email,                    -- NULL → ''
       COALESCE(subscribers.name, '') AS subscriber_name,           -- NULL → ''
       campaign_views.created_at
    FROM campaign_views
    LEFT JOIN campaigns ON (campaigns.id = campaign_views.campaign_id)
    LEFT JOIN subscribers ON (subscribers.id = campaign_views.subscriber_id)
    WHERE ...;

-- name: export-campaign-link-clicks
SELECT link_clicks.campaign_id,
       COALESCE(link_clicks.subscriber_id, 0) AS subscriber_id,
       COALESCE(subscribers.uuid::TEXT, '') AS subscriber_uuid,
       COALESCE(subscribers.email, '') AS email,
       COALESCE(subscribers.name, '') AS subscriber_name,
       links.url, link_clicks.created_at
    FROM link_clicks
    LEFT JOIN campaigns ON (campaigns.id = link_clicks.campaign_id)
    LEFT JOIN links ON (links.id = link_clicks.link_id)
    LEFT JOIN subscribers ON (subscribers.id = link_clicks.subscriber_id)
    WHERE ...;
```

**导出结构** ([templates.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/templates.go#L81-L103)):
```go
type CampaignViewExport struct {
    CampaignID     int       `db:"campaign_id"`
    CampaignUUID   string    `db:"campaign_uuid"`
    CampaignName   string    `db:"campaign_name"`
    SubscriberID   int       `db:"subscriber_id"`   // 可能为 0
    SubscriberUUID string    `db:"subscriber_uuid"` // 可能为 ""
    Email          string    `db:"email"`           // 可能为 ""
    SubscriberName string    `db:"subscriber_name"` // 可能为 ""
    CreatedAt      time.Time `db:"created_at"`
}
```

**CSV 输出字段**:
- Views CSV: `campaign_id, campaign_uuid, campaign_name, subscriber_id, subscriber_uuid, email, subscriber_name, created_at`
- Clicks CSV: 上述字段 + `url`

---

### 2.5 Schema 外键策略

**文件位置**: [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L155-L219)

#### 外键约束汇总

| 表名 | 字段 | 引用表 | ON DELETE 行为 | 说明 |
|------|------|--------|---------------|------|
| **campaign_views** | `subscriber_id` | subscribers | **SET NULL** | 删除订阅者时，subscriber_id 置空 |
| campaign_views | `campaign_id` | campaigns | **CASCADE** | 删除 campaign 时，同步删除浏览记录 |
| **link_clicks** | `subscriber_id` | subscribers | **SET NULL** | 删除订阅者时，subscriber_id 置空 |
| link_clicks | `campaign_id` | campaigns | **CASCADE** | 删除 campaign 时，同步删除点击记录 |
| link_clicks | `link_id` | links | **CASCADE** | 删除链接时，同步删除点击记录 |
| **bounces** | `subscriber_id` | subscribers | **CASCADE** | 删除订阅者时，同步删除退信记录 |

#### 关键表定义

```sql
-- campaign_views 表
CREATE TABLE campaign_views (
    id               BIGSERIAL PRIMARY KEY,
    campaign_id      INTEGER NOT NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    subscriber_id    INTEGER NULL REFERENCES subscribers(id) ON DELETE SET NULL,  -- ⚠️ 可空 + SET NULL
    created_at       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

-- link_clicks 表
CREATE TABLE link_clicks (
    id               BIGSERIAL PRIMARY KEY,
    campaign_id      INTEGER NULL REFERENCES campaigns(id) ON DELETE CASCADE,
    link_id          INTEGER NOT NULL REFERENCES links(id) ON DELETE CASCADE,
    subscriber_id    INTEGER NULL REFERENCES subscribers(id) ON DELETE SET NULL,  -- ⚠️ 可空 + SET NULL
    created_at       TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

**设计意图**:
- 注释明确说明: `"Subscribers may be deleted, but the view/link counts should remain."`
- **记录本身保留**，仅断开与订阅者的关联

---

### 2.6 隐私擦除逻辑

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L654-L670)

```go
func (a *App) WipeSubscriberData(c echo.Context) error {
    subUUID := c.Param("subUUID")
    // 调用 DeleteSubscribers 删除订阅者
    if err := a.core.DeleteSubscribers(nil, []string{subUUID}); err != nil {
        // ...
    }
}
```

**核心 SQL** ([subscribers.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L203-L205)):
```sql
DELETE FROM subscribers WHERE CASE WHEN ARRAY_LENGTH($1::INT[], 1) > 0 THEN id = ANY($1) ELSE uuid = ANY($2::UUID[]) END;
```

**擦除触发的外键连锁反应**:

| 表 | 外键行为 | 最终数据状态 |
|----|----------|--------------|
| **subscribers** | 主表删除 | ✅ 记录被完全删除 |
| **subscriber_lists** | CASCADE | ✅ 所有订阅关系被删除 |
| **campaign_views** | SET NULL | ⚠️ 记录保留，`subscriber_id = NULL` |
| **link_clicks** | SET NULL | ⚠️ 记录保留，`subscriber_id = NULL` |
| **bounces** | CASCADE | ✅ 所有退信记录被删除 |

---

## 三、问题现象解释

### 3.1 现象 1: 关闭 individual_tracking 后总点击数仍增长

**原因**:
1. `DisableTracking = false` → LinkRedirect 仍调用 `RegisterCampaignLinkClick` 记录点击
2. `IndividualTracking = false` → `subUUID` 被清空，但 SQL 仍执行 `INSERT`
3. `link_clicks` 表中新增记录，`subscriber_id = NULL`
4. Analytics 查询使用 `COUNT(*)` → 统计所有记录，包括 `subscriber_id=NULL`

**代码证据**:
- LinkRedirect: [public.go L546-L560](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L546-L560)
- register-link-click SQL: [links.sql L8-L18](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L8-L18)
- get-campaign-analytics-counts: [campaigns.sql L251-L267](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L251-L267)

---

### 3.2 现象 2: 导出的 clicks CSV 缺少订阅者身份

**原因**:
1. 记录写入时 `subscriber_id = NULL`（因 `IndividualTracking=false`）
2. 导出 SQL 使用 `LEFT JOIN subscribers ON (subscribers.id = link_clicks.subscriber_id)`
3. `NULL` 值无法匹配任何 `subscribers.id` → JOIN 结果为 NULL
4. `COALESCE(subscribers.email, '')` → 空字符串
5. `COALESCE(subscribers.name, '')` → 空字符串
6. `COALESCE(link_clicks.subscriber_id, 0)` → 0

**导出 CSV 示例行**:
```
campaign_id,campaign_uuid,campaign_name,subscriber_id,subscriber_uuid,email,subscriber_name,url,created_at
1,abc-123,Test Campaign,0,,,https://example.com,2024-01-01T00:00:00Z
```

**代码证据**:
- export-campaign-link-clicks SQL: [campaigns.sql L293-L307](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L293-L307)
- ExportCampaignAnalytics: [maintenance.go L134-L157](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go#L134-L157)

---

### 3.3 现象 3: 擦除订阅者数据后历史报表仍保留总数

**原因**:
1. 执行 `DELETE FROM subscribers` 触发外键 `ON DELETE SET NULL`
2. `campaign_views` 和 `link_clicks` 表中的记录**不被删除**，仅 `subscriber_id` 被置为 `NULL`
3. Analytics 查询使用 `COUNT(*)` → 记录数不变，总数保留
4. 即使之前 `individual_tracking=true`（有真实 `subscriber_id`），擦除后也变为 `NULL`，但 `COUNT(*)` 仍统计所有行

**代码证据**:
- schema.sql campaign_views 外键: [schema.sql L155-L166](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L155-L166)
- schema.sql link_clicks 外键: [schema.sql L206-L219](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L206-L219)

---

## 四、聚合保留 vs 身份字段

### 4.1 聚合保留统计（始终计数）

| 统计类型 | 数据来源 | 聚合方式 | 是否保留 |
|---------|---------|---------|---------|
| 总浏览数 | `campaign_views` 表行数 | `COUNT(*)` | ✅ 始终保留（除非删除 campaign） |
| 总点击数 | `link_clicks` 表行数 | `COUNT(*)` | ✅ 始终保留（除非删除 campaign） |
| 链接点击排行 | `link_clicks` + `links` | `COUNT(*)` 按 URL 分组 | ✅ 始终保留 |
| 独立用户浏览数 | `campaign_views` 去重 | `COUNT(DISTINCT subscriber_id)` | ⚠️ 仅 `individual_tracking=true` 时可用 |
| 独立用户点击数 | `link_clicks` 去重 | `COUNT(DISTINCT subscriber_id)` | ⚠️ 仅 `individual_tracking=true` 时可用 |

### 4.2 可能为空的身份字段

| 字段 | 为空场景 | 导出时值 |
|-----|---------|---------|
| `campaign_views.subscriber_id` | `IndividualTracking=false` 或订阅者已擦除 | `0` |
| `link_clicks.subscriber_id` | `IndividualTracking=false` 或订阅者已擦除 | `0` |
| `subscribers.uuid` (导出) | `subscriber_id=NULL` 导致 LEFT JOIN 无匹配 | `''` |
| `subscribers.email` (导出) | `subscriber_id=NULL` 导致 LEFT JOIN 无匹配 | `''` |
| `subscribers.name` (导出) | `subscriber_id=NULL` 导致 LEFT JOIN 无匹配 | `''` |

---

## 五、隐私开关影响矩阵

### 5.1 开关行为对比

| 配置项 | 值 | 对浏览追踪的影响 | 对点击追踪的影响 | 统计方式 |
|-------|----|----------------|----------------|---------|
| **`privacy.disable_tracking`** | `true` | ❌ 完全不记录 | ❌ 完全不记录 | 无统计 |
| **`privacy.disable_tracking`** | `false` | ✅ 记录（可能匿名） | ✅ 记录（可能匿名） | 有统计 |
| **`privacy.individual_tracking`** | `true` | ✅ 关联到具体订阅者 | ✅ 关联到具体订阅者 | 独立用户统计（DISTINCT） |
| **`privacy.individual_tracking`** | `false` | ⚠️ 匿名记录（subscriber_id=NULL） | ⚠️ 匿名记录（subscriber_id=NULL） | 总计数统计（COUNT(*)） |

### 5.2 完全停止追踪的条件

**只有 `privacy.disable_tracking = true` 才能完全停止追踪**:

1. **RegisterCampaignView**: [public.go L568-L572](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L568-L572)
   ```go
   if a.cfg.Privacy.DisableTracking {
       // 直接返回像素，不调用 RegisterCampaignView
       return c.Blob(http.StatusOK, "image/png", pixelPNG)
   }
   ```

2. **LinkRedirect**: [public.go L536-L543](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L536-L543)
   ```go
   if a.cfg.Privacy.DisableTracking {
       // 直接查询 URL 并重定向，不调用 RegisterCampaignLinkClick
       url, _ := a.core.GetLinkURL(linkUUID)
       return c.Redirect(http.StatusTemporaryRedirect, url)
   }
   ```

3. **模板渲染**: [manager_store.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/message.go 附近)
   ```go
   "TrackLink": func(url string, msg *CampaignMessage) string {
       if m.cfg.DisableTracking {
           return url  // 返回原始 URL，不生成追踪链接
       }
       // ...
   }
   ```

---

## 六、完整数据流示意图

```
邮件发送阶段
├── 模板渲染 TrackLink / TrackView
│   ├── DisableTracking=true → 返回原始 URL / 不生成追踪像素
│   └── DisableTracking=false
│       ├── IndividualTracking=true → 嵌入真实 subUUID
│       └── IndividualTracking=false → 嵌入 dummyUUID
│
用户交互阶段
├── 浏览像素请求 /campaign/{campUUID}/{subUUID}/px.png
│   └── RegisterCampaignView
│       ├── DisableTracking=true → 返回像素，不记录
│       └── DisableTracking=false
│           ├── IndividualTracking=true → subUUID 保留 → subscriber_id=真实ID
│           └── IndividualTracking=false → subUUID 清空 → subscriber_id=NULL
│           └── INSERT INTO campaign_views (campaign_id, subscriber_id)
│
└── 链接点击请求 /link/{linkUUID}/{campUUID}/{subUUID}
    └── LinkRedirect
        ├── DisableTracking=true → 仅重定向，不记录
        └── DisableTracking=false
            ├── IndividualTracking=true → subUUID 保留 → subscriber_id=真实ID
            └── IndividualTracking=false → subUUID 清空 → subscriber_id=NULL
            └── INSERT INTO link_clicks (campaign_id, subscriber_id, link_id)

统计查询阶段
├── Analytics 查询
│   ├── IndividualTracking=true → COUNT(DISTINCT subscriber_id) → 独立用户数
│   └── IndividualTracking=false → COUNT(*) → 总次数（含匿名记录）
│
└── CSV 导出
    ├── subscriber_id 有值 → JOIN subscribers → 身份信息完整
    └── subscriber_id=NULL → LEFT JOIN 无匹配 → COALESCE 转为 0/空字符串

数据擦除阶段
└── DELETE FROM subscribers WHERE uuid = ?
    ├── subscriber_lists → CASCADE → 记录删除
    ├── bounces → CASCADE → 记录删除
    ├── campaign_views → SET NULL → 记录保留，subscriber_id=NULL
    └── link_clicks → SET NULL → 记录保留，subscriber_id=NULL
        └── Analytics COUNT(*) → 总数不变
```

---

## 七、关键结论

| 问题 | 答案 |
|-----|------|
| **哪些统计是聚合保留？** | 总浏览数、总点击数、链接点击排行（基于 `COUNT(*)`），即使订阅者被擦除，这些统计值也不会改变 |
| **哪些身份字段可能为空？** | `subscriber_id`、`subscriber_uuid`、`email`、`subscriber_name`，在 `IndividualTracking=false` 或订阅者被擦除时为空 |
| **哪些开关会完全停止追踪？** | **只有 `privacy.disable_tracking = true`**，`individual_tracking=false` 仅停止身份关联，仍会记录匿名统计 |
| **为什么总点击数继续增长？** | `IndividualTracking=false` 时，`link_clicks` 表仍插入记录（`subscriber_id=NULL`），`COUNT(*)` 统计所有行 |
| **为什么导出 CSV 缺少身份？** | `subscriber_id=NULL` 导致 `LEFT JOIN subscribers` 无匹配，`COALESCE` 将 NULL 转为 0 或空字符串 |
| **为什么擦除后总数保留？** | 外键 `ON DELETE SET NULL` 仅断开关联，不删除记录，`COUNT(*)` 统计的是记录行数 |

---

## 八、代码参考索引

| 功能 | 文件 | 行号 |
|------|------|------|
| RegisterCampaignView | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L564-L592 |
| LinkRedirect | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L530-L563 |
| RegisterCampaignView SQL | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L483-L489 |
| RegisterLinkClick SQL | [links.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql) | L8-L18 |
| prepareQueries（动态查询选择） | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go) | L400-L429 |
| 通用计数查询 | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L251-L267 |
| 去重计数查询 | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L238-L249 |
| 链接统计查询 | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L269-L276 |
| ExportCampaignAnalytics | [maintenance.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/maintenance.go) | L89-L160 |
| 导出 SQL (views) | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L278-L291 |
| 导出 SQL (clicks) | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L293-L307 |
| WipeSubscriberData | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L654-L670 |
| DeleteSubscribers | [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go) | L462-L479 |
| campaign_views 外键 | [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql) | L155-L166 |
| link_clicks 外键 | [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql) | L206-L219 |
| 隐私设置定义 | [settings.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/settings.go) | L31-L41 |
