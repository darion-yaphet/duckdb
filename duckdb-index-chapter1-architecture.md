# DuckDB 索引系统深度解析 - 第一章：索引架构概述

本章深入分析 DuckDB 索引系统的整体架构设计，包括索引抽象层次、类型注册机制以及与存储引擎的集成方式。

## 1.1 索引系统设计理念

### 1.1.1 设计目标

DuckDB 作为一个嵌入式分析型数据库，其索引系统的设计目标与传统 OLTP 数据库有所不同：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DuckDB 索引设计目标                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 约束支持优先                                                             │
│     • PRIMARY KEY 约束验证                                                   │
│     • UNIQUE 约束保证                                                        │
│     • FOREIGN KEY 引用完整性                                                 │
│                                                                              │
│  2. 点查询加速                                                               │
│     • 等值查询优化                                                           │
│     • 主键快速定位                                                           │
│                                                                              │
│  3. 最小化存储开销                                                           │
│     • 自适应节点大小                                                         │
│     • 高效内存利用                                                           │
│                                                                              │
│  4. 延迟绑定                                                                 │
│     • 支持扩展索引类型                                                       │
│     • 数据库恢复时延迟加载                                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

与列式分析查询不同，索引在 DuckDB 中主要用于：
- **约束验证**：确保数据完整性（这是最核心的用途）
- **点查询**：快速定位特定行
- **外键检查**：验证引用关系

### 1.1.2 当前索引类型

目前 DuckDB 只实现了一种索引类型：

```cpp
// src/execution/index/index_type_set.cpp:7-15
IndexTypeSet::IndexTypeSet() {
    // Register the ART index type by default
    IndexType art_index_type;
    art_index_type.name = ART::TYPE_NAME;  // "ART"
    art_index_type.create_instance = ART::Create;
    art_index_type.create_plan = ART::CreatePlan;

    RegisterIndexType(art_index_type);
}
```

**ART (Adaptive Radix Tree)** 是 DuckDB 的唯一索引实现，但架构设计支持未来扩展其他索引类型。

## 1.2 索引抽象层次

DuckDB 索引系统采用三层抽象设计：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           索引类层次结构                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         ┌─────────────────┐                                 │
│                         │     Index       │  ← 抽象基类                      │
│                         │   (abstract)    │                                 │
│                         └────────┬────────┘                                 │
│                                  │                                           │
│              ┌───────────────────┴───────────────────┐                      │
│              │                                       │                      │
│              ▼                                       ▼                      │
│    ┌─────────────────┐                    ┌─────────────────┐              │
│    │   BoundIndex    │                    │  UnboundIndex   │              │
│    │  (内存中可用)    │                    │  (延迟绑定)      │              │
│    └────────┬────────┘                    └─────────────────┘              │
│             │                                                               │
│             ▼                                                               │
│    ┌─────────────────┐                                                      │
│    │      ART        │  ← 具体实现                                          │
│    │  (Adaptive      │                                                      │
│    │  Radix Tree)    │                                                      │
│    └─────────────────┘                                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2.1 Index 基类

`Index` 是所有索引的抽象基类，定义了索引的基本属性：

```cpp
// src/include/duckdb/storage/index.hpp:30-107
class Index {
protected:
    Index(const vector<column_t> &column_ids,
          TableIOManager &table_io_manager,
          AttachedDatabase &db);

    //! 索引列的物理 ID
    //! 例如：表 (a INT, gen AS (2*a), b INT, c VARCHAR)
    //! 在 (a,c) 上建索引，column_ids = [0, 2]（跳过虚拟列）
    vector<column_t> column_ids;

    //! column_ids 的无序集合（用于快速查找）
    unordered_set<column_t> column_id_set;

public:
    //! 关联的表 IO 管理器
    TableIOManager &table_io_manager;
    //! 附加的数据库实例
    AttachedDatabase &db;

public:
    //! 是否已绑定（纯虚函数）
    virtual bool IsBound() const = 0;

    //! 索引类型（ART、B+-tree 等）
    virtual const string &GetIndexType() const = 0;

    //! 索引名称
    virtual const string &GetIndexName() const = 0;

    //! 约束类型
    virtual IndexConstraintType GetConstraintType() const = 0;

    //! 便捷方法
    bool IsUnique() const {
        auto type = GetConstraintType();
        return type == IndexConstraintType::UNIQUE ||
               type == IndexConstraintType::PRIMARY;
    }

    bool IsPrimary() const {
        return GetConstraintType() == IndexConstraintType::PRIMARY;
    }

    bool IsForeign() const {
        return GetConstraintType() == IndexConstraintType::FOREIGN;
    }

    //! 删除索引
    virtual void CommitDrop() = 0;
};
```

**关键设计点**：

1. **物理列 ID**：`column_ids` 存储的是物理列 ID，跳过虚拟/生成列
2. **双重存储**：同时维护 `vector` 和 `unordered_set`，平衡有序访问和快速查找
3. **延迟绑定**：通过 `IsBound()` 区分已绑定和未绑定状态

### 1.2.2 索引约束类型

```cpp
// src/include/duckdb/common/enums/index_constraint_type.hpp:18-23
enum class IndexConstraintType : uint8_t {
    NONE = 0,     // 普通索引（无约束）
    UNIQUE = 1,   // UNIQUE 约束
    PRIMARY = 2,  // PRIMARY KEY 约束
    FOREIGN = 3   // FOREIGN KEY 约束
};
```

约束类型决定了索引的行为：

| 约束类型 | 唯一性检查 | 非空检查 | 引用检查 |
|----------|------------|----------|----------|
| NONE | ✗ | ✗ | ✗ |
| UNIQUE | ✓ | ✗ | ✗ |
| PRIMARY | ✓ | ✓ | ✗ |
| FOREIGN | ✗ | ✗ | ✓ |

## 1.3 BoundIndex：已绑定索引

`BoundIndex` 表示内存中完全可用的索引，是执行数据操作的核心类：

```cpp
// src/include/duckdb/execution/index/bound_index.hpp:46-205
class BoundIndex : public Index {
public:
    BoundIndex(const string &name,
               const string &index_type,
               IndexConstraintType index_constraint_type,
               const vector<column_t> &column_ids,
               TableIOManager &table_io_manager,
               const vector<unique_ptr<Expression>> &unbound_expressions,
               AttachedDatabase &db);

    //! 物理类型（用于内存操作）
    vector<PhysicalType> types;
    //! 逻辑类型（用于表达式求值）
    vector<LogicalType> logical_types;

    //! 索引元信息
    string name;
    string index_type;
    IndexConstraintType index_constraint_type;

    //! 未绑定表达式（用于序列化）
    //! 叶子节点是 BoundColumnRefExpression
    //! column_index 指向 Index::column_ids
    vector<unique_ptr<Expression>> unbound_expressions;

protected:
    //! 索引锁
    mutex lock;

    //! 绑定后的表达式（用于执行）
    //! 叶子节点是 BoundReferenceExpression
    //! 包含 DataChunk 中的偏移量
    vector<unique_ptr<Expression>> bound_expressions;

private:
    //! 表达式执行器
    ExpressionExecutor executor;
};
```

### 1.3.1 表达式绑定机制

BoundIndex 维护两套表达式：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        表达式绑定转换                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  unbound_expressions                      bound_expressions                 │
│  ┌───────────────────┐                   ┌───────────────────┐             │
│  │ BoundColumnRef    │                   │ BoundReference    │             │
│  │ • table_index=0   │  ──────────────▶  │ • index=物理列ID   │             │
│  │ • column_index=i  │     绑定转换       │                   │             │
│  └───────────────────┘                   └───────────────────┘             │
│                                                                              │
│  用途：                                   用途：                             │
│  • 序列化到磁盘                          • 执行时求值                        │
│  • 恢复时重新绑定                        • 从 DataChunk 提取数据             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

绑定转换的实现：

```cpp
// src/execution/index/bound_index.cpp:127-134
unique_ptr<Expression> BoundIndex::BindExpression(unique_ptr<Expression> root_expr) {
    ExpressionIterator::VisitExpressionMutable<BoundColumnRefExpression>(
        root_expr,
        [&](BoundColumnRefExpression &bound_colref, unique_ptr<Expression> &expr) {
            // 将 BoundColumnRefExpression 转换为 BoundReferenceExpression
            // column_ids[column_index] 获取实际的物理列 ID
            expr = make_uniq<BoundReferenceExpression>(
                expr->return_type,
                column_ids[bound_colref.binding.column_index]);
        });
    return root_expr;
}
```

### 1.3.2 核心操作接口

BoundIndex 定义了索引的核心操作：

```cpp
// 数据操作（纯虚函数，由 ART 实现）
virtual ErrorData Append(IndexLock &l, DataChunk &chunk, Vector &row_ids) = 0;
virtual ErrorData Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids) = 0;
virtual void Delete(IndexLock &state, DataChunk &entries, Vector &row_identifiers) = 0;

// 索引合并
virtual bool MergeIndexes(IndexLock &state, BoundIndex &other_index) = 0;

// 维护操作
virtual void Vacuum(IndexLock &l) = 0;
virtual void CommitDrop(IndexLock &index_lock) = 0;

// 约束验证
virtual void VerifyAppend(DataChunk &chunk, IndexAppendInfo &info,
                          optional_ptr<ConflictManager> manager);
virtual void VerifyConstraint(DataChunk &chunk, IndexAppendInfo &info,
                              ConflictManager &manager);

// 序列化
virtual IndexStorageInfo SerializeToDisk(QueryContext context,
                                         const case_insensitive_map_t<Value> &options);
virtual IndexStorageInfo SerializeToWAL(const case_insensitive_map_t<Value> &options);

// 诊断
virtual idx_t GetInMemorySize(IndexLock &state) = 0;
virtual string ToString(IndexLock &l, bool display_ascii = false) = 0;
virtual void Verify(IndexLock &l) = 0;
```

### 1.3.3 索引锁机制

所有操作都需要持有索引锁：

```cpp
// src/execution/index/bound_index.cpp:32-34
void BoundIndex::InitializeLock(IndexLock &state) {
    state.index_lock = unique_lock<mutex>(lock);
}

// 便捷方法：自动获取锁
ErrorData BoundIndex::Append(DataChunk &chunk, Vector &row_ids) {
    IndexLock l;
    InitializeLock(l);  // 获取锁
    return Append(l, chunk, row_ids);  // 调用带锁版本
}
```

### 1.3.4 追加模式

```cpp
// src/include/duckdb/execution/index/bound_index.hpp:32
enum class IndexAppendMode : uint8_t {
    DEFAULT = 0,           // 默认：检查约束
    IGNORE_DUPLICATES = 1, // 忽略重复（ON CONFLICT DO NOTHING）
    INSERT_DUPLICATES = 2  // 插入重复（WAL 重放时使用）
};

class IndexAppendInfo {
public:
    IndexAppendMode append_mode;
    optional_ptr<BoundIndex> delete_index;  // 用于 UPSERT 场景
};
```

## 1.4 UnboundIndex：延迟绑定索引

`UnboundIndex` 表示尚未绑定的索引，用于数据库恢复和扩展索引加载：

```cpp
// src/include/duckdb/execution/index/unbound_index.hpp:60-124
class UnboundIndex final : public Index {
private:
    //! 索引创建信息
    unique_ptr<CreateInfo> create_info;
    //! 序列化的存储信息
    IndexStorageInfo storage_info;

    //! WAL 重放时的缓冲操作
    BufferedIndexReplays buffered_replays;

    //! 列 ID 映射
    //! buffered ColumnDataCollection 中的 column[i] 对应
    //! 物理表的 mapped_column_ids[i] 列
    vector<StorageIndex> mapped_column_ids;

public:
    bool IsBound() const override { return false; }

    const string &GetIndexType() const override {
        return GetCreateInfo().index_type;
    }

    const string &GetIndexName() const override {
        return GetCreateInfo().index_name;
    }

    IndexConstraintType GetConstraintType() const override {
        return GetCreateInfo().constraint_type;
    }

    //! 缓冲 WAL 重放操作
    void BufferChunk(DataChunk &index_column_chunk,
                     Vector &row_ids,
                     const vector<StorageIndex> &mapped_column_ids_p,
                     BufferedIndexReplay replay_type);
};
```

### 1.4.1 WAL 重放缓冲

当数据库恢复时，可能需要在索引绑定之前缓冲一些操作：

```cpp
// src/include/duckdb/execution/index/unbound_index.hpp:19
enum class BufferedIndexReplay : uint8_t {
    INSERT_ENTRY = 0,  // 插入操作
    DEL_ENTRY = 1      // 删除操作
};

// 重放范围结构
struct ReplayRange {
    BufferedIndexReplay type;
    idx_t start;  // 起始位置（包含）
    idx_t end;    // 结束位置（不包含）
};

// 缓冲的重放操作
struct BufferedIndexReplays {
    vector<ReplayRange> ranges;                      // 操作顺序
    unique_ptr<ColumnDataCollection> buffered_inserts;  // 插入数据
    unique_ptr<ColumnDataCollection> buffered_deletes;  // 删除数据
};
```

**重放示例**：

```
ranges[0]: INSERT_ENTRY, [0, 6)   // 先插入 6 条
ranges[1]: DEL_ENTRY,    [0, 3)   // 再删除 3 条
ranges[2]: INSERT_ENTRY, [6, 12)  // 再插入 6 条

buffered_inserts: 包含所有 12 条插入数据
buffered_deletes: 包含所有 3 条删除数据

重放时按 ranges 顺序执行：
1. 从 buffered_inserts[0:6) 插入
2. 从 buffered_deletes[0:3) 删除
3. 从 buffered_inserts[6:12) 插入
```

### 1.4.2 绑定时重放

当 UnboundIndex 被绑定时，缓冲的操作会被重放：

```cpp
// src/execution/index/bound_index.cpp:183-249
void BoundIndex::ApplyBufferedReplays(
    const vector<LogicalType> &table_types,
    BufferedIndexReplays &buffered_replays,
    const vector<StorageIndex> &mapped_column_ids) {

    if (!buffered_replays.HasBufferedReplays()) {
        return;
    }

    array<BufferedReplayState, 2> replay_states;  // INSERT 和 DELETE 各一个
    DataChunk table_chunk;
    table_chunk.InitializeEmpty(table_types);

    for (const auto &replay_range : buffered_replays.ranges) {
        // 根据类型选择缓冲区
        auto &state = replay_states[static_cast<idx_t>(replay_range.type)];

        // 懒初始化扫描状态
        if (!state.scan_initialized) {
            state.buffer = buffered_replays.GetBuffer(replay_range.type);
            state.buffer->InitializeScan(state.scan_state);
            state.scan_initialized = true;
        }

        // 处理范围内的所有行
        idx_t current_row = replay_range.start;
        while (current_row < replay_range.end) {
            // 扫描下一个 chunk
            if (current_row >= state.scan_state.next_row_index) {
                state.buffer->Scan(state.scan_state, state.current_chunk);
            }

            // 计算本次处理的行数
            const auto rows_to_process = MinValue(
                state.current_chunk.size() - offset_in_chunk,
                replay_range.end - current_row);

            // 重放操作
            if (replay_range.type == BufferedIndexReplay::INSERT_ENTRY) {
                IndexAppendInfo append_info(IndexAppendMode::INSERT_DUPLICATES, nullptr);
                Append(table_chunk, row_ids, append_info);
            } else {
                Delete(table_chunk, row_ids);
            }
            current_row += rows_to_process;
        }
    }
}
```

## 1.5 索引类型注册机制

DuckDB 提供了可扩展的索引类型注册机制：

### 1.5.1 IndexType 结构

```cpp
// src/include/duckdb/execution/index/index_type.hpp:30-73

// 索引创建输入
struct CreateIndexInput {
    TableIOManager &table_io_manager;  // 表 IO 管理
    AttachedDatabase &db;              // 数据库实例
    IndexConstraintType constraint_type;  // 约束类型
    const string &name;                // 索引名称
    const vector<column_t> &column_ids;  // 索引列
    const vector<unique_ptr<Expression>> &unbound_expressions;  // 表达式
    const IndexStorageInfo &storage_info;  // 存储信息
    const case_insensitive_map_t<Value> &options;  // 选项
};

// 计划生成输入
struct PlanIndexInput {
    ClientContext &context;
    LogicalCreateIndex &op;
    PhysicalPlanGenerator &planner;
    PhysicalOperator &table_scan;
};

// 回调函数类型
typedef unique_ptr<BoundIndex> (*index_create_function_t)(CreateIndexInput &input);
typedef PhysicalOperator &(*index_plan_function_t)(PlanIndexInput &input);

// 索引类型定义
class IndexType {
public:
    string name;  // 索引类型名称（如 "ART"）

    // 创建物理计划的回调
    index_plan_function_t create_plan = nullptr;
    // 创建索引实例的回调
    index_create_function_t create_instance = nullptr;
};
```

### 1.5.2 IndexTypeSet 管理

```cpp
// src/execution/index/index_type_set.cpp

class IndexTypeSet {
    mutex lock;
    case_insensitive_map_t<IndexType> functions;

public:
    IndexTypeSet() {
        // 默认注册 ART 索引
        IndexType art_index_type;
        art_index_type.name = ART::TYPE_NAME;
        art_index_type.create_instance = ART::Create;
        art_index_type.create_plan = ART::CreatePlan;
        RegisterIndexType(art_index_type);
    }

    // 按名称查找索引类型
    optional_ptr<IndexType> FindByName(const string &name) {
        lock_guard<mutex> g(lock);
        auto entry = functions.find(name);
        if (entry == functions.end()) {
            return nullptr;
        }
        return &entry->second;
    }

    // 注册新索引类型
    void RegisterIndexType(const IndexType &index_type) {
        lock_guard<mutex> g(lock);
        if (functions.find(index_type.name) != functions.end()) {
            throw CatalogException(
                "Index type with name \"%s\" already exists!",
                index_type.name.c_str());
        }
        functions[index_type.name] = index_type;
    }
};
```

### 1.5.3 扩展索引类型

通过注册机制，扩展可以添加新的索引类型：

```cpp
// 扩展索引注册示例（伪代码）
void RegisterMyIndex(DatabaseInstance &db) {
    IndexType my_index_type;
    my_index_type.name = "MY_INDEX";
    my_index_type.create_instance = MyIndex::Create;
    my_index_type.create_plan = MyIndex::CreatePlan;

    auto &index_set = db.config.GetIndexTypes();
    index_set.RegisterIndexType(my_index_type);
}
```

## 1.6 索引与存储集成

### 1.6.1 TableIndexList

每个表通过 `TableIndexList` 管理其索引：

```cpp
// src/include/duckdb/storage/table/table_index_list.hpp:46-128

class TableIndexList {
private:
    mutex index_entries_lock;
    vector<unique_ptr<IndexEntry>> index_entries;
    idx_t unbound_count = 0;  // 未绑定索引计数

public:
    // 添加索引
    void AddIndex(unique_ptr<Index> index);

    // 移除索引
    void RemoveIndex(const string &name);
    void CommitDrop(const string &name);

    // 查找索引
    optional_ptr<BoundIndex> Find(const string &name);
    bool NameIsUnique(const string &name);

    // 绑定未绑定的索引
    void Bind(ClientContext &context,
              DataTableInfo &table_info,
              const char *index_type = nullptr);

    // 状态查询
    bool Empty();
    idx_t Count();
    bool HasUnbound();

    // 外键支持
    optional_ptr<IndexEntry> FindForeignKeyIndex(
        const vector<PhysicalIndex> &fk_keys,
        const ForeignKeyType fk_type);
    void VerifyForeignKey(...);

    // 序列化
    vector<IndexStorageInfo> SerializeToDisk(QueryContext context,
                                             const IndexSerializationInfo &info);

    // 检查点合并
    void MergeCheckpointDeltas(transaction_t checkpoint_id);
};
```

### 1.6.2 IndexEntry 结构

```cpp
// src/include/duckdb/storage/table/table_index_list.hpp:24-39

enum class IndexBindState : uint8_t {
    UNBOUND,   // 未绑定
    BINDING,   // 正在绑定
    BOUND      // 已绑定
};

struct IndexEntry {
    explicit IndexEntry(unique_ptr<Index> index);

    atomic<IndexBindState> bind_state;  // 原子状态（支持并发绑定）
    mutex lock;                         // 条目锁
    unique_ptr<Index> index;            // 索引本身

    // 事务支持
    unique_ptr<BoundIndex> deleted_rows_in_use;  // 正在使用中的删除行

    // 检查点支持
    unique_ptr<BoundIndex> added_data_during_checkpoint;  // 检查点期间添加的数据
    optional_idx last_written_checkpoint;  // 最后写入的检查点
};
```

### 1.6.3 索引绑定流程

```cpp
// src/storage/table_index_list.cpp:85-170
void TableIndexList::Bind(ClientContext &context,
                          DataTableInfo &table_info,
                          const char *index_type) {
    // 1. 快速检查是否有未绑定索引
    {
        lock_guard<mutex> lock(index_entries_lock);
        if (unbound_count == 0) {
            return;
        }
    }

    // 2. 获取表信息
    auto &catalog = table_info.GetDB().GetCatalog();
    auto &table = catalog.GetEntry<TableCatalogEntry>(...).Cast<DuckTableEntry>();

    vector<LogicalType> column_types;
    vector<string> column_names;
    for (auto &col : table.GetColumns().Logical()) {
        column_types.push_back(col.Type());
        column_names.push_back(col.Name());
    }

    // 3. 循环绑定所有索引
    unique_lock<mutex> lock(index_entries_lock);
    while (true) {
        // 查找下一个未绑定索引
        optional_ptr<IndexEntry> index_entry;
        for (auto &entry : index_entries) {
            if (!entry->index->IsBound() &&
                (index_type == nullptr || entry->index->GetIndexType() == index_type)) {
                index_entry = entry.get();
                break;
            }
        }

        if (!index_entry) {
            // 全部绑定完成
            D_ASSERT(unbound_count == 0);
            break;
        }

        // 状态机处理
        if (index_entry->bind_state == IndexBindState::BINDING) {
            // 其他线程正在绑定，释放锁等待
            lock.unlock();
            lock.lock();
            continue;
        } else if (index_entry->bind_state == IndexBindState::UNBOUND) {
            // 我们来绑定
            index_entry->bind_state = IndexBindState::BINDING;
            lock.unlock();
        }

        // 4. 创建 Binder 并绑定
        auto binder = Binder::CreateBinder(context);
        binder->bind_context.AddBaseTable(0, "", column_names, column_types, ...);

        IndexBinder idx_binder(*binder, context);
        auto &unbound_index = index_entry->index->Cast<UnboundIndex>();
        auto bound_idx = idx_binder.BindIndex(unbound_index);

        // 5. 应用缓冲的 WAL 重放
        if (unbound_index.HasBufferedReplays()) {
            bound_idx->ApplyBufferedReplays(
                physical_column_types,
                unbound_index.GetBufferedReplays(),
                unbound_index.GetMappedColumnIds());
        }

        // 6. 提交绑定结果
        lock.lock();
        index_entry->bind_state = IndexBindState::BOUND;
        index_entry->index = std::move(bound_idx);
        unbound_count--;
    }
}
```

**绑定状态机**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          索引绑定状态机                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│        ┌──────────┐                                                         │
│        │ UNBOUND  │  ← 初始状态（磁盘加载后）                                 │
│        └────┬─────┘                                                         │
│             │ 某线程开始绑定                                                  │
│             ▼                                                                │
│        ┌──────────┐                                                         │
│        │ BINDING  │  ← 正在绑定（其他线程等待）                               │
│        └────┬─────┘                                                         │
│             │ 绑定完成                                                       │
│             ▼                                                                │
│        ┌──────────┐                                                         │
│        │  BOUND   │  ← 绑定完成（可以使用）                                   │
│        └──────────┘                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1.7 目录集成

### 1.7.1 DuckIndexEntry

索引在目录系统中通过 `DuckIndexEntry` 表示：

```cpp
// src/include/duckdb/catalog/catalog_entry/duck_index_entry.hpp:17-52

// 索引与 DataTableInfo 的关联
struct IndexDataTableInfo {
    IndexDataTableInfo(shared_ptr<DataTableInfo> info_p, const string &index_name_p);

    shared_ptr<DataTableInfo> info;  // 指向表信息
    string index_name;               // 析构时要移除的索引名
};

class DuckIndexEntry : public IndexCatalogEntry {
public:
    DuckIndexEntry(Catalog &catalog,
                   SchemaCatalogEntry &schema,
                   CreateIndexInfo &create_info,
                   TableCatalogEntry &table);

    //! 关联的表信息
    shared_ptr<IndexDataTableInfo> info;

    //! CREATE INDEX 后的初始大小（用于自动检查点阈值）
    idx_t initial_index_size;

public:
    string GetSchemaName() const override;
    string GetTableName() const override;

    DataTableInfo &GetDataTableInfo() const;

    //! 复制（用于 ALTER 操作）
    unique_ptr<CatalogEntry> Copy(ClientContext &context) const override;

    //! 回滚
    void Rollback(CatalogEntry &prev_entry) override;

    //! 提交删除
    void CommitDrop();
};
```

### 1.7.2 索引存储信息

```cpp
// src/include/duckdb/storage/index_storage_info.hpp:20-75

// 固定大小分配器信息
struct FixedSizeAllocatorInfo {
    idx_t segment_size;                    // 段大小
    vector<idx_t> buffer_ids;              // 缓冲区 ID
    vector<BlockPointer> block_pointers;   // 磁盘块指针
    vector<idx_t> segment_counts;          // 每个缓冲区的段数
    vector<idx_t> allocation_sizes;        // 分配大小
    vector<idx_t> buffers_with_free_space; // 有空闲空间的缓冲区
};

// 索引存储信息（序列化用）
struct IndexStorageInfo {
    string name;                           // 索引名称
    idx_t root;                            // 根节点位置
    case_insensitive_map_t<Value> options; // 索引选项
    vector<FixedSizeAllocatorInfo> allocator_infos;  // 分配器信息
    vector<vector<IndexBufferInfo>> buffers;  // WAL 缓冲区
    BlockPointer root_block_ptr;           // 根块指针

    bool IsValid() const {
        return root_block_ptr.IsValid() || !allocator_infos.empty();
    }
};

// 索引基本信息
struct IndexInfo {
    bool is_unique;
    bool is_primary;
    bool is_foreign;
    unordered_set<column_t> column_set;
};
```

## 1.8 索引生命周期

### 1.8.1 创建流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         索引创建流程                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CREATE INDEX idx ON t(col)                                                 │
│           │                                                                  │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │     Parser      │  解析 SQL                                               │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │     Binder      │  绑定表和列                                             │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │LogicalCreateIndex│  逻辑计划                                              │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │ IndexType::     │  查找索引类型                                           │
│  │ create_plan     │  生成物理计划                                           │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌──────────────────────────────────────────────┐                           │
│  │        Physical Plan                          │                          │
│  │  TableScan → Projection → [Sort] → CreateART │                          │
│  └────────┬─────────────────────────────────────┘                           │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │   Execution     │  执行计划，填充索引                                      │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │ TableIndexList  │  注册到表                                               │
│  │   AddIndex      │                                                        │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │    Catalog      │  添加目录条目                                            │
│  │ DuckIndexEntry  │                                                        │
│  └─────────────────┘                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.8.2 恢复流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         索引恢复流程                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  数据库启动                                                                   │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────────┐                                                        │
│  │ 读取检查点      │  从磁盘加载索引元数据                                     │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │ 创建 UnboundIndex│  延迟绑定状态                                          │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │   重放 WAL      │  缓冲到 BufferedIndexReplays                            │
│  └────────┬────────┘                                                        │
│           ▼                                                                  │
│  ┌─────────────────┐                                                        │
│  │ 首次使用时绑定   │  转换为 BoundIndex                                      │
│  │                 │  应用缓冲的重放操作                                      │
│  └─────────────────┘                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.8.3 删除流程

```cpp
// 1. 从目录中标记删除
catalog.DropEntry(context, index_name);

// 2. 从表索引列表中移除
table_index_list.RemoveIndex(index_name);

// 3. 提交删除（释放资源）
table_index_list.CommitDrop(index_name);

// CommitDrop 调用链：
// TableIndexList::CommitDrop
//   → Index::CommitDrop
//     → BoundIndex::CommitDrop(IndexLock)
//       → ART 具体实现
```

## 1.9 检查点与事务集成

### 1.9.1 检查点期间的索引处理

```cpp
// src/storage/table_index_list.cpp:266-283
void TableIndexList::MergeCheckpointDeltas(transaction_t checkpoint_id) {
    lock_guard<mutex> lock(index_entries_lock);

    for (auto &entry : index_entries) {
        auto &index = *entry->index;
        if (!index.IsBound()) {
            continue;
        }

        lock_guard<mutex> guard(entry->lock);

        // 合并检查点期间添加的数据
        if (entry->added_data_during_checkpoint) {
            auto &bound_index = index.Cast<BoundIndex>();
            bound_index.MergeIndexes(*entry->added_data_during_checkpoint);
            entry->added_data_during_checkpoint.reset();
        }

        entry->last_written_checkpoint = checkpoint_id;
    }
}
```

### 1.9.2 事务中的索引更新

```cpp
// 检查索引是否受更新影响
bool BoundIndex::IndexIsUpdated(const vector<PhysicalIndex> &column_ids_p) const {
    for (auto &column : column_ids_p) {
        if (column_id_set.find(column.index) != column_id_set.end()) {
            return true;  // 更新涉及索引列
        }
    }
    return false;
}
```

## 1.10 小结

本章分析了 DuckDB 索引系统的整体架构：

1. **三层抽象**：`Index` → `BoundIndex`/`UnboundIndex` → `ART`
2. **延迟绑定**：支持扩展索引和高效恢复
3. **类型注册**：可扩展的索引类型系统
4. **存储集成**：`TableIndexList` 管理表的所有索引
5. **目录集成**：`DuckIndexEntry` 表示目录中的索引
6. **生命周期**：创建、恢复、删除的完整流程

下一章将深入 ART (Adaptive Radix Tree) 的核心实现。

---

## 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/storage/index.hpp` | Index 抽象基类 |
| `src/include/duckdb/execution/index/bound_index.hpp` | BoundIndex 定义 |
| `src/include/duckdb/execution/index/unbound_index.hpp` | UnboundIndex 定义 |
| `src/execution/index/bound_index.cpp` | BoundIndex 实现 |
| `src/include/duckdb/execution/index/index_type.hpp` | 索引类型系统 |
| `src/execution/index/index_type_set.cpp` | 索引类型注册 |
| `src/include/duckdb/storage/table/table_index_list.hpp` | TableIndexList 定义 |
| `src/storage/table_index_list.cpp` | TableIndexList 实现 |
| `src/include/duckdb/common/enums/index_constraint_type.hpp` | 约束类型枚举 |
| `src/include/duckdb/storage/index_storage_info.hpp` | 索引存储信息 |
| `src/include/duckdb/catalog/catalog_entry/duck_index_entry.hpp` | 目录条目 |
