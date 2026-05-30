# Listmonk Campaign 异步发送生命周期深度分析

## 1. 整体架构概览

```
main()
  └── go mgr.Run()
        ├── go scanCampaigns()  // 每5秒扫描一次，发现新的 scheduled/running campaign
        │     └── newPipe()     // 为每个 campaign 创建独立的管道
        │           └── go func() { wg.Wait(); cleanup() }  // 等待所有消息处理完成后清理
        ├── go worker() * N     // N个并发worker消费消息队列
        └── for p := range nextPipes { p.NextSubscribers() }  // 主循环拉取订阅者批次
```

---

## 2. 启动流程详解

### 2.1 main 启动入口

[main.go:257-262](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/main.go#L257-L262)

```go
// Start cronjobs.
initCron(core, db)

// Start the campaign manager workers.
go mgr.Run()
```

**关键点**：
- `initCron()` 主要用于慢查询缓存和数据库 vacuum，**不负责 campaign 的定时调度**
- campaign 的调度由 `mgr.Run()` 内部的 `scanCampaigns()` 负责

### 2.2 mgr.Run() 主循环

[manager.go:266-316](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L266-L316)

```go
func (m *Manager) Run() {
    // 1. 启动扫描协程
    if m.cfg.ScanCampaigns {
        go m.scanCampaigns(m.cfg.ScanInterval)  // 默认每5秒
    }

    // 2. 启动N个worker（并发数由 app.concurrency 配置）
    for i := 0; i < m.cfg.Concurrency; i++ {
        go m.worker()
    }

    // 3. 主循环：处理订阅者批次
    for p := range m.nextPipes {
        has, err := p.NextSubscribers()
        // ...
    }
}
```

---

## 3. scanCampaigns 扫描机制

### 3.1 扫描周期与逻辑

[manager.go:423-459](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L423-L459)

**扫描频率**：每5秒一次（`ScanInterval = time.Second * 5`）

**SQL查询逻辑**：[campaigns.sql:174-232](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L174-L232)

```sql
WHERE (status='running' OR (status='scheduled' AND NOW() >= campaigns.send_at))
AND NOT(campaigns.id = ANY($1::INT[]))  -- 排除正在处理的campaign
```

**关键流程**：
1. 调用 `getCurrentCampaigns()` 获取内存中正在运行的 campaign IDs 和已发送计数
2. 执行 `NextCampaigns` SQL 查询：
   - 找出所有 `running` 状态的 campaign
   - 找出所有 `scheduled` 状态且已到发送时间的 campaign
   - **排除当前内存中正在运行的 campaign**（防止重复处理）
3. 对每个符合条件的 campaign 调用 `newPipe()` 创建管道
4. 将管道放入 `nextPipes` channel 等待主循环处理

### 3.2 getCurrentCampaigns 的重要作用

[manager.go:562-581](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L562-L581)

```go
func (m *Manager) getCurrentCampaigns() ([]int64, []int64) {
    m.pipesMut.RLock()
    defer m.pipesMut.RUnlock()
    
    for _, p := range m.pipes {
        ids = append(ids, int64(p.camp.ID))
        counts = append(counts, p.sent.Load())
        p.sent.Store(0)  // 重置内存计数器
    }
    return ids, counts
}
```

**⚠️ 关键机制**：
- 返回当前内存中所有活跃的 campaign ID
- 返回每个 campaign 的已发送计数
- **重置内存中的 sent 计数器**
- 这些计数会在 `NextCampaigns` 查询中通过 `UPDATE campaigns SET sent = sent + uc.sent_count` 累加到数据库

---

## 4. newPipe 管道创建

### 4.1 管道结构

[pipe.go:13-24](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L13-L24)

```go
type pipe struct {
    camp       *models.Campaign
    rate       *ratecounter.RateCounter  // 每分钟发送速率统计
    wg         *sync.WaitGroup           // 等待所有消息处理完成
    sent       atomic.Int64              // 已发送计数（内存中，会被getCurrentCampaigns重置）
    lastID     atomic.Uint64             // 最后处理的订阅者ID
    errors     atomic.Uint64             // 错误计数
    stopped    atomic.Bool                // 是否已停止标记
    withErrors atomic.Bool                // 是否因错误而停止
    
    m *Manager
}
```

### 4.2 管道创建流程

[pipe.go:26-70](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L26-L70)

```go
func (m *Manager) newPipe(c *models.Campaign) (*pipe, error) {
    // 1. 验证messenger是否存在
    // 2. 编译模板
    // 3. 加载附件
    
    p := &pipe{...}
    p.wg.Add(1)  // ⚠️ 关键：先+1，确保Wait()立即阻塞
    
    // 启动清理协程
    go func() {
        p.wg.Wait()      // 等待所有消息处理完成
        p.cleanup()      // 执行清理
    }()
    
    m.pipes[c.ID] = p   // 加入活跃管道map
    return p, nil
}
```

**⚠️ 关键设计**：
- `wg.Add(1)` 是一个**伪计数器**，确保在第一个消息创建前 `Wait()` 就已经阻塞
- 这个 +1 会在 `NextSubscribers()` 返回 `false`（没有更多订阅者）时被 `wg.Done()` 抵消
- 这是整个生命周期中最容易出问题的地方之一

---

## 5. NextSubscribers 订阅者批次拉取

### 5.1 批次处理逻辑

[pipe.go:72-134](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L72-L134)

```go
func (p *pipe) NextSubscribers() (bool, error) {
    // 1. 从DB拉取下一批订阅者
    subs, err := p.m.store.NextSubscribers(p.camp.ID, p.m.cfg.BatchSize)
    if len(subs) == 0 {
        return false, nil  // 没有更多订阅者
    }
    
    // 2. 滑动窗口限流检查
    hasSliding := p.m.cfg.SlidingWindow && ...
    
    // 3. 为每个订阅者创建消息并推入队列
    for _, s := range subs {
        msg, err := p.newMessage(s)  // wg.Add(1)
        p.m.campMsgQ <- msg           // 阻塞式推送
        
        // 4. 滑动窗口限流
        if hasSliding {
            p.m.slidingCount++
            if p.m.slidingCount >= p.m.cfg.SlidingWindowRate {
                time.Sleep(wait)  // 阻塞等待窗口重置
                p.m.slidingCount = 0
            }
        }
    }
    return true, nil
}
```

### 5.2 数据库拉取逻辑

[manager_store.go:45-67](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/manager_store.go#L45-L67)

```go
func (s *store) NextSubscribers(campID, limit int) ([]models.Subscriber, error) {
    // 1. 获取campaign的运行时元数据（last_subscriber_id, max_subscriber_id, list_ids）
    var camps []runningCamp
    s.queries.GetRunningCampaign.Select(&camps, campID)
    
    // 2. 拉取批次订阅者（会自动更新last_subscriber_id）
    s.queries.NextCampaignSubscribers.Select(&out, ...)
}
```

**SQL关键逻辑**：[campaigns.sql:318-372](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql#L318-L372)

```sql
WITH subs AS (
    SELECT DISTINCT s.id
    FROM subscriber_lists sl
    JOIN subscribers s ON s.id = sl.subscriber_id
    WHERE
        sl.list_id = ANY($5::INT[])
        AND s.id > $3    -- last_subscriber_id（上一批的终点）
        AND s.id <= $4   -- max_subscriber_id（campaign的上限）
    ORDER BY s.id LIMIT $6
),
u AS (
    -- ⚠️ 自动更新数据库中的last_subscriber_id
    UPDATE campaigns
    SET last_subscriber_id = (SELECT MAX(id) FROM subs)
    WHERE id=$1
)
```

**⚠️ 关键副作用**：
- 每次拉取都会**自动更新数据库中的 `last_subscriber_id`**
- 这意味着即使进程崩溃，重启后也不会重复发送
- 但也意味着：**如果一批订阅者在内存中处理失败，他们将永远不会被重试**

---

## 6. newMessage 消息创建

[pipe.go:169-182](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L169-L182)

```go
func (p *pipe) newMessage(s models.Subscriber) (CampaignMessage, error) {
    msg, err := p.m.NewCampaignMessage(p.camp, s)
    if err != nil {
        return msg, err
    }
    
    msg.pipe = p
    p.wg.Add(1)  // ⚠️ 每个消息对应一个wg计数
    
    return msg, nil
}
```

---

## 7. Worker 消息消费与发送

### 7.1 Worker 主循环

[manager.go:463-558](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L463-L558)

```go
func (m *Manager) worker() {
    numMsg := 0  // 每秒速率限制计数器
    for {
        select {
        case msg, ok := <-m.campMsgQ:
            // 1. 检查campaign是否已停止
            if msg.pipe != nil && msg.pipe.stopped.Load() {
                msg.pipe.wg.Done()  // 直接减少计数，不发送
                continue
            }
            
            // 2. 每秒速率限制（每个worker独立计数）
            if numMsg >= m.cfg.MessageRate {
                time.Sleep(time.Second)
                numMsg = 0
            }
            numMsg++
            
            // 3. 实际发送
            err := m.messengers[msg.Campaign.Messenger].Push(out)
            
            // 4. 消息完成处理
            if msg.pipe != nil {
                msg.pipe.wg.Done()  // ⚠️ 无论成功失败，都减少计数
                
                if err != nil {
                    msg.pipe.OnError()  // 错误处理
                } else {
                    // 更新发送统计
                    msg.pipe.rate.Incr(1)
                    msg.pipe.sent.Add(1)
                }
            }
        }
    }
}
```

### 7.2 速率限制机制分析

**两种限流方式**：

| 限流类型 | 作用范围 | 配置项 | 实现位置 |
|---------|---------|--------|---------|
| 每秒速率限制 | 每个worker独立 | `app.message_rate` | worker() 内的 `numMsg` 计数器 |
| 滑动窗口限流 | 全局所有worker | `app.message_sliding_window` | NextSubscribers() 内的 `slidingCount` |

**⚠️ 重要发现**：
- 每秒速率限制是**每个worker独立计数**的
- 如果 `concurrency=10` 且 `message_rate=100`，实际每秒可能发送 10*100=1000 条
- 滑动窗口限流是**在拉取订阅者时阻塞**，而不是在发送时阻塞

### 7.3 错误回调机制 OnError

[pipe.go:136-151](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L136-L151)

```go
func (p *pipe) OnError() {
    if p.m.cfg.MaxSendErrors < 1 {
        return
    }
    
    count := p.errors.Add(1)
    if int(count) < p.m.cfg.MaxSendErrors {
        return
    }
    
    p.Stop(true)  // 标记为因错误停止
    p.m.log.Printf("error count exceeded %d. pausing campaign %s", ...)
}
```

**⚠️ 关键行为**：
- 错误计数达到阈值后，只是设置 `stopped=true` 和 `withErrors=true`
- **不会立即停止发送**，已在队列中的消息仍会被 worker 消费（但会被跳过）
- **不会重试失败的订阅者**，失败的订阅者就此丢失

---

## 8. StopCampaign 停止机制

### 8.1 Stop 方法

[pipe.go:153-167](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L153-L167)

```go
func (p *pipe) Stop(withErrors bool) {
    if p.stopped.Load() {
        return  // 幂等性：已停止则什么都不做
    }
    
    if withErrors {
        p.withErrors.Store(true)
    }
    p.stopped.Store(true)
}
```

**⚠️ 重要特性**：
- `Stop()` 只是**设置一个原子标记**，不会中断任何正在进行的操作
- worker 在消费消息时会检查这个标记，如果已停止则跳过发送
- 已在 `campMsgQ` 队列中的消息都会被遍历一遍，然后 `wg.Done()`
- 这是一个**优雅停止**，但不是**立即停止**

### 8.2 外部调用 StopCampaign

[manager.go:405-412](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L405-L412)

```go
func (m *Manager) StopCampaign(id int) {
    m.pipesMut.RLock()
    if p, ok := m.pipes[id]; ok {
        p.Stop(false)  // 手动停止，不是因错误停止
    }
    m.pipesMut.RUnlock()
}
```

---

## 9. UpdateCampaignCounts 计数更新机制

### 9.1 内存计数与数据库同步

[manager.go:562-581](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go#L562-L581)

```go
func (m *Manager) getCurrentCampaigns() ([]int64, []int64) {
    for _, p := range m.pipes {
        ids = append(ids, int64(p.camp.ID))
        counts = append(counts, p.sent.Load())
        p.sent.Store(0)  // ⚠️ 读取后立即重置
    }
    return ids, counts
}
```

**同步时序**：
```
scanCampaigns (每5秒)
  └── getCurrentCampaigns()
        ├── 读取所有p.sent的值
        ├── 重置p.sent为0
        └── 将值传递给NextCampaigns SQL
              └── UPDATE campaigns SET sent = sent + $sent_count
```

### 9.2 cleanup 中的最终更新

[pipe.go:194-197](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L194-L197)

```go
func (p *pipe) cleanup() {
    // 最终更新计数
    p.m.store.UpdateCampaignCounts(p.camp.ID, 0, int(p.sent.Load()), int(p.lastID.Load()))
    // ...
}
```

---

## 10. cleanup 清理与状态转换

### 10.1 cleanup 触发条件

`cleanup()` 仅在 `wg.Wait()` 返回后被调用，即：
- 所有消息的 `wg.Add(1)` 都被 `wg.Done()` 抵消
- 包括最开始的那个伪 +1 也被抵消（在 `NextSubscribers()` 返回 `false` 时）

### 10.2 cleanup 完整逻辑

[pipe.go:187-239](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L187-L239)

```go
func (p *pipe) cleanup() {
    defer delete(m.pipes, p.camp.ID)  // 从活跃管道移除
    
    // 1. 更新最终计数
    p.m.store.UpdateCampaignCounts(...)
    
    // 2. 情况A：因错误而停止 → 状态设为 paused
    if p.withErrors.Load() {
        p.m.store.UpdateCampaignStatus(p.camp.ID, models.CampaignStatusPaused)
        return
    }
    
    // 3. 情况B：手动停止（pause/cancel）→ 什么都不做，直接返回
    if p.stopped.Load() {
        p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
        return  // ⚠️ 不更新数据库状态！
    }
    
    // 4. 情况C：正常完成 → 设为 finished
    c, err := p.m.store.GetCampaign(p.camp.ID)  // 重新从DB获取最新状态
    if c.Status == models.CampaignStatusRunning || 
       c.Status == models.CampaignStatusScheduled {
        c.Status = models.CampaignStatusFinished
        p.m.store.UpdateCampaignStatus(p.camp.ID, models.CampaignStatusFinished)
    }
}
```

---

## 11. 核心问题分析：为什么 Campaign 没有自动 Finished

### 11.1 可能性1：WaitGroup 计数不匹配

**问题场景**：
```go
// newPipe 中
p.wg.Add(1)  // +1

// NextSubscribers 返回 false 时
p.wg.Done()  // -1

// 每个消息
p.wg.Add(1)   // +1
p.wg.Done()  // -1 （在worker中）
```

**风险点**：
- 如果 `NextSubscribers()` 因为错误提前返回，但是没有正确调用 `wg.Done()`
- 如果消息创建时出错，但 `wg.Add(1)` 已经执行

### 11.2 可能性2：手动 Stop 后数据库状态不更新

**关键代码**：[pipe.go:211-215](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L211-L215)

```go
// The campaign was manually stopped (pause, cancel).
if p.stopped.Load() {
    p.m.log.Printf("stop processing campaign (%s)", p.camp.Name)
    return  // ⚠️ 直接返回，不更新数据库状态！
}
```

**问题**：
- 如果用户在 UI 上点击了 Pause，`StopCampaign(id)` 会被调用
- 这会设置 `stopped=true` 但 `withErrors=false`
- 在 `cleanup()` 中，会直接 return，**不更新数据库中的状态**
- 但此时数据库中 campaign 可能仍处于 `running` 状态

### 11.3 可能性3：scanCampaigns 重复检测问题

**关键逻辑**：
```sql
WHERE (status='running' OR (status='scheduled' AND NOW() >= campaigns.send_at))
AND NOT(campaigns.id = ANY($1::INT[]))  -- 排除正在处理的campaign
```

**问题场景**：
1. 一个 running campaign 的管道因为某种原因（如队列满）被 Stop
2. Stop 后 cleanup 被触发，但因为是手动 stop，数据库状态没更新
3. 管道从 `m.pipes` 中被删除
4. 下一次 scanCampaigns 时，因为：
   - 数据库状态仍是 `running`
   - 该 ID 不在 `currentIDs` 中（管道已被删除）
5. 会**重新创建一个新的管道**，但此时 `last_subscriber_id` 已经到了末尾
6. `NextSubscribers()` 返回 `false`，伪计数器被 Done
7. 但是...新管道的 `wg.Wait()` 会立即触发 cleanup
8. cleanup 中因为是正常完成（stopped=false），会查询数据库
9. 但如果此时数据库状态被其他逻辑改变了...

### 11.4 可能性4：滑动窗口阻塞导致的状态不一致

[pipe.go:107-130](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go#L107-L130)

```go
if hasSliding {
    diff := time.Since(p.m.slidingStart)
    if diff >= p.m.cfg.SlidingWindowDuration {
        p.m.slidingStart = time.Now()
        p.m.slidingCount = 0
    }
    
    p.m.slidingCount++
    if p.m.slidingCount >= p.m.cfg.SlidingWindowRate {
        time.Sleep(wait)  // ⚠️ 这里会阻塞整个主循环！
        p.m.slidingCount = 0
    }
}
```

**问题**：
- 滑动窗口的 `time.Sleep()` 是在**主循环线程**中执行的
- 如果窗口设置很激进（如每分钟只允许发送10条），主循环会被长时间阻塞
- 在此期间：
  - `scanCampaigns` 仍在运行
  - `getCurrentCampaigns` 会读取并重置 `sent` 计数
  - 但因为主循环被阻塞，新的批次无法拉取
  - 可能导致计数更新和实际发送不同步

### 11.5 可能性5：并发配置过低导致发送速度慢

| 配置项 | 影响 | 过低的后果 |
|-------|------|-----------|
| `app.concurrency` | worker数量 | 消费campMsgQ的速度慢，队列积压 |
| `app.message_rate` | 每个worker每秒发送数 | 单个worker发送速度受限 |
| `app.batch_size` | 每次拉取的订阅者数 | 频繁查询数据库，拉取效率低 |

**发送速度计算公式（理论上限）**：
```
理论最大发送速度 = concurrency * message_rate (条/秒)
```

如果实际发送速度低于这个值，可能原因：
1. messenger.Push() 本身很慢（如SMTP服务器响应慢）
2. 滑动窗口限制了全局速度
3. campMsgQ 队列太小导致阻塞

---

## 12. 完整生命周期时序图

```
时间轴 →

T0: main() 启动
    ├─ initCron() 启动
    └─ go mgr.Run() 启动
        ├─ go scanCampaigns() 启动
        └─ go worker() * N 启动

T5s: 第一次 scanCampaigns 扫描
    ├─ 发现 scheduled campaign（send_at 已到）
    ├─ newPipe() 创建管道
    │   ├─ wg.Add(1) 伪计数
    │   └─ go func() { wg.Wait(); cleanup() }
    └─ nextPipes <- p

T5s+ε: 主循环从 nextPipes 取出 p
    └─ p.NextSubscribers()
        ├─ DB查询拉取 BatchSize 个订阅者
        ├─ 为每个订阅者 newMessage() → wg.Add(1)
        ├─ 推入 campMsgQ
        └─ 返回 true，p 重新入队 nextPipes

T5s+ε: Worker 从 campMsgQ 消费
    ├─ 检查 stopped 标记
    ├─ 速率限制检查
    ├─ messenger.Push() 发送
    ├─ wg.Done()
    ├─ 成功: sent.Add(1)
    └─ 失败: OnError()

... 重复拉取批次 ...

Tn: 最后一次 NextSubscribers()
    ├─ DB查询返回0个订阅者
    ├─ 返回 false
    └─ wg.Done() （抵消最开始的伪计数+1）

Tn+ε: wg.Wait() 返回
    └─ cleanup() 执行
        ├─ UpdateCampaignCounts()
        ├─ 检查 withErrors? → paused
        ├─ 检查 stopped? → return（不更新DB）
        └─ 正常结束 → UpdateCampaignStatus(finished)
```

---

## 13. 关键代码位置索引

| 功能模块 | 文件位置 | 关键行号 |
|---------|---------|---------|
| Manager 启动 | [manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go) | L266-L316 |
| Campaign 扫描 | [manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go) | L423-L459 |
| Worker 消息消费 | [manager.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/manager.go) | L463-L558 |
| 管道创建 | [pipe.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go) | L26-L70 |
| 订阅者批次拉取 | [pipe.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go) | L72-L134 |
| 清理与状态转换 | [pipe.go](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/manager/pipe.go) | L187-L239 |
| NextCampaigns SQL | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L174-L232 |
| NextCampaignSubscribers SQL | [campaigns.sql](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/campaigns.sql) | L318-L372 |

---

## 14. 调试建议

如果遇到 campaign 无法自动 finished 的问题，建议按以下顺序排查：

1. **检查日志**：搜索 `"stop processing campaign"` 或 `"set campaign.*to paused"`
2. **检查 WaitGroup 状态**：确认所有消息的 wg.Done() 都被正确调用
3. **检查数据库状态**：
   ```sql
   SELECT id, name, status, sent, to_send, last_subscriber_id, max_subscriber_id 
   FROM campaigns WHERE status IN ('running', 'scheduled');
   ```
4. **检查是否触发了错误阈值**：查看 `"error count exceeded"` 日志
5. **检查并发配置**：确认 concurrency、message_rate、batch_size 是否合理
6. **检查滑动窗口配置**：确认是否因为窗口过小导致主循环长时间阻塞
