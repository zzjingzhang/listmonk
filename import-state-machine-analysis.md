# 订阅者导入完整状态机推演

> 场景：一个非 superadmin 用户上传包含 10 万行订阅者的 CSV 压缩包，选择 overwrite 模式、指定多个列表（其中部分列表没有 manage 权限），导入到一半时用户点击停止，PostCB 仍触发了物化视图刷新和通知。

---

## 1. 端到端调用链总览

```
HTTP POST /api/import/subscribers
  └─ ImportSubscribers()                    — cmd/import.go
       ├─ 权限过滤: FilterListsByPerm()
       ├─ 参数校验: mode / sub_status / delim
       ├─ 文件落盘: io.Copy → /tmp/listmonk*
       ├─ NewSession(opt)                    — subimporter/importer.go
       │    └─ status = "importing"
       ├─ go sess.Start()                    — 消费者 goroutine
       ├─ ExtractZIP(path, 1)                — 同步解压
       └─ go sess.LoadCSV(path, delim)       — 生产者 goroutine
            ├─ countLines → status.Total
            ├─ mapCSVHeaders
            ├─ for each row:
            │    ├─ stop signal check
            │    ├─ ValidateFields()
            │    └─ subQueue <- sub
            └─ close(subQueue)

sess.Start() (消费者):
  for sub := range subQueue:
    ├─ tx.Begin (每批首次)
    ├─ stmt.Exec (UpsertSubscriber / UpsertBlocklistSubscriber)
    ├─ cur++ / total++
    └─ if cur % 10000 == 0 → tx.Commit → incrementImportCount
  queue closed:
    ├─ tx.Commit (残余批次)
    ├─ setStatus("finished")
    ├─ UpdateListDateStmt.Exec(listIDs)
    └─ sendNotif("finished") → PostCB()
         ├─ core.RefreshMatViews(true)
         └─ notifs.NotifySystem(...)
```

---

## 2. 状态机定义

### 2.1 Importer 全局状态

| 状态 | 常量 | 含义 | 可转移至 |
|------|------|------|----------|
| `none` | `StatusNone` | 空闲，无导入任务 | `importing` |
| `importing` | `StatusImporting` | 正在导入 | `stopping`, `failed` |
| `stopping` | `StatusStopping` | 收到停止信号，正在收尾 | `finished`, `failed` |
| `finished` | `StatusFinished` | 导入完成（正常或被停止后） | `none` |
| `failed` | `StatusFailed` | 导入失败 | `none` |

### 2.2 状态转移图

```
                 NewSession()
  none ──────────────────────────► importing
   ▲                                 │    │
   │                         Stop()  │    │ ExtractZIP/LoadCSV 错误
   │ Stop()                         │    │ (defer: failed=true)
   │ (非 importing 时)              ▼    ▼
   │                            stopping  failed
   │                               │
   │              Start() 消费完队列  │
   │                               ▼
   └─────────────────────────── finished
```

---

## 3. 阶段逐层推演

### 3.1 阶段一：ImportSubscribers Handler

**源码位置**：[import.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/import.go#L18-L119)

#### 3.1.1 并发守卫

```go
if a.importer.GetStats().Status == subimporter.StatusImporting {
    return echo.NewHTTPError(http.StatusBadRequest, ...)
}
```

Importer 是单例，同一时刻只允许一个导入会话。若已有导入运行中，直接拒绝。

#### 3.1.2 列表权限过滤（关键安全控制点）

```go
user := auth.GetUser(c)
opt.ListIDs = user.FilterListsByPerm(auth.PermTypeManage, opt.ListIDs)
```

**源码位置**：[models.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/models.go#L271-L312)

`FilterListsByPerm` 的过滤逻辑：

1. 若用户拥有 `lists:manage_all` 权限 → 直接返回原始 listIDs（不过滤）
2. 否则，遍历每个 listID，检查 `ListPermissionsMap[id]` 中是否存在 `list:manage` 权限
3. 仅保留有 manage 权限的列表 ID

**本场景行为**：用户请求 `listIDs = [1, 2, 3, 4, 5]`，其中列表 4、5 无 manage 权限 → 过滤后 `opt.ListIDs = [1, 2, 3]`。

> ⚠️ **问题 1：静默丢弃**。被过滤的列表 ID 不会以任何形式反馈给用户。前端无法得知部分列表因权限不足而被排除。

```go
if len(opt.ListIDs) == 0 && opt.Mode != subimporter.ModeBlocklist {
    return echo.NewHTTPError(http.StatusForbidden, ...)
}
```

若过滤后列表为空且模式为 subscribe → 403。但若过滤后仍有至少一个列表（如本场景的 [1,2,3]），则继续执行。

> ⚠️ **问题 2：Blocklist 模式无需列表**。当 `mode=blocklist` 时，`opt.ListIDs` 可以为空，完全绕过列表权限检查。这意味着任何有 `subscribers:import` 权限的用户都可以将订阅者加入黑名单，无需拥有任何列表的管理权限。

#### 3.1.3 模式与状态校验

```go
// Overwrite 兼容处理：旧字段 overwrite=true 自动开启两个细粒度字段
if opt.Overwrite {
    opt.OverwriteUserInfo = true
    opt.OverwriteSubStatus = true
}
```

#### 3.1.4 文件处理

上传文件先落盘至临时文件，再根据扩展名分支：

- `.csv` → 直接 `go sess.LoadCSV(path, delim)`
- `.zip` → **同步**调用 `sess.ExtractZIP(path, 1)`，再 `go sess.LoadCSV(csvPath, delim)`

> ⚠️ **问题 3：ExtractZIP 是同步阻塞的**。如果 ZIP 文件很大，HTTP 请求会在解压完成后才返回响应。此时 `go sess.Start()` 已经在运行并等待 subQueue，但由于 CSV 尚未加载，subQueue 为空，Start() 只是阻塞在 `range subQueue` 上。

---

### 3.2 阶段二：ExtractZIP

**源码位置**：[importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L373-L449)

```
状态: importing → (成功) importing / (失败) failed
```

| 步骤 | 操作 | 失败时状态 |
|------|------|-----------|
| 1 | `zip.OpenReader(srcPath)` | `failed` (defer) |
| 2 | `os.MkdirTemp` | `failed` (defer) |
| 3 | 遍历 ZIP 内条目，跳过非 `.csv` 文件 | — |
| 4 | 逐个提取 CSV 到临时目录 | `failed` (defer) |
| 5 | 最多提取 `maxCSVs` 个（当前为 1） | — |
| 6 | `failed = false`（defer 不触发） | — |

**本场景**：ZIP 内含一个 CSV 文件，成功提取后返回临时目录路径和文件名列表。

---

### 3.3 阶段三：LoadCSV（生产者）

**源码位置**：[importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L452-L585)

```
状态: importing → (正常完成) importing → (停止) importing → (错误) failed
```

#### 3.3.1 行数统计

```go
numLines, err := countLines(f)
s.im.status.Total = numLines - 1  // 排除 header
```

**本场景**：`Total = 100000`。

#### 3.3.2 CSV Header 映射

`mapCSVHeaders` 将 CSV 列头映射到已知字段（`email`, `name`, `attributes`），返回 `{字段名: 列索引}` 映射。若 `email` 列不存在，直接报错返回。

#### 3.3.3 逐行读取循环

```go
for {
    // ★ 停止信号检查点
    select {
    case <-s.im.stop:
        failed = false        // 不触发 defer 的 failed 设置
        close(s.subQueue)     // 关闭队列，通知消费者结束
        s.log.Println("stop request received")
        return nil
    default:
    }

    cols, err := rd.Read()
    // ... 错误处理 ...

    // 构造 SubReq
    sub.Email = row["email"]
    sub.Name = row["name"]

    // ★ ValidateFields
    sub, err = s.im.ValidateFields(sub)
    if err != nil {
        s.log.Printf("skipping line %d: %v", i, err, cols)
        continue   // 跳过无效行，不中断导入
    }

    // 解析 JSON attributes
    // ...

    // ★ 推入队列
    s.subQueue <- sub
}
```

**停止信号检查**只在每行读取的间隙执行。若 `subQueue` 已满（缓冲区 = `commitBatchSize = 10000`），`s.subQueue <- sub` 会阻塞，此时即使收到 stop 信号也无法响应，直到消费者腾出空间。

#### 3.3.4 正常完成

```go
close(s.subQueue)   // 通知消费者：生产结束
failed = false
```

---

### 3.4 阶段四：ValidateFields

**源码位置**：[importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L639-L664)

```
输入: SubReq{Email, Name}
  │
  ├─ Email 长度 > 1000 → 错误
  ├─ SanitizeEmail()
  │    ├─ 基础格式校验 (utils.SanitizeEmail)
  │    ├─ Domain Allowlist 检查（若配置）
  │    └─ Domain Blocklist 检查（若配置）
  ├─ Email 转小写
  ├─ Name 为空时从 Email @ 前缀推导
  └─ 返回清洗后的 SubReq
```

无效行仅 skip（日志记录 + continue），不会中断整个导入。

---

### 3.5 阶段五：Start（消费者）— 批量提交与模式处理

**源码位置**：[importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L273-L363)

#### 3.5.1 批量提交机制

```
每 10000 行 = 一个数据库事务
┌─────────────────────────────────────────────────┐
│ tx.Begin()                                       │
│ for i := 0; i < 10000; i++:                     │
│     stmt.Exec(uuid, email, name, attribs, ...)   │
│ tx.Commit() → incrementImportCount(10000)        │
└─────────────────────────────────────────────────┘
```

**关键特性**：
- 事务粒度为 `commitBatchSize = 10000`
- 只有 `tx.Commit()` 成功后，数据才真正写入数据库
- `tx.Rollback()` 会丢弃当前批次的所有行
- `incrementImportCount` 仅在 Commit 成功后调用

#### 3.5.2 Mode 处理：Subscribe vs Blocklist

**Subscribe 模式**（[upsert-subscriber SQL](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L114-L135)）：

```sql
WITH sub AS (
    INSERT INTO subscribers (uuid, email, name, attribs, status)
    VALUES($1, $2, $3, $4, 'enabled')
    ON CONFLICT (email)
    DO UPDATE SET
        name    = CASE WHEN $7 THEN $3 ELSE s.name END,       -- $7=OverwriteUserInfo
        attribs = CASE WHEN $7 THEN $4 ELSE s.attribs END,
        updated_at = NOW()
    RETURNING uuid, id, status
),
subs AS (
    INSERT INTO subscriber_lists (subscriber_id, list_id, status)
    SELECT sub.id, listID,
           CASE WHEN sub.status = 'blocklisted'
                THEN 'unsubscribed'
                ELSE $6::subscription_status END              -- $6=SubStatus
    FROM sub, UNNEST($5::INT[]) AS listID                    -- $5=ListIDs
    ON CONFLICT (subscriber_id, list_id) DO UPDATE
    SET updated_at = NOW(),
        status = CASE WHEN $8 THEN EXCLUDED.status            -- $8=OverwriteSubStatus
                      ELSE subscriber_lists.status END
)
SELECT uuid, id from sub;
```

**本场景参数映射**：

| 参数 | 值 | 含义 |
|------|----|------|
| $1 | UUID | 新订阅者 UUID |
| $2 | email | 邮箱 |
| $3 | name | 姓名 |
| $4 | attribs | 属性 JSON |
| $5 | [1,2,3] | **已过滤后的**列表 ID |
| $6 | SubStatus | 订阅状态 |
| $7 | true | OverwriteUserInfo（overwrite=true） |
| $8 | true | OverwriteSubStatus（overwrite=true） |

**Blocklist 模式**（[upsert-blocklist-subscriber SQL](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L137-L149)）：

```sql
WITH sub AS (
    INSERT INTO subscribers (uuid, email, name, attribs, status)
    VALUES($1, $2, $3, $4, 'blocklisted')
    ON CONFLICT (email) DO UPDATE SET status='blocklisted', updated_at=NOW()
    RETURNING id
)
UPDATE subscriber_lists SET status='unsubscribed', updated_at=NOW()
    WHERE subscriber_id = (SELECT id FROM sub);
```

Blocklist 模式仅使用 $1-$4，不涉及列表 ID。

#### 3.5.3 队列关闭后的收尾逻辑

```go
// 队列关闭且无残余记录
if cur == 0 {
    s.im.setStatus(StatusFinished)
    s.im.opt.UpdateListDateStmt.Exec(pq.Array(listIDs))
    s.im.sendNotif(StatusFinished)
    return
}

// 队列关闭但有残余记录
if err := tx.Commit(); err != nil {
    tx.Rollback()
    s.im.setStatus(StatusFailed)
    s.im.sendNotif(StatusFailed)
    return
}
s.im.incrementImportCount(cur)
s.im.setStatus(StatusFinished)
s.im.opt.UpdateListDateStmt.Exec(pq.Array(listIDs))
s.im.sendNotif(StatusFinished)
```

---

### 3.6 阶段六：Stop 机制

**源码位置**：[importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L588-L602)

#### 3.6.1 Importer.Stop()

```go
func (im *Importer) Stop() {
    if im.getStatus() != StatusImporting {
        // 非 importing 状态：清除为 none（用于清除已完成的导入状态）
        im.Lock()
        im.status = Status{Status: StatusNone}
        im.Unlock()
        return
    }

    select {
    case im.stop <- true:     // 发送停止信号
        im.setStatus(StatusStopping)
    default:                  // 通道已有信号，不重复发送
    }
}
```

#### 3.6.2 Stop 信号的传播路径

```
用户点击停止
  │
  └─ HTTP DELETE /api/import/subscribers
       └─ StopImportSubscribers() → a.importer.Stop()
            │
            ├─ im.stop <- true      (信号发送)
            └─ setStatus("stopping")
                  │
                  ▼
            LoadCSV() 循环内:
            select {
            case <-s.im.stop:        ← 捕获信号
                close(s.subQueue)    ← 关闭队列
                return nil
            }

            Start() 消费完队列:
            ├─ Commit 残余批次
            ├─ setStatus("finished") ← ★ 覆盖 "stopping"
            ├─ UpdateListDateStmt
            └─ sendNotif("finished") ← ★ 触发 PostCB
```

---

### 3.7 阶段七：PostCB 回调

**源码位置**：[init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L640-L662)

```go
PostCB: func(subject string, data any) error {
    core.RefreshMatViews(true)                         // 刷新物化视图
    notifs.NotifySystem(subject, notifs.TplImport, data, nil)  // 发送管理员通知
    return nil
},
```

`RefreshMatViews` 刷新以下三个物化视图：

| 视图 | 用途 |
|------|------|
| `dashboard_charts` | 仪表板图表数据 |
| `dashboard_counts` | 仪表板计数统计 |
| `list_sub_stats` | 列表订阅者统计 |

`NotifySystem` 向系统管理员邮箱发送导入状态通知邮件。

---

## 4. 本场景完整时间线推演

### 4.1 初始条件

| 项目 | 值 |
|------|----|
| 用户角色 | 非 superadmin，拥有 `subscribers:import` 权限 |
| 列表权限 | 有 manage 权限：[1, 2, 3]；无 manage 权限：[4, 5] |
| 上传文件 | ZIP 包含 1 个 CSV，100001 行（含 header） |
| 模式 | subscribe + overwrite=true |
| 请求列表 | [1, 2, 3, 4, 5] |

### 4.2 逐帧推演

| 时间 | 事件 | 状态 | 数据影响 |
|------|------|------|----------|
| T0 | POST /api/import/subscribers 到达 | `none` | — |
| T1 | FilterListsByPerm 过滤 → ListIDs=[1,2,3] | `none` | 列表 4、5 被静默排除 |
| T2 | 文件落盘 /tmp/listmonk* | `none` | — |
| T3 | NewSession(opt) | `importing` | status.Total=0, Imported=0 |
| T4 | go sess.Start() 启动消费者 | `importing` | 消费者阻塞等待 subQueue |
| T5 | ExtractZIP 同步执行 | `importing` | CSV 提取到临时目录 |
| T6 | go sess.LoadCSV() 启动生产者 | `importing` | countLines → Total=100000 |
| T7 | LoadCSV 读取 header，开始逐行生产 | `importing` | — |
| T8 | 生产者推入第 1 行，消费者取出并 Exec | `importing` | cur=1, 事务 tx-1 开始 |
| ... | ... | ... | ... |
| T9 | 第 10000 行处理完毕，tx-1 Commit | `importing` | Imported=10000, 10000 行已写入 DB |
| T10 | 第 20000 行处理完毕，tx-2 Commit | `importing` | Imported=20000 |
| T11 | 第 30000 行处理完毕，tx-3 Commit | `importing` | Imported=30000 |
| T12 | 第 40000 行处理完毕，tx-4 Commit | `importing` | Imported=40000 |
| T13 | 第 50000 行处理完毕，tx-5 Commit | `importing` | Imported=50000 |
| T14 | **用户点击停止** → DELETE /api/import/subscribers | `stopping` | im.stop <- true |
| T15 | LoadCSV 循环检测到 stop 信号 | `stopping` | close(subQueue), 日志 "stop request received" |
| T16 | 生产者停止，此时可能有若干行已在 subQueue 中但未被消费 | `stopping` | — |
| T17 | Start() 继续消费 subQueue 中残余记录 | `stopping` | cur 继续增长 |
| T18 | 残余记录消费完毕（假设 cur=3756） | `stopping` | — |
| T19 | Start() 执行 tx.Commit()（残余批次） | `stopping` | **53756 行已写入 DB** |
| T20 | incrementImportCount(3756) | `stopping` | Imported=53756 |
| T21 | **setStatus("finished")** ★ | `finished` | 覆盖 "stopping" |
| T22 | **UpdateListDateStmt.Exec([1,2,3])** | `finished` | 列表 1/2/3 的 updated_at 更新 |
| T23 | **sendNotif("finished")** → PostCB | `finished` | 物化视图刷新 + 管理员通知邮件 |

---

## 5. 数据写入分析

### 5.1 已写入的数据

| 数据类型 | 说明 |
|----------|------|
| subscribers 表 | 约 50000~53756 行订阅者记录（取决于停止时机），仅插入到列表 [1,2,3] |
| subscriber_lists 表 | 对应的订阅关系记录，status 由 SubStatus 决定 |
| lists 表 | 列表 1/2/3 的 updated_at 被更新 |
| 物化视图 | dashboard_charts / dashboard_counts / list_sub_stats 被刷新 |

### 5.2 未写入的数据

| 数据类型 | 原因 |
|----------|------|
| 列表 4、5 的订阅关系 | FilterListsByPerm 已在入口处过滤，订阅者从未关联到这两个列表 |
| CSV 中未被读取的行（约 46000+ 行） | LoadCSV 收到 stop 信号后停止读取 |
| 当前未提交事务中的部分行（如有） | 若 Stop 时恰好有未提交的批次，该批次中的行未写入 |

### 5.3 数据不一致风险

| 风险 | 说明 |
|------|------|
| 部分导入 | 只有约 53% 的订阅者被导入，但状态报告为 "finished" |
| Overwrite 半执行 | 已存在的订阅者可能已被 overwrite 更新了 name/attribs/status，但后续行未被处理 |
| 物化视图与实际不一致 | 物化视图基于已提交数据刷新，但 Total=100000 与 Imported=53756 不匹配，前端显示进度为 53.7% |
| 列表统计偏差 | 列表 1/2/3 的订阅者数仅包含已导入部分，与 CSV 中的全量预期不符 |

---

## 6. 状态报告分析

### 6.1 GetStats 返回值

```json
{
    "name": "subscribers.zip",
    "total": 100000,
    "imported": 53756,
    "status": "finished"    // ★ 应为 "stopped"
}
```

### 6.2 日志输出

```
2026/05/31 10:00:00 processing 'subscribers.zip'
2026/05/31 10:00:01 extracting 'import.csv'
2026/05/31 10:00:01 extracted 'import.csv'
2026/05/31 10:00:05 imported 10000
2026/05/31 10:00:10 imported 20000
2026/05/31 10:00:15 imported 30000
2026/05/31 10:00:20 imported 40000
2026/05/31 10:00:25 imported 50000
2026/05/31 10:00:27 stop request received
2026/05/31 10:00:28 imported finished    // ★ 无法区分是正常完成还是停止后完成
```

### 6.3 问题汇总

| 问题 | 描述 |
|------|------|
| 状态误报 | 停止后状态为 "finished" 而非 "stopped"，前端无法区分 |
| 通知误发 | 即使是部分导入，管理员仍收到 "Finished" 通知 |
| 物化视图冗余刷新 | 停止导致的半完成导入仍触发全量物化视图刷新 |
| 进度不可靠 | Total=100000 暗示预期导入 10 万行，但实际仅导入约 5.3 万行 |

---

## 7. 越权导入风险分析

### 7.1 当前防护机制

| 防护层 | 位置 | 机制 |
|--------|------|------|
| API 权限 | handlers.go | `pm(a.ImportSubscribers, "subscribers:import")` — 需要全局 `subscribers:import` 权限 |
| 列表权限 | import.go:34 | `FilterListsByPerm(PermTypeManage, opt.ListIDs)` — 过滤无 manage 权限的列表 |
| 空列表拦截 | import.go:35 | 过滤后列表为空且非 blocklist 模式 → 403 |
| SQL 参数化 | upsert-subscriber | ListIDs 通过 `$5::INT[]` 参数传入，无法注入 |

### 7.2 潜在越权路径

#### 7.2.1 静默降级（当前存在）

用户指定列表 [1,2,3,4,5]，但只有 [1,2,3] 的 manage 权限。FilterListsByPerm **静默**返回 [1,2,3]，用户不知道列表 4、5 被排除。

**风险**：用户以为数据导入了 5 个列表，实际只导入了 3 个，造成数据完整性错觉。

#### 7.2.2 Blocklist 模式绕过列表权限（当前存在）

```go
if len(opt.ListIDs) == 0 && opt.Mode != subimporter.ModeBlocklist {
    return echo.NewHTTPError(http.StatusForbidden, ...)
}
```

当 `mode=blocklist` 时，即使 `ListIDs` 为空也允许执行。Blocklist SQL 不使用列表 ID，而是直接将订阅者状态设为 `blocklisted` 并将其所有订阅标记为 `unsubscribed`。

**风险**：拥有 `subscribers:import` 权限但无任何列表 manage 权限的用户，仍可通过 blocklist 模式影响所有列表中的订阅者状态。

#### 7.2.3 Overwrite 跨权限影响（当前存在）

upsert-subscriber SQL 使用 `ON CONFLICT (email) DO UPDATE`。当 `OverwriteUserInfo=true` 时，会覆盖已有订阅者的 name 和 attribs，即使该订阅者属于用户无权管理的列表。

**风险**：用户 A 对列表 [1,2,3] 有 manage 权限，导入时 overwrite=true。如果列表 [4,5] 中已有订阅者 `alice@example.com`，用户 A 的导入会覆盖 alice 的 name/attribs/substatus，即使用户 A 对列表 [4,5] 无权限。

#### 7.2.4 TOCTOU 竞态（理论风险）

权限检查在 ImportSubscribers handler 中一次性完成。导入过程可能持续数分钟甚至数小时。期间：
- 管理员可能撤销用户对某列表的权限
- 列表可能被删除

但 SQL 语句中列表 ID 在 session 创建时已固定，不会重新校验。

---

## 8. 修复建议

### 8.1 Stop 后状态应区分 "stopped" 与 "finished"

**问题**：Start() 在队列关闭后统一设置 `StatusFinished`，不区分正常完成和被停止。

**建议方案**：在 Session 中增加 `stopped` 标记：

```go
type Session struct {
    im       *Importer
    subQueue chan SubReq
    log      *log.Logger
    opt      SessionOpt
    stopped  bool          // 新增
}
```

LoadCSV 检测到 stop 信号时设置 `s.stopped = true`。Start() 收尾时检查此标记：

```go
finalStatus := StatusFinished
if s.stopped {
    finalStatus = StatusStopping  // 或新增 StatusStopped
}
s.im.setStatus(finalStatus)
s.im.sendNotif(finalStatus)
```

### 8.2 PostCB 应区分停止与完成

**问题**：无论正常完成还是被停止，PostCB 都会触发物化视图刷新和通知。

**建议方案**：PostCB 回调增加状态参数，仅在 `finished` 时执行刷新和通知：

```go
PostCB: func(status string, data any) error {
    if status == StatusFinished {
        core.RefreshMatViews(true)
        notifs.NotifySystem(subject, notifs.TplImport, data, nil)
    }
    return nil
},
```

或者在 `sendNotif` 中根据状态决定是否调用 PostCB。

### 8.3 列表过滤结果应反馈给用户

**问题**：FilterListsByPerm 静默丢弃无权限列表，用户无感知。

**建议方案**：在 ImportSubscribers handler 中比较过滤前后的列表差异：

```go
originalListIDs := opt.ListIDs
opt.ListIDs = user.FilterListsByPerm(auth.PermTypeManage, opt.ListIDs)

if len(opt.ListIDs) < len(originalListIDs) {
    // 在响应中包含被过滤的列表信息
    // 或返回 207 Multi-Status / 206 Partial
}
```

### 8.4 Overwrite 应受列表权限约束

**问题**：overwrite 模式会更新已有订阅者的信息，即使该订阅者属于用户无权管理的列表。

**建议方案**：在 upsert-subscriber SQL 中，仅当订阅者的现有订阅列表与用户有权限的列表有交集时，才执行 overwrite 更新。或在 SQL 层面增加列表权限过滤：

```sql
-- 仅对用户有权限的列表执行订阅关系更新
WITH sub AS (...),
subs AS (
    INSERT INTO subscriber_lists (subscriber_id, list_id, status)
    SELECT sub.id, listID, $6::subscription_status
    FROM sub, UNNEST($5::INT[]) AS listID
    ON CONFLICT (subscriber_id, list_id) DO UPDATE SET ...
)
-- 订阅者基础信息更新也应有条件
```

更实际的方案是在 upsert 逻辑中区分：仅更新 subscriber_lists 中用户有权限的列表关联，不更新 subscribers 表中可能影响其他列表的字段。

### 8.5 Blocklist 模式应增加权限检查

**问题**：blocklist 模式绕过列表权限检查，可影响所有列表中的订阅者。

**建议方案**：

```go
if opt.Mode == subimporter.ModeBlocklist {
    if !user.HasPerm(auth.PermSubscribersManage) {
        return echo.NewHTTPError(http.StatusForbidden, ...)
    }
}
```

或要求 blocklist 模式也需要至少一个列表的 manage 权限。

### 8.6 Stop 后应回滚当前事务批次

**问题**：Stop 后 Start() 仍会提交当前未提交的批次。

**建议方案**：Start() 应检查 stop 状态，在队列关闭后若检测到 stop 信号，回滚当前事务而非提交：

```go
if s.stopped && cur > 0 {
    tx.Rollback()
    s.im.setStatus(StatusStopping)
    s.im.sendNotif(StatusStopping)
    return
}
```

---

## 9. 完整状态机图

```
                              ┌──────────────────────────────────────────┐
                              │        ImportSubscribers Handler         │
                              │  ┌───────────────────────────────────┐   │
                              │  │ 1. 并发守卫 (StatusImporting?)    │   │
                              │  │ 2. 解析 params                    │   │
                              │  │ 3. FilterListsByPerm ★权限过滤   │   │
                              │  │ 4. Mode/Status/Delim 校验         │   │
                              │  │ 5. 文件落盘                       │   │
                              │  └──────────┬────────────────────────┘   │
                              │             │                             │
                              │             ▼                             │
                              │  ┌───────────────────────────────────┐   │
                              │  │      NewSession(opt)               │   │
                              │  │  status = "importing"              │   │
                              │  │  Overwrite → OverwriteUserInfo    │   │
                              │  │               + OverwriteSubStatus │   │
                              │  └──────────┬────────────────────────┘   │
                              │             │                             │
                              │      ┌──────┴──────┐                     │
                              │      │             │                     │
                              │      ▼             ▼                     │
                              │  .csv 文件    .zip 文件                  │
                              │      │             │                     │
                              │      │        ExtractZIP()               │
                              │      │        (同步, 可能→failed)        │
                              │      │             │                     │
                              │      └──────┬──────┘                     │
                              │             │                             │
                              └─────────────┼─────────────────────────────┘
                                            │
                     ┌──────────────────────┼──────────────────────┐
                     │                      │                      │
                     ▼                      ▼                      ▼
           ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
           │  go Start()     │    │ go LoadCSV()    │    │  stop channel   │
           │  (消费者)       │    │ (生产者)         │    │  (停止信号)     │
           └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
                    │                      │                      │
                    │    subQueue (10000)   │                      │
                    │◄─────────────────────┤                      │
                    │                      │                      │
                    │                      ├── for each row:      │
                    │                      │   ├── stop? ─────────┤
                    │                      │   │   close(queue)   │
                    │                      │   │   return         │
                    │                      │   ├── ValidateFields │
                    │                      │   └── queue <- sub   │
                    │                      │                      │
                    │                      └── close(queue)       │
                    │                             (正常完成)       │
                    │                                              │
                    ├── for sub := range queue:                    │
                    │   ├── tx.Begin (新批次)                     │
                    │   ├── stmt.Exec                             │
                    │   └── cur%10000==0 → Commit                 │
                    │                                              │
                    ├── queue closed:                              │
                    │   ├── Commit 残余批次                        │
                    │   ├── setStatus("finished") ★                │
                    │   ├── UpdateListDateStmt                     │
                    │   └── sendNotif → PostCB                     │
                    │        ├── RefreshMatViews                   │
                    │        └── NotifySystem                      │
                    │                                              │
                    └── error → Rollback → setStatus("failed")
```

---

## 10. 关键源码索引

| 组件 | 文件 | 关键行 |
|------|------|--------|
| ImportSubscribers Handler | [import.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/import.go#L18-L119) | L18-L119 |
| StopImportSubscribers Handler | [import.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/import.go#L132-L138) | L132-L138 |
| PostCB 初始化 | [init.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L640-L662) | L640-L662 |
| Importer 结构体 | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L50-L67) | L50-L67 |
| NewSession | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L167-L194) | L167-L194 |
| Start (消费者) | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L273-L363) | L273-L363 |
| Stop | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L588-L602) | L588-L602 |
| ExtractZIP | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L373-L449) | L373-L449 |
| LoadCSV | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L452-L585) | L452-L585 |
| ValidateFields | [importer.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/subimporter/importer.go#L639-L664) | L639-L664 |
| FilterListsByPerm | [models.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/auth/models.go#L271-L312) | L271-L312 |
| upsert-subscriber SQL | [subscribers.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L114-L135) | L114-L135 |
| upsert-blocklist-subscriber SQL | [subscribers.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/subscribers.sql#L137-L149) | L137-L149 |
| RefreshMatViews | [core.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/core/core.go#L91-L96) | L91-L96 |
| NotifySystem | [notifs.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/notifs/notifs.go#L66-L68) | L66-L68 |
| update-lists-date SQL | [lists.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/lists.sql#L76-L77) | L76-L77 |
