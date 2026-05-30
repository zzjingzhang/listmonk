# Listmonk 安装与升级 Schema/Settings 一致性保证机制深度分析

## 1. 问题描述

从旧版本升级到新版本的实例跳过了部分 migration，导致：
- `settings.smtp` 缺少 `msg_retry_delay` 字段
- `schema.sql` 里的默认结构与实际 DB 数据不一致
- 启动时 `checkSchema` 通过，但发送邮件时出现异常

本文从核心代码路径深入分析安装与升级机制，并给出解决方案。

---

## 2. 核心代码路径分析

### 2.1 安装流程 (install)

**入口**：[main.go:146-150](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/main.go#L146-L150)

```go
if ko.Bool("install") {
    install(migList[len(migList)-1].version, db, fs, !ko.Bool("yes"), ko.Bool("idempotent"))
    os.Exit(0)
}
```

**执行流程**：[install.go:20-118](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/install.go#L20-L118)

1. **幂等检查**：检查 `settings` 表是否存在，存在则跳过
2. **Schema 安装**：调用 `installSchema()` 执行完整的 `schema.sql`
3. **数据初始化**：创建示例列表、订阅者、模板、活动等
4. **版本记录**：记录最新的 migration 版本

**关键函数**：[install.go:120-133](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/install.go#L120-L133)
```go
func installSchema(curVer string, db *sqlx.DB, fs stuffbin.FileSystem) error {
    q, err := fs.Read("/schema.sql")  // 读取完整schema
    if err != nil { return err }
    if _, err := db.Exec(string(q)); err != nil { return err }
    return recordMigrationVersion(curVer, db)  // 记录最新版本
}
```

### 2.2 升级流程 (upgrade)

**入口**：[main.go:162-168](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/main.go#L162-L168)

**执行流程**：[upgrade.go:53-99](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/upgrade.go#L53-L99)

1. **获取待执行迁移**：调用 `getPendingMigrations()`
2. **逐个执行迁移**：按版本顺序执行每个 migration 函数
3. **记录版本**：每个 migration 成功后记录版本号

**关键逻辑**：
```go
func upgrade(db *sqlx.DB, fs stuffbin.FileSystem, prompt bool, record bool) {
    _, toRun, err := getPendingMigrations(db)  // 获取待执行列表
    // ...
    for _, m := range toRun {
        if err := m.fn(db, fs, ko, lo); err != nil {  // 执行迁移
            lo.Fatalf("error running migration %s: %v", m.version, err)
        }
        if record {  // nightly版本不记录
            recordMigrationVersion(m.version, db)
        }
    }
}
```

### 2.3 migList - 迁移列表

**定义**：[upgrade.go:28-48](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/upgrade.go#L28-L48)

```go
var migList = []migFunc{
    {"v0.4.0", migrations.V0_4_0},
    {"v0.7.0", migrations.V0_7_0},
    // ... 中间版本 ...
    {"v6.0.0", migrations.V6_0_0},
    {"v6.1.0", migrations.V6_1_0},
    {"v6.2.0", migrations.V6_2_0},  // 包含msg_retry_delay迁移
}
```

**特点**：
- 按语义版本严格排序
- 每个版本对应 `internal/migrations/` 目录下的一个 Go 文件
- 每个迁移函数要求是**幂等**的

### 2.4 recordMigrationVersion - 版本记录

**实现**：[install.go:281-286](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/install.go#L281-L286)

```go
func recordMigrationVersion(ver string, db *sqlx.DB) error {
    _, err := db.Exec(fmt.Sprintf(`INSERT INTO settings (key, value)
    VALUES('migrations', '["%s"]'::JSONB)
    ON CONFLICT (key) DO UPDATE SET value = settings.value || EXCLUDED.value`, ver))
    return err
}
```

**存储方式**：
- 存储在 `settings` 表中，`key = 'migrations'`
- 值为 JSONB 数组，按执行顺序追加版本号
- 例如：`["v0.4.0", "v0.7.0", ..., "v6.2.0"]`

### 2.5 getPendingMigrations - 待执行迁移计算

**实现**：[upgrade.go:123-142](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/upgrade.go#L123-L142)

```go
func getPendingMigrations(db *sqlx.DB) (string, []migFunc, error) {
    lastVer, err := getLastMigrationVersion(db)
    // ...
    var toRun []migFunc
    for i, m := range migList {
        if semver.Compare(m.version, lastVer) > 0 {  // 语义版本比较
            toRun = migList[i:]  // 返回所有版本号大于lastVer的迁移
            break
        }
    }
    return lastVer, toRun, nil
}
```

**获取最后版本**：[upgrade.go:146-158](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/upgrade.go#L146-L158)
```go
func getLastMigrationVersion(db *sqlx.DB) (string, error) {
    var v string
    if err := db.Get(&v, `
        SELECT COALESCE(
            (SELECT value->>-1 FROM settings WHERE key='migrations'),
        'v0.0.0')`); err != nil {
        // ...
    }
    return v, nil
}
```

**⚠️ 关键问题**：
- 只通过版本号比较，**不验证实际数据完整性**
- 如果版本记录被手动篡改或迁移执行失败但版本被错误记录，会导致后续迁移跳过

### 2.6 checkSchema - Schema 检查

**实现**：[install.go:304-313](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/install.go#L304-L313)

```go
func checkSchema(db *sqlx.DB) (bool, error) {
    if _, err := db.Exec(`SELECT id FROM templates LIMIT 1`); err != nil {
        if isTableNotExistErr(err) {
            return false, nil
        }
        return false, err
    }
    return true, nil
}
```

**⚠️ 严重设计缺陷**：
- 仅检查 `templates` 表是否存在
- **完全不检查**：
  - 表结构是否完整（字段、索引、约束）
  - Settings 数据是否完整
  - Migration 记录是否完整
- 这就是为什么"启动时 checkSchema 通过但发送出现异常"

### 2.7 v6.2.0 migration - msg_retry_delay 字段添加

**实现**：[v6.2.0.go:11-31](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/internal/migrations/v6.2.0.go#L11-L31)

```go
func V6_2_0(db *sqlx.DB, fs stuffbin.FileSystem, ko *koanf.Koanf, lo *log.Logger) error {
    // 幂等设计：只为缺少msg_retry_delay的SMTP条目添加默认值
    _, err := db.Exec(`
        UPDATE settings SET value = s.updated
        FROM (
            SELECT JSONB_AGG(
                CASE WHEN v ? 'msg_retry_delay' THEN v
                     ELSE JSONB_SET(v, '{msg_retry_delay}', '"10ms"'::JSONB)
                END
            ) AS updated FROM settings, JSONB_ARRAY_ELEMENTS(value) v WHERE key = 'smtp'
        ) s WHERE key = 'smtp'
        AND EXISTS (
            SELECT 1 FROM JSONB_ARRAY_ELEMENTS(value) v WHERE NOT (v ? 'msg_retry_delay')
        );
    `)
    return err
}
```

**特点**：
- ✅ 幂等设计：`EXISTS` 子句确保只在确实缺少字段时才更新
- ✅ 原子操作：单条 SQL 语句完成
- ❌ 但如果此 migration 被跳过，DB 中的 `smtp` 配置就会缺少 `msg_retry_delay`

### 2.8 Settings 读取路径

**完整路径**：

1. **启动时初始化**：[main.go:181-191](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/main.go#L181-L191)
   ```go
   qMap := readQueries(queryFilePath, fs)
   if q, ok := qMap["get-settings"]; ok {
       initSettings(q.Query, db, ko)  // 从DB加载settings到koanf
   }
   queries = prepareQueries(qMap, db, ko)
   ```

2. **SQL 查询**：[misc.sql:7-8](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/queries/misc.sql#L7-L8)
   ```sql
   SELECT JSON_OBJECT_AGG(key, value) AS settings FROM (SELECT * FROM settings ORDER BY key) t;
   ```
   将 settings 表中所有行聚合成一个 JSON 对象

3. **加载到配置**：[init.go:432-454](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/init.go#L432-L454)
   ```go
   func initSettings(query string, db *sqlx.DB, ko *koanf.Koanf) {
       var s types.JSONText
       db.Get(&s, query)  // 从DB读取
       var out map[string]any
       json.Unmarshal(s, &out)  // 解析JSON
       ko.Load(confmap.Provider(out, "."), nil)  // 加载到koanf
   }
   ```

4. **模型定义**：[models/settings.go:83-102](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/models/settings.go#L83-L102)
   ```go
   SMTP []struct {
       // ... 其他字段 ...
       MaxMsgRetries int    `json:"max_msg_retries"`
       MsgRetryDelay string `json:"msg_retry_delay"`  // 此字段需要v6.2.0迁移添加
       IdleTimeout   string `json:"idle_timeout"`
       // ...
   } `json:"smtp"`
   ```

5. **schema.sql 默认值**：[schema.sql:280-282](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/schema.sql#L280-L282)
   ```sql
   ('smtp',
       '[{"enabled":true, ..., "msg_retry_delay":"10ms", ...},
         {"enabled":false, ..., "msg_retry_delay":"10ms", ...}]'),
   ```

**⚠️ 问题链**：
1. 新安装：`schema.sql` 自带 `msg_retry_delay` ✅
2. 升级路径：v6.2.0 migration 添加 `msg_retry_delay` ✅
3. 如果 migration 被跳过：DB 中缺少字段 ❌
4. JSON Unmarshal 时，Go 会用零值（空字符串）填充缺失字段 ⚠️
5. 发送邮件时解析 duration 失败：`time.ParseDuration("")` → error ❌

---

## 3. 问题根因总结

| 检查点 | 当前实现 | 问题 |
|--------|----------|------|
| `checkSchema` | 仅检查 `templates` 表存在 | 无法检测字段缺失、数据不完整 |
| `getPendingMigrations` | 仅比较版本号 | 版本记录与实际数据可能不一致 |
| 升级执行 | 成功后才记录版本 | 如果中途失败，可能处于不一致状态 |
| Settings 读取 | 无完整性校验 | 缺失字段时 Go 用零值填充，可能导致运行时错误 |
| 幂等性 | 单个 migration 幂等 | 但整体流程无完整性校验 |

---

## 4. 解决方案

### 4.1 如何检测缺失的 Migration

**方案一：版本记录完整性检查**

```sql
-- 检查DB中记录的migration版本列表
SELECT value AS migrations FROM settings WHERE key = 'migrations';

-- 检查是否有跳过的版本（假设当前应包含v6.2.0）
SELECT 
    value->>-1 AS last_version,
    jsonb_array_length(value) AS migration_count
FROM settings WHERE key = 'migrations';
```

**方案二：Settings 字段完整性检查**

```sql
-- 检查smtp配置中是否所有条目都有msg_retry_delay字段
SELECT 
    key,
    jsonb_array_length(value) AS total_servers,
    COUNT(*) FILTER (WHERE v ? 'msg_retry_delay') AS servers_with_field,
    COUNT(*) FILTER (WHERE NOT (v ? 'msg_retry_delay')) AS servers_missing_field
FROM settings, jsonb_array_elements(value) v
WHERE key = 'smtp'
GROUP BY key, value;

-- 检查具体缺失字段的条目
SELECT v 
FROM settings, jsonb_array_elements(value) v 
WHERE key = 'smtp' AND NOT (v ? 'msg_retry_delay');
```

**方案三：Schema 表结构完整性检查**

```sql
-- 检查settings表结构
SELECT column_name, data_type, is_nullable, column_default
FROM information_schema.columns 
WHERE table_name = 'settings'
ORDER BY ordinal_position;

-- 检查所有预期的表是否存在
SELECT table_name 
FROM information_schema.tables 
WHERE table_schema = 'public'
AND table_name IN ('settings', 'subscribers', 'lists', 'campaigns', 'templates', 'migrations');
```

**自动化检测脚本**：
```bash
#!/bin/bash
# 检测缺失migration的快速脚本

DB_HOST="localhost"
DB_PORT="5432"
DB_USER="listmonk"
DB_NAME="listmonk"

# 1. 检查最后migration版本
LAST_VER=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t -c \
    "SELECT COALESCE(value->>-1, 'v0.0.0') FROM settings WHERE key='migrations'")

echo "最后记录的migration版本: $LAST_VER"

# 2. 检查smtp.msg_retry_delay字段完整性
MISSING=$(psql -h $DB_HOST -p $DB_PORT -U $DB_USER -d $DB_NAME -t -c \
    "SELECT COUNT(*) FROM settings, jsonb_array_elements(value) v 
     WHERE key = 'smtp' AND NOT (v ? 'msg_retry_delay')")

if [ "$MISSING" -gt 0 ]; then
    echo "⚠️  发现 $MISSING 个 SMTP 服务器配置缺少 msg_retry_delay 字段"
    echo "需要执行 v6.2.0 migration"
fi

# 3. 版本比较（需要semver工具）
REQUIRED_VER="v6.2.0"
if [ "$(printf '%s\n%s\n' "$LAST_VER" "$REQUIRED_VER" | sort -V | head -n1)" != "$REQUIRED_VER" ]; then
    echo "⚠️  版本 $LAST_VER 低于要求的 $REQUIRED_VER"
fi
```

### 4.2 如何安全补跑缺失的 Migration

**⚠️ 重要前置操作：备份数据库**
```bash
# 备份数据库
pg_dump -h localhost -U listmonk -d listmonk -Fc -f listmonk_backup_$(date +%Y%m%d_%H%M%S).dump

# 验证备份
pg_restore -l listmonk_backup_*.dump | head -20
```

**方法一：重新执行升级命令（推荐）**

如果只是版本记录正确但实际 migration 未执行：
```bash
# 直接运行升级（幂等的migration会安全执行）
./listmonk --upgrade --yes
```

**方法二：手动补跑特定 migration**

如果版本记录已被污染（显示已到 v6.2.0 但实际未执行）：

```sql
-- 步骤1: 查看当前migrations记录
SELECT value FROM settings WHERE key = 'migrations';

-- 步骤2: 回滚版本记录到v6.1.0（假设v6.2.0未实际执行）
-- 注意：只删除最后一个版本，保留历史记录
UPDATE settings 
SET value = value - (jsonb_array_length(value) - 1)
WHERE key = 'migrations'
AND value->>-1 = 'v6.2.0';

-- 验证
SELECT value FROM settings WHERE key = 'migrations';
```

然后重新执行升级：
```bash
./listmonk --upgrade --yes
```

**方法三：直接执行 v6.2.0 的 SQL（紧急情况）**

如果无法重新运行完整升级流程，可以直接执行 migration 对应的 SQL：

```sql
-- 直接执行v6.2.0的核心SQL（幂等）
UPDATE settings SET value = s.updated
FROM (
    SELECT JSONB_AGG(
        CASE WHEN v ? 'msg_retry_delay' THEN v
             ELSE JSONB_SET(v, '{msg_retry_delay}', '"10ms"'::JSONB)
        END
    ) AS updated FROM settings, JSONB_ARRAY_ELEMENTS(value) v WHERE key = 'smtp'
) s WHERE key = 'smtp'
AND EXISTS (
    SELECT 1 FROM JSONB_ARRAY_ELEMENTS(value) v WHERE NOT (v ? 'msg_retry_delay')
);

-- 验证结果
SELECT jsonb_pretty(value) FROM settings WHERE key = 'smtp';
```

**补跑后验证**：
```sql
-- 1. 检查字段是否已添加
SELECT 
    COUNT(*) FILTER (WHERE NOT (v ? 'msg_retry_delay')) AS missing_count
FROM settings, jsonb_array_elements(value) v 
WHERE key = 'smtp';

-- 2. 检查默认值是否正确
SELECT v->'msg_retry_delay' AS msg_retry_delay
FROM settings, jsonb_array_elements(value) v 
WHERE key = 'smtp';
```

### 4.3 如何验证 SQL 模板和 Settings JSON 结构兼容

**方案一：Settings JSON Schema 校验**

为 settings 定义 JSON Schema，在启动时进行校验：

```go
// 示例：在initSettings中增加结构校验
func initSettings(query string, db *sqlx.DB, ko *koanf.Koanf) {
    var s types.JSONText
    if err := db.Get(&s, query); err != nil {
        lo.Fatalf("error reading settings from DB: %s", err)
    }

    // ✅ 新增：反序列化到结构体进行完整性校验
    var settings models.Settings
    if err := json.Unmarshal(s, &settings); err != nil {
        lo.Fatalf("settings JSON结构校验失败: %v", err)
    }

    // ✅ 新增：关键字段非空校验
    for i, smtp := range settings.SMTP {
        if smtp.MsgRetryDelay == "" {
            lo.Fatalf("SMTP服务器[%d]缺少msg_retry_delay字段，请运行 --upgrade", i)
        }
        if _, err := time.ParseDuration(smtp.MsgRetryDelay); err != nil {
            lo.Fatalf("SMTP服务器[%d]的msg_retry_delay值无效: %v", i, err)
        }
    }

    // 原有逻辑...
}
```

**方案二：SQL Schema 完整性校验**

在 `checkSchema` 中增加更全面的检查：

```go
// 增强版checkSchema
func checkSchema(db *sqlx.DB) (bool, error) {
    // 原检查：表存在性
    tables := []string{"templates", "settings", "subscribers", "campaigns", "lists"}
    for _, table := range tables {
        if _, err := db.Exec(fmt.Sprintf("SELECT 1 FROM %s LIMIT 1", table)); err != nil {
            if isTableNotExistErr(err) {
                lo.Printf("缺失表: %s", table)
                return false, nil
            }
            return false, err
        }
    }

    // ✅ 新增：settings关键字段检查
    var count int
    err := db.Get(&count, `
        SELECT COUNT(*) FROM settings 
        WHERE key = 'smtp' 
        AND EXISTS (
            SELECT 1 FROM jsonb_array_elements(value) v 
            WHERE NOT (v ? 'msg_retry_delay')
        )
    `)
    if err != nil {
        return false, err
    }
    if count > 0 {
        lo.Printf("⚠️  发现 %d 个SMTP配置缺少msg_retry_delay字段", count)
        // 可以选择返回false要求升级，或记录警告
    }

    // ✅ 新增：migration版本检查
    var lastVer string
    err = db.Get(&lastVer, `
        SELECT COALESCE(value->>-1, 'v0.0.0') FROM settings WHERE key='migrations'
    `)
    if err == nil {
        expectedVer := migList[len(migList)-1].version
        if semver.Compare(lastVer, expectedVer) < 0 {
            lo.Printf("⚠️  数据库版本 %s 低于预期版本 %s", lastVer, expectedVer)
        }
    }

    return true, nil
}
```

**方案三：SQL 模板与 DB Schema 一致性校验**

```go
// 校验关键查询是否能正常准备
func validateQueries(qMap map[string]goyesqlx.Stmt, db *sqlx.DB) error {
    // 尝试准备所有查询，失败则说明schema不兼容
    for name, stmt := range qMap {
        if _, err := db.Preparex(stmt.Query); err != nil {
            return fmt.Errorf("查询 %s 准备失败，可能schema不兼容: %v", name, err)
        }
    }
    return nil
}
```

### 4.4 如何处理 Nightly 版本

**当前实现分析**：[main.go:153-179](file:///Users/zhangjing/Desktop/so-coders/0508-und-p/listmonk/cmd/main.go#L153-L179)

```go
// 检查是否为nightly版本
isNightly := strings.Contains(versionString, "nightly")

// ...

if ko.Bool("upgrade") {
    // nightly版本不记录migration版本
    upgrade(db, fs, !ko.Bool("yes"), !isNightly)
    os.Exit(0)
}

// nightly版本自动运行所有migration，但不记录版本
if isNightly {
    lo.Printf("auto-running all migrations for nightly %s since last major version", versionString)
    upgrade(db, fs, false, false)
} else {
    checkUpgrade(db)
}
```

**Nightly 版本的特殊处理逻辑**：

| 场景 | 正式版本 | Nightly 版本 |
|------|----------|-------------|
| 手动 `--upgrade` | 记录版本 | 不记录版本 |
| 启动自动检查 | `checkUpgrade` 检查待执行迁移，缺失则启动失败 | 自动运行所有 migration，不记录版本 |
| 版本记录 | 严格追加，用于增量升级 | 不更新，每次启动从上次正式版本开始运行 |

**Nightly 版本的设计考虑**：
- ✅ migration 是幂等的，可以安全重复执行
- ✅ nightly 版本之间 migration 可能多次调整，不记录版本可以确保每次都应用最新代码
- ❌ 但如果 migration 本身有破坏性变更，重复执行可能有问题

**Nightly 版本使用最佳实践**：

1. **不要在生产环境使用 nightly 版本**
   ```bash
   # 检查版本
   ./listmonk --version
   # 如果包含"nightly"字样，谨慎使用
   ```

2. **nightly 升级到正式版本的步骤**：
   ```bash
   # 1. 备份数据库
   pg_dump ... > backup.dump
   
   # 2. 停止nightly版本
   systemctl stop listmonk
   
   # 3. 替换为正式版本二进制
   cp listmonk_stable listmonk
   
   # 4. 显式执行升级（会记录版本）
   ./listmonk --upgrade --yes
   
   # 5. 启动服务
   systemctl start listmonk
   ```

3. **nightly 版本间迁移问题排查**：
   ```sql
   -- 查看最后记录的正式版本（nightly不会更新）
   SELECT value->>-1 AS last_stable_version FROM settings WHERE key = 'migrations';
   
   -- 手动检查数据完整性
   SELECT key, jsonb_typeof(value) FROM settings ORDER BY key;
   ```

---

## 5. 长期改进建议

### 5.1 增强 checkSchema

```go
// 建议的增强版checkSchema
func checkSchema(db *sqlx.DB) error {
    checks := []struct {
        name string
        fn   func(*sqlx.DB) error
    }{
        {"表存在性检查", checkTablesExist},
        {"Migration版本检查", checkMigrationVersion},
        {"Settings完整性检查", checkSettingsIntegrity},
        {"关键字段存在性检查", checkRequiredFields},
    }

    for _, check := range checks {
        if err := check.fn(db); err != nil {
            return fmt.Errorf("%s 失败: %w", check.name, err)
        }
    }
    return nil
}
```

### 5.2 增加 Migration 执行校验

每个 migration 执行后增加验证步骤：
```go
// 在upgrade循环中增加验证
for _, m := range toRun {
    lo.Printf("running migration %s", m.version)
    if err := m.fn(db, fs, ko, lo); err != nil {
        lo.Fatalf("error running migration %s: %v", m.version, err)
    }
    
    // ✅ 新增：执行后验证
    if validator, ok := migrationValidators[m.version]; ok {
        if err := validator(db); err != nil {
            lo.Fatalf("migration %s 验证失败: %v", m.version, err)
        }
    }
    
    recordMigrationVersion(m.version, db)
}
```

### 5.3 增加 Settings 结构版本号

在 settings 中增加结构版本字段：
```sql
INSERT INTO settings (key, value) VALUES 
    ('settings_schema_version', '"1.0"'),
    ...
```

启动时检查结构版本，不兼容则拒绝启动。

---

## 6. 快速修复 Checklist

针对本次 `msg_retry_delay` 缺失问题：

- [ ] **步骤 1**：备份数据库
  ```bash
  pg_dump -h localhost -U listmonk -d listmonk -Fc -f backup.dump
  ```

- [ ] **步骤 2**：检测缺失情况
  ```sql
  SELECT COUNT(*) FROM settings, jsonb_array_elements(value) v 
  WHERE key = 'smtp' AND NOT (v ? 'msg_retry_delay');
  ```

- [ ] **步骤 3**：检查版本记录
  ```sql
  SELECT value->>-1 AS last_version FROM settings WHERE key = 'migrations';
  ```

- [ ] **步骤 4**：补跑 migration
  - 方法A（推荐）：`./listmonk --upgrade --yes`
  - 方法B：先回滚版本记录再升级
  - 方法C：直接执行修复 SQL

- [ ] **步骤 5**：验证修复
  ```sql
  SELECT v->'host' AS host, v->'msg_retry_delay' AS delay
  FROM settings, jsonb_array_elements(value) v WHERE key = 'smtp';
  ```

- [ ] **步骤 6**：重启服务并测试邮件发送

---

## 7. 总结

| 问题 | 根因 | 解决方案 |
|------|------|----------|
| checkSchema 通过但数据异常 | checkSchema 仅检查表存在 | 增强 checkSchema，增加数据完整性校验 |
| migration 被跳过 | 仅按版本号比较，不验证实际执行 | 每个 migration 执行后增加验证步骤 |
| settings 缺字段 | JSON Unmarshal 用零值填充不报错 | 启动时反序列化到结构体并校验关键字段 |
| nightly 版本处理 | 不记录版本，每次重跑 | 仅用于开发测试，生产环境使用正式版本 |

**核心原则**：
1. Migration 必须幂等
2. 升级前必须备份
3. 完整性检查比存在性检查更重要
4. 早发现早修复（启动时校验比运行时报错好）
