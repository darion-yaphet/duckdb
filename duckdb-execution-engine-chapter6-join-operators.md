# DuckDB 执行引擎深度解析：第六章 - Join 算子实现

## 6.1 Join 算子概述

Join 是数据库查询处理中最核心也是最复杂的操作之一。DuckDB 实现了多种 Join 物理算子，根据查询特点和数据规模自动选择最优实现。

### 6.1.1 Join 类型分类

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DuckDB Join 类型体系                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 标准 Join 类型                                                       │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ INNER    - 返回两边都匹配的行                                        │   │
│  │ LEFT     - 返回左表所有行，右表匹配或 NULL                           │   │
│  │ RIGHT    - 返回右表所有行，左表匹配或 NULL                           │   │
│  │ OUTER    - 返回两表所有行（FULL OUTER JOIN）                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 半连接 Join 类型                                                     │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ SEMI     - 左表中存在匹配的行（用于 IN 子查询）                      │   │
│  │ ANTI     - 左表中不存在匹配的行（用于 NOT IN 子查询）                │   │
│  │ MARK     - 返回左表所有行，附加布尔标记列表示是否匹配                │   │
│  │ SINGLE   - 期望右表最多一行匹配（用于标量子查询）                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 右侧变体                                                             │   │
│  ├─────────────────────────────────────────────────────────────────────┤   │
│  │ RIGHT_SEMI  - 右表中存在匹配的行                                     │   │
│  │ RIGHT_ANTI  - 右表中不存在匹配的行                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.1.2 物理算子类型

DuckDB 提供了多种物理 Join 实现：

| 物理算子 | 适用场景 | 时间复杂度 |
|---------|---------|-----------|
| `PhysicalHashJoin` | 等值连接，大表连接 | O(n + m) |
| `PhysicalPiecewiseMergeJoin` | 范围条件（<, >, <=, >=） | O(n log n + m log m) |
| `PhysicalNestedLoopJoin` | 复杂条件，小表 | O(n * m) |
| `PhysicalCrossProduct` | 无条件笛卡尔积 | O(n * m) |
| `PhysicalIEJoin` | 不等式连接优化 | O((n+m) log(n+m)) |
| `PhysicalAsOfJoin` | 时间序列最近匹配 | O(n log m) |
| `PhysicalPositionalJoin` | 按位置对齐连接 | O(min(n, m)) |

### 6.1.3 Join 算子的 Pipeline 角色

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Join 在 Pipeline 中的双重角色                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pipeline 1 (Build 侧):                                                     │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────────────────────┐     │
│  │ TableScan   │───▶│   Filter    │───▶│ HashJoin (Sink)             │     │
│  │ (customers) │    │             │    │ - Sink(): 构建 Hash Table    │     │
│  └─────────────┘    └─────────────┘    │ - Finalize(): 完成构建       │     │
│                                         └─────────────────────────────┘     │
│                                                        │                     │
│                                                        │ 等待 Build 完成     │
│                                                        ▼                     │
│  Pipeline 2 (Probe 侧):                                                     │
│  ┌─────────────┐    ┌─────────────────────────────────────────────────┐     │
│  │ TableScan   │───▶│ HashJoin (Operator)                              │     │
│  │ (orders)    │    │ - ExecuteInternal(): 探测 Hash Table            │     │
│  └─────────────┘    └─────────────────────────────────────────────────┘     │
│                                         │                                    │
│                                         │ 可能变为 Source                    │
│                                         ▼                                    │
│  Pipeline 3 (Full/Right Outer):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HashJoin (Source)                                                    │   │
│  │ - GetDataInternal(): 扫描未匹配的 Build 侧数据                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.2 PhysicalHashJoin - 哈希连接

PhysicalHashJoin 是 DuckDB 中最重要的 Join 实现，用于处理等值连接（equi-join）。

### 6.2.1 类继承结构

```cpp
// src/include/duckdb/execution/operator/join/physical_hash_join.hpp
class PhysicalHashJoin : public PhysicalComparisonJoin {
public:
    // Join 键类型
    vector<LogicalType> condition_types;

    // 输出列信息
    JoinProjectionColumns payload_columns;      // RHS 非键列
    JoinProjectionColumns lhs_output_columns;   // LHS 输出列
    JoinProjectionColumns rhs_output_columns;   // RHS 输出列

    // 用于关联 MARK join
    vector<LogicalType> delim_types;

    // Join 键统计信息（用于 Perfect Hash Join 决策）
    vector<unique_ptr<BaseStatistics>> join_stats;
};
```

### 6.2.2 Build 阶段（Sink 接口）

Build 阶段将右表数据构建成哈希表：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Hash Join Build 阶段                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  输入 Chunk (RHS)                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ customer_id │ name        │ city      │ ...                         │   │
│  ├─────────────┼─────────────┼───────────┼─────────────────────────────┤   │
│  │     1       │ "Alice"     │ "Beijing" │                             │   │
│  │     2       │ "Bob"       │ "Shanghai"│                             │   │
│  │     3       │ "Carol"     │ "Beijing" │                             │   │
│  └─────────────┴─────────────┴───────────┴─────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 计算 Join 键                                                      │   │
│  │    join_key_executor.Execute(chunk, join_keys)                       │   │
│  │    结果: [1, 2, 3] (customer_id 列)                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. 收集 Min/Max 统计（用于动态过滤）                                  │   │
│  │    filter_pushdown->Sink(join_keys, local_filter_state)              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. 构建 Hash Table                                                   │   │
│  │    hash_table->Build(append_state, join_keys, payload_chunk)         │   │
│  │                                                                       │   │
│  │    Hash Table 内部结构:                                               │   │
│  │    ┌─────────────────────────────────────────────────────────────┐   │   │
│  │    │ Pointer Table (entries)                                      │   │   │
│  │    │ [0] → NULL                                                   │   │   │
│  │    │ [1] → Row(key=1, name="Alice", city="Beijing") → NULL       │   │   │
│  │    │ [2] → Row(key=2, name="Bob", city="Shanghai") → NULL        │   │   │
│  │    │ [3] → Row(key=3, name="Carol", city="Beijing") → NULL       │   │   │
│  │    │ ...                                                          │   │   │
│  │    └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Sink 实现

```cpp
// src/execution/operator/join/physical_hash_join.cpp
SinkResultType PhysicalHashJoin::Sink(ExecutionContext &context, DataChunk &chunk,
                                       OperatorSinkInput &input) const {
    auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
    auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

    // 1. 计算 Join 键
    lstate.join_keys.Reset();
    lstate.join_key_executor.Execute(chunk, lstate.join_keys);

    // 2. 动态过滤统计收集
    if (filter_pushdown && !gstate.skip_filter_pushdown) {
        filter_pushdown->Sink(lstate.join_keys, *lstate.local_filter_state);
    }

    // 3. 准备 Payload（非键列）
    if (payload_columns.col_types.empty()) {
        lstate.payload_chunk.SetCardinality(chunk.size());
    } else {
        lstate.payload_chunk.ReferenceColumns(chunk, payload_columns.col_idxs);
    }

    // 4. 构建线程本地 Hash Table
    lstate.hash_table->Build(lstate.append_state, lstate.join_keys, lstate.payload_chunk);

    return SinkResultType::NEED_MORE_INPUT;
}
```

#### Combine 与 Finalize

```cpp
// Combine: 合并线程本地 Hash Table
SinkCombineResultType PhysicalHashJoin::Combine(...) const {
    auto &gstate = input.global_state.Cast<HashJoinGlobalSinkState>();
    auto &lstate = input.local_state.Cast<HashJoinLocalSinkState>();

    // 刷新本地状态
    lstate.hash_table->GetSinkCollection().FlushAppendState(lstate.append_state);

    // 将本地 hash table 添加到全局列表
    auto guard = gstate.Lock();
    gstate.local_hash_tables.push_back(std::move(lstate.hash_table));

    // 合并过滤统计
    if (filter_pushdown) {
        filter_pushdown->Combine(*gstate.global_filter_state, *lstate.local_filter_state);
    }
    return SinkCombineResultType::FINISHED;
}

// Finalize: 最终化 Hash Table
SinkFinalizeType PhysicalHashJoin::Finalize(...) const {
    auto &sink = input.global_state.Cast<HashJoinGlobalSinkState>();
    auto &ht = *sink.hash_table;

    // 检查是否需要外部 Join
    sink.external = sink.temporary_memory_state->GetReservation() < sink.total_size;

    if (!sink.external) {
        // 内存 Join: 合并所有本地 hash table
        for (auto &local_ht : sink.local_hash_tables) {
            ht.Merge(*local_ht);
        }
        ht.Unpartition();

        // 尝试使用 Perfect Hash Join
        auto use_perfect_hash = sink.perfect_join_executor->CanDoPerfectHashJoin(...);
        if (use_perfect_hash) {
            use_perfect_hash = sink.perfect_join_executor->BuildPerfectHashTable(key_type);
        }

        // 生成动态过滤器
        if (filter_min_max) {
            filter_pushdown->FinalizeFilters(context, &ht, *this, ...);
        }

        // 调度并行 Finalize
        if (!use_perfect_hash) {
            sink.ScheduleFinalize(pipeline, event);
        }
    } else {
        // 外部 Join: 需要重新分区
        ...
    }
}
```

### 6.2.3 Probe 阶段（Operator 接口）

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Hash Join Probe 阶段                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  输入 Chunk (LHS - Probe 侧)                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ order_id │ customer_id │ amount │ date       │                      │   │
│  ├──────────┼─────────────┼────────┼────────────┼──────────────────────┤   │
│  │   101    │      1      │  100   │ 2024-01-01 │                      │   │
│  │   102    │      2      │  200   │ 2024-01-02 │                      │   │
│  │   103    │      5      │  150   │ 2024-01-03 │  (无匹配)            │   │
│  └──────────┴─────────────┴────────┴────────────┴──────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 计算 Probe 键                                                     │   │
│  │    probe_executor.Execute(input, lhs_join_keys)                      │   │
│  │    结果: [1, 2, 5]                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 2. 探测 Hash Table                                                   │   │
│  │    hash_table->Probe(scan_structure, lhs_join_keys, ...)             │   │
│  │                                                                       │   │
│  │    ScanStructure 状态:                                                │   │
│  │    ┌─────────────────────────────────────────────────────────────┐   │   │
│  │    │ pointers: [ptr_to_row_1, ptr_to_row_2, NULL]                │   │   │
│  │    │ count: 2 (匹配数)                                            │   │   │
│  │    │ sel_vector: [0, 1]                                           │   │   │
│  │    └─────────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 3. 构建结果                                                          │   │
│  │    scan_structure.Next(lhs_join_keys, lhs_output, chunk)             │   │
│  │                                                                       │   │
│  │    输出 Chunk:                                                        │   │
│  │    ┌─────────────────────────────────────────────────────────────┐   │   │
│  │    │ order_id │ c_id │ amount │ date       │ name    │ city      │   │   │
│  │    ├──────────┼──────┼────────┼────────────┼─────────┼───────────┤   │   │
│  │    │   101    │  1   │  100   │ 2024-01-01 │ "Alice" │ "Beijing" │   │   │
│  │    │   102    │  2   │  200   │ 2024-01-02 │ "Bob"   │ "Shanghai"│   │   │
│  │    └──────────┴──────┴────────┴────────────┴─────────┴───────────┘   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### ExecuteInternal 实现

```cpp
// src/execution/operator/join/physical_hash_join.cpp
OperatorResultType PhysicalHashJoin::ExecuteInternal(...) const {
    auto &state = state_p.Cast<HashJoinOperatorState>();
    auto &sink = sink_state->Cast<HashJoinGlobalSinkState>();

    // 空 Hash Table 处理
    if (sink.hash_table->Count() == 0) {
        if (EmptyResultIfRHSIsEmpty()) {
            return OperatorResultType::FINISHED;
        }
        // LEFT/OUTER Join: 输出左表 + NULL
        ConstructEmptyJoinResult(sink.hash_table->join_type, ...);
        return OperatorResultType::NEED_MORE_INPUT;
    }

    // Perfect Hash Join 路径
    if (sink.perfect_join_executor) {
        return sink.perfect_join_executor->ProbePerfectHashTable(...);
    }

    // 标准 Hash Join
    if (state.scan_structure.is_null) {
        // 新的输入 Chunk: 计算 Join 键并探测
        state.lhs_join_keys.Reset();
        state.probe_executor.Execute(input, state.lhs_join_keys);

        if (sink.external) {
            // 外部 Join: 可能需要 Spill
            sink.hash_table->ProbeAndSpill(...);
        } else {
            sink.hash_table->Probe(state.scan_structure, state.lhs_join_keys, ...);
        }
    }

    // 获取下一批匹配结果
    state.lhs_output.ReferenceColumns(input, lhs_output_columns.col_idxs);
    state.scan_structure.Next(state.lhs_join_keys, state.lhs_output, chunk);

    if (state.scan_structure.PointersExhausted() && chunk.size() == 0) {
        state.scan_structure.is_null = true;
        return OperatorResultType::NEED_MORE_INPUT;
    }
    return OperatorResultType::HAVE_MORE_OUTPUT;
}
```

### 6.2.4 Parallel Finalize 机制

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     并行 Hash Table Finalize                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  阶段 1: HashJoinTableInitEvent                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 并行初始化 Pointer Table (memset)                                    │   │
│  │                                                                       │   │
│  │ Task 1: 初始化 entries[0..131072]                                    │   │
│  │ Task 2: 初始化 entries[131072..262144]                               │   │
│  │ Task 3: 初始化 entries[262144..393216]                               │   │
│  │ ...                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼ InsertEvent                                                        │
│  阶段 2: HashJoinFinalizeEvent                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 并行填充 Hash Table                                                   │   │
│  │                                                                       │   │
│  │ Task 1: 处理 chunks[0..64]                                           │   │
│  │ Task 2: 处理 chunks[64..128]                                         │   │
│  │ Task 3: 处理 chunks[128..192]                                        │   │
│  │ ...                                                                   │   │
│  │                                                                       │   │
│  │ 每个 Task 调用:                                                       │   │
│  │   hash_table->Finalize(chunk_idx_from, chunk_idx_to, parallel=true)  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  数据倾斜处理:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ if (max_partition_size / total_size > 0.33) {                        │   │
│  │     // 数据严重倾斜，使用单线程 Finalize 避免原子操作竞争             │   │
│  │     FinalizeSingleThreaded()                                          │   │
│  │ }                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.3 JoinHashTable - 哈希表实现

### 6.3.1 内存布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     JoinHashTable 内存布局                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pointer Table (entries)                   Data Collection                  │
│  ┌──────────────────────┐                  ┌──────────────────────────────┐│
│  │ [0] NULL             │                  │ TupleDataCollection          ││
│  │ [1] ─────────────────┼─────────────────▶│ ┌────────────────────────┐   ││
│  │ [2] NULL             │                  │ │ Chunk 0:               │   ││
│  │ [3] ─────────────────┼──────────┐       │ │ [Row0][NEXT_PTR]      │   ││
│  │ [4] NULL             │          │       │ │ [Row1][NEXT_PTR]──────┼─┐ ││
│  │ [5] ─────────────────┼───────┐  │       │ │ [Row2][NEXT_PTR]      │ │ ││
│  │ ...                  │       │  │       │ └────────────────────────┘ │ ││
│  │                      │       │  │       │ ┌────────────────────────┐ │ ││
│  │ capacity = 2^n       │       │  └──────▶│ │ Chunk 1:               │ │ ││
│  │ bitmask = 2^n - 1    │       │          │ │ [Row3][NEXT_PTR]◀─────┼─┘ ││
│  └──────────────────────┘       │          │ │ [Row4][NEXT_PTR]      │   ││
│                                  │          │ └────────────────────────┘   ││
│                                  └─────────▶│ ...                          ││
│                                             └──────────────────────────────┘│
│                                                                             │
│  Row 格式:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ [Validity][Key1][Key2]...[Payload1][Payload2]...[NEXT_PTR/HASH]     │   │
│  │                                                                       │   │
│  │ - Validity: NULL 位图                                                 │   │
│  │ - KeyN: Join 键列                                                     │   │
│  │ - PayloadN: 非键列（需要输出的列）                                    │   │
│  │ - NEXT_PTR: 链表指针（用于处理哈希冲突）                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.3.2 核心接口

```cpp
// src/include/duckdb/execution/join_hashtable.hpp
class JoinHashTable {
public:
    // 构建接口
    void Build(PartitionedTupleDataAppendState &append_state,
               DataChunk &keys, DataChunk &input);

    // 合并多个 Hash Table
    void Merge(JoinHashTable &other);

    // 最终化（构建 Pointer Table）
    void AllocatePointerTable();
    void InitializePointerTable(idx_t entry_idx_from, idx_t entry_idx_to);
    void Finalize(idx_t chunk_idx_from, idx_t chunk_idx_to, bool parallel);

    // 探测接口
    void Probe(ScanStructure &scan_structure, DataChunk &keys,
               TupleDataChunkState &key_state, ProbeState &probe_state,
               optional_ptr<Vector> precomputed_hashes = nullptr);

    // Full/Right Outer 扫描
    void ScanFullOuter(JoinHTScanState &state, Vector &addresses, DataChunk &result) const;

    // 常量
    static constexpr double DEFAULT_LOAD_FACTOR = 2.0;  // 25%-50% 填充率
    static constexpr double EXTERNAL_LOAD_FACTOR = 1.5; // 外部 Join 使用更低填充率
    static constexpr idx_t MINIMUM_CAPACITY = 16384;
    static constexpr idx_t USE_SALT_THRESHOLD = 8192;   // 超过此容量使用 Salt
};
```

### 6.3.3 ScanStructure - 探测结果迭代器

```cpp
// 用于迭代探测结果的结构
struct ScanStructure {
    Vector pointers;                      // 指向匹配行的指针
    idx_t count;                          // 当前匹配数
    SelectionVector sel_vector;           // 匹配行的选择向量
    SelectionVector chain_match_sel_vector;
    SelectionVector chain_no_match_sel_vector;
    unsafe_unique_array<bool> found_match; // 用于 SEMI/ANTI/LEFT join
    bool finished;
    bool is_null;

    // 获取下一批匹配结果
    void Next(DataChunk &keys, DataChunk &left, DataChunk &result);

    // 检查是否还有更多结果
    bool PointersExhausted() const;

private:
    void NextInnerJoin(DataChunk &keys, DataChunk &left, DataChunk &result);
    void NextSemiJoin(DataChunk &keys, DataChunk &left, DataChunk &result);
    void NextAntiJoin(DataChunk &keys, DataChunk &left, DataChunk &result);
    void NextLeftJoin(DataChunk &keys, DataChunk &left, DataChunk &result);
    void NextMarkJoin(DataChunk &keys, DataChunk &left, DataChunk &result);
};
```

### 6.3.4 Salt 优化

对于大型 Hash Table，使用 Salt 避免不必要的键比较：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Salt 优化机制                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Hash 值结构:                                                                │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ 64-bit Hash Value                                                   │    │
│  ├────────────────────────────────────────────────────────────────────┤    │
│  │ [     Salt (高位)     ][        Index (低位)        ]              │    │
│  │        8-16 bits              用于定位 bucket                       │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  探测流程:                                                                   │
│  1. 计算 hash 值                                                            │
│  2. 使用 index = hash & bitmask 定位 bucket                                 │
│  3. 比较 salt（快速排除不匹配）                                             │
│  4. 只有 salt 匹配时才进行完整键比较                                        │
│                                                                             │
│  优势:                                                                       │
│  - Salt 比较非常快（单个整数比较）                                          │
│  - 大多数不匹配可以通过 salt 快速排除                                       │
│  - 减少昂贵的完整键比较次数                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.4 Perfect Hash Join - 完美哈希优化

当 Join 键满足特定条件时，可以使用 Perfect Hash Join 实现 O(1) 查找。

### 6.4.1 适用条件

```cpp
// src/execution/operator/join/perfect_hash_join_executor.cpp
bool PerfectHashJoinExecutor::CanDoPerfectHashJoin(const PhysicalHashJoin &op,
                                                    const Value &min, const Value &max) {
    // 条件 1: 只有一个等值 Join 键
    if (op.conditions.size() != 1) return false;

    // 条件 2: 键类型是整数类型
    if (!TypeIsIntegral(key_type.InternalType())) return false;

    // 条件 3: 有有效的 min/max 统计
    if (min.IsNull() || max.IsNull()) return false;

    // 条件 4: 值域范围足够小
    auto build_range = (max - min).GetValue<uint64_t>() + 1;
    if (build_range > 10 * ht.Count()) return false;  // 密度检查
    if (build_range > MAX_PERFECT_HASH_SIZE) return false;  // 大小限制

    // 保存统计信息
    perfect_join_statistics.build_min = min;
    perfect_join_statistics.build_max = max;
    perfect_join_statistics.build_range = build_range;
    perfect_join_statistics.is_build_dense = (build_range == ht.Count());

    return true;
}
```

### 6.4.2 实现原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Perfect Hash Join 原理                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  场景: SELECT * FROM orders o JOIN customers c ON o.customer_id = c.id     │
│        customers.id 范围: [1, 1000]                                         │
│                                                                             │
│  普通 Hash Join:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 计算 hash(customer_id)                                            │   │
│  │ 2. 定位 bucket: hash & bitmask                                       │   │
│  │ 3. 遍历链表比较键                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Perfect Hash Join:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Perfect Hash Table (数组):                                           │   │
│  │ ┌───────┬───────┬───────┬───────┬───────┐                            │   │
│  │ │ [0]   │ [1]   │ [2]   │ [3]   │ ...   │  索引 = customer_id - min  │   │
│  │ │ NULL  │ Row1  │ Row2  │ Row3  │       │                            │   │
│  │ └───────┴───────┴───────┴───────┴───────┘                            │   │
│  │                                                                       │   │
│  │ 查找: perfect_hash_table[customer_id - 1] (O(1))                     │   │
│  │                                                                       │   │
│  │ 构建 Bitmap:                                                          │   │
│  │ ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                            │   │
│  │ │ 0 │ 1 │ 1 │ 1 │ 0 │ 1 │ 0 │ 1 │ 1 │...│  标记哪些位置有值         │   │
│  │ └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  优势:                                                                       │
│  - 无需计算 hash 值                                                         │
│  - 无链表遍历                                                               │
│  - 直接数组索引访问                                                         │
│  - 缓存友好                                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.5 External Hash Join - 外部哈希连接

当数据量超过内存限制时，使用外部 Hash Join。

### 6.5.1 分区策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      External Hash Join 分区策略                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  初始分区数: 2^INITIAL_RADIX_BITS = 16                                      │
│                                                                             │
│  Build 侧分区:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 输入数据 ──▶ Hash ──▶ 分区                                           │   │
│  │                                                                       │   │
│  │ Partition 0: [rows with hash & 0xF == 0]                             │   │
│  │ Partition 1: [rows with hash & 0xF == 1]                             │   │
│  │ ...                                                                   │   │
│  │ Partition 15: [rows with hash & 0xF == 15]                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  动态重分区:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ if (max_partition_size > memory_reservation) {                       │   │
│  │     // 增加 radix bits，细化分区                                     │   │
│  │     SetRepartitionRadixBits(...)                                      │   │
│  │     // 例如: 4 bits → 6 bits，分区数 16 → 64                         │   │
│  │                                                                       │   │
│  │     // 并行重分区任务                                                 │   │
│  │     HashJoinRepartitionEvent → HashJoinRepartitionTask               │   │
│  │ }                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  分批处理:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Round 1: 处理 Partition 0-3（适合内存）                              │   │
│  │   - Build: 构建这些分区的 Hash Table                                 │   │
│  │   - Probe: 探测对应的 Probe 分区                                     │   │
│  │   - 输出结果                                                          │   │
│  │                                                                       │   │
│  │ Round 2: 处理 Partition 4-7                                          │   │
│  │ ...                                                                   │   │
│  │                                                                       │   │
│  │ 跟踪:                                                                 │   │
│  │ - current_partitions: 当前活跃分区的位图                             │   │
│  │ - completed_partitions: 已完成分区的位图                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.5.2 ProbeSpill 机制

```cpp
// Probe 侧数据的临时存储
struct ProbeSpill {
    // 分区后的 Probe 数据
    unique_ptr<PartitionedColumnData> global_partitions;

    // 当前轮次的数据消费者
    unique_ptr<ColumnDataConsumer> consumer;

    // 接口
    ProbeSpillLocalAppendState RegisterThread();
    void Append(DataChunk &chunk, ProbeSpillLocalAppendState &local_state);
    void Finalize();
    void PrepareNextProbe();
};
```

### 6.5.3 External Join 状态机

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   External Hash Join Source 状态机                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  enum HashJoinSourceStage {                                                 │
│      INIT,      // 初始化                                                   │
│      BUILD,     // 构建当前分区的 Hash Table                                │
│      PROBE,     // 探测当前分区                                             │
│      SCAN_HT,   // 扫描 Hash Table（Full/Right Outer）                      │
│      DONE       // 完成                                                      │
│  };                                                                         │
│                                                                             │
│  状态转换:                                                                   │
│  ┌────────┐                                                                 │
│  │  INIT  │                                                                 │
│  └────┬───┘                                                                 │
│       │ 初始化分区                                                          │
│       ▼                                                                     │
│  ┌────────┐     完成      ┌────────┐                                        │
│  │ BUILD  │──────────────▶│ PROBE  │                                        │
│  └────────┘               └────┬───┘                                        │
│       ▲                        │                                            │
│       │                        ▼ 完成                                       │
│       │                   ┌─────────┐                                       │
│       │                   │ SCAN_HT │ (如果是 Full/Right Outer)             │
│       │                   └────┬────┘                                       │
│       │                        │                                            │
│       │ 还有分区               │ 完成                                       │
│       └────────────────────────┤                                            │
│                                ▼                                            │
│                           ┌────────┐                                        │
│                           │  DONE  │                                        │
│                           └────────┘                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.6 PhysicalPiecewiseMergeJoin - 归并连接

用于范围条件（<, >, <=, >=）的 Join。

### 6.6.1 适用场景

```sql
-- Merge Join 适用于这类范围条件
SELECT * FROM t1, t2 WHERE t1.a < t2.b;
SELECT * FROM t1, t2 WHERE t1.a >= t2.b AND t1.a <= t2.c;
```

### 6.6.2 实现原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Piecewise Merge Join 原理                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  步骤 1: 排序两边数据                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ LHS (sorted):                    RHS (sorted):                       │   │
│  │ ┌───┬───┬───┬───┬───┐          ┌───┬───┬───┬───┬───┐                │   │
│  │ │ 1 │ 3 │ 5 │ 7 │ 9 │          │ 2 │ 4 │ 6 │ 8 │10 │                │   │
│  │ └───┴───┴───┴───┴───┘          └───┴───┴───┴───┴───┘                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  步骤 2: 分块比较（Piecewise）                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 对于条件 LHS.a < RHS.b:                                              │   │
│  │                                                                       │   │
│  │ RHS Chunk [2,4,6,8,10] 的最大值 = 10                                 │   │
│  │                                                                       │   │
│  │ LHS 中 < 10 的值: 1, 3, 5, 7, 9 → 全部匹配这个 RHS chunk 中某些值   │   │
│  │                                                                       │   │
│  │ 精确匹配对:                                                           │   │
│  │   LHS=1 匹配 RHS=[2,4,6,8,10]                                        │   │
│  │   LHS=3 匹配 RHS=[4,6,8,10]                                          │   │
│  │   LHS=5 匹配 RHS=[6,8,10]                                            │   │
│  │   ...                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  步骤 3: 使用 Block Iterator 高效遍历                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ template <SortKeyType SORT_KEY_TYPE>                                 │   │
│  │ idx_t TemplatedMergeJoinComplexBlocks(...) {                         │   │
│  │     BLOCK_ITERATOR l_ptr(l.state);                                   │   │
│  │     BLOCK_ITERATOR r_ptr(r.state);                                   │   │
│  │                                                                       │   │
│  │     while (true) {                                                   │   │
│  │         if (MergeJoinBefore(l_ptr[l.GetIndex()],                     │   │
│  │                             r_ptr[r.GetIndex()], strict)) {          │   │
│  │             // 找到匹配                                               │   │
│  │             l.result.set_index(result_count, l.entry_idx);           │   │
│  │             r.result.set_index(result_count, r.entry_idx);           │   │
│  │             result_count++;                                          │   │
│  │             l.entry_idx++;                                           │   │
│  │         } else {                                                     │   │
│  │             // 移动右指针                                             │   │
│  │             r.entry_idx++;                                           │   │
│  │             l.entry_idx = 0;  // 重置左指针                          │   │
│  │         }                                                            │   │
│  │     }                                                                │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.6.3 SortKeyType 优化

```cpp
// 根据排序键类型选择最优实现
enum class SortKeyType {
    NO_PAYLOAD_FIXED_8,    // 8 字节固定大小键，无 payload
    NO_PAYLOAD_FIXED_16,   // 16 字节固定大小键
    NO_PAYLOAD_FIXED_24,
    NO_PAYLOAD_FIXED_32,
    NO_PAYLOAD_VARIABLE_32, // 变长键
    PAYLOAD_FIXED_16,      // 带 payload 的键
    PAYLOAD_FIXED_24,
    PAYLOAD_FIXED_32,
    PAYLOAD_VARIABLE_32,
};
```

---

## 6.7 PhysicalNestedLoopJoin - 嵌套循环连接

最简单但也最慢的 Join 实现，用于复杂条件或小表。

### 6.7.1 适用条件

```cpp
// src/execution/operator/join/physical_nested_loop_join.cpp
bool PhysicalNestedLoopJoin::IsSupported(const vector<JoinCondition> &conditions,
                                          JoinType join_type) {
    if (join_type == JoinType::MARK) {
        return true;
    }
    for (auto &cond : conditions) {
        // 不支持复杂类型
        if (cond.left->return_type.InternalType() == PhysicalType::STRUCT ||
            cond.left->return_type.InternalType() == PhysicalType::LIST ||
            cond.left->return_type.InternalType() == PhysicalType::ARRAY) {
            return false;
        }
    }
    // SEMI/ANTI 只支持单条件（否则使用 Blockwise NL）
    if (join_type == JoinType::SEMI || join_type == JoinType::ANTI) {
        return conditions.size() == 1;
    }
    return true;
}
```

### 6.7.2 简单 Join vs 复杂 Join

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Nested Loop Join 两种模式                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  简单 Join (MARK, SEMI, ANTI):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ - 只需要知道是否存在匹配，不需要具体匹配行                           │   │
│  │ - 使用 bool found_match[STANDARD_VECTOR_SIZE]                        │   │
│  │                                                                       │   │
│  │ void ResolveSimpleJoin(...) {                                        │   │
│  │     // 计算左侧 Join 键                                               │   │
│  │     lhs_executor.Execute(input, left_condition);                     │   │
│  │                                                                       │   │
│  │     // 扫描右侧所有数据，标记匹配                                    │   │
│  │     bool found_match[STANDARD_VECTOR_SIZE] = {false};                │   │
│  │     NestedLoopJoinMark::Perform(left_condition, right_condition_data,│   │
│  │                                  found_match, conditions);           │   │
│  │                                                                       │   │
│  │     // 根据 found_match 构建结果                                     │   │
│  │     switch (join_type) {                                             │   │
│  │         case MARK: ConstructMarkJoinResult(...);                     │   │
│  │         case SEMI: ConstructSemiJoinResult(...);                     │   │
│  │         case ANTI: ConstructAntiJoinResult(...);                     │   │
│  │     }                                                                │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  复杂 Join (INNER, LEFT, RIGHT, OUTER):                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ - 需要输出所有匹配对                                                  │   │
│  │ - 可能产生多个输出 Chunk                                             │   │
│  │                                                                       │   │
│  │ OperatorResultType ResolveComplexJoin(...) {                         │   │
│  │     do {                                                             │   │
│  │         // 遍历左右两侧的 Chunk                                      │   │
│  │         if (state.fetch_next_right) {                                │   │
│  │             // 获取下一个右侧 Chunk                                  │   │
│  │             right_condition_data.Scan(state.condition_scan_state,    │   │
│  │                                        state.right_condition);       │   │
│  │         }                                                            │   │
│  │                                                                       │   │
│  │         // 嵌套循环匹配                                               │   │
│  │         match_count = NestedLoopJoinInner::Perform(                  │   │
│  │             state.left_tuple, state.right_tuple,                     │   │
│  │             left_condition, right_condition,                         │   │
│  │             lvector, rvector, conditions);                           │   │
│  │                                                                       │   │
│  │         if (match_count > 0) {                                       │   │
│  │             // 构建匹配结果                                           │   │
│  │             chunk.Slice(input, lvector, match_count);                │   │
│  │             chunk.Slice(right_payload, rvector, match_count, ...);   │   │
│  │         }                                                            │   │
│  │     } while (match_count == 0);                                      │   │
│  │                                                                       │   │
│  │     return HAVE_MORE_OUTPUT / NEED_MORE_INPUT;                       │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.8 PhysicalCrossProduct - 笛卡尔积

用于无条件的交叉连接。

### 6.8.1 实现策略

```cpp
// src/execution/operator/join/physical_cross_product.cpp
OperatorResultType CrossProductExecutor::Execute(DataChunk &input, DataChunk &output) {
    if (rhs.Count() == 0) {
        return OperatorResultType::FINISHED;
    }

    if (!NextValue(input, output)) {
        initialized = false;
        return OperatorResultType::NEED_MORE_INPUT;
    }

    // 选择较大的 Chunk 作为常量引用
    // 这样可以输出更大的结果 Chunk
    scan_input_chunk = input.size() < scan_chunk.size();

    // 常量 Chunk: 直接引用所有列
    auto &constant_chunk = scan_input_chunk ? scan_chunk : input;
    for (idx_t i = 0; i < constant_chunk.ColumnCount(); i++) {
        output.data[col_offset + i].Reference(constant_chunk.data[i]);
    }

    // 扫描 Chunk: 逐值引用
    auto &scan = scan_input_chunk ? input : scan_chunk;
    for (idx_t i = 0; i < scan.ColumnCount(); i++) {
        ConstantVector::Reference(output.data[col_offset + i],
                                   scan.data[i],
                                   position_in_chunk,
                                   scan.size());
    }

    return OperatorResultType::HAVE_MORE_OUTPUT;
}
```

---

## 6.9 动态 Join Filter

DuckDB 支持在 Join 执行期间生成并下推过滤器。

### 6.9.1 过滤器类型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      动态 Join Filter 类型                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. Min/Max Filter:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 收集 Build 侧 Join 键的 Min/Max 统计                                 │   │
│  │ 生成 probe_col >= min AND probe_col <= max 的过滤器                 │   │
│  │                                                                       │   │
│  │ 示例: Build 侧 customer_id 范围 [100, 500]                           │   │
│  │      → 下推 orders.customer_id >= 100 AND orders.customer_id <= 500 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. IN Filter:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 当 Build 侧数据量较小时，生成 IN-list 过滤器                         │   │
│  │                                                                       │   │
│  │ 条件:                                                                 │   │
│  │ - ht.Count() > 1 && ht.Count() <= dynamic_or_filter_threshold        │   │
│  │ - 等值连接                                                            │   │
│  │ - 值域不是连续的（否则 Min/Max 更高效）                              │   │
│  │                                                                       │   │
│  │ 示例: Build 侧 customer_id = [101, 203, 507]                         │   │
│  │      → 下推 orders.customer_id IN (101, 203, 507)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. Bloom Filter:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 概率性过滤器，可能有假阳性但无假阴性                                 │   │
│  │                                                                       │   │
│  │ 条件:                                                                 │   │
│  │ - 单键等值连接                                                        │   │
│  │ - 非 Perfect Hash Table                                               │   │
│  │ - probe_cardinality > build_cardinality                               │   │
│  │ - Build 侧有过滤条件                                                  │   │
│  │                                                                       │   │
│  │ 实现:                                                                 │   │
│  │ - BFTableFilter 封装 BloomFilter                                     │   │
│  │ - SelectivityOptionalFilter 包装（根据选择性决定是否应用）           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 6.9.2 Filter 下推实现

```cpp
// src/execution/operator/join/physical_hash_join.cpp
unique_ptr<DataChunk> JoinFilterPushdownInfo::FinalizeFilters(
    ClientContext &context,
    optional_ptr<JoinHashTable> ht,
    const PhysicalComparisonJoin &op,
    unique_ptr<DataChunk> final_min_max,
    const bool is_perfect_hashtable) const {

    for (idx_t filter_idx = 0; filter_idx < join_condition.size(); filter_idx++) {
        auto min_val = final_min_max->data[min_idx].GetValue(0);
        auto max_val = final_min_max->data[max_idx].GetValue(0);

        for (auto &info : probe_info) {
            auto filter_col_idx = info.columns[filter_idx].probe_column_index.column_index;

            // 尝试 IN Filter
            if (CanUseInFilter(context, ht, cmp)) {
                PushInFilter(info, *ht, op, filter_idx, filter_col_idx);
            }

            if (Value::NotDistinctFrom(min_val, max_val)) {
                // 单值: 生成等值过滤器
                auto constant_filter = make_uniq<ConstantFilter>(cmp, std::move(min_val));
                info.dynamic_filters->PushFilter(op, filter_col_idx, std::move(constant_filter));
            } else {
                // 范围: 生成 >= min 和 <= max 过滤器
                auto greater_equals = make_uniq<ConstantFilter>(
                    ExpressionType::COMPARE_GREATERTHANOREQUALTO, std::move(min_val));
                auto optional_ge = make_uniq<SelectivityOptionalFilter>(
                    std::move(greater_equals), ...);
                info.dynamic_filters->PushFilter(op, filter_col_idx, std::move(optional_ge));

                auto less_equals = make_uniq<ConstantFilter>(
                    ExpressionType::COMPARE_LESSTHANOREQUALTO, std::move(max_val));
                auto optional_le = make_uniq<SelectivityOptionalFilter>(
                    std::move(less_equals), ...);
                info.dynamic_filters->PushFilter(op, filter_col_idx, std::move(optional_le));

                // 尝试 Bloom Filter
                if (CanUseBloomFilter(context, ht, op, cmp, is_perfect_hashtable)) {
                    PushBloomFilter(info, *ht, op, filter_col_idx);
                }
            }
        }
    }
    return final_min_max;
}
```

---

## 6.10 Outer Join 处理

### 6.10.1 OuterJoinMarker

```cpp
// src/include/duckdb/execution/operator/join/outer_join_marker.hpp
class OuterJoinMarker {
public:
    explicit OuterJoinMarker(bool enabled);

    // 初始化匹配标记数组
    void Initialize(idx_t count);

    // 设置单个匹配
    void SetMatch(idx_t position);

    // 批量设置匹配
    void SetMatches(const SelectionVector &sel, idx_t count, idx_t base_idx = 0);

    // 构建 LEFT Join 的未匹配行结果
    void ConstructLeftJoinResult(DataChunk &left, DataChunk &result);

    // 扫描 RIGHT/FULL Outer Join 的未匹配行
    void Scan(OuterJoinGlobalScanState &gstate,
              OuterJoinLocalScanState &lstate,
              DataChunk &result);

private:
    bool enabled;
    unsafe_unique_array<bool> found_match;  // 匹配标记数组
    idx_t count;
};
```

### 6.10.2 工作流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Outer Join 处理流程                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LEFT OUTER JOIN:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 探测阶段: 记录 LHS 中哪些行找到了匹配                             │   │
│  │    left_outer.SetMatch(left_position);                               │   │
│  │                                                                       │   │
│  │ 2. 输入 Chunk 处理完毕时:                                            │   │
│  │    left_outer.ConstructLeftJoinResult(input, chunk);                 │   │
│  │    // 输出 LHS 中未匹配的行，RHS 列填充 NULL                         │   │
│  │                                                                       │   │
│  │ 3. 重置标记用于下一个 Chunk:                                         │   │
│  │    left_outer.Reset();                                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RIGHT/FULL OUTER JOIN:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Build 阶段: 初始化全局匹配标记数组                                │   │
│  │    right_outer.Initialize(rhs_count);                                │   │
│  │                                                                       │   │
│  │ 2. Probe 阶段: 记录 RHS 中哪些行被匹配                               │   │
│  │    right_outer.SetMatches(rvector, match_count, base_idx);           │   │
│  │                                                                       │   │
│  │ 3. Source 阶段: 扫描未匹配的 RHS 行                                  │   │
│  │    while (right_outer.Scan(gstate, lstate, chunk)) {                 │   │
│  │        // 输出 RHS 中未匹配的行，LHS 列填充 NULL                     │   │
│  │        for (found_match[i] == false) {                               │   │
│  │            emit(NULL..., rhs_row[i]);                                │   │
│  │        }                                                              │   │
│  │    }                                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 6.11 性能优化总结

### 6.11.1 Join 算子选择指南

| 条件特征 | 推荐算子 | 原因 |
|---------|---------|------|
| 等值连接 + 大表 | Hash Join | O(n+m) 时间复杂度 |
| 等值连接 + 小值域整数键 | Perfect Hash Join | O(1) 查找 |
| 范围条件 (<, >, <=, >=) | Merge Join | 利用排序避免全量扫描 |
| 复杂条件 + 小表 | Nested Loop | 支持任意条件 |
| 无条件 | Cross Product | 简单直接 |
| 时间序列最近匹配 | AsOf Join | 专门优化 |

### 6.11.2 Hash Join 优化技术

1. **Salt 优化**: 大表使用 salt 快速排除不匹配
2. **Perfect Hash**: 小值域整数键使用直接数组索引
3. **并行 Finalize**: 大表并行构建 Hash Table
4. **外部 Join**: 内存不足时分区处理
5. **动态过滤**: 生成 Min/Max、IN、Bloom Filter 下推

### 6.11.3 Build 侧选择

```
优化原则:
- 选择较小的表作为 Build 侧
- 考虑选择性后的有效大小
- 考虑统计信息可用性（Perfect Hash 需要统计）

优化器: BuildProbeSideOptimizer
```

---

## 6.12 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| PhysicalJoin | `src/execution/operator/join/physical_join.cpp` |
| PhysicalHashJoin | `src/execution/operator/join/physical_hash_join.cpp` |
| JoinHashTable | `src/execution/join_hashtable.cpp` |
| PerfectHashJoinExecutor | `src/execution/operator/join/perfect_hash_join_executor.cpp` |
| PhysicalPiecewiseMergeJoin | `src/execution/operator/join/physical_piecewise_merge_join.cpp` |
| PhysicalNestedLoopJoin | `src/execution/operator/join/physical_nested_loop_join.cpp` |
| PhysicalCrossProduct | `src/execution/operator/join/physical_cross_product.cpp` |
| JoinFilterPushdown | `src/include/duckdb/execution/operator/join/join_filter_pushdown.hpp` |
| OuterJoinMarker | `src/execution/operator/join/outer_join_marker.cpp` |

---

## 6.13 本章小结

本章深入分析了 DuckDB 的 Join 算子实现：

1. **PhysicalHashJoin** 是最常用的 Join 实现，支持并行 Build/Probe、Perfect Hash 优化和外部 Join
2. **JoinHashTable** 使用链式冲突处理，支持 Salt 优化和动态重分区
3. **PhysicalPiecewiseMergeJoin** 用于范围条件，基于排序的分块比较
4. **PhysicalNestedLoopJoin** 用于复杂条件或小表，支持简单和复杂两种模式
5. **动态过滤** 可以生成 Min/Max、IN 和 Bloom Filter 下推到 Probe 侧扫描
6. **OuterJoinMarker** 统一处理 LEFT/RIGHT/FULL Outer Join 的未匹配行输出

下一章将介绍聚合与分组算子的实现。
