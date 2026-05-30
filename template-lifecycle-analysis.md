# 模板生命周期分析：campaign / campaign_visual / tx

## 1. 模板类型常量定义

定义于 [models/templates.go#L12-L18](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/templates.go#L12-L18)：

| 常量 | 值 | 用途 |
|------|-----|------|
| `TemplateTypeCampaign` | `"campaign"` | 传统 HTML 包裹模板，需包含 `{{ template "content" . }}` 占位符 |
| `TemplateTypeCampaignVisual` | `"campaign_visual"` | 可视化编辑器模板，自包含完整 HTML，不设 `template_id` |
| `TemplateTypeTx` | `"tx"` | 事务性消息模板，有独立的 subject 字段 |

数据库枚举类型定义于 [schema.sql#L10](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L10)：

```sql
CREATE TYPE template_type AS ENUM ('campaign', 'campaign_visual', 'tx');
```

---

## 2. 数据库 Schema 与字段约束

表结构定义于 [schema.sql#L78-L90](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L78-L90)：

```sql
CREATE TABLE templates (
    id              SERIAL PRIMARY KEY,
    name            TEXT NOT NULL,
    type            template_type NOT NULL DEFAULT 'campaign',
    subject         TEXT NOT NULL,              -- ⚠ 无 DEFAULT，NOT NULL
    body            TEXT NOT NULL,
    body_source     TEXT NULL,                  -- 仅 campaign_visual 使用
    is_default      BOOLEAN NOT NULL DEFAULT false,
    created_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at      TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
CREATE UNIQUE INDEX ON templates (is_default) WHERE is_default = true;
```

### 字段约束汇总

| 字段 | DB 约束 | campaign | campaign_visual | tx |
|------|---------|----------|-----------------|-----|
| `name` | `NOT NULL` | 必填 | 必填 | 必填 |
| `type` | `NOT NULL DEFAULT 'campaign'` | 自动 | 自动 | 必填 |
| `subject` | `NOT NULL`（无默认值） | 允许空串 `""` | 允许空串 `""` | **必须非空** |
| `body` | `NOT NULL` | 必填 + 必含 `{{ template "content" . }}` | 必填（无内容验证） | 必填（无内容验证） |
| `body_source` | `NULL` | 不使用 | 可选（JSON 源） | 不使用 |
| `is_default` | `NOT NULL DEFAULT false` | 可设为默认 | **不可设为默认** | **不可设为默认** |

---

## 3. Go Handler 验证逻辑

### validateTemplate 函数

位于 [cmd/templates.go#L214-L230](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L214-L230)：

```go
func (a *App) validateTemplate(o models.Template) error {
    // 1. 所有类型：name 长度 1~stdInputMaxLen
    if !strHasLen(o.Name, 1, stdInputMaxLen) { ... }

    // 2. 仅 campaign 类型：body 必须包含 {{ template "content" . }}
    if o.Type == models.TemplateTypeCampaign && !regexpTplTag.MatchString(o.Body) { ... }

    // 3. 仅 tx 类型：subject 不能为空
    if o.Type == models.TemplateTypeTx && strings.TrimSpace(o.Subject) == "" { ... }

    return nil
}
```

### 验证规则差异表

| 验证项 | campaign | campaign_visual | tx |
|--------|----------|-----------------|-----|
| name 非空 | ✅ | ✅ | ✅ |
| body 含 `{{ template "content" . }}` | ✅ | ❌ 无验证 | ❌ 无验证 |
| subject 非空 | ❌ 不检查 | ❌ 不检查 | ✅ |
| body 非空 | 隐式（正则匹配要求） | ❌ **无验证** | ❌ **无验证** |

### Handler 中 subject 的强制清空

在 [CreateTemplate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L120-L121) 和 [UpdateTemplate](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L160-L161) 中：

```go
if o.Type == models.TemplateTypeCampaign || o.Type == models.TemplateTypeCampaignVisual {
    o.Subject = ""   // campaign 和 campaign_visual 的 subject 被强制清空
    funcs = a.manager.TemplateFuncs(nil)
} else {
    funcs = a.manager.GenericTemplateFuncs()  // tx 保留原始 subject
}
```

**设计意图**：campaign 的 subject 随每次活动变化，存储在 `campaigns.subject` 中；tx 的 subject 是模板固有的，存储在 `templates.subject` 中。

---

## 4. 核心 SQL 操作与 BUG 分析

### 4.1 create-template

位于 [queries/templates.sql#L11-L12](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L11-L12)：

```sql
INSERT INTO templates (name, type, subject, body, body_source) VALUES($1, $2, $3, $4, $5) RETURNING id;
```

行为正确，所有字段显式传入。

### 4.2 update-template — 存在 BUG

位于 [queries/templates.sql#L14-L21](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L14-L21)：

```sql
UPDATE templates SET
    name=(CASE WHEN $2 != '' THEN $2 ELSE name END),
    subject=(CASE WHEN $3 != '' THEN $3 ELSE name END),   -- ⚠ BUG: ELSE name 应为 ELSE subject
    body=(CASE WHEN $4 != '' THEN $4 ELSE body END),
    body_source=(CASE WHEN $5 != '' THEN $5 ELSE body_source END),
    updated_at=NOW()
WHERE id = $1;
```

**BUG 详情**：`subject` 的回退值是 `name` 而非 `subject`。其他字段（name/body/body_source）的回退模式均为 `ELSE <自身字段>`，唯独 subject 写成了 `ELSE name`。

**影响**：
- campaign/campaign_visual 模板更新时，Handler 传入 `subject = ""`，SQL 的 `$3 != ''` 为 false，导致 subject 被错误地设为模板的 name 值
- tx 模板更新时，因 subject 经 Go 验证后非空，`$3 != ''` 为 true，暂不受影响；但如果有人绕过 Go 验证直接调用 SQL，tx 的 subject 也会被错误覆盖

### 4.3 set-default-template — 类型限制

位于 [queries/templates.sql#L23-L27](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L23-L27)：

```sql
WITH u AS (
    UPDATE templates SET is_default=true WHERE id=$1 AND type='campaign' RETURNING id
)
UPDATE templates SET is_default=false WHERE id != $1;
```

**关键约束**：只有 `type='campaign'` 才能被设为默认模板。`campaign_visual` 和 `tx` 永远无法成为默认模板。

**潜在问题**：第二条 UPDATE 无条件执行，即使 CTE `u` 未匹配到任何行（即目标模板不是 campaign 类型），仍会将所有其他模板的 `is_default` 设为 false。这可能导致原本的默认模板丢失默认标记。

### 4.4 delete-template — 级联处理

位于 [queries/templates.sql#L29-L41](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L29-L41)：

```sql
WITH tpl AS (
    DELETE FROM templates WHERE id = $1
        AND (SELECT COUNT(id) FROM templates) > 1
        AND is_default = false
    RETURNING id
),
def AS (
    SELECT id FROM templates WHERE is_default = true
        AND (type='campaign' OR type='campaign_visual') LIMIT 1
),
up AS (
    UPDATE campaigns SET template_id = (SELECT id FROM def)
    WHERE (SELECT id FROM tpl) > 0 AND template_id = $1
)
SELECT id FROM tpl;
```

**删除保护**：
1. 只剩一个模板时不可删除（`COUNT > 1`）
2. 默认模板不可删除（`is_default = false`）

**级联重分配**：删除模板后，引用该模板的 campaign 会被重分配到默认模板。查找默认模板时允许 `type='campaign' OR type='campaign_visual'`。

**矛盾**：`set-default-template` 只允许 campaign 类型成为默认，但 `delete-template` 的重分配查找却包含 campaign_visual。由于 campaign_visual 永远不可能 `is_default = true`，这个 OR 条件永远不会匹配 campaign_visual 行，实质上无效。

---

## 5. Campaign 预览路径差异

### 5.1 模板预览（Template Preview）

入口：`GET /api/templates/:id/preview` 或 `POST /api/templates/preview`

实现位于 [cmd/templates.go#L232-L275](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/templates.go#L232-L275)：

```
┌──────────────────────────────────────────────────────────────┐
│                    模板预览路径                                │
├──────────────────┬───────────────────────────────────────────┤
│ campaign /       │ 1. 构造虚拟 Campaign 对象                  │
│ campaign_visual  │ 2. TemplateBody = tpl.Body                │
│                  │ 3. Body = dummyTpl（含 {{ template ... }}) │
│                  │ 4. camp.CompileTemplate()                 │
│                  │ 5. manager.NewCampaignMessage() → 渲染     │
├──────────────────┼───────────────────────────────────────────┤
│ tx               │ 1. tpl.Compile(GenericTemplateFuncs)      │
│                  │ 2. 构造 TxMessage{Subject: tpl.Subject}   │
│                  │ 3. m.Render(dummySubscriber, &tpl) → 渲染  │
└──────────────────┴───────────────────────────────────────────┘
```

**差异**：
- campaign/campaign_visual 走 Campaign 渲染管道，需要 `TemplateBody`（模板外壳）和 `Body`（内容体）两部分
- tx 走独立的 TxMessage 渲染管道，直接使用 `tpl.Body` + `tpl.Subject`

### 5.2 Campaign 预览（Campaign Preview）

入口：`GET /api/campaigns/:id/preview` 或 `POST /api/campaigns/:id/preview`

实现位于 [cmd/campaigns.go#L137-L196](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L137-L196)：

核心 SQL（`get-campaign-for-preview`）位于 [queries/campaigns.sql#L152-L163](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L152-L163)：

```sql
SELECT campaigns.*, COALESCE(templates.body, '') AS template_body
FROM campaigns
LEFT JOIN templates ON (
    templates.id = (CASE WHEN $2=0 THEN campaigns.template_id ELSE $2 END)
)
WHERE campaigns.id = $1;
```

**⚠️ 严重 BUG**：`COALESCE(templates.body, '')` — 当 campaign 的 `template_id` 为 NULL 时（模板被删除后 FK 约束 `ON DELETE SET NULL` 置空），`template_body` 变为空串 `''`，Campaign 编译模板时无外壳包裹，导致预览失败。

对比其他查询的正确回退逻辑：
- `get-campaign`（[campaigns.sql#L86-L97](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L86-L97)）：`COALESCE(templates.body, (SELECT body FROM templates WHERE is_default = true LIMIT 1), '')`
- `next-campaigns`（[campaigns.sql#L183](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L183)）：`COALESCE(templates.body, (SELECT body FROM templates WHERE is_default = true LIMIT 1), '')`

`get-campaign-for-preview` **缺少**到默认模板的回退！

### 5.3 Visual Campaign 的特殊预览逻辑

位于 [cmd/campaigns.go#L151-L170](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L151-L170)：

```go
// For visual content, template ID for previewing is irrelevant.
if contentType == models.CampaignContentTypeVisual || tplID < 1 {
    tplID = 0
}
// ...
if contentType == models.CampaignContentTypeVisual {
    camp.TemplateBody = ""
}
```

Visual 类型的 campaign 不需要模板外壳，`TemplateBody` 被清空，直接使用自身的 `Body` 作为完整 HTML。

---

## 6. Campaign 创建时的默认模板选择

SQL 位于 [queries/campaigns.sql#L4-L24](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L4-L24)：

```sql
WITH tpl AS (
    SELECT
        (CASE WHEN type = 'campaign_visual' THEN NULL ELSE id END) AS id,
        (CASE WHEN type = 'campaign_visual' THEN body ELSE '' END) AS body,
        (CASE WHEN type = 'campaign_visual' THEN body_source ELSE NULL END) AS body_source,
        (CASE WHEN type = 'campaign_visual' THEN 'visual' ELSE 'richtext' END) AS content_type
    FROM templates
    WHERE
        CASE
            WHEN $14::INT IS NOT NULL THEN id = $14::INT
            ELSE $8 != 'visual' AND is_default = TRUE
        END
    LIMIT 1
)
```

**选择逻辑**：
1. 若指定了 `template_id`（`$14`），直接按 ID 查找
2. 若未指定 `template_id` 且 `content_type != 'visual'`，查找 `is_default = TRUE` 的模板
3. Visual 类型 campaign 不需要从 DB 查找模板

**⚠️ 问题**：当默认的 campaign 模板被删除后，`is_default = TRUE` 无匹配行。此时：
- `tpl` CTE 返回空结果
- campaign 的 `body` 变为 `''`
- `template_id` 变为 `NULL`
- 新 campaign 创建成功但无模板内容

---

## 7. 默认模板安装数据

位于 [cmd/install.go#L190-L241](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/install.go#L190-L241)：

安装时创建 4 个模板：

| 序号 | 名称 | type | subject | is_default | 来源文件 |
|------|------|------|---------|------------|----------|
| 1 | Default campaign template | `campaign` | `""` | **true** | `default.tpl` |
| 2 | Default archive template | `campaign` | `""` | false | `default-archive.tpl` |
| 3 | Sample transactional template | `tx` | `"Welcome {{ .Subscriber.Name }}"` | false | `sample-tx.tpl` |
| 4 | Sample visual template | `campaign_visual` | `""` | false | `default-visual.tpl` + `default-visual.json` |

**关键观察**：
- 只有模板 1 被设为 `is_default = true`
- 模板 2（归档模板）也是 campaign 类型但不是默认的
- campaign_visual 模板不是默认模板，且不能被设为默认
- tx 模板有 subject，其他类型的 subject 为空串

### 默认模板模板内容

[campaign 类型模板 default.tpl](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/static/email-templates/default.tpl) 必须包含 `{{ template "content" . }}` 占位符，作为 campaign 内容的包裹外壳。Campaign 的 body 会被注入到该占位符位置。

---

## 8. 迁移历史中的模板类型演变

| 版本 | 迁移文件 | 变更 |
|------|----------|------|
| v2.2.0 | [migrations/v2.2.0.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/migrations/v2.2.0.go) | 新增 `template_type` 枚举（`campaign`, `tx`）；新增 `subject` 列（NOT NULL DEFAULT '' 后 DROP DEFAULT）；插入示例 tx 模板 |
| v5.0.0 | [migrations/v5.0.0.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/migrations/v5.0.0.go) | 新增 `campaign_visual` 枚举值；新增 `body_source` 列；FK 约束改为 `ON DELETE SET NULL`；插入示例 visual 模板 |

**注意**：v5.0.0 将 `template_id` 的 FK 约束从默认行为改为 `ON DELETE SET NULL`。这意味着删除模板后，campaign 的 `template_id` 会被置为 NULL 而非阻止删除。这正是预览失败问题的根源之一。

---

## 9. 问题场景复盘：删除默认 campaign 模板后导入 visual 模板

### 场景步骤

1. 管理员删除了默认的 campaign 模板（id=1, type=campaign, is_default=true）
2. 删除操作：因为 `is_default = true`，SQL 的 `is_default = false` 条件不满足，**删除被阻止**

**但假设通过其他方式（如只剩一个 campaign 模板时）**，或场景变为：

1. 系统有：Default campaign（is_default=true）+ Sample visual + Sample tx
2. 管理员删除 Default campaign → 被阻止（它是默认模板）
3. 管理员先设另一个模板为默认，再删除旧默认 → 但只有 campaign 类型可设默认，如果唯一的 campaign 模板就是默认的那个...

**更现实的场景**：

1. 管理员创建第二个 campaign 模板并设为默认
2. 删除原默认 campaign 模板 → 允许（不再是默认，且模板数 > 1）
3. 此后删除了所有 campaign 类型模板，只留下 campaign_visual 和 tx
4. 此时 `is_default = true` 无任何行满足

### 故障 1：已有 campaign 预览失败

**根因**：[get-campaign-for-preview](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L152-L163) SQL 缺少默认模板回退。

```
campaign.template_id → NULL（模板被删除，FK ON DELETE SET NULL）
    → LEFT JOIN templates 无匹配
    → COALESCE(templates.body, '') = ''
    → template_body = ''
    → camp.CompileTemplate() 失败（空模板体无法包裹内容）
```

而 `get-campaign` 和 `next-campaigns` 都有回退：
```sql
COALESCE(templates.body, (SELECT body FROM templates WHERE is_default = true LIMIT 1), '')
```

### 故障 2：新 campaign 创建找不到默认模板

**根因**：[create-campaign](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L4-L24) SQL 在无 `template_id` 时查找 `is_default = TRUE`。

```
所有 campaign 类型模板被删除
    → 无 is_default = TRUE 的行
    → tpl CTE 返回空
    → 新 campaign 的 template_id = NULL, body = ''
```

**`set-default-template` 限制**：只有 `type='campaign'` 可设为默认，即使有 campaign_visual 模板也无法成为默认。

### 故障 3：tx 模板 subject 被错误地允许为空

**根因**：[update-template](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L14-L21) SQL 中的 subject 回退值错误。

```sql
subject=(CASE WHEN $3 != '' THEN $3 ELSE name END)  -- BUG: ELSE name
```

虽然 Go `validateTemplate` 对 tx 类型检查了 subject 非空，但 SQL 层面的回退逻辑是错误的：

- **对 tx 模板更新**：Go 验证通过后 subject 非空，`$3 != ''` 为 true，暂时不受影响
- **对 campaign/campaign_visual 模板更新**：Handler 强制 `subject = ""`，SQL 回退到 `name`，导致 subject 被设为模板名称
- **绕过 Go 验证的场景**：如果直接调用 SQL 或 API 绕过验证，tx 的空 subject 会被替换为 `name` 而非保持原值，表面上看 subject 不为空但内容已被篡改

**正确的 SQL 应为**：
```sql
subject=(CASE WHEN $3 != '' THEN $3 ELSE subject END)
```

---

## 10. 完整生命周期对比总结

### 必填字段

| 字段 | campaign | campaign_visual | tx |
|------|----------|-----------------|-----|
| name | ✅ 必填 | ✅ 必填 | ✅ 必填 |
| type | ✅ 自动 | ✅ 自动 | ✅ 必填 |
| subject | ❌ 不适用（存空串） | ❌ 不适用（存空串） | ✅ **必须非空** |
| body | ✅ 必填 + 必含 content 占位符 | ✅ 必填（⚠ 无内容验证） | ✅ 必填（⚠ 无内容验证） |
| body_source | ❌ 不使用 | 可选（JSON） | ❌ 不使用 |

### 默认约束

| 约束 | campaign | campaign_visual | tx |
|------|----------|-----------------|-----|
| 可设 `is_default = true` | ✅ | ❌ | ❌ |
| 创建时 subject 处理 | Handler 强制清空为 `""` | Handler 强制清空为 `""` | 保留原值 |
| DB `subject` NOT NULL | 存空串满足 | 存空串满足 | 存实际值满足 |

### 删除约束

| 约束 | 说明 |
|------|------|
| 全局最少 1 个模板 | `COUNT(id) > 1` 才允许删除 |
| 默认模板不可删 | `is_default = false` 才允许删除 |
| 级联重分配 | campaign 的 `template_id` 被重指向默认模板 |
| FK ON DELETE SET NULL | v5.0.0 后 campaign.template_id FK 约束改为 SET NULL |

### 预览路径

| 类型 | 预览路径 | TemplateBody 来源 | Subject 来源 |
|------|----------|-------------------|--------------|
| campaign 模板 | Campaign 渲染管道 | `tpl.Body`（模板外壳） | 虚拟值 |
| campaign_visual 模板 | Campaign 渲染管道 | `tpl.Body`（完整 HTML） | 虚拟值 |
| tx 模板 | TxMessage 渲染管道 | `tpl.Body` | `tpl.Subject` |
| campaign 预览 | Campaign 渲染管道 | `get-campaign-for-preview`（⚠ 无默认回退） | `camp.Subject` |
| visual campaign 预览 | Campaign 渲染管道 | 清空（自包含） | `camp.Subject` |

---

## 11. BUG 汇总与修复建议

### BUG-1：update-template SQL subject 回退值错误

**位置**：[queries/templates.sql#L17](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L17)

**现状**：`subject=(CASE WHEN $3 != '' THEN $3 ELSE name END)`

**应改为**：`subject=(CASE WHEN $3 != '' THEN $3 ELSE subject END)`

### BUG-2：get-campaign-for-preview 缺少默认模板回退

**位置**：[queries/campaigns.sql#L153](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L153)

**现状**：`COALESCE(templates.body, '') AS template_body`

**应改为**：`COALESCE(templates.body, (SELECT body FROM templates WHERE is_default = true LIMIT 1), '') AS template_body`

### BUG-3：set-default-template 无 CTE 匹配时仍清除所有 is_default

**位置**：[queries/templates.sql#L23-L27](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/templates.sql#L23-L27)

**现状**：当目标模板不是 campaign 类型时，CTE `u` 返回空结果，但第二条 UPDATE 仍执行，将所有模板的 `is_default` 设为 false，导致默认模板丢失。

**建议**：第二条 UPDATE 应加条件 `WHERE id != $1 AND (SELECT count(*) FROM u) > 0`。

### 设计缺陷-1：campaign_visual 不能成为默认模板

当所有 campaign 类型模板被删除后，系统无法为新 campaign 提供默认模板。如果允许 campaign_visual 成为默认模板，需同时调整 `set-default-template` 和 `create-campaign` SQL 的逻辑。

### 设计缺陷-2：campaign_visual 和 tx 类型无 body 非空验证

`validateTemplate` 函数只对 campaign 类型验证 body 内容（含 `{{ template "content" . }}`），对 campaign_visual 和 tx 类型无任何 body 验证，允许创建 body 为空的模板。
