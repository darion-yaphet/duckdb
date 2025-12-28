# DuckDB 函数系统深度解析（四）：表函数与数据源抽象

## 引言

表函数（Table Function）是 DuckDB 中用于生成表数据的函数类型，是数据源抽象的核心机制。从 `read_csv()` 读取文件到 `generate_series()` 生成序列，表函数为各种数据源提供了统一的接口。本章深入分析表函数的核心结构、状态管理模型和下推优化支持。

## 1. TableFunction 核心结构

### 1.1 类定义

```
源文件: src/include/duckdb/function/table_function.hpp
```

```cpp
class TableFunction : public SimpleNamedParameterFunction {
public:
    TableFunction(string name,
                  const vector<LogicalType> &arguments,
                  table_function_t function,
                  table_function_bind_t bind = nullptr,
                  table_function_init_global_t init_global = nullptr,
                  table_function_init_local_t init_local = nullptr);

public:
    //! 绑定函数：确定列定义和参数
    table_function_bind_t bind;
    //! 绑定替换：返回替代 TableRef
    table_function_bind_replace_t bind_replace;
    //! 绑定操作符：返回自定义 LogicalOperator
    table_function_bind_operator_t bind_operator;
    //! 全局状态初始化
    table_function_init_global_t init_global;
    //! 局部状态初始化
    table_function_init_local_t init_local;
    //! 主函数：生成数据
    table_function_t function;
    //! In-Out 函数（接收输入）
    table_in_out_function_t in_out_function;
    //! In-Out 最终函数
    table_in_out_function_final_t in_out_function_final;
    //! 列统计信息
    table_statistics_t statistics;
    //! 依赖关系
    table_function_dependency_t dependency;
    //! 基数估算
    table_function_cardinality_t cardinality;
    //! 复杂过滤下推
    table_function_pushdown_complex_filter_t pushdown_complex_filter;
    //! 表达式下推支持
    table_function_pushdown_expression_t pushdown_expression;
    //! 字符串表示
    table_function_to_string_t to_string;
    //! 动态字符串表示
    table_function_dynamic_to_string_t dynamic_to_string;
    //! 扫描进度
    table_function_progress_t table_scan_progress;
    //! 分区数据
    table_function_get_partition_data_t get_partition_data;
    //! 获取绑定信息
    table_function_get_bind_info_t get_bind_info;
    //! 类型下推
    table_function_type_pushdown_t type_pushdown;
    //! 多文件读取器
    table_function_get_multi_file_reader_t get_multi_file_reader;
    //! 支持的下推类型
    table_function_supports_pushdown_type_t supports_pushdown_type;
    //! 分区信息
    table_function_get_partition_info_t get_partition_info;
    //! 分区统计
    table_function_get_partition_stats_t get_partition_stats;
    //! 虚拟列
    table_function_get_virtual_columns_t get_virtual_columns;
    //! 行 ID 列
    table_function_get_row_id_columns get_row_id_columns;
    //! 扫描顺序
    table_function_set_scan_order set_scan_order;

    //! 序列化/反序列化
    table_function_serialize_t serialize;
    table_function_deserialize_t deserialize;

    //! 下推支持标志
    bool projection_pushdown;  // 列裁剪
    bool filter_pushdown;      // 过滤下推
    bool filter_prune;         // 过滤列剪枝
    bool sampling_pushdown;    // 采样下推
    bool late_materialization; // 延迟物化

    //! 额外信息
    shared_ptr<TableFunctionInfo> function_info;
    //! 顺序保持类型
    OrderPreservationType order_preservation_type;
    //! 初始化时机
    TableFunctionInitialization global_initialization;
};
```

### 1.2 回调函数类型

```cpp
//! 绑定函数：返回 FunctionData 和列定义
typedef unique_ptr<FunctionData> (*table_function_bind_t)(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names);

//! 全局状态初始化
typedef unique_ptr<GlobalTableFunctionState> (*table_function_init_global_t)(
    ClientContext &context,
    TableFunctionInitInput &input);

//! 局部状态初始化
typedef unique_ptr<LocalTableFunctionState> (*table_function_init_local_t)(
    ExecutionContext &context,
    TableFunctionInitInput &input,
    GlobalTableFunctionState *global_state);

//! 主函数：填充输出 DataChunk
typedef void (*table_function_t)(
    ClientContext &context,
    TableFunctionInput &data,
    DataChunk &output);

//! 基数估算
typedef unique_ptr<NodeStatistics> (*table_function_cardinality_t)(
    ClientContext &context,
    const FunctionData *bind_data);

//! 扫描进度
typedef double (*table_function_progress_t)(
    ClientContext &context,
    const FunctionData *bind_data,
    const GlobalTableFunctionState *global_state);
```

## 2. 状态管理模型

### 2.1 三层状态结构

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Table Function State Model                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    FunctionData (bind_data)                  │   │
│  │  - 绑定时创建，不可变                                        │   │
│  │  - 存储参数信息、文件路径、列定义                            │   │
│  │  - 所有线程共享                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│                              ▼                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │               GlobalTableFunctionState                       │   │
│  │  - 执行时创建，跨线程共享                                    │   │
│  │  - 管理全局进度、文件列表、分区分配                          │   │
│  │  - 需要线程安全                                              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                              │                                      │
│              ┌───────────────┼───────────────┐                     │
│              ▼               ▼               ▼                     │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐      │
│  │ LocalState T1   │ │ LocalState T2   │ │ LocalState T3   │      │
│  │ - 线程私有      │ │ - 线程私有      │ │ - 线程私有      │      │
│  │ - 当前文件/块   │ │ - 当前文件/块   │ │ - 当前文件/块   │      │
│  │ - 解析器状态    │ │ - 解析器状态    │ │ - 解析器状态    │      │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 GlobalTableFunctionState

```cpp
struct GlobalTableFunctionState {
public:
    // 最大线程数
    constexpr static const int64_t MAX_THREADS = 999999999;

public:
    virtual ~GlobalTableFunctionState();

    //! 返回可使用的最大线程数
    virtual idx_t MaxThreads() const {
        return 1;  // 默认单线程
    }

    template <class TARGET>
    TARGET &Cast() { ... }
};

// 示例：CSV 读取器全局状态
struct CSVGlobalState : public GlobalTableFunctionState {
    //! 文件列表
    vector<string> files;
    //! 当前文件索引（原子变量）
    atomic<idx_t> current_file_idx;
    //! 文件大小总和
    idx_t total_size;
    //! 已处理大小
    atomic<idx_t> processed_size;

    idx_t MaxThreads() const override {
        return files.size();  // 每个文件一个线程
    }
};
```

### 2.3 LocalTableFunctionState

```cpp
struct LocalTableFunctionState {
    virtual ~LocalTableFunctionState();

    template <class TARGET>
    TARGET &Cast() { ... }
};

// 示例：CSV 读取器局部状态
struct CSVLocalState : public LocalTableFunctionState {
    //! 当前文件路径
    string current_file;
    //! 文件句柄
    unique_ptr<FileHandle> file_handle;
    //! CSV 解析器
    unique_ptr<CSVParser> parser;
    //! 当前行号
    idx_t current_row;
    //! 是否已完成
    bool finished;
};
```

## 3. 绑定输入与输出

### 3.1 TableFunctionBindInput

```cpp
struct TableFunctionBindInput {
    TableFunctionBindInput(vector<Value> &inputs,
                           named_parameter_map_t &named_parameters,
                           vector<LogicalType> &input_table_types,
                           vector<string> &input_table_names,
                           optional_ptr<TableFunctionInfo> info,
                           optional_ptr<Binder> binder,
                           TableFunction &table_function,
                           const TableFunctionRef &ref);

    //! 位置参数
    vector<Value> &inputs;
    //! 命名参数
    named_parameter_map_t &named_parameters;
    //! 输入表类型（用于 In-Out 函数）
    vector<LogicalType> &input_table_types;
    //! 输入表列名
    vector<string> &input_table_names;
    //! 额外函数信息
    optional_ptr<TableFunctionInfo> info;
    //! 绑定器
    optional_ptr<Binder> binder;
    //! 表函数引用
    TableFunction &table_function;
    //! SQL 中的函数引用
    const TableFunctionRef &ref;
};
```

### 3.2 TableFunctionInitInput

```cpp
struct TableFunctionInitInput {
    TableFunctionInitInput(optional_ptr<const FunctionData> bind_data_p,
                           vector<column_t> column_ids_p,
                           const vector<idx_t> &projection_ids_p,
                           optional_ptr<TableFilterSet> filters_p,
                           optional_ptr<SampleOptions> sample_options_p = nullptr,
                           optional_ptr<const PhysicalOperator> op_p = nullptr);

    //! 绑定数据
    optional_ptr<const FunctionData> bind_data;
    //! 需要的列 ID
    vector<column_t> column_ids;
    //! 列索引
    vector<ColumnIndex> column_indexes;
    //! 投影 ID
    const vector<idx_t> projection_ids;
    //! 过滤器集合
    optional_ptr<TableFilterSet> filters;
    //! 采样选项
    optional_ptr<SampleOptions> sample_options;
    //! 物理操作符
    optional_ptr<const PhysicalOperator> op;

    //! 是否可以移除过滤列
    bool CanRemoveFilterColumns() const;
};
```

### 3.3 TableFunctionInput

```cpp
struct TableFunctionInput {
    TableFunctionInput(optional_ptr<const FunctionData> bind_data_p,
                       optional_ptr<LocalTableFunctionState> local_state_p,
                       optional_ptr<GlobalTableFunctionState> global_state_p);

    //! 绑定数据
    optional_ptr<const FunctionData> bind_data;
    //! 局部状态
    optional_ptr<LocalTableFunctionState> local_state;
    //! 全局状态
    optional_ptr<GlobalTableFunctionState> global_state;
    //! 异步结果
    AsyncResult async_result;
    //! 异步执行模式
    AsyncResultsExecutionMode results_execution_mode;
};
```

## 4. 典型表函数实现

### 4.1 generate_series 函数

```cpp
// 绑定数据
struct GenerateSeriesBindData : public TableFunctionData {
    int64_t start;
    int64_t stop;
    int64_t step;
};

// 全局状态
struct GenerateSeriesGlobalState : public GlobalTableFunctionState {
    atomic<int64_t> current;
    int64_t stop;
    int64_t step;

    idx_t MaxThreads() const override {
        // 串行生成
        return 1;
    }
};

// 绑定函数
static unique_ptr<FunctionData> GenerateSeriesBind(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names) {

    auto result = make_uniq<GenerateSeriesBindData>();

    // 解析参数
    result->start = input.inputs[0].GetValue<int64_t>();
    result->stop = input.inputs[1].GetValue<int64_t>();
    result->step = input.inputs.size() > 2
                       ? input.inputs[2].GetValue<int64_t>()
                       : 1;

    if (result->step == 0) {
        throw BinderException("Step cannot be zero");
    }

    // 定义返回列
    return_types.push_back(LogicalType::BIGINT);
    names.push_back("generate_series");

    return std::move(result);
}

// 全局初始化
static unique_ptr<GlobalTableFunctionState> GenerateSeriesInitGlobal(
    ClientContext &context,
    TableFunctionInitInput &input) {

    auto &bind_data = input.bind_data->Cast<GenerateSeriesBindData>();
    auto result = make_uniq<GenerateSeriesGlobalState>();

    result->current = bind_data.start;
    result->stop = bind_data.stop;
    result->step = bind_data.step;

    return std::move(result);
}

// 主函数
static void GenerateSeriesFunction(
    ClientContext &context,
    TableFunctionInput &data,
    DataChunk &output) {

    auto &state = data.global_state->Cast<GenerateSeriesGlobalState>();

    idx_t count = 0;
    auto result_data = FlatVector::GetData<int64_t>(output.data[0]);

    while (count < STANDARD_VECTOR_SIZE) {
        int64_t current = state.current.fetch_add(state.step);

        // 检查是否超出范围
        if (state.step > 0) {
            if (current > state.stop) break;
        } else {
            if (current < state.stop) break;
        }

        result_data[count++] = current;
    }

    output.SetCardinality(count);
}

// 基数估算
static unique_ptr<NodeStatistics> GenerateSeriesCardinality(
    ClientContext &context,
    const FunctionData *bind_data_p) {

    auto &bind_data = bind_data_p->Cast<GenerateSeriesBindData>();

    int64_t count = (bind_data.stop - bind_data.start) / bind_data.step + 1;
    if (count < 0) count = 0;

    return make_uniq<NodeStatistics>(count, count);
}

// 注册函数
TableFunction GenerateSeriesFun::GetFunction() {
    TableFunction func("generate_series",
                       {LogicalType::BIGINT, LogicalType::BIGINT},
                       GenerateSeriesFunction,
                       GenerateSeriesBind,
                       GenerateSeriesInitGlobal);
    func.cardinality = GenerateSeriesCardinality;
    return func;
}
```

### 4.2 duckdb_tables 系统函数

```cpp
// 绑定数据
struct DuckDBTablesBindData : public TableFunctionData {
    vector<CatalogEntry *> entries;
    idx_t current_idx;
};

// 绑定函数
static unique_ptr<FunctionData> DuckDBTablesBind(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names) {

    auto result = make_uniq<DuckDBTablesBindData>();

    // 定义返回列
    names.emplace_back("database_name");
    return_types.emplace_back(LogicalType::VARCHAR);

    names.emplace_back("database_oid");
    return_types.emplace_back(LogicalType::BIGINT);

    names.emplace_back("schema_name");
    return_types.emplace_back(LogicalType::VARCHAR);

    names.emplace_back("schema_oid");
    return_types.emplace_back(LogicalType::BIGINT);

    names.emplace_back("table_name");
    return_types.emplace_back(LogicalType::VARCHAR);

    names.emplace_back("table_oid");
    return_types.emplace_back(LogicalType::BIGINT);

    names.emplace_back("internal");
    return_types.emplace_back(LogicalType::BOOLEAN);

    names.emplace_back("temporary");
    return_types.emplace_back(LogicalType::BOOLEAN);

    names.emplace_back("column_count");
    return_types.emplace_back(LogicalType::BIGINT);

    names.emplace_back("estimated_size");
    return_types.emplace_back(LogicalType::BIGINT);

    // 收集所有表
    auto &catalog = Catalog::GetCatalog(context, INVALID_CATALOG);
    catalog.ScanSchemas(context, [&](SchemaCatalogEntry &schema) {
        schema.Scan(context, CatalogType::TABLE_ENTRY,
                    [&](CatalogEntry &entry) {
            result->entries.push_back(&entry);
        });
    });

    result->current_idx = 0;
    return std::move(result);
}

// 主函数
static void DuckDBTablesFunction(
    ClientContext &context,
    TableFunctionInput &data,
    DataChunk &output) {

    auto &bind_data = data.bind_data->Cast<DuckDBTablesBindData>();

    idx_t count = 0;
    while (bind_data.current_idx < bind_data.entries.size() &&
           count < STANDARD_VECTOR_SIZE) {

        auto entry = bind_data.entries[bind_data.current_idx++];
        auto &table = entry->Cast<TableCatalogEntry>();

        // 填充列数据
        output.SetValue(0, count, table.catalog.GetName());
        output.SetValue(1, count, Value::BIGINT(table.catalog.GetOid()));
        output.SetValue(2, count, table.schema.name);
        output.SetValue(3, count, Value::BIGINT(table.schema.oid));
        output.SetValue(4, count, table.name);
        output.SetValue(5, count, Value::BIGINT(table.oid));
        output.SetValue(6, count, table.internal);
        output.SetValue(7, count, table.temporary);
        output.SetValue(8, count, Value::BIGINT(table.GetColumns().LogicalColumnCount()));
        output.SetValue(9, count, Value::BIGINT(table.GetStorageInfo(context).estimated_size));

        count++;
    }

    output.SetCardinality(count);
}
```

## 5. 下推优化支持

### 5.1 投影下推

```cpp
// 启用投影下推
table_function.projection_pushdown = true;

// 在 init_global 中使用
static unique_ptr<GlobalTableFunctionState> MyInitGlobal(
    ClientContext &context,
    TableFunctionInitInput &input) {

    auto state = make_uniq<MyGlobalState>();

    // 只读取需要的列
    for (auto &col_id : input.column_ids) {
        state->columns_to_read.push_back(col_id);
    }

    return state;
}
```

### 5.2 过滤下推

```cpp
// 启用过滤下推
table_function.filter_pushdown = true;

// 在 init_global 中处理过滤器
static unique_ptr<GlobalTableFunctionState> MyInitGlobal(
    ClientContext &context,
    TableFunctionInitInput &input) {

    auto state = make_uniq<MyGlobalState>();

    if (input.filters) {
        for (auto &filter : input.filters->filters) {
            auto col_idx = filter.first;
            auto &table_filter = filter.second;

            // 转换过滤器
            state->AddFilter(col_idx, table_filter);
        }
    }

    return state;
}
```

### 5.3 复杂过滤下推

```cpp
table_function.pushdown_complex_filter = MyPushdownComplexFilter;

static void MyPushdownComplexFilter(
    ClientContext &context,
    LogicalGet &get,
    FunctionData *bind_data,
    vector<unique_ptr<Expression>> &filters) {

    auto &data = bind_data->Cast<MyBindData>();

    // 遍历过滤表达式
    for (auto it = filters.begin(); it != filters.end();) {
        auto &filter = *it;

        // 尝试下推
        if (CanPushdown(filter)) {
            data.pushed_filters.push_back(std::move(filter));
            it = filters.erase(it);
        } else {
            ++it;
        }
    }
}
```

### 5.4 过滤列剪枝

```cpp
// 启用过滤列剪枝
table_function.filter_prune = true;

// 效果：WHERE j = 42 后，如果 j 不在 SELECT 中，则不返回 j
// SELECT i FROM t WHERE j = 42;
// -> j 列不需要离开表函数
```

## 6. In-Out 表函数

### 6.1 概念

In-Out 表函数接收输入表并产生输出表：

```sql
SELECT * FROM my_in_out_function(input_table);
```

### 6.2 实现模式

```cpp
struct InOutBindData : public TableFunctionData {
    vector<LogicalType> input_types;
    vector<string> input_names;
};

static unique_ptr<FunctionData> InOutBind(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names) {

    auto result = make_uniq<InOutBindData>();

    // 保存输入表信息
    result->input_types = input.input_table_types;
    result->input_names = input.input_table_names;

    // 定义输出列（可以与输入不同）
    return_types = input.input_table_types;
    names = input.input_table_names;

    // 添加额外输出列
    return_types.push_back(LogicalType::BIGINT);
    names.push_back("row_number");

    return std::move(result);
}

// In-Out 主函数
static OperatorResultType InOutFunction(
    ExecutionContext &context,
    TableFunctionInput &data,
    DataChunk &input,
    DataChunk &output) {

    auto &state = data.local_state->Cast<InOutLocalState>();

    // 处理输入块
    output.SetCardinality(input.size());

    // 复制输入列到输出
    for (idx_t i = 0; i < input.ColumnCount(); i++) {
        output.data[i].Reference(input.data[i]);
    }

    // 添加行号列
    auto row_numbers = FlatVector::GetData<int64_t>(output.data[input.ColumnCount()]);
    for (idx_t i = 0; i < input.size(); i++) {
        row_numbers[i] = state.current_row++;
    }

    return OperatorResultType::NEED_MORE_INPUT;
}

// 最终函数（处理完所有输入后）
static OperatorFinalizeResultType InOutFinal(
    ExecutionContext &context,
    TableFunctionInput &data,
    DataChunk &output) {

    // 无需额外输出
    output.SetCardinality(0);
    return OperatorFinalizeResultType::FINISHED;
}

// 注册
TableFunction func("my_in_out", {LogicalType::TABLE}, nullptr, InOutBind);
func.in_out_function = InOutFunction;
func.in_out_function_final = InOutFinal;
```

## 7. 进度报告与统计

### 7.1 扫描进度

```cpp
table_function.table_scan_progress = MyProgress;

static double MyProgress(
    ClientContext &context,
    const FunctionData *bind_data,
    const GlobalTableFunctionState *global_state) {

    auto &state = global_state->Cast<MyGlobalState>();

    if (state.total_size == 0) {
        return 100.0;
    }

    return (state.processed_size * 100.0) / state.total_size;
}
```

### 7.2 基数估算

```cpp
table_function.cardinality = MyCardinality;

static unique_ptr<NodeStatistics> MyCardinality(
    ClientContext &context,
    const FunctionData *bind_data) {

    auto &data = bind_data->Cast<MyBindData>();

    // 返回估算的行数
    idx_t estimated_rows = data.file_size / data.avg_row_size;

    return make_uniq<NodeStatistics>(estimated_rows, estimated_rows);
}
```

### 7.3 列统计

```cpp
table_function.statistics = MyStatistics;

static unique_ptr<BaseStatistics> MyStatistics(
    ClientContext &context,
    const FunctionData *bind_data,
    column_t column_index) {

    auto &data = bind_data->Cast<MyBindData>();

    // 返回列的统计信息
    if (column_index < data.column_stats.size()) {
        return data.column_stats[column_index]->Copy();
    }

    return nullptr;
}
```

## 8. 并行执行

### 8.1 MaxThreads

```cpp
struct ParallelGlobalState : public GlobalTableFunctionState {
    vector<string> files;
    atomic<idx_t> next_file;

    idx_t MaxThreads() const override {
        return files.size();
    }
};
```

### 8.2 工作分配模式

```cpp
// 全局初始化：准备工作列表
static unique_ptr<GlobalTableFunctionState> ParallelInitGlobal(
    ClientContext &context,
    TableFunctionInitInput &input) {

    auto &bind_data = input.bind_data->Cast<ParallelBindData>();
    auto state = make_uniq<ParallelGlobalState>();

    state->files = bind_data.files;
    state->next_file = 0;

    return state;
}

// 局部初始化：获取工作
static unique_ptr<LocalTableFunctionState> ParallelInitLocal(
    ExecutionContext &context,
    TableFunctionInitInput &input,
    GlobalTableFunctionState *global_state) {

    auto &gstate = global_state->Cast<ParallelGlobalState>();
    auto lstate = make_uniq<ParallelLocalState>();

    // 获取下一个文件
    idx_t file_idx = gstate.next_file.fetch_add(1);
    if (file_idx < gstate.files.size()) {
        lstate->current_file = gstate.files[file_idx];
        lstate->finished = false;
    } else {
        lstate->finished = true;
    }

    return lstate;
}

// 主函数：处理当前工作
static void ParallelFunction(
    ClientContext &context,
    TableFunctionInput &data,
    DataChunk &output) {

    auto &gstate = data.global_state->Cast<ParallelGlobalState>();
    auto &lstate = data.local_state->Cast<ParallelLocalState>();

    while (true) {
        if (lstate->finished) {
            // 尝试获取新工作
            idx_t file_idx = gstate.next_file.fetch_add(1);
            if (file_idx >= gstate.files.size()) {
                output.SetCardinality(0);
                return;
            }
            lstate->current_file = gstate.files[file_idx];
            lstate->OpenFile();
        }

        // 读取数据
        auto count = lstate->ReadChunk(output);
        if (count > 0) {
            output.SetCardinality(count);
            return;
        }

        // 当前文件已完成
        lstate->finished = true;
    }
}
```

## 9. 函数注册

### 9.1 TableFunctionSet

```cpp
TableFunctionSet GetMyFunctions() {
    TableFunctionSet set("my_function");

    // 无参数版本
    set.AddFunction(TableFunction({}, MyFunction, MyBind, MyInitGlobal));

    // 带参数版本
    set.AddFunction(TableFunction(
        {LogicalType::VARCHAR},
        MyFunctionWithPath,
        MyBindWithPath,
        MyInitGlobal));

    // 带命名参数版本
    TableFunction func({LogicalType::VARCHAR}, MyFunction, MyBind);
    func.named_parameters["delimiter"] = LogicalType::VARCHAR;
    func.named_parameters["header"] = LogicalType::BOOLEAN;
    set.AddFunction(func);

    return set;
}
```

### 9.2 替换扫描

```cpp
// 替换扫描：拦截表引用并替换为表函数
static unique_ptr<TableRef> MyReplacementScan(
    ClientContext &context,
    ReplacementScanInput &input,
    optional_ptr<ReplacementScanData> data) {

    auto &table_name = input.table_name;

    // 检查是否是我们要处理的模式
    if (!StringUtil::EndsWith(table_name, ".csv")) {
        return nullptr;
    }

    // 创建表函数引用
    auto table_function = make_uniq<TableFunctionRef>();
    table_function->function = make_uniq<FunctionExpression>(
        "read_csv", vector<unique_ptr<ParsedExpression>>());

    // 添加文件路径参数
    table_function->function->children.push_back(
        make_uniq<ConstantExpression>(Value(table_name)));

    return std::move(table_function);
}

// 注册替换扫描
db.config.replacement_scans.emplace_back(MyReplacementScan);
```

## 10. 总结

### 10.1 表函数设计要点

1. **状态分离**：FunctionData (不可变) → GlobalState (共享) → LocalState (私有)
2. **并行友好**：通过 MaxThreads 和原子操作支持并行
3. **下推优化**：支持投影、过滤、采样下推
4. **进度报告**：提供扫描进度和统计信息

### 10.2 回调函数检查清单

| 回调 | 必需 | 用途 |
|------|------|------|
| bind | ✅ | 列定义、参数验证 |
| function | ✅ | 数据生成 |
| init_global | ⚪ | 全局状态初始化 |
| init_local | ⚪ | 局部状态初始化 |
| cardinality | ⚪ | 优化器行数估算 |
| statistics | ⚪ | 列统计信息 |
| table_scan_progress | ⚪ | 进度报告 |
| pushdown_complex_filter | ⚪ | 复杂过滤下推 |
| to_string | ⚪ | EXPLAIN 输出 |

### 10.3 下推标志

| 标志 | 默认 | 用途 |
|------|------|------|
| projection_pushdown | false | 列裁剪 |
| filter_pushdown | false | 过滤条件下推 |
| filter_prune | false | 过滤列剪枝 |
| sampling_pushdown | false | 采样下推 |
| late_materialization | false | 延迟物化 |

下一章将深入分析宏函数的实现，包括 SQL 表达式宏和表宏的展开机制。
