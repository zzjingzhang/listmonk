# Campaign 归档功能安全漏洞深度分析

## 1. 归档数据生成路径总览

### 1.1 公开访问入口

| Endpoint | Handler | 描述 |
|----------|---------|------|
| `GET /archive` | [CampaignArchivesPage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L98-L116) | HTML 归档列表页 |
| `GET /archive.xml` | [GetCampaignArchivesFeed](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L52-L96) | RSS Feed |
| `GET /archive/:id` | [CampaignArchivePage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L118-L174) | 单篇归档详情页 |
| `GET /archive/latest` | [CampaignArchivePageLatest](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L176-L191) | 最新归档详情 |
| `GET /api/public/archive` | [GetCampaignArchives](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L26-L50) | JSON API |

路由注册于 [handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go#L281-L286)，所有公开端点均无认证保护。

### 1.2 数据流向图

```
HTTP Request
    ↓
archive.go handlers (5个公开端点)
    ↓
├─ getCampaignArchives() → GetArchivedCampaigns() (列表)
│                          [SQL: get-archived-campaigns]
│
└─ CampaignArchivePage() → GetArchivedCampaign() (单个)
                           [SQL: get-campaign]
    ↓
compileArchiveCampaigns()
    ├─ CompileTemplate()  [编译 campaign 模板]
    └─ json.Unmarshal(ArchiveMeta) → dummy subscriber
    ↓
NewCampaignMessage()
    ├─ 渲染模板 body
    ├─ 生成 TrackLink / TrackView / UnsubscribeURL
    └─ 返回渲染后的 HTML
```

---

## 2. Archive Handlers 深度分析

### 2.1 列表查询 - getCampaignArchives

**位置**: [archive.go:193-236](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L193-L236)

```go
func (a *App) getCampaignArchives(offset, limit int, renderBody bool) ([]campArchive, int, error) {
    pubCamps, total, err := a.core.GetArchivedCampaigns(offset, limit)
    // ...
    msgs, err := a.compileArchiveCampaigns(pubCamps)
    // ...
    if renderBody {
        msg, err := a.manager.NewCampaignMessage(camp, m.Subscriber)
        archive.Content = string(msg.Body())
    }
}
```

**调用链**:
- `GetArchivedCampaigns` → SQL 查询 `get-archived-campaigns`
- `compileArchiveCampaigns` → 编译模板 + 反序列化 ArchiveMeta
- `NewCampaignMessage` → 最终渲染（仅 RSS 和 latest endpoint 需要）

### 2.2 单个查询 - CampaignArchivePage

**位置**: [archive.go:118-174](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L118-L174)

```go
func (a *App) CampaignArchivePage(c echo.Context) error {
    pubCamp, err := a.core.GetArchivedCampaign(0, uuid, slug)
    // 只检查了 Type != CampaignTypeRegular，没有检查 status!
    
    out, err := a.compileArchiveCampaigns([]models.Campaign{pubCamp})
    msg, err := a.manager.NewCampaignMessage(camp, out[0].Subscriber)
    
    return c.HTML(http.StatusOK, string(msg.Body()))
}
```

**关键问题**: 仅检查了 `Type == regular`，但**没有检查 campaign 的 status**。

### 2.3 最新归档 - CampaignArchivePageLatest

**位置**: [archive.go:176-191](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L176-L191)

```go
func (a *App) CampaignArchivePageLatest(c echo.Context) error {
    camps, _, err := a.getCampaignArchives(0, 1, true)
    // 直接输出渲染后的 HTML
    return c.HTML(http.StatusOK, camp.Content)
}
```

此端点调用 `getCampaignArchives(0, 1, true)`，使用列表查询路径。

---

## 3. Core Archived Campaign 查询逻辑

### 3.1 GetArchivedCampaigns (列表查询)

**位置**: [core/campaigns.go:145-160](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L145-L160)

**SQL**: [queries/campaigns.sql:99-108](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L99-L108)

```sql
SELECT COUNT(*) OVER () AS total, campaigns.*,
    COALESCE(templates.body, ...) AS template_body
FROM campaigns
LEFT JOIN templates ON (
    CASE WHEN $3 = 'default' THEN templates.id = campaigns.template_id
    ELSE templates.id = campaigns.archive_template_id END
)
WHERE campaigns.archive=true 
  AND campaigns.type='regular' 
  AND campaigns.status=ANY('{running, paused, finished}')  -- 关键过滤
ORDER by campaigns.created_at DESC OFFSET $1 LIMIT $2;
```

**过滤条件**:
- `archive = true`
- `type = 'regular'`
- `status IN ('running', 'paused', 'finished')` ✓ **正确**

### 3.2 GetArchivedCampaign (单个查询)

**位置**: [core/campaigns.go:72-85](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L72-L85)

```go
func (c *Core) GetArchivedCampaign(id int, uuid, archiveSlug string) (models.Campaign, error) {
    out, err := c.getCampaign(id, uuid, archiveSlug, campaignTplArchive)
    if err != nil { return out, err }
    
    if !out.Archive {  // 只检查了 archive 标志
        return models.Campaign{}, echo.NewHTTPError(...)
    }
    
    return out, nil  // ⚠️ 没有检查 status!
}
```

**调用的 SQL**: [queries/campaigns.sql:85-97](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L85-L97)

```sql
SELECT campaigns.*,
    COALESCE(templates.body, ...) AS template_body
FROM campaigns
LEFT JOIN templates ON (
    CASE WHEN $4 = 'default' THEN templates.id = campaigns.template_id
    ELSE templates.id = campaigns.archive_template_id END
)
WHERE CASE
        WHEN $1 > 0 THEN campaigns.id = $1
        WHEN $3 != '' THEN campaigns.archive_slug = $3
        ELSE uuid = $2
      END;
-- ⚠️ 完全没有 status 和 type 过滤!
```

**过滤条件**:
- 无 status 过滤 ✗ **严重缺陷**
- 无 type 过滤（handler 层补了检查）
- 仅在 Go 层检查了 `Archive = true`

### 3.3 缺陷对比

| 查询函数 | SQL 过滤 | Go 层过滤 | 能否泄露 draft |
|----------|----------|-----------|----------------|
| `GetArchivedCampaigns` | `status IN ('running','paused','finished')` | 无 | 否 ✓ |
| `GetArchivedCampaign` | 无 status 过滤 | 仅 `archive=true` | **是** ✗ |

**攻击路径**:
1. 用户创建一个 draft campaign，勾选 "Enable archive"
2. 获得 campaign 的 UUID 或设置 archive_slug
3. 直接访问 `https://example.com/archive/{uuid-or-slug}`
4. **无需任何认证即可看到 draft 内容**

---

## 4. Campaign Status/Type 过滤分析

### 4.1 Campaign 状态定义

**位置**: [models/campaigns.go:18-24](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L18-L24)

```go
const (
    CampaignStatusDraft     = "draft"
    CampaignStatusScheduled = "scheduled"
    CampaignStatusRunning   = "running"
    CampaignStatusPaused    = "paused"
    CampaignStatusFinished  = "finished"
    CampaignStatusCancelled = "cancelled"
)
```

### 4.2 状态流转

**位置**: [core/campaigns.go:256-286](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L256-L286)

```go
switch status {
case models.CampaignStatusDraft:
    // 只能从 scheduled 转为 draft
case models.CampaignStatusScheduled:
    // 只能从 draft 或 paused 转为 scheduled
case models.CampaignStatusRunning:
    // 只能从 paused 或 draft 转为 running
// ...
}
```

### 4.3 Archive 标志的设置时机

Archive 标志可以在**任何状态**下设置：
- 创建 campaign 时勾选 (create-campaign SQL)
- 更新 campaign 时修改 (update-campaign SQL)
- 单独通过 `/api/campaigns/:id/archive` 端点修改

**问题**: Archive 标志与 Status 是两个独立字段，**没有联动校验**。用户可以在 draft 状态下启用 archive。

---

## 5. Template 编译和 Public Message 模板分析

### 5.1 compileArchiveCampaigns

**位置**: [archive.go:238-277](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L238-L277)

```go
func (a *App) compileArchiveCampaigns(camps []models.Campaign) ([]manager.CampaignMessage, error) {
    for _, c := range camps {
        camp := c
        // 1. 编译模板
        if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil { ... }
        
        // 2. 反序列化 ArchiveMeta 为 subscriber
        var sub models.Subscriber
        if err := json.Unmarshal([]byte(camp.ArchiveMeta), &sub); err != nil { ... }
        
        // 3. 构建 CampaignMessage
        m := manager.CampaignMessage{
            Campaign:   &camp,
            Subscriber: sub,
        }
        
        // 4. 编译 subject 模板
        if camp.SubjectTpl != nil {
            if err := camp.SubjectTpl.ExecuteTemplate(&b, models.ContentTpl, m); err != nil { ... }
            camp.Subject = b.String()
        }
        
        out = append(out, m)
    }
    return out, nil
}
```

### 5.2 ArchiveMeta 的来源

`ArchiveMeta` 是在 campaign **发送时**保存的第一个订阅者的 JSON 序列化数据。

**问题**:
- 如果 campaign 从未发送过（如 draft），`ArchiveMeta` 可能为空或不完整
- 如果 campaign 已发送，`ArchiveMeta` 包含**真实订阅者的 UUID**

### 5.3 NewCampaignMessage 与链接追踪变量

**位置**: [manager/message.go:10-29](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/message.go#L10-L29)

```go
func (m *Manager) NewCampaignMessage(c *models.Campaign, s models.Subscriber) (CampaignMessage, error) {
    msg := CampaignMessage{
        Campaign:   c,
        Subscriber: s,
        subject:  c.Subject,
        from:     c.FromEmail,
        to:       s.Email,
        unsubURL: fmt.Sprintf(m.cfg.UnsubURL, c.UUID, s.UUID),  // ⚠️ 使用 s.UUID
    }
    if err := msg.render(); err != nil { return msg, err }
    return msg, nil
}
```

### 5.4 模板函数中的变量替换

**位置**: [manager/manager.go:347-399](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L347-L399)

```go
func (m *Manager) TemplateFuncs(c *models.Campaign) template.FuncMap {
    return template.FuncMap{
        "TrackLink": func(url string, msg *CampaignMessage) string {
            subUUID := msg.Subscriber.UUID
            if !m.cfg.IndividualTracking { subUUID = dummyUUID }
            return m.trackLink(url, msg.Campaign.UUID, subUUID)  // ⚠️ 使用真实 UUID
        },
        
        "TrackView": func(msg *CampaignMessage) template.HTML {
            subUUID := msg.Subscriber.UUID
            if !m.cfg.IndividualTracking { subUUID = dummyUUID }
            return template.HTML(fmt.Sprintf(`<img src="%s" />`,
                fmt.Sprintf(m.cfg.ViewTrackURL, msg.Campaign.UUID, subUUID)))
        },
        
        "UnsubscribeURL": func(msg *CampaignMessage) string {
            return msg.unsubURL  // ⚠️ 来自 NewCampaignMessage，包含 s.UUID
        },
        
        "MessageURL": func(msg *CampaignMessage) string {
            return fmt.Sprintf(m.cfg.MessageURL, c.UUID, msg.Subscriber.UUID)
        },
        // ...
    }
}
```

### 5.5 链接追踪/退订变量泄露问题

**场景**: Campaign 已发送，`ArchiveMeta` 保存了订阅者 A 的数据（UUID = `sub-a-123`）

当任意用户访问归档页面时：
1. `ArchiveMeta` 被反序列化为 subscriber，`UUID = sub-a-123`
2. `NewCampaignMessage` 生成 `unsubURL = /subscription/{campUUID}/sub-a-123`
3. 模板中的 `{{ UnsubscribeURL }}` 被替换为包含订阅者 A UUID 的 URL
4. **任何访问归档页面的用户都可以点击该链接取消订阅者 A 的订阅**

**泄露的数据**:
- 真实订阅者的 UUID
- 可操作的退订链接（CSRF 风险）
- 可操作的管理偏好链接 (`ManageURL`)
- 追踪链接中的订阅者 UUID

---

## 6. 与 PreviewCampaign、ViewCampaignMessage 的差异对比

### 6.1 核心差异对照表

| 对比项 | Archive 流程 | PreviewCampaign | ViewCampaignMessage |
|--------|-------------|-----------------|---------------------|
| **权限** | 完全公开 | 需要认证 (`campaigns:get`) | 公开但需正确 UUID |
| **Handler** | [archive.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go) | [campaigns.go:136-196](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L136-L196) | [public.go:146-193](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L146-L193) |
| **Status 过滤** | 列表: 有<br>单个: **无** | 无（认证后可预览所有） | 无（但需两个 UUID） |
| **Campaign UUID** | 真实 UUID | **替换为 dummyUUID** | 真实 UUID |
| **Subscriber 来源** | `ArchiveMeta` 反序列化 | 固定 `dummySubscriber` | DB 查询真实用户 |
| **Subscriber UUID** | 发送时的真实用户 | `00000000-0000-...` | 真实用户 UUID |
| **追踪链接泄露** | 高（真实 UUID） | 低（dummy UUID） | 低（仅本人可见） |
| **退订链接风险** | 高（可操作他人） | 低（dummy UUID） | 低（仅本人） |

### 6.2 PreviewCampaign 的安全措施

**位置**: [campaigns.go:136-196](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L136-L196)

```go
func (a *App) PreviewCampaign(c echo.Context) error {
    // 1. 权限检查
    if err := a.checkCampaignPerm(auth.PermTypeGet, id, c); err != nil { return err }
    
    // 2. 关键安全措施: 替换为 dummy UUID
    camp.UUID = dummySubscriber.UUID  // "00000000-0000-..."
    
    // 3. 编译模板
    if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil { ... }
    
    // 4. 使用固定 dummySubscriber
    msg, err := a.manager.NewCampaignMessage(&camp, dummySubscriber)
    
    return c.HTML(http.StatusOK, string(msg.Body()))
}
```

**dummySubscriber 定义**: [subscribers.go:49-55](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L49-L55)

```go
var dummySubscriber = models.Subscriber{
    Email:   "demo@listmonk.app",
    Name:    "Demo Subscriber",
    UUID:    dummyUUID,  // "00000000-0000-0000-0000-000000000000"
    Attribs: models.JSON{"city": "Bengaluru"},
}
```

### 6.3 ViewCampaignMessage 的安全机制

**位置**: [public.go:146-193](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L146-L193)

```go
func (a *App) ViewCampaignMessage(c echo.Context) error {
    campUUID := c.Param("campUUID")
    subUUID := c.Param("subUUID")
    
    // 需要同时知道 campUUID 和 subUUID
    camp, err := a.core.GetCampaign(0, campUUID, "")
    sub, err := a.core.GetSubscriber(0, subUUID, "")
    
    if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil { ... }
    msg, err := a.manager.NewCampaignMessage(&camp, sub)
    
    return c.HTML(http.StatusOK, string(msg.Body()))
}
```

**安全机制**:
- 需要同时知道 campaign UUID 和 subscriber UUID
- 每个订阅者看到的是包含自己 UUID 的链接
- 不能通过猜测 UUID 访问他人的视图（需同时猜对两个 UUID）

---

## 7. 漏洞总结与风险评估

### 7.1 漏洞 1: Draft 内容泄露（高危）

**影响范围**: `/archive/:id` 端点

**漏洞描述**:
- `GetArchivedCampaign` 仅检查 `archive=true`，未检查 `status`
- 只要 campaign 设置了 `archive=true`，无论状态是 draft/scheduled/cancelled，都可以通过 UUID 或 slug 直接访问
- 列表页虽然不会显示 draft，但直接访问详情页可以绕过

**攻击场景**:
1. 攻击者知道/猜测到 campaign UUID 或 slug
2. 直接访问 `https://example.com/archive/{uuid}`
3. 获得未发布的 draft 内容

**CVSS 评分**: 7.5 (高危)
- 攻击复杂度: 低（只需知道 UUID）
- 影响范围: 机密性（泄露未发布内容）

### 7.2 漏洞 2: 订阅者 UUID 泄露（高危）

**影响范围**: 所有归档端点，尤其 `/archive/:id` 和 `/archive/latest`

**漏洞描述**:
- 归档页面使用 `ArchiveMeta` 中保存的真实订阅者数据
- 模板中 `{{ TrackLink }}`、`{{ TrackView }}`、`{{ UnsubscribeURL }}`、`{{ MessageURL }}` 都会包含该订阅者的真实 UUID
- 所有访问归档页面的用户都能看到这些链接

**攻击场景**:
1. 攻击者访问归档页面
2. 查看 HTML 源代码，提取链接中的订阅者 UUID
3. 结合 `/subscription/:campUUID/:subUUID` 端点，可以：
   - 取消该订阅者的订阅
   - 修改其订阅偏好
   - 查看其订阅详情

**CVSS 评分**: 8.1 (高危)
- 攻击复杂度: 低
- 影响范围: 机密性（泄露用户 UUID）+ 完整性（可操作账户）

### 7.3 漏洞 3: 未发送 Campaign 的 ArchiveMeta 为空（中危）

**影响范围**: 从未发送过的 campaign（draft/scheduled）

**漏洞描述**:
- 如果 campaign 从未发送，`ArchiveMeta` 可能为空或无效 JSON
- `json.Unmarshal` 会失败，返回 500 错误
- 或者反序列化为空 subscriber，导致模板渲染异常

**风险**: 可能导致模板变量未正确替换，显示原始模板代码 `{{ ... }}`

---

## 8. 修复建议

### 8.1 修复 Draft 泄露：GetArchivedCampaign 添加 status 过滤

**修改位置**: [core/campaigns.go:72-85](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L72-L85)

```go
func (c *Core) GetArchivedCampaign(id int, uuid, archiveSlug string) (models.Campaign, error) {
    out, err := c.getCampaign(id, uuid, archiveSlug, campaignTplArchive)
    if err != nil {
        return out, err
    }

    if !out.Archive {
        return models.Campaign{}, echo.NewHTTPError(http.StatusBadRequest,
            c.i18n.Ts("globals.messages.notFound", "name", "{globals.terms.campaign}"))
    }
    
    // 新增: 检查 status，只允许 running/paused/finished
    if out.Status != models.CampaignStatusRunning && 
       out.Status != models.CampaignStatusPaused && 
       out.Status != models.CampaignStatusFinished {
        return models.Campaign{}, echo.NewHTTPError(http.StatusBadRequest,
            c.i18n.Ts("globals.messages.notFound", "name", "{globals.terms.campaign}"))
    }

    return out, nil
}
```

### 8.2 修复 1b: SQL 层面统一过滤（深度防御）

**修改位置**: [queries/campaigns.sql:85-97](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L85-L97)

```sql
SELECT campaigns.*,
    COALESCE(templates.body, (SELECT body FROM templates WHERE is_default = true LIMIT 1), '') AS template_body
FROM campaigns
LEFT JOIN templates ON (
    CASE WHEN $4 = 'default' THEN templates.id = campaigns.template_id
    ELSE templates.id = campaigns.archive_template_id END
)
WHERE CASE
        WHEN $1 > 0 THEN campaigns.id = $1
        WHEN $3 != '' THEN campaigns.archive_slug = $3
        ELSE uuid = $2
      END
  -- 新增: 如果是 archive 查询，添加 status 过滤
  AND ($4 != 'archive' OR campaigns.status = ANY('{running, paused, finished}'));
```

### 8.3 修复订阅者 UUID 泄露：使用 dummy subscriber

**参考 PreviewCampaign 的实现**，修改 [archive.go:238-277](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L238-L277)

```go
func (a *App) compileArchiveCampaigns(camps []models.Campaign) ([]manager.CampaignMessage, error) {
    var (
        b   = bytes.Buffer{}
        out = make([]manager.CampaignMessage, 0, len(camps))
    )
    for _, c := range camps {
        camp := c
        
        // 关键修复 1: 替换 campaign UUID 为 dummyUUID，防止追踪统计污染
        originalUUID := camp.UUID
        camp.UUID = dummySubscriber.UUID
        
        if err := camp.CompileTemplate(a.manager.TemplateFuncs(&camp)); err != nil {
            // ...
        }
        
        // 恢复 UUID 用于生成归档 URL
        camp.UUID = originalUUID
        
        // 关键修复 2: 使用固定 dummySubscriber，而不是反序列化 ArchiveMeta
        // ArchiveMeta 仅用于保留用户自定义的属性用于模板渲染
        var sub models.Subscriber
        if len(camp.ArchiveMeta) > 0 {
            if err := json.Unmarshal([]byte(camp.ArchiveMeta), &sub); err != nil {
                // ...
            }
        }
        
        // 关键修复 3: 强制覆盖 UUID 为 dummyUUID
        sub.UUID = dummySubscriber.UUID
        sub.Email = dummySubscriber.Email
        sub.Name = dummySubscriber.Name
        
        m := manager.CampaignMessage{
            Campaign:   &camp,
            Subscriber: sub,
        }
        
        // ... subject 编译逻辑保持不变
        
        out = append(out, m)
    }
    
    return out, nil
}
```

### 8.4 修复 4: ArchiveMeta 保存时使用 dummy 数据

在 campaign 发送完成后更新 `ArchiveMeta` 时，应该保存 dummy 数据而不是真实订阅者数据。

或者，在归档渲染时完全忽略 `ArchiveMeta`，始终使用 `dummySubscriber`。

### 8.5 修复 5: Handler 层深度防御

在 [CampaignArchivePage](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go#L118-L174) 增加二次校验：

```go
func (a *App) CampaignArchivePage(c echo.Context) error {
    pubCamp, err := a.core.GetArchivedCampaign(0, uuid, slug)
    if err != nil || pubCamp.Type != models.CampaignTypeRegular {
        // ... 原有逻辑
    }
    
    // 新增: 二次校验 status（深度防御）
    if pubCamp.Status != models.CampaignStatusRunning && 
       pubCamp.Status != models.CampaignStatusPaused && 
       pubCamp.Status != models.CampaignStatusFinished {
        return c.Render(http.StatusNotFound, tplMessage,
            makeMsgTpl(a.i18n.T("public.notFoundTitle"), "", a.i18n.T("public.campaignNotFound")))
    }
    
    // ... 后续逻辑
}
```

### 8.6 修复 6: Archive 标志与 Status 联动校验

在设置 `archive=true` 时，校验 campaign 必须处于合适的状态：

```go
// 在 UpdateCampaignArchive 或 UpdateCampaign 中增加
if enabled && cm.Status == models.CampaignStatusDraft {
    return echo.NewHTTPError(http.StatusBadRequest, 
        a.i18n.T("campaigns.cannotArchiveDraft"))
}
```

---

## 9. 临时缓解措施

如果不能立即部署代码修复，可以采取以下临时措施：

1. **关闭公开归档功能**: 在设置中禁用 `EnablePublicArchive`
2. **数据库层面过滤**: 创建数据库视图或触发器，阻止 draft campaign 的 archive=true 设置
3. **WAF 规则**: 在 WAF 层面拦截包含明显 draft 特征的请求（需要业务规则）
4. **审计现有数据**: 检查是否有 draft campaign 设置了 `archive=true`:
   ```sql
   SELECT id, uuid, name, status, archive, archive_slug 
   FROM campaigns 
   WHERE archive=true AND status NOT IN ('running', 'paused', 'finished');
   ```

---

## 10. 相关文件索引

| 文件 | 说明 |
|------|------|
| [cmd/archive.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/archive.go) | 归档 handlers 主文件 |
| [internal/core/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go) | Core 层 campaign 查询逻辑 |
| [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | SQL 查询定义 |
| [internal/manager/message.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/message.go) | CampaignMessage 构造与渲染 |
| [internal/manager/manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go) | 模板函数定义 (TrackLink/UnsubscribeURL 等) |
| [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go) | PreviewCampaign 参考实现 |
| [cmd/public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | ViewCampaignMessage 参考实现 |
| [cmd/handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go) | 路由注册 |
| [models/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go) | Campaign 模型与状态定义 |
