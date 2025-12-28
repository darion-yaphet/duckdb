# DuckDB 索引系统深度解析 - 第六章：索引在查询中的应用

本章深入分析 DuckDB 索引系统在查询执行中的应用，包括索引扫描初始化、迭代器实现、范围查询优化、物理计划生成以及索引持久化机制。

## 6.1 索引扫描状态

### 6.1.1 ARTIndexScanState 定义

```cpp
// src/execution/index/art/art.cpp
struct ARTIndexScanState : public IndexScanState {
    //! 扫描谓词值
    //! 单个谓词用于点查询，两个谓词用于范围扫描
    Value values[2];

    //! 谓词表达式类型
    ExpressionType expressions[2];

    //! 是否已检查（用于缓存）
    bool checked = false;

    //! 所有扫描到的行ID
    set<row_t> row_ids;
};
```

### 6.1.2 扫描状态初始化

```cpp
// 单谓词初始化（等值或单边范围）
static unique_ptr<IndexScanState> InitializeScanSinglePredicate(
    const Value &value, const ExpressionType expression_type) {
    auto result = make_uniq<ARTIndexScanState>();
    result->values[0] = value;
    result->expressions[0] = expression_type;
    return std::move(result);
}

// 双谓词初始化（双边范围）
static unique_ptr<IndexScanState> InitializeScanTwoPredicates(
    const Value &low_value, const ExpressionType low_expression_type,
    const Value &high_value, const ExpressionType high_expression_type) {
    auto result = make_uniq<ARTIndexScanState>();
    result->values[0] = low_value;
    result->expressions[0] = low_expression_type;
    result->values[1] = high_value;
    result->expressions[1] = high_expression_type;
    return std::move(result);
}
```

---

## 6.2 索引扫描匹配

### 6.2.1 TryInitializeScan 方法

`TryInitializeScan` 尝试将过滤表达式匹配到索引扫描：

```cpp
unique_ptr<IndexScanState> ART::TryInitializeScan(
    const Expression &expr, const Expression &filter_expr) {

    Value low_value, high_value, equal_value;
    ExpressionType low_comparison_type = ExpressionType::INVALID;
    ExpressionType high_comparison_type = ExpressionType::INVALID;

    // 创建比较表达式匹配器
    ComparisonExpressionMatcher matcher;
    matcher.expr_type = make_uniq<ComparisonExpressionTypeMatcher>();
    matcher.matchers.push_back(make_uniq<ExpressionEqualityMatcher>(expr));
    matcher.matchers.push_back(make_uniq<ConstantExpressionMatcher>());
    matcher.policy = SetMatcher::Policy::UNORDERED;

    vector<reference<Expression>> bindings;
    auto filter_match = matcher.Match(
        const_cast<Expression &>(filter_expr), bindings);

    if (filter_match) {
        // 匹配成功: bindings[0]=表达式, bindings[1]=索引列, bindings[2]=常量
        auto &comparison = bindings[0].get().Cast<BoundComparisonExpression>();
        auto constant_value = bindings[2].get().Cast<BoundConstantExpression>().value;
        auto comparison_type = comparison.GetExpressionType();

        // 如果常量在左边，翻转比较类型
        if (comparison.left->GetExpressionType() == ExpressionType::VALUE_CONSTANT) {
            comparison_type = FlipComparisonExpression(comparison_type);
        }

        // 根据比较类型分类
        if (comparison_type == ExpressionType::COMPARE_EQUAL) {
            equal_value = constant_value;
        } else if (comparison_type == ExpressionType::COMPARE_GREATERTHANOREQUALTO ||
                   comparison_type == ExpressionType::COMPARE_GREATERTHAN) {
            low_value = constant_value;
            low_comparison_type = comparison_type;
        } else {
            high_value = constant_value;
            high_comparison_type = comparison_type;
        }
    } else if (filter_expr.GetExpressionType() == ExpressionType::COMPARE_BETWEEN) {
        // 处理 BETWEEN 表达式
        auto &between = filter_expr.Cast<BoundBetweenExpression>();
        if (!between.input->Equals(expr)) {
            return nullptr;
        }
        // ... 提取上下界 ...
    }

    // 无法使用索引
    if (equal_value.IsNull() && low_value.IsNull() && high_value.IsNull()) {
        return nullptr;
    }

    // 根据谓词类型初始化扫描状态
    if (!equal_value.IsNull()) {
        return InitializeScanSinglePredicate(equal_value, ExpressionType::COMPARE_EQUAL);
    }
    if (!low_value.IsNull() && !high_value.IsNull()) {
        return InitializeScanTwoPredicates(low_value, low_comparison_type,
                                           high_value, high_comparison_type);
    }
    // 单边范围...
}
```

### 6.2.2 支持的谓词类型

```
┌─────────────────────────────────────────────────────────────────┐
│ 谓词类型                │ 索引扫描方法                          │
├─────────────────────────┼───────────────────────────────────────┤
│ column = value          │ SearchEqual                           │
│ column > value          │ SearchGreater (equal=false)           │
│ column >= value         │ SearchGreater (equal=true)            │
│ column < value          │ SearchLess (equal=false)              │
│ column <= value         │ SearchLess (equal=true)               │
│ column BETWEEN a AND b  │ SearchCloseRange                      │
│ a <= column AND column <= b │ SearchCloseRange                  │
└─────────────────────────┴───────────────────────────────────────┘
```

---

## 6.3 Iterator 迭代器

### 6.3.1 Iterator 类设计

```cpp
// src/include/duckdb/execution/index/art/iterator.hpp
class Iterator {
public:
    explicit Iterator(ART &art) : art(art), status(GateStatus::GATE_NOT_SET) {}

    //! 扫描范围内的行ID
    bool Scan(const ARTKey &upper_bound, const idx_t max_count,
              set<row_t> &row_ids, const bool equal);

    //! 查找最小值
    void FindMinimum(const Node &node);

    //! 查找下界（大于等于指定键）
    bool LowerBound(const Node &node, const ARTKey &key, const bool equal);

    //! 前进到下一个叶子
    bool Next();

    //! 当前键
    IteratorKey current_key;

private:
    ART &art;
    GateStatus status;
    Node last_leaf;              // 当前叶子节点
    stack<IteratorEntry> nodes;  // 遍历栈
    bool entered_nested_leaf = false;
    idx_t nested_depth = 0;
    uint8_t row_id[Prefix::ROW_ID_SIZE];

    void PopNode();
};
```

### 6.3.2 FindMinimum 查找最小值

```cpp
void Iterator::FindMinimum(const Node &node) {
    reference<const Node> ref(node);

    while (ref.get().HasMetadata()) {
        // 到达叶子节点
        if (ref.get().IsAnyLeaf()) {
            last_leaf = ref.get();
            return;
        }

        // 处理门控节点
        if (ref.get().GetGateStatus() == GateStatus::GATE_SET) {
            status = GateStatus::GATE_SET;
            entered_nested_leaf = true;
            nested_depth = 0;
        }

        // 遍历前缀
        if (ref.get().GetType() == NType::PREFIX) {
            Prefix prefix(art, ref.get());
            for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
                current_key.Push(prefix.data[i]);
                if (status == GateStatus::GATE_SET) {
                    row_id[nested_depth++] = prefix.data[i];
                }
            }
            nodes.emplace(ref.get(), 0);
            ref = *prefix.ptr;
            continue;
        }

        // 获取最左子节点
        uint8_t byte = 0;
        auto next = ref.get().GetNextChild(art, byte);
        D_ASSERT(next);

        current_key.Push(byte);
        if (status == GateStatus::GATE_SET) {
            row_id[nested_depth++] = byte;
        }
        nodes.emplace(ref.get(), byte);
        ref = *next;
    }
}
```

### 6.3.3 LowerBound 下界查找

```cpp
bool Iterator::LowerBound(const Node &node, const ARTKey &key, const bool equal) {
    reference<const Node> ref(node);
    idx_t depth = 0;

    while (ref.get().HasMetadata()) {
        // 到达叶子或门控节点
        if (ref.get().IsAnyLeaf() || ref.get().GetGateStatus() == GateStatus::GATE_SET) {
            if (!equal && current_key.Contains(key)) {
                return Next();  // 键相等但不包含等于
            }
            if (ref.get().GetGateStatus() == GateStatus::GATE_SET) {
                FindMinimum(ref.get());
            } else {
                last_leaf = ref.get();
            }
            return true;
        }

        if (ref.get().GetType() != NType::PREFIX) {
            auto next_byte = key[depth];
            auto child = ref.get().GetNextChild(art, next_byte);

            if (!child) {
                return Next();  // 无匹配子节点
            }

            current_key.Push(next_byte);
            nodes.emplace(ref.get(), next_byte);

            // 找到的子节点大于目标键
            if (next_byte > key[depth]) {
                FindMinimum(*child);
                return true;
            }

            ref = *child;
            depth++;
            continue;
        }

        // 处理前缀
        Prefix prefix(art, ref.get());
        for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
            current_key.Push(prefix.data[i]);
        }
        nodes.emplace(ref.get(), 0);

        // 比较前缀与键
        for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
            if (prefix.data[i] < key[depth + i]) {
                return Next();  // 前缀小于键，需要下一个
            }
            if (prefix.data[i] > key[depth + i]) {
                FindMinimum(*prefix.ptr);  // 前缀大于键，找最小
                return true;
            }
        }

        depth += prefix.data[Prefix::Count(art)];
        ref = *prefix.ptr;
    }
}
```

### 6.3.4 Scan 扫描方法

```cpp
bool Iterator::Scan(const ARTKey &upper_bound, const idx_t max_count,
                    set<row_t> &row_ids, const bool equal) {
    bool has_next;
    do {
        // 检查上界
        if (!upper_bound.Empty()) {
            if (status == GateStatus::GATE_NOT_SET || entered_nested_leaf) {
                if (current_key.GreaterThan(upper_bound, equal, nested_depth)) {
                    return true;  // 超过上界
                }
            }
        }

        // 根据叶子类型提取行ID
        switch (last_leaf.GetType()) {
        case NType::LEAF_INLINED:
            if (row_ids.size() + 1 > max_count) {
                return false;  // 达到最大数量
            }
            row_ids.insert(last_leaf.GetRowId());
            break;

        case NType::LEAF:
            if (!Leaf::DeprecatedGetRowIds(art, last_leaf, row_ids, max_count)) {
                return false;
            }
            break;

        case NType::NODE_7_LEAF:
        case NType::NODE_15_LEAF:
        case NType::NODE_256_LEAF: {
            uint8_t byte = 0;
            while (last_leaf.GetNextByte(art, byte)) {
                if (row_ids.size() + 1 > max_count) {
                    return false;
                }
                row_id[ROW_ID_SIZE - 1] = byte;
                ARTKey key(&row_id[0], ROW_ID_SIZE);
                row_ids.insert(key.GetRowId());
                if (byte == NumericLimits<uint8_t>::Maximum()) {
                    break;
                }
                byte++;
            }
            break;
        }
        }

        entered_nested_leaf = false;
        has_next = Next();
    } while (has_next);
    return true;
}
```

---

## 6.4 范围查询实现

### 6.4.1 等值查询（SearchEqual）

```cpp
bool ART::SearchEqual(ARTKey &key, idx_t max_count, set<row_t> &row_ids) {
    // 精确查找键
    auto leaf = ARTOperator::Lookup(*this, tree, key, 0);
    if (!leaf) {
        return true;  // 未找到
    }

    // 从叶子开始扫描
    Iterator it(*this);
    it.FindMinimum(*leaf);
    ARTKey empty_key = ARTKey();
    return it.Scan(empty_key, max_count, row_ids, false);
}
```

### 6.4.2 大于查询（SearchGreater）

```cpp
bool ART::SearchGreater(ARTKey &key, bool equal, idx_t max_count, set<row_t> &row_ids) {
    if (!tree.HasMetadata()) {
        return true;
    }

    Iterator it(*this);

    // 找到满足条件的最小值
    if (!it.LowerBound(tree, key, equal)) {
        return true;  // 无满足条件的值
    }

    // 继续扫描（无上界）
    return it.Scan(ARTKey(), max_count, row_ids, false);
}
```

### 6.4.3 小于查询（SearchLess）

```cpp
bool ART::SearchLess(ARTKey &upper_bound, bool equal, idx_t max_count, set<row_t> &row_ids) {
    if (!tree.HasMetadata()) {
        return true;
    }

    Iterator it(*this);

    // 从最小值开始
    it.FindMinimum(tree);

    // 检查最小值是否已超过上界
    if (it.current_key.GreaterThan(upper_bound, equal, it.GetNestedDepth())) {
        return true;
    }

    // 扫描到上界
    return it.Scan(upper_bound, max_count, row_ids, equal);
}
```

### 6.4.4 范围查询（SearchCloseRange）

```cpp
bool ART::SearchCloseRange(ARTKey &lower_bound, ARTKey &upper_bound,
                           bool left_equal, bool right_equal,
                           idx_t max_count, set<row_t> &row_ids) {
    Iterator it(*this);

    // 找到满足下界的最小值
    if (!it.LowerBound(tree, lower_bound, left_equal)) {
        return true;
    }

    // 扫描到上界
    return it.Scan(upper_bound, max_count, row_ids, right_equal);
}
```

### 6.4.5 统一扫描接口

```cpp
bool ART::Scan(IndexScanState &state, const idx_t max_count, set<row_t> &row_ids) {
    auto &scan_state = state.Cast<ARTIndexScanState>();
    ArenaAllocator arena_allocator(Allocator::Get(db));

    // 创建键
    auto key = ARTKey::CreateKey(arena_allocator, types[0], scan_state.values[0]);
    key.VerifyKeyLength(MAX_KEY_LEN * prefix_count);

    if (scan_state.values[1].IsNull()) {
        // 单谓词
        lock_guard<mutex> l(lock);
        switch (scan_state.expressions[0]) {
        case ExpressionType::COMPARE_EQUAL:
            return SearchEqual(key, max_count, row_ids);
        case ExpressionType::COMPARE_GREATERTHANOREQUALTO:
            return SearchGreater(key, true, max_count, row_ids);
        case ExpressionType::COMPARE_GREATERTHAN:
            return SearchGreater(key, false, max_count, row_ids);
        case ExpressionType::COMPARE_LESSTHANOREQUALTO:
            return SearchLess(key, true, max_count, row_ids);
        case ExpressionType::COMPARE_LESSTHAN:
            return SearchLess(key, false, max_count, row_ids);
        default:
            throw InternalException("Index scan type not implemented");
        }
    }

    // 双谓词（范围查询）
    lock_guard<mutex> l(lock);
    auto upper_bound = ARTKey::CreateKey(arena_allocator, types[0], scan_state.values[1]);

    bool left_equal = scan_state.expressions[0] == ExpressionType::COMPARE_GREATERTHANOREQUALTO;
    bool right_equal = scan_state.expressions[1] == ExpressionType::COMPARE_LESSTHANOREQUALTO;
    return SearchCloseRange(key, upper_bound, left_equal, right_equal, max_count, row_ids);
}
```

---

## 6.5 物理计划生成

### 6.5.1 CreatePlan 方法

`ART::CreatePlan` 为 CREATE INDEX 生成物理执行计划：

```cpp
// src/execution/index/art/plan_art.cpp
PhysicalOperator &ART::CreatePlan(PlanIndexInput &input) {
    auto &op = input.op;
    auto &planner = input.planner;

    // 1. 创建投影算子：提取索引列
    vector<LogicalType> new_column_types;
    vector<unique_ptr<Expression>> select_list;
    for (idx_t i = 0; i < op.expressions.size(); i++) {
        new_column_types.push_back(op.expressions[i]->return_type);
        select_list.push_back(std::move(op.expressions[i]));
    }
    // 添加行ID列
    new_column_types.emplace_back(LogicalType::ROW_TYPE);
    select_list.push_back(make_uniq<BoundReferenceExpression>(
        LogicalType::ROW_TYPE, op.info->scan_types.size() - 1));

    auto &proj = planner.Make<PhysicalProjection>(
        new_column_types, std::move(select_list), op.estimated_cardinality);
    proj.children.push_back(input.table_scan);

    // 2. 可选的 NOT NULL 过滤器
    reference<PhysicalOperator> prev_op(proj);
    auto is_alter = op.alter_table_info != nullptr;

    if (!is_alter) {
        vector<unique_ptr<Expression>> filter_select_list;
        for (idx_t i = 0; i < new_column_types.size() - 1; i++) {
            auto is_not_null_expr = make_uniq<BoundOperatorExpression>(
                ExpressionType::OPERATOR_IS_NOT_NULL, LogicalType::BOOLEAN);
            auto bound_ref = make_uniq<BoundReferenceExpression>(new_column_types[i], i);
            is_not_null_expr->children.push_back(std::move(bound_ref));
            filter_select_list.push_back(std::move(is_not_null_expr));
        }
        prev_op = planner.Make<PhysicalFilter>(...);
    }

    // 3. 决定是否排序
    auto sort = true;
    if (op.unbound_expressions.size() > 1) {
        sort = false;  // 复合键不排序
    } else if (op.unbound_expressions[0]->return_type.InternalType() == PhysicalType::VARCHAR) {
        sort = false;  // VARCHAR 暂不排序
    }

    // 4. 创建 CREATE INDEX 算子
    auto &create_idx = planner.Make<PhysicalCreateARTIndex>(...);

    if (!sort) {
        create_idx.children.push_back(prev_op);
        return create_idx;
    }

    // 5. 添加 ORDER BY 算子（用于排序构建）
    vector<BoundOrderByNode> orders;
    for (idx_t i = 0; i < new_column_types.size() - 1; i++) {
        auto col_expr = make_uniq_base<Expression, BoundReferenceExpression>(
            new_column_types[i], i);
        orders.emplace_back(OrderType::ASCENDING, OrderByNullType::NULLS_FIRST,
                            std::move(col_expr));
    }

    auto &order = planner.Make<PhysicalOrder>(...);
    order.children.push_back(prev_op);
    create_idx.children.push_back(order);

    return create_idx;
}
```

### 6.5.2 物理计划树结构

```
CREATE INDEX 物理计划:

┌─────────────────────────────────────────────────────────────────┐
│                    PhysicalCreateARTIndex                       │
│                    (创建索引算子)                                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                ┌───────────┴───────────┐
                │ 排序?                  │
                ├───────────────────────┤
                │ YES        │ NO       │
                ▼            ▼          │
    ┌───────────────────┐    │          │
    │   PhysicalOrder   │    │          │
    │   (排序算子)       │    │          │
    └─────────┬─────────┘    │          │
              │              │          │
              └──────────────┴──────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │     PhysicalFilter      │
              │   (NOT NULL 过滤器)      │
              └───────────┬─────────────┘
                          │
                          ▼
              ┌─────────────────────────┐
              │   PhysicalProjection    │
              │   (索引列 + 行ID)        │
              └───────────┬─────────────┘
                          │
                          ▼
              ┌─────────────────────────┐
              │     PhysicalTableScan   │
              │   (全表扫描)             │
              └─────────────────────────┘
```

---

## 6.6 索引持久化

### 6.6.1 序列化到磁盘

```cpp
IndexStorageInfo ART::SerializeToDisk(QueryContext context,
                                       const case_insensitive_map_t<Value> &options) {
    lock_guard<mutex> guard(lock);

    // 检查是否使用旧版存储格式
    auto v1_0_0_option = options.find("v1_0_0_storage");
    bool v1_0_0_storage = v1_0_0_option == options.end() ||
                          v1_0_0_option->second != Value(false);

    // 准备序列化
    auto info = PrepareSerialize(options, v1_0_0_storage);
    auto allocator_count = v1_0_0_storage ? DEPRECATED_ALLOCATOR_COUNT : ALLOCATOR_COUNT;

    // 写入部分块
    WritePartialBlocks(context, v1_0_0_storage);

    // 收集分配器信息
    for (idx_t i = 0; i < allocator_count; i++) {
        info.allocator_infos.push_back((*allocators)[i]->GetInfo());
    }

    return info;
}
```

### 6.6.2 序列化到 WAL

```cpp
IndexStorageInfo ART::SerializeToWAL(const case_insensitive_map_t<Value> &options) {
    auto v1_0_0_option = options.find("v1_0_0_storage");
    bool v1_0_0_storage = v1_0_0_option == options.end() ||
                          v1_0_0_option->second != Value(false);

    auto info = PrepareSerialize(options, v1_0_0_storage);
    auto allocator_count = v1_0_0_storage ? DEPRECATED_ALLOCATOR_COUNT : ALLOCATOR_COUNT;

    // 初始化 WAL 序列化
    for (idx_t i = 0; i < allocator_count; i++) {
        info.buffers.push_back((*allocators)[i]->InitSerializationToWAL());
    }

    for (idx_t i = 0; i < allocator_count; i++) {
        info.allocator_infos.push_back((*allocators)[i]->GetInfo());
    }

    return info;
}
```

### 6.6.3 IndexStorageInfo 结构

```cpp
struct IndexStorageInfo {
    string name;                   // 索引名称
    idx_t root;                    // 根节点指针
    case_insensitive_map_t<Value> options;  // 配置选项
    vector<FixedSizeAllocatorInfo> allocator_infos;  // 分配器信息
    vector<vector<IndexBufferInfo>> buffers;  // WAL 缓冲区
    BlockPointer root_block_ptr;   // 旧版兼容
};
```

### 6.6.4 反序列化

```cpp
void ART::Deserialize(const BlockPointer &pointer) {
    D_ASSERT(pointer.IsValid());

    auto &metadata_manager = table_io_manager.GetMetadataManager();
    MetadataReader reader(metadata_manager, pointer);

    // 读取根节点
    tree = reader.Read<Node>();

    // 读取分配器状态
    for (idx_t i = 0; i < DEPRECATED_ALLOCATOR_COUNT; i++) {
        (*allocators)[i]->Deserialize(metadata_manager, reader.Read<BlockPointer>());
    }
}

void ART::InitAllocators(const IndexStorageInfo &info) {
    for (idx_t i = 0; i < info.allocator_infos.size(); i++) {
        (*allocators)[i]->Init(info.allocator_infos[i]);
    }
}
```

---

## 6.7 ARTKey 生成

### 6.7.1 键生成模板

```cpp
template <class T, bool IS_NOT_NULL>
static void TemplatedGenerateKeys(ArenaAllocator &allocator, Vector &input,
                                   idx_t count, unsafe_vector<ARTKey> &keys) {
    UnifiedVectorFormat data;
    input.ToUnifiedFormat(count, data);
    auto input_data = UnifiedVectorFormat::GetData<T>(data);

    for (idx_t i = 0; i < count; i++) {
        auto idx = data.sel->get_index(i);
        if (IS_NOT_NULL || data.validity.RowIsValid(idx)) {
            ARTKey::CreateARTKey<T>(allocator, keys[i], input_data[idx]);
            continue;
        }
        // NULL 值使用空键
        keys[i] = ARTKey();
    }
}
```

### 6.7.2 复合键连接

```cpp
template <class T, bool IS_NOT_NULL>
static void ConcatenateKeys(ArenaAllocator &allocator, Vector &input,
                             idx_t count, unsafe_vector<ARTKey> &keys) {
    UnifiedVectorFormat data;
    input.ToUnifiedFormat(count, data);
    auto input_data = UnifiedVectorFormat::GetData<T>(data);

    for (idx_t i = 0; i < count; i++) {
        auto idx = data.sel->get_index(i);

        if (IS_NOT_NULL) {
            auto other_key = ARTKey::CreateARTKey<T>(allocator, input_data[idx]);
            keys[i].Concat(allocator, other_key);
            continue;
        }

        // 之前的列是 NULL
        if (keys[i].Empty()) {
            continue;
        }

        // 当前列是 NULL，整个键变为 NULL
        if (!data.validity.RowIsValid(idx)) {
            keys[i] = ARTKey();
            continue;
        }

        // 连接键
        auto other_key = ARTKey::CreateARTKey<T>(allocator, input_data[idx]);
        keys[i].Concat(allocator, other_key);
    }
}
```

### 6.7.3 支持的数据类型

```cpp
// GenerateKeysInternal 支持的类型
case PhysicalType::BOOL:
case PhysicalType::INT8:
case PhysicalType::INT16:
case PhysicalType::INT32:
case PhysicalType::INT64:
case PhysicalType::INT128:
case PhysicalType::UINT8:
case PhysicalType::UINT16:
case PhysicalType::UINT32:
case PhysicalType::UINT64:
case PhysicalType::UINT128:
case PhysicalType::FLOAT:
case PhysicalType::DOUBLE:
case PhysicalType::VARCHAR:
```

---

## 6.8 索引扫描流程总结

```
┌─────────────────────────────────────────────────────────────────┐
│                      索引扫描完整流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 优化器尝试匹配索引                                           │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ TryInitializeScan(expr, filter_expr)                   │  │
│     │ • 使用 ExpressionMatcher 匹配谓词                       │  │
│     │ • 提取比较类型和常量值                                   │  │
│     │ • 创建 ARTIndexScanState                               │  │
│     └────────────────────────────────────────────────────────┘  │
│                           ↓                                      │
│  2. 执行器调用索引扫描                                           │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ ART::Scan(state, max_count, row_ids)                   │  │
│     │ • 创建 ARTKey                                          │  │
│     │ • 根据谓词类型选择扫描方法                               │  │
│     └────────────────────────────────────────────────────────┘  │
│                           ↓                                      │
│  3. Iterator 遍历 ART                                           │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ a. LowerBound 或 FindMinimum                           │  │
│     │ b. Scan 迭代叶子节点                                    │  │
│     │ c. 提取行ID到 set<row_t>                               │  │
│     │ d. Next 前进到下一个叶子                                │  │
│     └────────────────────────────────────────────────────────┘  │
│                           ↓                                      │
│  4. 返回行ID用于表扫描                                          │
│     ┌────────────────────────────────────────────────────────┐  │
│     │ row_ids → TableScan 过滤器                             │  │
│     │ 只扫描匹配的行                                          │  │
│     └────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6.9 本章小结

本章详细分析了 DuckDB 索引系统在查询执行中的应用：

1. **索引扫描状态**：`ARTIndexScanState` 存储谓词值和表达式类型，支持点查询和范围扫描。

2. **谓词匹配**：`TryInitializeScan` 使用表达式匹配器识别可索引的比较谓词，支持等值、比较和 BETWEEN 表达式。

3. **Iterator 迭代器**：核心遍历组件，实现 `FindMinimum`、`LowerBound`、`Scan` 和 `Next` 操作，支持有序遍历和范围查询。

4. **范围查询**：四种查询方法（SearchEqual、SearchGreater、SearchLess、SearchCloseRange）覆盖所有比较操作。

5. **物理计划生成**：`CreatePlan` 生成完整的索引创建物理计划，包括投影、过滤、排序和创建算子。

6. **持久化机制**：支持磁盘序列化和 WAL 序列化两种模式，兼容旧版存储格式。

7. **键生成**：模板化的键生成支持多种数据类型，支持复合键的连接操作。

---

## 6.10 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/execution/index/art/art.cpp` | ART 主实现（扫描、序列化） |
| `src/include/duckdb/execution/index/art/iterator.hpp` | Iterator 定义 |
| `src/execution/index/art/iterator.cpp` | Iterator 实现 |
| `src/execution/index/art/plan_art.cpp` | 物理计划生成 |
| `src/include/duckdb/storage/table/scan_state.hpp` | IndexScanState 定义 |
| `src/include/duckdb/storage/index_storage_info.hpp` | IndexStorageInfo 定义 |
| `src/execution/operator/schema/physical_create_art_index.cpp` | 物理算子实现 |

---

## 索引系统系列总结

本系列六章完整覆盖了 DuckDB 的索引系统：

1. **第一章**：索引架构总览，三层抽象设计
2. **第二章**：ART 核心原理和 ARTKey 编码
3. **第三章**：节点类型体系和内存管理
4. **第四章**：核心操作（构建、插入、删除、合并）
5. **第五章**：约束实现和冲突管理
6. **第六章**：查询执行和持久化

DuckDB 的 ART 索引实现展现了现代数据库索引设计的多个特点：
- 自适应节点策略平衡空间和时间效率
- 前缀压缩减少内存占用和树高度
- 嵌套叶子优雅处理重复键
- 事务双值叶子支持并发更新
- 分配器分离实现高效内存管理
- 完善的序列化支持持久化和恢复
