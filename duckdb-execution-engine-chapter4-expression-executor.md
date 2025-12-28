# DuckDB 执行引擎深度解析 - 第四章：表达式执行器

## 引言

表达式执行器（ExpressionExecutor）是 DuckDB 执行引擎的核心组件之一，负责计算 SQL 查询中的各种表达式，包括算术运算、比较操作、函数调用、CASE WHEN 逻辑等。它将上一章介绍的向量化数据结构（Vector、DataChunk）与表达式树结合，实现高效的批量表达式求值。

本章深入分析 ExpressionExecutor 的架构设计、状态管理、各类表达式的执行实现，以及自适应过滤等高级优化技术。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     表达式执行器架构                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  表达式树 (Expression Tree)           ExpressionExecutor                │
│  ────────────────────────           ──────────────────────              │
│                                                                         │
│       BoundFunctionExpression        ┌──────────────────────┐          │
│            (a + b * c)               │ ExpressionExecutor   │          │
│              /    \                  │ ┌────────────────┐   │          │
│    BoundRef   BoundFunction          │ │ expressions[]  │   │          │
│      (a)        (b * c)              │ │ states[]       │   │          │
│                /     \               │ │ chunk          │   │          │
│         BoundRef  BoundRef           │ └────────────────┘   │          │
│           (b)       (c)              └──────────┬───────────┘          │
│                                                 │                       │
│                                                 ▼                       │
│                                      ┌──────────────────────┐          │
│                                      │ ExpressionState      │          │
│                                      │ (每个表达式的状态)     │          │
│                                      │ ┌────────────────┐   │          │
│                                      │ │ child_states[] │   │          │
│                                      │ │ intermediate   │   │          │
│                                      │ │ _chunk         │   │          │
│                                      │ └────────────────┘   │          │
│                                      └──────────────────────┘          │
│                                                                         │
│  输入 DataChunk ──────────────────────────────────────► 输出 Vector    │
│  [col_a, col_b, col_c]              Execute()           [result]       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.1 ExpressionExecutor 架构

### 4.1.1 ExpressionExecutor 类结构

```cpp
// src/include/duckdb/execution/expression_executor.hpp

class ExpressionExecutor {
public:
    //! 要执行的表达式列表
    vector<const Expression *> expressions;

    //! 当前物理算子的 DataChunk，用于解析列引用
    DataChunk *chunk = nullptr;

private:
    //! 客户端上下文
    optional_ptr<ClientContext> context;

    //! 表达式状态数组（每个表达式一个状态）
    vector<unique_ptr<ExpressionExecutorState>> states;

public:
    // 构造函数：从单个或多个表达式创建
    explicit ExpressionExecutor(ClientContext &context);
    ExpressionExecutor(ClientContext &context, const Expression *expression);
    ExpressionExecutor(ClientContext &context, const Expression &expression);
    ExpressionExecutor(ClientContext &context, const vector<unique_ptr<Expression>> &expressions);

    // 添加表达式
    void AddExpression(const Expression &expr);

    // 执行所有表达式，结果存入 DataChunk
    void Execute(DataChunk *input, DataChunk &result);

    // 执行单个表达式，结果存入 Vector
    void ExecuteExpression(DataChunk &input, Vector &result);
    void ExecuteExpression(idx_t expr_idx, Vector &result);

    // 执行布尔表达式并生成选择向量
    idx_t SelectExpression(DataChunk &input, SelectionVector &sel);

    // 静态方法：评估标量表达式
    static Value EvaluateScalar(ClientContext &context, const Expression &expr,
                                bool allow_unfoldable = false);
};
```

### 4.1.2 执行流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ExpressionExecutor 执行流程                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 初始化阶段 (AddExpression)                                           │
│     ┌─────────────────────────────────────────────────┐                │
│     │ for each expression:                            │                │
│     │   1. 创建 ExpressionExecutorState               │                │
│     │   2. 调用 InitializeState() 递归初始化子表达式    │                │
│     │   3. 分配中间结果缓冲区                          │                │
│     └─────────────────────────────────────────────────┘                │
│                                                                         │
│  2. 执行阶段 (Execute)                                                   │
│     ┌─────────────────────────────────────────────────┐                │
│     │ for each expression:                            │                │
│     │   1. SetChunk(input) - 设置输入数据              │                │
│     │   2. Execute(expr, state, sel, count, result)   │                │
│     │      └─ 根据表达式类型分发到具体执行方法          │                │
│     │   3. Verify() - 验证结果（调试模式）             │                │
│     └─────────────────────────────────────────────────┘                │
│                                                                         │
│  3. 递归执行（以函数表达式为例）                                          │
│     ┌─────────────────────────────────────────────────┐                │
│     │ Execute(BoundFunctionExpression):               │                │
│     │   1. 重置 intermediate_chunk                    │                │
│     │   2. 递归执行每个子表达式                        │                │
│     │   3. 调用函数回调 function.GetFunctionCallback() │                │
│     │   4. 验证 NULL 处理                             │                │
│     └─────────────────────────────────────────────────┘                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.2 ExpressionState：表达式状态

每个表达式在执行时都需要维护状态，包括子表达式状态和中间结果缓冲区。

### 4.2.1 ExpressionState 结构

```cpp
// src/include/duckdb/execution/expression_executor_state.hpp

struct ExpressionState {
    //! 对应的表达式引用
    const Expression &expr;

    //! 根状态引用（访问 executor）
    ExpressionExecutorState &root;

    //! 子表达式状态数组
    vector<unique_ptr<ExpressionState>> child_states;

    //! 子表达式的类型列表
    vector<LogicalType> types;

    //! 中间结果 DataChunk（存储子表达式结果）
    DataChunk intermediate_chunk;

    //! 初始化标志
    vector<bool> initialize;

public:
    // 添加子表达式
    void AddChild(Expression &child_expr);

    // 完成初始化（分配 intermediate_chunk）
    void Finalize();
};

struct ExpressionExecutorState {
    //! 根表达式状态
    unique_ptr<ExpressionState> root_state;

    //! 关联的 executor
    ExpressionExecutor *executor = nullptr;
};
```

### 4.2.2 状态初始化

```cpp
void ExpressionState::AddChild(Expression &child_expr) {
    types.push_back(child_expr.return_type);
    initialize.push_back(false);
    // 递归初始化子表达式状态
    auto child_state = ExpressionExecutor::InitializeState(child_expr, root);
    child_states.push_back(std::move(child_state));
}

void ExpressionState::Finalize() {
    if (!types.empty()) {
        // 分配中间结果缓冲区
        intermediate_chunk.Initialize(GetAllocator(), types);
    }
}
```

### 4.2.3 特化状态类型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ExpressionState 类型层次                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ExpressionState (基类)                                                  │
│       │                                                                 │
│       ├── ExecuteFunctionState                                          │
│       │   └── 函数执行状态                                               │
│       │       • local_state: 函数本地状态                                │
│       │       • output_dictionary: 字典优化缓存                          │
│       │       • TryExecuteDictionaryExpression(): 字典优化               │
│       │                                                                 │
│       ├── CaseExpressionState                                           │
│       │   └── CASE WHEN 执行状态                                         │
│       │       • true_sel: 真值选择向量                                   │
│       │       • false_sel: 假值选择向量                                  │
│       │                                                                 │
│       └── ConjunctionState                                              │
│           └── AND/OR 执行状态                                            │
│               • adaptive_filter: 自适应过滤器                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.3 表达式类型与执行

DuckDB 支持多种绑定表达式类型，每种类型都有对应的执行实现。

### 4.3.1 表达式类型概览

```cpp
// 绑定表达式类型（部分）
enum class ExpressionClass : uint8_t {
    BOUND_AGGREGATE,     // 聚合表达式
    BOUND_BETWEEN,       // BETWEEN 表达式
    BOUND_CASE,          // CASE WHEN 表达式
    BOUND_CAST,          // 类型转换表达式
    BOUND_COLUMN_REF,    // 列引用表达式
    BOUND_COMPARISON,    // 比较表达式
    BOUND_CONJUNCTION,   // 逻辑连接表达式 (AND/OR)
    BOUND_CONSTANT,      // 常量表达式
    BOUND_FUNCTION,      // 函数表达式
    BOUND_OPERATOR,      // 操作符表达式
    BOUND_PARAMETER,     // 参数表达式
    BOUND_REF,           // 引用表达式
    BOUND_SUBQUERY,      // 子查询表达式
    BOUND_WINDOW,        // 窗口表达式
    // ...
};
```

### 4.3.2 执行分发

```cpp
void ExpressionExecutor::Execute(const Expression &expr, ExpressionState *state,
                                  const SelectionVector *sel, idx_t count, Vector &result) {
    switch (expr.GetExpressionClass()) {
    case ExpressionClass::BOUND_BETWEEN:
        Execute(expr.Cast<BoundBetweenExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CASE:
        Execute(expr.Cast<BoundCaseExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CAST:
        Execute(expr.Cast<BoundCastExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_COMPARISON:
        Execute(expr.Cast<BoundComparisonExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CONJUNCTION:
        Execute(expr.Cast<BoundConjunctionExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CONSTANT:
        Execute(expr.Cast<BoundConstantExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_FUNCTION:
        Execute(expr.Cast<BoundFunctionExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_REF:
        Execute(expr.Cast<BoundReferenceExpression>(), state, sel, count, result);
        break;
    // ... 其他类型
    }
}
```

---

## 4.4 常量与引用表达式

### 4.4.1 常量表达式 (BoundConstantExpression)

常量表达式是最简单的表达式类型，直接返回预计算的常量值。

```cpp
// src/execution/expression_executor/execute_constant.cpp

void ExpressionExecutor::Execute(const BoundConstantExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    D_ASSERT(expr.value.type() == expr.return_type);
    // 直接引用常量值，创建 CONSTANT_VECTOR
    result.Reference(expr.value);
}
```

**执行特点**：
- 零复制：直接引用预存储的 Value
- 结果是 CONSTANT_VECTOR 类型
- 无论 count 多大，只存储一个值

### 4.4.2 引用表达式 (BoundReferenceExpression)

引用表达式用于访问输入 DataChunk 中的特定列。

```cpp
// src/execution/expression_executor/execute_reference.cpp

void ExpressionExecutor::Execute(const BoundReferenceExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    D_ASSERT(expr.index != DConstants::INVALID_INDEX);
    D_ASSERT(expr.index < chunk->ColumnCount());

    if (sel) {
        // 有选择向量时，切片输入列
        result.Slice(chunk->data[expr.index], *sel, count);
    } else {
        // 无选择向量时，直接引用输入列
        result.Reference(chunk->data[expr.index]);
    }
}
```

**执行特点**：
- 通过 `expr.index` 定位输入列
- 支持 SelectionVector 切片
- 零复制引用或切片

---

## 4.5 函数表达式

函数表达式是最常见的表达式类型，涵盖算术运算、字符串操作、日期函数等。

### 4.5.1 函数执行实现

```cpp
// src/execution/expression_executor/execute_function.cpp

void ExpressionExecutor::Execute(const BoundFunctionExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    // 1. 重置中间结果缓冲区
    state->intermediate_chunk.Reset();
    auto &arguments = state->intermediate_chunk;

    // 2. 递归执行每个子表达式（函数参数）
    if (!state->types.empty()) {
        for (idx_t i = 0; i < expr.children.size(); i++) {
            D_ASSERT(state->types[i] == expr.children[i]->return_type);
            Execute(*expr.children[i], state->child_states[i].get(), sel, count,
                    arguments.data[i]);
        }
    }
    arguments.SetCardinality(count);

    // 3. 尝试字典优化
    auto &execute_function_state = state->Cast<ExecuteFunctionState>();
    if (!execute_function_state.TryExecuteDictionaryExpression(expr, arguments, *state, result)) {
        // 4. 调用函数回调
        expr.function.GetFunctionCallback()(arguments, *state, result);
    }

    // 5. 验证 NULL 处理（调试模式）
    VerifyNullHandling(expr, arguments, result);
}
```

### 4.5.2 字典优化 (Dictionary Optimization)

对于满足条件的函数，可以在字典级别执行一次，避免对每行重复计算。

```cpp
bool ExecuteFunctionState::TryExecuteDictionaryExpression(
    const BoundFunctionExpression &expr, DataChunk &args,
    ExpressionState &state, Vector &result) {

    static constexpr idx_t MAX_DICTIONARY_SIZE_THRESHOLD = 20000;
    static constexpr double CHUNK_FILL_RATIO_THRESHOLD = 0.5;

    // 检查是否满足优化条件
    if (!input_col_idx.IsValid()) {
        return false;  // 表达式不符合优化条件
    }

    const auto &unary_input = args.data[input_col_idx.GetIndex()];
    if (unary_input.GetVectorType() != VectorType::DICTIONARY_VECTOR) {
        return false;  // 输入不是字典向量
    }

    const auto input_dictionary_size = DictionaryVector::DictionarySize(unary_input).GetIndex();
    if (input_dictionary_size >= MAX_DICTIONARY_SIZE_THRESHOLD) {
        return false;  // 字典太大
    }

    // 执行字典优化
    if (!output_dictionary || current_input_dictionary_id != input_dictionary_id) {
        // 重新计算：在整个字典上执行函数
        output_dictionary = DictionaryVector::CreateReusableDictionary(
            result.GetType(), input_dictionary_size);

        // 分批处理字典（每批 STANDARD_VECTOR_SIZE）
        for (idx_t offset = 0; offset < input_dictionary_size;
             offset += STANDARD_VECTOR_SIZE) {
            const auto count = MinValue(input_dictionary_size - offset, STANDARD_VECTOR_SIZE);

            // 在字典切片上执行函数
            Vector output_intermediate(result.GetType());
            expr.function.GetFunctionCallback()(input_chunk, state, output_intermediate);

            // 复制结果到输出字典
            VectorOperations::Copy(output_intermediate, output_dictionary->data,
                                   count, 0, offset);
        }
        current_input_dictionary_id = input_dictionary_id;
    }

    // 结果引用字典
    result.Dictionary(output_dictionary, DictionaryVector::SelVector(unary_input));
    return true;
}
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        字典优化示例                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景：SELECT UPPER(city) FROM users                                    │
│                                                                         │
│  输入（字典编码）:                                                       │
│    字典: ["Beijing", "Shanghai", "Guangzhou"]                          │
│    选择: [0, 1, 0, 2, 1, 0, 0, 1, 2, 0, ...]  (2048个索引)             │
│                                                                         │
│  传统执行（无优化）:                                                     │
│    对 2048 行分别调用 UPPER()                                           │
│    计算量: 2048 次字符串操作                                             │
│                                                                         │
│  字典优化执行:                                                           │
│    1. 在字典上执行: UPPER(["Beijing","Shanghai","Guangzhou"])           │
│       → ["BEIJING", "SHANGHAI", "GUANGZHOU"]                           │
│    2. 结果使用相同的选择向量                                             │
│    计算量: 3 次字符串操作                                                │
│                                                                         │
│  优化条件:                                                               │
│    • 函数是确定性的 (IsConsistent)                                       │
│    • 函数是非易失的 (!IsVolatile)                                        │
│    • 函数不会抛出异常 (!CanThrow)                                        │
│    • 字典大小 < 20000                                                   │
│    • 只有一个非常量输入                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.6 比较表达式

比较表达式用于实现 `=`, `<>`, `<`, `>`, `<=`, `>=` 等操作。

### 4.6.1 比较执行实现

```cpp
// src/execution/expression_executor/execute_comparison.cpp

void ExpressionExecutor::Execute(const BoundComparisonExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    // 执行子表达式
    state->intermediate_chunk.Reset();
    auto &left = state->intermediate_chunk.data[0];
    auto &right = state->intermediate_chunk.data[1];

    Execute(*expr.left, state->child_states[0].get(), sel, count, left);
    Execute(*expr.right, state->child_states[1].get(), sel, count, right);

    // 根据比较类型调用对应的向量操作
    switch (expr.GetExpressionType()) {
    case ExpressionType::COMPARE_EQUAL:
        VectorOperations::Equals(left, right, result, count);
        break;
    case ExpressionType::COMPARE_NOTEQUAL:
        VectorOperations::NotEquals(left, right, result, count);
        break;
    case ExpressionType::COMPARE_LESSTHAN:
        VectorOperations::LessThan(left, right, result, count);
        break;
    case ExpressionType::COMPARE_GREATERTHAN:
        VectorOperations::GreaterThan(left, right, result, count);
        break;
    case ExpressionType::COMPARE_LESSTHANOREQUALTO:
        VectorOperations::LessThanEquals(left, right, result, count);
        break;
    case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
        VectorOperations::GreaterThanEquals(left, right, result, count);
        break;
    case ExpressionType::COMPARE_DISTINCT_FROM:
        VectorOperations::DistinctFrom(left, right, result, count);
        break;
    case ExpressionType::COMPARE_NOT_DISTINCT_FROM:
        VectorOperations::NotDistinctFrom(left, right, result, count);
        break;
    }
}
```

### 4.6.2 Select 模式

比较表达式支持 Select 模式，直接生成选择向量而不是布尔结果。

```cpp
idx_t ExpressionExecutor::Select(const BoundComparisonExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count,
                                 SelectionVector *true_sel, SelectionVector *false_sel) {
    // 执行子表达式
    state->intermediate_chunk.Reset();
    auto &left = state->intermediate_chunk.data[0];
    auto &right = state->intermediate_chunk.data[1];

    Execute(*expr.left, state->child_states[0].get(), sel, count, left);
    Execute(*expr.right, state->child_states[1].get(), sel, count, right);

    // 返回满足条件的行数
    switch (expr.GetExpressionType()) {
    case ExpressionType::COMPARE_EQUAL:
        return VectorOperations::Equals(left, right, sel, count, true_sel, false_sel);
    case ExpressionType::COMPARE_LESSTHAN:
        return VectorOperations::LessThan(left, right, sel, count, true_sel, false_sel);
    // ...
    }
}
```

### 4.6.3 模板化比较实现

```cpp
template <class OP>
static idx_t TemplatedSelectOperation(Vector &left, Vector &right,
                                       optional_ptr<const SelectionVector> sel, idx_t count,
                                       optional_ptr<SelectionVector> true_sel,
                                       optional_ptr<SelectionVector> false_sel,
                                       optional_ptr<ValidityMask> null_mask) {
    // 更新 NULL 掩码
    if (null_mask) {
        UpdateNullMask(left, sel, count, *null_mask);
        UpdateNullMask(right, sel, count, *null_mask);
    }

    // 根据物理类型选择模板实例
    switch (left.GetType().InternalType()) {
    case PhysicalType::INT32:
        return BinaryExecutor::Select<int32_t, int32_t, OP>(
            left, right, sel.get(), count, true_sel.get(), false_sel.get());
    case PhysicalType::INT64:
        return BinaryExecutor::Select<int64_t, int64_t, OP>(
            left, right, sel.get(), count, true_sel.get(), false_sel.get());
    case PhysicalType::VARCHAR:
        return BinaryExecutor::Select<string_t, string_t, OP>(
            left, right, sel.get(), count, true_sel.get(), false_sel.get());
    // ... 其他类型
    }
}
```

---

## 4.7 逻辑连接表达式 (AND/OR)

### 4.7.1 AND 执行语义

AND 操作采用短路求值策略：一旦发现 false，立即返回。

```cpp
// src/execution/expression_executor/execute_conjunction.cpp

idx_t ExpressionExecutor::Select(const BoundConjunctionExpression &expr, ExpressionState *state_p,
                                 const SelectionVector *sel, idx_t count,
                                 SelectionVector *true_sel, SelectionVector *false_sel) {
    auto &state = state_p->Cast<ConjunctionState>();

    if (expr.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
        // AND: 逐步过滤
        auto filter_state = state.adaptive_filter->BeginFilter();
        const SelectionVector *current_sel = sel;
        idx_t current_count = count;
        idx_t false_count = 0;

        for (idx_t i = 0; i < expr.children.size(); i++) {
            // 使用自适应过滤器的排列顺序
            idx_t child_idx = state.adaptive_filter->permutation[i];

            // 执行子条件
            idx_t tcount = Select(*expr.children[child_idx],
                                  state.child_states[child_idx].get(),
                                  current_sel, current_count,
                                  true_sel, temp_false.get());

            // 将失败的行移到 false_sel
            idx_t fcount = current_count - tcount;
            if (fcount > 0 && false_sel) {
                for (idx_t j = 0; j < fcount; j++) {
                    false_sel->set_index(false_count++, temp_false->get_index(j));
                }
            }

            current_count = tcount;
            if (current_count == 0) {
                break;  // 短路：所有行都失败了
            }

            // 后续只处理通过的行
            current_sel = true_sel;
        }

        state.adaptive_filter->EndFilter(filter_state);
        return current_count;
    }
    // OR 逻辑类似但相反...
}
```

### 4.7.2 自适应过滤器 (Adaptive Filter)

DuckDB 使用自适应过滤器动态调整 AND 条件的执行顺序，将选择性高（过滤效果好）的条件提前执行。

```cpp
// src/include/duckdb/execution/adaptive_filter.hpp

class AdaptiveFilter {
public:
    //! 条件执行顺序（动态调整）
    vector<idx_t> permutation;

private:
    //! 迭代计数器
    idx_t iteration_count = 0;
    //! 交换索引
    idx_t swap_idx = 0;
    //! 观察间隔
    idx_t observe_interval = 0;

public:
    // 开始过滤，记录开始时间
    AdaptiveFilterState BeginFilter() const;

    // 结束过滤，根据执行时间调整顺序
    void EndFilter(AdaptiveFilterState state);

    // 根据运行时统计调整排列
    void AdaptRuntimeStatistics(double duration);
};
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    自适应过滤器工作原理                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景：WHERE a > 100 AND b < 50 AND c = 'X'                             │
│                                                                         │
│  初始顺序: [0, 1, 2]  →  a > 100, b < 50, c = 'X'                       │
│                                                                         │
│  运行时观察:                                                             │
│  ───────────                                                            │
│  Batch 1:  a > 100 过滤 5%,  b < 50 过滤 30%,  c = 'X' 过滤 90%         │
│  Batch 2:  类似统计...                                                  │
│                                                                         │
│  自适应调整:                                                             │
│  ───────────                                                            │
│  检测到 c = 'X' 最具选择性（过滤最多行）                                  │
│  新顺序: [2, 1, 0]  →  c = 'X', b < 50, a > 100                         │
│                                                                         │
│  优化效果:                                                               │
│  ───────────                                                            │
│  先执行 c = 'X' 过滤掉 90% 的行                                          │
│  剩余 10% 行再执行 b < 50 和 a > 100                                    │
│  显著减少计算量                                                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.8 CASE WHEN 表达式

### 4.8.1 CASE 执行实现

```cpp
// src/execution/expression_executor/execute_case.cpp

struct CaseExpressionState : public ExpressionState {
    CaseExpressionState(const Expression &expr, ExpressionExecutorState &root)
        : ExpressionState(expr, root),
          true_sel(STANDARD_VECTOR_SIZE),
          false_sel(STANDARD_VECTOR_SIZE) {}

    SelectionVector true_sel;   // 条件为真的行
    SelectionVector false_sel;  // 条件为假的行
};

void ExpressionExecutor::Execute(const BoundCaseExpression &expr, ExpressionState *state_p,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    auto &state = state_p->Cast<CaseExpressionState>();

    auto current_true_sel = &state.true_sel;
    auto current_false_sel = &state.false_sel;
    auto current_sel = sel;
    idx_t current_count = count;

    // 遍历每个 WHEN ... THEN ... 分支
    for (idx_t i = 0; i < expr.case_checks.size(); i++) {
        auto &case_check = expr.case_checks[i];

        // 执行 WHEN 条件，获取 true/false 选择向量
        idx_t tcount = Select(*case_check.when_expr, check_state,
                              current_sel, current_count,
                              current_true_sel, current_false_sel);

        if (tcount == 0) {
            // 所有行都是 false，跳过这个分支
            continue;
        }

        idx_t fcount = current_count - tcount;

        if (fcount == 0 && current_count == count) {
            // 第一个分支全部为 true，直接执行 THEN 表达式
            Execute(*case_check.then_expr, then_state, sel, count, result);
            return;
        } else {
            // 执行 THEN 表达式，填充结果
            Execute(*case_check.then_expr, then_state, current_true_sel, tcount, intermediate_result);
            FillSwitch(intermediate_result, result, *current_true_sel, tcount);
        }

        // 继续处理 false 的行
        current_sel = current_false_sel;
        current_count = fcount;

        if (fcount == 0) {
            // 所有行都已处理
            break;
        }
    }

    // 处理剩余行（ELSE 分支）
    if (current_count > 0) {
        if (current_count == count) {
            // 所有行都走 ELSE
            Execute(*expr.else_expr, else_state, sel, count, result);
        } else {
            Execute(*expr.else_expr, else_state, current_sel, current_count, intermediate_result);
            FillSwitch(intermediate_result, result, *current_sel, current_count);
        }
    }
}
```

### 4.8.2 FillSwitch：结果填充

```cpp
template <class T>
void TemplatedFillLoop(Vector &vector, Vector &result,
                       const SelectionVector &sel, sel_t count) {
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto res = FlatVector::GetData<T>(result);
    auto &result_mask = FlatVector::Validity(result);

    if (vector.GetVectorType() == VectorType::CONSTANT_VECTOR) {
        // 常量填充
        auto data = ConstantVector::GetData<T>(vector);
        if (ConstantVector::IsNull(vector)) {
            for (idx_t i = 0; i < count; i++) {
                result_mask.SetInvalid(sel.get_index(i));
            }
        } else {
            for (idx_t i = 0; i < count; i++) {
                res[sel.get_index(i)] = *data;
            }
        }
    } else {
        // 一般情况
        UnifiedVectorFormat vdata;
        vector.ToUnifiedFormat(count, vdata);
        auto data = UnifiedVectorFormat::GetData<T>(vdata);

        for (idx_t i = 0; i < count; i++) {
            auto source_idx = vdata.sel->get_index(i);
            auto res_idx = sel.get_index(i);

            res[res_idx] = data[source_idx];
            result_mask.Set(res_idx, vdata.validity.RowIsValid(source_idx));
        }
    }
}
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CASE WHEN 执行示例                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  SQL: CASE WHEN a > 50 THEN 'high'                                      │
│            WHEN a > 20 THEN 'medium'                                    │
│            ELSE 'low' END                                               │
│                                                                         │
│  输入:  a = [60, 30, 10, 80, 25, 5, 70, 15]                             │
│                                                                         │
│  执行过程:                                                               │
│  ───────────                                                            │
│  Round 1: 评估 a > 50                                                   │
│    true_sel  = [0, 3, 6]        → 索引 0,3,6 的值 > 50                  │
│    false_sel = [1, 2, 4, 5, 7]  → 其余行                                │
│    填充: result[0,3,6] = 'high'                                        │
│                                                                         │
│  Round 2: 评估 a > 20 (只对 false_sel)                                   │
│    当前行: [30, 10, 25, 5, 15]                                          │
│    true_sel  = [1, 4]           → 30, 25 > 20                           │
│    false_sel = [2, 5, 7]        → 其余行                                │
│    填充: result[1,4] = 'medium'                                        │
│                                                                         │
│  Round 3: ELSE 分支                                                      │
│    填充: result[2,5,7] = 'low'                                         │
│                                                                         │
│  最终结果: ['high','medium','low','high','medium','low','high','low']  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.9 类型转换表达式

### 4.9.1 Cast 执行实现

```cpp
// src/execution/expression_executor/execute_cast.cpp

void ExpressionExecutor::Execute(const BoundCastExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    auto lstate = ExecuteFunctionState::GetFunctionState(*state);

    // 执行子表达式
    state->intermediate_chunk.Reset();
    auto &child = state->intermediate_chunk.data[0];
    Execute(*expr.child, state->child_states[0].get(), sel, count, child);

    // 设置转换参数
    string error_message;
    auto error_ref = expr.try_cast ? &error_message : nullptr;
    CastParameters parameters(expr.bound_cast.cast_data.get(), false, error_ref, lstate);
    parameters.query_location = expr.GetQueryLocation();

    // 调用转换函数
    expr.bound_cast.function(child, result, count, parameters);
}
```

### 4.9.2 转换函数架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    类型转换架构                                          │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BoundCastExpression                                                    │
│       │                                                                 │
│       ├── child: Expression*           // 源表达式                      │
│       │                                                                 │
│       └── bound_cast: BoundCastInfo                                     │
│               │                                                         │
│               ├── function: cast_function_t                             │
│               │   └── 实际的转换函数                                     │
│               │                                                         │
│               ├── cast_data: unique_ptr<BoundCastData>                  │
│               │   └── 转换相关的额外数据                                 │
│               │                                                         │
│               └── init_local_state: cast_init_state_t                   │
│                   └── 初始化本地状态的函数                               │
│                                                                         │
│  转换函数签名:                                                           │
│  bool cast_function_t(Vector &source, Vector &result, idx_t count,     │
│                       CastParameters &parameters);                      │
│                                                                         │
│  常见转换:                                                               │
│  • INTEGER → BIGINT    (无损扩展)                                       │
│  • VARCHAR → INTEGER   (解析转换)                                       │
│  • TIMESTAMP → DATE    (截断转换)                                       │
│  • ANY → VARCHAR       (字符串化)                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.10 Select vs Execute 模式

ExpressionExecutor 支持两种执行模式：Execute 模式和 Select 模式。

### 4.10.1 模式对比

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Execute vs Select 模式                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Execute 模式                                                            │
│  ────────────                                                           │
│  • 返回: Vector (结果值)                                                 │
│  • 用途: 投影、计算、返回结果                                            │
│  • 示例: SELECT a + b                                                   │
│                                                                         │
│  void Execute(Expression &expr, ..., Vector &result) {                  │
│      // 计算表达式，结果存入 result                                       │
│  }                                                                      │
│                                                                         │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  Select 模式                                                             │
│  ───────────                                                            │
│  • 返回: SelectionVector (满足条件的行索引)                              │
│  • 用途: 过滤、WHERE 条件                                               │
│  • 示例: WHERE a > 100                                                  │
│                                                                         │
│  idx_t Select(Expression &expr, ...,                                    │
│               SelectionVector *true_sel,                                │
│               SelectionVector *false_sel) {                             │
│      // 评估条件，分类到 true_sel 和 false_sel                           │
│      return true_count;  // 返回满足条件的行数                           │
│  }                                                                      │
│                                                                         │
│  优势:                                                                   │
│  • Select 模式避免创建布尔 Vector                                        │
│  • 直接生成 SelectionVector 供后续算子使用                               │
│  • 支持 AND/OR 短路求值                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.10.2 SelectExpression 方法

```cpp
idx_t ExpressionExecutor::SelectExpression(DataChunk &input, SelectionVector &sel) {
    D_ASSERT(expressions.size() == 1);
    SetChunk(&input);
    // 使用 Select 模式执行布尔表达式
    return Select(*expressions[0], states[0]->root_state.get(),
                  nullptr, input.size(), &sel, nullptr);
}

idx_t ExpressionExecutor::SelectExpression(DataChunk &input, SelectionVector &result_sel,
                                            optional_ptr<SelectionVector> current_sel,
                                            idx_t current_count) {
    D_ASSERT(expressions.size() == 1);
    SetChunk(&input);
    // 在已有选择向量上继续过滤
    return Select(*expressions[0], states[0]->root_state.get(),
                  current_sel.get(), current_count, &result_sel, nullptr);
}
```

---

## 4.11 NULL 值处理

### 4.11.1 默认 NULL 处理

DuckDB 中大多数函数遵循 SQL 标准的 NULL 语义：任何参数为 NULL，结果也为 NULL。

```cpp
// 验证 NULL 处理（调试模式）
static void VerifyNullHandling(const BoundFunctionExpression &expr, DataChunk &args,
                               Vector &result) {
#ifdef DEBUG
    if (args.data.empty() ||
        expr.function.GetNullHandling() != FunctionNullHandling::DEFAULT_NULL_HANDLING) {
        return;
    }

    // 合并所有参数的 ValidityMask
    idx_t count = args.size();
    ValidityMask combined_mask(count);
    for (auto &arg : args.data) {
        UnifiedVectorFormat arg_data;
        arg.ToUnifiedFormat(count, arg_data);

        for (idx_t i = 0; i < count; i++) {
            auto idx = arg_data.sel->get_index(i);
            if (!arg_data.validity.RowIsValid(idx)) {
                combined_mask.SetInvalid(i);
            }
        }
    }

    // 验证：参数为 NULL 时，结果也必须为 NULL
    UnifiedVectorFormat result_data;
    result.ToUnifiedFormat(count, result_data);
    for (idx_t i = 0; i < count; i++) {
        if (!combined_mask.RowIsValid(i)) {
            auto idx = result_data.sel->get_index(i);
            D_ASSERT(!result_data.validity.RowIsValid(idx));
        }
    }
#endif
}
```

### 4.11.2 特殊 NULL 处理

某些函数有特殊的 NULL 处理逻辑：

```cpp
enum class FunctionNullHandling : uint8_t {
    DEFAULT_NULL_HANDLING,   // 默认：任何参数 NULL → 结果 NULL
    SPECIAL_HANDLING         // 特殊：函数自己处理 NULL
};

// 特殊处理的例子
// COALESCE(a, b, c) → 返回第一个非 NULL 值
// IFNULL(a, b) → 如果 a 为 NULL 返回 b
// IS NULL / IS NOT NULL → NULL 参与比较
```

---

## 4.12 性能优化总结

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    表达式执行优化技术                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 向量化执行                                                           │
│     • 批量处理 2048 行，分摊函数调用开销                                  │
│     • 中间结果使用 DataChunk 缓存                                        │
│                                                                         │
│  2. 字典优化                                                             │
│     • 对字典编码数据，只计算唯一值                                        │
│     • 显著减少字符串操作等昂贵计算                                        │
│                                                                         │
│  3. 常量折叠                                                             │
│     • 常量表达式预计算为 CONSTANT_VECTOR                                 │
│     • 执行时零开销                                                        │
│                                                                         │
│  4. 延迟物化                                                             │
│     • Select 模式直接生成 SelectionVector                                │
│     • 避免创建不必要的布尔 Vector                                        │
│                                                                         │
│  5. 短路求值                                                             │
│     • AND/OR 表达式尽早终止                                              │
│     • 减少不必要的子表达式计算                                            │
│                                                                         │
│  6. 自适应过滤                                                           │
│     • 动态调整 AND 条件顺序                                               │
│     • 将高选择性条件提前执行                                              │
│                                                                         │
│  7. 引用优化                                                             │
│     • 列引用和常量使用 Reference 而非复制                                 │
│     • 最小化内存分配                                                      │
│                                                                         │
│  8. 状态复用                                                             │
│     • ExpressionState 和 intermediate_chunk 预分配                       │
│     • 每次执行只 Reset 不重新分配                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.13 总结

本章深入分析了 DuckDB 的表达式执行器：

| 组件 | 职责 | 关键特性 |
|------|------|----------|
| ExpressionExecutor | 表达式执行入口 | 管理表达式列表和状态 |
| ExpressionState | 表达式执行状态 | 子状态树 + 中间结果 |
| Execute 模式 | 计算表达式值 | 返回 Vector |
| Select 模式 | 评估布尔条件 | 返回 SelectionVector |
| 函数执行 | 调用标量函数 | 字典优化支持 |
| 比较执行 | 比较操作 | 模板化实现 |
| 逻辑连接 | AND/OR 操作 | 短路求值 + 自适应过滤 |
| CASE WHEN | 条件分支 | 增量填充结果 |
| Cast | 类型转换 | 可插拔转换函数 |

---

## 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| ExpressionExecutor | `src/include/duckdb/execution/expression_executor.hpp` |
| ExpressionState | `src/include/duckdb/execution/expression_executor_state.hpp` |
| 常量执行 | `src/execution/expression_executor/execute_constant.cpp` |
| 引用执行 | `src/execution/expression_executor/execute_reference.cpp` |
| 函数执行 | `src/execution/expression_executor/execute_function.cpp` |
| 比较执行 | `src/execution/expression_executor/execute_comparison.cpp` |
| 逻辑连接 | `src/execution/expression_executor/execute_conjunction.cpp` |
| CASE 执行 | `src/execution/expression_executor/execute_case.cpp` |
| Cast 执行 | `src/execution/expression_executor/execute_cast.cpp` |
| 自适应过滤 | `src/include/duckdb/execution/adaptive_filter.hpp` |

---

## 下一章预告

第五章将深入分析 **扫描与过滤算子**，探讨 DuckDB 如何从存储层读取数据、应用列裁剪、执行谓词下推等优化技术。
