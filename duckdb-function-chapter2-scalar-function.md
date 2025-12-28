# DuckDB 函数系统深度解析（二）：标量函数实现机制

## 引言

标量函数（Scalar Function）是 DuckDB 中最常用的函数类型，对输入的每一行计算并返回一个值。本章深入分析标量函数的核心结构、向量化执行器模板，以及典型函数的实现细节。

## 1. ScalarFunction 核心结构

### 1.1 类定义

```
源文件: src/include/duckdb/function/scalar_function.hpp
```

```cpp
class ScalarFunction : public BaseScalarFunction {
public:
    ScalarFunction(string name,
                   vector<LogicalType> arguments,
                   LogicalType return_type,
                   scalar_function_t function,
                   bind_scalar_function_t bind = nullptr,
                   bind_scalar_function_extended_t bind_extended = nullptr,
                   function_statistics_t statistics = nullptr,
                   init_local_state_t init_local_state = nullptr,
                   LogicalType varargs = LogicalType(LogicalTypeId::INVALID),
                   FunctionStability stability = FunctionStability::CONSISTENT,
                   FunctionNullHandling null_handling = FunctionNullHandling::DEFAULT_NULL_HANDLING,
                   bind_lambda_function_t bind_lambda = nullptr);

public:
    //! 主执行函数
    scalar_function_t function;
    //! 绑定回调
    bind_scalar_function_t bind;
    //! 扩展绑定回调
    bind_scalar_function_extended_t bind_extended;
    //! 线程局部状态初始化
    init_local_state_t init_local_state;
    //! 统计传播
    function_statistics_t statistics;
    //! Lambda 绑定
    bind_lambda_function_t bind_lambda;
    //! 表达式绑定
    function_bind_expression_t bind_expression;
    //! 修改的数据库
    get_modified_databases_t get_modified_databases;
    //! 序列化/反序列化
    function_serialize_t serialize;
    function_deserialize_t deserialize;
    //! 额外函数信息
    shared_ptr<ScalarFunctionInfo> function_info;
};
```

### 1.2 回调函数类型

```cpp
//! 主执行函数：接收输入 DataChunk，输出到 Vector
typedef std::function<void(DataChunk &, ExpressionState &, Vector &)> scalar_function_t;

//! 绑定函数：类型推导和 FunctionData 创建
typedef unique_ptr<FunctionData> (*bind_scalar_function_t)(
    ClientContext &context,
    ScalarFunction &bound_function,
    vector<unique_ptr<Expression>> &arguments);

//! 线程局部状态初始化
typedef unique_ptr<FunctionLocalState> (*init_local_state_t)(
    ExpressionState &state,
    const BoundFunctionExpression &expr,
    FunctionData *bind_data);

//! 统计信息传播
typedef unique_ptr<BaseStatistics> (*function_statistics_t)(
    ClientContext &context,
    FunctionStatisticsInput &input);

//! Lambda 参数类型绑定
typedef LogicalType (*bind_lambda_function_t)(
    ClientContext &context,
    const vector<LogicalType> &function_child_types,
    idx_t parameter_idx);
```

### 1.3 函数创建示例

```cpp
// 创建 upper 函数
ScalarFunction upper_func(
    "upper",                              // 函数名
    {LogicalType::VARCHAR},               // 参数类型
    LogicalType::VARCHAR,                 // 返回类型
    UpperFunction,                        // 执行函数
    nullptr,                              // 无需绑定
    nullptr,                              // 无扩展绑定
    nullptr,                              // 无统计传播
    nullptr,                              // 无局部状态
    LogicalType::INVALID,                 // 无变参
    FunctionStability::CONSISTENT,        // 稳定函数
    FunctionNullHandling::DEFAULT_NULL_HANDLING  // 默认 NULL 处理
);
```

## 2. 向量化执行器模板

### 2.1 执行器层次

DuckDB 提供一系列模板化执行器，简化向量化函数实现：

```
┌─────────────────────────────────────────────────────────────┐
│                    Executor Templates                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
│  │UnaryExecutor  │  │BinaryExecutor │  │TernaryExecutor│   │
│  │               │  │               │  │               │   │
│  │ f(a) → r      │  │ f(a,b) → r    │  │ f(a,b,c) → r  │   │
│  └───────────────┘  └───────────────┘  └───────────────┘   │
│                                                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │              GenericExecutor (N 参数)                  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 UnaryExecutor：一元函数

```cpp
// src/include/duckdb/common/vector_operations/unary_executor.hpp

class UnaryExecutor {
public:
    // 标准执行：OP::Operation(input) → result
    template <class INPUT_TYPE, class RESULT_TYPE, class OP>
    static void Execute(Vector &input, Vector &result, idx_t count) {
        // 获取输入数据指针
        UnifiedVectorFormat input_data;
        input.ToUnifiedFormat(count, input_data);
        auto input_ptr = UnifiedVectorFormat::GetData<INPUT_TYPE>(input_data);

        // 获取输出数据指针
        auto result_ptr = FlatVector::GetData<RESULT_TYPE>(result);
        auto &result_validity = FlatVector::Validity(result);

        // 处理每个元素
        if (input_data.validity.AllValid()) {
            // 无 NULL，快速路径
            for (idx_t i = 0; i < count; i++) {
                auto idx = input_data.sel->get_index(i);
                result_ptr[i] = OP::Operation(input_ptr[idx]);
            }
        } else {
            // 有 NULL，需要检查有效性
            for (idx_t i = 0; i < count; i++) {
                auto idx = input_data.sel->get_index(i);
                if (input_data.validity.RowIsValid(idx)) {
                    result_ptr[i] = OP::Operation(input_ptr[idx]);
                } else {
                    result_validity.SetInvalid(i);
                }
            }
        }
    }

    // 带额外数据的执行
    template <class INPUT_TYPE, class RESULT_TYPE, class OP>
    static void GenericExecute(Vector &input, Vector &result,
                               idx_t count, void *dataptr) {
        // 类似上面，但传递 dataptr 给 Operation
        // OP::Operation(input, mask, idx, dataptr)
    }
};
```

### 2.3 BinaryExecutor：二元函数

```cpp
// src/include/duckdb/common/vector_operations/binary_executor.hpp

class BinaryExecutor {
public:
    // 标准执行
    template <class LEFT_TYPE, class RIGHT_TYPE, class RESULT_TYPE, class OP>
    static void ExecuteStandard(Vector &left, Vector &right,
                                Vector &result, idx_t count) {
        UnifiedVectorFormat left_data, right_data;
        left.ToUnifiedFormat(count, left_data);
        right.ToUnifiedFormat(count, right_data);

        auto ldata = UnifiedVectorFormat::GetData<LEFT_TYPE>(left_data);
        auto rdata = UnifiedVectorFormat::GetData<RIGHT_TYPE>(right_data);
        auto result_data = FlatVector::GetData<RESULT_TYPE>(result);

        // 处理四种情况：两边都有效、左边有 NULL、右边有 NULL、两边都有 NULL
        if (left_data.validity.AllValid() && right_data.validity.AllValid()) {
            // 快速路径：无 NULL
            for (idx_t i = 0; i < count; i++) {
                auto lidx = left_data.sel->get_index(i);
                auto ridx = right_data.sel->get_index(i);
                result_data[i] = OP::Operation(ldata[lidx], rdata[ridx]);
            }
        } else {
            // 慢速路径：检查 NULL
            auto &result_validity = FlatVector::Validity(result);
            for (idx_t i = 0; i < count; i++) {
                auto lidx = left_data.sel->get_index(i);
                auto ridx = right_data.sel->get_index(i);
                if (left_data.validity.RowIsValid(lidx) &&
                    right_data.validity.RowIsValid(ridx)) {
                    result_data[i] = OP::Operation(ldata[lidx], rdata[ridx]);
                } else {
                    result_validity.SetInvalid(i);
                }
            }
        }
    }

    // 带函数指针的执行
    template <class LEFT_TYPE, class RIGHT_TYPE, class RESULT_TYPE>
    static void Execute(Vector &left, Vector &right, Vector &result,
                        idx_t count,
                        RESULT_TYPE (*func)(LEFT_TYPE, RIGHT_TYPE)) {
        // 使用函数指针而非模板参数
    }
};
```

### 2.4 TernaryExecutor：三元函数

```cpp
class TernaryExecutor {
public:
    template <class A_TYPE, class B_TYPE, class C_TYPE, class RESULT_TYPE, class OP>
    static void ExecuteStandard(Vector &a, Vector &b, Vector &c,
                                Vector &result, idx_t count) {
        // 类似 BinaryExecutor，但处理三个输入
        UnifiedVectorFormat a_data, b_data, c_data;
        a.ToUnifiedFormat(count, a_data);
        b.ToUnifiedFormat(count, b_data);
        c.ToUnifiedFormat(count, c_data);

        // ... 执行逻辑
    }
};
```

### 2.5 ScalarFunction 静态辅助方法

```cpp
class ScalarFunction {
public:
    // NOP 函数：直接复制输入到输出
    static void NopFunction(DataChunk &input, ExpressionState &state, Vector &result);

    // 一元函数包装
    template <class TA, class TR, class OP>
    static void UnaryFunction(DataChunk &input, ExpressionState &state, Vector &result) {
        D_ASSERT(input.ColumnCount() >= 1);
        UnaryExecutor::Execute<TA, TR, OP>(input.data[0], result, input.size());
    }

    // 二元函数包装
    template <class TA, class TB, class TR, class OP>
    static void BinaryFunction(DataChunk &input, ExpressionState &state, Vector &result) {
        D_ASSERT(input.ColumnCount() == 2);
        BinaryExecutor::ExecuteStandard<TA, TB, TR, OP>(
            input.data[0], input.data[1], result, input.size());
    }

    // 三元函数包装
    template <class TA, class TB, class TC, class TR, class OP>
    static void TernaryFunction(DataChunk &input, ExpressionState &state, Vector &result) {
        D_ASSERT(input.ColumnCount() == 3);
        TernaryExecutor::ExecuteStandard<TA, TB, TC, TR, OP>(
            input.data[0], input.data[1], input.data[2], result, input.size());
    }
};
```

## 3. 典型标量函数实现

### 3.1 字符串大小写转换

```
源文件: src/function/scalar/string/caseconvert.cpp
```

```cpp
// 操作结构体
struct UpperFun {
    // 获取函数实现
    static ScalarFunction GetFunction() {
        return ScalarFunction(
            {LogicalType::VARCHAR},
            LogicalType::VARCHAR,
            ScalarStringFunction<ASCIIUpperCase, ICUUpperCase>);
    }
};

// ASCII 快速路径
struct ASCIIUpperCase {
    static void Operation(const char *input, idx_t len, char *output) {
        for (idx_t i = 0; i < len; i++) {
            char c = input[i];
            if (c >= 'a' && c <= 'z') {
                output[i] = c - 32;  // 转大写
            } else {
                output[i] = c;
            }
        }
    }

    static bool RequiresICU(const char *input, idx_t len) {
        // 检查是否包含非 ASCII 字符
        for (idx_t i = 0; i < len; i++) {
            if (input[i] & 0x80) {
                return true;  // 需要 ICU 处理 UTF-8
            }
        }
        return false;
    }
};

// 通用字符串函数模板
template <class ASCII_OP, class ICU_OP>
static void ScalarStringFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    UnaryExecutor::ExecuteString<string_t, string_t>(
        args.data[0], result, args.size(),
        [&](string_t input) {
            auto input_data = input.GetData();
            auto input_len = input.GetSize();

            // 尝试 ASCII 快速路径
            if (!ASCII_OP::RequiresICU(input_data, input_len)) {
                auto output = StringVector::EmptyString(result, input_len);
                ASCII_OP::Operation(input_data, input_len, output.GetDataWriteable());
                output.Finalize();
                return output;
            }

            // 回退到 ICU
            return ICU_OP::Operation(input, result);
        });
}
```

### 3.2 算术运算符

```
源文件: src/function/scalar/operator/add.cpp
```

```cpp
// 加法操作结构体
struct AddOperator {
    template <class TA, class TB, class TR>
    static inline TR Operation(TA left, TB right) {
        return left + right;
    }
};

// 带溢出检查的加法
struct AddOperatorOverflowCheck {
    template <class TA, class TB, class TR>
    static inline TR Operation(TA left, TB right) {
        TR result;
        if (!TryAddOperator::Operation(left, right, result)) {
            throw OutOfRangeException("Overflow in addition");
        }
        return result;
    }
};

// 尝试加法（用于 TRY_CAST 等场景）
struct TryAddOperator {
    template <class TA, class TB, class TR>
    static inline bool Operation(TA left, TB right, TR &result) {
        // 使用编译器内置溢出检查
        return !__builtin_add_overflow(left, right, &result);
    }
};

// 注册加法函数
ScalarFunctionSet GetAddFunction() {
    ScalarFunctionSet add("+");

    // 整数类型
    add.AddFunction(ScalarFunction(
        {LogicalType::TINYINT, LogicalType::TINYINT},
        LogicalType::TINYINT,
        ScalarFunction::BinaryFunction<int8_t, int8_t, int8_t, AddOperator>));

    add.AddFunction(ScalarFunction(
        {LogicalType::INTEGER, LogicalType::INTEGER},
        LogicalType::INTEGER,
        ScalarFunction::BinaryFunction<int32_t, int32_t, int32_t, AddOperator>));

    add.AddFunction(ScalarFunction(
        {LogicalType::BIGINT, LogicalType::BIGINT},
        LogicalType::BIGINT,
        ScalarFunction::BinaryFunction<int64_t, int64_t, int64_t, AddOperator>));

    // 浮点类型
    add.AddFunction(ScalarFunction(
        {LogicalType::DOUBLE, LogicalType::DOUBLE},
        LogicalType::DOUBLE,
        ScalarFunction::BinaryFunction<double, double, double, AddOperator>));

    // 日期 + 间隔
    add.AddFunction(ScalarFunction(
        {LogicalType::DATE, LogicalType::INTERVAL},
        LogicalType::TIMESTAMP,
        DateAddFunction));

    return add;
}
```

### 3.3 子字符串函数

```cpp
// substring(string, start, length)
struct SubstringFun {
    static void SubstringFunction(DataChunk &args, ExpressionState &state, Vector &result) {
        auto &str_vec = args.data[0];
        auto &start_vec = args.data[1];
        auto &length_vec = args.data[2];

        TernaryExecutor::Execute<string_t, int64_t, int64_t, string_t>(
            str_vec, start_vec, length_vec, result, args.size(),
            [&](string_t str, int64_t start, int64_t length) {
                // SQL 索引从 1 开始
                if (start < 1) {
                    // 负索引处理
                    length += start - 1;
                    start = 1;
                }
                if (length <= 0) {
                    return string_t();
                }

                auto str_data = str.GetData();
                auto str_len = str.GetSize();

                // 转换为 0 索引
                idx_t byte_start = start - 1;
                if (byte_start >= str_len) {
                    return string_t();
                }

                idx_t byte_length = MinValue<idx_t>(length, str_len - byte_start);
                return StringVector::AddString(result, str_data + byte_start, byte_length);
            });
    }

    static ScalarFunction GetFunction() {
        return ScalarFunction(
            "substring",
            {LogicalType::VARCHAR, LogicalType::BIGINT, LogicalType::BIGINT},
            LogicalType::VARCHAR,
            SubstringFunction);
    }
};
```

## 4. Lambda 函数支持

### 4.1 ListLambdaBindData

```
源文件: src/include/duckdb/function/lambda_functions.hpp
```

```cpp
struct ListLambdaBindData final : public FunctionData {
    //! 返回类型
    LogicalType return_type;
    //! Lambda 表达式
    unique_ptr<Expression> lambda_expr;
    //! 是否包含索引参数
    bool has_index;
    //! 是否有初始值（用于 reduce）
    bool has_initial;

    unique_ptr<FunctionData> Copy() const override {
        auto lambda_expr_copy = lambda_expr ? lambda_expr->Copy() : nullptr;
        return make_uniq<ListLambdaBindData>(
            return_type, std::move(lambda_expr_copy), has_index, has_initial);
    }
};
```

### 4.2 list_transform 实现

```cpp
// list_transform(list, x -> expr)
// 对列表每个元素应用 lambda 表达式

static void ListTransformFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    // 获取 lambda 信息
    auto &func_expr = state.expr.Cast<BoundFunctionExpression>();
    auto &bind_info = func_expr.bind_info->Cast<ListLambdaBindData>();

    auto &list_column = args.data[0];
    auto count = args.size();

    // 获取列表数据
    UnifiedVectorFormat list_data;
    list_column.ToUnifiedFormat(count, list_data);
    auto list_entries = UnifiedVectorFormat::GetData<list_entry_t>(list_data);

    // 获取子向量
    auto &child_vector = ListVector::GetEntry(list_column);
    auto child_count = ListVector::GetListSize(list_column);

    // 准备结果列表
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto result_entries = FlatVector::GetData<list_entry_t>(result);
    auto &result_child = ListVector::GetEntry(result);

    // 准备 lambda 执行
    DataChunk lambda_input;
    lambda_input.Initialize(Allocator::Get(state.context),
                            {child_vector.GetType()});

    DataChunk lambda_result;
    lambda_result.Initialize(Allocator::Get(state.context),
                             {bind_info.return_type});

    // 创建表达式执行器
    ExpressionExecutor executor(state.context);
    executor.AddExpression(*bind_info.lambda_expr);

    idx_t result_offset = 0;
    for (idx_t i = 0; i < count; i++) {
        auto list_idx = list_data.sel->get_index(i);

        if (!list_data.validity.RowIsValid(list_idx)) {
            FlatVector::SetNull(result, i, true);
            continue;
        }

        auto &entry = list_entries[list_idx];
        result_entries[i].offset = result_offset;
        result_entries[i].length = entry.length;

        // 对每个子元素执行 lambda
        for (idx_t j = 0; j < entry.length; j++) {
            // 准备输入
            lambda_input.Reset();
            lambda_input.SetCardinality(1);
            // 复制单个元素到输入
            VectorOperations::Copy(child_vector, lambda_input.data[0],
                                   entry.offset + j, entry.offset + j + 1, 0);

            // 执行 lambda
            lambda_result.Reset();
            executor.Execute(lambda_input, lambda_result);

            // 复制结果
            VectorOperations::Copy(lambda_result.data[0], result_child,
                                   0, 1, result_offset + j);
        }

        result_offset += entry.length;
    }

    ListVector::SetListSize(result, result_offset);
}
```

### 4.3 list_filter 实现

```cpp
// list_filter(list, x -> predicate)
// 过滤列表，保留满足条件的元素

static void ListFilterFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &bind_info = state.expr.Cast<BoundFunctionExpression>()
                          .bind_info->Cast<ListLambdaBindData>();

    // ... 类似 list_transform 的设置 ...

    idx_t result_offset = 0;
    for (idx_t i = 0; i < count; i++) {
        auto list_idx = list_data.sel->get_index(i);

        if (!list_data.validity.RowIsValid(list_idx)) {
            FlatVector::SetNull(result, i, true);
            continue;
        }

        auto &entry = list_entries[list_idx];
        result_entries[i].offset = result_offset;

        idx_t kept_count = 0;
        for (idx_t j = 0; j < entry.length; j++) {
            // 执行 lambda 得到布尔值
            lambda_input.Reset();
            lambda_input.SetCardinality(1);
            VectorOperations::Copy(child_vector, lambda_input.data[0],
                                   entry.offset + j, entry.offset + j + 1, 0);

            lambda_result.Reset();
            executor.Execute(lambda_input, lambda_result);

            // 检查谓词结果
            auto filter_result = FlatVector::GetData<bool>(lambda_result.data[0]);
            if (filter_result[0]) {
                // 保留此元素
                VectorOperations::Copy(child_vector, result_child,
                                       entry.offset + j, entry.offset + j + 1,
                                       result_offset + kept_count);
                kept_count++;
            }
        }

        result_entries[i].length = kept_count;
        result_offset += kept_count;
    }

    ListVector::SetListSize(result, result_offset);
}
```

## 5. 绑定回调详解

### 5.1 类型推导

```cpp
// 示例：array_extract 函数的绑定
static unique_ptr<FunctionData> ArrayExtractBind(
    ClientContext &context,
    ScalarFunction &bound_function,
    vector<unique_ptr<Expression>> &arguments) {

    // 获取数组类型
    auto &array_type = arguments[0]->return_type;
    if (array_type.id() != LogicalTypeId::ARRAY &&
        array_type.id() != LogicalTypeId::LIST) {
        throw BinderException("array_extract requires an array or list argument");
    }

    // 推导返回类型为元素类型
    bound_function.return_type = ArrayType::GetChildType(array_type);

    // 无需额外绑定数据
    return nullptr;
}
```

### 5.2 创建 FunctionData

```cpp
// 示例：regexp_matches 函数的绑定
struct RegexpBindData : public FunctionData {
    unique_ptr<RE2> regex;
    string pattern;
    RE2::Options options;

    unique_ptr<FunctionData> Copy() const override {
        auto result = make_uniq<RegexpBindData>();
        result->pattern = pattern;
        result->options = options;
        result->regex = make_uniq<RE2>(pattern, options);
        return result;
    }

    bool Equals(const FunctionData &other_p) const override {
        auto &other = other_p.Cast<RegexpBindData>();
        return pattern == other.pattern;
    }
};

static unique_ptr<FunctionData> RegexpMatchesBind(
    ClientContext &context,
    ScalarFunction &bound_function,
    vector<unique_ptr<Expression>> &arguments) {

    // 检查模式是否为常量
    if (!arguments[1]->IsFoldable()) {
        throw BinderException("Pattern must be a constant string");
    }

    // 计算常量模式
    auto pattern_value = ExpressionExecutor::EvaluateScalar(context, *arguments[1]);
    if (pattern_value.IsNull()) {
        return nullptr;
    }

    auto result = make_uniq<RegexpBindData>();
    result->pattern = StringValue::Get(pattern_value);

    // 编译正则表达式
    result->regex = make_uniq<RE2>(result->pattern, result->options);
    if (!result->regex->ok()) {
        throw BinderException("Invalid regex pattern: %s", result->regex->error());
    }

    return std::move(result);
}
```

### 5.3 统计传播

```cpp
// 示例：abs 函数的统计传播
static unique_ptr<BaseStatistics> AbsStatistics(
    ClientContext &context,
    FunctionStatisticsInput &input) {

    auto &child_stats = input.child_stats;
    if (child_stats.empty() || !NumericStats::HasMinMax(child_stats[0])) {
        return nullptr;
    }

    auto &stats = child_stats[0];
    auto min_val = NumericStats::Min(stats);
    auto max_val = NumericStats::Max(stats);

    // abs 的范围是 [0, max(abs(min), abs(max))]
    Value new_min = Value::BIGINT(0);
    Value new_max = Value::BIGINT(
        MaxValue(AbsValue(min_val.GetValue<int64_t>()),
                 AbsValue(max_val.GetValue<int64_t>())));

    auto result = NumericStats::CreateEmpty(input.expr.return_type);
    NumericStats::SetMin(result, new_min);
    NumericStats::SetMax(result, new_max);
    return result.ToUnique();
}
```

## 6. NULL 处理模式

### 6.1 DEFAULT_NULL_HANDLING

默认模式下，执行器自动处理 NULL：

```cpp
// ExpressionExecutor 中的 NULL 处理
void ExecuteExpression(ExpressionState &state, Vector &result) {
    auto &func = state.expr.Cast<BoundFunctionExpression>();

    if (func.function.null_handling == FunctionNullHandling::DEFAULT_NULL_HANDLING) {
        // 检查输入是否有 NULL
        bool has_null = false;
        for (auto &child : state.child_states) {
            if (!child->validity.AllValid()) {
                has_null = true;
                break;
            }
        }

        if (has_null) {
            // 设置结果中的 NULL
            auto &result_validity = FlatVector::Validity(result);
            for (idx_t i = 0; i < count; i++) {
                bool is_null = false;
                for (auto &child : state.child_states) {
                    if (!child->validity.RowIsValid(i)) {
                        is_null = true;
                        break;
                    }
                }
                if (is_null) {
                    result_validity.SetInvalid(i);
                }
            }
        }
    }

    // 调用实际函数
    func.function.function(input_chunk, state, result);
}
```

### 6.2 SPECIAL_HANDLING

需要感知 NULL 的函数使用特殊处理：

```cpp
// coalesce 函数实现
static void CoalesceFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    result.SetVectorType(VectorType::CONSTANT_VECTOR);

    for (idx_t i = 0; i < args.size(); i++) {
        // 遍历每一行
        bool found = false;

        for (idx_t col = 0; col < args.ColumnCount(); col++) {
            auto &arg = args.data[col];
            UnifiedVectorFormat arg_data;
            arg.ToUnifiedFormat(args.size(), arg_data);

            auto idx = arg_data.sel->get_index(i);
            if (arg_data.validity.RowIsValid(idx)) {
                // 找到非 NULL 值
                VectorOperations::Copy(arg, result, i, i + 1, i);
                found = true;
                break;
            }
        }

        if (!found) {
            FlatVector::SetNull(result, i, true);
        }

        result.SetVectorType(VectorType::FLAT_VECTOR);
    }
}

// 注册时指定特殊 NULL 处理
ScalarFunction coalesce_func(
    "coalesce",
    {LogicalType::ANY},
    LogicalType::ANY,
    CoalesceFunction,
    nullptr,
    nullptr,
    nullptr,
    nullptr,
    LogicalType::ANY,  // 变参
    FunctionStability::CONSISTENT,
    FunctionNullHandling::SPECIAL_HANDLING  // 特殊处理
);
```

## 7. 函数注册

### 7.1 通过 FunctionList 注册

```cpp
// src/function/function_list.cpp

// 函数定义宏
#define DUCKDB_SCALAR_FUNCTION(FUNC)                                      \
    { #FUNC, nullptr, nullptr, nullptr, nullptr, nullptr,                 \
      FUNC::GetFunction, nullptr, nullptr, nullptr }

#define DUCKDB_SCALAR_FUNCTION_SET(FUNC)                                  \
    { #FUNC, nullptr, nullptr, nullptr, nullptr, nullptr,                 \
      nullptr, FUNC::GetFunctions, nullptr, nullptr }

// 函数表
static const StaticFunctionDefinition functions[] = {
    DUCKDB_SCALAR_FUNCTION(AbsFun),
    DUCKDB_SCALAR_FUNCTION(LowerFun),
    DUCKDB_SCALAR_FUNCTION(UpperFun),
    DUCKDB_SCALAR_FUNCTION_SET(OperatorAddFun),
    DUCKDB_SCALAR_FUNCTION_SET(SubstringFun),
    // ... 更多函数
    FINAL_FUNCTION
};
```

### 7.2 函数实现模式

```cpp
// 标准函数实现模式
struct MyFunctionFun {
    // 获取单个函数
    static ScalarFunction GetFunction() {
        return ScalarFunction(
            "my_function",
            {LogicalType::VARCHAR},
            LogicalType::VARCHAR,
            MyFunctionImplementation);
    }

    // 或获取函数集（多个重载）
    static ScalarFunctionSet GetFunctions() {
        ScalarFunctionSet set("my_function");
        set.AddFunction(ScalarFunction({LogicalType::INTEGER}, ...));
        set.AddFunction(ScalarFunction({LogicalType::VARCHAR}, ...));
        return set;
    }
};
```

## 8. 总结

### 8.1 标量函数设计要点

1. **向量化**：使用 Executor 模板处理整个向量
2. **NULL 处理**：默认模式自动处理，特殊需求用 SPECIAL_HANDLING
3. **类型安全**：通过 bind 回调进行类型推导
4. **性能优化**：编译期正则表达式、常量折叠支持
5. **统计传播**：支持优化器决策

### 8.2 函数实现检查清单

| 检查项 | 说明 |
|--------|------|
| 参数类型 | 定义所有支持的参数组合 |
| 返回类型 | 固定或通过 bind 推导 |
| NULL 处理 | 选择合适的处理模式 |
| 稳定性 | CONSISTENT/VOLATILE/CONSISTENT_WITHIN_QUERY |
| 绑定数据 | 需要预计算的数据 |
| 局部状态 | 线程私有数据 |
| 统计传播 | 优化器信息 |
| 序列化 | 计划持久化支持 |

下一章将深入分析聚合函数的状态机模型，包括 update/combine/finalize 生命周期和窗口聚合支持。
