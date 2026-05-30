# Campaign URL 追踪链路全流程分析

> 从模板编写到用户点击，完整推演一个 URL 如何变成可追踪短链，以及各环节对统计的影响。

---

## 1. 全局概览：URL 的生命周期

```
用户编写模板
  │
  ├─ {{ TrackLink "https://example.com" }}     ← 显式调用
  ├─ https://example.com@TrackLink             ← 简写形式
  └─ {{ UnsubscribeURL }} / {{ MessageURL }}   ← 系统内置 URL
  │
  ▼
[1] CompileTemplate ── 正则替换 → 标准模板函数调用
  │
  ▼
[2] TemplateFuncs ── TrackLink 闭包注册 → 判断 privacy 配置
  │
  ▼
[3] manager.trackLink() ── 内存缓存查询 → 首次则调 CreateLink
  │
  ▼
[4] store.CreateLink() ── INSERT INTO links ... ON CONFLICT → 去重并返回 link UUID
  │
  ▼
[5] 格式化追踪 URL ── {root}/link/{linkUUID}/{campUUID}/{subUUID}
  │
  ▼
邮件发送 → 用户点击
  │
  ▼
[6] LinkRedirect handler ── 解析三个 UUID → 判断 privacy → 记录点击
  │
  ▼
[7] register-link-click SQL ── INSERT INTO link_clicks → 含/不含 subscriber_id
  │
  ▼
[8] 统计查询 (get-campaign-link-counts) ── COUNT(*) 或 COUNT(DISTINCT subscriber_id)
```

---

## 2. 阶段一：CompileTemplate — 简写与正则替换

**入口**: [models/campaigns.go CompileTemplate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L138-L242)

**核心正则定义**: [models/common.go regTplFuncs](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/common.go#L43-L68)

`CompileTemplate` 在编译模板体之前，对内容执行三轮正则替换 (`regTplFuncs`)：

### 2.1 TrackLink 显式形式

```
输入: {{ TrackLink "https://example.com" }}
正则: {{\s*TrackLink\s+"([^"]+)"\s*}}
输出: {{ TrackLink "https://example.com" . }}
```

将缺少点上下文的调用补全为带 `.`（即 `CampaignMessage`）的形式，以便模板引擎执行时能将当前订阅者信息传入。

### 2.2 TrackLink 简写形式 (`@TrackLink`)

```
输入: <a href="https://example.com@TrackLink">链接</a>
正则: (https?://[\p{L}\p{N}_\-\.~!#$&'()*+,/:;=?@\[\]%]*)@TrackLink
输出: <a href="{{ TrackLink "https://example.com" . }}">链接</a>
```

**关键细节**：正则匹配 URL 部分时使用了 RFC 3986 允许的字符集，`@TrackLink` 前面的整段 URL（含查询参数）都会被捕获为 `$1`。这是为 WYSIWYG 编辑器设计的，因为富文本编辑器会破坏 `{{ "" }}` 中的引号。

### 2.3 其他系统 URL

```
输入: {{ UnsubscribeURL }}
正则: {{(\s+)?(TrackView|UnsubscribeURL|ManageURL|OptinURL|MessageURL)(\s+)?}}
输出: {{ UnsubscribeURL . }}
```

同样补全上下文参数。

### 2.4 "同一 URL 多次渲染" 的根源

**当模板中同一原始 URL 以不同形式出现时**，正则替换会产生相同的结果：

```html
<a href="{{ TrackLink "https://example.com" . }}">方式一</a>
<a href="https://example.com@TrackLink">方式二</a>
```

两者在 `CompileTemplate` 后都变为 `{{ TrackLink "https://example.com" . }}`，在后续模板执行时走完全相同的逻辑路径，**不会产生不同的追踪 URL**。

但需要注意：`UnsubscribeURL`、`MessageURL`、`TrackView` 是**独立于 TrackLink 的模板函数**，它们各自生成不同格式的 URL，与 TrackLink 的追踪机制无关：

| 模板函数 | 生成的 URL 格式 | 用途 |
|----------|----------------|------|
| `TrackLink` | `{root}/link/{linkUUID}/{campUUID}/{subUUID}` | 点击追踪 |
| `TrackView` | `{root}/campaign/{campUUID}/{subUUID}/px.png` | 打开追踪（像素图） |
| `UnsubscribeURL` | `{root}/subscription/{campUUID}/{subUUID}` | 退订链接 |
| `MessageURL` | `{root}/campaign/{campUUID}/{subUUID}` | 在线查看 |

---

## 3. 阶段二：TemplateFuncs — TrackLink 闭包与 Privacy 判断

**入口**: [manager/manager.go TemplateFuncs](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L338-L399)

模板编译时，`TemplateFuncs` 将 `TrackLink` 注册为闭包函数：

```go
"TrackLink": func(url string, msg *CampaignMessage) string {
    if m.cfg.DisableTracking {
        return url                              // [A] 全局禁用 → 返回原始 URL
    }

    subUUID := msg.Subscriber.UUID
    if !m.cfg.IndividualTracking {
        subUUID = dummyUUID                     // [B] 非个人追踪 → 使用 dummy UUID
    }

    return m.trackLink(url, msg.Campaign.UUID, subUUID)
}
```

### 3.1 DisableTracking = true 的影响

- `TrackLink` 直接返回原始 URL，**不经过 `trackLink()` 函数**
- 邮件中的链接就是原始 URL，点击时不会经过 `/link/` 路由
- **后果**：不会在 `links` 表中创建记录，不会产生任何点击统计

### 3.2 IndividualTracking = false 的影响

- `subUUID` 被替换为 `dummyUUID` = `"00000000-0000-0000-0000-000000000000"`
- 所有订阅者的追踪 URL 中 subUUID 部分相同
- **后果**：点击时无法区分不同订阅者，`link_clicks` 表中 `subscriber_id` 为 NULL

### 3.3 Campaign UUID 的来源

`msg.Campaign.UUID` 是 campaign 创建时生成的 UUID（非自增 ID），在整个追踪链路中作为 URL 路径参数传递，用于在 `register-link-click` SQL 中反查 `campaigns.id`。

### 3.4 Subscriber UUID 的来源

`msg.Subscriber.UUID` 来自 `subscribers` 表，在 [NewCampaignMessage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/message.go#L13-L29) 中绑定到 `CampaignMessage`：

```go
msg := CampaignMessage{
    Campaign:   c,
    Subscriber: s,
    unsubURL:   fmt.Sprintf(m.cfg.UnsubURL, c.UUID, s.UUID),
}
```

每次模板执行 `{{ TrackLink ... }}` 时，`msg` 就是当前订阅者的 `CampaignMessage`，因此同一模板为不同订阅者生成的追踪 URL 仅在 `subUUID` 段不同（若 `IndividualTracking=true`）。

---

## 4. 阶段三：manager.trackLink — 内存缓存与链接注册

**入口**: [manager/manager.go trackLink](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L582-L613)

```go
func (m *Manager) trackLink(url, campUUID, subUUID string) string {
    if m.cfg.DisableTracking {
        return url
    }

    url = strings.ReplaceAll(url, "&amp;", "&")    // [1] HTML 实体还原

    m.linksMut.RLock()
    if uu, ok := m.links[url]; ok {                // [2] 内存缓存命中
        m.linksMut.RUnlock()
        return fmt.Sprintf(m.cfg.LinkTrackURL, uu, campUUID, subUUID)
    }
    m.linksMut.RUnlock()

    uu, err := m.store.CreateLink(url)              // [3] 首次：写 DB
    if err != nil {
        m.log.Printf("error registering tracking for link '%s': %v", url, err)
        return url                                  // [4] 失败降级
    }

    m.linksMut.Lock()
    m.links[url] = uu                               // [5] 写入内存缓存
    m.linksMut.Unlock()

    return fmt.Sprintf(m.cfg.LinkTrackURL, uu, campUUID, subUUID)
}
```

### 4.1 内存缓存 (`m.links`) 的作用

`m.links` 是 `map[string]string` 类型，key 是原始 URL，value 是 link UUID。**同一个 URL 在同一 campaign 的所有订阅者中只注册一次**，后续直接从缓存读取 UUID。

这意味着：
- 同一原始 URL → 同一 `linkUUID`（全局唯一，跨 campaign 共享）
- 同一原始 URL + 同一 campaign → 同一 `linkUUID` + 同一 `campUUID`
- 唯一区分订阅者的变量是 `subUUID`（若 `IndividualTracking=true`）

### 4.2 `&amp;` 还原

`url = strings.ReplaceAll(url, "&amp;", "&")` — 处理 HTML 编码的 ampersand。如果模板中出现了 `&amp;`（WYSIWYG 编辑器常见），在此处还原为 `&`，保证 URL 作为缓存 key 时与数据库一致。

### 4.3 失败降级

如果 `CreateLink` 失败（如数据库不可用），`trackLink` 返回原始 URL，**不写入缓存**。此时：
- 用户点击时直接访问原始网站
- 无追踪记录
- 每次渲染该 URL 都会重试 `CreateLink`（因为缓存中没有）

### 4.4 LinkTrackURL 格式

定义在 [init.go initUrlConfig](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L473)：

```go
LinkTrackURL: fmt.Sprintf("%s/link/%%s/%%s/%%s", root)
```

最终格式：`{root}/link/{linkUUID}/{campUUID}/{subUUID}`

**注意参数顺序**：`linkUUID` 在前，`campUUID` 在中，`subUUID` 在后。这与路由定义 [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L277) 一致：

```go
g.GET("/link/:linkUUID/:campUUID/:subUUID", noIndex(a.hasUUID(a.LinkRedirect, ...)))
```

---

## 5. 阶段四：store.CreateLink — 数据库去重

**入口**: [manager_store.go CreateLink](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/manager_store.go#L106-L122)

```go
func (s *store) CreateLink(url string) (string, error) {
    uu, err := uuid.NewV4()
    if err != nil {
        return "", err
    }

    var out string
    if err := s.queries.CreateLink.Get(&out, uu, url); err != nil {
        return "", err
    }
    return out, nil
}
```

**SQL** ([links.sql create-link](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L2-L3))：

```sql
INSERT INTO links (uuid, url) VALUES($1, $2)
ON CONFLICT (url) DO UPDATE SET url=EXCLUDED.url
RETURNING uuid;
```

### 5.1 去重机制

`links` 表的 `url` 列有 `UNIQUE` 约束（[schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L202)）：

```sql
url TEXT NOT NULL UNIQUE,
```

- **首次插入**：新 UUID + URL 写入，`RETURNING uuid` 返回新 UUID
- **URL 已存在**（`ON CONFLICT`）：`DO UPDATE SET url=EXCLUDED.url` 触发，返回**已存在的 UUID**

**关键含义**：同一原始 URL 在全局范围内（跨所有 campaign）共享同一个 `linkUUID`。`links` 表存储的是 **URL → linkUUID** 的全局映射，与 campaign 和 subscriber 无关。

### 5.2 `ON CONFLICT DO UPDATE` 的微妙之处

`DO UPDATE SET url=EXCLUDED.url` 实际上什么都没改（用同一个值覆盖自己），但这样做是为了让 `RETURNING uuid` 能返回已存在行的 UUID。这是 PostgreSQL 的常见模式：利用 `ON CONFLICT` + `RETURNING` 实现 "INSERT OR GET" 的原子操作。

### 5.3 对统计的影响

因为同一 URL 跨 campaign 共享 linkUUID，所以：
- `link_clicks` 表必须同时存储 `campaign_id` 和 `link_id` 才能区分是哪个 campaign 的点击
- 同一 URL 在不同 campaign 中被使用，会分别产生 `link_clicks` 记录

---

## 6. 阶段五：完整追踪 URL 的组成

经过上述所有步骤，最终生成的追踪 URL 为：

```
https://listmonk.example.com/link/{linkUUID}/{campUUID}/{subUUID}
```

| 段 | 来源 | 含义 | 唯一性 |
|----|------|------|--------|
| `linkUUID` | `links` 表（全局去重） | 标识原始 URL | 同一 URL 全局共享 |
| `campUUID` | `campaigns` 表 | 标识 campaign | 同一 campaign 内相同 |
| `subUUID` | `subscribers` 表 或 `dummyUUID` | 标识订阅者 | 若 `IndividualTracking=true` 则每人不同；否则全是 `00000000-0000-0000-0000-000000000000` |

### 6.1 "同一 URL 多次渲染" 场景的 URL 变化

假设 campaign 模板中 `https://example.com` 出现 3 次（TrackLink 显式、@TrackLink 简写、按钮链接），为两个不同订阅者 A 和 B 渲染时：

**IndividualTracking = true 时**：

| 订阅者 | 第1次出现 | 第2次出现 | 第3次出现 |
|--------|----------|----------|----------|
| A | `/link/abc-123/camp-456/sub-A/` | `/link/abc-123/camp-456/sub-A/` | `/link/abc-123/camp-456/sub-A/` |
| B | `/link/abc-123/camp-456/sub-B/` | `/link/abc-123/camp-456/sub-B/` | `/link/abc-123/camp-456/sub-B/` |

三个 URL 完全相同（同一订阅者内），因为 `linkUUID` 和 `campUUID` 相同，`subUUID` 也相同。

**IndividualTracking = false 时**：

| 订阅者 | 第1次出现 | 第2次出现 | 第3次出现 |
|--------|----------|----------|----------|
| A | `/link/abc-123/camp-456/00000000-0000-.../` | 同左 | 同左 |
| B | `/link/abc-123/camp-456/00000000-0000-.../` | 同左 | 同左 |

所有订阅者的 URL 完全一样。

**UnsubscribeURL / TrackView / MessageURL 不经过 TrackLink**：

```
UnsubscribeURL → /subscription/{campUUID}/{subUUID}
TrackView      → /campaign/{campUUID}/{subUUID}/px.png
MessageURL     → /campaign/{campUUID}/{subUUID}
```

这些 URL 使用不同的路由和处理逻辑，与 TrackLink 的 `/link/` 路由无关。

---

## 7. 阶段六：LinkRedirect — 用户点击时的处理

**入口**: [public.go LinkRedirect](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L530-L563)

```go
func (a *App) LinkRedirect(c echo.Context) error {
    linkUUID := c.Param("linkUUID")       // links.uuid
    campUUID := c.Param("campUUID")       // campaigns.uuid
    subUUID  := c.Param("subUUID")        // subscribers.uuid 或 dummyUUID

    // [1] 全局追踪禁用 → 只查 URL，不记录点击
    if a.cfg.Privacy.DisableTracking {
        url, err := a.core.GetLinkURL(linkUUID)
        // ... 错误处理
        return c.Redirect(http.StatusTemporaryRedirect, url)
    }

    // [2] 个人追踪禁用 → 清空 subUUID
    if !a.cfg.Privacy.IndividualTracking {
        subUUID = ""
    }

    // [3] 记录点击 + 获取原始 URL
    url, err := a.core.RegisterCampaignLinkClick(linkUUID, campUUID, subUUID)
    // ... 错误处理
    return c.Redirect(http.StatusTemporaryRedirect, url)
}
```

### 7.1 DisableTracking = true 时的行为

- 调用 [GetLinkURL](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L437-L446)，执行 `SELECT url FROM links WHERE uuid = $1`
- **不记录任何点击**
- 用户仍被正确重定向到原始 URL
- **注意**：即使 `DisableTracking=true`，邮件中如果已经生成了追踪 URL（在 `DisableTracking` 从 true 切换为 false 之前的 campaign），点击仍然会到达此 handler

### 7.2 IndividualTracking = false 时的行为

- `subUUID` 被设为空字符串 `""`
- 传给 `RegisterCampaignLinkClick` 的第三个参数是 `""`

### 7.3 错误处理

如果 `linkUUID` 在 `links` 表中不存在，`RegisterCampaignLinkClick` 返回错误，用户看到错误页面而非被重定向。**不会出现"跳转 URL 正确但统计缺失"的情况** — 要么两者都成功，要么都失败。

---

## 8. 阶段七：register-link-click SQL — 点击记录的写入

**SQL**: [links.sql register-link-click](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L8-L18)

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

### 8.1 subscriber_id 的三种情况

| $3 (subUUID) | CASE 条件 | 子查询结果 | subscriber_id |
|---------------|-----------|-----------|---------------|
| 真实 UUID | `subUUID != ''` → TRUE | `SELECT id FROM subscribers WHERE uuid = subUUID` | 正常的 subscriber ID |
| `dummyUUID` | `dummyUUID != ''` → TRUE | `SELECT id FROM subscribers WHERE uuid = '00000000-...'` | **NULL**（dummy subscriber 不存在于 DB） |
| `""` (空串) | `"" != ''` → FALSE | FALSE 条件，子查询返回 NULL | **NULL** |

### 8.2 "link_clicks 没有 subscriber_id" 的根因分析

**情况 A**：`IndividualTracking = false`，且 `DisableTracking = false`

- 模板渲染阶段：`subUUID = dummyUUID`，追踪 URL 中包含 `00000000-0000-0000-0000-000000000000`
- 点击阶段：`LinkRedirect` 将 `subUUID` 设为 `""`
- SQL 中 `$3 = ""`，CASE 走 FALSE 分支，`subscriber_id = NULL`
- **设计意图**：匿名统计，不追踪个人

**情况 B**：`IndividualTracking = false`，但 `DisableTracking` 从 true 切换为 false

- 如果 campaign 在 `DisableTracking = true` 时创建并发送，邮件中不含追踪 URL
- 如果 campaign 在 `DisableTracking = true` 时创建，后改为 false 再发送，模板在发送时重新编译
- **不会出现"URL 中有 dummyUUID 但 LinkRedirect 不清空"的情况**，因为 LinkRedirect 始终会根据当前配置决定是否清空 subUUID

**情况 C**：`IndividualTracking = true`，但 subscriber 已被删除

- `subUUID` 是真实 UUID，但 `subscribers` 表中已无此记录
- 子查询 `SELECT id FROM subscribers WHERE uuid = $3` 返回空
- `subscriber_id = NULL`
- 这正是 `link_clicks` 表外键定义为 `ON DELETE SET NULL` 的原因

### 8.3 "跳转 URL 正确但统计缺失" 的可能原因

1. **`DisableTracking = true`**：LinkRedirect 只查 URL 不记录点击，但用户仍被正确重定向
2. **campaign UUID 无效**：`linkUUID` 有效（查到 link），但 `campUUID` 对应的 campaign 不存在
   - 此时 `campaign_id` 子查询返回 NULL
   - `link_clicks.campaign_id` 是 `INTEGER NULL`（允许 NULL）
   - 记录会插入但 `campaign_id = NULL`，在按 campaign 统计时会被遗漏
3. **并发竞态**：极端情况下，`links` 表中的记录被删除后、点击到达前，`linkUUID` 查询失败

---

## 9. 阶段八：统计查询 — privacy 配置对聚合的影响

**入口**: [init.go prepareQueries](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L400-L429)

```go
func prepareQueries(qMap goyesql.Queries, db *sqlx.DB, ko *koanf.Koanf) *models.Queries {
    countQuery = "get-campaign-analytics-counts"
    linkSel    = "*"
    if ko.Bool("privacy.individual_tracking") {
        countQuery = "get-campaign-analytics-unique-counts"
        linkSel = "DISTINCT subscriber_id"
    }

    qMap["get-campaign-link-counts"].Query = fmt.Sprintf(
        qMap["get-campaign-link-counts"].Query, linkSel)
    // ...
}
```

**SQL 模板** ([campaigns.sql get-campaign-link-counts](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L269-L274))：

```sql
-- raw: true
-- %s = * or DISTINCT subscriber_id
SELECT COUNT(%s) AS "count", url
    FROM link_clicks
    LEFT JOIN links ON (link_clicks.link_id = links.id)
    WHERE campaign_id=ANY($1) AND link_clicks.created_at >= $2 AND link_clicks.created_at <= $3
    GROUP BY links.url ORDER BY "count" DESC LIMIT 50;
```

### 9.1 IndividualTracking 对统计查询的影响

| IndividualTracking | linkSel | 实际 SQL | 统计含义 |
|--------------------|---------|----------|----------|
| `false` (默认) | `*` | `COUNT(*)` | **总点击次数**（含同一用户重复点击） |
| `true` | `DISTINCT subscriber_id` | `COUNT(DISTINCT subscriber_id)` | **独立点击用户数** |

### 9.2 IndividualTracking = true 时 COUNT(DISTINCT subscriber_id) 的陷阱

当 `IndividualTracking = true`：
- `link_clicks.subscriber_id` 记录了真实 ID
- `COUNT(DISTINCT subscriber_id)` 统计的是有多少**不同用户**点击了该链接
- 但 `subscriber_id` 可以为 NULL（用户被删除后 ON DELETE SET NULL）
- `COUNT(DISTINCT subscriber_id)` **不计算 NULL 值**
- 因此被删除用户的点击记录不会被计入"独立用户"统计

### 9.3 IndividualTracking = false 时 COUNT(*) 的含义

- 所有 `subscriber_id` 都是 NULL
- `COUNT(*)` 统计的是**总点击次数**
- 无法区分"10 个人各点1次"和"1 个人点了10次"

### 9.4 链接去重对统计的影响

由于 `links` 表以 URL 去重（`url UNIQUE`），同一原始 URL 在所有 campaign 中共享同一个 `link_id`。统计查询通过 `LEFT JOIN links ON (link_clicks.link_id = links.id)` 关联原始 URL，然后按 `links.url` 分组。

**跨 Campaign 聚合**：如果传入多个 `campaign_id`，同一 URL 在不同 campaign 中的点击会合并统计。

**重要**：`links` 表的去重是**严格 URL 匹配**。以下两个 URL 会被视为不同的链接：
- `https://example.com/page`
- `https://example.com/page?utm_source=newsletter`

即使是同一页面，只要 URL 字符串不同，就会产生不同的 `linkUUID`，在统计中显示为不同条目。

---

## 10. 综合问题分析

### 10.1 "同一 URL 被多次渲染"

**现象**：某封 campaign 邮件中同一 URL 以 TrackLink 简写、ViewURL 和 UnsubscribeURL 形式同时出现。

**分析**：

| URL 类型 | 经过 TrackLink? | 生成追踪 URL? | 点击记录位置 |
|----------|-----------------|--------------|-------------|
| TrackLink / @TrackLink | ✅ 是 | ✅ `/link/...` | `link_clicks` |
| TrackView | ❌ 否 | `/campaign/.../px.png` | `campaign_views` |
| UnsubscribeURL | ❌ 否 | `/subscription/...` | `subscriber_lists` 状态变更 |
| MessageURL | ❌ 否 | `/campaign/...` | 无追踪 |

**结论**：TrackLink、TrackView、UnsubscribeURL 是三个完全独立的模板函数，不会互相影响。用户看到"同一 URL 多次出现"可能是因为邮件模板中同时使用了：
- `<a href="{{ TrackLink "https://example.com" }}">了解更多</a>` — 点击追踪
- `{{ UnsubscribeURL }}` — 退订链接（指向不同的路由）
- `{{ TrackView }}` — 打开追踪（不是链接，是像素图）

它们虽然都涉及 URL，但**不是同一个 URL 的多次渲染**，而是不同功能的独立 URL。

### 10.2 "link_clicks 没有 subscriber_id"

**根因**：由两个独立但互补的机制产生：

**机制 1 — 模板渲染时** ([TemplateFuncs](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L344-L349))：
```go
subUUID := msg.Subscriber.UUID
if !m.cfg.IndividualTracking {
    subUUID = dummyUUID  // "00000000-0000-0000-0000-000000000000"
}
```
当 `IndividualTracking=false` 时，URL 中嵌入 dummyUUID。

**机制 2 — 点击处理时** ([LinkRedirect](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L551-L554))：
```go
subUUID := c.Param("subUUID")
if !a.cfg.Privacy.IndividualTracking {
    subUUID = ""  // 清空
}
```
当 `IndividualTracking=false` 时，URL 中的 dummyUUID 被清空。

**机制 3 — SQL 写入时** ([register-link-click](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L14-L15))：
```sql
(CASE WHEN $3::TEXT != '' THEN subscribers.uuid = $3::UUID ELSE FALSE END)
```
空字符串 → FALSE → 子查询返回 NULL → `subscriber_id = NULL`。

**双重保障**：即使 URL 中包含 dummyUUID（机制1未能清空），机制2在服务端仍会清空。但注意：
- 如果 `IndividualTracking=true` 时发送了邮件，后改为 `false`，URL 中已有真实 UUID
- 点击时机制2会清空 subUUID，**subscriber_id 仍为 NULL**
- 这意味着**配置变更期间**的点击记录可能不一致

### 10.3 "跳转 URL 正确但统计缺失"

| 场景 | 跳转 | 统计 | 原因 |
|------|------|------|------|
| `DisableTracking = true` | ✅ 正确 | ❌ 无记录 | LinkRedirect 不调用 RegisterCampaignLinkClick |
| `campUUID` 无效 | ❌ 错误页 | ❌ 无记录 | SQL 中 campaign_id 子查询返回 NULL，INSERT 可能因约束失败 |
| `linkUUID` 无效 | ❌ 错误页 | ❌ 无记录 | CTE 中 link 查询返回空，INSERT 失败 |
| `subUUID` 无效（已被删除）| ✅ 正确 | ⚠️ 记录存在但 subscriber_id=NULL | 外键 ON DELETE SET NULL |
| 配置从 DisableTracking=true 切为 false | ✅ 正确 | ⚠️ 部分缺失 | 旧邮件中的 URL 是原始 URL，不经过追踪路由 |

---

## 11. 关键配置影响总结

### 11.1 privacy.disable_tracking

| 维度 | `false`（默认） | `true` |
|------|-----------------|--------|
| 模板渲染 | 生成追踪 URL | 返回原始 URL |
| links 表 | 记录 URL | 不记录 |
| 点击重定向 | 记录点击 + 重定向 | 仅重定向 |
| 统计数据 | 有 | 无 |
| 退订链接 | 不受影响 | 不受影响 |

### 11.2 privacy.individual_tracking

| 维度 | `false`（默认） | `true` |
|------|-----------------|--------|
| URL 中的 subUUID | dummyUUID → 被清空 | 真实 UUID |
| link_clicks.subscriber_id | NULL | 真实 ID（用户存在时） |
| 统计查询 | `COUNT(*)` — 总点击次数 | `COUNT(DISTINCT subscriber_id)` — 独立用户数 |
| 同一用户多次点击 | 每次都计数 | 只计1次（DISTINCT） |
| 用户删除后 | subscriber_id 本身就是 NULL | ON DELETE SET NULL → 变为 NULL |

### 11.3 链接去重（links 表 ON CONFLICT）

| 维度 | 影响 |
|------|------|
| 同一 URL 跨 Campaign | 共享 linkUUID，但 link_clicks 中 campaign_id 不同 |
| URL 微小差异 | 视为不同链接（严格字符串匹配） |
| `&amp;` vs `&` | trackLink 中有 `strings.ReplaceAll` 还原，保证一致性 |
| 内存缓存 | 同一 Manager 实例生命周期内，URL → UUID 只查一次 DB |

### 11.4 Campaign UUID 与 Subscriber UUID

| UUID | 作用 | 缺失后果 |
|------|------|----------|
| campUUID | 在 register-link-click 中反查 campaign_id | campaign_id=NULL，统计遗漏 |
| subUUID | 在 register-link-click 中反查 subscriber_id | subscriber_id=NULL，个人追踪失效 |
| linkUUID | 查找原始 URL + link_id | 重定向失败 + 统计缺失 |

---

## 12. 核心代码参考索引

| 阶段 | 文件 | 行号 | 功能 |
|------|------|------|------|
| 正则替换 | [models/common.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/common.go#L43-L68) | L43-L68 | regTplFuncs 定义 |
| 模板编译 | [models/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L138-L242) | L138-L242 | CompileTemplate |
| TrackLink 闭包 | [manager/manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L338-L399) | L338-L399 | TemplateFuncs |
| trackLink 函数 | [manager/manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L582-L613) | L582-L613 | trackLink |
| dummyUUID | [manager/manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L30) | L30 | dummyUUID 常量 |
| CreateLink | [manager_store.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/manager_store.go#L106-L122) | L106-L122 | CreateLink |
| LinkRedirect | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L530-L563) | L530-L563 | LinkRedirect |
| RegisterCampaignLinkClick | [campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L448-L461) | L448-L461 | RegisterCampaignLinkClick |
| create-link SQL | [links.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L2-L3) | L2-L3 | INSERT ON CONFLICT |
| register-link-click SQL | [links.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/links.sql#L8-L18) | L8-L18 | INSERT INTO link_clicks |
| get-campaign-link-counts SQL | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L269-L274) | L269-L274 | 动态统计查询 |
| prepareQueries | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L400-L429) | L400-L429 | 根据隐私配置准备查询 |
| LinkTrackURL 定义 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L473) | L473 | URL 格式模板 |
| 路由注册 | [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L277) | L277 | /link/:linkUUID/:campUUID/:subUUID |
| NewCampaignMessage | [message.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/message.go#L13-L29) | L13-L29 | 消息创建与渲染 |
| link_clicks 表定义 | [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L206-L219) | L206-L219 | 含 ON DELETE SET NULL |
| links 表定义 | [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L198-L204) | L198-L204 | 含 url UNIQUE |
| Privacy 配置 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L82-L100) | L82-L100 | Config.Privacy 结构体 |
| initCampaignManager | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L590-L620) | L590-L620 | Manager 初始化与配置传递 |
