# DuckDB 执行引擎深度解析（六）：哈希表与内存管理

## 引言

在前五章中，我们深入分析了 DuckDB 的执行模型、向量化数据结构、表达式执行器、物理算子和 Pipeline 并行执行。本章将聚焦于 **哈希表与内存管理** 这两个关键主题。

哈希表是数据库中最重要的数据结构之一，广泛用于 Hash Join、Hash Aggregate 等操作。而内存管理直接影响查询的性能和稳定性。DuckDB 在这两个方面都采用了精心设计的方案，既保证了高性能，又能处理超出内存容量的大规模数据。

## 1. JoinHashTable：连接哈希表

### 1.1 整体架构

`JoinHashTable` 是用于 Hash Join 的核心数据结构，采用线性探测开放寻址法：

```cpp
// src/include/duckdb/execution/join_hashtable.hpp

class JoinHashTable {
public:
    //! 用于计算盐值（salt）的阈值
    //! 当容量超过 8192 时使用 salt 优化比较
    static constexpr const idx_t USE_SALT_THRESHOLD = 8192;

    //! 默认负载因子 2.0（填充率 25%-50%）
    static constexpr double DEFAULT_LOAD_FACTOR = 2.0;
    //! 外部哈希表负载因子 1.5（填充率 33%-67%）
    static constexpr double EXTERNAL_LOAD_FACTOR = 1.5;
    //! 指针表最小容量
    static constexpr idx_t MINIMUM_CAPACITY = 16384;
    //! 初始 Radix 分区位数
    static constexpr const idx_t INITIAL_RADIX_BITS = 4;

    JoinHashTable(ClientContext &context, const PhysicalOperator &op,
                  const vector<JoinCondition> &conditions,
                  vector<LogicalType> build_types, JoinType type,
                  const vector<idx_t> &output_columns);

    //! 添加数据到哈希表
    void Build(PartitionedTupleDataAppendState &append_state,
               DataChunk &keys, DataChunk &input);

    //! 合并另一个哈希表
    void Merge(JoinHashTable &other);

    //! 分配指针表
    void AllocatePointerTable();

    //! 初始化指针表
    void InitializePointerTable(idx_t entry_idx_from, idx_t entry_idx_to);

    //! 完成构建，创建实际的哈希表
    void Finalize(idx_t chunk_idx_from, idx_t chunk_idx_to, bool parallel);

    //! 探测哈希表
    void Probe(ScanStructure &scan_structure, DataChunk &keys,
               TupleDataChunkState &key_state, ProbeState &probe_state,
               optional_ptr<Vector> precomputed_hashes = nullptr);

    //! 扫描 Full Outer Join 结果
    void ScanFullOuter(JoinHTScanState &state, Vector &addresses, DataChunk &result) const;

private:
    ClientContext &context;
    const PhysicalOperator &op;
    BufferManager &buffer_manager;

    //! Join 条件
    const vector<JoinCondition> &conditions;
    //! 等值比较的键类型
    vector<LogicalType> equality_types;
    //! Build 侧数据类型
    vector<LogicalType> build_types;

    //! 数据收集（Sink 阶段）
    unique_ptr<PartitionedTupleData> sink_collection;
    //! 数据存储（Finalize 后）
    unique_ptr<TupleDataCollection> data_collection;

    //! 哈希映射表
    AllocatedData hash_map;
    ht_entry_t *entries = nullptr;

    //! 容量和掩码
    idx_t capacity = DConstants::INVALID_INDEX;
    uint64_t bitmask = DConstants::INVALID_INDEX;

    //! 行大小信息
    idx_t entry_size;
    idx_t tuple_size;
    idx_t pointer_offset;

    //! 布隆过滤器
    BloomFilter bloom_filter;
    bool should_build_bloom_filter = false;

    //! Radix 分区位数（外部哈希连接）
    idx_t radix_bits;
    ValidityMask current_partitions;
    ValidityMask completed_partitions;
};
```

### 1.2 存储结构

```
┌─────────────────────────────────────────────────────────────┐
│              JoinHashTable 存储结构                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Tuple Data (链表结构):                                      │
│  ┌──────────────────────────────────────────────┐           │
│  │ [VALIDITY] [KEY1] [KEY2] [...] [PAY1] [PAY2] │ [NEXT]    │
│  │           ← Data Layout →                    │           │
│  └──────────────────────────────────────────────┘           │
│  ┌──────────────────────────────────────────────┐           │
│  │ [VALIDITY] [KEY1] [KEY2] [...] [PAY1] [PAY2] │ [NEXT]    │
│  └──────────────────────────────────────────────┘           │
│  ...                                                        │
│                                                             │
│  ═══════════════════════════════════════════════════════   │
│                                                             │
│  Pointer Table (指针表):                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [NULL] [PTR→] [NULL] [PTR→] [PTR→] [NULL] ... [NULL]   │ │
│  │   0      1      2      3      4      5          N-1    │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  冲突处理: 链表连接同一桶的多个元素                          │
│                                                             │
│     hash(k1) = 1 ──┐       hash(k2) = 3 ──┐                │
│                    ▼                       ▼                │
│     Bucket[1] → Tuple1 → Tuple5 → NULL                     │
│     Bucket[3] → Tuple2 → Tuple3 → Tuple7 → NULL            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 Build 流程

```cpp
// Build 阶段：插入数据到哈希表

void JoinHashTable::Build(PartitionedTupleDataAppendState &append_state,
                          DataChunk &keys, DataChunk &input) {
    // 1. 准备键值：过滤 NULL
    SelectionVector sel(STANDARD_VECTOR_SIZE);
    idx_t count = PrepareKeys(keys, vector_data, current_sel, sel, true);

    if (count == 0) {
        return;
    }

    // 2. 计算哈希值
    Vector hashes(LogicalType::HASH);
    Hash(keys, sel, count, hashes);

    // 3. 追加数据到分区存储
    sink_collection->Append(append_state, chunk_state, sel, count);

    // 4. 如果需要去重，进行即时插入
    if (!insert_duplicate_keys) {
        InsertHashes(hashes, count, chunk_state, insert_state, false);
    }
}
```

### 1.4 Finalize 流程

```cpp
// Finalize：构建指针表

void JoinHashTable::Finalize(idx_t chunk_idx_from, idx_t chunk_idx_to, bool parallel) {
    // 遍历所有数据块
    TupleDataChunkIterator iterator(*data_collection, ...);

    do {
        idx_t count = iterator.GetCurrentChunkCount();

        // 获取行地址和哈希值
        Vector addresses(LogicalType::POINTER);
        Vector hashes(LogicalType::HASH);
        iterator.GetChunkData(addresses, hashes, ...);

        // 插入到哈希表
        InsertHashesInternal(hashes, addresses, count, parallel);

    } while (iterator.Next());
}

// 线性探测插入
void InsertHashesInternal(Vector &hashes, Vector &addresses, idx_t count, bool parallel) {
    auto hash_data = FlatVector::GetData<hash_t>(hashes);
    auto addr_data = FlatVector::GetData<data_ptr_t>(addresses);

    for (idx_t i = 0; i < count; i++) {
        auto &entry = entries[hash_data[i] & bitmask];

        // 线性探测找到空位或链表尾部
        while (entry.IsOccupied()) {
            // 检查是否需要跟随链表
            // ...
        }

        // 设置指针
        if (parallel) {
            // 原子操作
            entry.SetPointerAtomic(addr_data[i]);
        } else {
            entry.SetPointer(addr_data[i]);
        }
    }
}
```

### 1.5 Probe 流程

```cpp
// 探测哈希表
void JoinHashTable::Probe(ScanStructure &scan_structure, DataChunk &keys,
                          TupleDataChunkState &key_state, ProbeState &probe_state,
                          optional_ptr<Vector> precomputed_hashes) {
    // 1. 计算哈希（或使用预计算的）
    Vector &hashes = precomputed_hashes ? *precomputed_hashes : probe_state.hashes_dense_v;
    if (!precomputed_hashes) {
        Hash(keys, *current_sel, count, hashes);
    }

    // 2. 布隆过滤器预过滤（如果启用）
    if (bloom_filter.IsSet()) {
        bloom_filter.Lookup(hashes, current_sel, count);
    }

    // 3. 获取初始指针
    GetRowPointers(keys, key_state, probe_state, hashes,
                   current_sel, count, scan_structure.pointers,
                   scan_structure.sel_vector, has_sel);

    // 4. 设置扫描结构
    scan_structure.count = count;
    scan_structure.finished = false;
}

// ScanStructure::Next 处理匹配
void ScanStructure::Next(DataChunk &keys, DataChunk &left, DataChunk &result) {
    switch (ht.join_type) {
    case JoinType::INNER:
        NextInnerJoin(keys, left, result);
        break;
    case JoinType::LEFT:
        NextLeftJoin(keys, left, result);
        break;
    case JoinType::SEMI:
        NextSemiJoin(keys, left, result);
        break;
    case JoinType::ANTI:
        NextAntiJoin(keys, left, result);
        break;
    case JoinType::MARK:
        NextMarkJoin(keys, left, result);
        break;
    case JoinType::SINGLE:
        NextSingleJoin(keys, left, result);
        break;
    // ...
    }
}
```

### 1.6 Salt 优化

当哈希表容量超过 L3 缓存时，使用 Salt 减少不必要的内存访问：

```
┌─────────────────────────────────────────────────────────────┐
│                    Salt 优化原理                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  传统方式（无 Salt）:                                        │
│  ┌──────────────────────────────────────────────┐           │
│  │ 1. 计算 hash(key)                            │           │
│  │ 2. 定位 bucket = hash & bitmask             │           │
│  │ 3. 加载 entries[bucket].pointer             │           │
│  │ 4. 加载 tuple data（可能 cache miss）        │           │
│  │ 5. 比较 key                                  │           │
│  │ 6. 不匹配则跟随链表...                       │           │
│  └──────────────────────────────────────────────┘           │
│                                                             │
│  使用 Salt 优化:                                             │
│  ┌──────────────────────────────────────────────┐           │
│  │ Entry = [Pointer (48-bit)] [Salt (16-bit)]   │           │
│  │                                               │           │
│  │ 1. 计算 hash(key)                            │           │
│  │ 2. 定位 bucket                               │           │
│  │ 3. 加载 entries[bucket] (8 bytes)           │           │
│  │ 4. 比较 salt = hash >> 48                   │           │
│  │    └─ Salt 不匹配：跳过，无需加载 tuple      │           │
│  │    └─ Salt 匹配：加载 tuple 验证 key         │           │
│  └──────────────────────────────────────────────┘           │
│                                                             │
│  优势: 减少 cache miss，salt 不匹配时避免加载实际数据        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. GroupedAggregateHashTable：聚合哈希表

### 2.1 类设计

```cpp
// src/include/duckdb/execution/aggregate_hashtable.hpp

class GroupedAggregateHashTable : public BaseAggregateHashTable {
public:
    //! 负载因子 1.5
    constexpr static double LOAD_FACTOR = 1.5;

    GroupedAggregateHashTable(ClientContext &context, Allocator &allocator,
                              vector<LogicalType> group_types,
                              vector<LogicalType> payload_types,
                              const vector<BoundAggregateExpression *> &aggregates,
                              idx_t initial_capacity = InitialCapacity(),
                              idx_t radix_bits = 0);

    //! 添加数据并计算聚合
    idx_t AddChunk(DataChunk &groups, DataChunk &payload,
                   const unsafe_vector<idx_t> &filter);

    //! 查找或创建分组
    idx_t FindOrCreateGroups(DataChunk &groups, Vector &group_hashes,
                             Vector &addresses_out, SelectionVector &new_groups_out);

    //! 扫描结果
    bool Scan(AggregateHTScanState &scan_state, DataChunk &distinct_rows,
              DataChunk &payload_rows);

    //! 合并另一个哈希表
    void Combine(GroupedAggregateHashTable &other);

    //! 调整大小
    void Resize(idx_t size);

private:
    ClientContext &context;
    RowMatcher row_matcher;

    //! Radix 分区位数
    idx_t radix_bits;
    //! 分区数据存储
    unique_ptr<PartitionedTupleData> partitioned_data;
    //! 非分区数据（优化小数据量场景）
    unique_ptr<PartitionedTupleData> unpartitioned_data;

    //! 分组数量
    idx_t count;
    //! 容量
    idx_t capacity;
    //! 哈希映射
    AllocatedData hash_map;
    ht_entry_t *entries;
    //! 掩码
    hash_t bitmask;

    //! 聚合状态分配器
    shared_ptr<ArenaAllocator> aggregate_allocator;
};
```

### 2.2 聚合数据布局

```
┌─────────────────────────────────────────────────────────────┐
│              聚合哈希表行布局                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  TupleDataLayout:                                           │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [VALIDITY] │ [GROUP_COL1] [GROUP_COL2] ... │ [AGG_STATE] │ │
│  │  flag_width │         data_width          │  aggr_width │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  示例: SELECT city, COUNT(*), SUM(amount) GROUP BY city     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ [1 byte]  │ [city: varchar*] │ [count: 8] [sum: 8]     │ │
│  │ validity  │    group key     │    aggregate states     │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  变长数据存储在堆上:                                         │
│  Row → [fixed_data] → heap → "New York"                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.3 AddChunk 流程

```cpp
idx_t GroupedAggregateHashTable::AddChunk(DataChunk &groups, DataChunk &payload,
                                          const unsafe_vector<idx_t> &filter) {
    // 1. 计算分组键的哈希值
    Vector hashes(LogicalType::HASH);
    groups.Hash(hashes);

    // 2. 可选：更新 HyperLogLog 估计
    if (enable_hll) {
        hll.Update(hashes, groups.size());
    }

    // 3. 查找或创建分组
    Vector addresses(LogicalType::POINTER);
    SelectionVector new_groups(STANDARD_VECTOR_SIZE);
    idx_t new_group_count = FindOrCreateGroups(groups, hashes, addresses, new_groups);

    // 4. 更新聚合状态
    UpdateAggregates(payload, filter);

    return new_group_count;
}

idx_t GroupedAggregateHashTable::FindOrCreateGroupsInternal(
    DataChunk &groups, Vector &group_hashes,
    Vector &addresses, SelectionVector &new_groups) {

    auto hash_data = FlatVector::GetData<hash_t>(group_hashes);
    auto addr_data = FlatVector::GetData<data_ptr_t>(addresses);
    idx_t new_group_count = 0;

    for (idx_t i = 0; i < groups.size(); i++) {
        hash_t hash = hash_data[i];
        idx_t bucket = hash & bitmask;

        // 线性探测
        while (true) {
            auto &entry = entries[bucket];

            if (!entry.IsOccupied()) {
                // 空槽：创建新分组
                data_ptr_t new_row = AllocateNewGroup();
                InitializeGroupRow(new_row, groups, i, hash);
                entry.SetPointer(new_row);
                addr_data[i] = new_row;
                new_groups.set_index(new_group_count++, i);
                count++;
                break;
            }

            // 检查是否匹配
            data_ptr_t existing = entry.GetPointer();
            if (row_matcher.Match(existing, groups, i)) {
                // 找到匹配的分组
                addr_data[i] = existing;
                break;
            }

            // 线性探测下一个槽
            bucket = (bucket + 1) & bitmask;
        }
    }

    // 检查是否需要 Resize
    if (count >= ResizeThreshold()) {
        Resize(capacity * 2);
    }

    return new_group_count;
}
```

### 2.4 Resize 机制

```cpp
void GroupedAggregateHashTable::Resize(idx_t new_capacity) {
    D_ASSERT(new_capacity > capacity);

    // 分配新的指针表
    idx_t new_size = new_capacity * sizeof(ht_entry_t);
    auto new_hash_map = Allocator::Get(context).Allocate(new_size);
    memset(new_hash_map.get(), 0, new_size);

    auto new_entries = reinterpret_cast<ht_entry_t *>(new_hash_map.get());
    hash_t new_bitmask = new_capacity - 1;

    // 重新插入所有条目
    ReinsertTuples(new_entries, new_bitmask);

    // 更新状态
    hash_map = std::move(new_hash_map);
    entries = new_entries;
    capacity = new_capacity;
    bitmask = new_bitmask;
}
```

## 3. RadixPartitionedHashTable

### 3.1 Radix 分区原理

当数据量很大时，使用 Radix 分区减少单个哈希表的大小：

```
┌─────────────────────────────────────────────────────────────┐
│                  Radix 分区原理                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  原始数据:                                                   │
│  ┌─────────────────────────────────────────────┐            │
│  │ Row1  Row2  Row3  Row4  Row5  Row6  Row7 ...│            │
│  └─────────────────────────────────────────────┘            │
│                     │                                       │
│                     ▼ 按 hash 的高 radix_bits 位分区        │
│                                                             │
│  radix_bits = 2 → 4 个分区:                                 │
│                                                             │
│  Partition 0 (hash & 0x3 == 0)                             │
│  ┌────────────────────┐                                     │
│  │ Row1  Row5  Row9  ...│                                   │
│  └────────────────────┘                                     │
│                                                             │
│  Partition 1 (hash & 0x3 == 1)                             │
│  ┌────────────────────┐                                     │
│  │ Row2  Row6  Row10 ...│                                   │
│  └────────────────────┘                                     │
│                                                             │
│  Partition 2 (hash & 0x3 == 2)                             │
│  ┌────────────────────┐                                     │
│  │ Row3  Row7  Row11 ...│                                   │
│  └────────────────────┘                                     │
│                                                             │
│  Partition 3 (hash & 0x3 == 3)                             │
│  ┌────────────────────┐                                     │
│  │ Row4  Row8  Row12 ...│                                   │
│  └────────────────────┘                                     │
│                                                             │
│  优势:                                                       │
│  1. 每个分区独立处理，可并行                                 │
│  2. 单个分区更易放入 CPU 缓存                                │
│  3. 支持外部（磁盘）哈希操作                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 RadixPartitionedHashTable 接口

```cpp
// src/include/duckdb/execution/radix_partitioned_hashtable.hpp

class RadixPartitionedHashTable {
public:
    RadixPartitionedHashTable(GroupingSet &grouping_set,
                              const GroupedAggregateData &op,
                              TupleDataValidityType group_validity);

    //! 创建底层哈希表
    unique_ptr<GroupedAggregateHashTable> CreateHT(ClientContext &context,
                                                    idx_t capacity,
                                                    idx_t radix_bits) const;

    // Sink 接口
    unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context) const;
    unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context) const;

    void Sink(ExecutionContext &context, DataChunk &chunk,
              OperatorSinkInput &input, DataChunk &aggregate_input_chunk,
              const unsafe_vector<idx_t> &filter) const;

    void Combine(ExecutionContext &context, GlobalSinkState &gstate,
                 LocalSinkState &lstate) const;

    void Finalize(ClientContext &context, GlobalSinkState &gstate) const;

    // Source 接口
    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) const;
    SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                             GlobalSinkState &sink, OperatorSourceInput &input) const;

private:
    GroupingSet &grouping_set;
    const GroupedAggregateData &op;
    vector<LogicalType> group_types;
    const TupleDataValidityType group_validity;
    shared_ptr<TupleDataLayout> layout_ptr;
};
```

## 4. TupleDataLayout：行数据布局

### 4.1 布局设计

`TupleDataLayout` 定义了行数据的内存布局：

```cpp
// src/include/duckdb/common/types/row/tuple_data_layout.hpp

class TupleDataLayout {
public:
    //! 初始化布局
    void Initialize(vector<LogicalType> types_p,
                    Aggregates aggregates_p,
                    TupleDataValidityType validity_type);

    //! 列数
    idx_t ColumnCount() const { return types.size(); }

    //! 聚合数
    idx_t AggregateCount() const { return aggregates.size(); }

    //! 总行宽度
    idx_t GetRowWidth() const { return row_width; }

    //! 数据偏移（跳过 validity）
    idx_t GetDataOffset() const { return flag_width; }

    //! 聚合状态偏移
    idx_t GetAggrOffset() const { return flag_width + data_width; }

    //! 各列偏移
    const vector<idx_t> &GetOffsets() const { return offsets; }

    //! 是否所有列都是定长
    bool AllConstant() const { return all_constant; }

    //! 变长列索引
    const vector<idx_t> &GetVariableColumns() const { return variable_columns; }

private:
    vector<LogicalType> types;          // 列类型
    Aggregates aggregates;              // 聚合函数

    idx_t flag_width;                   // Validity 宽度
    idx_t data_width;                   // 数据宽度
    idx_t aggr_width;                   // 聚合状态宽度
    idx_t row_width;                    // 总行宽度

    vector<idx_t> offsets;              // 各列偏移
    bool all_constant;                  // 是否全定长
    vector<idx_t> variable_columns;     // 变长列索引
    idx_t heap_size_offset;             // 堆大小偏移

    TupleDataValidityType validity_type; // Validity 类型
};
```

### 4.2 布局计算

```
┌─────────────────────────────────────────────────────────────┐
│                   行数据布局示例                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  类型: [INT, VARCHAR, DOUBLE]                                │
│  聚合: [COUNT(*), SUM(amount)]                               │
│                                                             │
│  布局:                                                       │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  Offset   │ Content         │ Size                     │ │
│  ├───────────┼─────────────────┼──────────────────────────┤ │
│  │     0     │ Validity Mask   │ 1 byte (3 bits used)     │ │
│  │     1     │ INT column      │ 4 bytes                  │ │
│  │     5     │ VARCHAR ptr     │ 8 bytes (指向堆)         │ │
│  │    13     │ DOUBLE column   │ 8 bytes                  │ │
│  │    21     │ COUNT state     │ 8 bytes                  │ │
│  │    29     │ SUM state       │ 16 bytes (含 NULL 标志)  │ │
│  │    45     │ Heap size       │ 4 bytes                  │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  row_width = 49 (可能对齐到 56 或 64)                        │
│                                                             │
│  变长数据存储:                                               │
│  ┌────────────┐     ┌──────────────────────────────────────┐│
│  │ Row Data   │ ──► │ Heap: [len][string data...]          ││
│  └────────────┘     └──────────────────────────────────────┘│
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 5. 内存管理系统

### 5.1 Allocator 抽象

```cpp
// src/include/duckdb/common/allocator.hpp

class Allocator {
    static constexpr const idx_t MAXIMUM_ALLOC_SIZE = 281474976710656ULL;  // 281TB

public:
    Allocator();
    Allocator(allocate_function_ptr_t allocate_function,
              free_function_ptr_t free_function,
              reallocate_function_ptr_t reallocate_function,
              unique_ptr<PrivateAllocatorData> private_data);

    //! 分配内存
    data_ptr_t AllocateData(idx_t size);

    //! 释放内存
    void FreeData(data_ptr_t pointer, idx_t size);

    //! 重新分配
    data_ptr_t ReallocateData(data_ptr_t pointer, idx_t old_size, idx_t new_size);

    //! 返回包装的 AllocatedData
    AllocatedData Allocate(idx_t size) {
        return AllocatedData(*this, AllocateData(size), size);
    }

    //! 获取默认分配器
    static Allocator &DefaultAllocator();

    //! 获取特定上下文的分配器
    static Allocator &Get(ClientContext &context);
    static Allocator &Get(DatabaseInstance &db);

private:
    allocate_function_ptr_t allocate_function;
    free_function_ptr_t free_function;
    reallocate_function_ptr_t reallocate_function;
    unique_ptr<PrivateAllocatorData> private_data;
};

//! BufferAllocator 通过 BufferManager 分配内存，支持内存追踪
struct BufferAllocator {
    static Allocator &Get(ClientContext &context);
    static Allocator &Get(DatabaseInstance &db);
};
```

### 5.2 BufferManager

```cpp
// src/include/duckdb/storage/buffer_manager.hpp

class BufferManager {
public:
    //! 分配临时内存块
    virtual shared_ptr<BlockHandle> AllocateTemporaryMemory(
        MemoryTag tag, idx_t block_size, bool can_destroy = true) = 0;

    //! 分配并固定内存
    virtual BufferHandle Allocate(MemoryTag tag, idx_t block_size,
                                  bool can_destroy = true) = 0;

    //! 重新分配
    virtual void ReAllocate(shared_ptr<BlockHandle> &handle, idx_t block_size) = 0;

    //! 固定（Pin）块
    virtual BufferHandle Pin(shared_ptr<BlockHandle> &handle) = 0;

    //! 取消固定（Unpin）块
    virtual void Unpin(shared_ptr<BlockHandle> &handle) = 0;

    //! 预取块
    virtual void Prefetch(vector<shared_ptr<BlockHandle>> &handles) = 0;

    //! 当前使用的内存
    virtual idx_t GetUsedMemory() const = 0;

    //! 最大可用内存
    virtual idx_t GetMaxMemory() const = 0;

    //! 当前使用的交换空间
    virtual idx_t GetUsedSwap() const = 0;

    //! 块分配大小
    virtual idx_t GetBlockAllocSize() const = 0;

    //! 获取临时内存管理器
    virtual TemporaryMemoryManager &GetTemporaryMemoryManager();

    //! 写临时缓冲区到磁盘
    virtual void WriteTemporaryBuffer(MemoryTag tag, block_id_t block_id,
                                      FileBuffer &buffer);

    //! 从磁盘读取临时缓冲区
    virtual unique_ptr<FileBuffer> ReadTemporaryBuffer(
        QueryContext context, MemoryTag tag, BlockHandle &block,
        unique_ptr<FileBuffer> buffer);
};
```

### 5.3 TemporaryMemoryManager

```cpp
// src/include/duckdb/storage/temporary_memory_manager.hpp

class TemporaryMemoryState {
public:
    //! 设置剩余所需大小
    void SetRemainingSize(idx_t new_remaining_size);

    //! 更新预留量
    void UpdateReservation(ClientContext &context);

    //! 获取预留量
    idx_t GetReservation() const;

    //! 设置最小预留量
    void SetMinimumReservation(idx_t new_minimum_reservation);

private:
    TemporaryMemoryManager &temporary_memory_manager;
    atomic<idx_t> remaining_size;        // 剩余所需大小
    atomic<idx_t> minimum_reservation;   // 最小预留
    atomic<idx_t> reservation;           // 当前预留量
    atomic<idx_t> materialization_penalty; // 物化惩罚权重
};

class TemporaryMemoryManager {
    //! 每状态每线程最小预留：512 * 256KB = 128MB
    static constexpr idx_t MINIMUM_RESERVATION_PER_STATE_PER_THREAD =
        512ULL * DEFAULT_BLOCK_ALLOC_SIZE;
    //! 或者内存限制的 1/16
    static constexpr idx_t MINIMUM_RESERVATION_MEMORY_LIMIT_DIVISOR = 16ULL;
    //! 最大使用内存限制的 90%
    static constexpr double MAXIMUM_MEMORY_LIMIT_RATIO = 0.9;

public:
    //! 注册一个临时内存状态
    unique_ptr<TemporaryMemoryState> Register(ClientContext &context);

private:
    mutex lock;
    idx_t memory_limit;
    bool has_temporary_directory;
    idx_t num_threads;

    //! 活跃状态集合
    reference_set_t<TemporaryMemoryState> active_states;
    //! 总预留量
    idx_t reservation;
    //! 总剩余大小
    idx_t remaining_size;
};
```

### 5.4 内存预留机制

```
┌─────────────────────────────────────────────────────────────┐
│                 临时内存预留机制                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  多个并发状态竞争有限内存:                                   │
│                                                             │
│  Memory Limit: 8GB                                          │
│  ┌─────────────────────────────────────────────┐            │
│  │■■■■■■■■■■■■■■■■■■■■░░░░░░░░░░░░░░░░░░░░░░░░│            │
│  │ Used (5GB)         │ Available (3GB)       │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  活跃状态:                                                   │
│  ┌──────────────────────────────────────────────┐           │
│  │ State 1 (HashJoin Build)                    │           │
│  │   remaining: 2GB, reservation: 1.5GB        │           │
│  ├──────────────────────────────────────────────┤           │
│  │ State 2 (HashAggregate)                     │           │
│  │   remaining: 1GB, reservation: 0.8GB        │           │
│  ├──────────────────────────────────────────────┤           │
│  │ State 3 (OrderBy)                           │           │
│  │   remaining: 500MB, reservation: 0.4GB      │           │
│  └──────────────────────────────────────────────┘           │
│                                                             │
│  预留计算:                                                   │
│  1. 最小保证: min(512*block_size, limit/16)                 │
│  2. 按剩余大小和权重分配额外内存                             │
│  3. 总预留不超过 limit * 0.9                                │
│                                                             │
│  当内存不足时:                                               │
│  1. 降低预留量                                               │
│  2. 触发外部算法（如外部排序、外部哈希）                     │
│  3. 将数据溢出到临时目录                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 6. 外部哈希操作

### 6.1 外部 Hash Join

当哈希表无法完全放入内存时，DuckDB 使用外部哈希连接：

```
┌─────────────────────────────────────────────────────────────┐
│                 外部 Hash Join 流程                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Phase 1: 分区 Build 侧数据                                  │
│  ┌─────────────────────────────────────────────┐            │
│  │ Build 数据流                                │            │
│  │     │                                        │            │
│  │     ▼ 按 hash 分区                          │            │
│  │ ┌────────┬────────┬────────┬────────┐       │            │
│  │ │ Part 0 │ Part 1 │ Part 2 │ Part 3 │       │            │
│  │ └────────┴────────┴────────┴────────┘       │            │
│  │     │        │        │        │            │            │
│  │     ▼        ▼        ▼        ▼            │            │
│  │   Disk    Memory   Disk     Disk            │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Phase 2: 分区 Probe 侧并探测                                │
│  ┌─────────────────────────────────────────────┐            │
│  │ Probe 数据流                                │            │
│  │     │                                        │            │
│  │     ▼ 按 hash 分区                          │            │
│  │ ┌────────┬────────┬────────┬────────┐       │            │
│  │ │ Part 0 │ Part 1 │ Part 2 │ Part 3 │       │            │
│  │ └────────┴────────┴────────┴────────┘       │            │
│  │                                              │            │
│  │ 如果 Build Part 1 在内存中:                  │            │
│  │   直接探测                                   │            │
│  │                                              │            │
│  │ 如果 Build Part 在磁盘:                      │            │
│  │   溢出 Probe Part 到磁盘                     │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Phase 3: 逐分区处理                                         │
│  ┌─────────────────────────────────────────────┐            │
│  │ for each partition:                         │            │
│  │   1. 加载 Build Part 到内存                 │            │
│  │   2. 构建哈希表                              │            │
│  │   3. 加载并探测 Probe Part                  │            │
│  │   4. 输出结果                                │            │
│  │   5. 释放分区                                │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 ProbeSpill

```cpp
// JoinHashTable 中的 ProbeSpill 结构

struct ProbeSpill {
public:
    ProbeSpill(JoinHashTable &ht, ClientContext &context,
               const vector<LogicalType> &probe_types);

    //! 注册线程
    ProbeSpillLocalAppendState RegisterThread();

    //! 追加数据
    void Append(DataChunk &chunk, ProbeSpillLocalAppendState &local_state);

    //! 完成合并
    void Finalize();

    //! 准备下一轮探测
    void PrepareNextProbe();

    //! 数据消费器
    unique_ptr<ColumnDataConsumer> consumer;

private:
    JoinHashTable &ht;
    ClientContext &context;
    const vector<LogicalType> &probe_types;

    //! 分区数据
    unique_ptr<PartitionedColumnData> global_partitions;
    //! 全局溢出收集
    unique_ptr<ColumnDataCollection> global_spill_collection;
};
```

## 7. 布隆过滤器优化

### 7.1 布隆过滤器原理

```
┌─────────────────────────────────────────────────────────────┐
│                    布隆过滤器优化                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Build 阶段: 构建布隆过滤器                                  │
│  ┌─────────────────────────────────────────────┐            │
│  │ 对每个 Build 键:                            │            │
│  │   h1 = hash1(key)                           │            │
│  │   h2 = hash2(key)                           │            │
│  │   h3 = hash3(key)                           │            │
│  │   设置 bits[h1], bits[h2], bits[h3] = 1    │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  Probe 阶段: 过滤不匹配的键                                  │
│  ┌─────────────────────────────────────────────┐            │
│  │ 对每个 Probe 键:                            │            │
│  │   h1 = hash1(key)                           │            │
│  │   h2 = hash2(key)                           │            │
│  │   h3 = hash3(key)                           │            │
│  │   if any bit is 0:                          │            │
│  │     肯定不匹配，跳过                         │            │
│  │   else:                                      │            │
│  │     可能匹配，继续探测哈希表                 │            │
│  └─────────────────────────────────────────────┘            │
│                                                             │
│  优势:                                                       │
│  1. 快速过滤不匹配的 Probe 行                                │
│  2. 减少对主哈希表的访问                                     │
│  3. 可下推到 TableScan（动态过滤器）                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 代码集成

```cpp
// JoinHashTable 中的布隆过滤器

class JoinHashTable {
    BloomFilter bloom_filter;
    bool should_build_bloom_filter = false;

public:
    void SetBuildBloomFilter(bool should_build) {
        this->should_build_bloom_filter = should_build;
    }

    BloomFilter &GetBloomFilter() {
        return bloom_filter;
    }
};

// Probe 时使用布隆过滤器
void JoinHashTable::Probe(ScanStructure &scan_structure, ...) {
    // 计算哈希
    Vector &hashes = ...;
    Hash(keys, *current_sel, count, hashes);

    // 布隆过滤器预过滤
    if (bloom_filter.IsSet()) {
        bloom_filter.Lookup(hashes, current_sel, count);
        // count 可能减少，跳过了不匹配的行
    }

    // 继续正常探测...
}
```

## 8. 总结

### 8.1 哈希表设计要点

| 特性 | JoinHashTable | GroupedAggregateHashTable |
|------|--------------|---------------------------|
| 用途 | Hash Join | Hash Aggregate |
| 冲突解决 | 链表 + 线性探测 | 纯线性探测 |
| 负载因子 | 2.0 (内部) / 1.5 (外部) | 1.5 |
| Salt 优化 | 容量 > 8192 时启用 | 支持 |
| 分区支持 | Radix 分区 | Radix 分区 |
| 外部算法 | 支持 | 支持 |

### 8.2 内存管理层次

```
┌────────────────────────────────────────────────────────────────┐
│                     内存管理层次结构                            │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  应用层:                                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ JoinHashTable / GroupedAggregateHashTable                │  │
│  │ ↓ 使用 TupleDataLayout 管理行布局                        │  │
│  │ ↓ 通过 TemporaryMemoryState 预留内存                     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  内存预留层:                                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ TemporaryMemoryManager                                   │  │
│  │ ↓ 协调多个算子的内存使用                                  │  │
│  │ ↓ 计算最优预留分配                                        │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  缓冲管理层:                                                    │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ BufferManager                                            │  │
│  │ ↓ 块级内存管理（Pin/Unpin）                               │  │
│  │ ↓ 驱逐策略和临时文件管理                                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                     │
│  分配层:                                                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Allocator / BufferAllocator                              │  │
│  │ ↓ 实际内存分配（malloc/jemalloc）                         │  │
│  │ ↓ 内存使用追踪                                            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### 8.3 关键设计决策

1. **线性探测 + 链表**：JoinHashTable 结合两种方式，兼顾缓存效率和处理多重匹配
2. **Salt 优化**：减少大哈希表的 cache miss
3. **Radix 分区**：支持并行和外部算法
4. **动态内存预留**：多状态公平竞争有限内存
5. **布隆过滤器**：预过滤不匹配数据，减少探测开销
6. **外部算法支持**：优雅处理超出内存容量的数据

本章完成了 DuckDB 执行引擎系列的最后一个主题。通过六章的深入分析，我们全面了解了 DuckDB 如何通过向量化执行、高效的数据结构、灵活的算子设计、并行 Pipeline 和智能内存管理来实现高性能的分析查询处理。
