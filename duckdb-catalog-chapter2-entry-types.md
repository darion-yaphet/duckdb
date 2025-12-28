# DuckDB Catalog 系统深度解析 - 第二章：CatalogEntry 类型体系

本章详细分析 DuckDB 中各种 CatalogEntry 类型的实现，包括 Schema、Table、View、Function、Index、Sequence、Type 等核心条目类型。

## 2.1 Schema 条目

### 2.1.1 SchemaCatalogEntry

Schema 是数据库对象的命名空间容器：

```cpp
// src/include/duckdb/catalog/catalog_entry/schema_catalog_entry.hpp

class SchemaCatalogEntry : public InCatalogEntry {
public:
    static constexpr const CatalogType Type = CatalogType::SCHEMA_ENTRY;
    static constexpr const char *Name = "schema";

    SchemaCatalogEntry(Catalog &catalog, CreateSchemaInfo &info);

    //===--------------------------------------------------------------------===//
    // 对象创建方法（纯虚函数）
    //===--------------------------------------------------------------------===//
    virtual optional_ptr<CatalogEntry> CreateTable(
        CatalogTransaction transaction, BoundCreateTableInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateView(
        CatalogTransaction transaction, CreateViewInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateFunction(
        CatalogTransaction transaction, CreateFunctionInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateIndex(
        CatalogTransaction transaction, CreateIndexInfo &info,
        TableCatalogEntry &table) = 0;
    virtual optional_ptr<CatalogEntry> CreateSequence(
        CatalogTransaction transaction, CreateSequenceInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateType(
        CatalogTransaction transaction, CreateTypeInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateCollation(
        CatalogTransaction transaction, CreateCollationInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateTableFunction(
        CatalogTransaction transaction, CreateTableFunctionInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreateCopyFunction(
        CatalogTransaction transaction, CreateCopyFunctionInfo &info) = 0;
    virtual optional_ptr<CatalogEntry> CreatePragmaFunction(
        CatalogTransaction transaction, CreatePragmaFunctionInfo &info) = 0;

    //===--------------------------------------------------------------------===//
    // 查找与遍历
    //===--------------------------------------------------------------------===//
    virtual optional_ptr<CatalogEntry> LookupEntry(
        CatalogTransaction transaction, const EntryLookupInfo &lookup_info) = 0;

    virtual void Scan(ClientContext &context, CatalogType type,
                      const std::function<void(CatalogEntry &)> &callback) = 0;

    //===--------------------------------------------------------------------===//
    // 修改与删除
    //===--------------------------------------------------------------------===//
    virtual void DropEntry(ClientContext &context, DropInfo &info) = 0;
    virtual void Alter(CatalogTransaction transaction, AlterInfo &info) = 0;
};
```

### 2.1.2 DuckSchemaEntry 实现

`DuckSchemaEntry` 是 DuckDB 原生的 Schema 实现：

```cpp
// src/catalog/catalog_entry/duck_schema_entry.cpp

DuckSchemaEntry::DuckSchemaEntry(Catalog &catalog, CreateSchemaInfo &info)
    : SchemaCatalogEntry(catalog, info),
      // 表和视图共享同一个 CatalogSet（使用 DefaultViewGenerator 生成系统视图）
      tables(catalog, catalog.IsSystemCatalog()
                 ? make_uniq<DefaultViewGenerator>(catalog, *this) : nullptr),
      // 索引
      indexes(catalog),
      // 表函数（使用 DefaultTableFunctionGenerator）
      table_functions(catalog, catalog.IsSystemCatalog()
                 ? make_uniq<DefaultTableFunctionGenerator>(catalog, *this) : nullptr),
      // Copy 函数
      copy_functions(catalog),
      // Pragma 函数
      pragma_functions(catalog),
      // 标量/聚合函数（使用 DefaultFunctionGenerator）
      functions(catalog, catalog.IsSystemCatalog()
                 ? make_uniq<DefaultFunctionGenerator>(catalog, *this) : nullptr),
      // 序列
      sequences(catalog),
      // 排序规则
      collations(catalog),
      // 类型（使用 DefaultTypeGenerator）
      types(catalog, make_uniq<DefaultTypeGenerator>(catalog, *this)) {
}
```

### 2.1.3 Schema 内部 CatalogSet 分布

```
┌─────────────────────────────────────────────────────────────────┐
│                     DuckSchemaEntry                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  CatalogSet tables          → TABLE_ENTRY, VIEW_ENTRY           │
│  CatalogSet indexes         → INDEX_ENTRY                       │
│  CatalogSet table_functions → TABLE_FUNCTION_ENTRY,             │
│                               TABLE_MACRO_ENTRY                  │
│  CatalogSet copy_functions  → COPY_FUNCTION_ENTRY               │
│  CatalogSet pragma_functions→ PRAGMA_FUNCTION_ENTRY             │
│  CatalogSet functions       → SCALAR_FUNCTION_ENTRY,            │
│                               AGGREGATE_FUNCTION_ENTRY,          │
│                               MACRO_ENTRY                        │
│  CatalogSet sequences       → SEQUENCE_ENTRY                    │
│  CatalogSet collations      → COLLATION_ENTRY                   │
│  CatalogSet types           → TYPE_ENTRY                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1.4 GetCatalogSet 方法

```cpp
// 根据 CatalogType 返回对应的 CatalogSet
CatalogSet &DuckSchemaEntry::GetCatalogSet(CatalogType type) {
    switch (type) {
    case CatalogType::VIEW_ENTRY:
    case CatalogType::TABLE_ENTRY:
        return tables;  // 表和视图共用同一个 CatalogSet
    case CatalogType::INDEX_ENTRY:
        return indexes;
    case CatalogType::TABLE_FUNCTION_ENTRY:
    case CatalogType::TABLE_MACRO_ENTRY:
        return table_functions;
    case CatalogType::COPY_FUNCTION_ENTRY:
        return copy_functions;
    case CatalogType::PRAGMA_FUNCTION_ENTRY:
        return pragma_functions;
    case CatalogType::AGGREGATE_FUNCTION_ENTRY:
    case CatalogType::SCALAR_FUNCTION_ENTRY:
    case CatalogType::MACRO_ENTRY:
        return functions;
    case CatalogType::SEQUENCE_ENTRY:
        return sequences;
    case CatalogType::COLLATION_ENTRY:
        return collations;
    case CatalogType::TYPE_ENTRY:
        return types;
    default:
        throw InternalException("Unsupported catalog type in schema");
    }
}
```

---

## 2.2 Table 条目

### 2.2.1 TableCatalogEntry

表是数据库中最核心的对象：

```cpp
// src/include/duckdb/catalog/catalog_entry/table_catalog_entry.hpp

class TableCatalogEntry : public StandardEntry {
public:
    static constexpr const CatalogType Type = CatalogType::TABLE_ENTRY;
    static constexpr const char *Name = "table";

    TableCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                      CreateTableInfo &info);

protected:
    //! 列定义列表
    ColumnList columns;
    //! 约束列表
    vector<unique_ptr<Constraint>> constraints;

public:
    //===--------------------------------------------------------------------===//
    // 列操作
    //===--------------------------------------------------------------------===//
    bool ColumnExists(const string &name) const;
    const ColumnDefinition &GetColumn(const string &name) const;
    const ColumnDefinition &GetColumn(LogicalIndex idx) const;
    const ColumnList &GetColumns() const;
    vector<LogicalType> GetTypes() const;
    LogicalIndex GetColumnIndex(string &name, bool if_exists = false) const;

    //===--------------------------------------------------------------------===//
    // 约束
    //===--------------------------------------------------------------------===//
    const vector<unique_ptr<Constraint>> &GetConstraints() const;
    optional_ptr<Constraint> GetPrimaryKey() const;
    bool HasPrimaryKey() const;

    //===--------------------------------------------------------------------===//
    // 存储与统计（纯虚函数）
    //===--------------------------------------------------------------------===//
    virtual DataTable &GetStorage();
    virtual unique_ptr<BaseStatistics> GetStatistics(
        ClientContext &context, column_t column_id) = 0;
    virtual TableStorageInfo GetStorageInfo(ClientContext &context) = 0;

    //===--------------------------------------------------------------------===//
    // 扫描函数
    //===--------------------------------------------------------------------===//
    virtual TableFunction GetScanFunction(
        ClientContext &context, unique_ptr<FunctionData> &bind_data) = 0;

    //! 是否为 DuckDB 原生表
    virtual bool IsDuckTable() const { return false; }
};
```

### 2.2.2 DuckTableEntry 实现

`DuckTableEntry` 管理实际的物理存储：

```cpp
// src/catalog/catalog_entry/duck_table_entry.cpp

class DuckTableEntry : public TableCatalogEntry {
public:
    DuckTableEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                   BoundCreateTableInfo &info,
                   shared_ptr<DataTable> inherited_storage = nullptr);

    bool IsDuckTable() const override { return true; }

private:
    //! 物理存储
    shared_ptr<DataTable> storage;
    //! 列依赖管理器（用于生成列）
    ColumnDependencyManager column_dependency_manager;
};
```

### 2.2.3 DuckTableEntry 构造过程

```cpp
DuckTableEntry::DuckTableEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                               BoundCreateTableInfo &info,
                               shared_ptr<DataTable> inherited_storage)
    : TableCatalogEntry(catalog, schema, info.Base()),
      storage(std::move(inherited_storage)),
      column_dependency_manager(std::move(info.column_dependency_manager)) {

    if (storage) {
        // 继承现有存储（如 ALTER TABLE 操作）
        if (!info.indexes.empty()) {
            storage->SetIndexStorageInfo(std::move(info.indexes));
        }
        return;
    }

    // 创建物理存储
    vector<ColumnDefinition> column_defs;
    for (auto &col_def : columns.Physical()) {
        column_defs.push_back(col_def.Copy());
    }
    storage = make_shared_ptr<DataTable>(
        catalog.GetAttached(),
        StorageManager::Get(catalog).GetTableIOManager(&info),
        schema.name, name,
        std::move(column_defs),
        std::move(info.data));

    // 为 UNIQUE、PRIMARY KEY、FOREIGN KEY 约束创建索引
    for (idx_t i = 0; i < constraints.size(); i++) {
        auto &constraint = constraints[i];
        if (constraint->type == ConstraintType::UNIQUE) {
            auto &unique = constraint->Cast<UniqueConstraint>();
            IndexConstraintType constraint_type =
                unique.is_primary_key ? IndexConstraintType::PRIMARY
                                      : IndexConstraintType::UNIQUE;
            auto column_indexes = unique.GetLogicalIndexes(columns);
            auto index_info = GetIndexInfo(constraint_type, false, info.base, i);
            storage->AddIndex(columns, column_indexes, constraint_type, index_info);
        }
        // ... 处理 FOREIGN KEY 约束
    }
}
```

### 2.2.4 ALTER TABLE 操作

DuckTableEntry 支持丰富的 ALTER TABLE 操作：

```cpp
unique_ptr<CatalogEntry> DuckTableEntry::AlterEntry(
    ClientContext &context, AlterInfo &info) {

    auto &table_info = info.Cast<AlterTableInfo>();
    switch (table_info.alter_table_type) {
    case AlterTableType::RENAME_COLUMN:
        return RenameColumn(context, table_info.Cast<RenameColumnInfo>());

    case AlterTableType::RENAME_TABLE: {
        auto copied_table = Copy(context);
        copied_table->name = rename_info.new_table_name;
        storage->SetTableName(rename_info.new_table_name);
        return copied_table;
    }

    case AlterTableType::ADD_COLUMN:
        return AddColumn(context, table_info.Cast<AddColumnInfo>());

    case AlterTableType::REMOVE_COLUMN:
        return RemoveColumn(context, table_info.Cast<RemoveColumnInfo>());

    case AlterTableType::ALTER_COLUMN_TYPE:
        return ChangeColumnType(context, table_info.Cast<ChangeColumnTypeInfo>());

    case AlterTableType::SET_DEFAULT:
        return SetDefault(context, table_info.Cast<SetDefaultInfo>());

    case AlterTableType::SET_NOT_NULL:
        return SetNotNull(context, table_info.Cast<SetNotNullInfo>());

    case AlterTableType::DROP_NOT_NULL:
        return DropNotNull(context, table_info.Cast<DropNotNullInfo>());

    case AlterTableType::ADD_CONSTRAINT:
        return AddConstraint(context, table_info.Cast<AddConstraintInfo>());

    // ... 更多操作
    }
}
```

### 2.2.5 ALTER TABLE 的 Copy-On-Write 模式

```
ALTER TABLE 流程（以 ADD COLUMN 为例）:

1. 创建新的 CreateTableInfo
   ├── 复制原有列定义
   └── 添加新列定义

2. 绑定新的表信息
   └── Binder::BindCreateTableInfo()

3. 创建新的 DataTable（包含新列）
   └── make_shared_ptr<DataTable>(context, *storage, new_column, default_value)

4. 创建新的 DuckTableEntry
   └── make_uniq<DuckTableEntry>(catalog, schema, *bound_create_info, new_storage)

5. 返回新条目（将替换 CatalogSet 中的旧条目）

┌─────────────┐     ALTER TABLE      ┌─────────────┐
│ 旧 Entry    │ ──────────────────→  │ 新 Entry    │
│             │                      │             │
│ storage ────┼─┐                    │ storage ────┼──→ 新 DataTable
│             │ │                    │             │
└─────────────┘ │                    └─────────────┘
                │
                └──→ 旧 DataTable（可能被回收）
```

---

## 2.3 View 条目

### 2.3.1 ViewCatalogEntry

视图存储 SQL 查询定义：

```cpp
// src/include/duckdb/catalog/catalog_entry/view_catalog_entry.hpp

class ViewCatalogEntry : public StandardEntry {
public:
    static constexpr const CatalogType Type = CatalogType::VIEW_ENTRY;
    static constexpr const char *Name = "view";

    ViewCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                     CreateViewInfo &info);

    //! 视图的 SQL 查询
    unique_ptr<SelectStatement> query;
    //! 列别名
    vector<string> aliases;
    //! 列类型
    vector<LogicalType> types;
    //! 列名
    vector<string> names;
    //! 原始 SQL 字符串
    string sql;
    //! 列注释
    vector<Value> column_comments;

    const SelectStatement &GetQuery();
    unique_ptr<CreateInfo> GetInfo() const override;
    unique_ptr<CatalogEntry> AlterEntry(ClientContext &context, AlterInfo &info) override;
    unique_ptr<CatalogEntry> Copy(ClientContext &context) const override;
    string ToSQL() const override;
};
```

### 2.3.2 View 与 Table 共享 CatalogSet

```cpp
// DuckSchemaEntry 中，VIEW_ENTRY 和 TABLE_ENTRY 使用同一个 CatalogSet
CatalogSet &DuckSchemaEntry::GetCatalogSet(CatalogType type) {
    switch (type) {
    case CatalogType::VIEW_ENTRY:
    case CatalogType::TABLE_ENTRY:
        return tables;  // 共享 tables CatalogSet
    // ...
    }
}
```

这意味着：
- 表和视图不能同名
- 可以用视图替换表（CREATE OR REPLACE VIEW）
- 查找时按名称匹配，不区分是表还是视图

---

## 2.4 Function 条目

### 2.4.1 函数类型层次

```
FunctionEntry (抽象)
│
│   成员: descriptions (函数描述列表)
│
├── ScalarFunctionCatalogEntry
│       成员: ScalarFunctionSet functions
│       说明: 无状态的行级函数（如 sin, cos, concat）
│
├── AggregateFunctionCatalogEntry
│       成员: AggregateFunctionSet functions
│       说明: 有状态的聚合函数（如 sum, count, avg）
│
├── TableFunctionCatalogEntry
│       成员: TableFunctionSet functions
│       说明: 返回表的函数（如 read_csv, range）
│
├── MacroCatalogEntry (抽象)
│   │
│   ├── ScalarMacroCatalogEntry
│   │       成员: ScalarMacroFunction
│   │       说明: 用户定义的标量宏
│   │
│   └── TableMacroCatalogEntry
│           成员: TableMacroFunction
│           说明: 用户定义的表宏
│
├── PragmaFunctionCatalogEntry
│       成员: PragmaFunctionSet functions
│       说明: PRAGMA 函数
│
└── CopyFunctionCatalogEntry
        成员: CopyFunction
        说明: COPY 格式处理函数（如 CSV, PARQUET）
```

### 2.4.2 ScalarFunctionCatalogEntry

```cpp
// src/catalog/catalog_entry/scalar_function_catalog_entry.cpp

class ScalarFunctionCatalogEntry : public FunctionEntry {
public:
    ScalarFunctionCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                               CreateScalarFunctionInfo &info)
        : FunctionEntry(CatalogType::SCALAR_FUNCTION_ENTRY, catalog, schema, info),
          functions(info.functions) {
        // 为每个函数重载设置 catalog 和 schema 名称
        for (auto &function : functions.functions) {
            function.catalog_name = catalog.GetAttached().GetName();
            function.schema_name = schema.name;
        }
    }

    //! 函数重载集合
    ScalarFunctionSet functions;

    unique_ptr<CatalogEntry> AlterEntry(CatalogTransaction transaction,
                                        AlterInfo &info) override;
};
```

### 2.4.3 函数重载添加

```cpp
unique_ptr<CatalogEntry> ScalarFunctionCatalogEntry::AlterEntry(
    CatalogTransaction transaction, AlterInfo &info) {

    auto &add_overloads = info.Cast<AddScalarFunctionOverloadInfo>();

    // 合并新的重载到现有函数集
    ScalarFunctionSet new_set = functions;
    if (!new_set.MergeFunctionSet(add_overloads.new_overloads->functions, true)) {
        throw BinderException(
            "Failed to add new function overloads: function overload already exists");
    }

    // 创建新的 entry
    CreateScalarFunctionInfo new_info(std::move(new_set));
    new_info.internal = internal;
    new_info.descriptions = descriptions;
    new_info.descriptions.insert(new_info.descriptions.end(),
                                  add_overloads.new_overloads->descriptions.begin(),
                                  add_overloads.new_overloads->descriptions.end());
    return make_uniq<ScalarFunctionCatalogEntry>(catalog, schema, new_info);
}
```

### 2.4.4 CreateFunction 分发

DuckSchemaEntry 根据函数类型创建不同的 CatalogEntry：

```cpp
optional_ptr<CatalogEntry> DuckSchemaEntry::CreateFunction(
    CatalogTransaction transaction, CreateFunctionInfo &info) {

    unique_ptr<StandardEntry> function;
    switch (info.type) {
    case CatalogType::SCALAR_FUNCTION_ENTRY:
        function = make_uniq_base<StandardEntry, ScalarFunctionCatalogEntry>(
            catalog, *this, info.Cast<CreateScalarFunctionInfo>());
        break;

    case CatalogType::TABLE_FUNCTION_ENTRY:
        function = make_uniq_base<StandardEntry, TableFunctionCatalogEntry>(
            catalog, *this, info.Cast<CreateTableFunctionInfo>());
        break;

    case CatalogType::MACRO_ENTRY:
        function = make_uniq_base<StandardEntry, ScalarMacroCatalogEntry>(
            catalog, *this, info.Cast<CreateMacroInfo>());
        break;

    case CatalogType::TABLE_MACRO_ENTRY:
        function = make_uniq_base<StandardEntry, TableMacroCatalogEntry>(
            catalog, *this, info.Cast<CreateMacroInfo>());
        break;

    case CatalogType::AGGREGATE_FUNCTION_ENTRY:
        function = make_uniq_base<StandardEntry, AggregateFunctionCatalogEntry>(
            catalog, *this, info.Cast<CreateAggregateFunctionInfo>());
        break;

    default:
        throw InternalException("Unknown function type");
    }

    function->internal = info.internal;
    return AddEntry(transaction, std::move(function), info.on_conflict);
}
```

---

## 2.5 Index 条目

### 2.5.1 IndexCatalogEntry

```cpp
// src/include/duckdb/catalog/catalog_entry/index_catalog_entry.hpp

class IndexCatalogEntry : public StandardEntry {
public:
    static constexpr const CatalogType Type = CatalogType::INDEX_ENTRY;
    static constexpr const char *Name = "index";

    IndexCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                      CreateIndexInfo &info, TableCatalogEntry &table);

    //! 索引名称
    string index_name;
    //! 关联的表名
    string table_name;
    //! 是否唯一
    bool is_unique;
    //! 是否为主键
    bool is_primary;
    //! 索引列信息
    vector<IndexColumnInfo> column_infos;
    //! 原始 SQL
    string sql;

    virtual unique_ptr<CreateInfo> GetInfo() const override;
    virtual string ToSQL() const override;
};
```

### 2.5.2 DuckIndexEntry

```cpp
// src/include/duckdb/catalog/catalog_entry/duck_index_entry.hpp

class DuckIndexEntry : public IndexCatalogEntry {
public:
    DuckIndexEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                   CreateIndexInfo &info, TableCatalogEntry &table);

    //! 物理索引
    shared_ptr<Index> index;
    //! 初始索引大小
    idx_t initial_index_size;
};
```

### 2.5.3 索引创建流程

```cpp
optional_ptr<CatalogEntry> DuckSchemaEntry::CreateIndex(
    CatalogTransaction transaction,
    CreateIndexInfo &info,
    TableCatalogEntry &table) {

    // 添加对表的依赖
    info.dependencies.AddDependency(table);

    // 检查索引名是否唯一
    if (info.on_conflict != OnCreateConflict::IGNORE_ON_CONFLICT &&
        !table.GetStorage().IndexNameIsUnique(info.index_name)) {
        throw CatalogException("An index with this name already exists!");
    }

    // 创建索引条目
    auto index = make_uniq<DuckIndexEntry>(catalog, *this, info, table);
    auto dependencies = index->dependencies;
    return AddEntryInternal(transaction, std::move(index),
                            info.on_conflict, dependencies);
}
```

---

## 2.6 Sequence 条目

### 2.6.1 SequenceCatalogEntry

序列用于生成自增数值：

```cpp
// src/include/duckdb/catalog/catalog_entry/sequence_catalog_entry.hpp

class SequenceCatalogEntry : public StandardEntry {
public:
    static constexpr const CatalogType Type = CatalogType::SEQUENCE_ENTRY;

    SequenceCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                         CreateSequenceInfo &info);

    //! 序列属性
    int64_t start_value;        // 起始值
    int64_t min_value;          // 最小值
    int64_t max_value;          // 最大值
    int64_t increment;          // 增量
    bool cycle;                 // 是否循环

    //! 序列状态
    atomic<int64_t> counter;    // 当前计数
    idx_t usage_count;          // 使用次数

    //! 获取下一个值
    int64_t NextValue();
    int64_t CurrentValue();
};
```

### 2.6.2 序列使用示例

```sql
-- 创建序列
CREATE SEQUENCE my_seq START 1 INCREMENT 1;

-- 获取下一个值
SELECT nextval('my_seq');  -- 1
SELECT nextval('my_seq');  -- 2

-- 查看当前值
SELECT currval('my_seq');  -- 2
```

---

## 2.7 Type 条目

### 2.7.1 TypeCatalogEntry

用户定义类型（主要用于 ENUM）：

```cpp
// src/include/duckdb/catalog/catalog_entry/type_catalog_entry.hpp

class TypeCatalogEntry : public StandardEntry {
public:
    static constexpr const CatalogType Type = CatalogType::TYPE_ENTRY;

    TypeCatalogEntry(Catalog &catalog, SchemaCatalogEntry &schema,
                     CreateTypeInfo &info);

    //! 用户定义的类型
    LogicalType user_type;

    unique_ptr<CreateInfo> GetInfo() const override;
    string ToSQL() const override;
};
```

### 2.7.2 ENUM 类型示例

```sql
-- 创建 ENUM 类型
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');

-- 使用 ENUM
CREATE TABLE person (
    name VARCHAR,
    current_mood mood
);

INSERT INTO person VALUES ('Alice', 'happy');
```

---

## 2.8 条目创建通用流程

### 2.8.1 AddEntryInternal

所有条目创建最终调用 `AddEntryInternal`：

```cpp
optional_ptr<CatalogEntry> DuckSchemaEntry::AddEntryInternal(
    CatalogTransaction transaction,
    unique_ptr<StandardEntry> entry,
    OnCreateConflict on_conflict,
    LogicalDependencyList dependencies) {

    auto entry_name = entry->name;
    auto entry_type = entry->type;
    auto result = entry.get();

    // 获取对应类型的 CatalogSet
    auto &set = GetCatalogSet(entry_type);

    // 添加对 Schema 的依赖
    dependencies.AddDependency(*this);

    // 处理冲突策略
    if (on_conflict == OnCreateConflict::IGNORE_ON_CONFLICT) {
        auto old_entry = set.GetEntry(transaction, entry_name);
        if (old_entry) {
            return nullptr;  // 已存在，忽略
        }
    }

    if (on_conflict == OnCreateConflict::REPLACE_ON_CONFLICT) {
        auto old_entry = set.GetEntry(transaction, entry_name);
        if (old_entry) {
            // 检查依赖循环
            if (dependencies.Contains(*old_entry)) {
                throw CatalogException(
                    "CREATE OR REPLACE is not allowed to depend on itself");
            }
            // 检查类型匹配
            if (old_entry->type != entry_type) {
                throw CatalogException(
                    "Existing object is of type %s, trying to replace with type %s",
                    CatalogTypeToString(old_entry->type),
                    CatalogTypeToString(entry_type));
            }
            // 删除旧条目
            OnDropEntry(transaction, *old_entry);
            set.DropEntry(transaction, entry_name, false, entry->internal);
        }
    }

    // 创建条目
    if (!set.CreateEntry(transaction, entry_name, std::move(entry), dependencies)) {
        if (on_conflict == OnCreateConflict::ERROR_ON_CONFLICT) {
            throw CatalogException::EntryAlreadyExists(entry_type, entry_name);
        }
        return nullptr;
    }

    return result;
}
```

### 2.8.2 创建流程图

```
CreateTable/View/Function/...
         │
         ▼
┌──────────────────────────────┐
│  1. 创建具体的 Entry 对象     │
│     (DuckTableEntry,         │
│      ViewCatalogEntry, ...)  │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  2. AddEntryInternal         │
│     • 获取 CatalogSet        │
│     • 添加 Schema 依赖       │
│     • 处理冲突策略           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  3. CatalogSet.CreateEntry   │
│     • 检查名称唯一性         │
│     • 注册依赖关系           │
│     • 设置事务时间戳         │
│     • 添加到 map             │
└──────────────┬───────────────┘
               │
               ▼
        返回创建的 Entry
```

---

## 2.9 条目删除流程

### 2.9.1 DropEntry

```cpp
void DuckSchemaEntry::DropEntry(ClientContext &context, DropInfo &info) {
    auto &set = GetCatalogSet(info.type);

    // 查找条目
    auto transaction = GetCatalogTransaction(context);
    auto existing_entry = set.GetEntry(transaction, info.name);
    if (!existing_entry) {
        throw InternalException("Failed to drop entry - entry not found");
    }

    // 检查类型匹配
    if (existing_entry->type != info.type) {
        throw CatalogException(
            "Existing object is of type %s, trying to drop type %s",
            CatalogTypeToString(existing_entry->type),
            CatalogTypeToString(info.type));
    }

    // 处理外键约束
    vector<unique_ptr<AlterForeignKeyInfo>> fk_arrays;
    if (existing_entry->type == CatalogType::TABLE_ENTRY) {
        auto &table_entry = existing_entry->Cast<TableCatalogEntry>();
        FindForeignKeyInformation(table_entry, AlterForeignKeyType::AFT_DELETE, fk_arrays);
    }

    // 触发删除回调
    OnDropEntry(transaction, *existing_entry);

    // 执行删除
    if (!set.DropEntry(transaction, info.name, info.cascade, info.allow_drop_internal)) {
        throw InternalException("Could not drop element");
    }

    // 更新外键引用
    for (auto &fk_info : fk_arrays) {
        Alter(transaction, *fk_info);
    }
}
```

---

## 2.10 本章小结

本章详细介绍了 DuckDB 各种 CatalogEntry 类型：

1. **Schema 条目**：
   - `DuckSchemaEntry` 包含多个 CatalogSet 管理不同类型的对象
   - 表和视图共享同一个 CatalogSet

2. **Table 条目**：
   - `DuckTableEntry` 持有 `DataTable` 物理存储
   - ALTER TABLE 使用 Copy-On-Write 模式
   - 支持丰富的修改操作

3. **View 条目**：
   - 存储 SQL 查询定义
   - 与表共享命名空间

4. **Function 条目**：
   - 多种函数类型（Scalar, Aggregate, Table, Macro, Pragma, Copy）
   - 支持函数重载

5. **其他条目**：
   - Index：关联表的索引
   - Sequence：自增数值生成器
   - Type：用户定义类型（ENUM）

6. **通用流程**：
   - 创建：`AddEntryInternal` → `CatalogSet.CreateEntry`
   - 删除：`DropEntry` → `CatalogSet.DropEntry`
   - 支持冲突策略（ERROR, IGNORE, REPLACE）

---

## 2.11 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/catalog/catalog_entry/schema_catalog_entry.cpp` | Schema 基类 |
| `src/catalog/catalog_entry/duck_schema_entry.cpp` | DuckDB Schema 实现 |
| `src/catalog/catalog_entry/table_catalog_entry.cpp` | Table 基类 |
| `src/catalog/catalog_entry/duck_table_entry.cpp` | DuckDB Table 实现 |
| `src/catalog/catalog_entry/view_catalog_entry.cpp` | View 实现 |
| `src/catalog/catalog_entry/scalar_function_catalog_entry.cpp` | 标量函数 |
| `src/catalog/catalog_entry/table_function_catalog_entry.cpp` | 表函数 |
| `src/catalog/catalog_entry/scalar_macro_catalog_entry.cpp` | 标量宏 |
| `src/catalog/catalog_entry/table_macro_catalog_entry.cpp` | 表宏 |
| `src/catalog/catalog_entry/index_catalog_entry.cpp` | 索引基类 |
| `src/catalog/catalog_entry/duck_index_entry.cpp` | DuckDB 索引 |
| `src/catalog/catalog_entry/sequence_catalog_entry.cpp` | 序列 |
| `src/catalog/catalog_entry/type_catalog_entry.cpp` | 用户定义类型 |
