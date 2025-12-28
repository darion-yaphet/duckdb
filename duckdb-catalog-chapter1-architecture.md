# DuckDB Catalog 系统深度解析 - 第一章：Catalog 架构概述

本章深入分析 DuckDB Catalog 系统的整体架构，包括 Catalog 抽象层次、AttachedDatabase 与多数据库支持，以及 CatalogEntry 的继承体系。

## 1.1 Catalog 概述

### 1.1.1 什么是 Catalog

Catalog（目录）是数据库系统中管理元数据的核心组件。在 DuckDB 中，Catalog 负责：

1. **对象注册**：表、视图、函数、索引、类型等数据库对象的创建与删除
2. **名称解析**：将 SQL 中的标识符解析为具体的数据库对象
3. **依赖追踪**：管理对象之间的依赖关系（如视图依赖于表）
4. **事务支持**：确保元数据修改的 ACID 特性
5. **Schema 管理**：组织和隔离不同命名空间下的对象

### 1.1.2 Catalog 在查询处理中的角色

```
┌─────────────────────────────────────────────────────────────────┐
│                      查询处理流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  SQL: SELECT * FROM customers WHERE id = 1                      │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐                                                │
│  │   Parser    │  ← 语法分析，生成 AST                           │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐     ┌───────────────┐                          │
│  │   Binder    │ ←──→│   Catalog     │  ← 名称解析              │
│  │  (符号解析)  │     │               │    - customers → 表对象   │
│  └──────┬──────┘     │  • 查找表定义  │    - id → 列定义          │
│         │            │  • 获取列信息  │    - 验证权限             │
│         ▼            │  • 解析函数    │                          │
│  ┌─────────────┐     └───────────────┘                          │
│  │   Planner   │                                                │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐                                                │
│  │  Optimizer  │                                                │
│  └──────┬──────┘                                                │
│         │                                                        │
│         ▼                                                        │
│  ┌─────────────┐                                                │
│  │  Executor   │                                                │
│  └─────────────┘                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.2 Catalog 抽象层次

### 1.2.1 Catalog 基类

DuckDB 的 Catalog 采用抽象工厂模式，定义了统一的接口：

```cpp
// src/include/duckdb/catalog/catalog.hpp
class Catalog {
public:
    explicit Catalog(AttachedDatabase &db);
    virtual ~Catalog();

    //! 获取关联的 AttachedDatabase
    AttachedDatabase &GetAttached();
    DatabaseInstance &GetDatabase();

    //! 获取 Catalog 名称（来自 AttachedDatabase）
    const string &GetName() const;

    //! 是否为 DuckDB 原生 Catalog
    virtual bool IsDuckCatalog() { return false; }

    //! 初始化 Catalog
    virtual void Initialize(bool load_builtin) = 0;

    //! 获取 Catalog 类型（如 "duckdb", "sqlite", "postgres"）
    virtual string GetCatalogType() = 0;

    //===--------------------------------------------------------------------===//
    // Schema 操作
    //===--------------------------------------------------------------------===//
    virtual optional_ptr<CatalogEntry> CreateSchema(
        CatalogTransaction transaction, CreateSchemaInfo &info) = 0;

    virtual optional_ptr<SchemaCatalogEntry> LookupSchema(
        CatalogTransaction transaction,
        const EntryLookupInfo &schema_lookup,
        OnEntryNotFound if_not_found) = 0;

    virtual void ScanSchemas(
        ClientContext &context,
        std::function<void(SchemaCatalogEntry &)> callback) = 0;

    //===--------------------------------------------------------------------===//
    // 对象创建（委托给 Schema）
    //===--------------------------------------------------------------------===//
    optional_ptr<CatalogEntry> CreateTable(CatalogTransaction, BoundCreateTableInfo &);
    optional_ptr<CatalogEntry> CreateView(CatalogTransaction, CreateViewInfo &);
    optional_ptr<CatalogEntry> CreateFunction(CatalogTransaction, CreateFunctionInfo &);
    optional_ptr<CatalogEntry> CreateIndex(CatalogTransaction, CreateIndexInfo &);
    optional_ptr<CatalogEntry> CreateSequence(CatalogTransaction, CreateSequenceInfo &);
    optional_ptr<CatalogEntry> CreateType(CatalogTransaction, CreateTypeInfo &);
    // ... 更多创建方法

    //===--------------------------------------------------------------------===//
    // 对象查找
    //===--------------------------------------------------------------------===//
    optional_ptr<CatalogEntry> GetEntry(
        ClientContext &context,
        const string &schema,
        const EntryLookupInfo &lookup_info,
        OnEntryNotFound if_not_found);

    //===--------------------------------------------------------------------===//
    // 物理计划生成（虚方法，由子类实现）
    //===--------------------------------------------------------------------===//
    virtual PhysicalOperator &PlanCreateTableAs(...) = 0;
    virtual PhysicalOperator &PlanInsert(...) = 0;
    virtual PhysicalOperator &PlanDelete(...) = 0;
    virtual PhysicalOperator &PlanUpdate(...) = 0;

protected:
    AttachedDatabase &db;  // 关联的 AttachedDatabase
};
```

### 1.2.2 DuckCatalog 实现

`DuckCatalog` 是 DuckDB 原生的 Catalog 实现：

```cpp
// src/include/duckdb/catalog/duck_catalog.hpp
class DuckCatalog : public Catalog {
public:
    explicit DuckCatalog(AttachedDatabase &db);
    ~DuckCatalog() override;

    bool IsDuckCatalog() override { return true; }

    string GetCatalogType() override { return "duckdb"; }

    void Initialize(bool load_builtin) override;

    //! Schema 操作
    optional_ptr<CatalogEntry> CreateSchema(
        CatalogTransaction transaction, CreateSchemaInfo &info) override;

    optional_ptr<SchemaCatalogEntry> LookupSchema(
        CatalogTransaction transaction,
        const EntryLookupInfo &schema_lookup,
        OnEntryNotFound if_not_found) override;

    void ScanSchemas(ClientContext &context,
                     std::function<void(SchemaCatalogEntry &)> callback) override;

    //! 获取依赖管理器
    optional_ptr<DependencyManager> GetDependencyManager() override;

    //! 获取写锁
    mutex &GetWriteLock() { return write_lock; }

    //! 获取 Schema CatalogSet
    CatalogSet &GetSchemaCatalogSet();

private:
    //! 依赖管理器
    unique_ptr<DependencyManager> dependency_manager;

    //! 写锁（保护并发修改）
    mutex write_lock;

    //! Schema 集合（CatalogSet 管理所有 Schema）
    unique_ptr<CatalogSet> schemas;

    //! 加密相关
    bool is_encrypted = false;
    string encryption_key_id;
};
```

### 1.2.3 DuckCatalog 初始化

```cpp
// src/catalog/duck_catalog.cpp
DuckCatalog::DuckCatalog(AttachedDatabase &db)
    : Catalog(db),
      dependency_manager(make_uniq<DependencyManager>(*this)),
      schemas(make_uniq<CatalogSet>(
          *this,
          IsSystemCatalog() ? make_uniq<DefaultSchemaGenerator>(*this) : nullptr)) {
}

void DuckCatalog::Initialize(bool load_builtin) {
    // 获取系统事务
    auto data = CatalogTransaction::GetSystemTransaction(GetDatabase());

    // 创建默认 Schema "main"
    CreateSchemaInfo info;
    info.schema = DEFAULT_SCHEMA;  // "main"
    info.internal = true;
    info.on_conflict = OnCreateConflict::IGNORE_ON_CONFLICT;
    CreateSchema(data, info);

    // 加载内置函数
    if (load_builtin) {
        BuiltinFunctions builtin(data, *this);
        builtin.Initialize();

        // 注册函数列表
        FunctionList::RegisterFunctions(*this, data);
    }

    Verify();
}
```

### 1.2.4 Catalog 抽象层次图

```
┌─────────────────────────────────────────────────────────────────┐
│                      Catalog (抽象基类)                          │
│                                                                  │
│  核心接口:                                                       │
│  • CreateSchema / LookupSchema / ScanSchemas                    │
│  • CreateTable / CreateView / CreateFunction / ...              │
│  • GetEntry / DropEntry / Alter                                 │
│  • PlanCreateTableAs / PlanInsert / PlanDelete / ...            │
│                                                                  │
│  成员:                                                           │
│  • AttachedDatabase &db                                         │
├─────────────────────────────────────────────────────────────────┤
│                              ↓                                   │
│         ┌───────────────────┴───────────────────┐               │
│         ↓                                       ↓               │
│  ┌─────────────────────────┐    ┌─────────────────────────┐    │
│  │     DuckCatalog         │    │   外部 Catalog 实现      │    │
│  │     (DuckDB 原生)        │    │   (扩展提供)             │    │
│  │                         │    │                         │    │
│  │  成员:                   │    │  • PostgresCatalog      │    │
│  │  • dependency_manager   │    │  • SQLiteCatalog        │    │
│  │  • schemas (CatalogSet) │    │  • MySQLCatalog         │    │
│  │  • write_lock           │    │  • ...                  │    │
│  │                         │    │                         │    │
│  │  特性:                   │    │  特性:                   │    │
│  │  • 完整 MVCC 支持        │    │  • 远程数据库连接        │    │
│  │  • 依赖管理              │    │  • 协议适配              │    │
│  │  • 持久化存储            │    │  • 类型映射              │    │
│  └─────────────────────────┘    └─────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1.3 AttachedDatabase 与多数据库支持

### 1.3.1 AttachedDatabase 类

DuckDB 支持同时附加多个数据库，每个数据库由 `AttachedDatabase` 表示：

```cpp
// src/include/duckdb/main/attached_database.hpp

enum class AttachedDatabaseType {
    READ_WRITE_DATABASE,  // 读写数据库
    READ_ONLY_DATABASE,   // 只读数据库
    SYSTEM_DATABASE,      // 系统数据库
    TEMP_DATABASE,        // 临时数据库
};

class AttachedDatabase : public CatalogEntry {
public:
    //! 创建系统数据库（无存储）
    explicit AttachedDatabase(DatabaseInstance &db,
                              AttachedDatabaseType type = AttachedDatabaseType::SYSTEM_DATABASE);

    //! 创建带存储的附加数据库
    AttachedDatabase(DatabaseInstance &db, Catalog &catalog,
                     string name, string file_path, AttachOptions &options);

    //! 获取组件
    Catalog &GetCatalog();
    StorageManager &GetStorageManager();
    TransactionManager &GetTransactionManager();
    DatabaseInstance &GetDatabase();

    //! 属性查询
    const string &GetName() const;
    bool IsSystem() const;
    bool IsTemporary() const;
    bool IsReadOnly() const;

private:
    DatabaseInstance &db;                          // 数据库实例
    unique_ptr<StorageManager> storage;            // 存储管理器
    unique_ptr<Catalog> catalog;                   // Catalog
    unique_ptr<TransactionManager> transaction_manager;  // 事务管理器
    AttachedDatabaseType type;                     // 数据库类型
};
```

### 1.3.2 多数据库架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      DatabaseInstance                            │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    DatabaseManager                         │  │
│  │                                                            │  │
│  │  管理所有 AttachedDatabase:                                 │  │
│  │                                                            │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │  │
│  │  │   "system"      │  │   "memory"      │                 │  │
│  │  │ AttachedDatabase│  │ AttachedDatabase│                 │  │
│  │  │ (SYSTEM)        │  │ (主数据库)       │                 │  │
│  │  └────────┬────────┘  └────────┬────────┘                 │  │
│  │           │                    │                           │  │
│  │           ▼                    ▼                           │  │
│  │     ┌──────────┐         ┌──────────┐                     │  │
│  │     │DuckCatalog│         │DuckCatalog│                     │  │
│  │     └──────────┘         └──────────┘                     │  │
│  │                                                            │  │
│  │  ┌─────────────────┐  ┌─────────────────┐                 │  │
│  │  │  "temp"         │  │  "attached_db"  │                 │  │
│  │  │ AttachedDatabase│  │ AttachedDatabase│                 │  │
│  │  │ (TEMP)          │  │ (附加的外部库)   │                 │  │
│  │  └────────┬────────┘  └────────┬────────┘                 │  │
│  │           │                    │                           │  │
│  │           ▼                    ▼                           │  │
│  │     ┌──────────┐         ┌──────────┐                     │  │
│  │     │DuckCatalog│         │DuckCatalog│                     │  │
│  │     │ (临时表)  │         │ (持久化) │                     │  │
│  │     └──────────┘         └──────────┘                     │  │
│  │                                                            │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3.3 特殊 Catalog

DuckDB 中有几个特殊的 Catalog：

```cpp
// 系统 Catalog（存储内置函数、类型等）
static constexpr const char *SYSTEM_CATALOG = "system";

// 临时 Catalog（存储临时表）
static constexpr const char *TEMP_CATALOG = "temp";

// 默认 Schema
static constexpr const char *DEFAULT_SCHEMA = "main";

// PostgreSQL 兼容 Schema
static constexpr const char *PG_CATALOG = "pg_catalog";
static constexpr const char *INFORMATION_SCHEMA = "information_schema";
```

### 1.3.4 获取 Catalog

```cpp
// src/catalog/catalog.cpp

// 获取系统 Catalog
Catalog &Catalog::GetSystemCatalog(ClientContext &context) {
    return Catalog::GetSystemCatalog(*context.db);
}

// 获取指定名称的 Catalog
optional_ptr<Catalog> Catalog::GetCatalogEntry(
    CatalogEntryRetriever &retriever,
    const string &catalog_name) {

    auto &context = retriever.GetContext();
    auto &db_manager = DatabaseManager::Get(context);

    // 特殊处理临时 Catalog
    if (catalog_name == TEMP_CATALOG) {
        return &ClientData::Get(context).temporary_objects->GetCatalog();
    }

    // 特殊处理系统 Catalog
    if (catalog_name == SYSTEM_CATALOG) {
        return &GetSystemCatalog(context);
    }

    // 查找附加的数据库
    auto entry = db_manager.GetDatabase(
        context,
        IsInvalidCatalog(catalog_name) ? GetDefaultCatalog(retriever) : catalog_name);

    if (!entry) {
        return nullptr;
    }
    return &entry->GetCatalog();
}
```

---

## 1.4 CatalogEntry 继承体系

### 1.4.1 CatalogEntry 基类

所有 Catalog 对象都继承自 `CatalogEntry`：

```cpp
// src/include/duckdb/catalog/catalog_entry.hpp

class CatalogEntry {
public:
    CatalogEntry(CatalogType type, Catalog &catalog, string name);
    CatalogEntry(CatalogType type, string name, idx_t oid);
    virtual ~CatalogEntry();

    //===--------------------------------------------------------------------===//
    // 核心属性
    //===--------------------------------------------------------------------===//
    idx_t oid;                    // 对象唯一标识符
    CatalogType type;             // 条目类型（TABLE, VIEW, FUNCTION 等）
    optional_ptr<CatalogSet> set; // 所属的 CatalogSet
    string name;                  // 条目名称

    //===--------------------------------------------------------------------===//
    // 状态标志
    //===--------------------------------------------------------------------===//
    bool deleted;                 // 是否已删除
    bool temporary;               // 是否为临时对象
    bool internal;                // 是否为内部对象

    //===--------------------------------------------------------------------===//
    // MVCC 相关
    //===--------------------------------------------------------------------===//
    atomic<transaction_t> timestamp;  // 创建/修改时的事务时间戳

    //===--------------------------------------------------------------------===//
    // 元数据
    //===--------------------------------------------------------------------===//
    Value comment;                                    // 注释
    InsertionOrderPreservingMap<string> tags;        // 标签

    //===--------------------------------------------------------------------===//
    // 版本链（MVCC）
    //===--------------------------------------------------------------------===//
private:
    unique_ptr<CatalogEntry> child;   // 子条目（更旧的版本）
    optional_ptr<CatalogEntry> parent; // 父条目（更新的版本）

public:
    void SetChild(unique_ptr<CatalogEntry> child);
    unique_ptr<CatalogEntry> TakeChild();
    bool HasChild() const;
    bool HasParent() const;
    CatalogEntry &Child();
    CatalogEntry &Parent();

    //===--------------------------------------------------------------------===//
    // 虚方法
    //===--------------------------------------------------------------------===//
    virtual unique_ptr<CatalogEntry> AlterEntry(ClientContext &context, AlterInfo &info);
    virtual unique_ptr<CatalogEntry> Copy(ClientContext &context) const;
    virtual unique_ptr<CreateInfo> GetInfo() const;
    virtual string ToSQL() const;

    virtual Catalog &ParentCatalog();
    virtual SchemaCatalogEntry &ParentSchema();
};
```

### 1.4.2 InCatalogEntry

`InCatalogEntry` 持有对 Catalog 的引用：

```cpp
// src/include/duckdb/catalog/catalog_entry.hpp

class InCatalogEntry : public CatalogEntry {
public:
    InCatalogEntry(CatalogType type, Catalog &catalog, string name);
    ~InCatalogEntry() override;

    //! 所属的 Catalog
    Catalog &catalog;

public:
    Catalog &ParentCatalog() override { return catalog; }
    const Catalog &ParentCatalog() const override { return catalog; }
};
```

### 1.4.3 StandardEntry

`StandardEntry` 表示属于 Schema 的对象：

```cpp
// src/include/duckdb/catalog/standard_entry.hpp

class StandardEntry : public InCatalogEntry {
public:
    StandardEntry(CatalogType type, SchemaCatalogEntry &schema,
                  Catalog &catalog, string name)
        : InCatalogEntry(type, catalog, std::move(name)), schema(schema) {
    }

    //! 所属的 Schema
    SchemaCatalogEntry &schema;

    //! 依赖列表
    LogicalDependencyList dependencies;

public:
    SchemaCatalogEntry &ParentSchema() override { return schema; }
    const SchemaCatalogEntry &ParentSchema() const override { return schema; }
};
```

### 1.4.4 完整继承层次

```
CatalogEntry                           // 抽象基类
│
│   属性: oid, type, name, deleted, temporary, internal, timestamp
│   版本链: child, parent
│
├── InCatalogEntry                     // 带 Catalog 引用
│   │
│   │   属性: Catalog &catalog
│   │
│   ├── SchemaCatalogEntry             // Schema 条目
│   │   │
│   │   │   虚方法: CreateTable, CreateView, CreateFunction, ...
│   │   │           LookupEntry, Scan, DropEntry, Alter
│   │   │
│   │   └── DuckSchemaEntry            // DuckDB Schema 实现
│   │           成员: tables, indexes, table_functions, functions,
│   │                  sequences, types, collations, copy_functions, ...
│   │
│   └── StandardEntry                  // 属于 Schema 的条目
│       │
│       │   属性: SchemaCatalogEntry &schema
│       │         LogicalDependencyList dependencies
│       │
│       ├── TableCatalogEntry          // 表
│       │   │   成员: columns, constraints
│       │   └── DuckTableEntry         // DuckDB 表实现
│       │           成员: storage (DataTable &)
│       │
│       ├── ViewCatalogEntry           // 视图
│       │       成员: query, aliases, types
│       │
│       ├── IndexCatalogEntry          // 索引
│       │   └── DuckIndexEntry         // DuckDB 索引
│       │
│       ├── SequenceCatalogEntry       // 序列
│       │       成员: usage_count, counter, start_value, ...
│       │
│       ├── TypeCatalogEntry           // 用户定义类型
│       │       成员: user_type
│       │
│       ├── CollateCatalogEntry        // 排序规则
│       │       成员: function
│       │
│       └── FunctionEntry              // 函数（抽象）
│           │   成员: functions (函数重载集合)
│           │
│           ├── ScalarFunctionCatalogEntry    // 标量函数
│           ├── AggregateFunctionCatalogEntry // 聚合函数
│           ├── TableFunctionCatalogEntry     // 表函数
│           ├── MacroCatalogEntry             // 标量宏
│           ├── TableMacroCatalogEntry        // 表宏
│           ├── PragmaFunctionCatalogEntry    // Pragma 函数
│           └── CopyFunctionCatalogEntry      // Copy 函数
│
└── DependencyEntry                    // 依赖关系条目
    ├── DependencySubjectEntry         // 被依赖对象
    └── DependencyDependentEntry       // 依赖方对象
```

### 1.4.5 CatalogType 枚举

```cpp
// src/include/duckdb/common/enums/catalog_type.hpp

enum class CatalogType : uint8_t {
    INVALID = 0,

    // 基础对象
    TABLE_ENTRY = 1,           // 表
    SCHEMA_ENTRY = 2,          // Schema
    VIEW_ENTRY = 3,            // 视图
    INDEX_ENTRY = 4,           // 索引
    PREPARED_STATEMENT = 5,    // 预编译语句
    SEQUENCE_ENTRY = 6,        // 序列
    COLLATION_ENTRY = 7,       // 排序规则
    TYPE_ENTRY = 8,            // 用户定义类型
    DATABASE_ENTRY = 9,        // 数据库

    // 函数类型
    TABLE_FUNCTION_ENTRY = 25,      // 表函数
    SCALAR_FUNCTION_ENTRY = 26,     // 标量函数
    AGGREGATE_FUNCTION_ENTRY = 27,  // 聚合函数
    PRAGMA_FUNCTION_ENTRY = 28,     // Pragma 函数
    COPY_FUNCTION_ENTRY = 29,       // Copy 函数
    MACRO_ENTRY = 30,               // 标量宏
    TABLE_MACRO_ENTRY = 31,         // 表宏

    // 依赖
    DEPENDENCY_ENTRY = 40,

    // 特殊
    RENAMED_ENTRY = 100,       // 重命名占位符
    DELETED_ENTRY = 101        // 删除占位符
};
```

---

## 1.5 Catalog 与 Schema 的关系

### 1.5.1 层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                     DuckCatalog                                  │
│                                                                  │
│  成员: schemas (CatalogSet)                                      │
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                    CatalogSet (schemas)                    │  │
│  │                                                            │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │  │
│  │  │   "main"    │  │"pg_catalog" │  │   "temp"    │        │  │
│  │  │ DuckSchema  │  │ DuckSchema  │  │ DuckSchema  │        │  │
│  │  │   Entry     │  │   Entry     │  │   Entry     │        │  │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │  │
│  │         │                │                │                │  │
│  └─────────┼────────────────┼────────────────┼────────────────┘  │
│            │                │                │                   │
│            ▼                ▼                ▼                   │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              DuckSchemaEntry 内部结构                        ││
│  │                                                              ││
│  │  CatalogSet tables       → TableCatalogEntry 集合            ││
│  │  CatalogSet indexes      → IndexCatalogEntry 集合            ││
│  │  CatalogSet functions    → ScalarFunctionCatalogEntry 集合   ││
│  │  CatalogSet table_functions → TableFunctionCatalogEntry 集合 ││
│  │  CatalogSet sequences    → SequenceCatalogEntry 集合         ││
│  │  CatalogSet types        → TypeCatalogEntry 集合             ││
│  │  CatalogSet collations   → CollateCatalogEntry 集合          ││
│  │  ...                                                         ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### 1.5.2 对象创建流程

以创建表为例：

```cpp
// Catalog::CreateTable 的调用链

// 1. 用户调用
Catalog::CreateTable(transaction, BoundCreateTableInfo &info)
    │
    ▼
// 2. 获取目标 Schema
auto &schema = GetSchema(transaction, info.base->schema);
    │
    ▼
// 3. 委托给 Schema 创建
return CreateTable(transaction, schema, info);
    │
    ▼
// 4. Schema 实际创建
schema.CreateTable(transaction, info);
    │
    ▼
// 5. DuckSchemaEntry 实现
DuckSchemaEntry::CreateTable(transaction, info) {
    // 创建 DuckTableEntry
    auto entry = make_uniq<DuckTableEntry>(catalog, *this, info);

    // 添加到 tables CatalogSet
    return AddEntry(transaction, tables, info, std::move(entry));
}
```

---

## 1.6 CatalogTransaction

### 1.6.1 结构定义

`CatalogTransaction` 封装了 Catalog 操作的事务上下文：

```cpp
// src/include/duckdb/catalog/catalog_transaction.hpp

struct CatalogTransaction {
    optional_ptr<DatabaseInstance> db;
    optional_ptr<ClientContext> context;
    optional_ptr<Transaction> transaction;
    transaction_t transaction_id;
    transaction_t start_time;

    CatalogTransaction(Catalog &catalog, ClientContext &context);
    CatalogTransaction(DatabaseInstance &db, transaction_t transaction_id_p,
                       transaction_t start_time_p);

    bool HasContext() const { return context; }
    ClientContext &GetContext();

    //! 创建系统事务（用于初始化）
    static CatalogTransaction GetSystemCatalogTransaction(ClientContext &context);
    static CatalogTransaction GetSystemTransaction(DatabaseInstance &db);
};
```

### 1.6.2 事务上下文使用

```cpp
// src/catalog/catalog_transaction.cpp

CatalogTransaction::CatalogTransaction(Catalog &catalog, ClientContext &context)
    : db(&catalog.GetDatabase()), context(&context) {

    // 获取活跃事务
    auto &meta_transaction = MetaTransaction::Get(context);
    transaction = &meta_transaction.GetTransaction(catalog.GetAttached());

    // 记录事务时间戳
    transaction_id = Transaction::GetCurrentTransactionId(*transaction);
    start_time = Transaction::GetCurrentLocalActiveTimestamp(*transaction);
}

// 系统事务用于 Catalog 初始化（在任何用户事务之前）
CatalogTransaction CatalogTransaction::GetSystemTransaction(DatabaseInstance &db) {
    return CatalogTransaction(db, 1, 1);  // transaction_id=1, start_time=1
}
```

---

## 1.7 本章小结

本章介绍了 DuckDB Catalog 系统的整体架构：

1. **Catalog 抽象层次**：
   - `Catalog` 是抽象基类，定义了统一的接口
   - `DuckCatalog` 是 DuckDB 原生实现，包含 `CatalogSet schemas`、`DependencyManager` 等核心组件
   - 外部扩展可以实现自己的 Catalog（如 PostgresCatalog）

2. **AttachedDatabase**：
   - 每个附加的数据库由 `AttachedDatabase` 表示
   - 包含 `Catalog`、`StorageManager`、`TransactionManager`
   - 支持多数据库同时工作，实现跨数据库查询

3. **CatalogEntry 继承体系**：
   - `CatalogEntry` 是所有条目的基类，包含 oid、type、name 等核心属性
   - `InCatalogEntry` 持有 Catalog 引用
   - `StandardEntry` 表示属于 Schema 的对象
   - 具体条目类型包括 Table、View、Function、Index、Sequence、Type 等

4. **层次组织**：
   - Catalog → Schema → 具体对象
   - 每层都使用 `CatalogSet` 管理其下级对象

5. **CatalogTransaction**：
   - 封装 Catalog 操作的事务上下文
   - 提供系统事务用于初始化

---

## 1.8 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/catalog.hpp` | Catalog 抽象基类定义 |
| `src/catalog/catalog.cpp` | Catalog 基类实现 |
| `src/include/duckdb/catalog/duck_catalog.hpp` | DuckCatalog 定义 |
| `src/catalog/duck_catalog.cpp` | DuckCatalog 实现 |
| `src/include/duckdb/catalog/catalog_entry.hpp` | CatalogEntry 基类 |
| `src/catalog/catalog_entry.cpp` | CatalogEntry 实现 |
| `src/include/duckdb/catalog/standard_entry.hpp` | StandardEntry 定义 |
| `src/include/duckdb/main/attached_database.hpp` | AttachedDatabase 定义 |
| `src/include/duckdb/catalog/catalog_transaction.hpp` | CatalogTransaction 定义 |
| `src/catalog/catalog_transaction.cpp` | CatalogTransaction 实现 |
