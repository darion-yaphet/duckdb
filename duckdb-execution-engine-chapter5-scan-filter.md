# DuckDB 执行引擎深度解析 - 第五章：扫描与过滤算子

## 引言

扫描和过滤算子是查询执行的起点，负责从存储层读取数据并应用过滤条件。DuckDB 通过列裁剪（Projection Pushdown）、谓词下推（Filter Pushdown）、动态过滤（Dynamic Filter）等技术，最大限度地减少数据读取量，提升查询性能。

本章深入分析 PhysicalTableScan、PhysicalFilter、PhysicalProjection 等核心算子的实现，以及 TableFilter 机制如何实现谓词下推。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     扫描与过滤算子架构                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       查询优化阶段                                 │  │
│  │  ┌─────────────┐     ┌─────────────┐     ┌─────────────────┐     │  │
│  │  │ 列裁剪      │ → │ 谓词下推     │ → │ 生成 TableFilter  │     │  │
│  │  │ (仅读取    │     │ (WHERE条件  │     │ (传递给存储层)   │     │  │
│  │  │  需要的列) │     │  下推到扫描) │     │                  │     │  │
│  │  └─────────────┘     └─────────────┘     └─────────────────┘     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                   │                                     │
│                                   ▼                                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                       执行阶段                                     │  │
│  │                                                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────┐ │  │
│  │  │                   PhysicalTableScan                          │ │  │
│  │  │  • TableFunction: 数据源接口                                  │ │  │
│  │  │  • column_ids: 需要读取的列索引                               │ │  │
│  │  │  • table_filters: 下推的过滤条件                              │ │  │
│  │  │  • dynamic_filters: 运行时动态过滤                            │ │  │
│  │  └──────────────────────────┬──────────────────────────────────┘ │  │
│  │                              │                                    │  │
│  │                              ▼                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────┐ │  │
│  │  │                   PhysicalFilter                             │ │  │
│  │  │  • expression: 不能下推的复杂过滤条件                         │ │  │
│  │  │  • SelectExpression() → SelectionVector                      │ │  │
│  │  └──────────────────────────┬──────────────────────────────────┘ │  │
│  │                              │                                    │  │
│  │                              ▼                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────┐ │  │
│  │  │                   PhysicalProjection                         │ │  │
│  │  │  • select_list: 投影表达式列表                                │ │  │
│  │  │  • ExpressionExecutor.Execute()                              │ │  │
│  │  └─────────────────────────────────────────────────────────────┘ │  │
│  │                                                                    │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.1 PhysicalTableScan：表扫描算子

PhysicalTableScan 是最常见的数据源算子，负责从存储层读取表数据。它通过 TableFunction 接口与不同的数据源（本地表、Parquet 文件、远程数据等）交互。

### 5.1.1 PhysicalTableScan 结构

```cpp
// src/include/duckdb/execution/operator/scan/physical_table_scan.hpp

class PhysicalTableScan : public PhysicalOperator {
public:
    //! 表函数（数据源接口）
    TableFunction function;

    //! 绑定数据（表函数的上下文）
    unique_ptr<FunctionData> bind_data;

    //! 所有可返回列的类型
    vector<LogicalType> returned_types;

    //! 需要读取的列索引
    vector<ColumnIndex> column_ids;

    //! 投影后保留的列索引（filter_prune 优化）
    vector<idx_t> projection_ids;

    //! 列名
    vector<string> names;

    //! 下推的过滤条件
    unique_ptr<TableFilterSet> table_filters;

    //! 动态过滤条件（来自 Join 等算子）
    shared_ptr<DynamicTableFilterSet> dynamic_filters;

    //! 额外信息（采样率、文件过滤等）
    ExtraOperatorInfo extra_info;

    //! 函数参数
    vector<Value> parameters;
};
```

### 5.1.2 表扫描状态

```cpp
// src/execution/operator/scan/physical_table_scan.cpp

// 全局状态：跨线程共享
class TableScanGlobalSourceState : public GlobalSourceState {
public:
    TableScanGlobalSourceState(ClientContext &context, const PhysicalTableScan &op) {
        // 合并动态过滤器
        if (op.dynamic_filters && op.dynamic_filters->HasFilters()) {
            table_filters = op.dynamic_filters->GetFinalTableFilters(op, op.table_filters.get());
        }

        // 初始化表函数的全局状态
        if (op.function.init_global) {
            auto filters = table_filters ? *table_filters : GetTableFilters(op);
            TableFunctionInitInput input(op.bind_data.get(), op.column_ids, op.projection_ids,
                                         filters, op.extra_info.sample_options, &op);
            global_state = op.function.init_global(context, input);
            if (global_state) {
                max_threads = global_state->MaxThreads();
            }
        }
    }

    idx_t max_threads = 0;
    unique_ptr<GlobalTableFunctionState> global_state;
    unique_ptr<TableFilterSet> table_filters;  // 合并后的过滤器

    idx_t MaxThreads() override {
        return max_threads;
    }
};

// 本地状态：每线程独立
class TableScanLocalSourceState : public LocalSourceState {
public:
    TableScanLocalSourceState(ExecutionContext &context, TableScanGlobalSourceState &gstate,
                              const PhysicalTableScan &op) {
        if (op.function.init_local) {
            TableFunctionInitInput input(op.bind_data.get(), op.column_ids, op.projection_ids,
                                         gstate.GetTableFilters(op), op.extra_info.sample_options, &op);
            local_state = op.function.init_local(context, input, gstate.global_state.get());
        }
    }

    unique_ptr<LocalTableFunctionState> local_state;
};
```

### 5.1.3 GetData 执行逻辑

```cpp
SourceResultType PhysicalTableScan::GetDataInternal(ExecutionContext &context, DataChunk &chunk,
                                                     OperatorSourceInput &input) const {
    D_ASSERT(!column_ids.empty());
    auto &g_state = input.global_state.Cast<TableScanGlobalSourceState>();
    auto &l_state = input.local_state.Cast<TableScanLocalSourceState>();

    // 构造表函数输入
    TableFunctionInput data(bind_data.get(), l_state.local_state.get(), g_state.global_state.get());

    if (function.function) {
        // 调用表函数获取数据
        function.function(context.client, data, chunk);

        // 处理异步结果
        switch (data.async_result.GetResultType()) {
        case AsyncResultType::BLOCKED:
            // 阻塞等待
            return SourceResultType::BLOCKED;
        case AsyncResultType::FINISHED:
            return SourceResultType::FINISHED;
        case AsyncResultType::HAVE_MORE_OUTPUT:
            return SourceResultType::HAVE_MORE_OUTPUT;
        default:
            if (chunk.size() > 0) {
                return SourceResultType::HAVE_MORE_OUTPUT;
            }
            return SourceResultType::FINISHED;
        }
    }

    // 处理 in-out 函数（如表值函数）
    if (function.in_out_function) {
        function.in_out_function(context, data, g_state.input_chunk, chunk);
    }

    return chunk.size() == 0 ? SourceResultType::FINISHED : SourceResultType::HAVE_MORE_OUTPUT;
}
```

---

## 5.2 TableFunction：表函数接口

TableFunction 是 DuckDB 与数据源交互的统一接口，支持内置表、外部文件、远程数据等多种数据源。

### 5.2.1 TableFunction 结构

```cpp
// src/include/duckdb/function/table_function.hpp

struct TableFunction : public SimpleNamedParameterFunction {
    //! 绑定函数：解析参数，准备元数据
    table_function_bind_t bind;

    //! 绑定替换函数：替换表函数为其他逻辑
    table_function_bind_replace_t bind_replace;

    //! 全局状态初始化
    table_function_init_global_t init_global;

    //! 本地状态初始化
    table_function_init_local_t init_local;

    //! 主函数：获取数据
    table_function_t function;

    //! In-Out 函数：处理表值函数
    table_in_out_function_t in_out_function;

    //! 统计信息函数
    table_statistics_t statistics;

    //! 进度函数
    table_function_progress_t table_scan_progress;

    //! 功能标志
    bool projection_pushdown;      // 支持列裁剪
    bool filter_pushdown;          // 支持过滤下推
    bool filter_prune;             // 可以移除过滤列
    bool cardinality_function;     // 提供基数估计
};
```

### 5.2.2 初始化输入

```cpp
struct TableFunctionInitInput {
    //! 绑定数据
    optional_ptr<const FunctionData> bind_data;

    //! 需要读取的列 ID
    vector<column_t> column_ids;
    vector<ColumnIndex> column_indexes;

    //! 投影后保留的列索引
    const vector<idx_t> projection_ids;

    //! 下推的过滤条件
    optional_ptr<TableFilterSet> filters;

    //! 采样选项
    optional_ptr<SampleOptions> sample_options;

    //! 检查是否可以移除过滤列
    bool CanRemoveFilterColumns() const {
        if (projection_ids.empty()) {
            return false;  // 没有过滤列
        }
        if (projection_ids.size() == column_ids.size()) {
            return false;  // 过滤列在后续计划中使用
        }
        return true;  // 可以移除不需要的过滤列
    }
};
```

### 5.2.3 表函数执行流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    TableFunction 执行流程                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 绑定阶段 (Bind)                                                      │
│     ┌─────────────────────────────────────────────────────────────────┐│
│     │ bind(context, input) → FunctionData                             ││
│     │   • 解析函数参数                                                 ││
│     │   • 获取表元数据（列类型、列名）                                  ││
│     │   • 返回 returned_types 和 names                                ││
│     └─────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  2. 全局初始化 (InitGlobal)                                              │
│     ┌─────────────────────────────────────────────────────────────────┐│
│     │ init_global(context, input) → GlobalTableFunctionState          ││
│     │   • 初始化扫描状态                                               ││
│     │   • 确定并行度 (MaxThreads)                                      ││
│     │   • 接收 column_ids 和 filters                                  ││
│     └─────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  3. 本地初始化 (InitLocal)                                               │
│     ┌─────────────────────────────────────────────────────────────────┐│
│     │ init_local(context, input, global_state) → LocalTableFunctionState│
│     │   • 为每个线程初始化本地状态                                      ││
│     │   • 分配线程私有的缓冲区                                         ││
│     └─────────────────────────────────────────────────────────────────┘│
│                                                                         │
│  4. 数据获取 (Function)                                                  │
│     ┌─────────────────────────────────────────────────────────────────┐│
│     │ function(context, input, chunk)                                 ││
│     │   • 读取一批数据到 DataChunk                                     ││
│     │   • 应用 table_filters 过滤                                     ││
│     │   • 只返回 column_ids 指定的列                                  ││
│     │   • 返回空 chunk 表示扫描结束                                    ││
│     └─────────────────────────────────────────────────────────────────┘│
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.3 TableFilter：过滤条件下推

TableFilter 是下推到存储层的过滤条件，允许在数据读取时就进行过滤，避免将不需要的数据传递到上层算子。

### 5.3.1 TableFilter 类型

```cpp
// src/include/duckdb/planner/table_filter.hpp

enum class TableFilterType : uint8_t {
    CONSTANT_COMPARISON = 0,  // 常量比较 (=C, >C, >=C, <C, <=C)
    IS_NULL = 1,              // IS NULL
    IS_NOT_NULL = 2,          // IS NOT NULL
    CONJUNCTION_OR = 3,       // OR 组合
    CONJUNCTION_AND = 4,      // AND 组合
    STRUCT_EXTRACT = 5,       // 结构体子字段过滤
    OPTIONAL_FILTER = 6,      // 可选过滤（优化提示）
    IN_FILTER = 7,            // IN (C1, C2, C3, ...)
    DYNAMIC_FILTER = 8,       // 动态过滤（运行时生成）
    EXPRESSION_FILTER = 9,    // 任意表达式过滤
    BLOOM_FILTER = 10,        // 布隆过滤器
};
```

### 5.3.2 TableFilter 基类

```cpp
class TableFilter {
public:
    explicit TableFilter(TableFilterType filter_type_p) : filter_type(filter_type_p) {}
    virtual ~TableFilter() {}

    TableFilterType filter_type;

public:
    //! 检查统计信息：判断 Segment 是否可能包含满足条件的值
    virtual FilterPropagateResult CheckStatistics(BaseStatistics &stats) const = 0;

    //! 转换为字符串（用于 EXPLAIN）
    virtual string ToString(const string &column_name) const = 0;

    //! 复制过滤器
    virtual unique_ptr<TableFilter> Copy() const = 0;

    //! 转换为表达式（用于执行）
    virtual unique_ptr<Expression> ToExpression(const Expression &column) const = 0;
};
```

### 5.3.3 TableFilterSet

```cpp
//! 非复合过滤器集合（每个过滤器只涉及单列）
class TableFilterSet {
public:
    //! 列索引 → 过滤器的映射
    map<idx_t, unique_ptr<TableFilter>> filters;

public:
    //! 添加过滤器
    void PushFilter(const ColumnIndex &col_idx, unique_ptr<TableFilter> filter);

    //! 复制整个过滤器集合
    unique_ptr<TableFilterSet> Copy() const;
};
```

### 5.3.4 过滤器传播结果

```cpp
enum class FilterPropagateResult : uint8_t {
    NO_PRUNING_POSSIBLE = 0,     // 无法剪枝：必须扫描
    FILTER_ALWAYS_TRUE = 1,      // 过滤器恒真：可以跳过检查
    FILTER_ALWAYS_FALSE = 2,     // 过滤器恒假：可以跳过整个 Segment
    FILTER_TRUE_OR_NULL = 3,     // 结果可能是 TRUE 或 NULL
    FILTER_FALSE_OR_NULL = 4,    // 结果可能是 FALSE 或 NULL
};
```

### 5.3.5 Zonemap 过滤

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Zonemap 过滤（Segment 跳过）                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景：WHERE age > 50                                                   │
│                                                                         │
│  Segment 统计信息:                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Segment 1: min=18, max=35   ← age > 50 恒假，跳过整个 Segment    │   │
│  │ Segment 2: min=30, max=65   ← 可能包含匹配值，需要扫描           │   │
│  │ Segment 3: min=55, max=80   ← age > 50 可能为真，需要扫描        │   │
│  │ Segment 4: min=22, max=45   ← age > 50 恒假，跳过整个 Segment    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  CheckStatistics 实现:                                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ FilterPropagateResult CheckStatistics(BaseStatistics &stats) {  │   │
│  │     auto &col_stats = stats.Cast<NumericStatistics>();          │   │
│  │     auto min_val = col_stats.GetMin();                          │   │
│  │     auto max_val = col_stats.GetMax();                          │   │
│  │                                                                  │   │
│  │     // 如果 max < 过滤值，整个 Segment 都不满足                   │   │
│  │     if (max_val < filter_value) {                               │   │
│  │         return FilterPropagateResult::FILTER_ALWAYS_FALSE;      │   │
│  │     }                                                            │   │
│  │     // 如果 min > 过滤值，整个 Segment 都满足                     │   │
│  │     if (min_val > filter_value) {                               │   │
│  │         return FilterPropagateResult::FILTER_ALWAYS_TRUE;       │   │
│  │     }                                                            │   │
│  │     // 否则需要逐行检查                                          │   │
│  │     return FilterPropagateResult::NO_PRUNING_POSSIBLE;          │   │
│  │ }                                                                │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  优化效果:                                                               │
│  • 跳过不可能包含匹配值的 Segment                                        │
│  • 减少 I/O 和解压缩开销                                                │
│  • 对有序或聚类数据特别有效                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.4 动态过滤 (Dynamic Filter)

动态过滤是运行时生成的过滤条件，通常来自 Join 操作的 Build 端。

### 5.4.1 DynamicTableFilterSet

```cpp
// src/include/duckdb/planner/table_filter.hpp

class DynamicTableFilterSet {
public:
    //! 清除某个算子的过滤器
    void ClearFilters(const PhysicalOperator &op);

    //! 添加动态过滤器
    void PushFilter(const PhysicalOperator &op, idx_t column_index, unique_ptr<TableFilter> filter);

    //! 检查是否有过滤器
    bool HasFilters() const;

    //! 获取最终的过滤器集合（合并静态和动态）
    unique_ptr<TableFilterSet> GetFinalTableFilters(
        const PhysicalTableScan &scan,
        optional_ptr<TableFilterSet> existing_filters) const;

private:
    mutable mutex lock;
    reference_map_t<const PhysicalOperator, unique_ptr<TableFilterSet>> filters;
};
```

### 5.4.2 动态过滤工作流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    动态过滤工作流程                                       │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景：SELECT * FROM orders o JOIN customers c ON o.cid = c.id          │
│        WHERE c.country = 'China'                                        │
│                                                                         │
│  执行计划:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ HashJoin (o.cid = c.id)                                         │   │
│  │   ├── Probe: TableScan(orders)  ← 动态过滤接收方                 │   │
│  │   └── Build: TableScan(customers) → Filter(country='China')     │   │
│  │              ↑ 动态过滤生成方                                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  执行过程:                                                               │
│  ─────────                                                              │
│  1. Build 阶段：扫描 customers，应用 country='China' 过滤               │
│     结果：只有中国客户的 id 集合 {1001, 1005, 1008, ...}                │
│                                                                         │
│  2. 生成动态过滤器：                                                     │
│     • 如果 id 集合较小：生成 IN (1001, 1005, 1008, ...) 过滤器          │
│     • 如果 id 集合较大：生成 Bloom Filter                               │
│     • 推送到 orders 表的扫描算子                                        │
│                                                                         │
│  3. Probe 阶段：扫描 orders 时应用动态过滤器                             │
│     • 跳过 cid 不在集合中的行                                           │
│     • 大幅减少需要 Join 的数据量                                        │
│                                                                         │
│  代码实现:                                                               │
│  ─────────                                                              │
│  // Build 完成后生成动态过滤器                                          │
│  if (scan->dynamic_filters) {                                          │
│      auto bloom_filter = CreateBloomFilter(build_keys);                │
│      scan->dynamic_filters->PushFilter(*this, key_column_idx,          │
│                                         std::move(bloom_filter));       │
│  }                                                                      │
│                                                                         │
│  // 扫描时合并动态过滤器                                                 │
│  if (op.dynamic_filters && op.dynamic_filters->HasFilters()) {         │
│      table_filters = op.dynamic_filters->GetFinalTableFilters(         │
│          op, op.table_filters.get());                                  │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.5 PhysicalFilter：过滤算子

当过滤条件无法完全下推到存储层时（如涉及多列的复杂条件），PhysicalFilter 负责在执行层进行过滤。

### 5.5.1 PhysicalFilter 结构

```cpp
// src/execution/operator/filter/physical_filter.cpp

class PhysicalFilter : public CachingPhysicalOperator {
public:
    //! 过滤表达式
    unique_ptr<Expression> expression;

    PhysicalFilter(PhysicalPlan &physical_plan, vector<LogicalType> types,
                   vector<unique_ptr<Expression>> select_list, idx_t estimated_cardinality)
        : CachingPhysicalOperator(physical_plan, PhysicalOperatorType::FILTER,
                                  std::move(types), estimated_cardinality) {
        D_ASSERT(!select_list.empty());

        if (select_list.size() == 1) {
            // 单个条件
            expression = std::move(select_list[0]);
        } else {
            // 多个条件组合为 AND
            auto conjunction = make_uniq<BoundConjunctionExpression>(
                ExpressionType::CONJUNCTION_AND);
            for (auto &expr : select_list) {
                conjunction->children.push_back(std::move(expr));
            }
            expression = std::move(conjunction);
        }
    }
};
```

### 5.5.2 FilterState

```cpp
class FilterState : public CachingOperatorState {
public:
    explicit FilterState(ExecutionContext &context, Expression &expr)
        : executor(context.client, expr), sel(STANDARD_VECTOR_SIZE) {}

    //! 表达式执行器
    ExpressionExecutor executor;

    //! 选择向量（存储满足条件的行索引）
    SelectionVector sel;
};
```

### 5.5.3 Execute 实现

```cpp
OperatorResultType PhysicalFilter::ExecuteInternal(ExecutionContext &context, DataChunk &input,
                                                    DataChunk &chunk, GlobalOperatorState &gstate,
                                                    OperatorState &state_p) const {
    auto &state = state_p.Cast<FilterState>();

    // 使用 Select 模式执行布尔表达式
    // 返回满足条件的行数，并填充 sel
    idx_t result_count = state.executor.SelectExpression(input, state.sel);

    if (result_count == input.size()) {
        // 没有行被过滤：直接引用输入
        chunk.Reference(input);
    } else {
        // 有行被过滤：使用选择向量切片
        chunk.Slice(input, state.sel, result_count);
    }

    return OperatorResultType::NEED_MORE_INPUT;
}
```

### 5.5.4 过滤执行流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    PhysicalFilter 执行流程                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  输入 DataChunk (2048 行):                                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ col_a: [1, 2, 3, 4, 5, 6, 7, 8, ...]                            │   │
│  │ col_b: [10, 25, 5, 30, 15, 8, 22, 40, ...]                      │   │
│  │ col_c: ['A', 'B', 'A', 'C', 'B', 'A', 'C', 'B', ...]           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  过滤条件: col_b > 20 AND col_c = 'B'                                   │
│                                                                         │
│  执行步骤:                                                               │
│  ─────────                                                              │
│  1. SelectExpression(input, sel)                                       │
│     • 对 input 执行布尔表达式                                           │
│     • 填充 sel: [1, 7, ...]  (满足条件的行索引)                         │
│     • 返回 result_count = 2                                            │
│                                                                         │
│  2. 判断过滤效果:                                                        │
│     • 如果 result_count == input.size(): 无过滤，Reference(input)       │
│     • 否则: Slice(input, sel, result_count)                            │
│                                                                         │
│  输出 DataChunk (2 行):                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ col_a: [2, 8]                                                   │   │
│  │ col_b: [25, 40]                                                 │   │
│  │ col_c: ['B', 'B']                                               │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  关键优化:                                                               │
│  ─────────                                                              │
│  • 使用 Select 模式而非 Execute 模式：直接生成 SelectionVector          │
│  • 无行被过滤时使用 Reference：零复制                                    │
│  • 有行被过滤时使用 Slice：创建 DICTIONARY_VECTOR                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.6 PhysicalProjection：投影算子

PhysicalProjection 负责计算投影表达式，产生输出列。

### 5.6.1 PhysicalProjection 结构

```cpp
// src/execution/operator/projection/physical_projection.cpp

class PhysicalProjection : public PhysicalOperator {
public:
    //! 投影表达式列表
    vector<unique_ptr<Expression>> select_list;

    PhysicalProjection(PhysicalPlan &physical_plan, vector<LogicalType> types,
                       vector<unique_ptr<Expression>> select_list, idx_t estimated_cardinality)
        : PhysicalOperator(physical_plan, PhysicalOperatorType::PROJECTION,
                          std::move(types), estimated_cardinality),
          select_list(std::move(select_list)) {}
};
```

### 5.6.2 ProjectionState

```cpp
class ProjectionState : public OperatorState {
public:
    explicit ProjectionState(ExecutionContext &context,
                             const vector<unique_ptr<Expression>> &expressions)
        : executor(context.client, expressions) {}

    //! 表达式执行器
    ExpressionExecutor executor;
};
```

### 5.6.3 Execute 实现

```cpp
OperatorResultType PhysicalProjection::Execute(ExecutionContext &context, DataChunk &input,
                                                DataChunk &chunk, GlobalOperatorState &gstate,
                                                OperatorState &state_p) const {
    auto &state = state_p.Cast<ProjectionState>();

    // 执行所有投影表达式
    state.executor.Execute(input, chunk);

    return OperatorResultType::NEED_MORE_INPUT;
}
```

### 5.6.4 投影表达式类型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    投影表达式类型                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 简单列引用                                                           │
│     SELECT col_a, col_b FROM table                                     │
│     • 使用 BoundReferenceExpression                                     │
│     • 直接引用输入列，零复制                                             │
│                                                                         │
│  2. 算术表达式                                                           │
│     SELECT col_a + col_b * 2 FROM table                                │
│     • 使用 BoundFunctionExpression                                      │
│     • 递归计算子表达式                                                   │
│                                                                         │
│  3. 函数调用                                                             │
│     SELECT UPPER(name), YEAR(date) FROM table                          │
│     • 使用 BoundFunctionExpression                                      │
│     • 调用标量函数                                                       │
│                                                                         │
│  4. CASE 表达式                                                          │
│     SELECT CASE WHEN a > 0 THEN 'positive' ELSE 'negative' END         │
│     • 使用 BoundCaseExpression                                          │
│     • 条件分支执行                                                       │
│                                                                         │
│  5. 常量                                                                 │
│     SELECT 1, 'hello' FROM table                                       │
│     • 使用 BoundConstantExpression                                      │
│     • 返回 CONSTANT_VECTOR                                              │
│                                                                         │
│  6. 类型转换                                                             │
│     SELECT CAST(col AS VARCHAR) FROM table                             │
│     • 使用 BoundCastExpression                                          │
│     • 调用类型转换函数                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.7 PhysicalColumnDataScan：中间数据扫描

PhysicalColumnDataScan 用于扫描存储在内存中的中间结果，常见于 CTE、子查询物化等场景。

### 5.7.1 结构与状态

```cpp
// src/execution/operator/scan/physical_column_data_scan.cpp

class PhysicalColumnDataScan : public PhysicalOperator {
public:
    //! 要扫描的列数据集合
    optionally_owned_ptr<ColumnDataCollection> collection;

    //! CTE 索引（用于递归 CTE）
    idx_t cte_index;
};

// 全局状态
class PhysicalColumnDataGlobalScanState : public GlobalSourceState {
public:
    explicit PhysicalColumnDataGlobalScanState(const ColumnDataCollection &collection)
        : max_threads(MaxValue<idx_t>(collection.ChunkCount(), 1)) {
        collection.InitializeScan(global_scan_state);
    }

    ColumnDataParallelScanState global_scan_state;
    const idx_t max_threads;

    idx_t MaxThreads() override { return max_threads; }
};

// 本地状态
class PhysicalColumnDataLocalScanState : public LocalSourceState {
public:
    ColumnDataLocalScanState local_scan_state;
};
```

### 5.7.2 GetData 实现

```cpp
SourceResultType PhysicalColumnDataScan::GetDataInternal(ExecutionContext &context,
                                                          DataChunk &chunk,
                                                          OperatorSourceInput &input) const {
    auto &gstate = input.global_state.Cast<PhysicalColumnDataGlobalScanState>();
    auto &lstate = input.local_state.Cast<PhysicalColumnDataLocalScanState>();

    // 从 ColumnDataCollection 扫描数据
    collection->Scan(gstate.global_scan_state, lstate.local_scan_state, chunk);

    return chunk.size() == 0 ? SourceResultType::FINISHED : SourceResultType::HAVE_MORE_OUTPUT;
}
```

### 5.7.3 Pipeline 依赖

```cpp
void PhysicalColumnDataScan::BuildPipelines(Pipeline &current, MetaPipeline &meta_pipeline) {
    auto &state = meta_pipeline.GetState();

    switch (type) {
    case PhysicalOperatorType::DELIM_SCAN: {
        // Delim Join 依赖
        auto entry = state.delim_join_dependencies.find(*this);
        auto delim_dependency = entry->second.get().shared_from_this();
        current.AddDependency(delim_dependency);
        // ...
        return;
    }
    case PhysicalOperatorType::CTE_SCAN: {
        // CTE 依赖
        auto entry = state.cte_dependencies.find(*this);
        auto cte_dependency = entry->second.get().shared_from_this();
        current.AddDependency(cte_dependency);
        // ...
        return;
    }
    // ...
    }

    state.SetPipelineSource(current, *this);
}
```

---

## 5.8 列裁剪优化 (Projection Pushdown)

列裁剪将投影操作下推到扫描层，只读取查询需要的列。

### 5.8.1 列裁剪示例

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    列裁剪优化                                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  原始查询:                                                               │
│  SELECT name, age FROM users WHERE country = 'China'                   │
│                                                                         │
│  表结构:                                                                 │
│  users(id INT, name VARCHAR, age INT, email VARCHAR, country VARCHAR)  │
│                                                                         │
│  无列裁剪:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ TableScan(users) → [id, name, age, email, country]              │   │
│  │ Filter(country = 'China')                                       │   │
│  │ Projection(name, age)                                           │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  问题：读取所有 5 列，浪费 I/O                                           │
│                                                                         │
│  有列裁剪:                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ TableScan(users)                                                │   │
│  │   column_ids = [1, 2, 4]  ← 只读取 name, age, country          │   │
│  │   table_filters = {4: country = 'China'}                       │   │
│  │ Projection(name, age)  ← 移除过滤列 country                     │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│  优势：只读取 3 列，减少 40% I/O                                         │
│                                                                         │
│  实现机制:                                                               │
│  ─────────                                                              │
│  1. column_ids: 需要读取的列索引 [1, 2, 4]                              │
│  2. projection_ids: 最终输出的列索引 [0, 1]（对应 column_ids 的子集）   │
│  3. filter_prune = true: 允许移除过滤后不需要的列                       │
│                                                                         │
│  TableFunctionInitInput::CanRemoveFilterColumns() 判断:                 │
│  • projection_ids.size() < column_ids.size() → 可以移除过滤列           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.9 扫描与过滤的 Pipeline 集成

### 5.9.1 典型 Pipeline 结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│           扫描 → 过滤 → 投影 Pipeline                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SQL: SELECT name, salary * 1.1 AS new_salary                          │
│       FROM employees                                                    │
│       WHERE department = 'Engineering' AND salary > 50000              │
│                                                                         │
│  Pipeline 结构:                                                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Source: PhysicalTableScan(employees)                            │   │
│  │   • column_ids = [name, salary, department]                     │   │
│  │   • table_filters = {department = 'Engineering'}                │   │
│  │                      ↑ 可下推的简单条件                          │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Operator: PhysicalFilter                                        │   │
│  │   • expression = salary > 50000                                 │   │
│  │                  ↑ 无法下推（或选择不下推）的条件                 │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Operator: PhysicalProjection                                    │   │
│  │   • select_list = [                                             │   │
│  │       BoundReferenceExpression(name),                           │   │
│  │       BoundFunctionExpression(salary * 1.1)                     │   │
│  │     ]                                                            │   │
│  └────────────────────────────┬────────────────────────────────────┘   │
│                               │                                         │
│                               ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ Sink: Result                                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.9.2 并行扫描

```cpp
// PhysicalTableScan 支持并行扫描
bool PhysicalTableScan::ParallelSource() const {
    if (!function.function) {
        // in-out 函数不支持并行
        return false;
    }
    return true;  // 普通表函数支持并行
}

// 全局状态提供并行度
idx_t TableScanGlobalSourceState::MaxThreads() override {
    return max_threads;  // 由表函数决定
}
```

---

## 5.10 性能优化总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    扫描与过滤优化技术                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 列裁剪 (Projection Pushdown)                                         │
│     • 只读取查询需要的列                                                 │
│     • 减少 I/O 和内存使用                                                │
│     • filter_prune: 移除过滤后不需要的列                                 │
│                                                                         │
│  2. 谓词下推 (Filter Pushdown)                                           │
│     • 将简单条件下推到存储层                                             │
│     • 支持 =, <, >, <=, >=, IS NULL, IS NOT NULL, IN                   │
│     • 减少返回到执行层的数据量                                           │
│                                                                         │
│  3. Zonemap 过滤                                                         │
│     • 利用 Segment 统计信息跳过不匹配的数据块                            │
│     • 对有序/聚类数据特别有效                                            │
│     • 结合压缩减少解压缩开销                                             │
│                                                                         │
│  4. 动态过滤 (Dynamic Filter)                                            │
│     • 运行时从 Join Build 端生成过滤条件                                 │
│     • Bloom Filter 或 IN 列表                                            │
│     • 大幅减少 Join Probe 端的数据量                                     │
│                                                                         │
│  5. 并行扫描                                                             │
│     • 多线程并行读取不同的数据分区                                        │
│     • GlobalState 协调任务分配                                           │
│     • LocalState 存储线程私有状态                                        │
│                                                                         │
│  6. Select 模式过滤                                                      │
│     • 直接生成 SelectionVector 而非布尔 Vector                           │
│     • 避免不必要的数据复制                                               │
│     • 无行过滤时零复制 Reference                                         │
│                                                                         │
│  7. 延迟物化                                                             │
│     • 过滤后再读取需要的列                                               │
│     • 使用 SelectionVector 跟踪有效行                                    │
│     • 减少不必要的数据移动                                               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5.11 总结

本章深入分析了 DuckDB 的扫描与过滤算子：

| 算子 | 职责 | 关键特性 |
|------|------|----------|
| PhysicalTableScan | 表数据源 | TableFunction 接口 + 列裁剪 + 谓词下推 |
| TableFunction | 数据源抽象 | 支持多种数据源（本地/外部/远程） |
| TableFilterSet | 过滤条件集 | 下推到存储层的过滤器 |
| DynamicTableFilterSet | 动态过滤 | 运行时生成（Join 优化） |
| PhysicalFilter | 执行层过滤 | ExpressionExecutor + SelectionVector |
| PhysicalProjection | 投影计算 | ExpressionExecutor 执行投影表达式 |
| PhysicalColumnDataScan | 中间数据扫描 | CTE/物化子查询 |

---

## 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| PhysicalTableScan | `src/execution/operator/scan/physical_table_scan.cpp` |
| PhysicalTableScan Header | `src/include/duckdb/execution/operator/scan/physical_table_scan.hpp` |
| PhysicalFilter | `src/execution/operator/filter/physical_filter.cpp` |
| PhysicalProjection | `src/execution/operator/projection/physical_projection.cpp` |
| PhysicalColumnDataScan | `src/execution/operator/scan/physical_column_data_scan.cpp` |
| TableFilter | `src/include/duckdb/planner/table_filter.hpp` |
| TableFunction | `src/include/duckdb/function/table_function.hpp` |

---

## 下一章预告

第六章将深入分析 **Join 算子实现**，探讨 DuckDB 的 Hash Join、Merge Join、Nested Loop Join 等算法，以及 Build/Probe 端优化和动态过滤生成机制。
