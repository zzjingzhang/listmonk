# Campaign 状态机分析

## 1. 状态定义

Campaign 共有 6 种状态，定义在 [models/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L18-L24)：

| 状态 | 说明 |
|------|------|
| `draft` | 草稿 |
| `scheduled` | 已定时/排期 |
| `running` | 运行中 |
| `paused` | 已暂停 |
| `finished` | 已完成 |
| `cancelled` | 已取消 |

---

## 2. 状态转换规则

状态转换逻辑在 [internal/core/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L250-L303) 的 `UpdateCampaignStatus` 函数中定义。

### 状态转换矩阵

| 目标状态 \ 当前状态 | draft | scheduled | running | paused | finished | cancelled |
|-------------------|:-----:|:---------:|:-------:|:------:|:--------:|:---------:|
| **→ draft** | - | ✅ | ❌ | ❌ | ❌ | ❌ |
| **→ scheduled** | ✅ | - | ❌ | ✅ | ❌ | ❌ |
| **→ running** | ✅ | ❌¹ | - | ✅ | ❌ | ❌ |
| **→ paused** | ❌ | ❌ | ✅ | - | ❌ | ❌ |
| **→ cancelled** | ❌ | ❌ | ✅ | ✅ | ❌ | - |
| **→ finished** | ❌ | ❌ | ✅² | ❌ | - | ❌ |

> 注：
> ¹ 当 campaign 设置了 `send_at` 时间时，直接设置 `running` 会自动变为 `scheduled`
> ² `finished` 状态由系统自动设置（发送完成），不能通过 API 手动设置

### 转换规则详解

#### draft → scheduled
- **条件**：必须设置 `send_at`（发送时间）且时间在未来
- **错误信息**：`campaigns.needsSendAt`

#### draft → running
- **条件**：无特殊条件（立即发送）
- **注意**：如果设置了 `send_at`，SQL 层会自动将状态改为 `scheduled`

#### scheduled → draft
- **条件**：只能从 `scheduled` 撤销回 `draft`
- **错误信息**：`campaigns.onlyScheduledAsDraft`

#### scheduled → running
- **条件**：不允许直接转换，需要先改为 `draft` 或 `paused`
- **错误信息**：`campaigns.onlyDraftAsScheduled`

#### paused → scheduled
- **条件**：必须设置 `send_at`

#### paused → running
- **条件**：无特殊条件

#### running → paused
- **条件**：只能从 `running` 暂停
- **错误信息**：`campaigns.onlyActivePause`
- **副作用**：manager 会停止正在进行的发送

#### running/paused → cancelled
- **条件**：只能从 `running` 或 `paused` 取消
- **错误信息**：`campaigns.onlyActiveCancel`
- **副作用**：manager 会停止正在进行的发送

---

## 3. 各状态下允许的字段变更

可编辑状态判断在 [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L830-L834) 的 `canEditCampaign` 函数中定义。

### 可编辑状态（允许修改所有字段）

| 状态 | 是否可编辑 | 可修改字段 |
|------|:----------:|-----------|
| **draft** | ✅ | 所有字段 |
| **scheduled** | ✅ | 所有字段 |
| **paused** | ✅ | 所有字段 |
| **running** | ❌ | 禁止修改 |
| **finished** | ❌ | 禁止修改 |
| **cancelled** | ❌ | 禁止修改 |

### 可修改字段列表

在可编辑状态下，通过 `UpdateCampaign` 可以修改以下字段（定义在 [internal/core/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L214-L247) 和 [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L388-L433)）：

| 字段 | 说明 | 特殊行为 |
|------|------|----------|
| `name` | 活动名称 | - |
| `subject` | 邮件主题 | - |
| `from_email` | 发件人 | - |
| `body` | 邮件正文 | - |
| `altbody` | 纯文本备选 | - |
| `content_type` | 内容类型 | visual 类型不保存 template_id |
| `send_at` | 发送时间 | **重要**：如果 scheduled 状态下清空此字段，状态自动变回 draft |
| `headers` | 邮件头 | - |
| `attribs` | 属性 | - |
| `tags` | 标签 | - |
| `messenger` | 发送通道 | - |
| `template_id` | 模板ID | visual 内容类型设为 NULL |
| `lists` | 关联列表 | **更新 campaign_lists 历史记录** |
| `archive` | 是否归档 | - |
| `archive_slug` | 归档 Slug | - |
| `archive_template_id` | 归档模板ID | visual 内容类型设为 NULL |
| `archive_meta` | 归档元数据 | - |
| `media` | 附件 | - |
| `body_source` | 可视化编辑器源码 | - |

### campaign_lists 历史记录机制

在 [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L431-L433) 中：

```sql
INSERT INTO campaign_lists (campaign_id, list_id, list_name)
    (SELECT $1 as campaign_id, id, name FROM lists WHERE id=ANY($14::INT[]))
    ON CONFLICT (campaign_id, list_id) DO UPDATE SET list_name = EXCLUDED.list_name;
```

**行为说明**：
- 更新 campaign 关联的列表时，`campaign_lists` 表会保存列表的**历史名称**
- 使用 `ON CONFLICT` 子句，当列表 ID 已存在时更新列表名称
- 即使列表被删除，campaign 仍保留历史名称记录
- 设计目的：保证已发送/归档的 campaign 能显示当时的列表名称

---

## 4. 各状态下允许的行为

### 行为矩阵

| 行为 \ 状态 | draft | scheduled | running | paused | finished | cancelled |
|------------|:-----:|:---------:|:-------:|:------:|:--------:|:---------:|
| **编辑字段** | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ |
| **设置为 draft** | - | ✅ | ❌ | ❌ | ❌ | ❌ |
| **设置为 scheduled** | ✅* | - | ❌ | ✅* | ❌ | ❌ |
| **设置为 running** | ✅ | ❌ | - | ✅ | ❌ | ❌ |
| **设置为 paused** | ❌ | ❌ | ✅ | - | ❌ | ❌ |
| **设置为 cancelled** | ❌ | ❌ | ✅ | ✅ | ❌ | - |
| **删除** | ✅ | ✅** | ❌ | ❌ | ✅ | ✅ |
| **预览** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| **测试发送** | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

> 注：
> * 需要设置有效的 `send_at` 时间
> ** 注释说明是 "Only scheduled campaigns that have not started yet can be deleted"，但代码实际未做严格校验

---

## 5. Regular 与 Optin Campaign 差异

定义在 [models/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go#L25-L26)：
- `CampaignTypeRegular = "regular"`
- `CampaignTypeOptin = "optin"`

### 核心差异

| 维度 | Regular Campaign | Optin Campaign |
|------|------------------|----------------|
| **用途** | 发送营销/通讯邮件给已订阅用户 | 发送确认邮件给未确认订阅者（双重确认） |
| **订阅者筛选** | 已确认 (confirmed) 或 非取消订阅 | 未确认 (unconfirmed) |
| **列表要求** | 任意列表 | 必须包含双重确认 (double opt-in) 列表 |
| **默认内容** | 无 | 自动生成包含确认链接的邮件模板 |

### 订阅者筛选逻辑

筛选逻辑在 [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L206-L212)（next-campaigns）和 [L351-L362](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L351-L362)（next-campaign-subscribers）中定义：

```sql
CASE
    -- optin campaign: 只选择未确认的订阅者（双重确认列表）
    WHEN camps.type = 'optin' THEN sl.status = 'unconfirmed' AND cl.optin = 'double'
    
    -- regular campaign:
    WHEN cl.optin = 'double' THEN sl.status = 'confirmed'  -- 双重确认列表：只选已确认的
    ELSE sl.status != 'unsubscribed'                       -- 单确认列表：选所有非取消订阅的
END
```

### Optin Campaign 创建逻辑

在 [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L265-L288) 的 `CreateCampaign` 中：

1. **列表校验**：必须包含至少一个双重确认 (double opt-in) 列表
   - 校验失败错误：`campaigns.noOptinLists`

2. **自动生成邮件内容**：调用 `makeOptinCampaignMessage` 生成包含确认链接的邮件模板
   - 链接格式：`{{ OptinURL }}?l=<list-uuid>`
   - 使用 `notifs.Tpls` 的 `optin-campaign` 模板

3. **使用场景**：用于订阅者完成表单提交后，发送确认邮件完成双重确认流程

### Regular Campaign 行为

1. **默认类型**：未指定类型时默认为 `regular` ([cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L272-L274))

2. **订阅者范围**：
   - 双重确认列表：只发送给 `confirmed` 状态的订阅者
   - 单确认列表：发送给所有非 `unsubscribed` 状态的订阅者
   - 排除：已拉黑 (blocklisted) 的订阅者

---

## 6. 运营场景分析（用户场景推演）

**场景**：运营人员将 draft campaign 设置为 scheduled，又马上修改列表、模板和内容，随后暂停、恢复、取消并尝试删除。

### 操作流程推演

| 步骤 | 操作 | 当前状态 | 结果 | 说明 |
|------|------|----------|------|------|
| 1 | 初始状态 | `draft` | - | 可编辑所有字段 |
| 2 | 设置为 scheduled | `draft` → `scheduled` | ✅ 成功 | 需已设置 send_at 且在未来 |
| 3 | 修改列表、模板、内容 | `scheduled` | ✅ 成功 | scheduled 状态可编辑<br>**campaign_lists 更新历史名称** |
| 4 | 尝试暂停 | `scheduled` → `paused` | ❌ 失败 | 错误：`onlyActivePause`<br>只能从 running 暂停 |
| 5 | 改为 draft 再运行 | `scheduled` → `draft` → `running` | ✅ 成功 | 先撤销定时，再立即发送 |
| 6 | 暂停 | `running` → `paused` | ✅ 成功 | 停止发送进程 |
| 7 | 恢复（改为 running） | `paused` → `running` | ✅ 成功 | 继续发送 |
| 8 | 取消 | `running` → `cancelled` | ✅ 成功 | 停止发送，标记为取消 |
| 9 | 尝试删除 | `cancelled` | ✅ 成功 | 代码允许删除所有非运行状态 |

### 关键要点

1. **scheduled 状态可以修改内容**：这意味着定时发送前可以随时调整
2. **列表修改会更新历史名称**：`campaign_lists` 表的 `list_name` 会被更新为当前名称
3. **scheduled 不能直接暂停**：必须先改为 draft 或直接 running
4. **取消后可以删除**：代码层面允许删除 cancelled 状态的 campaign

---

## 7. 字段验证规则 (validateCampaignFields)

验证逻辑在 [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L663-L747) 中定义：

| 字段 | 规则 |
|------|------|
| `from_email` | 可选，默认使用全局配置；格式校验 |
| `name` | 必填，1 ~ stdInputMaxLen 字符 |
| `subject` | 必填，1 ~ 5000 字符（支持 Go 模板） |
| `content_type` | richtext/html/plain/visual/markdown，默认 richtext |
| `send_at` | 可选，若设置必须是未来时间 |
| `lists` | 必填，至少选择一个列表 |
| `messenger` | 必填，必须是已注册的 messenger |
| `body` | 必填，模板语法校验 |
| `headers` | 可选，默认空数组 |
| `attribs` | 可选，JSON 格式校验 |
| `archive_slug` | 可选，自动格式化为字母数字连字符 |

---

## 8. 关键代码位置汇总

| 功能 | 文件 | 行号 |
|------|------|------|
| 状态常量定义 | [models/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/campaigns.go) | L18-L32 |
| 可编辑状态判断 | [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go) | L830-L834 |
| 状态转换校验 | [internal/core/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go) | L257-L286 |
| UpdateCampaign 核心 | [internal/core/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go) | L214-L247 |
| UpdateCampaign SQL | [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L388-L433 |
| UpdateCampaignStatus SQL | [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L443-L452 |
| 字段验证 | [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go) | L663-L747 |
| Regular/Optin 筛选逻辑 | [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L206-L212, L351-L362 |
| campaign_lists 历史归档 | [queries/campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L431-L433 |
