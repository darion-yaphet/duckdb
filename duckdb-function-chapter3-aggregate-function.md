# DuckDB 函数系统深度解析（三）：聚合函数状态机模型

## 引言

聚合函数（Aggregate Function）将多行数据聚合为单个结果，是 SQL 分析查询的核心。与标量函数不同，聚合函数需要维护中间状态，并支持并行聚合时的状态合并。本章深入分析 DuckDB 聚合函数的状态机模型、执行器实现和典型函数设计。

## 1. AggregateFunction 核心结构

### 1.1 类定义

```
源文件: src/include/duckdb/function/aggregate_function.hpp
```

```cpp
class AggregateFunction : public BaseScalarFunction {
public:
    AggregateFunction(const string &name,
                      const vector<LogicalType> &arguments,
                      const LogicalType &return_type,
                      aggregate_size_t state_size,
                      aggregate_initialize_t initialize,
                      aggregate_update_t update,
                      aggregate_combine_t combine,
                      aggregate_finalize_t finalize,
                      FunctionNullHandling null_handling = FunctionNullHandling::DEFAULT_NULL_HANDLING,
                      aggregate_simple_update_t simple_update = nullptr,
                      bind_aggregate_function_t bind = nullptr,
                      aggregate_destructor_t destructor = nullptr,
                      aggregate_statistics_t statistics = nullptr,
                      aggregate_window_t window = nullptr,
                      aggregate_serialize_t serialize = nullptr,
                      aggregate_deserialize_t deserialize = nullptr);

public:
    //! 状态大小计算
    aggregate_size_t state_size;
    //! 状态初始化
    aggregate_initialize_t initialize;
    //! 状态更新（分散模式）
    aggregate_update_t update;
    //! 状态合并
    aggregate_combine_t combine;
    //! 最终结果计算
    aggregate_finalize_t finalize;
    //! 简单更新（非分组聚合优化）
    aggregate_simple_update_t simple_update;
    //! 窗口聚合自定义计算
    aggregate_window_t window;
    //! 窗口聚合初始化
    aggregate_wininit_t window_init;
    //! 绑定回调
    bind_aggregate_function_t bind;
    //! 状态销毁
    aggregate_destructor_t destructor;
    //! 统计传播
    aggregate_statistics_t statistics;
    //! 序列化/反序列化
    aggregate_serialize_t serialize;
    aggregate_deserialize_t deserialize;
    //! 顺序依赖性
    AggregateOrderDependent order_dependent;
    //! DISTINCT 依赖性
    AggregateDistinctDependent distinct_dependent;
};
```

### 1.2 回调函数类型

```cpp
//! 状态大小
typedef idx_t (*aggregate_size_t)(const AggregateFunction &function);

//! 状态初始化
typedef void (*aggregate_initialize_t)(const AggregateFunction &function, data_ptr_t state);

//! 状态更新（分散模式：多行更新到多个状态）
typedef void (*aggregate_update_t)(Vector inputs[],
                                   AggregateInputData &aggr_input_data,
                                   idx_t input_count,
                                   Vector &state,
                                   idx_t count);

//! 状态合并
typedef void (*aggregate_combine_t)(Vector &state,
                                    Vector &combined,
                                    AggregateInputData &aggr_input_data,
                                    idx_t count);

//! 最终结果计算
typedef void (*aggregate_finalize_t)(Vector &state,
                                     AggregateInputData &aggr_input_data,
                                     Vector &result,
                                     idx_t count,
                                     idx_t offset);

//! 简单更新（多行更新到单个状态）
typedef void (*aggregate_simple_update_t)(Vector inputs[],
                                          AggregateInputData &aggr_input_data,
                                          idx_t input_count,
                                          data_ptr_t state,
                                          idx_t count);

//! 窗口聚合计算
typedef void (*aggregate_window_t)(AggregateInputData &aggr_input_data,
                                   const WindowPartitionInput &partition,
                                   const_data_ptr_t g_state,
                                   data_ptr_t l_state,
                                   const SubFrames &subframes,
                                   Vector &result,
                                   idx_t rid);

//! 状态销毁
typedef void (*aggregate_destructor_t)(Vector &state,
                                       AggregateInputData &aggr_input_data,
                                       idx_t count);
```

## 2. 聚合函数生命周期

### 2.1 状态机模型

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Aggregate Function Lifecycle                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│    ┌──────────────┐                                                 │
│    │  state_size  │  计算状态所需内存大小                           │
│    └──────┬───────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│    ┌──────────────┐                                                 │
│    │  initialize  │  初始化状态（如 sum=0, count=0）                │
│    └──────┬───────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│    ┌──────────────┐                                                 │
│    │    update    │◄─────┐  累积输入数据                            │
│    └──────┬───────┘      │                                          │
│           │              │                                          │
│           ▼              │                                          │
│    ┌──────────────┐      │                                          │
│    │   (更多行)   │──────┘                                          │
│    └──────┬───────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│    ┌──────────────┐                                                 │
│    │   combine    │  合并多个状态（并行聚合）                       │
│    └──────┬───────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│    ┌──────────────┐                                                 │
│    │   finalize   │  计算最终结果                                   │
│    └──────┬───────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│    ┌──────────────┐                                                 │
│    │  destructor  │  清理状态（如释放字符串）                       │
│    └──────────────┘                                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 分散更新 vs 简单更新

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Update Modes                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Scatter Update (update)          Simple Update (simple_update)     │
│  用于分组聚合                      用于非分组聚合                    │
│                                                                      │
│  ┌─────┐  ┌─────┐  ┌─────┐       ┌─────┐  ┌─────┐  ┌─────┐         │
│  │Row 1│  │Row 2│  │Row 3│       │Row 1│  │Row 2│  │Row 3│         │
│  │ A=1 │  │ A=2 │  │ A=1 │       │ V=1 │  │ V=2 │  │ V=3 │         │
│  └──┬──┘  └──┬──┘  └──┬──┘       └──┬──┘  └──┬──┘  └──┬──┘         │
│     │        │        │             │        │        │             │
│     ▼        ▼        ▼             └────────┼────────┘             │
│  ┌─────┐  ┌─────┐                           │                       │
│  │State│  │State│                           ▼                       │
│  │ A=1 │  │ A=2 │                     ┌──────────┐                  │
│  └─────┘  └─────┘                     │ Single   │                  │
│                                       │ State    │                  │
│  多行 → 多状态                        └──────────┘                  │
│  (通过 state vector 索引)              多行 → 单状态                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. AggregateExecutor 模板

### 3.1 状态管理辅助方法

```cpp
class AggregateFunction {
public:
    // 状态大小
    template <class STATE>
    static idx_t StateSize(const AggregateFunction &) {
        return sizeof(STATE);
    }

    // 状态初始化
    template <class STATE, class OP,
              AggregateDestructorType destructor_type = AggregateDestructorType::STANDARD>
    static void StateInitialize(const AggregateFunction &, data_ptr_t state) {
        OP::Initialize(*reinterpret_cast<STATE *>(state));
    }

    // 一元分散更新
    template <class STATE, class T, class OP>
    static void UnaryScatterUpdate(Vector inputs[],
                                   AggregateInputData &aggr_input_data,
                                   idx_t input_count,
                                   Vector &states,
                                   idx_t count) {
        D_ASSERT(input_count == 1);
        AggregateExecutor::UnaryScatter<STATE, T, OP>(
            inputs[0], states, aggr_input_data, count);
    }

    // 一元简单更新
    template <class STATE, class INPUT_TYPE, class OP>
    static void UnaryUpdate(Vector inputs[],
                            AggregateInputData &aggr_input_data,
                            idx_t input_count,
                            data_ptr_t state,
                            idx_t count) {
        D_ASSERT(input_count == 1);
        AggregateExecutor::UnaryUpdate<STATE, INPUT_TYPE, OP>(
            inputs[0], aggr_input_data, state, count);
    }

    // 状态合并
    template <class STATE, class OP>
    static void StateCombine(Vector &source,
                             Vector &target,
                             AggregateInputData &aggr_input_data,
                             idx_t count) {
        AggregateExecutor::Combine<STATE, OP>(source, target, aggr_input_data, count);
    }

    // 状态终结
    template <class STATE, class RESULT_TYPE, class OP>
    static void StateFinalize(Vector &states,
                              AggregateInputData &aggr_input_data,
                              Vector &result,
                              idx_t count,
                              idx_t offset) {
        AggregateExecutor::Finalize<STATE, RESULT_TYPE, OP>(
            states, aggr_input_data, result, count, offset);
    }

    // 状态销毁
    template <class STATE, class OP>
    static void StateDestroy(Vector &states,
                             AggregateInputData &aggr_input_data,
                             idx_t count) {
        AggregateExecutor::Destroy<STATE, OP>(states, aggr_input_data, count);
    }
};
```

### 3.2 AggregateExecutor 实现

```cpp
class AggregateExecutor {
public:
    // 一元分散更新
    template <class STATE, class INPUT_TYPE, class OP>
    static void UnaryScatter(Vector &input, Vector &states,
                             AggregateInputData &aggr_input_data, idx_t count) {
        UnifiedVectorFormat input_data;
        input.ToUnifiedFormat(count, input_data);

        UnifiedVectorFormat states_data;
        states.ToUnifiedFormat(count, states_data);

        auto input_ptr = UnifiedVectorFormat::GetData<INPUT_TYPE>(input_data);
        auto states_ptr = (STATE **)states_data.data;

        for (idx_t i = 0; i < count; i++) {
            auto input_idx = input_data.sel->get_index(i);
            auto state_idx = states_data.sel->get_index(i);

            if (input_data.validity.RowIsValid(input_idx)) {
                OP::Operation(states_ptr[state_idx], aggr_input_data,
                              input_ptr[input_idx]);
            }
        }
    }

    // 一元简单更新
    template <class STATE, class INPUT_TYPE, class OP>
    static void UnaryUpdate(Vector &input, AggregateInputData &aggr_input_data,
                            data_ptr_t state, idx_t count) {
        UnifiedVectorFormat input_data;
        input.ToUnifiedFormat(count, input_data);

        auto input_ptr = UnifiedVectorFormat::GetData<INPUT_TYPE>(input_data);
        auto state_ptr = reinterpret_cast<STATE *>(state);

        for (idx_t i = 0; i < count; i++) {
            auto idx = input_data.sel->get_index(i);
            if (input_data.validity.RowIsValid(idx)) {
                OP::Operation(state_ptr, aggr_input_data, input_ptr[idx]);
            }
        }
    }

    // 状态合并
    template <class STATE, class OP>
    static void Combine(Vector &source, Vector &target,
                        AggregateInputData &aggr_input_data, idx_t count) {
        UnifiedVectorFormat source_data, target_data;
        source.ToUnifiedFormat(count, source_data);
        target.ToUnifiedFormat(count, target_data);

        auto source_ptr = (STATE **)source_data.data;
        auto target_ptr = (STATE **)target_data.data;

        for (idx_t i = 0; i < count; i++) {
            auto src_idx = source_data.sel->get_index(i);
            auto tgt_idx = target_data.sel->get_index(i);
            OP::Combine(*source_ptr[src_idx], target_ptr[tgt_idx], aggr_input_data);
        }
    }

    // 最终计算
    template <class STATE, class RESULT_TYPE, class OP>
    static void Finalize(Vector &states, AggregateInputData &aggr_input_data,
                         Vector &result, idx_t count, idx_t offset) {
        UnifiedVectorFormat states_data;
        states.ToUnifiedFormat(count, states_data);

        auto states_ptr = (STATE **)states_data.data;
        auto result_ptr = FlatVector::GetData<RESULT_TYPE>(result);
        auto &result_validity = FlatVector::Validity(result);

        for (idx_t i = 0; i < count; i++) {
            auto state_idx = states_data.sel->get_index(i);
            OP::Finalize(states_ptr[state_idx], aggr_input_data,
                         result_ptr[offset + i], result_validity, offset + i);
        }
    }
};
```

## 4. 典型聚合函数实现

### 4.1 COUNT 函数

```
源文件: src/function/aggregate/distributive/count.cpp
```

```cpp
// 状态定义
struct CountState {
    int64_t count;
};

// 操作定义
struct CountOperation {
    // 初始化
    template <class STATE>
    static void Initialize(STATE &state) {
        state.count = 0;
    }

    // 更新
    template <class STATE, class INPUT_TYPE>
    static void Operation(STATE &state, AggregateInputData &, INPUT_TYPE &input) {
        state.count++;
    }

    // 合并
    template <class STATE>
    static void Combine(STATE &source, STATE *target, AggregateInputData &) {
        target->count += source.count;
    }

    // 终结
    template <class STATE, class RESULT_TYPE>
    static void Finalize(STATE &state, AggregateInputData &,
                         RESULT_TYPE &result, ValidityMask &mask, idx_t idx) {
        result = state.count;
    }

    // COUNT 有特殊 NULL 处理
    static bool IgnoreNull() {
        return true;  // 忽略 NULL 值
    }
};

// COUNT(*) 变体：计数所有行，包括 NULL
struct CountStarOperation : public CountOperation {
    template <class STATE>
    static void Operation(STATE &state, AggregateInputData &) {
        state.count++;
    }
};

// 函数注册
AggregateFunction CountFun::GetFunction() {
    return AggregateFunction::UnaryAggregate<CountState, int64_t, int64_t, CountOperation>(
        LogicalType::ANY, LogicalType::BIGINT);
}
```

### 4.2 SUM 函数

```cpp
// 状态定义（带溢出检查）
template <class T>
struct SumState {
    T value;
    bool isset;
};

// 操作定义
template <class INPUT_TYPE, class SUM_TYPE>
struct SumOperation {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.value = 0;
        state.isset = false;
    }

    template <class STATE>
    static void Operation(STATE &state, AggregateInputData &, INPUT_TYPE &input) {
        state.value += input;
        state.isset = true;
    }

    template <class STATE>
    static void Combine(STATE &source, STATE *target, AggregateInputData &) {
        if (source.isset) {
            target->value += source.value;
            target->isset = true;
        }
    }

    template <class STATE, class RESULT_TYPE>
    static void Finalize(STATE &state, AggregateInputData &,
                         RESULT_TYPE &result, ValidityMask &mask, idx_t idx) {
        if (!state.isset) {
            mask.SetInvalid(idx);  // 无输入返回 NULL
        } else {
            result = state.value;
        }
    }
};

// 不同类型的 SUM 实现
AggregateFunctionSet SumFun::GetFunctions() {
    AggregateFunctionSet set("sum");

    // INTEGER → HUGEINT（避免溢出）
    set.AddFunction(AggregateFunction::UnaryAggregate<
        SumState<hugeint_t>, int32_t, hugeint_t,
        SumOperation<int32_t, hugeint_t>>(
        LogicalType::INTEGER, LogicalType::HUGEINT));

    // BIGINT → HUGEINT
    set.AddFunction(AggregateFunction::UnaryAggregate<
        SumState<hugeint_t>, int64_t, hugeint_t,
        SumOperation<int64_t, hugeint_t>>(
        LogicalType::BIGINT, LogicalType::HUGEINT));

    // DOUBLE → DOUBLE
    set.AddFunction(AggregateFunction::UnaryAggregate<
        SumState<double>, double, double,
        SumOperation<double, double>>(
        LogicalType::DOUBLE, LogicalType::DOUBLE));

    return set;
}
```

### 4.3 AVG 函数

```cpp
// 状态定义
template <class T>
struct AvgState {
    T sum;
    uint64_t count;
};

// 操作定义
template <class INPUT_TYPE, class SUM_TYPE>
struct AvgOperation {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.sum = 0;
        state.count = 0;
    }

    template <class STATE>
    static void Operation(STATE &state, AggregateInputData &, INPUT_TYPE &input) {
        state.sum += input;
        state.count++;
    }

    template <class STATE>
    static void Combine(STATE &source, STATE *target, AggregateInputData &) {
        target->sum += source.sum;
        target->count += source.count;
    }

    template <class STATE, class RESULT_TYPE>
    static void Finalize(STATE &state, AggregateInputData &,
                         RESULT_TYPE &result, ValidityMask &mask, idx_t idx) {
        if (state.count == 0) {
            mask.SetInvalid(idx);
        } else {
            result = state.sum / static_cast<RESULT_TYPE>(state.count);
        }
    }
};
```

### 4.4 MIN/MAX 函数

```
源文件: src/function/aggregate/distributive/minmax.cpp
```

```cpp
// MIN 状态
template <class T>
struct MinState {
    T value;
    bool isset;
};

// MIN 操作
template <class INPUT_TYPE>
struct MinOperation {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.isset = false;
    }

    template <class STATE>
    static void Operation(STATE &state, AggregateInputData &, INPUT_TYPE &input) {
        if (!state.isset || input < state.value) {
            state.value = input;
            state.isset = true;
        }
    }

    template <class STATE>
    static void Combine(STATE &source, STATE *target, AggregateInputData &) {
        if (source.isset) {
            if (!target->isset || source.value < target->value) {
                target->value = source.value;
                target->isset = true;
            }
        }
    }

    template <class STATE, class RESULT_TYPE>
    static void Finalize(STATE &state, AggregateInputData &,
                         RESULT_TYPE &result, ValidityMask &mask, idx_t idx) {
        if (!state.isset) {
            mask.SetInvalid(idx);
        } else {
            result = state.value;
        }
    }
};

// MAX 操作（与 MIN 类似，只是比较方向相反）
template <class INPUT_TYPE>
struct MaxOperation {
    template <class STATE>
    static void Operation(STATE &state, AggregateInputData &, INPUT_TYPE &input) {
        if (!state.isset || input > state.value) {
            state.value = input;
            state.isset = true;
        }
    }
    // ... 其他方法类似
};
```

### 4.5 STRING_AGG 函数

```cpp
// 字符串聚合状态（需要特殊内存管理）
struct StringAggState {
    string value;
    bool first;
};

struct StringAggOperation {
    template <class STATE>
    static void Initialize(STATE &state) {
        state.first = true;
    }

    static void Operation(StringAggState &state, AggregateInputData &aggr_input_data,
                          string_t &input, string_t &delimiter) {
        if (!state.first) {
            state.value += delimiter.GetString();
        }
        state.value += input.GetString();
        state.first = false;
    }

    static void Combine(StringAggState &source, StringAggState *target,
                        AggregateInputData &) {
        if (!source.first) {
            if (!target->first) {
                // 需要分隔符 - 但 combine 时没有分隔符信息
                // 这是 STRING_AGG 不能完全并行的原因
            }
            target->value += source.value;
            target->first = false;
        }
    }

    template <class RESULT_TYPE>
    static void Finalize(StringAggState &state, AggregateInputData &,
                         RESULT_TYPE &result, ValidityMask &mask, idx_t idx) {
        if (state.first) {
            mask.SetInvalid(idx);
        } else {
            result = StringVector::AddString(result_vector, state.value);
        }
    }

    // 需要析构器释放 string
    static void Destroy(StringAggState &state, AggregateInputData &) {
        state.value.~basic_string();
    }
};
```

## 5. 窗口聚合支持

### 5.1 WindowPartitionInput

```cpp
struct WindowPartitionInput {
    WindowPartitionInput(ExecutionContext &context,
                         const ColumnDataCollection *inputs,
                         const idx_t count,
                         const vector<column_t> &column_ids,
                         const vector<bool> &all_valid,
                         const ValidityMask &filter_mask,
                         const FrameStats &stats,
                         InterruptState &interrupt_state);

    ExecutionContext &context;
    const ColumnDataCollection *inputs;  // 分区数据
    const idx_t count;                    // 行数
    const vector<column_t> column_ids;    // 列索引
    const vector<bool> &all_valid;        // 列是否全部有效
    const ValidityMask &filter_mask;      // 过滤掩码
    const FrameStats stats;               // 帧统计
    InterruptState &interrupt_state;      // 中断状态
};
```

### 5.2 FrameStats：帧边界信息

```cpp
// 相对于当前行的帧边界
struct FrameDelta {
    int64_t begin = 0;  // 帧起始偏移
    int64_t end = 0;    // 帧结束偏移
};

// 帧统计（BEGIN 和 END 的范围）
using FrameStats = array<FrameDelta, 2>;
```

### 5.3 自定义窗口函数示例

```cpp
// LEAD/LAG 窗口函数
struct LeadLagOperation {
    // 窗口初始化
    static void WindowInit(AggregateInputData &aggr_input_data,
                          const WindowPartitionInput &partition,
                          data_ptr_t g_state) {
        // 初始化全局窗口状态
    }

    // 窗口计算
    static void Window(AggregateInputData &aggr_input_data,
                       const WindowPartitionInput &partition,
                       const_data_ptr_t g_state,
                       data_ptr_t l_state,
                       const SubFrames &subframes,
                       Vector &result,
                       idx_t rid) {
        auto &bind_data = aggr_input_data.bind_data->Cast<LeadLagBindData>();
        auto offset = bind_data.offset;

        // 计算目标行
        int64_t target_row = rid + offset;  // LEAD: +offset, LAG: -offset

        if (target_row < 0 || target_row >= partition.count) {
            // 超出范围，返回默认值
            if (bind_data.default_value.IsNull()) {
                FlatVector::SetNull(result, rid, true);
            } else {
                result.SetValue(rid, bind_data.default_value);
            }
        } else {
            // 复制目标行的值
            auto value = partition.inputs->GetValue(bind_data.column_idx, target_row);
            result.SetValue(rid, value);
        }
    }
};

// 注册窗口函数
AggregateFunction GetLeadFunction() {
    return AggregateFunction(
        {LogicalType::ANY, LogicalType::BIGINT, LogicalType::ANY},  // 参数
        LogicalType::ANY,  // 返回类型
        AggregateFunction::StateSize<LeadLagState>,
        AggregateFunction::StateInitialize<LeadLagState, LeadLagOperation>,
        LeadLagOperation::WindowInit,
        LeadLagOperation::Window,
        LeadLagBind,
        nullptr,  // 无析构
        nullptr,  // 无统计
        nullptr,  // 无序列化
        nullptr); // 无反序列化
}
```

## 6. 排序聚合（ORDER BY 子句）

### 6.1 SortedAggregateFunction

```
源文件: src/function/aggregate/sorted_aggregate_function.cpp
```

某些聚合函数对输入顺序敏感：
```sql
SELECT string_agg(name, ',' ORDER BY name) FROM t;
SELECT first(value ORDER BY timestamp) FROM t;
```

```cpp
struct SortedAggregateBindData : public FunctionData {
    //! 原始聚合函数
    AggregateFunction function;
    //! 排序信息
    vector<BoundOrderByNode> order_bys;
    //! 子函数绑定数据
    unique_ptr<FunctionData> child_bind_data;
};

void FunctionBinder::BindSortedAggregate(ClientContext &context,
                                          BoundAggregateExpression &expr,
                                          const vector<unique_ptr<Expression>> &groups,
                                          optional_ptr<vector<GroupingSet>> grouping_sets) {
    // 检查是否有 ORDER BY
    if (expr.order_bys.empty()) {
        return;
    }

    // 创建排序聚合包装器
    auto bind_data = make_uniq<SortedAggregateBindData>();
    bind_data->function = expr.function;
    bind_data->order_bys = std::move(expr.order_bys);
    bind_data->child_bind_data = std::move(expr.bind_info);

    // 替换为排序聚合函数
    expr.function = GetSortedAggregateFunction(expr.function);
    expr.bind_info = std::move(bind_data);
}
```

### 6.2 顺序依赖性

```cpp
enum class AggregateOrderDependent : uint8_t {
    NOT_ORDER_DEPENDENT,  // 结果与顺序无关（如 SUM）
    ORDER_DEPENDENT       // 结果与顺序相关（如 FIRST）
};

// 注册时指定
AggregateFunction first_func = ...;
first_func.order_dependent = AggregateOrderDependent::ORDER_DEPENDENT;
```

## 7. DISTINCT 聚合

### 7.1 DISTINCT 依赖性

```cpp
enum class AggregateDistinctDependent : uint8_t {
    NOT_DISTINCT_DEPENDENT,  // DISTINCT 无影响（如 MIN）
    DISTINCT_DEPENDENT       // DISTINCT 影响结果（如 COUNT）
};
```

### 7.2 DISTINCT 处理

```sql
SELECT count(DISTINCT category) FROM products;
```

DISTINCT 聚合的处理流程：
1. 收集所有输入值
2. 去重
3. 对去重后的值执行聚合

```cpp
struct DistinctAggregateState {
    // 存储已见过的值
    unordered_set<Value> seen_values;
};

struct DistinctCountOperation {
    template <class STATE>
    static void Initialize(STATE &state) {
        // 使用 placement new 初始化 unordered_set
        new (&state.seen_values) unordered_set<Value>();
    }

    template <class STATE, class INPUT_TYPE>
    static void Operation(STATE &state, AggregateInputData &, INPUT_TYPE &input) {
        state.seen_values.insert(Value::CreateValue(input));
    }

    template <class STATE>
    static void Finalize(STATE &state, AggregateInputData &,
                         int64_t &result, ValidityMask &mask, idx_t idx) {
        result = state.seen_values.size();
    }

    template <class STATE>
    static void Destroy(STATE &state, AggregateInputData &) {
        state.seen_values.~unordered_set();
    }
};
```

## 8. 绑定回调

### 8.1 类型推导

```cpp
// FIRST/LAST 函数的绑定
static unique_ptr<FunctionData> FirstLastBind(ClientContext &context,
                                               AggregateFunction &function,
                                               vector<unique_ptr<Expression>> &arguments) {
    // 返回类型与输入类型相同
    function.return_type = arguments[0]->return_type;

    // 创建绑定数据
    auto bind_data = make_uniq<FirstLastBindData>();
    bind_data->type = function.return_type;
    return bind_data;
}
```

### 8.2 参数验证

```cpp
// PERCENTILE_CONT 函数的绑定
static unique_ptr<FunctionData> PercentileContBind(ClientContext &context,
                                                    AggregateFunction &function,
                                                    vector<unique_ptr<Expression>> &arguments) {
    // 验证百分位参数是常量
    if (!arguments[1]->IsFoldable()) {
        throw BinderException("Percentile must be a constant value");
    }

    // 计算百分位值
    auto percentile_value = ExpressionExecutor::EvaluateScalar(context, *arguments[1]);
    auto percentile = percentile_value.GetValue<double>();

    if (percentile < 0 || percentile > 1) {
        throw BinderException("Percentile must be between 0 and 1");
    }

    auto bind_data = make_uniq<PercentileBindData>();
    bind_data->percentile = percentile;

    // 移除百分位参数（已编译到绑定数据）
    arguments.pop_back();
    function.arguments.pop_back();

    return bind_data;
}
```

## 9. 创建聚合函数的便捷方法

### 9.1 UnaryAggregate 模板

```cpp
template <class STATE, class INPUT_TYPE, class RESULT_TYPE, class OP>
static AggregateFunction UnaryAggregate(const LogicalType &input_type,
                                        LogicalType return_type,
                                        FunctionNullHandling null_handling =
                                            FunctionNullHandling::DEFAULT_NULL_HANDLING) {
    return AggregateFunction(
        {input_type},
        return_type,
        AggregateFunction::StateSize<STATE>,
        AggregateFunction::StateInitialize<STATE, OP>,
        AggregateFunction::UnaryScatterUpdate<STATE, INPUT_TYPE, OP>,
        AggregateFunction::StateCombine<STATE, OP>,
        AggregateFunction::StateFinalize<STATE, RESULT_TYPE, OP>,
        null_handling,
        AggregateFunction::UnaryUpdate<STATE, INPUT_TYPE, OP>);
}
```

### 9.2 BinaryAggregate 模板

```cpp
template <class STATE, class A_TYPE, class B_TYPE, class RESULT_TYPE, class OP>
static AggregateFunction BinaryAggregate(const LogicalType &a_type,
                                         const LogicalType &b_type,
                                         LogicalType return_type) {
    return AggregateFunction(
        {a_type, b_type},
        return_type,
        AggregateFunction::StateSize<STATE>,
        AggregateFunction::StateInitialize<STATE, OP>,
        AggregateFunction::BinaryScatterUpdate<STATE, A_TYPE, B_TYPE, OP>,
        AggregateFunction::StateCombine<STATE, OP>,
        AggregateFunction::StateFinalize<STATE, RESULT_TYPE, OP>,
        AggregateFunction::BinaryUpdate<STATE, A_TYPE, B_TYPE, OP>);
}
```

## 10. 总结

### 10.1 聚合函数设计要点

1. **状态管理**：定义紧凑的状态结构
2. **可合并性**：combine 必须是可交换和可结合的
3. **NULL 处理**：使用 isset 标记或计数
4. **内存管理**：复杂状态需要 destructor
5. **并行友好**：避免状态间依赖

### 10.2 回调函数检查清单

| 回调 | 必需 | 用途 |
|------|------|------|
| state_size | ✅ | 状态内存大小 |
| initialize | ✅ | 状态初始化 |
| update | ✅ | 分散更新 |
| combine | ✅ | 状态合并 |
| finalize | ✅ | 最终计算 |
| simple_update | ⚪ | 非分组优化 |
| destructor | ⚪ | 复杂状态清理 |
| window | ⚪ | 窗口计算 |
| bind | ⚪ | 类型推导 |
| statistics | ⚪ | 统计传播 |

### 10.3 状态设计模式

```cpp
// 模式 1：简单累加
struct SimpleState {
    int64_t value;
};

// 模式 2：带有效标记
struct OptionalState {
    double value;
    bool isset;
};

// 模式 3：多值状态
struct MultiState {
    int64_t sum;
    uint64_t count;
};

// 模式 4：复杂状态（需要 destructor）
struct ComplexState {
    string buffer;
    vector<int64_t> values;
};
```

下一章将深入分析表函数的实现，包括数据源抽象、状态管理和下推优化。
