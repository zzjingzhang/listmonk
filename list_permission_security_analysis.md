# List级权限跨模块一致性安全分析报告

## 概述

本报告分析了listmonk系统中list级权限控制的跨模块一致性问题。基于以下核心权限函数进行分析：
- `ListRole` - 列表角色模型
- `GetPermittedLists` - 获取用户有权限的列表
- `FilterListsByPerm` - 按权限过滤列表ID
- `GetLists` / `QueryLists` - 查询列表
- `maskRestrictedSubLists` - 掩码受限的订阅者列表
- `hasSubPerm` - 检查订阅者权限
- `checkCampaignPerm` - 检查campaign权限

## 核心权限模型

### 权限检查机制

系统采用三层权限检查机制：
1. **全局权限**：`lists:get_all`, `lists:manage_all`, `campaigns:get_all` 等
2. **角色级权限**：通过 `ListRole` 定义的列表角色权限
3. **细粒度权限**：`HasListPerm`, `hasSubPerm`, `checkCampaignPerm` 等函数

---

## 容易遗漏的权限检查路径

### 路径1：GetRunningCampaignStats - 运行中Campaign统计API

**问题描述**

`GetRunningCampaignStats` API返回所有正在运行的campaign统计信息，但没有根据用户的list级权限进行过滤。角色A（只有list1权限）可以看到list2相关campaign的运行状态。

**源码证据**

[campaigns.go:485-516](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L485-L516)

```go
func (a *App) GetRunningCampaignStats(c echo.Context) error {
    // 直接获取所有运行中的campaign，无权限过滤
    out, err := a.core.GetRunningCampaignStats()
    if err != nil {
        return err
    }
    // ... 返回结果，无任何权限过滤
}
```

[core/campaigns.go:366-382](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go#L366-L382)

```go
func (c *Core) GetRunningCampaignStats() ([]models.CampaignStats, error) {
    out := []models.CampaignStats{}
    // SQL查询没有任何权限过滤条件
    if err := c.q.GetCampaignStatus.Select(&out, models.CampaignStatusRunning); err != nil {
        // ...
    }
    return out, nil
}
```

**风险影响**
- 信息泄露：用户可以看到无权访问的列表对应的campaign运行状态
- 数据暴露：发送速率、已发送数量等敏感信息可能泄露

**修复策略**
1. 在handler层获取用户有权限的list IDs
2. 修改core层函数，添加`getAll`和`permittedIDs`参数
3. SQL查询中添加list权限过滤条件

**测试策略**
```
测试用例1：角色A（只有list1权限）调用API
  - 预期：只返回关联到list1的campaign统计
  - 实际（当前）：返回所有运行中的campaign

测试用例2：超级管理员调用API
  - 预期：返回所有campaign统计
```

---

### 路径2：GetCampaignViewAnalytics - Campaign分析数据API

**问题描述**

`GetCampaignViewAnalytics` API接受campaign ID数组并返回分析数据（views, clicks, bounces），但没有验证用户是否有权限访问这些campaign。

**源码证据**

[campaigns.go:603-642](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go#L603-L642)

```go
func (a *App) GetCampaignViewAnalytics(c echo.Context) error {
    ids, err := parseStringIDs(c.Request().URL.Query()["id"])
    if err != nil {
        return echo.NewHTTPError(http.StatusBadRequest, ...)
    }

    // 仅验证参数格式，不验证campaign访问权限
    var (
        typ  = c.Param("type")
        from = c.QueryParams().Get("from")
        to   = c.QueryParams().Get("to")
    )

    // Campaign link stats
    if typ == "links" {
        // 直接查询，无权限检查
        out, err := a.core.GetCampaignAnalyticsLinks(ids, typ, from, to)
        if err != nil {
            return err
        }
        return c.JSON(http.StatusOK, okResp{out})
    }

    // Get the analytics numbers from the DB for the campaigns.
    out, err := a.core.GetCampaignAnalyticsCounts(ids, typ, from, to)
    // ...
}
```

**风险影响**
- 越权访问：用户可以通过猜测campaign ID获取任何campaign的分析数据
- 数据泄露：view数量、click数量、bounce数量等敏感营销数据泄露

**修复策略**
1. 对每个传入的campaign ID调用`checkCampaignPerm`进行权限验证
2. 或者先过滤出用户有权限访问的campaign ID列表

**测试策略**
```
测试用例1：角色A（只有list1权限）请求list2相关campaign的分析
  - 预期：返回403权限错误或空结果
  - 实际（当前）：返回分析数据

测试用例2：角色A请求list1相关campaign的分析
  - 预期：正常返回数据
```

---

### 路径3：GetSubscriberBounces - 订阅者Bounce记录API

**问题描述**

`GetSubscriberBounces` API返回指定订阅者的bounce记录，但没有验证用户是否有权限访问该订阅者（通过list权限）。

**源码证据**

[bounce.go:58-68](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/bounce.go#L58-L68)

```go
func (a *App) GetSubscriberBounces(c echo.Context) error {
    // 直接获取subscriber ID，无权限检查
    subID := getID(c)
    
    // 直接查询数据库，无权限过滤
    out, _, err := a.core.QueryBounces(0, subID, "", "", "", 0, 1000)
    if err != nil {
        return err
    }

    return c.JSON(http.StatusOK, okResp{out})
}
```

对比已正确实现权限检查的`GetSubscriber`：

[subscribers.go:58-77](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L58-L77)

```go
func (a *App) GetSubscriber(c echo.Context) error {
    user := auth.GetUser(c)

    // ✅ 正确：检查订阅者权限
    id := getID(c)
    if err := a.hasSubPerm(user, []int{id}); err != nil {
        return err
    }
    // ...
}
```

**风险影响**
- 越权访问：用户可以查看任何订阅者的bounce历史
- 隐私泄露：邮箱地址、bounce原因等敏感信息泄露

**修复策略**
1. 添加与`GetSubscriber`相同的`hasSubPerm`权限检查
2. 确保用户至少有权访问该订阅者所属的一个列表

**测试策略**
```
测试用例1：角色A（只有list1权限）访问只属于list2的订阅者的bounce
  - 预期：返回403权限错误
  - 实际（当前）：返回bounce记录

测试用例2：角色A访问属于list1的订阅者的bounce
  - 预期：正常返回数据
```

---

### 路径4：DeleteSubscriberBounces - 删除订阅者Bounce API

**问题描述**

`DeleteSubscriberBounces` API删除指定订阅者的所有bounce记录，但同样缺少list级权限验证。

**源码证据**

[subscribers.go:681-690](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L681-L690)

```go
func (a *App) DeleteSubscriberBounces(c echo.Context) error {
    // 直接获取ID并删除，无任何权限检查
    id := getID(c)
    if err := a.core.DeleteSubscriberBounces(id, ""); err != nil {
        return err
    }

    return c.JSON(http.StatusOK, okResp{true})
}
```

**风险影响**
- 越权操作：用户可以删除任何订阅者的bounce记录
- 数据篡改：恶意用户可以清除bounce历史，影响发送信誉计算

**修复策略**
1. 添加`hasSubPerm`权限检查
2. 确保用户有manage权限才能删除

**测试策略**
```
测试用例1：角色A（只有list1 get权限）尝试删除list2订阅者的bounce
  - 预期：返回403权限错误
  - 实际（当前）：删除成功

测试用例2：角色A尝试删除list1订阅者的bounce
  - 预期：需要manage权限，get权限应该拒绝
```

---

### 路径5：GetSubscriberActivity - 订阅者活动API

**问题描述**

`GetSubscriberActivity` API虽然检查了订阅者访问权限，但返回的活动数据（campaign views和clicks）可能包含用户无权访问的campaign信息，缺少二次过滤。

**源码证据**

[subscribers.go:79-96](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go#L79-L96)

```go
func (a *App) GetSubscriberActivity(c echo.Context) error {
    user := auth.GetUser(c)

    // ✅ 第一步：检查订阅者访问权限（只要有一个列表有权限即可通过）
    id := getID(c)
    if err := a.hasSubPerm(user, []int{id}); err != nil {
        return err
    }

    // ❌ 问题：直接返回所有活动数据，不过滤campaign权限
    out, err := a.core.GetSubscriberActivity(id)
    if err != nil {
        return err
    }

    return c.JSON(http.StatusOK, okResp{out})
}
```

[core/subscribers.go:218-230](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/subscribers.go#L218-L230)

```go
func (c *Core) GetSubscriberActivity(id int) (models.SubscriberActivity, error) {
    out := models.SubscriberActivity{}
    // SQL查询返回该订阅者的所有campaign views和clicks，无权限过滤
    if err := c.q.GetSubscriberActivity.Get(&out, id); err != nil {
        // ...
    }
    return out, nil
}
```

**风险影响**
- 间接信息泄露：通过活动数据可以推断出用户无权访问的campaign的存在和行为
- 数据粒度问题：即使只允许访问list1，也能看到该订阅者在list2 campaign上的活动

**修复策略**
1. 获取用户有权限访问的campaign列表
2. 在core层或handler层过滤掉无权访问的campaign的活动记录
3. 或者修改SQL查询，添加list权限JOIN条件

**测试策略**
```
测试用例1：角色A（只有list1权限）查看同时属于list1和list2的订阅者活动
  - 预期：只返回list1相关campaign的活动记录
  - 实际（当前）：返回所有campaign的活动记录
```

---

## 权限检查一致性对比矩阵

| 功能模块 | 函数/API | 全局权限检查 | List级权限检查 | 数据过滤 |
|---------|---------|-------------|---------------|---------|
| **列表** | GetLists | ✅ | ✅ | ✅ |
| | GetList | ✅ | ✅ | - |
| | DeleteLists | ✅ | ✅ | ✅ |
| **订阅者** | GetSubscriber | ✅ | ✅ | ✅ (maskRestrictedSubLists) |
| | QuerySubscribers | ✅ | ✅ | ✅ |
| | DeleteSubscriber | ✅ | ✅ | - |
| | GetSubscriberActivity | ✅ | ⚠️ (部分) | ❌ |
| | GetSubscriberBounces | ✅ | ❌ | ❌ |
| | DeleteSubscriberBounces | ✅ | ❌ | ❌ |
| **Campaign** | GetCampaigns | ✅ | ✅ | ✅ |
| | GetCampaign | ✅ | ✅ | - |
| | CreateCampaign | ✅ | ✅ | ✅ (FilterListsByPerm) |
| | UpdateCampaign | ✅ | ✅ | ✅ |
| | GetRunningCampaignStats | ✅ | ❌ | ❌ |
| | GetCampaignViewAnalytics | ✅ | ❌ | ❌ |

---

## 根因分析

1. **权限检查分散**：权限检查逻辑分散在各个handler中，没有统一的中间件或切面
2. **新旧代码不一致**：新功能可能遗漏了权限检查模式
3. **多层过滤缺失**：只检查了顶层资源权限，但嵌套数据未过滤
4. **缺少代码规范**：没有明确的权限检查编码规范和Code Review checklist

---

## 修复建议优先级

| 优先级 | 路径 | 风险等级 | 修复难度 |
|-------|------|---------|---------|
| P0 | DeleteSubscriberBounces | 高 | 低 |
| P0 | GetSubscriberBounces | 高 | 低 |
| P1 | GetCampaignViewAnalytics | 中高 | 中 |
| P1 | GetRunningCampaignStats | 中 | 中 |
| P2 | GetSubscriberActivity | 中 | 高 |

---

## 长期防护措施

1. **统一权限中间件**：为所有需要list权限检查的路由创建统一的中间件
2. **代码规范文档**：编写权限检查编码指南
3. **自动化安全测试**：添加权限边界的集成测试
4. **Code Review Checklist**：在PR模板中添加权限检查项

---

## 附录：参考源码文件

| 文件 | 说明 |
|-----|------|
| [internal/auth/models.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/models.go) | 权限模型和核心权限函数 |
| [cmd/lists.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/lists.go) | 列表API处理函数 |
| [cmd/subscribers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/subscribers.go) | 订阅者API处理函数 |
| [cmd/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/campaigns.go) | Campaign API处理函数 |
| [cmd/bounce.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/bounce.go) | Bounce API处理函数 |
| [internal/core/lists.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/lists.go) | 列表核心逻辑 |
| [internal/core/campaigns.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/campaigns.go) | Campaign核心逻辑 |
| [cmd/handlers.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/handlers.go) | 路由和中间件配置 |
