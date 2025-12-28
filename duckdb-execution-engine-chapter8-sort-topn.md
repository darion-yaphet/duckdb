# DuckDB 执行引擎深度解析：第八章 - 排序与 TopN 算子

## 8.1 章节概述

排序是数据库系统中最基础也是最复杂的操作之一。本章深入分析 DuckDB 的排序算子（PhysicalOrder）和 TopN 算子（PhysicalTopN）的实现，涵盖排序键编码、堆优化、外部排序以及动态过滤等高级特性。

```
┌────────────────────────────────────────────────────────────────────┐
│                    排序与 TopN 算子架构                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ORDER BY col1, col2         ───────►   PhysicalOrder             │
│                                              │                     │
│                                              ▼                     │
│                               ┌──────────────────────────────┐    │
│                               │          Sort 类               │    │
│                               ├──────────────────────────────┤    │
│                               │  • SortedRun (已排序片段)      │    │
│                               │  • SortedRunMerger (归并)     │    │
│                               │  • 排序键编码 (可比较)         │    │
│                               └──────────────────────────────┘    │
│                                                                    │
│   ORDER BY + LIMIT N          ───────►   PhysicalTopN             │
│                                              │                     │
│                                              ▼                     │
│                               ┌──────────────────────────────┐    │
│                               │        TopNHeap               │    │
│                               ├──────────────────────────────┤    │
│                               │  • 堆结构 (std::push_heap)    │    │
│                               │  • 边界值过滤优化             │    │
│                               │  • 动态过滤下推               │    │
│                               └──────────────────────────────┘    │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8.2 PhysicalOrder：全量排序算子

### 8.2.1 算子概述

`PhysicalOrder` 是 DuckDB 中处理 `ORDER BY` 子句的物理算子。它是一个 **Pipeline Breaker**，需要收集所有输入数据后才能输出有序结果。

```cpp
// src/include/duckdb/execution/operator/order/physical_order.hpp

class PhysicalOrder : public PhysicalOperator {
public:
    PhysicalOrder(PhysicalPlan &physical_plan, vector<LogicalType> types,
                  vector<BoundOrderByNode> orders, vector<idx_t> projections,
                  idx_t estimated_cardinality, bool is_index_sort = false);

    //! 排序键表达式和排序方向
    vector<BoundOrderByNode> orders;
    //! 输出列映射
    vector<idx_t> projections;
    //! 是否为索引排序（近似排序）
    bool is_index_sort;
};
```

### 8.2.2 Sort 类：排序核心实现

`PhysicalOrder` 的核心逻辑委托给 `Sort` 类实现。Sort 类遵循 PhysicalOperator 接口，但作为独立组件可被复用。

```cpp
// src/include/duckdb/common/sorting/sort.hpp

class Sort {
public:
    Sort(ClientContext &context, const vector<BoundOrderByNode> &orders,
         const vector<LogicalType> &input_types, vector<idx_t> projection_map,
         bool is_index_sort = false);

    //===--------------------------------------------------------------------===//
    // Sink 接口 - 收集数据
    //===--------------------------------------------------------------------===//
    unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context) const;
    unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context) const;
    SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                        OperatorSinkInput &input) const;
    SinkCombineResultType Combine(ExecutionContext &context,
                                  OperatorSinkCombineInput &input) const;
    SinkFinalizeType Finalize(ClientContext &context,
                              OperatorSinkFinalizeInput &input) const;

    //===--------------------------------------------------------------------===//
    // Source 接口 - 输出排序结果
    //===--------------------------------------------------------------------===//
    unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
                                                      GlobalSourceState &gstate) const;
    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context,
                                                        GlobalSinkState &sink) const;
    SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                             OperatorSourceInput &input) const;

private:
    //! 排序键表达式（创建可比较的排序键）
    unique_ptr<Expression> create_sort_key;
    unique_ptr<Expression> decode_sort_key;
    //! 排序键布局
    shared_ptr<TupleDataLayout> key_layout;
    //! Payload 布局（去除重复的排序键列）
    shared_ptr<TupleDataLayout> payload_layout;
    vector<idx_t> payload_projection_map;
    //! 输出列映射
    vector<SortProjectionColumn> output_projection_columns;
};
```

### 8.2.3 排序键编码

DuckDB 使用特殊的排序键编码技术，将多列排序键转换为单个可比较的二进制字符串：

```
┌────────────────────────────────────────────────────────────────────┐
│                        排序键编码原理                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   输入行:  (name: "Alice", age: 30, score: 95.5)                  │
│   排序:    ORDER BY age DESC, score ASC                           │
│                                                                    │
│                           ▼                                        │
│                                                                    │
│   ┌────────────────────────────────────────────────────────────┐  │
│   │ CreateSortKey                                               │  │
│   ├────────────────────────────────────────────────────────────┤  │
│   │  1. 对每个排序列应用编码规则:                               │  │
│   │     - ASC:  按原值编码                                      │  │
│   │     - DESC: 按位取反                                        │  │
│   │     - NULLS FIRST: NULL → 0x00                             │  │
│   │     - NULLS LAST:  NULL → 0xFF                             │  │
│   │                                                             │  │
│   │  2. 拼接成单一二进制串                                      │  │
│   └────────────────────────────────────────────────────────────┘  │
│                           ▼                                        │
│                                                                    │
│   排序键:  [0xE1 0x...] (age DESC) + [0x42 0x...] (score ASC)    │
│                                                                    │
│   优点:                                                           │
│   • 使用 memcmp 直接比较，无需逐列比较                            │
│   • 利用 CPU 缓存和 SIMD 指令加速                                 │
│   • 简化外部排序的归并逻辑                                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.2.4 SortedRun：已排序数据片段

每个线程在 Sink 阶段创建本地的 `SortedRun`：

```cpp
// src/include/duckdb/common/sorting/sorted_run.hpp

class SortedRun {
public:
    SortedRun(ClientContext &context, const Sort &sort, bool is_index_sort);

    //! 添加数据
    void Sink(DataChunk &key, DataChunk &payload);
    //! 完成排序（如果 external=true，物理重排数据）
    void Finalize(bool external);
    //! 销毁指定范围的数据（用于释放已扫描数据）
    void DestroyData(idx_t tuple_idx_begin, idx_t tuple_idx_end);
    //! 元组数量
    idx_t Count() const;
    //! 大小（字节）
    idx_t SizeInBytes() const;

public:
    ClientContext &context;
    const Sort &sort;

    //! 排序键集合
    unique_ptr<TupleDataCollection> key_data;
    //! Payload 集合
    unique_ptr<TupleDataCollection> payload_data;
    TupleDataAppendState key_append_state;
    TupleDataAppendState payload_append_state;

    //! 是否为索引排序
    const bool is_index_sort;
    //! 是否已完成排序
    bool finalized;
};
```

### 8.2.5 SortedRunMerger：多路归并

当有多个 SortedRun（来自多个线程或外部排序的多个分区）时，使用归并器合并：

```cpp
// src/include/duckdb/common/sorting/sorted_run_merger.hpp

class SortedRunMerger {
public:
    SortedRunMerger(const Sort &sort, vector<unique_ptr<SortedRun>> &&sorted_runs,
                    idx_t partition_size, bool external, bool is_index_sort);

    //===--------------------------------------------------------------------===//
    // Source 接口
    //===--------------------------------------------------------------------===//
    unique_ptr<LocalSourceState> GetLocalSourceState(ExecutionContext &context,
                                                      GlobalSourceState &gstate) const;
    unique_ptr<GlobalSourceState> GetGlobalSourceState(ClientContext &context) const;
    SourceResultType GetData(ExecutionContext &context, DataChunk &chunk,
                             OperatorSourceInput &input) const;

public:
    const Sort &sort;
    vector<unique_ptr<SortedRun>> sorted_runs;
    const idx_t total_count;
    const idx_t partition_size;
    const bool external;    // 是否为外部排序
    const bool is_index_sort;
};
```

```
┌────────────────────────────────────────────────────────────────────┐
│                        多路归并过程                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   SortedRun 1    SortedRun 2    SortedRun 3    SortedRun 4       │
│   [A, D, G]      [B, E, H]      [C, F, I]      [J, K, L]          │
│       │              │              │              │              │
│       └──────────────┴──────────────┴──────────────┘              │
│                           │                                        │
│                           ▼                                        │
│                  ┌─────────────────┐                              │
│                  │ K-way Merger    │                              │
│                  │ (优先队列)       │                              │
│                  └─────────────────┘                              │
│                           │                                        │
│                           ▼                                        │
│           输出: [A, B, C, D, E, F, G, H, I, J, K, L]              │
│                                                                    │
│   实现细节:                                                        │
│   • 使用最小堆维护每个 run 的当前最小元素                          │
│   • 每次取出堆顶元素输出，并从对应 run 补充下一个元素              │
│   • 支持并行扫描（按分区划分输出范围）                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.2.6 PhysicalOrder 执行流程

```cpp
// src/execution/operator/order/physical_order.cpp

// ===== Sink 阶段 =====
class OrderGlobalSinkState : public GlobalSinkState {
public:
    OrderGlobalSinkState(const PhysicalOrder &op, ClientContext &context)
        : sort(context, op.orders, op.children[0].get().types,
               op.projections, op.is_index_sort),
          state(sort.GetGlobalSinkState(context)) {
    }

public:
    Sort sort;                         // Sort 实例
    unique_ptr<GlobalSinkState> state; // Sort 的全局状态
};

SinkResultType PhysicalOrder::Sink(ExecutionContext &context, DataChunk &chunk,
                                   OperatorSinkInput &input) const {
    auto &gstate = input.global_state.Cast<OrderGlobalSinkState>();
    auto &lstate = input.local_state.Cast<OrderLocalSinkState>();

    // 延迟初始化本地状态
    if (!lstate.state) {
        lstate.state = gstate.sort.GetLocalSinkState(context);
    }

    // 委托给 Sort 类处理
    OperatorSinkInput sort_input {*gstate.state, *lstate.state, input.interrupt_state};
    return gstate.sort.Sink(context, chunk, sort_input);
}

SinkCombineResultType PhysicalOrder::Combine(ExecutionContext &context,
                                              OperatorSinkCombineInput &input) const {
    auto &gstate = input.global_state.Cast<OrderGlobalSinkState>();
    auto &lstate = input.local_state.Cast<OrderLocalSinkState>();
    if (!lstate.state) {
        return SinkCombineResultType::FINISHED;
    }
    // 合并本地 SortedRun 到全局
    OperatorSinkCombineInput sort_input {*gstate.state, *lstate.state, input.interrupt_state};
    return gstate.sort.Combine(context, sort_input);
}

SinkFinalizeType PhysicalOrder::Finalize(Pipeline &pipeline, Event &event,
                                          ClientContext &context,
                                          OperatorSinkFinalizeInput &input) const {
    auto &gstate = input.global_state.Cast<OrderGlobalSinkState>();
    OperatorSinkFinalizeInput sort_input {*gstate.state, input.interrupt_state};
    // 完成排序（可能触发归并）
    return gstate.sort.Finalize(context, sort_input);
}

// ===== Source 阶段 =====
SourceResultType PhysicalOrder::GetDataInternal(ExecutionContext &context,
                                                 DataChunk &chunk,
                                                 OperatorSourceInput &input) const {
    auto &gstate = input.global_state.Cast<OrderGlobalSourceState>();
    auto &lstate = input.local_state.Cast<OrderLocalSourceState>();
    OperatorSourceInput sort_input {*gstate.state, *lstate.state, input.interrupt_state};
    // 从排序结果中读取数据
    return gstate.sort.GetData(context, chunk, sort_input);
}
```

---

## 8.3 PhysicalTopN：堆优化的部分排序

### 8.3.1 算子概述

当查询包含 `ORDER BY ... LIMIT N` 且 N 较小时，使用完整排序是浪费的。`PhysicalTopN` 使用堆结构维护 Top N 个元素，时间复杂度从 O(n log n) 降至 O(n log k)。

```cpp
// src/include/duckdb/execution/operator/order/physical_top_n.hpp

class PhysicalTopN : public PhysicalOperator {
public:
    PhysicalTopN(PhysicalPlan &physical_plan, vector<LogicalType> types,
                 vector<BoundOrderByNode> orders, idx_t limit, idx_t offset,
                 shared_ptr<DynamicFilterData> dynamic_filter,
                 idx_t estimated_cardinality);

    //! 排序键
    vector<BoundOrderByNode> orders;
    //! LIMIT 数量
    idx_t limit;
    //! OFFSET 数量
    idx_t offset;
    //! 动态过滤器（用于过滤下推）
    shared_ptr<DynamicFilterData> dynamic_filter;
};
```

### 8.3.2 TopNHeap：堆数据结构

```cpp
// src/execution/operator/order/physical_top_n.cpp

struct TopNEntry {
    string_t sort_key;  // 编码后的排序键
    idx_t index;        // 在 heap_data 中的索引

    bool operator<(const TopNEntry &other) const {
        return sort_key < other.sort_key;
    }
};

class TopNHeap {
public:
    TopNHeap(ClientContext &context, const vector<LogicalType> &payload_types,
             const vector<BoundOrderByNode> &orders, idx_t limit, idx_t offset);

    //! 添加数据
    void Sink(DataChunk &input, optional_ptr<TopNBoundaryValue> boundary_value);
    //! 合并另一个堆
    void Combine(TopNHeap &other);
    //! 压缩堆（移除多余数据）
    void Reduce();
    //! 完成堆构建
    void Finalize();
    //! 扫描结果
    void Scan(TopNScanState &state, DataChunk &chunk, idx_t &pos);

public:
    Allocator &allocator;
    BufferManager &buffer_manager;
    ArenaAllocator arena_allocator;

    //! 堆结构（最大堆）
    unsafe_arena_vector<TopNEntry> heap;
    //! Payload 数据
    DataChunk heap_data;
    //! 堆大小 = limit + offset
    idx_t heap_size;

    //! 排序表达式执行器
    ExpressionExecutor executor;
    //! 排序键 Chunk
    DataChunk sort_chunk;
    DataChunk sort_keys;
    //! 排序键堆（存储非内联字符串）
    StringHeap sort_key_heap;
};
```

### 8.3.3 堆操作策略

TopN 使用两种不同的策略处理小堆和大堆：

```cpp
// 小堆阈值
static constexpr idx_t SMALL_HEAP_THRESHOLD = 100;

void TopNHeap::Sink(DataChunk &input, optional_ptr<TopNBoundaryValue> global_boundary) {
    // 计算排序键
    sort_chunk.Reset();
    executor.Execute(input, sort_chunk);

    // 如果有全局边界值，预先过滤不可能进入 TopN 的行
    if (global_boundary) {
        if (!CheckBoundaryValues(sort_chunk, input, *global_boundary)) {
            return; // 整个 chunk 都不满足条件
        }
    }

    // 生成排序键
    sort_keys.Reset();
    auto &sort_keys_vec = sort_keys.data[0];
    CreateSortKeyHelpers::CreateSortKey(sort_chunk, modifiers, sort_keys_vec);

    // 根据堆大小选择策略
    if (heap_size <= SMALL_HEAP_THRESHOLD) {
        AddSmallHeap(input, sort_keys_vec);  // 小堆：逐行添加
    } else {
        AddLargeHeap(input, sort_keys_vec);  // 大堆：批量添加
    }

    // 更新全局边界值
    if (heap.size() >= heap_size && global_boundary) {
        global_boundary->UpdateValue(heap.front().sort_key);
    }
}
```

```
┌────────────────────────────────────────────────────────────────────┐
│                     TopN 堆操作示意                                 │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ORDER BY score DESC LIMIT 3                                     │
│   输入: [50, 30, 80, 20, 90, 70, 60]                              │
│                                                                    │
│   步骤 1: 插入 50                                                  │
│   heap: [50]                                                      │
│                                                                    │
│   步骤 2: 插入 30                                                  │
│   heap: [50, 30]                                                  │
│                                                                    │
│   步骤 3: 插入 80                                                  │
│   heap: [80, 30, 50]  (堆化后)                                    │
│                                                                    │
│   步骤 4: 插入 20 → 堆满，20 < 30(堆顶)                            │
│   注意: 这是最小堆视角，堆顶是当前 Top3 中最小的                    │
│   20 不够资格进入 Top3，跳过                                       │
│                                                                    │
│   步骤 5: 插入 90 → 90 > 30(堆顶)                                  │
│   弹出 30，插入 90                                                 │
│   heap: [50, 80, 90] → 堆化 → [50, 90, 80]                        │
│                                                                    │
│   步骤 6: 插入 70 → 70 > 50(堆顶)                                  │
│   弹出 50，插入 70                                                 │
│   heap: [70, 90, 80]                                              │
│                                                                    │
│   步骤 7: 插入 60 → 60 < 70(堆顶)，跳过                           │
│                                                                    │
│   最终: heap 包含 [70, 90, 80]                                    │
│   Finalize 后排序: [90, 80, 70]                                   │
│                                                                    │
│   注: DuckDB 使用 std::push_heap/pop_heap 维护最大堆              │
│   堆顶是当前 TopN 中"最差"的元素（对于 DESC 是最小值）            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.3.4 小堆策略 vs 大堆策略

```cpp
void TopNHeap::AddSmallHeap(DataChunk &input, Vector &sort_keys_vec) {
    // 对于小堆，逐行检查并插入
    constexpr idx_t BASE_INDEX = NumericLimits<uint32_t>::Maximum();

    auto sort_key_values = FlatVector::GetData<string_t>(sort_keys_vec);
    bool any_added = false;

    for (idx_t r = 0; r < input.size(); r++) {
        auto &sort_key = sort_key_values[r];
        if (!EntryShouldBeAdded(sort_key)) {
            continue;  // 不够资格进入 TopN
        }

        TopNEntry entry;
        entry.sort_key = sort_key;
        entry.index = BASE_INDEX + r;  // 临时索引
        AddEntryToHeap(entry);
        any_added = true;
    }

    if (!any_added) return;

    // 为添加的条目复制 payload 数据
    idx_t match_count = 0;
    for (auto &entry : heap) {
        if (entry.index < BASE_INDEX) continue;

        // 非内联字符串需要复制到 sort_key_heap
        if (!entry.sort_key.IsInlined()) {
            entry.sort_key = sort_key_heap.AddBlob(entry.sort_key);
        }

        matching_sel.set_index(match_count, entry.index - BASE_INDEX);
        entry.index = heap_data.size() + match_count;
        match_count++;
    }

    heap_data.Append(input, true, &matching_sel, match_count);
}

void TopNHeap::AddLargeHeap(DataChunk &input, Vector &sort_keys_vec) {
    // 对于大堆，先添加到堆再复制数据
    auto sort_key_values = FlatVector::GetData<string_t>(sort_keys_vec);
    idx_t base_index = heap_data.size();
    idx_t match_count = 0;

    for (idx_t r = 0; r < input.size(); r++) {
        auto &sort_key = sort_key_values[r];
        if (!EntryShouldBeAdded(sort_key)) {
            continue;
        }

        TopNEntry entry;
        entry.sort_key = sort_key.IsInlined() ? sort_key : sort_key_heap.AddBlob(sort_key);
        entry.index = base_index + match_count;
        AddEntryToHeap(entry);
        matching_sel.set_index(match_count++, r);
    }

    if (match_count > 0) {
        heap_data.Append(input, true, &matching_sel, match_count);
    }
}
```

### 8.3.5 边界值优化

当堆满后，可以利用堆顶元素作为边界值，提前过滤不可能进入 TopN 的行：

```cpp
struct TopNBoundaryValue {
    explicit TopNBoundaryValue(const PhysicalTopN &op)
        : op(op), boundary_vector(op.orders[0].expression->return_type),
          boundary_modifiers(op.orders[0].type, op.orders[0].null_order) {}

    const PhysicalTopN &op;
    mutex lock;
    string boundary_value;           // 当前边界值（排序键格式）
    bool is_set = false;
    Vector boundary_vector;          // 解码后的边界值
    OrderModifiers boundary_modifiers;

    void UpdateValue(string_t boundary_val) {
        unique_lock<mutex> l(lock);
        if (!is_set || boundary_val < string_t(boundary_value)) {
            boundary_value = boundary_val.GetString();
            is_set = true;

            // 如果有动态过滤器，更新过滤条件并下推到扫描算子
            if (op.dynamic_filter) {
                CreateSortKeyHelpers::DecodeSortKey(boundary_val, boundary_vector,
                                                     0, boundary_modifiers);
                auto new_dynamic_value = boundary_vector.GetValue(0);
                l.unlock();
                op.dynamic_filter->SetValue(std::move(new_dynamic_value));
            }
        }
    }
};
```

```
┌────────────────────────────────────────────────────────────────────┐
│                     边界值动态过滤                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   SELECT * FROM sales ORDER BY amount DESC LIMIT 10               │
│                                                                    │
│   ┌─────────────┐          ┌─────────────┐                        │
│   │  TableScan  │ ◄─────── │ DynamicFilter │                       │
│   └──────┬──────┘          │ amount > $500 │                       │
│          │                 └───────────────┘                       │
│          ▼                        ▲                                │
│   ┌─────────────┐                 │                                │
│   │   TopNHeap  │ ────────────────┘                                │
│   │  limit=10   │   当堆满后，最小值 = $500                        │
│   └─────────────┘   更新过滤条件: amount > $500                    │
│                                                                    │
│   优化效果:                                                        │
│   • 初始阶段: 扫描所有数据                                         │
│   • 堆满后: 只扫描 amount > 当前最小值 的数据                      │
│   • 边界值随时更新，过滤越来越严格                                  │
│   • 显著减少后续需要处理的数据量                                   │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.3.6 堆数据压缩

随着数据的添加和淘汰，`heap_data` 中会积累无效数据。`Reduce()` 方法周期性清理：

```cpp
void TopNHeap::Reduce() {
    if (heap_data.size() < ReduceThreshold()) {
        return;  // 未达到压缩阈值
    }

    // 创建新的数据存储
    StringHeap new_sort_heap;
    DataChunk new_heap_data;
    new_heap_data.Initialize(allocator, payload_types, heap.size());

    // 只保留堆中还存在的条目
    SelectionVector new_payload_sel(heap.size());
    for (idx_t i = 0; i < heap.size(); i++) {
        auto &entry = heap[i];

        // 迁移排序键到新堆
        if (!entry.sort_key.IsInlined()) {
            entry.sort_key = new_sort_heap.AddBlob(entry.sort_key);
        }

        // 记录数据位置
        new_payload_sel.set_index(i, entry.index);
        entry.index = i;  // 新索引
    }

    // 复制有效数据
    new_heap_data.Slice(heap_data, new_payload_sel, heap.size());
    new_heap_data.Flatten();

    // 替换旧数据
    sort_key_heap.Destroy();
    sort_key_heap.Move(new_sort_heap);
    heap_data.Reference(new_heap_data);
}

idx_t ReduceThreshold() const {
    // 阈值 = max(5 * VECTOR_SIZE, 2 * heap_size)
    return MaxValue<idx_t>(STANDARD_VECTOR_SIZE * 5ULL, 2ULL * heap_size);
}
```

### 8.3.7 TopN 完整执行流程

```cpp
// ===== Sink 阶段 =====
SinkResultType PhysicalTopN::Sink(ExecutionContext &context, DataChunk &chunk,
                                   OperatorSinkInput &input) const {
    auto &gstate = input.global_state.Cast<TopNGlobalSinkState>();
    auto &sink = input.local_state.Cast<TopNLocalSinkState>();

    // 使用全局边界值进行早期过滤
    sink.heap.Sink(chunk, &gstate.boundary_value);
    sink.heap.Reduce();  // 周期性压缩

    return SinkResultType::NEED_MORE_INPUT;
}

// ===== Combine 阶段 =====
SinkCombineResultType PhysicalTopN::Combine(ExecutionContext &context,
                                             OperatorSinkCombineInput &input) const {
    auto &gstate = input.global_state.Cast<TopNGlobalSinkState>();
    auto &lstate = input.local_state.Cast<TopNLocalSinkState>();

    // 先排序本地堆（便于合并时快速终止）
    lstate.heap.Finalize();

    lock_guard<mutex> guard(gstate.lock);
    // 合并到全局堆
    gstate.heap.Combine(lstate.heap);

    return SinkCombineResultType::FINISHED;
}

// ===== Finalize 阶段 =====
SinkFinalizeType PhysicalTopN::Finalize(Pipeline &pipeline, Event &event,
                                         ClientContext &context,
                                         OperatorSinkFinalizeInput &input) const {
    auto &gstate = input.global_state.Cast<TopNGlobalSinkState>();
    // 最终排序
    gstate.heap.Finalize();
    return SinkFinalizeType::READY;
}

// ===== Source 阶段 =====
SourceResultType PhysicalTopN::GetDataInternal(ExecutionContext &context, DataChunk &chunk,
                                                OperatorSourceInput &input) const {
    if (limit == 0) {
        return SourceResultType::FINISHED;
    }

    auto &sink = sink_state->Cast<TopNGlobalSinkState>();
    auto &gstate = input.global_state.Cast<TopNGlobalSourceState>();
    auto &lstate = input.local_state.Cast<TopNLocalSourceState>();

    // 分批扫描结果
    if (lstate.pos == lstate.end) {
        auto guard = gstate.Lock();
        lstate.pos = gstate.state.pos;
        gstate.state.pos += TopNGlobalSourceState::TUPLES_PER_BATCH;
        lstate.end = gstate.state.pos;
        lstate.batch_index = gstate.batch_index++;
    }

    sink.heap.Scan(gstate.state, chunk, lstate.pos);

    return chunk.size() == 0 ? SourceResultType::FINISHED
                              : SourceResultType::HAVE_MORE_OUTPUT;
}
```

---

## 8.4 外部排序

### 8.4.1 内存压力处理

当数据量超过可用内存时，DuckDB 使用外部排序：

```
┌────────────────────────────────────────────────────────────────────┐
│                       外部排序流程                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Phase 1: 创建初始 Runs                                          │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  读取数据 → 内存排序 → 写入临时文件                          │ │
│   │                                                              │ │
│   │  内存容量: 1GB                                               │ │
│   │  数据量: 10GB                                                │ │
│   │                                                              │ │
│   │  Run1 (1GB) → temp_1.duckdb                                 │ │
│   │  Run2 (1GB) → temp_2.duckdb                                 │ │
│   │  ...                                                         │ │
│   │  Run10 (1GB) → temp_10.duckdb                               │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   Phase 2: 多路归并                                               │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  ┌─────┐ ┌─────┐ ┌─────┐       ┌─────┐                     │ │
│   │  │Run1 │ │Run2 │ │Run3 │  ...  │Run10│                     │ │
│   │  └──┬──┘ └──┬──┘ └──┬──┘       └──┬──┘                     │ │
│   │     │       │       │             │                         │ │
│   │     └───────┴───────┴─────┬───────┘                         │ │
│   │                           │                                  │ │
│   │                    ┌──────▼──────┐                          │ │
│   │                    │  K-way Merge │                          │ │
│   │                    │  (优先队列)   │                          │ │
│   │                    └──────┬──────┘                          │ │
│   │                           │                                  │ │
│   │                           ▼                                  │ │
│   │                    排序后的输出流                             │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   优化:                                                           │
│   • 使用 Buffer Pool 管理内存页                                   │
│   • 预取策略减少 I/O 等待                                         │
│   • 可能需要多轮归并（如果 run 数量太多）                          │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.4.2 SortedRun 的外部存储

```cpp
void SortedRun::Finalize(bool external) {
    if (finalized) return;

    if (external) {
        // 外部排序模式：物理重排数据
        // 按排序键顺序重新组织 payload 数据
        // 便于后续顺序读取
    } else {
        // 内存排序模式：只排序索引
        // payload 数据保持原位，通过索引访问
    }

    finalized = true;
}
```

---

## 8.5 性能优化技术

### 8.5.1 排序键优化

```
┌────────────────────────────────────────────────────────────────────┐
│                    排序键优化技术                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  1. 前缀编码                                                       │
│     对于可变长度类型（字符串），只存储排序需要的前缀               │
│     减少内存占用和比较开销                                         │
│                                                                    │
│  2. NULL 编码                                                      │
│     NULLS FIRST: NULL → 0x00 (最小值)                             │
│     NULLS LAST:  NULL → 0xFF (最大值)                             │
│     使用单字节标记，无需特殊处理                                   │
│                                                                    │
│  3. 降序优化                                                       │
│     DESC 排序：按位取反                                            │
│     使得同样的 < 比较可以处理升序和降序                            │
│                                                                    │
│  4. 紧凑存储                                                       │
│     整数类型：定长编码                                             │
│     浮点类型：IEEE 754 规范化编码                                  │
│     字符串：长度前缀 + 内容                                        │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 8.5.2 TopN vs Order 选择

优化器根据以下条件选择使用 TopN 还是 Order：

| 条件 | TopN | Order |
|------|------|-------|
| LIMIT 存在且较小 | ✓ | |
| LIMIT 不存在 | | ✓ |
| LIMIT 接近总行数 | | ✓ |
| 有 OFFSET 但 LIMIT+OFFSET 较小 | ✓ | |
| 需要全量排序结果 | | ✓ |

### 8.5.3 并行排序

```
┌────────────────────────────────────────────────────────────────────┐
│                      并行排序策略                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                     Sink 阶段（并行）                        │ │
│   ├─────────────────────────────────────────────────────────────┤ │
│   │                                                              │ │
│   │   Thread 1         Thread 2         Thread 3                │ │
│   │   ┌─────────┐     ┌─────────┐      ┌─────────┐             │ │
│   │   │LocalSink│     │LocalSink│      │LocalSink│             │ │
│   │   │SortedRun│     │SortedRun│      │SortedRun│             │ │
│   │   └────┬────┘     └────┬────┘      └────┬────┘             │ │
│   │        │               │                │                   │ │
│   └────────┼───────────────┼────────────────┼───────────────────┘ │
│            │               │                │                     │
│            ▼               ▼                ▼                     │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                   Combine 阶段（串行）                       │ │
│   ├─────────────────────────────────────────────────────────────┤ │
│   │  收集所有 SortedRun，准备归并                                │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│                              ▼                                     │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                  Finalize 阶段（归并）                       │ │
│   ├─────────────────────────────────────────────────────────────┤ │
│   │  使用 SortedRunMerger 进行多路归并                          │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                              │                                     │
│                              ▼                                     │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                   Source 阶段（并行）                        │ │
│   ├─────────────────────────────────────────────────────────────┤ │
│   │  多线程并行扫描归并后的有序数据                              │ │
│   │  按分区划分扫描范围                                          │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 8.6 源文件索引

| 组件 | 文件路径 |
|------|----------|
| PhysicalOrder | `src/execution/operator/order/physical_order.cpp` |
| PhysicalTopN | `src/execution/operator/order/physical_top_n.cpp` |
| Sort 类 | `src/include/duckdb/common/sorting/sort.hpp` |
| SortedRun | `src/include/duckdb/common/sorting/sorted_run.hpp` |
| SortedRunMerger | `src/include/duckdb/common/sorting/sorted_run_merger.hpp` |
| 排序键创建 | `src/function/create_sort_key.cpp` |
| TupleDataLayout | `src/include/duckdb/common/types/row/tuple_data_layout.hpp` |

---

## 8.7 本章小结

本章深入分析了 DuckDB 的排序与 TopN 算子实现：

1. **PhysicalOrder** 通过 Sort 类实现完整排序，使用排序键编码技术将多列排序键转换为可直接比较的二进制串

2. **SortedRun** 表示已排序的数据片段，支持内存排序和外部排序两种模式

3. **SortedRunMerger** 实现多路归并，将多个 SortedRun 合并为单一有序流

4. **PhysicalTopN** 使用堆结构高效维护 Top N 元素，避免完整排序
   - 小堆策略（≤100）：逐行检查并插入
   - 大堆策略（>100）：批量处理
   - 边界值优化：利用堆顶值提前过滤
   - 动态过滤：将边界条件下推到扫描算子

5. **外部排序** 通过创建多个 SortedRun 并归并的方式处理超大数据集

6. **并行执行** 在 Sink 阶段并行创建本地 SortedRun，Finalize 阶段归并，Source 阶段并行扫描输出

排序算子是典型的 Pipeline Breaker，需要收集所有输入数据后才能输出。TopN 算子通过堆结构和边界值优化显著减少了需要处理的数据量，是 `ORDER BY ... LIMIT` 查询的重要优化手段。
