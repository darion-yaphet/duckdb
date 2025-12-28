# DuckDB 执行引擎深度解析：第七章 - 聚合与分组算子

## 7.1 聚合算子概述

聚合（Aggregation）是数据库查询处理中最常见的操作之一。DuckDB 实现了多种聚合算子，针对不同场景进行优化。

### 7.1.1 聚合类型分类

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DuckDB 聚合类型体系                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  按分组方式分类:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 分组聚合 (Grouped Aggregate)                                        │   │
│  │ - SELECT city, COUNT(*) FROM users GROUP BY city                    │   │
│  │ - 每个分组产生一行结果                                               │   │
│  │                                                                       │   │
│  │ 无分组聚合 (Ungrouped/Scalar Aggregate)                              │   │
│  │ - SELECT COUNT(*), SUM(amount) FROM orders                          │   │
│  │ - 整个表产生一行结果                                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  按 DISTINCT 修饰符分类:                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 普通聚合 (Non-Distinct)                                              │   │
│  │ - COUNT(x), SUM(x), AVG(x)                                          │   │
│  │ - 直接聚合所有值                                                     │   │
│  │                                                                       │   │
│  │ 去重聚合 (Distinct)                                                   │   │
│  │ - COUNT(DISTINCT x), SUM(DISTINCT x)                                │   │
│  │ - 先去重再聚合                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  特殊聚合类型:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ GROUPING SETS                                                        │   │
│  │ - 多个分组集的联合聚合                                               │   │
│  │ - GROUP BY GROUPING SETS ((a), (b), (a, b))                         │   │
│  │                                                                       │   │
│  │ CUBE / ROLLUP                                                        │   │
│  │ - GROUPING SETS 的语法糖                                             │   │
│  │ - GROUP BY CUBE(a, b) → 所有子集组合                                │   │
│  │ - GROUP BY ROLLUP(a, b) → 层次聚合                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.1.2 物理算子类型

| 物理算子 | 适用场景 | 特点 |
|---------|---------|------|
| `PhysicalHashAggregate` | 通用分组聚合 | 使用 Hash Table 分组 |
| `PhysicalPerfectHashAggregate` | 小整数分组键 | 直接数组索引，O(1) |
| `PhysicalUngroupedAggregate` | 无分组聚合 | 单一聚合状态 |
| `PhysicalWindow` | 窗口函数 | 支持 PARTITION BY + ORDER BY |
| `PhysicalStreamingWindow` | 流式窗口 | 无需全量物化 |

---

## 7.2 AggregateFunction - 聚合函数接口

### 7.2.1 核心回调函数

```cpp
// src/include/duckdb/function/aggregate_function.hpp
class AggregateFunction : public BaseScalarFunction {
public:
    // 状态大小回调: 返回聚合状态的字节数
    aggregate_size_t state_size;         // idx_t (*)(const AggregateFunction &)

    // 状态初始化回调: 初始化聚合状态
    aggregate_initialize_t initialize;   // void (*)(const AggregateFunction &, data_ptr_t state)

    // 状态更新回调: 将新值添加到聚合状态（用于 Hash Table）
    aggregate_update_t update;           // void (*)(Vector[], AggregateInputData&, idx_t, Vector&, idx_t)

    // 状态合并回调: 合并两个聚合状态（用于并行聚合）
    aggregate_combine_t combine;         // void (*)(Vector&, Vector&, AggregateInputData&, idx_t)

    // 状态最终化回调: 从聚合状态生成最终结果
    aggregate_finalize_t finalize;       // void (*)(Vector&, AggregateInputData&, Vector&, idx_t, idx_t)

    // 简单更新回调: 直接更新单一状态（用于无分组聚合）
    aggregate_simple_update_t simple_update;

    // 窗口函数回调: 自定义窗口计算逻辑
    aggregate_window_t window;

    // 状态析构回调: 释放聚合状态资源
    aggregate_destructor_t destructor;
};
```

### 7.2.2 聚合状态生命周期

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       聚合状态生命周期                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 初始化 (Initialize)                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ state_size() → 分配内存                                              │   │
│  │ initialize(state) → 初始化状态                                       │   │
│  │                                                                       │   │
│  │ 示例 (SUM):                                                           │   │
│  │ struct SumState { double sum; bool isset; }                          │   │
│  │ Initialize: sum = 0, isset = false                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  2. 更新 (Update) - 多次调用                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ // 分组聚合: 批量更新多个状态                                         │   │
│  │ update(inputs, aggr_input_data, input_count, states, count)          │   │
│  │                                                                       │   │
│  │ // 无分组聚合: 更新单一状态                                           │   │
│  │ simple_update(inputs, aggr_input_data, input_count, state, count)    │   │
│  │                                                                       │   │
│  │ 示例 (SUM):                                                           │   │
│  │ sum += value; isset = true;                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  3. 合并 (Combine) - 并行聚合时使用                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ combine(source_state, target_state, aggr_input_data, count)          │   │
│  │                                                                       │   │
│  │ 示例 (SUM):                                                           │   │
│  │ target.sum += source.sum;                                            │   │
│  │ target.isset = target.isset || source.isset;                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  4. 最终化 (Finalize)                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ finalize(states, aggr_input_data, result, count, offset)             │   │
│  │                                                                       │   │
│  │ 示例 (SUM):                                                           │   │
│  │ result[i] = isset ? sum : NULL;                                      │   │
│  │                                                                       │   │
│  │ 示例 (AVG):                                                           │   │
│  │ result[i] = count > 0 ? sum / count : NULL;                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  5. 析构 (Destroy) - 可选                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ destructor(states, aggr_input_data, count)                           │   │
│  │                                                                       │   │
│  │ 用于复杂状态（如 STRING_AGG 需要释放字符串内存）                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.3 PhysicalHashAggregate - 哈希分组聚合

### 7.3.1 类结构

```cpp
// src/include/duckdb/execution/operator/aggregate/physical_hash_aggregate.hpp
class PhysicalHashAggregate : public PhysicalOperator {
public:
    //! 分组聚合数据
    GroupedAggregateData grouped_aggregate_data;

    //! 分组集（用于 GROUPING SETS/CUBE/ROLLUP）
    vector<GroupingSet> grouping_sets;

    //! 每个分组集的 Radix Hash Table
    vector<HashAggregateGroupingData> groupings;

    //! DISTINCT 聚合信息
    unique_ptr<DistinctAggregateCollectionInfo> distinct_collection_info;
};

struct GroupedAggregateData {
    //! 分组表达式
    vector<unique_ptr<Expression>> groups;
    //! 分组列类型
    vector<LogicalType> group_types;
    //! 聚合表达式
    vector<unique_ptr<Expression>> aggregates;
    //! 聚合输入类型
    vector<LogicalType> payload_types;
    //! 聚合返回类型
    vector<LogicalType> aggregate_return_types;
};
```

### 7.3.2 执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PhysicalHashAggregate 执行流程                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Pipeline 角色: Sink + Source                                               │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Sink 阶段: 接收数据并更新聚合状态                                     │   │
│  │                                                                       │   │
│  │ 输入 Chunk:                                                           │   │
│  │ ┌────────┬────────┬────────┐                                          │   │
│  │ │ city   │ amount │ count  │                                          │   │
│  │ ├────────┼────────┼────────┤                                          │   │
│  │ │ Beijing│  100   │   1    │                                          │   │
│  │ │ Beijing│  200   │   1    │                                          │   │
│  │ │Shanghai│  150   │   1    │                                          │   │
│  │ └────────┴────────┴────────┘                                          │   │
│  │                                                                       │   │
│  │ 1. 提取分组键: [Beijing, Beijing, Shanghai]                           │   │
│  │ 2. 提取聚合输入: [amount, count] 列                                   │   │
│  │ 3. 调用 RadixPartitionedHashTable.Sink()                             │   │
│  │    → 计算 Hash → 找到/创建分组 → 更新聚合状态                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼ 所有数据处理完毕                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Combine 阶段: 合并线程本地 Hash Table                                 │   │
│  │                                                                       │   │
│  │ Thread 1 HT: {Beijing: (sum=100, cnt=1)}                             │   │
│  │ Thread 2 HT: {Beijing: (sum=200, cnt=1), Shanghai: (sum=150, cnt=1)} │   │
│  │                                                                       │   │
│  │           ↓ Combine                                                   │   │
│  │                                                                       │   │
│  │ Global HT: {Beijing: (sum=300, cnt=2), Shanghai: (sum=150, cnt=1)}   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Finalize 阶段: 最终化 Hash Table                                      │   │
│  │                                                                       │   │
│  │ - 合并所有分区                                                        │   │
│  │ - 处理 DISTINCT 聚合（如果有）                                        │   │
│  │ - 准备扫描                                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Source 阶段: 扫描 Hash Table 输出结果                                 │   │
│  │                                                                       │   │
│  │ 输出 Chunk:                                                           │   │
│  │ ┌────────┬────────────┬────────────┐                                  │   │
│  │ │ city   │ SUM(amount)│ COUNT(*)   │                                  │   │
│  │ ├────────┼────────────┼────────────┤                                  │   │
│  │ │ Beijing│    300     │     2      │                                  │   │
│  │ │Shanghai│    150     │     1      │                                  │   │
│  │ └────────┴────────────┴────────────┘                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.3.3 Sink 实现

```cpp
// src/execution/operator/aggregate/physical_hash_aggregate.cpp
SinkResultType PhysicalHashAggregate::Sink(ExecutionContext &context, DataChunk &chunk,
                                           OperatorSinkInput &input) const {
    auto &local_state = input.local_state.Cast<HashAggregateLocalSinkState>();
    auto &global_state = input.global_state.Cast<HashAggregateGlobalSinkState>();

    // 1. 处理 DISTINCT 聚合（先收集到独立的 Hash Table）
    if (distinct_collection_info) {
        SinkDistinct(context, chunk, input);
    }

    // 2. 如果只有 DISTINCT 聚合，可以跳过主 Hash Table
    if (CanSkipRegularSink()) {
        return SinkResultType::NEED_MORE_INPUT;
    }

    // 3. 准备聚合输入 Chunk
    DataChunk &aggregate_input_chunk = local_state.aggregate_input_chunk;
    idx_t aggregate_input_idx = 0;
    for (auto &aggregate : grouped_aggregate_data.aggregates) {
        auto &aggr = aggregate->Cast<BoundAggregateExpression>();
        for (auto &child_expr : aggr.children) {
            auto &bound_ref = child_expr->Cast<BoundReferenceExpression>();
            aggregate_input_chunk.data[aggregate_input_idx++].Reference(chunk.data[bound_ref.index]);
        }
    }
    aggregate_input_chunk.SetCardinality(chunk.size());

    // 4. 对每个 Grouping Set 执行 Sink
    for (idx_t i = 0; i < groupings.size(); i++) {
        auto &grouping = groupings[i];
        auto &table = grouping.table_data;
        OperatorSinkInput sink_input{*grouping_global_state.table_state,
                                      *grouping_local_state.table_state, interrupt_state};
        table.Sink(context, chunk, sink_input, aggregate_input_chunk, non_distinct_filter);
    }

    return SinkResultType::NEED_MORE_INPUT;
}
```

### 7.3.4 RadixPartitionedHashTable

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    RadixPartitionedHashTable 结构                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  目的: 支持并行聚合和外部聚合（内存不足时）                                 │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 分区策略                                                             │   │
│  │                                                                       │   │
│  │ Hash 值结构:                                                          │   │
│  │ ┌────────────────────────────────────────────────────────────────┐   │   │
│  │ │ [     Radix Bits (高位)     ][      Hash Bits (低位)      ]   │   │   │
│  │ │        用于分区                        用于 Hash Table        │   │   │
│  │ └────────────────────────────────────────────────────────────────┘   │   │
│  │                                                                       │   │
│  │ 分区数 = 2^radix_bits                                                 │   │
│  │ 默认: 1-2 bits (2-4 个分区)                                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  内存布局:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Partition 0                   Partition 1                           │   │
│  │ ┌─────────────────────────┐   ┌─────────────────────────┐           │   │
│  │ │ GroupedAggregateHashTable│   │ GroupedAggregateHashTable│          │   │
│  │ │ ┌─────────────────────┐ │   │ ┌─────────────────────┐ │           │   │
│  │ │ │ entries (pointers)  │ │   │ │ entries (pointers)  │ │           │   │
│  │ │ └─────────────────────┘ │   │ └─────────────────────┘ │           │   │
│  │ │ ┌─────────────────────┐ │   │ ┌─────────────────────┐ │           │   │
│  │ │ │ TupleDataCollection │ │   │ │ TupleDataCollection │ │           │   │
│  │ │ │ [Group1|AggState1]  │ │   │ │ [Group3|AggState3]  │ │           │   │
│  │ │ │ [Group2|AggState2]  │ │   │ │ [Group4|AggState4]  │ │           │   │
│  │ │ └─────────────────────┘ │   │ └─────────────────────┘ │           │   │
│  │ └─────────────────────────┘   └─────────────────────────┘           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  行格式 (TupleDataLayout):                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ [Validity][Group1][Group2]...[AggState1][AggState2]...[RowID/Next]  │   │
│  │                                                                       │   │
│  │ - Validity: NULL 位图                                                 │   │
│  │ - GroupN: 分组键值                                                    │   │
│  │ - AggStateN: 聚合状态（由 state_size 确定大小）                       │   │
│  │ - RowID/Next: 用于链表冲突处理                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.4 PhysicalPerfectHashAggregate - 完美哈希聚合

### 7.4.1 适用条件

```cpp
// 优化器决定是否使用 Perfect Hash Aggregate
bool UsesPerfectHashAggregate(const vector<unique_ptr<BaseStatistics>> &group_stats) {
    // 条件 1: 所有分组键都是整数类型
    // 条件 2: 所有分组键都有 min/max 统计信息
    // 条件 3: 值域范围不太大 (range < threshold)
    // 条件 4: 没有 GROUPING SETS
    // 条件 5: 没有 DISTINCT 聚合

    for (auto &stats : group_stats) {
        if (!stats || !stats->HasMinMax()) return false;
        auto range = stats->GetMax() - stats->GetMin();
        if (range > PERFECT_HT_THRESHOLD) return false;
    }
    return true;
}
```

### 7.4.2 实现原理

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Perfect Hash Aggregate 原理                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  场景: SELECT age, COUNT(*) FROM users GROUP BY age                         │
│        age 范围: [18, 65]                                                   │
│                                                                             │
│  普通 Hash Aggregate:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. 计算 hash(age)                                                    │   │
│  │ 2. 定位 bucket: hash & bitmask                                       │   │
│  │ 3. 遍历链表比较键                                                    │   │
│  │ 4. 更新聚合状态                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Perfect Hash Aggregate:                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 直接数组索引:                                                         │   │
│  │                                                                       │   │
│  │ AggregateState states[65 - 18 + 1];  // 48 个槽位                    │   │
│  │                                                                       │   │
│  │ 索引 = age - 18                                                       │   │
│  │                                                                       │   │
│  │ ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐                          │   │
│  │ │ [0] │ [1] │ [2] │ ... │[46] │[47] │     │                          │   │
│  │ │age18│age19│age20│     │age64│age65│     │                          │   │
│  │ │cnt=5│cnt=3│cnt=8│     │cnt=2│cnt=1│     │                          │   │
│  │ └─────┴─────┴─────┴─────┴─────┴─────┴─────┘                          │   │
│  │                                                                       │   │
│  │ 查找/更新: states[age - 18].count++ (O(1))                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  多列分组:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ GROUP BY year, month                                                 │   │
│  │ year: [2020, 2024], range = 5                                        │   │
│  │ month: [1, 12], range = 12                                           │   │
│  │                                                                       │   │
│  │ 总槽位数 = 5 * 12 = 60                                               │   │
│  │ 索引 = (year - 2020) * 12 + (month - 1)                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  优势:                                                                       │
│  - 无需计算 Hash                                                            │
│  - 无冲突处理                                                               │
│  - O(1) 访问                                                                │
│  - 缓存友好                                                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.5 PhysicalUngroupedAggregate - 无分组聚合

### 7.5.1 类结构

```cpp
// src/execution/operator/aggregate/physical_ungrouped_aggregate.cpp
class PhysicalUngroupedAggregate : public PhysicalOperator {
public:
    //! 聚合表达式列表
    vector<unique_ptr<Expression>> aggregates;

    //! DISTINCT 聚合信息
    unique_ptr<DistinctAggregateCollectionInfo> distinct_collection_info;
    unique_ptr<DistinctAggregateData> distinct_data;
};
```

### 7.5.2 执行流程

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  PhysicalUngroupedAggregate 执行流程                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SQL: SELECT COUNT(*), SUM(amount), AVG(price) FROM orders                  │
│                                                                             │
│  状态结构:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ UngroupedAggregateState                                              │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ aggregate_data[0]: CountState { count: int64 }                  │ │   │
│  │ │ aggregate_data[1]: SumState { sum: double, isset: bool }        │ │   │
│  │ │ aggregate_data[2]: AvgState { sum: double, count: int64 }       │ │   │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  并行执行:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Thread 1:                    Thread 2:                              │   │
│  │ LocalState {                 LocalState {                           │   │
│  │   count: 1000                  count: 2000                          │   │
│  │   sum: 50000                   sum: 80000                           │   │
│  │   avg_sum: 25000               avg_sum: 40000                       │   │
│  │   avg_cnt: 500                 avg_cnt: 1000                        │   │
│  │ }                            }                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                            │                                      │
│         └────────────┬───────────────┘                                      │
│                      ▼ Combine                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ GlobalState {                                                        │   │
│  │   count: 3000                                                        │   │
│  │   sum: 130000                                                        │   │
│  │   avg_sum: 65000                                                     │   │
│  │   avg_cnt: 1500                                                      │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                      │                                                      │
│                      ▼ Finalize                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 结果 Chunk (1 行):                                                   │   │
│  │ ┌────────────┬────────────┬────────────┐                             │   │
│  │ │ COUNT(*)   │ SUM(amount)│ AVG(price) │                             │   │
│  │ ├────────────┼────────────┼────────────┤                             │   │
│  │ │    3000    │   130000   │   43.33    │  (65000/1500)               │   │
│  │ └────────────┴────────────┴────────────┘                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.5.3 核心实现

```cpp
// 状态更新
void LocalUngroupedAggregateState::Sink(DataChunk &payload_chunk, idx_t payload_idx, idx_t aggr_idx) {
    auto &aggregate = state.aggregate_expressions[aggr_idx]->Cast<BoundAggregateExpression>();
    idx_t payload_cnt = aggregate.children.size();
    auto start_of_input = payload_cnt == 0 ? nullptr : &payload_chunk.data[payload_idx];

    AggregateInputData aggr_input_data(state.bind_data[aggr_idx], allocator);

    // 直接调用 simple_update（不需要 Hash Table）
    aggregate.function.GetStateSimpleUpdateCallback()(
        start_of_input,
        aggr_input_data,
        payload_cnt,
        state.aggregate_data[aggr_idx].get(),
        payload_chunk.size()
    );
}

// 状态合并
void GlobalUngroupedAggregateState::Combine(LocalUngroupedAggregateState &other) {
    lock_guard<mutex> glock(lock);  // 需要加锁

    for (idx_t aggr_idx = 0; aggr_idx < state.aggregate_expressions.size(); aggr_idx++) {
        auto &aggregate = state.aggregate_expressions[aggr_idx]->Cast<BoundAggregateExpression>();

        if (aggregate.IsDistinct()) continue;  // DISTINCT 单独处理

        Vector source_state(Value::POINTER(CastPointerToValue(other.state.aggregate_data[aggr_idx].get())));
        Vector dest_state(Value::POINTER(CastPointerToValue(state.aggregate_data[aggr_idx].get())));

        AggregateInputData aggr_input_data(aggregate.bind_info.get(), allocator,
                                           AggregateCombineType::ALLOW_DESTRUCTIVE);
        aggregate.function.GetStateCombineCallback()(source_state, dest_state, aggr_input_data, 1);
    }
}
```

---

## 7.6 DISTINCT 聚合处理

### 7.6.1 处理策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       DISTINCT 聚合处理策略                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SQL: SELECT COUNT(DISTINCT city), SUM(DISTINCT amount) FROM orders        │
│                                                                             │
│  策略: 为每个 DISTINCT 聚合创建独立的 Hash Table                            │
│                                                                             │
│  执行流程:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 1: 收集去重值                                                  │   │
│  │                                                                       │   │
│  │ 输入数据:                                                             │   │
│  │ ┌────────┬────────┐                                                   │   │
│  │ │ city   │ amount │                                                   │   │
│  │ ├────────┼────────┤                                                   │   │
│  │ │ Beijing│  100   │                                                   │   │
│  │ │ Beijing│  100   │  ← 重复                                          │   │
│  │ │Shanghai│  200   │                                                   │   │
│  │ │ Beijing│  200   │                                                   │   │
│  │ └────────┴────────┘                                                   │   │
│  │                                                                       │   │
│  │ Distinct HT 1 (for city):     Distinct HT 2 (for amount):            │   │
│  │ ┌─────────────────────┐       ┌─────────────────────┐                 │   │
│  │ │ {Beijing}           │       │ {100}               │                 │   │
│  │ │ {Shanghai}          │       │ {200}               │                 │   │
│  │ └─────────────────────┘       └─────────────────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Phase 2: 扫描去重 HT，更新聚合状态                                    │   │
│  │                                                                       │   │
│  │ Distinct HT 1 扫描:                                                   │   │
│  │   → count(Beijing) → count(Shanghai)                                 │   │
│  │   → COUNT(DISTINCT city) = 2                                         │   │
│  │                                                                       │   │
│  │ Distinct HT 2 扫描:                                                   │   │
│  │   → sum(100) → sum(200)                                              │   │
│  │   → SUM(DISTINCT amount) = 300                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  优化: 相同输入的 DISTINCT 聚合共享同一个 HT                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ COUNT(DISTINCT x) 和 SUM(DISTINCT x) 共享一个 HT                     │   │
│  │ → table_map 记录哪些聚合共享哪个表                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.6.2 事件调度

```cpp
// DISTINCT 聚合使用事件系统分阶段执行
class HashAggregateDistinctFinalizeEvent : public BasePipelineEvent {
public:
    void Schedule() override {
        // 创建并行任务扫描 Distinct HT
        idx_t n_tasks = CreateGlobalSources();
        vector<shared_ptr<Task>> tasks;
        for (idx_t i = 0; i < n_tasks; i++) {
            tasks.push_back(make_uniq<HashAggregateDistinctFinalizeTask>(...));
        }
        SetTasks(std::move(tasks));
    }

    void FinishEvent() override {
        // Distinct 处理完成后，触发主聚合 Finalize
        auto new_event = make_shared_ptr<HashAggregateFinalizeEvent>(context, pipeline.get(), op, gstate);
        this->InsertEvent(std::move(new_event));
    }
};
```

---

## 7.7 PhysicalWindow - 窗口函数

### 7.7.1 窗口函数概述

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          窗口函数类型                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  聚合窗口函数:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ SUM(x) OVER (PARTITION BY a ORDER BY b ROWS BETWEEN ...)            │   │
│  │ COUNT(*) OVER (PARTITION BY a ORDER BY b)                           │   │
│  │ AVG(x) OVER (...)                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  排名窗口函数:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ ROW_NUMBER() OVER (ORDER BY x)                                       │   │
│  │ RANK() OVER (ORDER BY x)                                             │   │
│  │ DENSE_RANK() OVER (ORDER BY x)                                       │   │
│  │ PERCENT_RANK() OVER (...)                                            │   │
│  │ NTILE(n) OVER (...)                                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  值窗口函数:                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ FIRST_VALUE(x) OVER (...)                                            │   │
│  │ LAST_VALUE(x) OVER (...)                                             │   │
│  │ NTH_VALUE(x, n) OVER (...)                                           │   │
│  │ LAG(x, offset, default) OVER (...)                                   │   │
│  │ LEAD(x, offset, default) OVER (...)                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.7.2 窗口函数执行模型

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PhysicalWindow 执行流程                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SQL: SELECT id, value,                                                     │
│             SUM(value) OVER (PARTITION BY category ORDER BY id              │
│                              ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)      │
│       FROM data                                                             │
│                                                                             │
│  Phase 1: Sink - 收集所有数据                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 输入数据按 PARTITION BY + ORDER BY 排序后物化                        │   │
│  │                                                                       │   │
│  │ ┌────┬──────────┬───────┬────────────┐                                │   │
│  │ │ id │ category │ value │ (sorted)   │                                │   │
│  │ ├────┼──────────┼───────┼────────────┤                                │   │
│  │ │ 1  │    A     │  10   │ Partition A│                                │   │
│  │ │ 2  │    A     │  20   │            │                                │   │
│  │ │ 3  │    A     │  30   │            │                                │   │
│  │ │ 1  │    B     │  15   │ Partition B│                                │   │
│  │ │ 2  │    B     │  25   │            │                                │   │
│  │ └────┴──────────┴───────┴────────────┘                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  Phase 2: Finalize - 计算窗口边界                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 分区边界: [(0, 3), (3, 5)]                                           │   │
│  │                                                                       │   │
│  │ 对于每行，计算 Frame 边界:                                            │   │
│  │ Row 0: frame = [0, 0]   (ROWS 2 PRECEDING, 只有当前行)               │   │
│  │ Row 1: frame = [0, 1]   (前一行 + 当前行)                            │   │
│  │ Row 2: frame = [0, 2]   (前两行 + 当前行)                            │   │
│  │ Row 3: frame = [3, 3]   (新分区开始，只有当前行)                     │   │
│  │ Row 4: frame = [3, 4]   (前一行 + 当前行)                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         ▼                                                                    │
│  Phase 3: Source - 计算窗口函数值                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 对于每行，根据 Frame 计算聚合:                                        │   │
│  │                                                                       │   │
│  │ Row 0: SUM(value[0:0]) = 10                                          │   │
│  │ Row 1: SUM(value[0:1]) = 10 + 20 = 30                                │   │
│  │ Row 2: SUM(value[0:2]) = 10 + 20 + 30 = 60                           │   │
│  │ Row 3: SUM(value[3:3]) = 15                                          │   │
│  │ Row 4: SUM(value[3:4]) = 15 + 25 = 40                                │   │
│  │                                                                       │   │
│  │ 输出:                                                                 │   │
│  │ ┌────┬──────────┬───────┬─────────────┐                               │   │
│  │ │ id │ category │ value │ window_sum  │                               │   │
│  │ ├────┼──────────┼───────┼─────────────┤                               │   │
│  │ │ 1  │    A     │  10   │     10      │                               │   │
│  │ │ 2  │    A     │  20   │     30      │                               │   │
│  │ │ 3  │    A     │  30   │     60      │                               │   │
│  │ │ 1  │    B     │  15   │     15      │                               │   │
│  │ │ 2  │    B     │  25   │     40      │                               │   │
│  │ └────┴──────────┴───────┴─────────────┘                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.7.3 Frame 计算优化

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      窗口 Frame 计算优化                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 增量计算 (Incremental Aggregation)                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 适用于: SUM, COUNT, AVG 等可增量更新的聚合                           │   │
│  │                                                                       │   │
│  │ ROWS BETWEEN 2 PRECEDING AND CURRENT ROW:                            │   │
│  │                                                                       │   │
│  │ Row 2: sum = Row1.sum - value[0] + value[2]                          │   │
│  │        (不需要重新计算整个 Frame)                                     │   │
│  │                                                                       │   │
│  │ 复杂度: O(1) per row (而不是 O(frame_size))                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. 分段树 (Segment Tree)                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 适用于: 任意范围查询的聚合                                            │   │
│  │                                                                       │   │
│  │           [0-7: sum=120]                                              │   │
│  │          /              \                                             │   │
│  │    [0-3: sum=60]    [4-7: sum=60]                                    │   │
│  │    /        \        /        \                                       │   │
│  │ [0-1:30] [2-3:30] [4-5:30] [6-7:30]                                  │   │
│  │                                                                       │   │
│  │ 范围查询复杂度: O(log n)                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. 自定义窗口函数                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ aggregate_window_t 回调允许自定义计算逻辑:                           │   │
│  │                                                                       │   │
│  │ void window_callback(                                                │   │
│  │     AggregateInputData &aggr_input_data,                             │   │
│  │     const WindowPartitionInput &partition,  // 分区数据              │   │
│  │     const_data_ptr_t g_state,               // 全局状态              │   │
│  │     data_ptr_t l_state,                     // 本地状态              │   │
│  │     const SubFrames &subframes,             // Frame 边界            │   │
│  │     Vector &result,                          // 结果向量              │   │
│  │     idx_t rid                                // 行索引               │   │
│  │ );                                                                   │   │
│  │                                                                       │   │
│  │ 使用场景: FIRST_VALUE, LAST_VALUE, NTH_VALUE, LAG, LEAD              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.8 GROUPING SETS / CUBE / ROLLUP

### 7.8.1 概念

```sql
-- GROUPING SETS: 多个分组集的联合
SELECT year, quarter, SUM(sales)
FROM data
GROUP BY GROUPING SETS ((year), (quarter), (year, quarter), ());

-- CUBE: 所有可能的分组组合
SELECT year, quarter, SUM(sales)
FROM data
GROUP BY CUBE(year, quarter);
-- 等价于 GROUPING SETS ((year, quarter), (year), (quarter), ())

-- ROLLUP: 层次聚合
SELECT year, quarter, month, SUM(sales)
FROM data
GROUP BY ROLLUP(year, quarter, month);
-- 等价于 GROUPING SETS ((year, quarter, month), (year, quarter), (year), ())
```

### 7.8.2 实现策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GROUPING SETS 实现策略                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  每个 Grouping Set 对应一个独立的 RadixPartitionedHashTable:                │
│                                                                             │
│  GROUP BY GROUPING SETS ((year), (quarter), ())                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HashAggregateGroupingData[0]: (year)                                 │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ RadixPartitionedHashTable                                       │ │   │
│  │ │ Groups: {2022: sum=100}, {2023: sum=200}, {2024: sum=150}       │ │   │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HashAggregateGroupingData[1]: (quarter)                              │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ RadixPartitionedHashTable                                       │ │   │
│  │ │ Groups: {Q1: sum=120}, {Q2: sum=130}, {Q3: sum=100}, {Q4: sum=100}│ │  │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HashAggregateGroupingData[2]: ()  - 空分组集                         │   │
│  │ ┌─────────────────────────────────────────────────────────────────┐ │   │
│  │ │ RadixPartitionedHashTable                                       │ │   │
│  │ │ Groups: {(): sum=450}  - 全局聚合                               │ │   │
│  │ └─────────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GROUPING() 函数:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ GROUPING(year, quarter) 返回一个位图:                                │   │
│  │ - 位 0: year 是否在分组中 (0=是, 1=否)                               │   │
│  │ - 位 1: quarter 是否在分组中                                         │   │
│  │                                                                       │   │
│  │ Grouping Set (year, quarter): GROUPING = 0 (0b00)                    │   │
│  │ Grouping Set (year):          GROUPING = 1 (0b01)                    │   │
│  │ Grouping Set (quarter):       GROUPING = 2 (0b10)                    │   │
│  │ Grouping Set ():              GROUPING = 3 (0b11)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 7.9 性能优化总结

### 7.9.1 聚合算子选择指南

| 查询特征 | 推荐算子 | 原因 |
|---------|---------|------|
| 无 GROUP BY | `UngroupedAggregate` | 单一状态，无需 Hash Table |
| GROUP BY 小整数列 | `PerfectHashAggregate` | O(1) 数组访问 |
| GROUP BY 任意列 | `HashAggregate` | 通用 Hash Table |
| GROUPING SETS | `HashAggregate` | 多个 Hash Table |
| 窗口函数 | `Window` / `StreamingWindow` | 专门优化 |

### 7.9.2 并行聚合策略

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          并行聚合策略                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. 线程本地聚合 + 全局合并                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 每个线程维护独立的 Hash Table                                        │   │
│  │ Combine 阶段合并到全局 Hash Table                                    │   │
│  │                                                                       │   │
│  │ 优点: 无锁 Sink，高并行度                                            │   │
│  │ 缺点: 内存使用较高（每线程一个 HT）                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. 分区聚合                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 使用 Radix 分区，每个分区由一个线程处理                              │   │
│  │ 分区内无竞争                                                          │   │
│  │                                                                       │   │
│  │ 适用于: 外部聚合（内存不足时）                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. 无分组聚合优化                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 使用 simple_update 直接更新状态（不需要 Hash）                       │   │
│  │ Combine 时使用 combine 回调合并状态                                  │   │
│  │                                                                       │   │
│  │ 特别优化: COUNT(*) 使用原子计数器                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7.9.3 内存优化

1. **列裁剪**: 只物化需要的列
2. **延迟物化**: 在需要时才提取实际值
3. **分区处理**: 内存不足时分批处理分区
4. **DISTINCT 共享**: 相同输入的 DISTINCT 聚合共享 Hash Table

---

## 7.10 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| PhysicalHashAggregate | `src/execution/operator/aggregate/physical_hash_aggregate.cpp` |
| PhysicalPerfectHashAggregate | `src/execution/operator/aggregate/physical_perfecthash_aggregate.cpp` |
| PhysicalUngroupedAggregate | `src/execution/operator/aggregate/physical_ungrouped_aggregate.cpp` |
| PhysicalWindow | `src/execution/operator/aggregate/physical_window.cpp` |
| GroupedAggregateData | `src/execution/operator/aggregate/grouped_aggregate_data.cpp` |
| DistinctAggregateData | `src/execution/operator/aggregate/distinct_aggregate_data.cpp` |
| RadixPartitionedHashTable | `src/execution/radix_partitioned_hashtable.cpp` |
| AggregateFunction | `src/include/duckdb/function/aggregate_function.hpp` |
| AggregateExecutor | `src/common/vector_operations/aggregate_executor.cpp` |

---

## 7.11 本章小结

本章深入分析了 DuckDB 的聚合与分组算子实现：

1. **AggregateFunction** 定义了聚合的核心接口：state_size、initialize、update、combine、finalize
2. **PhysicalHashAggregate** 是通用分组聚合实现，使用 RadixPartitionedHashTable 支持并行和外部聚合
3. **PhysicalPerfectHashAggregate** 针对小值域整数分组键优化，使用直接数组索引
4. **PhysicalUngroupedAggregate** 处理无分组聚合，使用 simple_update 直接更新单一状态
5. **DISTINCT 聚合**使用独立 Hash Table 先去重再聚合
6. **PhysicalWindow** 实现窗口函数，支持增量计算和自定义窗口回调
7. **GROUPING SETS** 为每个分组集创建独立 Hash Table

下一章将介绍排序与 TopN 算子的实现。
