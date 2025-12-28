# 第七章：用户自定义函数（UDF）

## 7.1 UDF 概述

用户自定义函数（User-Defined Functions, UDF）允许开发者在不修改 DuckDB 核心代码的情况下扩展其功能。DuckDB 提供了多种 UDF 接口：C++ 模板接口、C API 接口，以及通过扩展机制的高级接口。

### 7.1.1 UDF 接口层次

```
┌──────────────────────────────────────────────────────────────────────┐
│                      DuckDB UDF 接口层次                              │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 高级接口（Extension/Python）                                    ││
│  │  - Extension 扩展机制                                           ││
│  │  - Python UDF（通过 duckdb-python）                             ││
│  │  - 自动类型转换和内存管理                                        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ C API 接口                                                      ││
│  │  - duckdb_create_scalar_function                                ││
│  │  - duckdb_create_aggregate_function                             ││
│  │  - 语言无关，可从任何支持 C ABI 的语言调用                        ││
│  └─────────────────────────────────────────────────────────────────┘│
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ C++ 模板接口（UDFWrapper）                                      ││
│  │  - CreateScalarFunction<TR, ARGS...>                            ││
│  │  - CreateAggregateFunction<UDF_OP, STATE, TR, TA>               ││
│  │  - 编译时类型检查，零运行时开销                                   ││
│  └─────────────────────────────────────────────────────────────────┘│
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │ 底层函数接口                                                    ││
│  │  - ScalarFunction / AggregateFunction                           ││
│  │  - 完全控制，最大灵活性                                          ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 7.1.2 支持的 UDF 类型

| UDF 类型 | 描述 | 示例 |
|----------|------|------|
| 标量 UDF | 行级计算，每行输入产生一行输出 | `my_add(a, b)` |
| 聚合 UDF | 多行聚合，整组输入产生单行输出 | `my_sum(x)` |
| 表 UDF | 数据生成器，产生表数据 | `my_range(n)` |

## 7.2 C++ 模板接口（UDFWrapper）

### 7.2.1 UDFWrapper 结构

```cpp
// src/include/duckdb/function/udf_function.hpp
struct UDFWrapper {
public:
    //! 创建标量函数（自动类型推导）
    template <typename TR, typename... ARGS>
    static scalar_function_t CreateScalarFunction(const string &name, TR (*udf_func)(ARGS...));

    //! 创建标量函数（显式类型指定）
    template <typename TR, typename... ARGS>
    static scalar_function_t CreateScalarFunction(const string &name,
                                                   const vector<LogicalType> &args,
                                                   const LogicalType &ret_type,
                                                   TR (*udf_func)(ARGS...));

    //! 注册标量函数
    template <typename TR, typename... ARGS>
    static void RegisterFunction(const string &name, scalar_function_t udf_function,
                                 ClientContext &context,
                                 LogicalType varargs = LogicalType::INVALID);

    //! 创建聚合函数
    template <typename UDF_OP, typename STATE, typename TR, typename TA>
    static AggregateFunction CreateAggregateFunction(const string &name);

    //! 通用聚合函数创建
    static AggregateFunction CreateAggregateFunction(
        const string &name,
        const vector<LogicalType> &arguments,
        const LogicalType &return_type,
        aggregate_size_t state_size,
        aggregate_initialize_t initialize,
        aggregate_update_t update,
        aggregate_combine_t combine,
        aggregate_finalize_t finalize,
        aggregate_simple_update_t simple_update = nullptr,
        bind_aggregate_function_t bind = nullptr,
        aggregate_destructor_t destructor = nullptr);

    //! 注册聚合函数
    static void RegisterAggrFunction(AggregateFunction aggr_function,
                                     ClientContext &context,
                                     LogicalType varargs = LogicalType::INVALID);
};
```

### 7.2.2 类型映射

C++ 类型与 SQL 类型的映射：

```cpp
template <typename T>
inline static LogicalType GetArgumentType() {
    if (std::is_same<T, bool>()) {
        return LogicalType(LogicalTypeId::BOOLEAN);
    } else if (std::is_same<T, int8_t>()) {
        return LogicalType(LogicalTypeId::TINYINT);
    } else if (std::is_same<T, int16_t>()) {
        return LogicalType(LogicalTypeId::SMALLINT);
    } else if (std::is_same<T, int32_t>()) {
        return LogicalType(LogicalTypeId::INTEGER);
    } else if (std::is_same<T, int64_t>()) {
        return LogicalType(LogicalTypeId::BIGINT);
    } else if (std::is_same<T, float>()) {
        return LogicalType(LogicalTypeId::FLOAT);
    } else if (std::is_same<T, double>()) {
        return LogicalType(LogicalTypeId::DOUBLE);
    } else if (std::is_same<T, string_t>()) {
        return LogicalType(LogicalTypeId::VARCHAR);
    } else {
        throw std::runtime_error("Unrecognized type!");
    }
}
```

类型映射表：

```
┌──────────────────────────────────────────────────────────────────────┐
│                      C++ 类型 ↔ SQL 类型映射                          │
├────────────────────────┬─────────────────────────────────────────────┤
│ C++ 类型               │ SQL 类型                                    │
├────────────────────────┼─────────────────────────────────────────────┤
│ bool                   │ BOOLEAN                                     │
│ int8_t                 │ TINYINT                                     │
│ int16_t                │ SMALLINT                                    │
│ int32_t                │ INTEGER                                     │
│ int64_t                │ BIGINT                                      │
│ float                  │ FLOAT                                       │
│ double                 │ DOUBLE                                      │
│ string_t               │ VARCHAR / CHAR / BLOB                       │
│ date_t                 │ DATE                                        │
│ dtime_t                │ TIME                                        │
│ timestamp_t            │ TIMESTAMP / TIMESTAMP_MS / TIMESTAMP_NS     │
└────────────────────────┴─────────────────────────────────────────────┘
```

### 7.2.3 标量 UDF 示例

**一元函数：**

```cpp
// 定义 C++ 函数
int64_t my_double(int64_t x) {
    return x * 2;
}

// 创建并注册
auto udf = UDFWrapper::CreateScalarFunction<int64_t, int64_t>("my_double", my_double);
UDFWrapper::RegisterFunction<int64_t, int64_t>("my_double", udf, context);
```

**二元函数：**

```cpp
// 定义加法函数
double my_add(double a, double b) {
    return a + b;
}

// 创建函数
auto udf = UDFWrapper::CreateScalarFunction<double, double, double>(
    "my_add", my_add);

// 显式指定类型
vector<LogicalType> args = {LogicalType::DOUBLE, LogicalType::DOUBLE};
auto udf_explicit = UDFWrapper::CreateScalarFunction<double, double, double>(
    "my_add", args, LogicalType::DOUBLE, my_add);
```

**三元函数：**

```cpp
// 定义条件选择函数
int64_t my_if(bool cond, int64_t then_val, int64_t else_val) {
    return cond ? then_val : else_val;
}

auto udf = UDFWrapper::CreateScalarFunction<int64_t, bool, int64_t, int64_t>(
    "my_if", my_if);
```

### 7.2.4 执行器适配

UDFWrapper 内部使用向量化执行器适配用户函数：

```cpp
// 一元函数执行器适配
struct UnaryUDFExecutor {
    template <class INPUT_TYPE, class RESULT_TYPE>
    static RESULT_TYPE Operation(INPUT_TYPE input, ValidityMask &mask,
                                  idx_t idx, void *dataptr) {
        // 将 void* 转换回用户函数指针
        typedef RESULT_TYPE (*unary_function_t)(INPUT_TYPE);
        auto udf = (unary_function_t)dataptr;
        return udf(input);
    }
};

// 创建一元函数
template <typename TR, typename TA>
inline static scalar_function_t CreateUnaryFunction(const string &name, TR (*udf_func)(TA)) {
    scalar_function_t udf_function = [=](DataChunk &input, ExpressionState &state,
                                          Vector &result) -> void {
        UnaryExecutor::GenericExecute<TA, TR, UnaryUDFExecutor>(
            input.data[0], result, input.size(), (void *)udf_func);
    };
    return udf_function;
}

// 创建二元函数
template <typename TR, typename TA, typename TB>
inline static scalar_function_t CreateBinaryFunction(const string &name, TR (*udf_func)(TA, TB)) {
    scalar_function_t udf_function = [=](DataChunk &input, ExpressionState &state,
                                          Vector &result) -> void {
        BinaryExecutor::Execute<TA, TB, TR>(
            input.data[0], input.data[1], result, input.size(), udf_func);
    };
    return udf_function;
}
```

## 7.3 聚合 UDF 模板接口

### 7.3.1 聚合状态定义

聚合 UDF 需要定义状态结构和操作：

```cpp
// 定义聚合状态
struct MySumState {
    double sum;
    bool is_null;
};

// 定义聚合操作
struct MySumOperation {
    // 初始化状态
    template <class STATE>
    static void Initialize(STATE &state) {
        state.sum = 0;
        state.is_null = true;
    }

    // 更新状态（处理单个输入值）
    template <class INPUT_TYPE, class STATE, class OP>
    static void Operation(STATE &state, const INPUT_TYPE &input,
                          AggregateUnaryInput &unary_input) {
        state.sum += input;
        state.is_null = false;
    }

    // 合并两个状态（并行聚合）
    template <class STATE, class OP>
    static void Combine(const STATE &source, STATE &target,
                        AggregateInputData &aggr_input_data) {
        target.sum += source.sum;
        if (!source.is_null) {
            target.is_null = false;
        }
    }

    // 最终化（生成结果）
    template <class T, class STATE>
    static void Finalize(STATE &state, T &target, AggregateFinalizeData &finalize_data) {
        if (state.is_null) {
            finalize_data.ReturnNull();
        } else {
            target = state.sum;
        }
    }

    // 简单更新优化（可选）
    template <class INPUT_TYPE, class STATE, class OP>
    static void SimpleUpdate(STATE &state, const INPUT_TYPE &input,
                             AggregateInputData &aggr_input_data) {
        Operation<INPUT_TYPE, STATE, OP>(state, input, aggr_input_data);
    }

    // 是否需要析构
    static bool IgnoreNull() {
        return true;
    }
};
```

### 7.3.2 创建聚合 UDF

```cpp
// 使用模板创建一元聚合函数
auto my_sum = UDFWrapper::CreateAggregateFunction<MySumOperation, MySumState, double, double>(
    "my_sum");

// 使用模板创建二元聚合函数
auto my_covar = UDFWrapper::CreateAggregateFunction<
    MyCovarOperation, MyCovarState, double, double, double>("my_covar");

// 注册聚合函数
UDFWrapper::RegisterAggrFunction(my_sum, context);
```

### 7.3.3 通用聚合函数创建

对于需要更多控制的场景，可以使用通用接口：

```cpp
// 定义各个回调函数
idx_t my_state_size() {
    return sizeof(MySumState);
}

void my_initialize(data_ptr_t state) {
    auto &s = *reinterpret_cast<MySumState *>(state);
    s.sum = 0;
    s.is_null = true;
}

void my_update(Vector inputs[], AggregateInputData &aggr_input_data,
               idx_t input_count, Vector &state, idx_t count) {
    // 更新逻辑
}

void my_combine(Vector &state, Vector &combined,
                AggregateInputData &aggr_input_data, idx_t count) {
    // 合并逻辑
}

void my_finalize(Vector &state, AggregateInputData &aggr_input_data,
                 Vector &result, idx_t count, idx_t offset) {
    // 最终化逻辑
}

// 创建聚合函数
auto my_agg = UDFWrapper::CreateAggregateFunction(
    "my_agg",
    {LogicalType::DOUBLE},       // 参数类型
    LogicalType::DOUBLE,          // 返回类型
    my_state_size,
    my_initialize,
    my_update,
    my_combine,
    my_finalize,
    nullptr,                      // simple_update
    nullptr,                      // bind
    nullptr                       // destructor
);
```

## 7.4 C API 接口

### 7.4.1 标量函数 C API

C API 提供了语言无关的 UDF 接口：

```c
// 创建标量函数
duckdb_scalar_function duckdb_create_scalar_function();

// 设置函数属性
void duckdb_scalar_function_set_name(duckdb_scalar_function function, const char *name);
void duckdb_scalar_function_add_parameter(duckdb_scalar_function function, duckdb_logical_type type);
void duckdb_scalar_function_set_return_type(duckdb_scalar_function function, duckdb_logical_type type);

// 设置函数实现
void duckdb_scalar_function_set_function(duckdb_scalar_function function,
                                          duckdb_scalar_function_t execute_func);
void duckdb_scalar_function_set_bind(duckdb_scalar_function function,
                                      duckdb_scalar_function_bind_t bind);

// 设置函数属性
void duckdb_scalar_function_set_varargs(duckdb_scalar_function function, duckdb_logical_type type);
void duckdb_scalar_function_set_special_handling(duckdb_scalar_function function);
void duckdb_scalar_function_set_volatile(duckdb_scalar_function function);

// 设置额外信息
void duckdb_scalar_function_set_extra_info(duckdb_scalar_function function,
                                            void *extra_info,
                                            duckdb_delete_callback_t destroy);

// 注册函数
duckdb_state duckdb_register_scalar_function(duckdb_connection connection,
                                              duckdb_scalar_function function);

// 清理
void duckdb_destroy_scalar_function(duckdb_scalar_function *function);
```

### 7.4.2 C API 使用示例

```c
// 定义执行函数
void my_add_execute(duckdb_function_info info, duckdb_data_chunk input,
                    duckdb_vector output) {
    // 获取输入数据
    idx_t count = duckdb_data_chunk_get_size(input);
    duckdb_vector vec_a = duckdb_data_chunk_get_vector(input, 0);
    duckdb_vector vec_b = duckdb_data_chunk_get_vector(input, 1);

    // 获取数据指针
    int64_t *data_a = (int64_t *)duckdb_vector_get_data(vec_a);
    int64_t *data_b = (int64_t *)duckdb_vector_get_data(vec_b);
    int64_t *result = (int64_t *)duckdb_vector_get_data(output);

    // 执行计算
    for (idx_t i = 0; i < count; i++) {
        result[i] = data_a[i] + data_b[i];
    }
}

// 注册函数
void register_my_add(duckdb_connection conn) {
    // 创建函数
    duckdb_scalar_function func = duckdb_create_scalar_function();

    // 设置名称
    duckdb_scalar_function_set_name(func, "my_add");

    // 设置参数类型
    duckdb_logical_type bigint_type = duckdb_create_logical_type(DUCKDB_TYPE_BIGINT);
    duckdb_scalar_function_add_parameter(func, bigint_type);
    duckdb_scalar_function_add_parameter(func, bigint_type);

    // 设置返回类型
    duckdb_scalar_function_set_return_type(func, bigint_type);

    // 设置执行函数
    duckdb_scalar_function_set_function(func, my_add_execute);

    // 注册
    duckdb_register_scalar_function(conn, func);

    // 清理
    duckdb_destroy_logical_type(&bigint_type);
    duckdb_destroy_scalar_function(&func);
}
```

### 7.4.3 聚合函数 C API

```c
// 创建聚合函数
duckdb_aggregate_function duckdb_create_aggregate_function();

// 设置函数属性
void duckdb_aggregate_function_set_name(duckdb_aggregate_function function, const char *name);
void duckdb_aggregate_function_add_parameter(duckdb_aggregate_function function,
                                              duckdb_logical_type type);
void duckdb_aggregate_function_set_return_type(duckdb_aggregate_function function,
                                                duckdb_logical_type type);

// 设置聚合函数回调
void duckdb_aggregate_function_set_functions(
    duckdb_aggregate_function function,
    duckdb_aggregate_state_size state_size,
    duckdb_aggregate_init_t state_init,
    duckdb_aggregate_update_t update,
    duckdb_aggregate_combine_t combine,
    duckdb_aggregate_finalize_t finalize);

void duckdb_aggregate_function_set_destructor(duckdb_aggregate_function function,
                                               duckdb_aggregate_destroy_t destroy);

// 设置额外信息
void duckdb_aggregate_function_set_extra_info(duckdb_aggregate_function function,
                                               void *extra_info,
                                               duckdb_delete_callback_t destroy);

// 注册
duckdb_state duckdb_register_aggregate_function(duckdb_connection connection,
                                                 duckdb_aggregate_function function);

// 清理
void duckdb_destroy_aggregate_function(duckdb_aggregate_function *function);
```

### 7.4.4 C API 内部实现

C API 内部使用包装结构来桥接 C 接口和 C++ 实现：

```cpp
// 标量函数信息结构
struct CScalarFunctionInfo : public ScalarFunctionInfo {
    ~CScalarFunctionInfo() override {
        if (extra_info && delete_callback) {
            delete_callback(extra_info);
        }
    }

    duckdb_scalar_function_bind_t bind = nullptr;
    duckdb_scalar_function_t function = nullptr;
    duckdb_function_info extra_info = nullptr;
    duckdb_delete_callback_t delete_callback = nullptr;
};

// 标量函数执行包装
void CAPIScalarFunction(DataChunk &input, ExpressionState &state, Vector &result) {
    auto &function = state.expr.Cast<BoundFunctionExpression>();
    auto &bind_info = function.bind_info;
    auto &c_bind_info = bind_info->Cast<CScalarFunctionBindData>();

    // 展平输入
    auto all_const = input.AllConstant();
    input.Flatten();

    // 转换为 C 接口类型
    auto c_input = reinterpret_cast<duckdb_data_chunk>(&input);
    auto c_result = reinterpret_cast<duckdb_vector>(&result);

    // 调用 C 函数
    CScalarFunctionInternalFunctionInfo function_info(c_bind_info);
    c_bind_info.info.function(ToCScalarFunctionInfo(function_info), c_input, c_result);

    // 错误处理
    if (!function_info.success) {
        throw InvalidInputException(function_info.error);
    }

    // 常量优化
    if (all_const && (input.size() == 1 ||
        function.function.GetStability() != FunctionStability::VOLATILE)) {
        result.SetVectorType(VectorType::CONSTANT_VECTOR);
    }
}
```

## 7.5 函数重载集

### 7.5.1 创建重载集

```c
// 创建标量函数重载集
duckdb_scalar_function_set duckdb_create_scalar_function_set(const char *name);

// 添加函数到集合
duckdb_state duckdb_add_scalar_function_to_set(duckdb_scalar_function_set set,
                                                duckdb_scalar_function function);

// 注册整个集合
duckdb_state duckdb_register_scalar_function_set(duckdb_connection connection,
                                                  duckdb_scalar_function_set set);

// 清理
void duckdb_destroy_scalar_function_set(duckdb_scalar_function_set *set);
```

### 7.5.2 重载示例

```c
void register_my_add_overloaded(duckdb_connection conn) {
    // 创建重载集
    duckdb_scalar_function_set set = duckdb_create_scalar_function_set("my_add");

    // 添加 INTEGER 版本
    duckdb_scalar_function int_func = duckdb_create_scalar_function();
    duckdb_logical_type int_type = duckdb_create_logical_type(DUCKDB_TYPE_INTEGER);
    duckdb_scalar_function_add_parameter(int_func, int_type);
    duckdb_scalar_function_add_parameter(int_func, int_type);
    duckdb_scalar_function_set_return_type(int_func, int_type);
    duckdb_scalar_function_set_function(int_func, my_add_int_execute);
    duckdb_add_scalar_function_to_set(set, int_func);

    // 添加 BIGINT 版本
    duckdb_scalar_function bigint_func = duckdb_create_scalar_function();
    duckdb_logical_type bigint_type = duckdb_create_logical_type(DUCKDB_TYPE_BIGINT);
    duckdb_scalar_function_add_parameter(bigint_func, bigint_type);
    duckdb_scalar_function_add_parameter(bigint_func, bigint_type);
    duckdb_scalar_function_set_return_type(bigint_func, bigint_type);
    duckdb_scalar_function_set_function(bigint_func, my_add_bigint_execute);
    duckdb_add_scalar_function_to_set(set, bigint_func);

    // 添加 DOUBLE 版本
    duckdb_scalar_function double_func = duckdb_create_scalar_function();
    duckdb_logical_type double_type = duckdb_create_logical_type(DUCKDB_TYPE_DOUBLE);
    duckdb_scalar_function_add_parameter(double_func, double_type);
    duckdb_scalar_function_add_parameter(double_func, double_type);
    duckdb_scalar_function_set_return_type(double_func, double_type);
    duckdb_scalar_function_set_function(double_func, my_add_double_execute);
    duckdb_add_scalar_function_to_set(set, double_func);

    // 注册重载集
    duckdb_register_scalar_function_set(conn, set);

    // 清理
    duckdb_destroy_logical_type(&int_type);
    duckdb_destroy_logical_type(&bigint_type);
    duckdb_destroy_logical_type(&double_type);
    duckdb_destroy_scalar_function(&int_func);
    duckdb_destroy_scalar_function(&bigint_func);
    duckdb_destroy_scalar_function(&double_func);
    duckdb_destroy_scalar_function_set(&set);
}
```

## 7.6 绑定回调

### 7.6.1 绑定回调用途

绑定回调允许在函数绑定时执行自定义逻辑：

- 验证参数
- 推导返回类型
- 创建绑定数据
- 修改参数表达式

### 7.6.2 C++ 绑定回调

```cpp
unique_ptr<FunctionData> MyFunctionBind(ClientContext &context,
                                         ScalarFunction &bound_function,
                                         vector<unique_ptr<Expression>> &arguments) {
    // 验证参数数量
    if (arguments.size() < 1) {
        throw BinderException("my_function requires at least one argument");
    }

    // 根据参数类型推导返回类型
    auto &arg_type = arguments[0]->return_type;
    if (arg_type.id() == LogicalTypeId::INTEGER) {
        bound_function.SetReturnType(LogicalType::INTEGER);
    } else {
        bound_function.SetReturnType(LogicalType::DOUBLE);
    }

    // 创建绑定数据
    auto bind_data = make_uniq<MyFunctionBindData>();
    bind_data->some_setting = ExtractSetting(arguments);
    return std::move(bind_data);
}

// 设置绑定回调
ScalarFunction my_func("my_func", {LogicalType::ANY}, LogicalType::ANY, MyFunctionExecute);
my_func.bind = MyFunctionBind;
```

### 7.6.3 C API 绑定回调

```c
// 绑定回调函数签名
typedef void (*duckdb_scalar_function_bind_t)(duckdb_bind_info info);

// 绑定回调实现
void my_function_bind(duckdb_bind_info info) {
    // 获取参数数量
    idx_t arg_count = duckdb_scalar_function_bind_get_argument_count(info);

    // 获取参数表达式
    duckdb_expression arg = duckdb_scalar_function_bind_get_argument(info, 0);

    // 设置绑定数据
    MyBindData *data = malloc(sizeof(MyBindData));
    data->some_value = 42;
    duckdb_scalar_function_set_bind_data(info, data, free);

    // 如果需要复制绑定数据
    duckdb_scalar_function_set_bind_data_copy(info, my_bind_data_copy);

    // 错误处理
    if (arg_count < 1) {
        duckdb_scalar_function_bind_set_error(info, "Need at least one argument");
    }

    // 清理
    duckdb_destroy_expression(&arg);
}

// 设置绑定回调
duckdb_scalar_function_set_bind(func, my_function_bind);
```

## 7.7 错误处理

### 7.7.1 C++ 错误处理

```cpp
void MyFunctionExecute(DataChunk &input, ExpressionState &state, Vector &result) {
    try {
        // 执行逻辑
        for (idx_t i = 0; i < input.size(); i++) {
            if (!ValidateInput(input, i)) {
                throw InvalidInputException("Invalid input at row %d", i);
            }
            // 计算结果
        }
    } catch (const std::exception &e) {
        throw InvalidInputException("Error in my_function: %s", e.what());
    }
}
```

### 7.7.2 C API 错误处理

```c
void my_function_execute(duckdb_function_info info, duckdb_data_chunk input,
                          duckdb_vector output) {
    idx_t count = duckdb_data_chunk_get_size(input);
    duckdb_vector vec = duckdb_data_chunk_get_vector(input, 0);
    int64_t *data = (int64_t *)duckdb_vector_get_data(vec);

    for (idx_t i = 0; i < count; i++) {
        if (data[i] < 0) {
            // 设置错误并返回
            duckdb_scalar_function_set_error(info, "Input must be non-negative");
            return;
        }
    }

    // 继续正常执行...
}
```

## 7.8 UDF 注册流程

### 7.8.1 注册流程图解

```
┌──────────────────────────────────────────────────────────────────────┐
│                      UDF 注册流程                                     │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 创建函数对象                                                      │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ duckdb_create_scalar_function()                            │  │
│     │ → ScalarFunction 对象                                      │  │
│     └────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              ▼                                       │
│  2. 设置函数属性                                                      │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ - 名称：duckdb_scalar_function_set_name                     │  │
│     │ - 参数：duckdb_scalar_function_add_parameter               │  │
│     │ - 返回类型：duckdb_scalar_function_set_return_type         │  │
│     │ - 执行函数：duckdb_scalar_function_set_function            │  │
│     └────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              ▼                                       │
│  3. 验证函数配置                                                      │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ - 检查名称非空                                              │  │
│     │ - 检查执行函数已设置                                         │  │
│     │ - 检查返回类型有效                                           │  │
│     │ - 检查参数类型有效                                           │  │
│     └────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              ▼                                       │
│  4. 创建函数集合                                                      │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ ScalarFunctionSet set(function.name)                       │  │
│     │ set.AddFunction(function)                                  │  │
│     └────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              ▼                                       │
│  5. 注册到 Catalog                                                   │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ CreateScalarFunctionInfo info(set)                         │  │
│     │ info.on_conflict = ALTER_ON_CONFLICT                       │  │
│     │ catalog.CreateFunction(context, info)                      │  │
│     └────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│                              ▼                                       │
│  6. 函数可用                                                          │
│     ┌────────────────────────────────────────────────────────────┐  │
│     │ SELECT my_func(column) FROM table                          │  │
│     └────────────────────────────────────────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 7.8.2 Catalog 注册实现

```cpp
duckdb_state duckdb_register_scalar_function_set(duckdb_connection connection,
                                                  duckdb_scalar_function_set set) {
    if (!connection || !set) {
        return DuckDBError;
    }

    auto &scalar_function_set = GetCScalarFunctionSet(set);

    // 验证所有函数
    for (idx_t idx = 0; idx < scalar_function_set.Size(); idx++) {
        auto &scalar_function = scalar_function_set.GetFunctionReferenceByOffset(idx);
        auto &info = scalar_function.GetExtraFunctionInfo().Cast<CScalarFunctionInfo>();

        // 检查必要属性
        if (scalar_function.name.empty() || !info.function) {
            return DuckDBError;
        }

        // 检查返回类型
        if (TypeVisitor::Contains(scalar_function.GetReturnType(), LogicalTypeId::INVALID) ||
            TypeVisitor::Contains(scalar_function.GetReturnType(), LogicalTypeId::ANY)) {
            return DuckDBError;
        }

        // 检查参数类型
        for (const auto &argument : scalar_function.arguments) {
            if (TypeVisitor::Contains(argument, LogicalTypeId::INVALID)) {
                return DuckDBError;
            }
        }
    }

    try {
        auto con = reinterpret_cast<Connection *>(connection);
        con->context->RunFunctionInTransaction([&]() {
            auto &catalog = Catalog::GetSystemCatalog(*con->context);
            CreateScalarFunctionInfo sf_info(scalar_function_set);
            sf_info.on_conflict = OnCreateConflict::ALTER_ON_CONFLICT;
            catalog.CreateFunction(*con->context, sf_info);
        });
    } catch (...) {
        return DuckDBError;
    }

    return DuckDBSuccess;
}
```

## 7.9 高级 UDF 模式

### 7.9.1 变参函数

```c
// 创建变参函数
duckdb_scalar_function func = duckdb_create_scalar_function();
duckdb_scalar_function_set_name(func, "my_concat");

// 设置变参类型
duckdb_logical_type varchar_type = duckdb_create_logical_type(DUCKDB_TYPE_VARCHAR);
duckdb_scalar_function_set_varargs(func, varchar_type);
duckdb_scalar_function_set_return_type(func, varchar_type);

// 实现可处理任意数量的字符串参数
void my_concat_execute(duckdb_function_info info, duckdb_data_chunk input,
                       duckdb_vector output) {
    idx_t count = duckdb_data_chunk_get_size(input);
    idx_t col_count = duckdb_data_chunk_get_column_count(input);

    // 遍历所有列进行连接
    for (idx_t i = 0; i < count; i++) {
        string result;
        for (idx_t c = 0; c < col_count; c++) {
            // 连接每列的值
        }
        // 设置结果
    }
}
```

### 7.9.2 NULL 特殊处理

```c
// 设置 NULL 特殊处理
duckdb_scalar_function_set_special_handling(func);

void my_coalesce_execute(duckdb_function_info info, duckdb_data_chunk input,
                          duckdb_vector output) {
    // 需要自己处理 NULL 值
    idx_t count = duckdb_data_chunk_get_size(input);

    for (idx_t i = 0; i < count; i++) {
        // 遍历参数找到第一个非 NULL
        for (idx_t c = 0; c < col_count; c++) {
            if (!is_null(input, c, i)) {
                set_result(output, i, get_value(input, c, i));
                break;
            }
        }
    }
}
```

### 7.9.3 Volatile 函数

```c
// 标记为 volatile（每次调用可能返回不同结果）
duckdb_scalar_function_set_volatile(func);

void my_random_execute(duckdb_function_info info, duckdb_data_chunk input,
                        duckdb_vector output) {
    idx_t count = duckdb_data_chunk_get_size(input);
    double *result = (double *)duckdb_vector_get_data(output);

    for (idx_t i = 0; i < count; i++) {
        result[i] = rand() / (double)RAND_MAX;
    }
}
```

## 7.10 本章小结

本章详细介绍了 DuckDB 的用户自定义函数系统：

1. **多层接口**：
   - C++ 模板接口提供类型安全和编译时检查
   - C API 提供语言无关的扩展能力
   - 高级接口（Extension/Python）提供便捷的开发体验

2. **UDFWrapper 模板框架**：
   - 自动类型映射：C++ 类型 ↔ SQL 类型
   - 支持一元、二元、三元标量函数
   - 支持聚合函数的完整生命周期

3. **C API 设计**：
   - 完整的函数创建和配置接口
   - 支持函数重载集
   - 提供绑定回调和错误处理机制

4. **聚合 UDF**：
   - 状态结构定义
   - 操作接口：Initialize / Update / Combine / Finalize
   - 支持析构器和简单更新优化

5. **高级特性**：
   - 变参函数支持
   - NULL 特殊处理
   - Volatile 函数标记

UDF 系统是 DuckDB 可扩展性的基础，允许用户在不修改核心代码的情况下添加自定义功能。
