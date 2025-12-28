# DuckDB 存储引擎深度解析 - 第三章：数据组织

## 3.1 概述

DuckDB 采用列式存储架构，数据按照层次化结构组织。理解这一数据组织方式对于深入掌握 DuckDB 的存储机制至关重要。本章将详细剖析从表级别到最底层数据段的完整层次结构。

### 3.1.1 层次化存储架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         DataTable                                       │
│                    (物理表的抽象)                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    RowGroupCollection                            │   │
│  │                   (RowGroup 的集合)                              │   │
│  ├─────────────────────────────────────────────────────────────────┤   │
│  │                                                                   │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │   │
│  │  │   RowGroup 0  │  │   RowGroup 1  │  │   RowGroup N  │        │   │
│  │  │  (122880 rows)│  │  (122880 rows)│  │  (≤122880)    │        │   │
│  │  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘        │   │
│  │          │                  │                  │                 │   │
│  └──────────┼──────────────────┼──────────────────┼─────────────────┘   │
│             │                  │                  │                     │
│             ▼                  ▼                  ▼                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                         RowGroup                                  │  │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │  │
│  │  │ Column 0 │  │ Column 1 │  │ Column N │  │ RowVersionManager│  │  │
│  │  │(ColumnData)│(ColumnData)│(ColumnData)│  │   (版本信息)      │  │  │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────────────────┘  │  │
│  │       │             │             │                               │  │
│  └───────┼─────────────┼─────────────┼───────────────────────────────┘  │
│          │             │             │                                   │
│          ▼             ▼             ▼                                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        ColumnData                                  │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │                   ColumnSegmentTree                          │  │  │
│  │  │  ┌───────────┐  ┌───────────┐  ┌───────────┐               │  │  │
│  │  │  │ Segment 0 │──│ Segment 1 │──│ Segment N │               │  │  │
│  │  │  └───────────┘  └───────────┘  └───────────┘               │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │  ┌───────────────┐  ┌────────────────┐                           │  │
│  │  │ UpdateSegment │  │ValidityColumn  │                           │  │
│  │  │   (更新数据)   │  │  (NULL 掩码)   │                           │  │
│  │  └───────────────┘  └────────────────┘                           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.1.2 核心概念

| 组件 | 描述 | 默认大小/数量 |
|------|------|--------------|
| **DataTable** | 表的物理存储抽象 | 每表一个 |
| **RowGroupCollection** | RowGroup 的有序集合 | 每表一个 |
| **RowGroup** | 基本存储单元，包含固定数量的行 | 122,880 行 |
| **ColumnData** | 单列在 RowGroup 内的数据 | 每列一个 |
| **ColumnSegment** | 列数据的物理存储块 | ≤256KB |
| **RowVersionManager** | MVCC 版本信息管理 | 每 RowGroup 一个 |
| **UpdateSegment** | 未提交的更新数据 | 每列一个（可选） |

---

## 3.2 DataTable - 表级抽象

### 3.2.1 类定义

DataTable 是物理表在存储层的抽象，定义在 `src/include/duckdb/storage/data_table.hpp`：

```cpp
class DataTable : public enable_shared_from_this<DataTable> {
public:
    //! 从持久化数据构造
    DataTable(AttachedDatabase &db, shared_ptr<TableIOManager> table_io_manager,
              const string &schema, const string &table,
              vector<ColumnDefinition> column_definitions_p,
              unique_ptr<PersistentTableData> data = nullptr);

    //! ALTER TABLE 操作的增量构造
    DataTable(ClientContext &context, DataTable &parent,
              ColumnDefinition &new_column, Expression &default_value);
    DataTable(ClientContext &context, DataTable &parent, idx_t removed_column);
    DataTable(ClientContext &context, DataTable &parent, idx_t changed_idx,
              const LogicalType &target_type, ...);

    //! 数据库实例引用
    AttachedDatabase &db;

private:
    //! 表元数据信息
    shared_ptr<DataTableInfo> info;

    //! 列定义
    vector<ColumnDefinition> column_definitions;

    //! 追加操作的互斥锁
    mutex append_lock;

    //! RowGroup 集合
    shared_ptr<RowGroupCollection> row_groups;

    //! 表版本状态
    atomic<DataTableVersion> version;
};
```

### 3.2.2 表版本状态

```cpp
enum class DataTableVersion {
    MAIN_TABLE,  // 最新版本（未被 ALTER 或 DROP）
    ALTERED,     // 已被 ALTER（有新版本）
    DROPPED      // 已被 DROP
};
```

DuckDB 的 ALTER TABLE 采用**写时复制**策略：
1. 创建新的 DataTable 作为增量
2. 旧表标记为 `ALTERED`
3. 新表成为 `MAIN_TABLE`

### 3.2.3 核心操作

**扫描操作**：
```cpp
void DataTable::Scan(DuckTransaction &transaction, DataChunk &result,
                     TableScanState &state) {
    // 委托给 RowGroupCollection 执行扫描
    row_groups->Scan(transaction, result, state);
}

void DataTable::InitializeScan(ClientContext &context, DuckTransaction &transaction,
                                TableScanState &state,
                                const vector<StorageIndex> &column_ids,
                                optional_ptr<TableFilterSet> table_filters) {
    row_groups->InitializeScan(context, state, column_ids, table_filters);
}
```

**并行扫描**：
```cpp
idx_t DataTable::MaxThreads(ClientContext &context) const {
    // 返回 RowGroup 数量作为最大并行度
    return row_groups->GetSegmentCount();
}

void DataTable::InitializeParallelScan(ClientContext &context,
                                        ParallelTableScanState &state) {
    row_groups->InitializeParallelScan(state);
}

bool DataTable::NextParallelScan(ClientContext &context,
                                  ParallelTableScanState &state,
                                  TableScanState &scan_state) {
    return row_groups->NextParallelScan(context, state, scan_state);
}
```

**追加操作**：
```cpp
void DataTable::Append(DataChunk &chunk, TableAppendState &state) {
    // 追加到 RowGroupCollection
    row_groups->Append(chunk, state);
}

void DataTable::CommitAppend(transaction_t commit_id, idx_t row_start, idx_t count) {
    row_groups->CommitAppend(commit_id, row_start, count);
}
```

---

## 3.3 RowGroupCollection - RowGroup 集合管理

### 3.3.1 类定义

RowGroupCollection 管理一个表的所有 RowGroup，定义在 `src/include/duckdb/storage/table/row_group_collection.hpp`：

```cpp
class RowGroupCollection {
public:
    RowGroupCollection(shared_ptr<DataTableInfo> info,
                       TableIOManager &io_manager,
                       vector<LogicalType> types,
                       idx_t row_start, idx_t total_rows = 0);

private:
    //! 块管理器
    BlockManager &block_manager;

    //! 每个 RowGroup 的行数
    const idx_t row_group_size;

    //! 总行数
    atomic<idx_t> total_rows;

    //! 表信息
    shared_ptr<DataTableInfo> info;

    //! 列类型
    vector<LogicalType> types;

    //! RowGroup 段树的访问锁
    mutable mutex row_group_pointer_lock;

    //! RowGroup 段树
    shared_ptr<RowGroupSegmentTree> owned_row_groups;

    //! 表统计信息
    TableStatistics stats;

    //! 内存分配大小
    atomic<idx_t> allocation_size;

    //! 元数据指针（持久化时使用）
    MetaBlockPointer metadata_pointer;

    //! 是否需要新的 RowGroup
    bool requires_new_row_group;
};
```

### 3.3.2 RowGroup 大小配置

```cpp
// 默认 RowGroup 大小：122,880 行
#define DEFAULT_ROW_GROUP_SIZE 122880ULL

// 计算逻辑：
// - Vector 大小：2048
// - RowGroup 包含 60 个 Vector
// - 122880 = 2048 * 60
```

这个大小的选择是经过精心设计的：
1. **缓存友好**：适合 L2/L3 缓存
2. **向量化高效**：是 SIMD 向量大小的整数倍
3. **并行性好**：提供足够的并行粒度

### 3.3.3 扫描初始化

```cpp
void RowGroupCollection::InitializeScan(const QueryContext &context,
                                         CollectionScanState &state,
                                         const vector<StorageIndex> &column_ids,
                                         optional_ptr<TableFilterSet> table_filters) {
    auto row_groups = GetRowGroups();
    auto l = row_groups->Lock();

    state.Initialize(context, column_ids, types);
    state.column_ids = column_ids;
    state.table_filters = table_filters;

    // 获取第一个 RowGroup
    auto first_segment = row_groups->GetRootSegment(l);
    if (first_segment) {
        // 初始化第一个 RowGroup 的扫描
        first_segment->GetNode().InitializeScan(state, *first_segment);
    }
}
```

### 3.3.4 并行扫描协调

```cpp
bool RowGroupCollection::NextParallelScan(ClientContext &context,
                                           ParallelCollectionScanState &state,
                                           CollectionScanState &scan_state) {
    auto row_groups = GetRowGroups();

    while (true) {
        optional_ptr<SegmentNode<RowGroup>> next_row_group;
        idx_t vector_index = 0;
        idx_t max_row = 0;

        {
            // 原子获取下一个扫描任务
            lock_guard<mutex> l(state.lock);
            if (state.current_row_group) {
                // 在当前 RowGroup 内继续
                auto &row_group = state.current_row_group->GetNode();
                vector_index = state.vector_index;

                // 更新状态
                state.vector_index++;
                if (state.vector_index * STANDARD_VECTOR_SIZE >= row_group.count) {
                    // 当前 RowGroup 扫描完毕，移动到下一个
                    state.current_row_group =
                        row_groups->GetNextSegment(*state.current_row_group);
                    state.vector_index = 0;
                }
            }

            if (!state.current_row_group) {
                return false;  // 所有 RowGroup 已扫描完毕
            }

            next_row_group = state.current_row_group;
            max_row = state.current_row_group->GetRowEnd();
        }

        // 初始化扫描状态
        if (!InitializeScanInRowGroup(context, scan_state, *this,
                                       *next_row_group, vector_index, max_row)) {
            continue;  // RowGroup 被跳过，尝试下一个
        }

        return true;
    }
}
```

### 3.3.5 追加操作

```cpp
bool RowGroupCollection::Append(DataChunk &chunk, TableAppendState &state) {
    bool new_row_group = false;

    // 检查是否需要新的 RowGroup
    if (state.remaining == 0 || requires_new_row_group) {
        auto l = state.row_groups->Lock();
        AppendRowGroup(l, state.row_start);
        requires_new_row_group = false;
        new_row_group = true;
    }

    // 获取当前 RowGroup
    auto &current_row_group = state.row_groups->GetLastSegment(l)->GetNode();

    // 追加数据
    current_row_group.Append(state.current_row_group, chunk, chunk.size());

    // 更新状态
    state.row_start += chunk.size();
    total_rows += chunk.size();

    return new_row_group;
}

void RowGroupCollection::AppendRowGroup(SegmentLock &l, idx_t start_row) {
    auto new_row_group = make_shared_ptr<RowGroup>(*this, start_row);
    new_row_group->InitializeEmpty(types, ColumnDataType::MAIN_TABLE);

    auto row_groups = GetRowGroups();
    row_groups->AppendSegment(l, std::move(new_row_group));
}
```

---

## 3.4 RowGroup - 基本存储单元

### 3.4.1 类定义

RowGroup 是 DuckDB 的基本存储单元，定义在 `src/include/duckdb/storage/table/row_group.hpp`：

```cpp
class RowGroup : public SegmentBase<RowGroup> {
public:
    friend class ColumnData;

    RowGroup(RowGroupCollection &collection, idx_t count);
    RowGroup(RowGroupCollection &collection, RowGroupPointer pointer);
    RowGroup(RowGroupCollection &collection, PersistentRowGroupData &data);

private:
    //! 所属的 RowGroupCollection
    reference<RowGroupCollection> collection;

    //! MVCC 版本信息管理器
    atomic<optional_ptr<RowVersionManager>> version_info;
    shared_ptr<RowVersionManager> owned_version_info;

    //! 列数据（可延迟加载）
    mutable vector<shared_ptr<ColumnData>> columns;

    //! RowGroup 级别的锁
    mutable mutex row_group_lock;

    //! 列元数据指针
    vector<MetaBlockPointer> column_pointers;

    //! 列是否已加载
    mutable unique_ptr<atomic<bool>[]> is_loaded;

    //! 删除信息指针
    vector<MetaBlockPointer> deletes_pointers;

    //! 删除信息是否已加载
    atomic<bool> deletes_is_loaded;

    //! 内存分配大小
    atomic<idx_t> allocation_size;

    //! Row ID 列数据（用于并行扫描）
    mutable unique_ptr<ColumnData> row_id_column_data;
    mutable atomic<bool> row_id_is_loaded;

    //! 是否有未提交的更改
    atomic<bool> has_changes;
};
```

### 3.4.2 延迟加载机制

RowGroup 支持列的延迟加载：

```cpp
ColumnData &RowGroup::GetColumn(storage_t c) const {
    // 检查列是否已加载
    if (!is_loaded[c].load(std::memory_order_acquire)) {
        LoadColumn(c);
    }
    return *columns[c];
}

void RowGroup::LoadColumn(storage_t c) const {
    lock_guard<mutex> lock(row_group_lock);

    // 双重检查
    if (is_loaded[c].load(std::memory_order_acquire)) {
        return;
    }

    // 从磁盘加载列数据
    auto &block_manager = GetBlockManager();
    auto &info = GetTableInfo();
    auto &type = collection.get().GetTypes()[c];

    // 反序列化列数据
    auto column_data = ColumnData::Deserialize(
        block_manager, info, c, column_pointers[c], type);

    columns[c] = std::move(column_data);
    is_loaded[c].store(true, std::memory_order_release);
}
```

### 3.4.3 扫描实现

```cpp
void RowGroup::Scan(TransactionData transaction, CollectionScanState &state,
                    DataChunk &result) {
    // 模板化扫描实现
    TemplatedScan<TableScanType::TABLE_SCAN_REGULAR>(transaction, state, result);
}

template <TableScanType TYPE>
void RowGroup::TemplatedScan(TransactionData transaction, CollectionScanState &state,
                              DataChunk &result) {
    const auto max_count = MinValue<idx_t>(
        STANDARD_VECTOR_SIZE,
        count - state.vector_index * STANDARD_VECTOR_SIZE);

    idx_t current_count;

    if constexpr (TYPE == TableScanType::TABLE_SCAN_REGULAR) {
        // 获取可见行的选择向量
        current_count = GetSelVector(transaction, state.vector_index,
                                      state.row_group_state.row_validity,
                                      max_count);
    } else {
        // COMMITTED 扫描
        current_count = GetCommittedSelVector(...);
    }

    if (current_count == 0) {
        // 所有行都被删除
        return;
    }

    // 扫描每一列
    for (idx_t i = 0; i < state.column_ids.size(); i++) {
        auto &column = GetColumn(state.column_ids[i]);
        if (current_count == max_count) {
            // 全量扫描
            column.Scan(transaction, state.vector_index,
                        state.column_scans[i], result.data[i], current_count);
        } else {
            // 带选择向量的扫描
            column.Select(transaction, state.vector_index,
                          state.column_scans[i], result.data[i],
                          state.row_group_state.row_validity, current_count);
        }
    }

    result.SetCardinality(current_count);
}
```

### 3.4.4 版本信息获取

```cpp
idx_t RowGroup::GetSelVector(TransactionData transaction, idx_t vector_idx,
                              SelectionVector &sel_vector, idx_t max_count) {
    auto vinfo = GetVersionInfo();
    if (!vinfo) {
        // 没有版本信息，所有行都可见
        return max_count;
    }

    return vinfo->GetSelVector(transaction, vector_idx, sel_vector, max_count);
}

optional_ptr<RowVersionManager> RowGroup::GetVersionInfo() {
    auto vinfo = version_info.load(std::memory_order_acquire);
    if (vinfo) {
        return vinfo;
    }

    // 检查是否有未加载的删除信息
    if (!deletes_is_loaded.load() && !deletes_pointers.empty()) {
        LoadDeletes();
    }

    return version_info.load(std::memory_order_acquire);
}
```

### 3.4.5 Zonemap 过滤

RowGroup 支持使用统计信息进行过滤：

```cpp
bool RowGroup::CheckZonemap(ScanFilterInfo &filters) {
    // 遍历所有过滤条件
    for (auto &filter : filters.GetFilters()) {
        auto column_index = filter.GetColumnIndex();
        auto &column = GetColumn(column_index);

        // 检查列统计信息
        auto result = column.CheckZonemap(filter.GetFilter());

        if (result == FilterPropagateResult::FILTER_ALWAYS_FALSE) {
            // 整个 RowGroup 可以跳过
            return false;
        }
    }

    return true;
}

bool RowGroup::CheckZonemapSegments(CollectionScanState &state) {
    // 更细粒度的段级别过滤
    for (auto &filter : state.filters.GetFilters()) {
        auto column_index = filter.GetColumnIndex();
        auto &column = GetColumn(column_index);

        auto result = column.CheckZonemap(state.column_scans[column_index],
                                           filter.GetFilter());

        if (result == FilterPropagateResult::FILTER_ALWAYS_FALSE) {
            return false;
        }
    }

    return true;
}
```

---

## 3.5 ColumnData - 列数据管理

### 3.5.1 类层次结构

```
                    ColumnData (抽象基类)
                         │
         ┌───────────────┼───────────────────────┐
         │               │                       │
 StandardColumnData  ValidityColumnData   StructColumnData
         │                                       │
         │                               ┌───────┴───────┐
         │                               │               │
 ┌───────┴───────┐                ListColumnData  ArrayColumnData
 │               │
 │      子列(Validity)
 │
```

### 3.5.2 基类定义

```cpp
class ColumnData : public enable_shared_from_this<ColumnData> {
public:
    ColumnData(BlockManager &block_manager, DataTableInfo &info, idx_t column_index,
               LogicalType type, ColumnDataType data_type,
               optional_ptr<ColumnData> parent);

    //! 行数
    atomic<idx_t> count;

    //! 块管理器
    BlockManager &block_manager;

    //! 表信息
    DataTableInfo &info;

    //! 列索引
    idx_t column_index;

    //! 列类型
    LogicalType type;

protected:
    //! 段树（存储 ColumnSegment）
    ColumnSegmentTree data;

    //! 更新数据的锁
    mutable mutex update_lock;

    //! 更新段（未提交的更新）
    unique_ptr<UpdateSegment> updates;

    //! 统计信息的锁
    mutable mutex stats_lock;

    //! 段统计信息
    unique_ptr<SegmentStatistics> stats;

    //! 内存分配大小
    atomic<idx_t> allocation_size;

private:
    //! 数据类型（主表/事务本地）
    atomic<ColumnDataType> data_type;

    //! 父列（用于嵌套类型）
    optional_ptr<ColumnData> parent;

    //! 压缩函数
    atomic_ptr<const CompressionFunction> compression;
};
```

### 3.5.3 ColumnDataType 枚举

```cpp
enum class ColumnDataType {
    MAIN_TABLE,                    // 主表数据
    INITIAL_TRANSACTION_LOCAL,     // 初始事务本地数据
    TRANSACTION_LOCAL,             // 事务本地数据
    CHECKPOINT_TARGET              // Checkpoint 目标
};
```

### 3.5.4 StandardColumnData

StandardColumnData 是最常用的列数据类型，包含一个 ValidityColumnData 子列：

```cpp
class StandardColumnData : public ColumnData {
public:
    StandardColumnData(BlockManager &block_manager, DataTableInfo &info,
                       idx_t column_index, LogicalType type,
                       ColumnDataType data_type, optional_ptr<ColumnData> parent);

protected:
    //! NULL 值掩码
    shared_ptr<ValidityColumnData> validity;

public:
    // 扫描时同时处理数据和 validity
    idx_t Scan(TransactionData transaction, idx_t vector_index,
               ColumnScanState &state, Vector &result, idx_t target_count) override {
        // 扫描主数据
        auto scan_count = ScanVector(transaction, vector_index, state, result,
                                      target_count, ScanVectorMode::SCAN_ENTIRE_VECTOR);

        // 扫描 validity
        validity->ScanVector(transaction, vector_index, state.child_states[0],
                             result, scan_count, ScanVectorMode::SCAN_ENTIRE_VECTOR);

        return scan_count;
    }
};
```

### 3.5.5 列扫描流程

```cpp
idx_t ColumnData::ScanVector(TransactionData transaction, idx_t vector_index,
                              ColumnScanState &state, Vector &result,
                              idx_t target_scan, ScanVectorType scan_type,
                              ScanVectorMode mode) {

    // 1. 从段树获取基础数据
    auto scan_count = ScanVector(state, result, target_scan, scan_type);

    // 2. 合并更新数据
    if (updates && updates->HasUpdates(vector_index)) {
        FetchUpdates(transaction, vector_index, result, scan_count,
                     mode == ScanVectorMode::ALLOW_UPDATES, false);
    }

    return scan_count;
}

idx_t ColumnData::ScanVector(ColumnScanState &state, Vector &result,
                              idx_t remaining, ScanVectorType scan_type,
                              idx_t result_offset) {
    idx_t scanned = 0;

    while (remaining > 0 && state.current) {
        auto &segment = state.current->GetNode();

        // 计算当前段可扫描的行数
        idx_t scan_count = MinValue<idx_t>(
            remaining, segment.count - state.row_index);

        // 执行段扫描
        segment.Scan(state, scan_count, result, result_offset, scan_type);

        // 更新状态
        state.row_index += scan_count;
        result_offset += scan_count;
        remaining -= scan_count;
        scanned += scan_count;

        // 移动到下一个段
        if (state.row_index >= segment.count) {
            state.current = data.GetNextSegment(*state.current);
            state.row_index = 0;
            if (state.current) {
                state.current->GetNode().InitializeScan(state);
            }
        }
    }

    return scanned;
}
```

### 3.5.6 列追加流程

```cpp
void ColumnData::AppendData(BaseStatistics &stats, ColumnAppendState &state,
                             UnifiedVectorFormat &vdata, idx_t count) {
    idx_t offset = 0;

    while (count > 0) {
        // 获取当前段
        auto &segment = *state.current;

        // 追加到当前段
        idx_t appended = segment.Append(state, vdata, offset, count);

        offset += appended;
        count -= appended;

        // 段已满，创建新段
        if (count > 0) {
            auto l = data.Lock();
            AppendTransientSegment(l, state.current->start + state.current->count);
            state.current = &data.GetLastSegment(l)->GetNode();
            state.current->InitializeAppend(state);
        }
    }
}

void ColumnData::AppendTransientSegment(SegmentLock &l, idx_t start_row) {
    // 创建新的临时段
    auto &db = GetDatabase();
    auto &function = *GetCompressionFunction();

    auto segment = ColumnSegment::CreateTransientSegment(
        db, function, type, Storage::BLOCK_SIZE, block_manager);

    segment->start = start_row;
    data.AppendSegment(l, std::move(segment));
}
```

---

## 3.6 ColumnSegment - 列数据段

### 3.6.1 类定义

ColumnSegment 是列数据的物理存储单元：

```cpp
class ColumnSegment : public SegmentBase<ColumnSegment> {
public:
    //! 创建持久化段
    static unique_ptr<ColumnSegment> CreatePersistentSegment(
        DatabaseInstance &db, BlockManager &block_manager,
        block_id_t id, idx_t offset, const LogicalType &type_p,
        idx_t count, CompressionType compression_type,
        BaseStatistics statistics,
        unique_ptr<ColumnSegmentState> segment_state);

    //! 创建临时段
    static unique_ptr<ColumnSegment> CreateTransientSegment(
        DatabaseInstance &db, CompressionFunction &function,
        const LogicalType &type, idx_t segment_size,
        BlockManager &block_manager);

public:
    //! 数据库实例
    DatabaseInstance &db;

    //! 列类型
    LogicalType type;

    //! 类型大小
    idx_t type_size;

    //! 段类型（持久化/临时）
    ColumnSegmentType segment_type;

    //! 段统计信息
    SegmentStatistics stats;

    //! 存储块句柄
    shared_ptr<BlockHandle> block;

private:
    //! 压缩函数
    reference<CompressionFunction> function;

    //! 块 ID（持久化段）
    block_id_t block_id;

    //! 块内偏移（持久化段）
    idx_t offset;

    //! 段大小
    idx_t segment_size;

    //! 压缩状态
    unique_ptr<CompressedSegmentState> segment_state;
};
```

### 3.6.2 段类型

```cpp
enum class ColumnSegmentType : uint8_t {
    TRANSIENT,   // 临时段（内存中，未压缩）
    PERSISTENT   // 持久化段（磁盘上，可能压缩）
};
```

### 3.6.3 扫描实现

```cpp
void ColumnSegment::Scan(ColumnScanState &state, idx_t scan_count,
                          Vector &result, idx_t result_offset,
                          ScanVectorType scan_type) {
    if (scan_type == ScanVectorType::SCAN_ENTIRE_VECTOR &&
        result_offset == 0) {
        // 完整扫描
        Scan(state, scan_count, result);
    } else {
        // 部分扫描
        ScanPartial(state, scan_count, result, result_offset);
    }
}

void ColumnSegment::Scan(ColumnScanState &state, idx_t scan_count, Vector &result) {
    // 调用压缩函数的扫描方法
    function.get().scan_vector(state, scan_count, result);
}

void ColumnSegment::ScanPartial(ColumnScanState &state, idx_t scan_count,
                                 Vector &result, idx_t result_offset) {
    // 调用压缩函数的部分扫描方法
    function.get().scan_partial(state, scan_count, result, result_offset);
}
```

### 3.6.4 追加实现

```cpp
idx_t ColumnSegment::Append(ColumnAppendState &state, UnifiedVectorFormat &data,
                             idx_t offset, idx_t count) {
    D_ASSERT(segment_type == ColumnSegmentType::TRANSIENT);

    // 调用压缩函数的追加方法
    idx_t appended = function.get().append(state, data, offset, count);

    this->count += appended;
    return appended;
}

idx_t ColumnSegment::FinalizeAppend(ColumnAppendState &state) {
    D_ASSERT(segment_type == ColumnSegmentType::TRANSIENT);

    // 完成追加，可能进行压缩
    auto &append_state = state.append_state->Cast<CompressedAppendState>();
    return function.get().finalize_append(append_state, *this);
}
```

### 3.6.5 持久化转换

```cpp
void ColumnSegment::ConvertToPersistent(QueryContext context,
                                         optional_ptr<BlockManager> block_manager,
                                         block_id_t new_block_id) {
    D_ASSERT(segment_type == ColumnSegmentType::TRANSIENT);

    if (block_manager) {
        // 写入到新块
        auto new_block = block_manager->RegisterBlock(new_block_id);
        auto handle = block_manager->Pin(new_block);

        // 复制数据
        memcpy(handle.Ptr(), block->buffer->buffer, segment_size);

        // 更新块引用
        block = std::move(new_block);
    }

    segment_type = ColumnSegmentType::PERSISTENT;
    block_id = new_block_id;
}
```

---

## 3.7 SegmentTree - 段树管理

### 3.7.1 SegmentNode 结构

```cpp
template <class T>
struct SegmentNode {
    SegmentNode(idx_t row_start_p, shared_ptr<T> node_p, idx_t index_p)
        : row_start(row_start_p), node(std::move(node_p)),
          next(nullptr), index(index_p) {}

public:
    idx_t GetRowStart() const { return row_start; }
    idx_t GetRowEnd() const { return GetRowStart() + GetCount(); }
    idx_t GetCount() const { return GetNode().count; }
    idx_t GetIndex() const { return index; }
    T &GetNode() const { return *node; }

private:
    idx_t row_start;
    shared_ptr<T> node;
    atomic<SegmentNode<T> *> next;  // 支持无锁遍历
    idx_t index;
};
```

### 3.7.2 SegmentTree 类

```cpp
template <class T, bool SUPPORTS_LAZY_LOADING = false>
class SegmentTree {
public:
    explicit SegmentTree(idx_t base_row_id = 0)
        : finished_loading(true), base_row_id(base_row_id) {}

    //! 加锁
    SegmentLock Lock() const {
        return SegmentLock(node_lock);
    }

    //! 获取根段
    optional_ptr<SegmentNode<T>> GetRootSegment(SegmentLock &l) const {
        if (nodes.empty()) {
            LoadNextSegment(l);  // 延迟加载
        }
        return GetRootSegmentInternal();
    }

    //! 通过行号查找段（二分搜索）
    optional_ptr<SegmentNode<T>> GetSegment(SegmentLock &l, idx_t row_number) const {
        return nodes[GetSegmentIndex(l, row_number)].get();
    }

    //! 追加段
    void AppendSegment(SegmentLock &l, shared_ptr<T> segment) {
        LoadAllSegments(l);
        AppendSegmentInternal(l, std::move(segment));
    }

private:
    //! 节点列表
    mutable vector<unique_ptr<SegmentNode<T>>> nodes;

    //! 节点锁
    mutable mutex node_lock;

    //! 基础行 ID
    idx_t base_row_id;

    //! 是否加载完成
    mutable atomic<bool> finished_loading;
};
```

### 3.7.3 二分搜索查找

```cpp
idx_t SegmentTree::GetSegmentIndex(SegmentLock &l, idx_t row_number) const {
    // 确保段已加载
    while (nodes.empty() || row_number >= nodes.back()->GetRowEnd()) {
        if (!LoadNextSegment(l)) {
            break;
        }
    }

    // 二分搜索
    idx_t lower = 0;
    idx_t upper = nodes.size() - 1;

    while (lower <= upper) {
        idx_t index = (lower + upper) / 2;
        auto &entry = *nodes[index];

        if (row_number < entry.GetRowStart()) {
            upper = index - 1;
        } else if (row_number >= entry.GetRowEnd()) {
            lower = index + 1;
        } else {
            return index;  // 找到
        }
    }

    throw InternalException("Could not find segment for row %llu", row_number);
}
```

---

## 3.8 RowVersionManager - MVCC 版本管理

### 3.8.1 类定义

```cpp
class RowVersionManager {
public:
    explicit RowVersionManager(BufferManager &buffer_manager) noexcept;

    //! 获取可见行的选择向量
    idx_t GetSelVector(TransactionData transaction, idx_t vector_idx,
                       SelectionVector &sel_vector, idx_t max_count);

    //! 获取已提交行的选择向量
    idx_t GetCommittedSelVector(transaction_t start_time, transaction_t transaction_id,
                                 idx_t vector_idx, SelectionVector &sel_vector,
                                 idx_t max_count);

    //! 追加版本信息
    void AppendVersionInfo(TransactionData transaction, idx_t count,
                           idx_t row_group_start, idx_t row_group_end);

    //! 提交追加
    void CommitAppend(transaction_t commit_id, idx_t row_group_start, idx_t count);

    //! 删除行
    idx_t DeleteRows(idx_t vector_idx, transaction_t transaction_id,
                     row_t rows[], idx_t count);

    //! 提交删除
    void CommitDelete(idx_t vector_idx, transaction_t commit_id, const DeleteInfo &info);

private:
    mutex version_lock;

    //! 固定大小分配器（用于版本信息）
    FixedSizeAllocator allocator;

    //! 每个向量的版本信息
    vector<unique_ptr<ChunkInfo>> vector_info;

    //! 是否有未序列化的更改
    bool has_unserialized_changes;

    //! 存储指针
    vector<MetaBlockPointer> storage_pointers;
};
```

### 3.8.2 ChunkInfo 层次结构

```cpp
enum class ChunkInfoType : uint8_t {
    CONSTANT_INFO,  // 常量版本信息（整个 chunk 相同）
    VECTOR_INFO,    // 向量版本信息（每行不同）
    EMPTY_INFO      // 空信息
};

// 常量版本信息 - 整个 chunk 有相同的插入/删除 ID
class ChunkConstantInfo : public ChunkInfo {
public:
    transaction_t insert_id;  // 插入事务 ID
    transaction_t delete_id;  // 删除事务 ID
};

// 向量版本信息 - 每行有独立的版本信息
class ChunkVectorInfo : public ChunkInfo {
private:
    FixedSizeAllocator &allocator;

    //! 插入事务 ID（每行）
    IndexPointer inserted_data;

    //! 常量插入 ID（如果所有行相同）
    transaction_t constant_insert_id;

    //! 删除事务 ID（每行）
    IndexPointer deleted_data;
};
```

### 3.8.3 可见性判断

```cpp
idx_t ChunkVectorInfo::GetSelVector(transaction_t start_time,
                                     transaction_t transaction_id,
                                     SelectionVector &sel_vector,
                                     idx_t max_count) const {
    idx_t count = 0;

    auto inserted = GetInsertedPointer();
    auto deleted = GetDeletedPointer();

    for (idx_t i = 0; i < max_count; i++) {
        // 获取插入和删除事务 ID
        transaction_t insert_id = GetInsertId(inserted, i);
        transaction_t delete_id = GetDeleteId(deleted, i);

        // 判断可见性
        bool visible = IsVisible(insert_id, delete_id, start_time, transaction_id);

        if (visible) {
            sel_vector.set_index(count++, i);
        }
    }

    return count;
}

bool IsVisible(transaction_t insert_id, transaction_t delete_id,
               transaction_t start_time, transaction_t transaction_id) {
    // 检查插入是否可见
    if (insert_id >= TRANSACTION_ID_START) {
        // 未提交的插入
        if (insert_id != transaction_id) {
            return false;  // 其他事务的未提交插入
        }
    } else {
        // 已提交的插入
        if (insert_id > start_time) {
            return false;  // 在本事务开始后提交
        }
    }

    // 检查删除
    if (delete_id == NOT_DELETED_ID) {
        return true;  // 未删除
    }

    if (delete_id >= TRANSACTION_ID_START) {
        // 未提交的删除
        return delete_id != transaction_id;  // 本事务删除则不可见
    } else {
        // 已提交的删除
        return delete_id > start_time;  // 在本事务开始后删除则仍可见
    }
}
```

---

## 3.9 UpdateSegment - 更新数据管理

### 3.9.1 类定义

UpdateSegment 管理列的未提交更新：

```cpp
class UpdateSegment {
public:
    explicit UpdateSegment(ColumnData &column_data);

    ColumnData &column_data;

public:
    //! 检查是否有更新
    bool HasUpdates() const;
    bool HasUpdates(idx_t vector_index) const;
    bool HasUncommittedUpdates(idx_t vector_index);

    //! 获取更新数据
    void FetchUpdates(TransactionData transaction, idx_t vector_index, Vector &result);
    void FetchCommitted(idx_t vector_index, Vector &result);
    void FetchRow(TransactionData transaction, idx_t row_id, Vector &result,
                  idx_t result_idx);

    //! 执行更新
    void Update(TransactionData transaction, DataTable &data_table, idx_t column_index,
                Vector &update, row_t *ids, idx_t count, Vector &base_data,
                idx_t row_group_start);

    //! 回滚更新
    void RollbackUpdate(UpdateInfo &info);

    //! 清理更新
    void CleanupUpdate(UpdateInfo &info);

private:
    //! 更新段锁
    mutable StorageLock lock;

    //! 更新树根节点
    unique_ptr<UpdateNode> root;

    //! 更新统计信息
    SegmentStatistics stats;

    //! 字符串堆（用于字符串类型）
    StringHeap heap;

    //! 类型特定的函数指针
    initialize_update_function_t initialize_update_function;
    merge_update_function_t merge_update_function;
    fetch_update_function_t fetch_update_function;
    // ... 其他函数指针
};
```

### 3.9.2 UpdateNode 结构

```cpp
struct UpdateNode {
    explicit UpdateNode(BufferManager &manager);
    ~UpdateNode();

    //! Undo 缓冲区分配器
    UndoBufferAllocator allocator;

    //! 每个向量的更新信息指针
    vector<UndoBufferPointer> info;
};
```

### 3.9.3 更新流程

```cpp
void UpdateSegment::Update(TransactionData transaction, DataTable &data_table,
                            idx_t column_index, Vector &update, row_t *ids,
                            idx_t count, Vector &base_data, idx_t row_group_start) {
    // 1. 获取写锁
    auto lock_key = lock.GetExclusiveLock();

    // 2. 准备更新数据
    UnifiedVectorFormat update_format;
    update.ToUnifiedFormat(count, update_format);

    // 3. 获取有效更新（过滤掉相同值）
    SelectionVector sel(count);
    idx_t effective_count = get_effective_updates(update_format, ids, count,
                                                   sel, base_data, row_group_start);

    if (effective_count == 0) {
        return;  // 无有效更新
    }

    // 4. 更新统计信息
    {
        lock_guard<mutex> stats_guard(stats_lock);
        statistics_update_function(this, stats, update_format, effective_count, sel);
    }

    // 5. 初始化或合并更新
    for (idx_t i = 0; i < effective_count; i++) {
        idx_t vector_idx = (ids[sel.get_index(i)] - row_group_start) / STANDARD_VECTOR_SIZE;

        auto update_ptr = GetUpdateNode(*lock_key, vector_idx);

        if (!update_ptr.IsSet()) {
            // 初始化新的更新信息
            InitializeUpdateInfo(vector_idx);
            update_ptr = GetUpdateNode(*lock_key, vector_idx);

            initialize_update_function(...);
        } else {
            // 合并到现有更新
            merge_update_function(...);
        }
    }
}
```

---

## 3.10 复杂类型支持

### 3.10.1 StructColumnData

```cpp
class StructColumnData : public ColumnData {
public:
    StructColumnData(BlockManager &block_manager, DataTableInfo &info,
                     idx_t column_index, const LogicalType &type,
                     ColumnDataType data_type, optional_ptr<ColumnData> parent);

private:
    //! 有效性列
    shared_ptr<ValidityColumnData> validity;

    //! 子列
    vector<shared_ptr<ColumnData>> sub_columns;
};
```

### 3.10.2 ListColumnData

```cpp
class ListColumnData : public ColumnData {
public:
    ListColumnData(BlockManager &block_manager, DataTableInfo &info,
                   idx_t column_index, const LogicalType &type,
                   ColumnDataType data_type, optional_ptr<ColumnData> parent);

private:
    //! 有效性列
    shared_ptr<ValidityColumnData> validity;

    //! 子列数据
    shared_ptr<ColumnData> child_column;
};
```

### 3.10.3 列类型创建工厂

```cpp
shared_ptr<ColumnData> ColumnData::CreateColumn(BlockManager &block_manager,
                                                  DataTableInfo &info,
                                                  idx_t column_index,
                                                  const LogicalType &type,
                                                  ColumnDataType data_type,
                                                  optional_ptr<ColumnData> parent) {
    switch (type.InternalType()) {
    case PhysicalType::STRUCT:
        return make_shared_ptr<StructColumnData>(block_manager, info, column_index,
                                                  type, data_type, parent);
    case PhysicalType::LIST:
        return make_shared_ptr<ListColumnData>(block_manager, info, column_index,
                                                type, data_type, parent);
    case PhysicalType::ARRAY:
        return make_shared_ptr<ArrayColumnData>(block_manager, info, column_index,
                                                 type, data_type, parent);
    default:
        return make_shared_ptr<StandardColumnData>(block_manager, info, column_index,
                                                    type, data_type, parent);
    }
}
```

---

## 3.11 持久化数据结构

### 3.11.1 PersistentColumnData

```cpp
struct PersistentColumnData {
    PhysicalType physical_type;
    LogicalTypeId logical_type_id;

    //! 数据指针列表
    vector<DataPointer> pointers;

    //! 子列数据
    vector<PersistentColumnData> child_columns;

    //! 是否有未提交更新
    bool has_updates = false;

    void Serialize(Serializer &serializer) const;
    static PersistentColumnData Deserialize(Deserializer &deserializer);
};
```

### 3.11.2 PersistentRowGroupData

```cpp
struct PersistentRowGroupData {
    vector<LogicalType> types;
    vector<PersistentColumnData> column_data;
    idx_t start;
    idx_t count;

    void Serialize(Serializer &serializer) const;
    static PersistentRowGroupData Deserialize(Deserializer &deserializer);
};
```

### 3.11.3 DataPointer

```cpp
struct DataPointer {
    //! 段统计信息
    BaseStatistics statistics;

    //! 行数
    uint64_t row_count;

    //! 压缩类型
    CompressionType compression_type;

    //! 块位置
    BlockPointer block_pointer;
};
```

---

## 3.12 小结

DuckDB 的数据组织采用清晰的层次化结构：

### 核心设计原则

1. **列式存储**
   - 同一列的数据连续存储
   - 便于压缩和向量化处理
   - 只读取查询需要的列

2. **分块存储**
   - RowGroup 作为基本存储单元
   - 122,880 行/RowGroup 的大小设计
   - 支持并行扫描和 Zonemap 过滤

3. **延迟加载**
   - 列数据按需加载
   - 段树支持延迟加载
   - 减少内存占用

4. **MVCC 支持**
   - RowVersionManager 管理版本信息
   - ChunkInfo 追踪插入/删除
   - UpdateSegment 管理更新

5. **嵌套类型支持**
   - StructColumnData 支持结构体
   - ListColumnData 支持列表
   - 递归的子列结构

### 关键源文件

| 组件 | 头文件 | 实现文件 |
|------|--------|----------|
| DataTable | `storage/data_table.hpp` | `storage/data_table.cpp` |
| RowGroupCollection | `storage/table/row_group_collection.hpp` | `storage/table/row_group_collection.cpp` |
| RowGroup | `storage/table/row_group.hpp` | `storage/table/row_group.cpp` |
| ColumnData | `storage/table/column_data.hpp` | `storage/table/column_data.cpp` |
| StandardColumnData | `storage/table/standard_column_data.hpp` | `storage/table/standard_column_data.cpp` |
| ColumnSegment | `storage/table/column_segment.hpp` | `storage/table/column_segment.cpp` |
| SegmentTree | `storage/table/segment_tree.hpp` | - |
| RowVersionManager | `storage/table/row_version_manager.hpp` | `storage/table/row_version_manager.cpp` |
| UpdateSegment | `storage/table/update_segment.hpp` | `storage/table/update_segment.cpp` |
| ChunkInfo | `storage/table/chunk_info.hpp` | `storage/table/chunk_info.cpp` |

### 数据流示例

```
写入流程：
DataTable::Append()
    → RowGroupCollection::Append()
        → RowGroup::Append()
            → ColumnData::AppendData()
                → ColumnSegment::Append()

扫描流程：
DataTable::Scan()
    → RowGroupCollection::Scan()
        → RowGroup::Scan()
            → RowVersionManager::GetSelVector()  (过滤已删除行)
            → ColumnData::Scan()
                → ColumnSegment::Scan()
                → UpdateSegment::FetchUpdates()  (合并更新)
```

---

下一章，我们将深入探讨 DuckDB 的压缩算法，包括各种压缩函数的实现原理和选择策略。
