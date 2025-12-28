# DuckDB 执行引擎深度解析：第三章 表达式执行器

表达式执行是查询处理的核心环节。无论是投影（SELECT）、过滤（WHERE）、聚合（GROUP BY）还是连接（JOIN），都离不开表达式的计算。DuckDB 的 `ExpressionExecutor` 实现了高效的向量化表达式求值，支持各种表达式类型的批量处理。本章将深入分析表达式执行器的设计与实现。

---

## 3.1 表达式执行器概述

### 3.1.1 ExpressionExecutor 核心结构

```cpp
// src/include/duckdb/execution/expression_executor.hpp

class ExpressionExecutor {
public:
    //! 待执行的表达式列表
    vector<const Expression *> expressions;
    //! 当前输入 DataChunk（用于解析列引用）
    DataChunk *chunk = nullptr;

private:
    //! 客户端上下文
    optional_ptr<ClientContext> context;
    //! 表达式执行状态（每个表达式一个）
    vector<unique_ptr<ExpressionExecutorState>> states;
};
```

**整体架构**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ExpressionExecutor                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  expressions: [expr₀, expr₁, expr₂, ...]                                    │
│       ↓          ↓          ↓                                                │
│  states:    [state₀, state₁, state₂, ...]                                   │
│                                                                              │
│  chunk: DataChunk* ──→ 输入数据（列引用解析）                                 │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │                        Execute() 方法                                   │ │
│  │                                                                         │ │
│  │  for each expression:                                                   │ │
│  │      Execute(expr, state, sel, count, result_vector)                    │ │
│  │          ↓                                                              │ │
│  │      根据表达式类型分派到具体实现                                         │ │
│  │          ↓                                                              │ │
│  │      递归执行子表达式                                                    │ │
│  │          ↓                                                              │ │
│  │      向量化计算                                                          │ │
│  │          ↓                                                              │ │
│  │      结果写入 result_vector                                              │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 3.1.2 表达式类型

DuckDB 支持多种表达式类型，每种都有专门的执行逻辑：

```cpp
enum class ExpressionClass : uint8_t {
    BOUND_REF,          // 列引用: column_a
    BOUND_CONSTANT,     // 常量: 42, 'hello'
    BOUND_FUNCTION,     // 函数调用: UPPER(name), a + b
    BOUND_COMPARISON,   // 比较: a > b, x = y
    BOUND_CONJUNCTION,  // 逻辑连接: AND, OR
    BOUND_CAST,         // 类型转换: CAST(x AS INTEGER)
    BOUND_CASE,         // CASE 表达式
    BOUND_BETWEEN,      // BETWEEN 表达式
    BOUND_OPERATOR,     // 操作符: NOT, IS NULL
    BOUND_PARAMETER,    // 参数: $1, ?
    // ...
};
```

### 3.1.3 表达式状态管理

每个表达式都有对应的执行状态，用于存储中间结果和缓存数据：

```cpp
// src/include/duckdb/execution/expression_executor_state.hpp

struct ExpressionState {
    const Expression &expr;              // 关联的表达式
    ExpressionExecutorState &root;       // 根状态引用
    vector<unique_ptr<ExpressionState>> child_states;  // 子表达式状态
    vector<LogicalType> types;           // 子表达式类型
    DataChunk intermediate_chunk;        // 中间结果存储
};

struct ExpressionExecutorState {
    unique_ptr<ExpressionState> root_state;  // 根表达式状态
    ExpressionExecutor *executor = nullptr;  // 反向引用
};
```

**状态树示例**：

```
表达式: a + b > c AND d = 'hello'

ExpressionExecutorState
└── root_state: ConjunctionState (AND)
    ├── child_states[0]: ExpressionState (>)
    │   ├── child_states[0]: ExpressionState (+)
    │   │   ├── child_states[0]: ExpressionState (a) [列引用]
    │   │   └── child_states[1]: ExpressionState (b) [列引用]
    │   └── child_states[1]: ExpressionState (c) [列引用]
    └── child_states[1]: ExpressionState (=)
        ├── child_states[0]: ExpressionState (d) [列引用]
        └── child_states[1]: ExpressionState ('hello') [常量]
```

---

## 3.2 核心执行流程

### 3.2.1 Execute 入口

```cpp
// src/execution/expression_executor.cpp

void ExpressionExecutor::Execute(DataChunk *input, DataChunk &result) {
    SetChunk(input);  // 设置输入 chunk（用于列引用解析）

    // 为每个表达式执行计算
    for (idx_t i = 0; i < expressions.size(); i++) {
        ExecuteExpression(i, result.data[i]);
    }

    // 设置结果行数
    result.SetCardinality(input ? input->size() : 1);
    result.Verify();
}

void ExpressionExecutor::ExecuteExpression(idx_t expr_idx, Vector &result) {
    // 执行指定索引的表达式
    Execute(*expressions[expr_idx],
            states[expr_idx]->root_state.get(),
            nullptr,                    // sel: 无选择向量
            chunk ? chunk->size() : 1,  // count: 行数
            result);                    // 输出向量
}
```

### 3.2.2 类型分派

核心的 Execute 方法根据表达式类型分派到具体实现：

```cpp
void ExpressionExecutor::Execute(const Expression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    if (count == 0) return;

    // 类型检查
    D_ASSERT(result.GetType().id() == expr.return_type.id());

    // 根据表达式类型分派
    switch (expr.GetExpressionClass()) {
    case ExpressionClass::BOUND_REF:
        Execute(expr.Cast<BoundReferenceExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CONSTANT:
        Execute(expr.Cast<BoundConstantExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_FUNCTION:
        Execute(expr.Cast<BoundFunctionExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_COMPARISON:
        Execute(expr.Cast<BoundComparisonExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CONJUNCTION:
        Execute(expr.Cast<BoundConjunctionExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CASE:
        Execute(expr.Cast<BoundCaseExpression>(), state, sel, count, result);
        break;
    case ExpressionClass::BOUND_CAST:
        Execute(expr.Cast<BoundCastExpression>(), state, sel, count, result);
        break;
    // ... 其他类型
    }

    // 验证结果
    Verify(expr, result, count);
}
```

---

## 3.3 列引用执行 (BoundReferenceExpression)

列引用是最基本的表达式类型，直接从输入 DataChunk 获取指定列：

```cpp
// src/execution/expression_executor/execute_reference.cpp

void ExpressionExecutor::Execute(const BoundReferenceExpression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    D_ASSERT(expr.index < chunk->ColumnCount());

    if (sel) {
        // 有选择向量：切片输入列
        result.Slice(chunk->data[expr.index], *sel, count);
    } else {
        // 无选择向量：直接引用输入列
        result.Reference(chunk->data[expr.index]);
    }
}
```

**执行示例**：

```
输入 DataChunk:
  Column 0 (a): [10, 20, 30, 40, 50]
  Column 1 (b): [1, 2, 3, 4, 5]

表达式: BoundReferenceExpression { index: 0 }  // 引用列 'a'

执行结果 (无 sel):
  result.Reference(chunk->data[0])
  result: [10, 20, 30, 40, 50]  // 零拷贝引用

执行结果 (sel = [1, 3]):
  result.Slice(chunk->data[0], sel, 2)
  result: [20, 40]  // 切片后的数据
```

---

## 3.4 函数执行 (BoundFunctionExpression)

函数表达式是最常见的表达式类型，包括算术运算（+, -, *, /）、字符串函数（UPPER, CONCAT）等。

### 3.4.1 基本执行流程

```cpp
// src/execution/expression_executor/execute_function.cpp

void ExpressionExecutor::Execute(const BoundFunctionExpression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    // 1. 准备参数 DataChunk
    state->intermediate_chunk.Reset();
    auto &arguments = state->intermediate_chunk;

    // 2. 递归执行所有子表达式（参数）
    for (idx_t i = 0; i < expr.children.size(); i++) {
        Execute(*expr.children[i],
                state->child_states[i].get(),
                sel, count,
                arguments.data[i]);
    }
    arguments.SetCardinality(count);

    // 3. 尝试字典向量优化
    auto &execute_function_state = state->Cast<ExecuteFunctionState>();
    if (!execute_function_state.TryExecuteDictionaryExpression(expr, arguments, *state, result)) {
        // 4. 调用实际的函数实现
        expr.function.GetFunctionCallback()(arguments, *state, result);
    }
}
```

### 3.4.2 函数回调接口

DuckDB 的标量函数使用统一的回调接口：

```cpp
// 函数回调签名
typedef void (*scalar_function_t)(DataChunk &args,
                                  ExpressionState &state,
                                  Vector &result);

// 示例：加法函数实现
void AddFunction(DataChunk &args, ExpressionState &state, Vector &result) {
    auto &left = args.data[0];
    auto &right = args.data[1];
    idx_t count = args.size();

    // 向量化加法
    BinaryExecutor::Execute<int64_t, int64_t, int64_t>(
        left, right, result, count,
        [](int64_t a, int64_t b) { return a + b; }
    );
}
```

### 3.4.3 字典向量优化

对于输入是字典向量的情况，可以只对字典执行一次函数，避免重复计算：

```cpp
bool ExecuteFunctionState::TryExecuteDictionaryExpression(
    const BoundFunctionExpression &expr,
    DataChunk &args,
    ExpressionState &state,
    Vector &result) {

    static constexpr idx_t MAX_DICTIONARY_SIZE_THRESHOLD = 20000;

    // 检查是否满足优化条件
    if (!input_col_idx.IsValid()) return false;

    auto &unary_input = args.data[input_col_idx.GetIndex()];
    if (unary_input.GetVectorType() != VectorType::DICTIONARY_VECTOR) {
        return false;
    }

    auto dict_size = DictionaryVector::DictionarySize(unary_input);
    if (dict_size >= MAX_DICTIONARY_SIZE_THRESHOLD) {
        return false;  // 字典太大，不优化
    }

    // 在字典上执行函数（而不是在每个引用上执行）
    if (!output_dictionary || current_input_dictionary_id != input_dictionary_id) {
        // 创建输出字典
        output_dictionary = DictionaryVector::CreateReusableDictionary(
            result.GetType(), dict_size);

        // 对字典执行函数
        Vector output_intermediate(result.GetType());
        expr.function.GetFunctionCallback()(dict_chunk, state, output_intermediate);
        VectorOperations::Copy(output_intermediate, output_dictionary->data, dict_size, 0, 0);
    }

    // 结果引用输出字典
    result.Dictionary(output_dictionary, DictionaryVector::SelVector(unary_input));
    return true;
}
```

**优化效果**：

```
输入（字典向量，来自 Parquet 文件）:
  Dictionary: ["active", "inactive", "pending"]  (3 个唯一值)
  SelVector: [0, 1, 0, 2, 0, 1, 0, 0, 2, ...]    (10000 行)

函数: UPPER(status)

不优化: 执行 10000 次 UPPER()
优化后: 只执行 3 次 UPPER()，然后重用结果

加速比: ~3000x (对于这个例子)
```

---

## 3.5 比较表达式执行 (BoundComparisonExpression)

比较表达式处理各种比较操作符：`=`, `<>`, `<`, `>`, `<=`, `>=`。

### 3.5.1 Execute 模式

```cpp
// src/execution/expression_executor/execute_comparison.cpp

void ExpressionExecutor::Execute(const BoundComparisonExpression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    // 1. 准备中间结果
    state->intermediate_chunk.Reset();
    auto &left = state->intermediate_chunk.data[0];
    auto &right = state->intermediate_chunk.data[1];

    // 2. 递归执行左右子表达式
    Execute(*expr.left, state->child_states[0].get(), sel, count, left);
    Execute(*expr.right, state->child_states[1].get(), sel, count, right);

    // 3. 执行向量化比较
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
    // ...
    }
}
```

### 3.5.2 Select 模式

对于 WHERE 子句中的布尔表达式，使用 Select 模式直接生成选择向量，避免生成中间布尔向量：

```cpp
idx_t ExpressionExecutor::Select(const BoundComparisonExpression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  SelectionVector *true_sel,
                                  SelectionVector *false_sel) {
    // 执行左右子表达式
    state->intermediate_chunk.Reset();
    auto &left = state->intermediate_chunk.data[0];
    auto &right = state->intermediate_chunk.data[1];

    Execute(*expr.left, state->child_states[0].get(), sel, count, left);
    Execute(*expr.right, state->child_states[1].get(), sel, count, right);

    // 直接生成选择向量
    switch (expr.GetExpressionType()) {
    case ExpressionType::COMPARE_EQUAL:
        return VectorOperations::Equals(left, right, sel, count, true_sel, false_sel);
    case ExpressionType::COMPARE_LESSTHAN:
        return VectorOperations::LessThan(left, right, sel, count, true_sel, false_sel);
    // ...
    }
}
```

**Execute vs Select 对比**：

```
表达式: a > 10

Execute 模式:
  1. 计算 a > 10 → 布尔向量 [T, F, T, T, F, ...]
  2. 使用布尔向量过滤数据
  额外开销: 分配和填充布尔向量

Select 模式:
  1. 直接生成 true_sel = [0, 2, 3, ...]
  2. 直接使用选择向量
  优势: 无需中间布尔向量，更高效
```

### 3.5.3 向量化比较实现

```cpp
// 模板化的比较选择操作
template <class OP>
static idx_t TemplatedSelectOperation(Vector &left, Vector &right,
                                       const SelectionVector *sel, idx_t count,
                                       SelectionVector *true_sel,
                                       SelectionVector *false_sel) {
    switch (left.GetType().InternalType()) {
    case PhysicalType::INT32:
        return BinaryExecutor::Select<int32_t, int32_t, OP>(
            left, right, sel, count, true_sel, false_sel);
    case PhysicalType::INT64:
        return BinaryExecutor::Select<int64_t, int64_t, OP>(
            left, right, sel, count, true_sel, false_sel);
    case PhysicalType::VARCHAR:
        return BinaryExecutor::Select<string_t, string_t, OP>(
            left, right, sel, count, true_sel, false_sel);
    // ... 其他类型
    }
}

// BinaryExecutor::Select 实现
template <class LEFT_TYPE, class RIGHT_TYPE, class OP>
static idx_t Select(Vector &left, Vector &right,
                    const SelectionVector *sel, idx_t count,
                    SelectionVector *true_sel, SelectionVector *false_sel) {
    auto ldata = FlatVector::GetData<LEFT_TYPE>(left);
    auto rdata = FlatVector::GetData<RIGHT_TYPE>(right);

    idx_t true_count = 0, false_count = 0;
    for (idx_t i = 0; i < count; i++) {
        auto idx = sel ? sel->get_index(i) : i;
        if (OP::Operation(ldata[idx], rdata[idx])) {
            if (true_sel) true_sel->set_index(true_count++, idx);
        } else {
            if (false_sel) false_sel->set_index(false_count++, idx);
        }
    }
    return true_count;
}
```

---

## 3.6 逻辑连接执行 (BoundConjunctionExpression)

AND 和 OR 逻辑连接需要特殊处理以支持短路求值。

### 3.6.1 AND 表达式执行

```cpp
// src/execution/expression_executor/execute_conjunction.cpp

idx_t ExpressionExecutor::Select(const BoundConjunctionExpression &expr,
                                  ExpressionState *state_p,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  SelectionVector *true_sel,
                                  SelectionVector *false_sel) {
    auto &state = state_p->Cast<ConjunctionState>();

    if (expr.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
        // AND: 依次评估每个条件，只对通过的行继续
        const SelectionVector *current_sel = sel;
        idx_t current_count = count;

        for (idx_t i = 0; i < expr.children.size(); i++) {
            // 使用自适应过滤器决定执行顺序
            auto child_idx = state.adaptive_filter->permutation[i];

            // 只对当前选中的行评估条件
            idx_t tcount = Select(*expr.children[child_idx],
                                  state.child_states[child_idx].get(),
                                  current_sel, current_count,
                                  true_sel,          // 通过的行
                                  temp_false.get()); // 失败的行

            // 收集失败的行到 false_sel
            if (false_sel) {
                for (idx_t j = 0; j < current_count - tcount; j++) {
                    false_sel->set_index(false_count++, temp_false->get_index(j));
                }
            }

            // 下一轮只处理通过的行
            current_count = tcount;
            current_sel = true_sel;

            if (current_count == 0) break;  // 短路：全部失败
        }

        return current_count;
    }
    // OR 类似处理...
}
```

**短路求值可视化**：

```
表达式: a > 10 AND b < 5 AND c = 'active'

输入: 1000 行

第一轮 (a > 10):
  输入: 1000 行
  通过: 300 行 → 继续评估
  失败: 700 行 → 加入 false_sel

第二轮 (b < 5):
  输入: 300 行（只处理通过第一轮的）
  通过: 80 行 → 继续评估
  失败: 220 行 → 加入 false_sel

第三轮 (c = 'active'):
  输入: 80 行（只处理通过前两轮的）
  通过: 50 行 → 最终结果
  失败: 30 行 → 加入 false_sel

最终: true_sel = 50 行, false_sel = 950 行

优化效果:
  不优化: 评估 1000 + 1000 + 1000 = 3000 次
  短路:   评估 1000 + 300 + 80 = 1380 次
  减少:   54%
```

### 3.6.2 自适应过滤器

DuckDB 使用自适应过滤器动态调整条件的执行顺序，将选择性高的条件放在前面：

```cpp
struct ConjunctionState : public ExpressionState {
    unique_ptr<AdaptiveFilter> adaptive_filter;
};

class AdaptiveFilter {
public:
    vector<idx_t> permutation;  // 条件执行顺序

    FilterState BeginFilter() {
        return current_state;
    }

    void EndFilter(FilterState &state) {
        // 根据执行统计调整 permutation
        // 选择性高的条件排在前面
    }
};
```

---

## 3.7 CASE 表达式执行 (BoundCaseExpression)

CASE 表达式需要处理条件分支，向量化执行时需要特别小心。

### 3.7.1 执行流程

```cpp
// src/execution/expression_executor/execute_case.cpp

void ExpressionExecutor::Execute(const BoundCaseExpression &expr,
                                  ExpressionState *state_p,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    auto &state = state_p->Cast<CaseExpressionState>();

    auto current_true_sel = &state.true_sel;
    auto current_false_sel = &state.false_sel;
    auto current_sel = sel;
    idx_t current_count = count;

    // 处理每个 WHEN 子句
    for (idx_t i = 0; i < expr.case_checks.size(); i++) {
        auto &case_check = expr.case_checks[i];

        // 1. 评估 WHEN 条件
        idx_t tcount = Select(*case_check.when_expr,
                              check_state, current_sel, current_count,
                              current_true_sel,
                              current_false_sel);

        if (tcount == 0) {
            // 全部为假，跳过这个分支
            continue;
        }

        idx_t fcount = current_count - tcount;

        if (fcount == 0 && current_count == count) {
            // 全部为真且是第一个分支：直接执行 THEN
            Execute(*case_check.then_expr, then_state, sel, count, result);
            return;
        }

        // 2. 对满足条件的行执行 THEN
        Execute(*case_check.then_expr, then_state,
                current_true_sel, tcount, intermediate_result);

        // 3. 填充结果到正确位置
        FillSwitch(intermediate_result, result, *current_true_sel, tcount);

        // 4. 继续处理不满足条件的行
        current_sel = current_false_sel;
        current_count = fcount;

        if (fcount == 0) break;  // 全部处理完毕
    }

    // 处理 ELSE 分支
    if (current_count > 0) {
        Execute(*expr.else_expr, else_state,
                current_sel, current_count, intermediate_result);
        FillSwitch(intermediate_result, result, *current_sel, current_count);
    }
}
```

### 3.7.2 FillSwitch：结果填充

由于 CASE 表达式的不同分支可能写入结果的不同位置，需要 FillSwitch 函数将分支结果填充到正确位置：

```cpp
void ExpressionExecutor::FillSwitch(Vector &vector, Vector &result,
                                     const SelectionVector &sel, sel_t count) {
    switch (result.GetType().InternalType()) {
    case PhysicalType::INT32:
        TemplatedFillLoop<int32_t>(vector, result, sel, count);
        break;
    case PhysicalType::INT64:
        TemplatedFillLoop<int64_t>(vector, result, sel, count);
        break;
    case PhysicalType::VARCHAR:
        TemplatedFillLoop<string_t>(vector, result, sel, count);
        StringVector::AddHeapReference(result, vector);
        break;
    // ... 其他类型
    }
}

template <class T>
void TemplatedFillLoop(Vector &vector, Vector &result,
                       const SelectionVector &sel, sel_t count) {
    result.SetVectorType(VectorType::FLAT_VECTOR);
    auto res = FlatVector::GetData<T>(result);
    auto &result_mask = FlatVector::Validity(result);

    UnifiedVectorFormat vdata;
    vector.ToUnifiedFormat(count, vdata);
    auto data = UnifiedVectorFormat::GetData<T>(vdata);

    for (idx_t i = 0; i < count; i++) {
        auto source_idx = vdata.sel->get_index(i);
        auto res_idx = sel.get_index(i);  // 目标位置

        res[res_idx] = data[source_idx];
        result_mask.Set(res_idx, vdata.validity.RowIsValid(source_idx));
    }
}
```

**执行示例**：

```
表达式:
  CASE
    WHEN status = 'A' THEN 100
    WHEN status = 'B' THEN 200
    ELSE 0
  END

输入 status: ['A', 'B', 'C', 'A', 'B']
             [ 0    1    2    3    4 ]

第一轮 (status = 'A'):
  true_sel = [0, 3]
  执行 THEN: intermediate = [100, 100]
  FillSwitch: result[0] = 100, result[3] = 100
  false_sel = [1, 2, 4]

第二轮 (status = 'B'):
  输入: sel = [1, 2, 4]
  true_sel = [1, 4]
  执行 THEN: intermediate = [200, 200]
  FillSwitch: result[1] = 200, result[4] = 200
  false_sel = [2]

ELSE 分支:
  输入: sel = [2]
  执行 ELSE: intermediate = [0]
  FillSwitch: result[2] = 0

最终结果: [100, 200, 0, 100, 200]
```

---

## 3.8 常量和参数表达式

### 3.8.1 常量表达式

常量表达式返回一个常量值：

```cpp
// src/execution/expression_executor/execute_constant.cpp

void ExpressionExecutor::Execute(const BoundConstantExpression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    // 直接引用常量值（返回 CONSTANT_VECTOR）
    result.Reference(expr.value);
}
```

### 3.8.2 参数表达式

参数表达式（用于 Prepared Statement）从参数列表获取值：

```cpp
// src/execution/expression_executor/execute_parameter.cpp

void ExpressionExecutor::Execute(const BoundParameterExpression &expr,
                                  ExpressionState *state,
                                  const SelectionVector *sel,
                                  idx_t count,
                                  Vector &result) {
    // 获取参数值
    auto &value = expr.parameter_data->GetValue();
    // 返回常量向量
    result.Reference(value);
}
```

---

## 3.9 SelectExpression：过滤器执行

对于 WHERE 子句中的布尔表达式，ExpressionExecutor 提供了专门的 SelectExpression 接口：

```cpp
// 使用示例
ExpressionExecutor executor(context, where_expr);
SelectionVector true_sel(STANDARD_VECTOR_SIZE);

// 执行过滤，返回满足条件的行数
idx_t result_count = executor.SelectExpression(input_chunk, true_sel);

// true_sel 现在包含满足条件的行索引
// result_count 是满足条件的行数
```

```cpp
// 实现
idx_t ExpressionExecutor::SelectExpression(DataChunk &input,
                                            SelectionVector &sel) {
    D_ASSERT(expressions.size() == 1);
    SetChunk(&input);

    return Select(*expressions[0],
                  states[0]->root_state.get(),
                  nullptr,           // 无输入选择向量
                  input.size(),      // 处理所有行
                  &sel,              // 输出: 满足条件的行
                  nullptr);          // 不需要 false_sel
}
```

---

## 3.10 DefaultSelect：通用布尔过滤

对于没有专门 Select 实现的布尔表达式，使用 DefaultSelect 作为后备：

```cpp
idx_t ExpressionExecutor::DefaultSelect(const Expression &expr,
                                         ExpressionState *state,
                                         const SelectionVector *sel,
                                         idx_t count,
                                         SelectionVector *true_sel,
                                         SelectionVector *false_sel) {
    // 1. 先执行表达式得到布尔向量
    bool intermediate_bools[STANDARD_VECTOR_SIZE];
    Vector intermediate(LogicalType::BOOLEAN, data_ptr_cast(intermediate_bools));
    Execute(expr, state, sel, count, intermediate);

    // 2. 转换为统一格式
    UnifiedVectorFormat idata;
    intermediate.ToUnifiedFormat(count, idata);

    // 3. 遍历布尔向量生成选择向量
    auto bdata = UnifiedVectorFormat::GetData<uint8_t>(idata);
    idx_t true_count = 0, false_count = 0;

    for (idx_t i = 0; i < count; i++) {
        auto bidx = idata.sel->get_index(i);
        auto result_idx = sel ? sel->get_index(i) : i;

        if (idata.validity.RowIsValid(bidx) && bdata[bidx] > 0) {
            if (true_sel) true_sel->set_index(true_count++, result_idx);
        } else {
            if (false_sel) false_sel->set_index(false_count++, result_idx);
        }
    }

    return true_count;
}
```

---

## 3.11 NULL 值处理

表达式执行中的 NULL 值处理遵循 SQL 标准语义：

### 3.11.1 默认 NULL 处理

大多数函数使用默认 NULL 处理：任何参数为 NULL，结果也为 NULL：

```cpp
// 函数执行后验证 NULL 处理
static void VerifyNullHandling(const BoundFunctionExpression &expr,
                               DataChunk &args,
                               Vector &result) {
    if (expr.function.GetNullHandling() != FunctionNullHandling::DEFAULT_NULL_HANDLING) {
        return;
    }

    // 合并所有参数的 validity
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

    // 验证结果：如果任何参数为 NULL，结果必须为 NULL
    UnifiedVectorFormat result_data;
    result.ToUnifiedFormat(count, result_data);
    for (idx_t i = 0; i < count; i++) {
        if (!combined_mask.RowIsValid(i)) {
            D_ASSERT(!result_data.validity.RowIsValid(result_data.sel->get_index(i)));
        }
    }
}
```

### 3.11.2 比较中的 NULL 处理

比较操作中 NULL 的特殊处理：

```
标准比较 (=, <, >, etc.):
  NULL = 1      → NULL (未知)
  NULL = NULL   → NULL (未知)
  NULL < 5      → NULL (未知)

IS DISTINCT FROM:
  NULL IS DISTINCT FROM 1     → TRUE
  NULL IS DISTINCT FROM NULL  → FALSE
  1 IS DISTINCT FROM 2        → TRUE

IS NOT DISTINCT FROM:
  NULL IS NOT DISTINCT FROM NULL → TRUE
  NULL IS NOT DISTINCT FROM 1    → FALSE
```

---

## 3.12 本章小结

本章详细介绍了 DuckDB 的表达式执行器：

1. **类型分派**：ExpressionExecutor 根据表达式类型分派到具体实现，支持多种表达式类型。

2. **两种执行模式**：
   - **Execute 模式**：计算表达式值，返回结果向量
   - **Select 模式**：对于布尔表达式，直接生成选择向量，更高效

3. **递归执行**：表达式树递归执行，子表达式结果存储在 ExpressionState 的 intermediate_chunk 中。

4. **函数执行**：支持字典向量优化，对于来自存储的字典向量只计算一次。

5. **短路求值**：AND/OR 表达式通过选择向量实现短路求值，配合自适应过滤器优化执行顺序。

6. **CASE 表达式**：分支执行后通过 FillSwitch 填充到结果的正确位置。

7. **NULL 处理**：遵循 SQL 标准语义，支持默认 NULL 传播和 DISTINCT FROM 比较。

下一章我们将探讨物理算子的实现，了解这些表达式如何在查询执行的更大上下文中被使用。

---

## 源码参考

| 文件 | 描述 |
|------|------|
| `src/include/duckdb/execution/expression_executor.hpp` | ExpressionExecutor 类定义 |
| `src/execution/expression_executor.cpp` | 核心执行逻辑 |
| `src/include/duckdb/execution/expression_executor_state.hpp` | 表达式状态定义 |
| `src/execution/expression_executor/execute_reference.cpp` | 列引用执行 |
| `src/execution/expression_executor/execute_function.cpp` | 函数执行 |
| `src/execution/expression_executor/execute_comparison.cpp` | 比较执行 |
| `src/execution/expression_executor/execute_conjunction.cpp` | AND/OR 执行 |
| `src/execution/expression_executor/execute_case.cpp` | CASE 执行 |
| `src/execution/expression_executor/execute_constant.cpp` | 常量执行 |
| `src/execution/expression_executor/execute_cast.cpp` | 类型转换执行 |
