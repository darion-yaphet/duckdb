# 第十章：函数执行与优化

## 10.1 概述

函数的执行效率直接影响整体查询性能。DuckDB 在函数执行层面实现了多项优化：向量化执行器管理函数调用状态，表达式重写器进行编译期优化，自适应过滤动态调整条件求值顺序。本章深入分析这些优化机制的实现原理。

```
┌────────────────────────────────────────────────────────────────────────┐
│                     函数执行优化层次                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  编译期优化                              运行期优化                     │
│  ┌────────────────────┐                 ┌─────────────────────┐        │
│  │ ExpressionRewriter │                 │ ExpressionExecutor  │        │
│  ├────────────────────┤                 ├─────────────────────┤        │
│  │ • 常量折叠         │                 │ • 向量化执行        │        │
│  │ • 代数简化         │                 │ • 字典优化          │        │
│  │ • LIKE 优化        │                 │ • 短路求值          │        │
│  │ • 比较优化         │                 │ • 自适应过滤        │        │
│  └────────────────────┘                 └─────────────────────┘        │
│           │                                      │                     │
│           ▼                                      ▼                     │
│  ┌────────────────────┐                 ┌─────────────────────┐        │
│  │   Logical Plan     │ ───────────────▶│   Physical Plan     │        │
│  │   (优化后)          │                 │   (执行)             │        │
│  └────────────────────┘                 └─────────────────────┘        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## 10.2 ExpressionExecutor 核心机制

### 10.2.1 执行器基础结构

ExpressionExecutor 是向量化表达式执行的核心组件：

```cpp
// src/execution/expression_executor.cpp

class ExpressionExecutor {
public:
    // 构造器：支持单表达式和多表达式
    ExpressionExecutor(ClientContext &context);
    ExpressionExecutor(ClientContext &context, const Expression *expression);
    ExpressionExecutor(ClientContext &context, const vector<unique_ptr<Expression>> &exprs);

private:
    optional_ptr<ClientContext> context;           // 客户端上下文
    vector<const Expression *> expressions;       // 待执行表达式列表
    vector<unique_ptr<ExpressionExecutorState>> states;  // 执行状态
    optional_ptr<DataChunk> chunk;                // 当前输入数据块
    bool debug_vector_verification;               // 调试模式标志
};

// 执行器初始化
void ExpressionExecutor::AddExpression(const Expression &expr) {
    expressions.push_back(&expr);
    auto state = make_uniq<ExpressionExecutorState>();
    Initialize(expr, *state);  // 递归初始化状态树
    state->Verify();
    states.push_back(std::move(state));
}

void ExpressionExecutor::Initialize(const Expression &expression,
                                    ExpressionExecutorState &state) {
    state.executor = this;
    state.root_state = InitializeState(expression, state);  // 多态分派
}
```

### 10.2.2 表达式状态管理

```cpp
// src/include/duckdb/execution/expression_executor_state.hpp

struct ExpressionState {
    ExpressionState(const Expression &expr, ExpressionExecutorState &root);
    virtual ~ExpressionState() {}

    const Expression &expr;                      // 关联表达式
    ExpressionExecutorState &root;               // 根状态引用
    vector<unique_ptr<ExpressionState>> child_states;  // 子表达式状态
    vector<LogicalType> types;                   // 子表达式类型
    DataChunk intermediate_chunk;                // 中间结果存储
    vector<bool> initialize;                     // 初始化标记

public:
    void AddChild(Expression &child_expr);       // 添加子状态
    void Finalize();                             // 完成初始化
    ClientContext &GetContext();                 // 获取上下文
};

// 函数执行特化状态
struct ExecuteFunctionState : public ExpressionState {
    ExecuteFunctionState(const Expression &expr, ExpressionExecutorState &root);

    // 函数本地状态（如聚合累加器）
    unique_ptr<FunctionLocalState> local_state;

    // 字典优化相关
    optional_idx input_col_idx;                  // 非常量输入列索引
    buffer_ptr<VectorChildBuffer> output_dictionary;  // 字典结果缓存
    string current_input_dictionary_id;          // 当前字典 ID

    // 尝试字典优化执行
    bool TryExecuteDictionaryExpression(const BoundFunctionExpression &expr,
                                         DataChunk &args, ExpressionState &state,
                                         Vector &result);
};
```

### 10.2.3 执行主流程

```cpp
// 批量执行：输入 DataChunk → 输出 DataChunk
void ExpressionExecutor::Execute(DataChunk *input, DataChunk &result) {
    SetChunk(input);
    D_ASSERT(expressions.size() == result.ColumnCount());

    for (idx_t i = 0; i < expressions.size(); i++) {
        ExecuteExpression(i, result.data[i]);
    }
    result.SetCardinality(input ? input->size() : 1);
    result.Verify();
}

// 单表达式执行
void ExpressionExecutor::ExecuteExpression(idx_t expr_idx, Vector &result) {
    D_ASSERT(expr_idx < expressions.size());
    Execute(*expressions[expr_idx], states[expr_idx]->root_state.get(),
            nullptr, chunk ? chunk->size() : 1, result);
}

// 标量求值：用于常量折叠
Value ExpressionExecutor::EvaluateScalar(ClientContext &context, const Expression &expr,
                                          bool allow_unfoldable) {
    D_ASSERT(allow_unfoldable || expr.IsFoldable());
    D_ASSERT(expr.IsScalar());

    ExpressionExecutor executor(context, expr);
    Vector result(expr.return_type);
    executor.ExecuteExpression(result);

    D_ASSERT(allow_unfoldable || result.GetVectorType() == VectorType::CONSTANT_VECTOR);
    return result.GetValue(0);
}
```

## 10.3 函数执行实现

### 10.3.1 BoundFunctionExpression 执行

```cpp
// src/execution/expression_executor/execute_function.cpp

void ExpressionExecutor::Execute(const BoundFunctionExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    // 1. 重置中间结果块
    state->intermediate_chunk.Reset();
    auto &arguments = state->intermediate_chunk;

    // 2. 递归执行子表达式
    if (!state->types.empty()) {
        for (idx_t i = 0; i < expr.children.size(); i++) {
            D_ASSERT(state->types[i] == expr.children[i]->return_type);
            Execute(*expr.children[i], state->child_states[i].get(),
                    sel, count, arguments.data[i]);
#ifdef DEBUG
            if (expr.children[i]->return_type.id() == LogicalTypeId::VARCHAR) {
                arguments.data[i].UTFVerify(count);  // UTF-8 验证
            }
#endif
        }
    }
    arguments.SetCardinality(count);
    arguments.Verify();

    // 3. 尝试字典优化
    auto &execute_function_state = state->Cast<ExecuteFunctionState>();
    if (!execute_function_state.TryExecuteDictionaryExpression(expr, arguments, *state, result)) {
        // 4. 常规执行
        expr.function.GetFunctionCallback()(arguments, *state, result);
    }

    // 5. 验证 NULL 处理
    VerifyNullHandling(expr, arguments, result);
    D_ASSERT(result.GetType() == expr.return_type);
}
```

### 10.3.2 字典向量优化

当输入是字典向量时，可以只对字典中的唯一值执行函数，然后复用结果：

```cpp
bool ExecuteFunctionState::TryExecuteDictionaryExpression(
    const BoundFunctionExpression &expr, DataChunk &args,
    ExpressionState &state, Vector &result) {

    static constexpr idx_t MAX_DICTIONARY_SIZE_THRESHOLD = 20000;
    static constexpr double CHUNK_FILL_RATIO_THRESHOLD = 0.5;

    if (!input_col_idx.IsValid()) {
        return false;  // 表达式不符合优化条件
    }

    // 检查输入是否为字典向量
    const auto &unary_input = args.data[input_col_idx.GetIndex()];
    if (unary_input.GetVectorType() != VectorType::DICTIONARY_VECTOR) {
        return false;
    }

    const auto input_dictionary_size_opt = DictionaryVector::DictionarySize(unary_input);
    if (!input_dictionary_size_opt.IsValid()) {
        return false;  // 非存储来源的字典
    }

    const auto input_dictionary_size = input_dictionary_size_opt.GetIndex();
    if (input_dictionary_size >= MAX_DICTIONARY_SIZE_THRESHOLD) {
        return false;  // 字典太大，跳过优化
    }

    // 首次看到此字典或字典已变化
    if (!output_dictionary || current_input_dictionary_id != input_dictionary_id) {
        // 检查 chunk 填充率
        const auto chunk_fill_ratio = static_cast<double>(args.size()) / STANDARD_VECTOR_SIZE;
        if (input_dictionary_size > STANDARD_VECTOR_SIZE &&
            chunk_fill_ratio <= CHUNK_FILL_RATIO_THRESHOLD) {
            return false;  // 填充率低，不值得优化
        }

        // 执行优化：对字典值执行函数
        output_dictionary = DictionaryVector::CreateReusableDictionary(
            result.GetType(), input_dictionary_size);
        current_input_dictionary_id = input_dictionary_id;

        // 分批处理字典值
        DataChunk input_chunk;
        input_chunk.InitializeEmpty(args.GetTypes());
        for (idx_t offset = 0; offset < input_dictionary_size;
             offset += STANDARD_VECTOR_SIZE) {
            const auto count = MinValue<idx_t>(
                input_dictionary_size - offset, STANDARD_VECTOR_SIZE);

            Vector offset_input(DictionaryVector::Child(unary_input), offset, offset + count);
            input_chunk.data[input_col_idx.GetIndex()].Reference(offset_input);
            input_chunk.SetCardinality(count);

            Vector output_intermediate(result.GetType());
            expr.function.GetFunctionCallback()(input_chunk, state, output_intermediate);
            VectorOperations::Copy(output_intermediate, output_dictionary->data,
                                   count, 0, offset);
        }
    }

    // 结果引用字典
    result.Dictionary(output_dictionary, DictionaryVector::SelVector(unary_input));
    return true;
}
```

```
┌───────────────────────────────────────────────────────────────────┐
│                    字典向量优化示意图                              │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│  输入: DICTIONARY_VECTOR                                          │
│  ┌─────────────┐    ┌──────────────────────────┐                 │
│  │ Selection   │    │ Dictionary (唯一值)       │                 │
│  │ [2,0,1,0,2] │ ─▶ │ ["apple","banana","cherry"]│                │
│  └─────────────┘    └──────────────────────────┘                 │
│        │                        │                                 │
│        │                        ▼                                 │
│        │              ┌──────────────────────────┐                │
│        │              │ UPPER() 执行 (只执行3次) │                │
│        │              │ ["APPLE","BANANA","CHERRY"]│               │
│        │              └──────────────────────────┘                │
│        │                        │                                 │
│        ▼                        ▼                                 │
│  ┌─────────────────────────────────────────────┐                  │
│  │ 结果: DICTIONARY_VECTOR                       │                  │
│  │ Selection: [2,0,1,0,2]                       │                  │
│  │ Dictionary: ["APPLE","BANANA","CHERRY"]       │                  │
│  │                                               │                  │
│  │ 实际值: ["CHERRY","APPLE","BANANA","APPLE","CHERRY"]│           │
│  └─────────────────────────────────────────────┘                  │
│                                                                   │
│  优化效果: 原本执行5次 → 只执行3次                                 │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

### 10.3.3 NULL 处理验证

```cpp
static void VerifyNullHandling(const BoundFunctionExpression &expr, DataChunk &args,
                               Vector &result) {
#ifdef DEBUG
    if (args.data.empty() ||
        expr.function.GetNullHandling() != FunctionNullHandling::DEFAULT_NULL_HANDLING) {
        return;
    }

    // 组合所有参数的有效性掩码
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

    // 默认 NULL 处理：任一参数为 NULL，结果必须为 NULL
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

## 10.4 逻辑运算与短路求值

### 10.4.1 AND/OR 执行

```cpp
// src/execution/expression_executor/execute_conjunction.cpp

void ExpressionExecutor::Execute(const BoundConjunctionExpression &expr, ExpressionState *state,
                                 const SelectionVector *sel, idx_t count, Vector &result) {
    state->intermediate_chunk.Reset();

    for (idx_t i = 0; i < expr.children.size(); i++) {
        auto &current_result = state->intermediate_chunk.data[i];
        Execute(*expr.children[i], state->child_states[i].get(), sel, count, current_result);

        if (i == 0) {
            result.Reference(current_result);
        } else {
            Vector intermediate(LogicalType::BOOLEAN);
            switch (expr.GetExpressionType()) {
            case ExpressionType::CONJUNCTION_AND:
                VectorOperations::And(current_result, result, intermediate, count);
                break;
            case ExpressionType::CONJUNCTION_OR:
                VectorOperations::Or(current_result, result, intermediate, count);
                break;
            }
            result.Reference(intermediate);
        }
    }
}
```

### 10.4.2 选择执行（短路优化）

```cpp
idx_t ExpressionExecutor::Select(const BoundConjunctionExpression &expr,
                                  ExpressionState *state_p, const SelectionVector *sel,
                                  idx_t count, SelectionVector *true_sel,
                                  SelectionVector *false_sel) {
    auto &state = state_p->Cast<ConjunctionState>();

    if (expr.GetExpressionType() == ExpressionType::CONJUNCTION_AND) {
        // AND: 自适应过滤 + 短路求值
        auto filter_state = state.adaptive_filter->BeginFilter();
        const SelectionVector *current_sel = sel;
        idx_t current_count = count;
        idx_t false_count = 0;

        unique_ptr<SelectionVector> temp_true, temp_false;
        if (false_sel) temp_false = make_uniq<SelectionVector>(STANDARD_VECTOR_SIZE);
        if (!true_sel) {
            temp_true = make_uniq<SelectionVector>(STANDARD_VECTOR_SIZE);
            true_sel = temp_true.get();
        }

        // 按自适应排列顺序执行条件
        for (idx_t i = 0; i < expr.children.size(); i++) {
            idx_t child_idx = state.adaptive_filter->permutation[i];
            idx_t tcount = Select(*expr.children[child_idx],
                                  state.child_states[child_idx].get(),
                                  current_sel, current_count,
                                  true_sel, temp_false.get());
            idx_t fcount = current_count - tcount;

            if (fcount > 0 && false_sel) {
                // 收集失败元组
                for (idx_t j = 0; j < fcount; j++) {
                    false_sel->set_index(false_count++, temp_false->get_index(j));
                }
            }

            current_count = tcount;
            if (current_count == 0) {
                break;  // 短路：没有通过的元组了
            }
            if (current_count < count) {
                // 后续只评估通过的元组
                current_sel = true_sel;
            }
        }

        state.adaptive_filter->EndFilter(filter_state);
        return current_count;
    }
    // OR 类似但逻辑相反...
}
```

## 10.5 自适应过滤

### 10.5.1 AdaptiveFilter 结构

```cpp
// src/include/duckdb/execution/adaptive_filter.hpp

class AdaptiveFilter {
public:
    explicit AdaptiveFilter(const Expression &expr);
    explicit AdaptiveFilter(const TableFilterSet &table_filters);

    vector<idx_t> permutation;  // 条件执行顺序

public:
    void AdaptRuntimeStatistics(double duration);  // 更新统计
    AdaptiveFilterState BeginFilter() const;       // 开始计时
    void EndFilter(AdaptiveFilterState state);     // 结束计时并自适应

private:
    bool disable_permutations = false;

    // 自适应重排参数
    idx_t iteration_count = 0;     // 迭代计数
    idx_t swap_idx = 0;            // 当前交换位置
    idx_t right_random_border = 0; // 随机探索边界
    idx_t observe_interval = 0;    // 观察间隔
    idx_t execute_interval = 0;    // 执行间隔
    double runtime_sum = 0;        // 运行时间累计
    double prev_mean = 0;          // 前一次平均时间
    bool observe = false;          // 是否处于观察模式
    bool warmup = false;           // 是否处于预热阶段
    vector<idx_t> swap_likeliness; // 交换可能性
    RandomEngine generator;        // 随机数生成器
};
```

### 10.5.2 自适应策略

```
┌────────────────────────────────────────────────────────────────────────┐
│                    自适应过滤策略                                       │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  初始状态:                                                              │
│  conditions = [C1, C2, C3]   (按出现顺序)                               │
│  permutation = [0, 1, 2]                                               │
│                                                                        │
│  自适应过程:                                                            │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ 1. 预热阶段 (warmup)                                        │       │
│  │    - 执行多次，收集基准统计                                  │       │
│  │                                                             │       │
│  │ 2. 探索阶段 (observe)                                       │       │
│  │    - 随机交换两个条件的位置                                  │       │
│  │    - 测量新顺序的执行时间                                    │       │
│  │    - 如果更快，保留新顺序                                    │       │
│  │    - 如果更慢，恢复原顺序                                    │       │
│  │                                                             │       │
│  │ 3. 利用阶段 (execute)                                       │       │
│  │    - 使用当前最优顺序执行                                    │       │
│  │    - 周期性回到探索阶段                                      │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                        │
│  优化原理:                                                              │
│  - 高选择性条件放前面 → 更早短路，减少计算                              │
│  - 低成本条件放前面 → 快速过滤大量数据                                  │
│  - 动态适应数据分布变化                                                 │
│                                                                        │
│  示例:                                                                  │
│  原始: WHERE a > 100 AND b = 'x' AND c < 10                             │
│  如果 b = 'x' 过滤掉 99% 数据，自适应后:                                 │
│  优化: WHERE b = 'x' AND a > 100 AND c < 10                             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## 10.6 编译期优化规则

### 10.6.1 常量折叠

```cpp
// src/optimizer/rule/constant_folding.cpp

class ConstantFoldingExpressionMatcher : public FoldableConstantMatcher {
public:
    bool Match(Expression &expr, vector<reference<Expression>> &bindings) override {
        // 不匹配已经是常量的表达式
        if (expr.GetExpressionType() == ExpressionType::VALUE_CONSTANT) {
            return false;
        }
        return FoldableConstantMatcher::Match(expr, bindings);
    }
};

unique_ptr<Expression> ConstantFoldingRule::Apply(LogicalOperator &op,
                                                   vector<reference<Expression>> &bindings,
                                                   bool &changes_made, bool is_root) {
    auto &root = bindings[0].get();
    D_ASSERT(root.IsFoldable() && root.GetExpressionType() != ExpressionType::VALUE_CONSTANT);

    // 使用 ExpressionExecutor 计算常量值
    Value result_value;
    if (!ExpressionExecutor::TryEvaluateScalar(GetContext(), root, result_value)) {
        return nullptr;  // 计算失败（如除零）
    }

    // 替换为常量表达式
    return make_uniq<BoundConstantExpression>(result_value);
}
```

**示例：**
```sql
-- 优化前
SELECT x + 1 + 2 FROM t

-- 优化后（常量折叠）
SELECT x + 3 FROM t
```

### 10.6.2 代数简化

```cpp
// src/optimizer/rule/arithmetic_simplification.cpp

ArithmeticSimplificationRule::ArithmeticSimplificationRule(ExpressionRewriter &rewriter)
    : Rule(rewriter) {
    auto op = make_uniq<FunctionExpressionMatcher>();
    op->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    op->matchers.push_back(make_uniq<ExpressionMatcher>());
    op->policy = SetMatcher::Policy::SOME;
    op->function = make_uniq<ManyFunctionMatcher>(
        unordered_set<string> {"+", "-", "*", "//"});
    op->type = make_uniq<IntegerTypeMatcher>();
    root = std::move(op);
}

unique_ptr<Expression> ArithmeticSimplificationRule::Apply(...) {
    auto &root = bindings[0].get().Cast<BoundFunctionExpression>();
    auto &constant = bindings[1].get().Cast<BoundConstantExpression>();
    idx_t constant_child = root.children[0].get() == &constant ? 0 : 1;

    // NULL 参与运算结果为 NULL
    if (constant.value.IsNull()) {
        return make_uniq<BoundConstantExpression>(Value(root.return_type));
    }

    auto &func_name = root.function.name;
    if (func_name == "+") {
        if (constant.value == 0) {
            // x + 0 → x
            return std::move(root.children[1 - constant_child]);
        }
    } else if (func_name == "-") {
        if (constant_child == 1 && constant.value == 0) {
            // x - 0 → x
            return std::move(root.children[1 - constant_child]);
        }
    } else if (func_name == "*") {
        if (constant.value == 1) {
            // x * 1 → x
            return std::move(root.children[1 - constant_child]);
        } else if (constant.value == 0) {
            // x * 0 → 0 (保留 NULL 传播)
            return ExpressionRewriter::ConstantOrNull(
                std::move(root.children[1 - constant_child]),
                Value::Numeric(root.return_type, 0));
        }
    } else if (func_name == "//") {
        if (constant_child == 1) {
            if (constant.value == 1) {
                // x / 1 → x
                return std::move(root.children[1 - constant_child]);
            } else if (constant.value == 0) {
                // x / 0 → NULL
                return make_uniq<BoundConstantExpression>(Value(root.return_type));
            }
        }
    }
    return nullptr;
}
```

### 10.6.3 LIKE 模式优化

```cpp
// src/optimizer/rule/like_optimizations.cpp

LikeOptimizationRule::LikeOptimizationRule(ExpressionRewriter &rewriter) : Rule(rewriter) {
    auto func = make_uniq<FunctionExpressionMatcher>();
    func->matchers.push_back(make_uniq<ExpressionMatcher>());
    func->matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    func->policy = SetMatcher::Policy::ORDERED;
    func->function = make_uniq<ManyFunctionMatcher>(unordered_set<string> {"!~~", "~~"});
    root = std::move(func);
}

// 模式类型检测
static bool PatternIsConstant(const string &pattern) {
    // 没有 % 或 _，是精确匹配
    for (idx_t i = 0; i < pattern.size(); i++) {
        if (pattern[i] == '%' || pattern[i] == '_') {
            return false;
        }
    }
    return true;
}

static bool PatternIsPrefix(const string &pattern) {
    // 只有尾部 %，是前缀匹配
    // "abc%" → prefix_match
}

static bool PatternIsSuffix(const string &pattern) {
    // 只有头部 %，是后缀匹配
    // "%abc" → suffix_match
}

static bool PatternIsContains(const string &pattern) {
    // 头尾都有 %，是包含匹配
    // "%abc%" → contains
}
```

**LIKE 优化转换：**
```sql
-- 原始
WHERE name LIKE 'John'     → WHERE name = 'John'       (精确匹配)
WHERE name LIKE 'John%'    → WHERE prefix(name, 'John') (前缀)
WHERE name LIKE '%Smith'   → WHERE suffix(name, 'Smith') (后缀)
WHERE name LIKE '%ohn%'    → WHERE contains(name, 'ohn') (包含)
```

## 10.7 优化规则一览

| 规则名称 | 优化类型 | 示例 |
|---------|---------|------|
| ConstantFoldingRule | 常量折叠 | `1 + 2` → `3` |
| ArithmeticSimplificationRule | 代数简化 | `x * 1` → `x` |
| LikeOptimizationRule | LIKE 优化 | `LIKE 'abc'` → `= 'abc'` |
| ComparisonSimplificationRule | 比较简化 | 范围合并 |
| ConjunctionSimplificationRule | 逻辑简化 | `x AND TRUE` → `x` |
| CaseSimplificationRule | CASE 简化 | 常量条件消除 |
| DatePartSimplificationRule | 日期函数优化 | 特定日期部分提取 |
| DateTruncSimplificationRule | 日期截断优化 | 精度优化 |
| MoveConstantsRule | 常量移动 | `x + 1 = 3` → `x = 2` |
| EmptyNeedleRemovalRule | 空模式移除 | `LIKE ''` 优化 |
| EnumComparisonRule | 枚举比较优化 | 整数比较替代 |
| InClauseSimplificationRule | IN 子句优化 | 值列表优化 |
| DistributivityRule | 分配律应用 | 逻辑表达式重组 |

## 10.8 统计信息传播

### 10.8.1 函数统计回调

```cpp
// 函数可以定义 statistics 回调来传播统计信息
typedef unique_ptr<BaseStatistics> (*function_statistics_t)(
    ClientContext &context,
    FunctionStatisticsInput &input);

struct FunctionStatisticsInput {
    optional_ptr<FunctionData> bind_data;    // 绑定数据
    optional_ptr<ExpressionState> expr_state; // 执行状态
    vector<BaseStatistics> &child_stats;     // 子表达式统计
};
```

**示例：ABS 函数统计传播**
```cpp
unique_ptr<BaseStatistics> AbsStatistics(ClientContext &context,
                                          FunctionStatisticsInput &input) {
    auto &child_stats = input.child_stats[0];
    if (!NumericStats::HasMinMax(child_stats)) {
        return nullptr;
    }

    auto min = NumericStats::Min<int64_t>(child_stats);
    auto max = NumericStats::Max<int64_t>(child_stats);

    // ABS 后的范围
    int64_t new_min = std::min(std::abs(min), std::abs(max));
    int64_t new_max = std::max(std::abs(min), std::abs(max));
    if (min < 0 && max > 0) {
        new_min = 0;  // 跨零点
    }

    auto result = child_stats.Copy();
    NumericStats::SetMin(result, new_min);
    NumericStats::SetMax(result, new_max);
    return make_uniq<BaseStatistics>(std::move(result));
}
```

## 10.9 并行执行考虑

### 10.9.1 线程安全设计

```cpp
// 函数执行需要考虑的线程安全因素
struct FunctionData {
    // 绑定时创建，执行期只读 → 线程安全
    virtual unique_ptr<FunctionData> Copy() const = 0;
};

struct FunctionLocalState {
    // 每个线程独立创建 → 无需同步
    virtual ~FunctionLocalState() = default;
};

// 聚合函数的并行模型
class AggregateFunction {
    aggregate_combine_t combine;  // 合并来自不同线程的状态
    // 每个线程维护独立状态，最后通过 combine 合并
};
```

### 10.9.2 状态隔离策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    并行执行状态隔离                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Thread 1              Thread 2              Thread 3           │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐   │
│  │LocalState 1 │       │LocalState 2 │       │LocalState 3 │   │
│  │ • 累加器    │       │ • 累加器    │       │ • 累加器    │   │
│  │ • 计数器    │       │ • 计数器    │       │ • 计数器    │   │
│  └──────┬──────┘       └──────┬──────┘       └──────┬──────┘   │
│         │                     │                     │           │
│         └─────────────────────┼─────────────────────┘           │
│                               │                                 │
│                               ▼                                 │
│                      ┌─────────────────┐                        │
│                      │    Combine      │                        │
│                      │  (状态合并)      │                        │
│                      └────────┬────────┘                        │
│                               │                                 │
│                               ▼                                 │
│                      ┌─────────────────┐                        │
│                      │   Finalize      │                        │
│                      │  (最终结果)      │                        │
│                      └─────────────────┘                        │
│                                                                 │
│  保证：                                                          │
│  • LocalState 无竞争访问                                        │
│  • Combine 串行执行或使用锁                                      │
│  • FunctionData 只读共享                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 10.10 小结

本章详细介绍了 DuckDB 函数执行与优化的核心机制：

1. **ExpressionExecutor 架构**：
   - 状态树管理（ExpressionState 层次）
   - 向量化批处理执行
   - 递归子表达式求值

2. **运行期优化**：
   - 字典向量优化：减少重复计算
   - 短路求值：AND/OR 提前终止
   - 自适应过滤：动态调整条件顺序

3. **编译期优化规则**：
   - 常量折叠：编译期计算常量表达式
   - 代数简化：消除恒等运算
   - 模式优化：LIKE 转换为更高效函数

4. **并行执行支持**：
   - 线程本地状态隔离
   - 状态合并（Combine）协议
   - 只读共享绑定数据

这些优化共同作用，使 DuckDB 能够高效执行复杂的表达式计算，充分发挥向量化处理的性能优势。
