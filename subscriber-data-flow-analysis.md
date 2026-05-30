# Listmonk 订阅者数据处理流程分析

## 概述

本文档分析订阅者从点击campaign退订链接，经过偏好设置、数据导出、数据擦除等操作后，各数据表中数据的保留、置空和删除情况，以及隐私开关对结果的影响。

---

## 1. 核心处理函数分析

### 1.1 SubscriptionPage (订阅页面)

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L197-L248)

**功能**: 渲染订阅管理页面，展示订阅者信息和可管理的邮件列表

**数据操作**:
- **只读操作**: 从数据库读取订阅者信息和订阅列表
- **不修改数据**: 仅用于展示页面

**影响的表**:
| 表名 | 操作 | 说明 |
|------|------|------|
| subscribers | SELECT | 读取订阅者基本信息 |
| subscriber_lists | SELECT | 读取订阅关系 |
| lists | SELECT | 读取列表信息 |

---

### 1.2 SubscriptionPrefs (订阅偏好更新)

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L253-L343)

**功能**: 处理订阅者的偏好设置更新，包括：
- 更新订阅者姓名
- 取消部分列表订阅
- 批量退订或加入黑名单

**核心逻辑**:
```go
// 1. 简单退订（非管理模式）
if !req.Manage || blocklist {
    a.core.UnsubscribeByCampaign(subUUID, campUUID, blocklist)
}

// 2. 管理模式下更新偏好
// - 更新订阅者姓名
// - 对比请求中的列表与数据库中的列表
// - 对未勾选的列表执行退订操作
a.core.UnsubscribeLists([]int{sub.ID}, nil, unsubUUIDs)
```

**影响的表**:
| 表名 | 操作 | 字段变化 |
|------|------|----------|
| subscribers | UPDATE | name, updated_at (如果更新姓名) |
| subscriber_lists | UPDATE | status = 'unsubscribed', updated_at |

---

### 1.3 UnsubscribeByCampaign (按Campaign退订)

**文件位置**:
- Go代码: [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L493-L502)
- SQL: [subscribers.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L251-L267)

**核心SQL逻辑**:
```sql
WITH lists AS (
    SELECT list_id FROM campaign_lists
    LEFT JOIN campaigns ON (campaign_lists.campaign_id = campaigns.id)
    WHERE campaigns.uuid = $1
),
sub AS (
    UPDATE subscribers SET status = (CASE WHEN $3 IS TRUE THEN 'blocklisted' ELSE status END)
    WHERE uuid = $2 RETURNING id
)
UPDATE subscriber_lists SET status = 'unsubscribed', updated_at=NOW() WHERE
    subscriber_id = (SELECT id FROM sub) AND status != 'unsubscribed' AND
    CASE WHEN $3 IS FALSE THEN list_id = ANY(SELECT list_id FROM lists) ELSE list_id != 0 END;
```

**行为分析**:
- **blocklist = false**: 仅从campaign关联的列表中退订
- **blocklist = true**: 从所有列表退订，并将订阅者状态设为`blocklisted`

**影响的表**:
| 表名 | 操作 | 条件 | 字段变化 |
|------|------|------|----------|
| subscribers | UPDATE | blocklist=true | status = 'blocklisted' |
| subscriber_lists | UPDATE | 总是 | status = 'unsubscribed', updated_at |

---

### 1.4 SelfExportSubscriberData (导出个人数据)

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L598-L649)

**功能**: 导出订阅者的个人数据（Profile、订阅列表、浏览记录、点击记录）并通过邮件发送

**核心逻辑**:
```go
data, b, err := a.exportSubscriberData(0, subUUID, a.cfg.Privacy.Exportable)
// 导出内容受 privacy.exportable 配置控制
// 默认导出: ["profile", "subscriptions", "campaign_views", "link_clicks"]
```

**SQL查询**: [subscribers.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L404-L433)

**数据操作**:
- **只读操作**: 不修改任何数据
- **导出内容**: 取决于 `privacy.exportable` 配置

**可导出的数据项**:
| 数据项 | 说明 |
|--------|------|
| profile | 订阅者基本信息（id, uuid, email, name, attribs, status, created_at, updated_at） |
| subscriptions | 订阅列表（列表名、订阅状态、创建时间） |
| campaign_views | campaign浏览记录（campaign名称、浏览次数） |
| link_clicks | 链接点击记录（URL、点击次数） |

---

### 1.5 WipeSubscriberData (擦除订阅者数据)

**文件位置**: [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L654-L670)

**功能**: 允许订阅者删除自己的数据

**核心调用**:
```go
a.core.DeleteSubscribers(nil, []string{subUUID})
```

**实际调用**: DeleteSubscribers 函数

---

### 1.6 DeleteSubscribers (删除订阅者)

**文件位置**:
- Go代码: [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L463-L479)
- SQL: [subscribers.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L203-L205)

**核心SQL**:
```sql
DELETE FROM subscribers WHERE CASE WHEN ARRAY_LENGTH($1::INT[], 1) > 0 THEN id = ANY($1) ELSE uuid = ANY($2::UUID[]) END;
```

---

## 2. SQL外键策略分析

**文件位置**: [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql)

### 2.1 外键约束汇总

| 表名 | 字段 | 引用表 | ON DELETE行为 | 说明 |
|------|------|--------|---------------|------|
| **subscriber_lists** | subscriber_id | subscribers | **CASCADE** | 删除订阅者时，同步删除订阅关系 |
| subscriber_lists | list_id | lists | **CASCADE** | 删除列表时，同步删除订阅关系 |
| **campaign_views** | subscriber_id | subscribers | **SET NULL** | 删除订阅者时，subscriber_id置空 |
| campaign_views | campaign_id | campaigns | **CASCADE** | 删除campaign时，同步删除浏览记录 |
| **link_clicks** | subscriber_id | subscribers | **SET NULL** | 删除订阅者时，subscriber_id置空 |
| link_clicks | campaign_id | campaigns | **CASCADE** | 删除campaign时，同步删除点击记录 |
| link_clicks | link_id | links | **CASCADE** | 删除链接时，同步删除点击记录 |
| **bounces** | subscriber_id | subscribers | **CASCADE** | 删除订阅者时，同步删除退信记录 |
| bounces | campaign_id | campaigns | **SET NULL** | 删除campaign时，campaign_id置空 |

### 2.2 关键外键行为说明

#### subscriber_lists - ON DELETE CASCADE
```sql
subscriber_id INTEGER REFERENCES subscribers(id) ON DELETE CASCADE ON UPDATE CASCADE
```
- 删除订阅者时，**所有相关的订阅关系记录被删除**
- 这意味着擦除数据后，subscriber_lists中不会有残留记录

#### campaign_views - ON DELETE SET NULL
```sql
subscriber_id INTEGER NULL REFERENCES subscribers(id) ON DELETE SET NULL ON UPDATE CASCADE
```
- 删除订阅者时，**subscriber_id字段被置为NULL**
- **浏览记录本身保留**，但与订阅者的关联断开
- 这就是为什么"campaign analytics里仍有匿名点击/浏览记录"

#### link_clicks - ON DELETE SET NULL
```sql
subscriber_id INTEGER NULL REFERENCES subscribers(id) ON DELETE SET NULL ON UPDATE CASCADE
```
- 删除订阅者时，**subscriber_id字段被置为NULL**
- **点击记录本身保留**，但与订阅者的关联断开

#### bounces - ON DELETE CASCADE
```sql
subscriber_id INTEGER NOT NULL REFERENCES subscribers(id) ON DELETE CASCADE ON UPDATE CASCADE
```
- 删除订阅者时，**所有相关的退信记录被删除**
- 注意：subscriber_id是NOT NULL的，所以必须CASCADE

---

## 3. 完整操作流程数据变化分析

### 场景描述
订阅者点击campaign中的退订链接进入偏好页 → 取消部分列表 → 请求导出个人数据 → 要求擦除

### 3.1 阶段1：取消部分列表 (SubscriptionPrefs)

**执行操作**: `UnsubscribeLists`

**数据变化**:
| 表 | 操作 | 具体变化 |
|----|------|----------|
| subscribers | 无变化 | （除非更新姓名） |
| subscriber_lists | UPDATE | 被取消的列表：status = 'unsubscribed'，updated_at更新 |
| campaign_views | 无变化 | |
| link_clicks | 无变化 | |
| bounces | 无变化 | |

### 3.2 阶段2：导出个人数据 (SelfExportSubscriberData)

**数据变化**:
| 表 | 操作 | 具体变化 |
|----|------|----------|
| 所有表 | 只读 | 不修改任何数据，仅读取并导出 |

### 3.3 阶段3：数据擦除 (WipeSubscriberData → DeleteSubscribers)

**执行操作**: `DELETE FROM subscribers WHERE uuid = ?`

**触发的外键连锁反应**:

| 表 | 外键行为 | 最终数据状态 |
|----|----------|--------------|
| **subscribers** | 主表删除 | ✅ **记录被完全删除** |
| **subscriber_lists** | CASCADE | ✅ **所有订阅关系记录被删除** |
| **campaign_views** | SET NULL | ⚠️ **记录保留，但subscriber_id = NULL**（匿名化） |
| **link_clicks** | SET NULL | ⚠️ **记录保留，但subscriber_id = NULL**（匿名化） |
| **bounces** | CASCADE | ✅ **所有退信记录被删除** |

---

## 4. 隐私开关配置影响

**配置文件位置**: [settings.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/settings.go#L31-L41)

### 4.1 privacy.individual_tracking (个人追踪开关)

**默认值**: `false`

**影响位置**:
- [public.go LinkRedirect](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L551-L554)
- [public.go RegisterCampaignView](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go#L577-L580)

**代码逻辑**:
```go
// 如果individual_tracking禁用，不记录subscriber_id
subUUID := c.Param("subUUID")
if !a.cfg.Privacy.IndividualTracking {
    subUUID = ""  // 置空，不关联到具体订阅者
}
```

**对数据的影响**:

| individual_tracking | campaign_views.subscriber_id | link_clicks.subscriber_id |
|---------------------|-------------------------------|---------------------------|
| **true** (启用) | 记录真实的subscriber_id | 记录真实的subscriber_id |
| **false** (禁用，默认) | **NULL**（匿名记录） | **NULL**（匿名记录） |

**关键结论**:
- 当 `individual_tracking = false` 时，**浏览和点击记录从一开始就是匿名的**
- 即使不执行数据擦除，这些记录也无法关联到具体订阅者
- 这解释了为什么"campaign analytics里仍有匿名点击/浏览记录"——它们本来就是匿名的

### 4.2 privacy.disable_tracking (全局追踪开关)

**默认值**: `false`

**影响**: 完全禁用追踪，不记录任何浏览或点击

```go
if a.cfg.Privacy.DisableTracking {
    // 直接重定向，不记录点击
    return c.Redirect(http.StatusTemporaryRedirect, url)
}
```

### 4.3 privacy.allow_wipe (允许数据擦除)

**默认值**: `true`

**影响**: 控制是否显示和允许数据擦除功能

### 4.4 privacy.allow_export (允许数据导出)

**默认值**: `true`

**影响**: 控制是否显示和允许数据导出功能

### 4.5 privacy.exportable (可导出数据项)

**默认值**: `["profile", "subscriptions", "campaign_views", "link_clicks"]`

**影响**: 控制导出数据时包含哪些内容

---

## 5. 关键问题解答

### Q1: 为什么擦除后campaign analytics仍有浏览/点击记录？

**原因**:
1. `campaign_views` 和 `link_clicks` 表的外键策略是 `ON DELETE SET NULL`
2. 删除订阅者时，记录保留但 `subscriber_id` 被置为 `NULL`
3. 如果 `individual_tracking = false`（默认），这些记录**从一开始就是匿名的**

**设计意图**:
- 保留统计数据的完整性（总浏览量、总点击量）
- 保护隐私（断开与个人的关联）

### Q2: subscriber_lists状态为什么不符合预期？

**原因分析**:
- **取消部分列表**：使用 `UnsubscribeLists` → status = 'unsubscribed'（记录保留）
- **数据擦除**：使用 `DeleteSubscribers` → 触发CASCADE → **记录完全删除**

**两种操作的区别**:
| 操作 | subscriber_lists结果 |
|------|---------------------|
| 取消订阅 | 记录保留，status = 'unsubscribed' |
| 数据擦除 | 记录被完全删除（CASCADE） |

### Q3: bounces记录为什么不完全符合预期？

**原因**:
- bounces表的外键策略是 `ON DELETE CASCADE`
- 擦除订阅者时，**所有bounces记录被完全删除**
- 如果在擦除前有退信，擦除后这些记录会消失

---

## 6. 数据状态汇总表

### 6.1 取消部分列表后（未擦除）

| 表 | 数据状态 | subscriber_id | 其他字段 |
|----|----------|---------------|----------|
| subscribers | ✅ 保留 | 正常 | name可能更新 |
| subscriber_lists | ✅ 部分保留 | 正常 | 取消的列表：status = 'unsubscribed' |
| campaign_views | ✅ 全部保留 | 正常/NULL* | 不变 |
| link_clicks | ✅ 全部保留 | 正常/NULL* | 不变 |
| bounces | ✅ 全部保留 | 正常 | 不变 |

*取决于individual_tracking设置

### 6.2 数据擦除后

| 表 | 数据状态 | subscriber_id | 其他字段 |
|----|----------|---------------|----------|
| subscribers | ❌ 已删除 | - | - |
| subscriber_lists | ❌ 已删除（CASCADE） | - | - |
| campaign_views | ⚠️ 匿名保留 | NULL | 其他字段保留 |
| link_clicks | ⚠️ 匿名保留 | NULL | 其他字段保留 |
| bounces | ❌ 已删除（CASCADE） | - | - |

---

## 7. 隐私设置对最终结果的影响矩阵

| 操作 | individual_tracking=true | individual_tracking=false（默认） |
|------|--------------------------|----------------------------------|
| **浏览/点击记录** | 关联到具体订阅者 | 匿名记录（subscriber_id=NULL） |
| **擦除前可追溯** | 是 | 否 |
| **擦除后状态** | 变为匿名 | 保持匿名（无变化） |
| **统计数据完整性** | 完整 | 完整 |

| 操作 | disable_tracking=true | disable_tracking=false（默认） |
|------|----------------------|--------------------------------|
| **浏览记录** | 不记录 | 记录 |
| **点击记录** | 不记录 | 记录 |
| **Analytics统计** | 为空 | 有数据 |

---

## 8. 核心代码参考位置

| 功能 | 文件 | 行号 |
|------|------|------|
| SubscriptionPage | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L197-L248 |
| SubscriptionPrefs | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L253-L343 |
| UnsubscribeByCampaign | [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go) | L493-L502 |
| SelfExportSubscriberData | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L598-L649 |
| WipeSubscriberData | [public.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/public.go) | L654-L670 |
| DeleteSubscribers | [subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go) | L463-L479 |
| 外键定义 | [schema.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql) | L60-L71, L155-L166, L206-L219, L302-L315 |
| 隐私设置 | [settings.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/settings.go) | L31-L41 |
