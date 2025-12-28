# DuckDB Catalog 系统深度解析 - 系列大纲

本系列深入分析 DuckDB 的 Catalog（目录）系统实现，涵盖元数据管理、对象生命周期、依赖追踪和名称解析机制。

## 系列总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DuckDB Catalog 系统深度解析                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第一章：Catalog 架构概述                                           │     │
│  │  • Catalog 抽象层次（Catalog → DuckCatalog）                        │     │
│  │  • AttachedDatabase 与多数据库支持                                  │     │
│  │  • CatalogEntry 继承体系                                            │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第二章：CatalogEntry 类型体系                                       │     │
│  │  • Schema、Table、View、Index 等条目类型                            │     │
│  │  • StandardEntry 与 InCatalogEntry                                 │     │
│  │  • 条目的创建、修改与删除                                            │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第三章：CatalogSet 与 MVCC                                          │     │
│  │  • CatalogSet 条目集合管理                                          │     │
│  │  • 版本链与事务可见性                                                │     │
│  │  • CatalogTransaction 事务上下文                                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第四章：依赖管理系统                                                 │     │
│  │  • DependencyManager 依赖追踪                                       │     │
│  │  • Subject/Dependent 关系模型                                       │     │
│  │  • 级联删除与依赖验证                                                │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第五章：名称解析与搜索路径                                           │     │
│  │  • CatalogSearchPath 搜索路径                                       │     │
│  │  • 三级名称解析（catalog.schema.name）                               │     │
│  │  • EntryLookupInfo 与查找策略                                       │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第六章：默认条目与内置对象                                           │     │
│  │  • DefaultGenerator 默认生成器                                      │     │
│  │  • 内置函数、类型与 Schema                                          │     │
│  │  • 扩展注册机制                                                     │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第一章：Catalog 架构概述

### 1.1 Catalog 抽象层次

```
┌─────────────────────────────────────────────────────────────────┐
│                      Catalog (抽象基类)                          │
│  • GetSchema(): 获取 Schema                                     │
│  • CreateTable/View/Function(): 创建对象                         │
│  • GetEntry(): 查找条目                                          │
│  • DropEntry(): 删除条目                                         │
├─────────────────────────────────────────────────────────────────┤
│                              ↓                                   │
│         ┌───────────────────┴───────────────────┐               │
│         ↓                                       ↓               │
│  ┌─────────────────┐                    ┌─────────────────┐     │
│  │   DuckCatalog   │                    │  外部 Catalog   │     │
│  │ (DuckDB 原生)    │                    │ (扩展提供)       │     │
│  │                 │                    │                 │     │
│  │ • schemas       │                    │ • PostgreSQL    │     │
│  │ • dependency_   │                    │ • MySQL         │     │
│  │   manager       │                    │ • SQLite        │     │
│  │ • write_lock    │                    │ • ...           │     │
│  └─────────────────┘                    └─────────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

**核心内容**：
- `Catalog` 基类：`src/include/duckdb/catalog/catalog.hpp`
- `DuckCatalog` 实现：`src/catalog/duck_catalog.cpp`
- Catalog 类型系统与扩展支持
- 系统 Catalog vs 用户 Catalog

### 1.2 AttachedDatabase 与多数据库

```
DatabaseInstance
       │
       ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DatabaseManager                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │ system      │  │ memory      │  │ attached_db │              │
│  │ (系统库)     │  │ (临时库)     │  │ (附加库)     │              │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                │                │                      │
│         ▼                ▼                ▼                      │
│    ┌─────────┐     ┌─────────┐     ┌─────────┐                  │
│    │ Catalog │     │ Catalog │     │ Catalog │                  │
│    └─────────┘     └─────────┘     └─────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

**核心内容**：
- `AttachedDatabase` 与 Catalog 的关系
- ATTACH/DETACH 机制
- 跨数据库查询支持

### 1.3 CatalogEntry 继承体系

```cpp
// 类继承关系
CatalogEntry                       // 抽象基类
    │
    ├── InCatalogEntry             // 带 Catalog 引用
    │       │
    │       ├── SchemaCatalogEntry // Schema 条目
    │       │       │
    │       │       └── DuckSchemaEntry
    │       │
    │       └── StandardEntry      // 属于 Schema 的条目
    │               │
    │               ├── TableCatalogEntry
    │               │       └── DuckTableEntry
    │               ├── ViewCatalogEntry
    │               ├── IndexCatalogEntry
    │               │       └── DuckIndexEntry
    │               ├── SequenceCatalogEntry
    │               ├── TypeCatalogEntry
    │               ├── FunctionEntry
    │               │       ├── ScalarFunctionCatalogEntry
    │               │       ├── AggregateFunctionCatalogEntry
    │               │       ├── TableFunctionCatalogEntry
    │               │       └── MacroCatalogEntry
    │               └── CollateCatalogEntry
    │
    └── DependencyEntry            // 依赖关系条目
            ├── DependencySubjectEntry
            └── DependencyDependentEntry
```

**核心内容**：
- `CatalogEntry` 基类属性（oid、type、name、deleted、timestamp）
- `InCatalogEntry` 与 Catalog 关联
- `StandardEntry` 与 Schema 关联
- 各种具体条目类型的职责

---

## 第二章：CatalogEntry 类型体系

### 2.1 CatalogType 枚举

```cpp
enum class CatalogType : uint8_t {
    INVALID = 0,
    TABLE_ENTRY = 1,           // 表
    SCHEMA_ENTRY = 2,          // Schema
    VIEW_ENTRY = 3,            // 视图
    INDEX_ENTRY = 4,           // 索引
    PREPARED_STATEMENT = 5,    // 预编译语句
    SEQUENCE_ENTRY = 6,        // 序列
    COLLATION_ENTRY = 7,       // 排序规则
    TYPE_ENTRY = 8,            // 类型
    DATABASE_ENTRY = 9,        // 数据库

    // 函数类型
    TABLE_FUNCTION_ENTRY = 25,
    SCALAR_FUNCTION_ENTRY = 26,
    AGGREGATE_FUNCTION_ENTRY = 27,
    PRAGMA_FUNCTION_ENTRY = 28,
    COPY_FUNCTION_ENTRY = 29,
    MACRO_ENTRY = 30,
    TABLE_MACRO_ENTRY = 31,

    // 依赖类型
    DEPENDENCY_ENTRY = 40,

    // 特殊
    RENAMED_ENTRY = 100,
    DELETED_ENTRY = 101
};
```

### 2.2 Schema 条目

```cpp
class SchemaCatalogEntry : public InCatalogEntry {
public:
    // 创建各种对象
    virtual optional_ptr<CatalogEntry> CreateTable(...);
    virtual optional_ptr<CatalogEntry> CreateView(...);
    virtual optional_ptr<CatalogEntry> CreateFunction(...);
    virtual optional_ptr<CatalogEntry> CreateIndex(...);
    virtual optional_ptr<CatalogEntry> CreateSequence(...);
    virtual optional_ptr<CatalogEntry> CreateType(...);

    // 查找与遍历
    virtual optional_ptr<CatalogEntry> LookupEntry(...);
    virtual void Scan(CatalogType type, callback);

    // 删除与修改
    virtual void DropEntry(...);
    virtual void Alter(...);
};
```

**核心内容**：
- `SchemaCatalogEntry`：`src/catalog/catalog_entry/schema_catalog_entry.cpp`
- `DuckSchemaEntry` 实现：`src/catalog/catalog_entry/duck_schema_entry.cpp`
- Schema 内的 CatalogSet 管理

### 2.3 Table 条目

```cpp
class TableCatalogEntry : public StandardEntry {
protected:
    ColumnList columns;                          // 列定义
    vector<unique_ptr<Constraint>> constraints;  // 约束

public:
    // 列操作
    bool ColumnExists(const string &name) const;
    const ColumnDefinition &GetColumn(const string &name) const;
    const ColumnList &GetColumns() const;

    // 存储与统计
    virtual DataTable &GetStorage();
    virtual unique_ptr<BaseStatistics> GetStatistics(...);

    // 扫描函数
    virtual TableFunction GetScanFunction(...);
};
```

**核心内容**：
- `TableCatalogEntry`：`src/catalog/catalog_entry/table_catalog_entry.cpp`
- `DuckTableEntry` 实现：`src/catalog/catalog_entry/duck_table_entry.cpp`
- 与 DataTable 存储的关联
- ALTER TABLE 操作处理

### 2.4 Function 条目

```
┌─────────────────────────────────────────────────────────────────┐
│                      FunctionEntry                               │
│  • functions: FunctionSet (函数重载集合)                         │
├─────────────────────────────────────────────────────────────────┤
│                              ↓                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ ScalarFunc   │  │ AggregateFunc│  │ TableFunc    │           │
│  │ (标量函数)    │  │ (聚合函数)    │  │ (表函数)     │           │
│  │              │  │              │  │              │           │
│  │ • 无状态     │  │ • 有状态     │  │ • 返回表     │           │
│  │ • 行处理     │  │ • 分组处理   │  │ • 生成行     │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐                             │
│  │ MacroEntry   │  │ PragmaFunc   │                             │
│  │ (宏函数)      │  │ (Pragma 函数) │                             │
│  └──────────────┘  └──────────────┘                             │
└─────────────────────────────────────────────────────────────────┘
```

**核心内容**：
- 函数条目类型层次
- 函数重载与签名匹配
- Macro 展开机制

### 2.5 其他条目类型

| 类型 | 文件 | 说明 |
|------|------|------|
| ViewCatalogEntry | `view_catalog_entry.cpp` | 视图定义与 SQL 存储 |
| IndexCatalogEntry | `index_catalog_entry.cpp` | 索引元数据 |
| SequenceCatalogEntry | `sequence_catalog_entry.cpp` | 序列生成器 |
| TypeCatalogEntry | `type_catalog_entry.cpp` | 用户定义类型/ENUM |
| CollateCatalogEntry | `collate_catalog_entry.cpp` | 排序规则 |

---

## 第三章：CatalogSet 与 MVCC

### 3.1 CatalogSet 数据结构

```cpp
class CatalogSet {
private:
    DuckCatalog &catalog;
    mutex catalog_lock;                        // 目录锁
    CatalogEntryMap map;                       // 条目映射
    unique_ptr<DefaultGenerator> defaults;     // 默认生成器

public:
    // 创建条目
    bool CreateEntry(CatalogTransaction transaction,
                     const string &name,
                     unique_ptr<CatalogEntry> value,
                     const LogicalDependencyList &dependencies);

    // 查找条目
    optional_ptr<CatalogEntry> GetEntry(CatalogTransaction transaction,
                                        const string &name);

    // 删除条目
    bool DropEntry(CatalogTransaction transaction,
                   const string &name,
                   bool cascade);

    // 修改条目
    bool AlterEntry(CatalogTransaction transaction,
                    const string &name,
                    AlterInfo &alter_info);
};
```

### 3.2 MVCC 版本链

```
版本链结构:

CatalogEntryMap["table_name"]
         │
         ▼
    ┌─────────────────┐
    │  CatalogEntry   │ ← 最新版本 (timestamp=T3)
    │  (current)      │
    │  child ─────────┼──┐
    └─────────────────┘  │
                         ▼
                   ┌─────────────────┐
                   │  CatalogEntry   │ ← 前一版本 (timestamp=T2)
                   │  child ─────────┼──┐
                   └─────────────────┘  │
                                        ▼
                                  ┌─────────────────┐
                                  │  CatalogEntry   │ ← 初始版本 (timestamp=T1)
                                  │  child = null   │
                                  └─────────────────┘

可见性判断:
• Transaction T 可见条目 E 当:
  - E.timestamp < T.start_time (已提交)
  - 或 E.transaction_id == T.id (同一事务)
```

**核心内容**：
- 版本链的创建与遍历
- `GetEntryForTransaction()` 可见性判断
- 事务提交与回滚对版本链的影响

### 3.3 CatalogTransaction

```cpp
struct CatalogTransaction {
    optional_ptr<DatabaseInstance> db;
    optional_ptr<ClientContext> context;
    optional_ptr<Transaction> transaction;
    transaction_t transaction_id;
    transaction_t start_time;

    // 创建方法
    static CatalogTransaction GetSystemCatalogTransaction(ClientContext &context);
    static CatalogTransaction GetSystemTransaction(DatabaseInstance &db);
};
```

**核心内容**：
- `CatalogTransaction`：`src/catalog/catalog_transaction.cpp`
- 系统事务 vs 用户事务
- 与 Transaction 的协作

### 3.4 条目创建流程

```
CreateEntry 流程:

1. 获取 catalog_lock
         │
         ▼
2. 检查名称是否存在
         │
    ┌────┴────┐
    │ 存在?   │
    ├─YES─────┼──► 检查是否被删除
    │         │         │
    │         │    ┌────┴────┐
    │         │    │ 已删除? │
    │         │    ├─YES─────┼──► 创建新版本
    │         │    └─NO──────┼──► 返回失败
    │         │              │
    └─NO──────┼──────────────┘
              ▼
3. 调用 DependencyManager.AddObject()
         │
         ▼
4. 设置 entry.timestamp = transaction_id
         │
         ▼
5. 添加到 map
```

---

## 第四章：依赖管理系统

### 4.1 DependencyManager 架构

```cpp
class DependencyManager {
private:
    DuckCatalog &catalog;
    CatalogSet subjects;      // 被依赖对象集合
    CatalogSet dependents;    // 依赖方对象集合

public:
    // 添加依赖
    void AddObject(CatalogTransaction transaction,
                   CatalogEntry &object,
                   const LogicalDependencyList &dependencies);

    // 删除时检查
    catalog_entry_set_t CheckDropDependencies(
        CatalogTransaction transaction,
        CatalogEntry &object,
        bool cascade);

    // 扫描依赖
    void Scan(ClientContext &context, callback);

    // 导出排序
    void ReorderEntries(catalog_entry_vector_t &entries);
};
```

### 4.2 Subject/Dependent 模型

```
依赖关系示例:

View V 依赖于 Table T:

┌───────────────────────────────────────────────────────────────┐
│                    DependencyManager                           │
├───────────────────────────────────────────────────────────────┤
│                                                                │
│  subjects CatalogSet:                                          │
│  ┌─────────────────────────────────────────┐                  │
│  │ Key: MangleName(T)                       │                  │
│  │ Value: DependencySubjectEntry            │                  │
│  │   → dependent_list: [V]                  │                  │
│  │   → flags: DEPENDENCY_REGULAR            │                  │
│  └─────────────────────────────────────────┘                  │
│                                                                │
│  dependents CatalogSet:                                        │
│  ┌─────────────────────────────────────────┐                  │
│  │ Key: MangleName(V)                       │                  │
│  │ Value: DependencyDependentEntry          │                  │
│  │   → subject_list: [T]                    │                  │
│  │   → flags: DEPENDENCY_REGULAR            │                  │
│  └─────────────────────────────────────────┘                  │
│                                                                │
└───────────────────────────────────────────────────────────────┘
```

### 4.3 依赖类型

```cpp
// 依赖方标志
enum class DependencyDependentFlags : uint8_t {
    DEPENDENCY_REGULAR = 0,        // 普通依赖（阻止删除）
    DEPENDENCY_AUTOMATIC = 1,      // 自动依赖（级联删除）
    DEPENDENCY_OWNS = 2            // 所有权依赖
};

// 被依赖方标志
enum class DependencySubjectFlags : uint8_t {
    DEPENDENCY_REGULAR = 0,
    DEPENDENCY_OWNED_BY = 1        // 被拥有
};
```

### 4.4 级联删除

```
DROP TABLE T CASCADE:

1. 查找 T 的所有依赖方
         │
         ▼
2. 对每个依赖方 D:
   ├── 如果 D 是 REGULAR → 收集到删除列表
   ├── 如果 D 是 AUTOMATIC → 自动级联删除
   └── 如果 D 是 OWNS → 报错（不能删除被拥有的对象）
         │
         ▼
3. 递归处理每个 D 的依赖方
         │
         ▼
4. 按拓扑顺序删除所有收集的对象
```

**核心内容**：
- `DependencyManager`：`src/catalog/dependency_manager.cpp`
- `DependencyEntry` 类型
- EXPORT 排序与依赖图遍历

---

## 第五章：名称解析与搜索路径

### 5.1 三级名称解析

```
完整名称: catalog.schema.name

解析优先级:
1. 如果指定了 catalog → 直接查找
2. 如果只指定 schema.name → 按搜索路径查找
3. 如果只指定 name → 遍历搜索路径的所有 schema

┌─────────────────────────────────────────────────────────────────┐
│                      名称解析流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  输入: "my_table"                                                │
│         │                                                        │
│         ▼                                                        │
│  CatalogSearchPath: [memory.main, memory.pg_catalog, system.*]  │
│         │                                                        │
│         ▼                                                        │
│  尝试 1: memory.main.my_table                                    │
│         │ 找到? ──YES──► 返回                                    │
│         │                                                        │
│         ▼ NO                                                     │
│  尝试 2: memory.pg_catalog.my_table                              │
│         │ 找到? ──YES──► 返回                                    │
│         │                                                        │
│         ▼ NO                                                     │
│  尝试 3: system.*.my_table (所有 schema)                         │
│         │ 找到? ──YES──► 返回                                    │
│         │                                                        │
│         ▼ NO                                                     │
│  抛出 CatalogException                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 CatalogSearchPath

```cpp
class CatalogSearchPath {
private:
    ClientContext &context;
    vector<CatalogSearchEntry> paths;      // 完整路径
    vector<CatalogSearchEntry> set_paths;  // 用户设置的路径

public:
    // 设置搜索路径
    void Set(CatalogSearchEntry new_value, CatalogSetPathType set_type);
    void Set(vector<CatalogSearchEntry> new_paths, CatalogSetPathType set_type);

    // 获取默认
    const CatalogSearchEntry &GetDefault() const;
    string GetDefaultSchema(const string &catalog) const;
    string GetDefaultCatalog(const string &schema) const;

    // 查询
    bool SchemaInSearchPath(ClientContext &context,
                            const string &catalog_name,
                            const string &schema_name) const;
};

struct CatalogSearchEntry {
    string catalog;  // 数据库名
    string schema;   // Schema 名
};
```

### 5.3 EntryLookupInfo

```cpp
struct EntryLookupInfo {
    CatalogType type;              // 查找的条目类型
    string name;                   // 条目名称
    QueryErrorContext error_ctx;   // 错误上下文

    // 构造
    EntryLookupInfo(CatalogType type, const string &name,
                    QueryErrorContext error_context = {});
};
```

### 5.4 查找策略

```cpp
// CatalogLookupBehavior 控制查找行为
enum class CatalogLookupBehavior : uint8_t {
    STANDARD = 0,           // 标准查找
    LOWER_PRIORITY = 1,     // 降低优先级
    NEVER = 2               // 不查找此 catalog
};

// 查找流程
CatalogEntryLookup TryLookupEntry(
    CatalogEntryRetriever &retriever,
    const string &catalog,
    const string &schema,
    const EntryLookupInfo &lookup_info,
    OnEntryNotFound if_not_found);
```

**核心内容**：
- `CatalogSearchPath`：`src/catalog/catalog_search_path.cpp`
- `EntryLookupInfo`：`src/catalog/entry_lookup_info.cpp`
- SET search_path 语句处理

---

## 第六章：默认条目与内置对象

### 6.1 DefaultGenerator 机制

```cpp
class DefaultGenerator {
public:
    Catalog &catalog;
    atomic<bool> created_all_entries;

    // 创建默认条目
    virtual unique_ptr<CatalogEntry> CreateDefaultEntry(
        ClientContext &context, const string &entry_name);

    // 获取所有默认条目名称
    virtual vector<string> GetDefaultEntries() = 0;
};
```

### 6.2 默认生成器类型

```
┌─────────────────────────────────────────────────────────────────┐
│                      DefaultGenerator 层次                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────┐                                         │
│  │ DefaultSchemaGenerator │ ← 默认 Schema (main, pg_catalog)    │
│  └────────────────────┘                                         │
│                                                                  │
│  ┌────────────────────┐                                         │
│  │ DefaultFunctionGenerator │ ← 内置函数 (sin, cos, concat...)  │
│  └────────────────────┘                                         │
│                                                                  │
│  ┌────────────────────┐                                         │
│  │ DefaultTableFunctionGenerator │ ← 表函数 (range, generate_*) │
│  └────────────────────┘                                         │
│                                                                  │
│  ┌────────────────────┐                                         │
│  │ DefaultTypeGenerator │ ← 内置类型 (int, varchar, json...)    │
│  └────────────────────┘                                         │
│                                                                  │
│  ┌────────────────────┐                                         │
│  │ DefaultViewGenerator │ ← 系统视图 (duckdb_tables, ...)       │
│  └────────────────────┘                                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 懒加载机制

```
懒加载流程:

GetEntry("sin")
      │
      ▼
1. 检查 CatalogSet.map
      │
  ┌───┴───┐
  │ 存在? │
  ├─YES───┼──► 返回条目
  │       │
  └─NO────┼──┐
          │  ▼
2. 调用 DefaultGenerator.CreateDefaultEntry("sin")
          │
     ┌────┴────┐
     │ 成功?   │
     ├─YES─────┼──► 添加到 map, 返回条目
     │         │
     └─NO──────┼──► 返回 nullptr
```

### 6.4 内置 Schema

```cpp
// 默认 Schema
static constexpr const char *DEFAULT_SCHEMA = "main";
static constexpr const char *PG_CATALOG = "pg_catalog";
static constexpr const char *INFORMATION_SCHEMA = "information_schema";
static constexpr const char *TEMP_SCHEMA = "temp";

// pg_catalog 包含:
// - 系统表 (pg_class, pg_attribute, pg_type, ...)
// - 系统函数
// - PostgreSQL 兼容视图
```

### 6.5 扩展注册

```cpp
// 扩展可以注册自己的:
// 1. 函数
ExtensionHelper::RegisterFunction(db, ScalarFunction(...));

// 2. 类型
ExtensionHelper::RegisterType(db, CreateTypeInfo(...));

// 3. 表函数
ExtensionHelper::RegisterTableFunction(db, TableFunction(...));

// 4. 新的 Catalog (如 PostgreSQL 扫描器)
ExtensionHelper::RegisterCatalog(db, unique_ptr<Catalog>(...));
```

**核心内容**：
- 各种 DefaultGenerator 实现
- 系统视图与 information_schema
- 扩展如何向 Catalog 添加对象

---

## 核心源文件索引

### 基础架构

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/catalog.hpp` | Catalog 抽象基类 |
| `src/catalog/catalog.cpp` | Catalog 基类实现 |
| `src/include/duckdb/catalog/duck_catalog.hpp` | DuckCatalog 定义 |
| `src/catalog/duck_catalog.cpp` | DuckCatalog 实现 |
| `src/include/duckdb/catalog/catalog_entry.hpp` | CatalogEntry 基类 |
| `src/catalog/catalog_entry.cpp` | CatalogEntry 实现 |

### 条目类型

| 文件 | 说明 |
|------|------|
| `src/catalog/catalog_entry/schema_catalog_entry.cpp` | Schema 条目 |
| `src/catalog/catalog_entry/duck_schema_entry.cpp` | DuckDB Schema 实现 |
| `src/catalog/catalog_entry/table_catalog_entry.cpp` | 表条目基类 |
| `src/catalog/catalog_entry/duck_table_entry.cpp` | DuckDB 表实现 |
| `src/catalog/catalog_entry/view_catalog_entry.cpp` | 视图条目 |
| `src/catalog/catalog_entry/index_catalog_entry.cpp` | 索引条目 |
| `src/catalog/catalog_entry/sequence_catalog_entry.cpp` | 序列条目 |
| `src/catalog/catalog_entry/type_catalog_entry.cpp` | 类型条目 |
| `src/catalog/catalog_entry/scalar_function_catalog_entry.cpp` | 标量函数 |

### 集合与依赖

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/catalog_set.hpp` | CatalogSet 定义 |
| `src/catalog/catalog_set.cpp` | CatalogSet 实现 |
| `src/include/duckdb/catalog/dependency_manager.hpp` | 依赖管理器 |
| `src/catalog/dependency_manager.cpp` | 依赖管理实现 |
| `src/catalog/dependency_list.cpp` | 依赖列表 |

### 搜索与默认

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/catalog_search_path.hpp` | 搜索路径 |
| `src/catalog/catalog_search_path.cpp` | 搜索路径实现 |
| `src/catalog/catalog_entry_retriever.cpp` | 条目检索器 |
| `src/catalog/default/default_generator.cpp` | 默认生成器基类 |
| `src/catalog/default/default_functions.cpp` | 默认函数 |
| `src/catalog/default/default_types.cpp` | 默认类型 |
| `src/catalog/default/default_schemas.cpp` | 默认 Schema |
| `src/catalog/default/default_views.cpp` | 默认视图 |

---

## 写作计划

| 章节 | 预计篇幅 | 核心难点 |
|------|----------|----------|
| 第一章：Catalog 架构概述 | ~30KB | 抽象层次理解 |
| 第二章：CatalogEntry 类型体系 | ~35KB | 类型继承关系 |
| 第三章：CatalogSet 与 MVCC | ~40KB | 版本链与事务可见性 |
| 第四章：依赖管理系统 | ~30KB | 依赖图与级联删除 |
| 第五章：名称解析与搜索路径 | ~25KB | 三级解析逻辑 |
| 第六章：默认条目与内置对象 | ~30KB | 懒加载与扩展机制 |

**总计**: 约 190KB，6 章

---

## 与其他系列的关联

```
┌─────────────────────────────────────────────────────────────────┐
│                        系列关联图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  查询处理篇                Catalog 系统篇                        │
│  ┌─────────┐              ┌─────────────┐                       │
│  │ Binder  │──────────────│ Catalog     │                       │
│  │ 符号解析 │              │ 元数据管理   │                       │
│  └─────────┘              └─────────────┘                       │
│       ↑                          ↑                               │
│       │                          │                               │
│  ┌─────────┐              ┌─────────────┐                       │
│  │ Planner │──────────────│ CatalogEntry│                       │
│  │ 计划生成 │              │ 对象定义     │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
│  存储引擎篇                                                      │
│  ┌─────────┐              ┌─────────────┐                       │
│  │DataTable│──────────────│DuckTableEntry│                       │
│  │ 物理存储 │              │ 表元数据     │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
│  事务系统篇                                                      │
│  ┌─────────┐              ┌─────────────┐                       │
│  │Transaction│────────────│CatalogSet   │                       │
│  │ MVCC    │              │ 版本链       │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
│  索引系统篇                                                      │
│  ┌─────────┐              ┌─────────────┐                       │
│  │   ART   │──────────────│DuckIndexEntry│                       │
│  │ 索引实现 │              │ 索引元数据   │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 关键概念速查

| 概念 | 说明 |
|------|------|
| Catalog | 数据库元数据的顶层容器 |
| DuckCatalog | DuckDB 原生的 Catalog 实现 |
| AttachedDatabase | 附加的数据库实例 |
| CatalogEntry | Catalog 中的一个条目（表/视图/函数等） |
| CatalogSet | 条目的 MVCC 集合容器 |
| CatalogTransaction | Catalog 操作的事务上下文 |
| DependencyManager | 管理对象间依赖关系 |
| CatalogSearchPath | 名称解析的搜索路径 |
| DefaultGenerator | 内置对象的懒加载生成器 |
| StandardEntry | 属于 Schema 的条目基类 |
