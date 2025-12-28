# DuckDB 索引系统深度解析 - 第五章：索引约束与完整性

本章深入分析 DuckDB 索引系统中的约束实现机制，包括 UNIQUE、PRIMARY KEY 和 FOREIGN KEY 约束的验证流程、冲突管理以及事务集成。

## 5.1 约束类型体系

### 5.1.1 IndexConstraintType 枚举

DuckDB 定义了四种索引约束类型：

```cpp
// src/include/duckdb/common/enums/index_constraint_type.hpp
enum class IndexConstraintType : uint8_t {
    NONE = 0,    // 普通索引，无约束
    UNIQUE = 1,  // UNIQUE 约束
    PRIMARY = 2, // PRIMARY KEY 约束
    FOREIGN = 3  // FOREIGN KEY 约束
};
```

### 5.1.2 约束类型特性对比

```
┌─────────────────┬───────────┬───────────┬───────────┬───────────────────────┐
│ 约束类型         │ 唯一性    │ 允许NULL  │ 引用检查   │ 主要用途               │
├─────────────────┼───────────┼───────────┼───────────┼───────────────────────┤
│ NONE            │ 否        │ 是        │ 否        │ 加速查询               │
│ UNIQUE          │ 是        │ 是        │ 否        │ 保证列值唯一性          │
│ PRIMARY         │ 是        │ 否        │ 否        │ 主键约束               │
│ FOREIGN         │ 否        │ 是        │ 是        │ 引用完整性              │
└─────────────────┴───────────┴───────────┴───────────┴───────────────────────┘
```

### 5.1.3 Index 类约束查询方法

```cpp
// 在 Index 基类中定义
class Index {
public:
    //! 获取约束类型
    virtual IndexConstraintType GetConstraintType() const = 0;

    //! 便捷方法
    bool IsPrimary() const {
        return GetConstraintType() == IndexConstraintType::PRIMARY;
    }
    bool IsForeign() const {
        return GetConstraintType() == IndexConstraintType::FOREIGN;
    }
    bool IsUnique() const {
        auto type = GetConstraintType();
        return type == IndexConstraintType::UNIQUE ||
               type == IndexConstraintType::PRIMARY;
    }
};
```

---

## 5.2 UNIQUE 约束实现

### 5.2.1 唯一性检查原理

UNIQUE 约束在 ART 索引中通过检测重复键实现：

```cpp
// ART 中的唯一性检查
bool ART::IsUnique() const {
    return index_constraint_type == IndexConstraintType::UNIQUE ||
           index_constraint_type == IndexConstraintType::PRIMARY;
}
```

### 5.2.2 插入时的唯一性验证

当插入遇到已存在的键时：

```cpp
// ARTOperator::InsertIntoInlined
static ARTConflictType InsertIntoInlined(
    ArenaAllocator &arena, ART &art, Node &node,
    const ARTKey &key, const ARTKey &row_id, const idx_t depth,
    const GateStatus status, optional_ptr<ART> delete_art,
    const IndexAppendMode append_mode) {

    Node row_id_node;
    Leaf::New(row_id_node, row_id.GetRowId());

    // 非唯一索引或允许重复：合并叶子
    if (!art.IsUnique() || append_mode == IndexAppendMode::INSERT_DUPLICATES) {
        Leaf::MergeInlined(arena, art, node, row_id_node, status, depth);
        return ARTConflictType::NO_CONFLICT;
    }

    // 唯一索引：检查是否为事务更新
    if (!delete_art) {
        if (append_mode == IndexAppendMode::IGNORE_DUPLICATES) {
            return ARTConflictType::NO_CONFLICT;
        }
        return ARTConflictType::CONSTRAINT;  // 约束冲突！
    }

    // 检查 delete_art 中是否有匹配的删除记录
    auto delete_leaf = Lookup(*delete_art, delete_art->tree, key, 0);
    if (!delete_leaf) {
        return ARTConflictType::CONSTRAINT;
    }

    // 验证行ID是否匹配（同一行的 DELETE+INSERT）
    auto deleted_row_id = delete_leaf->GetRowId();
    auto this_row_id = node.GetRowId();
    if (deleted_row_id != this_row_id) {
        return ARTConflictType::CONSTRAINT;
    }

    // 允许事务内的原地更新
    Leaf::MergeInlined(arena, art, node, row_id_node, status, depth);
    return ARTConflictType::NO_CONFLICT;
}
```

### 5.2.3 冲突类型枚举

```cpp
enum class ARTConflictType : uint8_t {
    NO_CONFLICT = 0,    // 无冲突
    CONSTRAINT = 1,     // 约束冲突（唯一性违反）
    TRANSACTION = 2     // 事务冲突
};
```

### 5.2.4 批量构建时的唯一性检查

```cpp
// ARTBuilder::Build
ARTConflictType ARTBuilder::Build() {
    while (!s.empty()) {
        auto entry = s.top();
        s.pop();

        // ... 处理公共前缀 ...

        // 到达叶子时检查重复
        if (start.len == entry.depth) {
            auto row_id_count = entry.end - entry.start + 1;

            // 唯一索引不允许多个行ID
            if (art.IsUnique() && row_id_count != 1) {
                return ARTConflictType::CONSTRAINT;
            }

            // ... 创建叶子节点 ...
        }
    }
    return ARTConflictType::NO_CONFLICT;
}
```

---

## 5.3 PRIMARY KEY 约束

### 5.3.1 PRIMARY KEY 的特殊处理

PRIMARY KEY 在 UNIQUE 基础上增加 NOT NULL 检查：

```cpp
// PhysicalCreateARTIndex::Sink
SinkResultType PhysicalCreateARTIndex::Sink(
    ExecutionContext &context, DataChunk &chunk, OperatorSinkInput &input) const {

    auto &l_state = input.local_state.Cast<CreateARTIndexLocalSinkState>();

    // ALTER TABLE 时检查 NULL 值
    if (alter_table_info) {
        auto row_count = l_state.key_chunk.size();
        for (idx_t i = 0; i < l_state.key_chunk.ColumnCount(); i++) {
            if (VectorOperations::HasNull(l_state.key_chunk.data[i], row_count)) {
                throw ConstraintException("NOT NULL constraint failed: %s",
                                          info->index_name);
            }
        }
    }

    // ... 继续处理 ...
}
```

### 5.3.2 物理计划中的 NOT NULL 过滤

```cpp
// plan_art.cpp::CreatePlan
PhysicalOperator &ART::CreatePlan(PlanIndexInput &input) {
    // ... 创建投影 ...

    // 非 ALTER 时添加 NOT NULL 过滤
    auto is_alter = op.alter_table_info != nullptr;
    if (!is_alter) {
        vector<unique_ptr<Expression>> filter_select_list;
        auto not_null_type = ExpressionType::OPERATOR_IS_NOT_NULL;

        for (idx_t i = 0; i < new_column_types.size() - 1; i++) {
            auto is_not_null_expr = make_uniq<BoundOperatorExpression>(
                not_null_type, LogicalType::BOOLEAN);
            auto bound_ref = make_uniq<BoundReferenceExpression>(
                new_column_types[i], i);
            is_not_null_expr->children.push_back(std::move(bound_ref));
            filter_select_list.push_back(std::move(is_not_null_expr));
        }

        prev_op = planner.Make<PhysicalFilter>(
            std::move(filter_types), std::move(filter_select_list),
            op.estimated_cardinality);
    }
}
```

---

## 5.4 FOREIGN KEY 约束

### 5.4.1 外键验证流程

外键约束涉及两个表：主键表（被引用）和外键表（引用者）。

```cpp
// TableIndexList::VerifyForeignKey
void TableIndexList::VerifyForeignKey(
    optional_ptr<LocalTableStorage> storage,
    const vector<PhysicalIndex> &fk_keys,
    DataChunk &chunk,
    ConflictManager &conflict_manager) {

    // 确定外键类型
    auto fk_type = conflict_manager.GetVerifyExistenceType() == VerifyExistenceType::APPEND_FK
                   ? ForeignKeyType::FK_TYPE_PRIMARY_KEY_TABLE
                   : ForeignKeyType::FK_TYPE_FOREIGN_KEY_TABLE;

    // 查找匹配的外键索引
    auto entry = FindForeignKeyIndex(fk_keys, fk_type);
    auto &index = *entry->index;
    D_ASSERT(index.IsBound());

    // 获取删除索引（用于事务处理）
    optional_ptr<BoundIndex> delete_index;
    if (storage) {
        delete_index = storage->delete_indexes.Find(index.GetIndexName());
    }
    IndexAppendInfo index_append_info(IndexAppendMode::DEFAULT, delete_index);

    // 验证约束
    lock_guard<mutex> entry_lock(entry->lock);
    auto &main_index = index.Cast<BoundIndex>();
    main_index.VerifyConstraint(chunk, index_append_info, conflict_manager);

    // 检查检查点期间添加的数据
    if (entry->added_data_during_checkpoint) {
        IndexAppendInfo added_during_checkpoint_info;
        entry->added_data_during_checkpoint->VerifyConstraint(
            chunk, added_during_checkpoint_info, conflict_manager);
    }
}
```

### 5.4.2 外键索引匹配

```cpp
bool IsForeignKeyIndex(const vector<PhysicalIndex> &fk_keys,
                       Index &index, ForeignKeyType fk_type) {
    // 检查索引类型
    if (fk_type == ForeignKeyType::FK_TYPE_PRIMARY_KEY_TABLE
        ? !index.IsUnique() : !index.IsForeign()) {
        return false;
    }

    // 检查列数匹配
    if (fk_keys.size() != index.GetColumnIds().size()) {
        return false;
    }

    // 检查列匹配
    auto &column_ids = index.GetColumnIds();
    for (auto &fk_key : fk_keys) {
        bool found = false;
        for (auto &index_key : column_ids) {
            if (fk_key.index == index_key) {
                found = true;
                break;
            }
        }
        if (!found) {
            return false;
        }
    }
    return true;
}
```

### 5.4.3 外键验证场景

```
场景1: 插入外键表记录
┌─────────────────────────────────────────────────────────────────┐
│ INSERT INTO orders (customer_id, ...) VALUES (123, ...)         │
├─────────────────────────────────────────────────────────────────┤
│ 验证流程:                                                        │
│ 1. 在主键表 (customers) 的索引中查找 customer_id = 123          │
│ 2. 如果找到 → 允许插入                                           │
│ 3. 如果未找到 → 抛出约束冲突                                     │
└─────────────────────────────────────────────────────────────────┘

场景2: 删除主键表记录
┌─────────────────────────────────────────────────────────────────┐
│ DELETE FROM customers WHERE id = 123                             │
├─────────────────────────────────────────────────────────────────┤
│ 验证流程:                                                        │
│ 1. 在外键表 (orders) 的索引中查找 customer_id = 123             │
│ 2. 如果未找到 → 允许删除                                         │
│ 3. 如果找到 → 抛出约束冲突（除非有 CASCADE）                     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.5 ConflictManager 冲突管理器

### 5.5.1 设计概述

`ConflictManager` 在约束验证期间跟踪冲突，支持 `ON CONFLICT DO` 语义：

```cpp
// src/include/duckdb/common/types/conflict_manager.hpp
class ConflictManager {
public:
    ConflictManager(const VerifyExistenceType lookup_type,
                    const idx_t chunk_size,
                    optional_ptr<ConflictInfo> conflict_info = nullptr);

    //! 添加命中记录（找到冲突）
    bool AddHit(const idx_t index_in_chunk, const row_t row_id);

    //! 添加第二个命中（事务双值叶子）
    bool AddSecondHit(const idx_t index_in_chunk, const row_t row_id);

    //! 添加 NULL（某些约束需要特殊处理）
    bool AddNull(const idx_t index_in_chunk);

    //! 完成查找
    void FinishLookup();

    //! 获取第一个无效索引
    optional_idx GetFirstInvalidIndex(const idx_t count, const bool negate = false);

    //! 设置冲突管理模式
    void SetMode(const ConflictManagerMode mode_p);

private:
    VerifyExistenceType verify_existence_type;
    idx_t chunk_size;
    optional_ptr<ConflictInfo> conflict_info;
    ConflictManagerMode mode;

    struct ConflictData {
        idx_t count = 0;
        unique_ptr<SelectionVector> sel;
        unique_ptr<SelectionVector> inverted_sel;
        unique_ptr<Vector> row_ids;
        row_t *row_ids_data;
        ValidityArray validity;

        void Insert(const idx_t index_in_chunk, const row_t row_id);
        row_t GetRowId(const idx_t index_in_chunk);
    };

    array<ConflictData, 2> conflict_data;
};
```

### 5.5.2 冲突管理模式

```cpp
enum class ConflictManagerMode : uint8_t {
    SCAN,   // 收集冲突但不抛出异常（用于 ON CONFLICT DO）
    THROW   // 发现冲突时抛出异常
};
```

### 5.5.3 验证存在性类型

```cpp
enum class VerifyExistenceType : uint8_t {
    APPEND,         // 插入新行
    APPEND_FK,      // 插入外键引用
    DELETE_FK       // 删除被引用的主键
};
```

### 5.5.4 冲突处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    约束验证与冲突处理                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  输入: DataChunk (待插入/删除的数据)                             │
│         ↓                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ ConflictManager 初始化                                      ││
│  │ • 设置 verify_existence_type                                ││
│  │ • 设置 chunk_size                                           ││
│  │ • 可选: conflict_info (ON CONFLICT DO)                      ││
│  └─────────────────────────────────────────────────────────────┘│
│         ↓                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 索引扫描                                                     ││
│  │ for each row in chunk:                                      ││
│  │   lookup(key) in index                                      ││
│  │   if hit:                                                   ││
│  │     conflict_manager.AddHit(row_idx, existing_row_id)       ││
│  │   if null_key:                                              ││
│  │     conflict_manager.AddNull(row_idx)                       ││
│  └─────────────────────────────────────────────────────────────┘│
│         ↓                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 冲突决策                                                     ││
│  │ if mode == THROW && has_conflicts:                          ││
│  │   throw ConstraintException                                 ││
│  │ if mode == SCAN:                                            ││
│  │   return conflicts for ON CONFLICT DO handling              ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5.6 事务中的约束处理

### 5.6.1 事务双值叶子

唯一索引支持事务内的 DELETE+INSERT 操作，通过双值叶子实现：

```cpp
// ARTOperator::Insert 中的事务处理
if (status == GateStatus::GATE_NOT_SET &&
    active_node.GetGateStatus() == GateStatus::GATE_SET) {
    if (!art.IsUnique()) {
        // 非唯一索引：进入嵌套 ART
        active_key_ref = row_id;
        depth = 0;
        status = GateStatus::GATE_SET;
        continue;
    }
    // 唯一索引的事务冲突
    return ARTConflictType::TRANSACTION;
}
```

### 5.6.2 Delete 索引

事务维护独立的删除索引来跟踪待删除的键：

```cpp
// LocalTableStorage 中的删除索引
class LocalTableStorage {
    TableIndexList delete_indexes;  // 跟踪本事务删除的键
};

// 验证时使用删除索引
optional_ptr<BoundIndex> delete_index;
if (storage) {
    delete_index = storage->delete_indexes.Find(index.GetIndexName());
}
IndexAppendInfo index_append_info(IndexAppendMode::DEFAULT, delete_index);
```

### 5.6.3 ConflictManager 的事务处理

```cpp
// 处理双值叶子的两个行ID
bool ConflictManager::AddHit(const idx_t index_in_chunk, const row_t row_id) {
    auto callback = [&]() {
        AddRowId(index_in_chunk, row_id);
    };
    return AddHit(index_in_chunk, callback);
}

bool ConflictManager::AddSecondHit(const idx_t index_in_chunk, const row_t row_id) {
    // 记录第二个可能的行ID
    AddSecondRowId(index_in_chunk, row_id);
    return false;  // 不立即抛出，等待最终决定
}

// 最终决定哪个行ID可见
void ConflictManager::Finalize(const std::function<bool(const row_t row_id)> &is_visible) {
    // 对于每个冲突，检查哪个行ID对当前事务可见
    for (each conflict) {
        auto row_id_1 = conflict_data[FIRST].GetRowId(idx);
        auto row_id_2 = conflict_data[SECOND].GetRowId(idx);

        if (is_visible(row_id_1)) {
            // 使用第一个行ID
        } else if (is_visible(row_id_2)) {
            // 使用第二个行ID
            conflict_data[FIRST].row_ids_data[...] = row_id_2;
        }
    }
}
```

---

## 5.7 IndexAppendMode 追加模式

### 5.7.1 追加模式枚举

```cpp
enum class IndexAppendMode : uint8_t {
    DEFAULT,            // 默认：唯一索引检查冲突
    IGNORE_DUPLICATES,  // 忽略重复
    INSERT_DUPLICATES   // 允许插入重复（WAL 重放）
};
```

### 5.7.2 追加模式使用场景

```
┌─────────────────────────────────────────────────────────────────┐
│ 追加模式                │ 使用场景                              │
├─────────────────────────┼───────────────────────────────────────┤
│ DEFAULT                 │ 普通插入操作                          │
│ IGNORE_DUPLICATES       │ INSERT ... ON CONFLICT DO NOTHING     │
│ INSERT_DUPLICATES       │ WAL 重放（事务日志恢复）               │
└─────────────────────────┴───────────────────────────────────────┘
```

### 5.7.3 IndexAppendInfo 结构

```cpp
struct IndexAppendInfo {
    IndexAppendMode mode;
    optional_ptr<BoundIndex> delete_index;

    IndexAppendInfo(IndexAppendMode mode_p = IndexAppendMode::DEFAULT,
                    optional_ptr<BoundIndex> delete_idx = nullptr)
        : mode(mode_p), delete_index(delete_idx) {}
};
```

---

## 5.8 约束异常处理

### 5.8.1 ConstraintException

约束违反时抛出 `ConstraintException`：

```cpp
// 唯一性约束违反
if (conflict_type == ARTConflictType::CONSTRAINT) {
    throw ConstraintException("Data contains duplicates on indexed column(s)");
}

// NOT NULL 约束违反
if (VectorOperations::HasNull(chunk.data[i], row_count)) {
    throw ConstraintException("NOT NULL constraint failed: %s", index_name);
}

// 外键约束违反
if (!found_in_parent_table) {
    throw ConstraintException("FOREIGN KEY constraint failed");
}
```

### 5.8.2 错误信息生成

```cpp
// BoundIndex::AppendRowError
string BoundIndex::AppendRowError(DataChunk &input, idx_t index) {
    string error;
    for (idx_t c = 0; c < input.ColumnCount(); c++) {
        if (c > 0) {
            error += ", ";
        }
        error += input.GetValue(c, index).ToString();
    }
    return error;
}
```

---

## 5.9 约束验证时序

### 5.9.1 INSERT 操作的约束检查

```
INSERT INTO table VALUES (...):

1. 解析并绑定语句
2. 执行表达式计算
3. 对每个相关索引:
   ┌────────────────────────────────────────────────────────────┐
   │ a. 初始化 ConflictManager                                  │
   │ b. 调用 index.VerifyAppend() 或 index.VerifyConstraint()  │
   │ c. 在索引中查找待插入的键                                   │
   │ d. 记录冲突（如果有）                                       │
   │ e. 根据模式决定是否抛出异常                                 │
   └────────────────────────────────────────────────────────────┘
4. 如果通过所有约束检查:
   ┌────────────────────────────────────────────────────────────┐
   │ a. 调用 index.Append() 添加索引条目                        │
   │ b. 插入数据到存储层                                         │
   └────────────────────────────────────────────────────────────┘
5. 提交或回滚
```

### 5.9.2 UPDATE 操作的约束检查

```
UPDATE table SET column = value WHERE ...:

1. 找到需要更新的行
2. 对每个受影响的索引:
   ┌────────────────────────────────────────────────────────────┐
   │ a. 检查索引列是否被更新                                     │
   │    if (!index.IndexIsUpdated(updated_columns)) continue;   │
   │ b. 从索引中删除旧键                                         │
   │    index.Delete(old_keys, row_ids);                        │
   │ c. 验证新键的约束                                           │
   │ d. 将新键添加到索引                                         │
   │    index.Append(new_keys, row_ids);                        │
   └────────────────────────────────────────────────────────────┘
3. 更新存储层数据
```

### 5.9.3 DELETE 操作的约束检查

```
DELETE FROM table WHERE ...:

1. 找到需要删除的行
2. 检查外键约束:
   ┌────────────────────────────────────────────────────────────┐
   │ 对每个引用此表的外键:                                       │
   │ a. 在子表中查找引用此行的记录                               │
   │ b. 如果找到且无 CASCADE → 约束冲突                         │
   │ c. 如果找到且有 CASCADE → 级联删除                         │
   └────────────────────────────────────────────────────────────┘
3. 从所有索引中删除条目
   ┌────────────────────────────────────────────────────────────┐
   │ for each index:                                            │
   │   index.Delete(keys, row_ids);                             │
   └────────────────────────────────────────────────────────────┘
4. 从存储层删除数据
```

---

## 5.10 本章小结

本章详细分析了 DuckDB 索引系统中的约束实现：

1. **约束类型体系**：四种约束类型（NONE、UNIQUE、PRIMARY、FOREIGN）各有不同的验证逻辑和使用场景。

2. **UNIQUE 约束**：通过 ART 索引的键唯一性检查实现，在插入和批量构建时验证。支持 `ON CONFLICT DO` 语义。

3. **PRIMARY KEY 约束**：在 UNIQUE 基础上增加 NOT NULL 检查，通过物理计划中的 Filter 算子或显式检查实现。

4. **FOREIGN KEY 约束**：通过 `VerifyForeignKey` 在主键表和外键表之间验证引用完整性。

5. **ConflictManager**：核心冲突跟踪机制，支持 SCAN（收集冲突）和 THROW（抛出异常）两种模式，实现 `ON CONFLICT DO` 功能。

6. **事务集成**：通过删除索引和双值叶子支持事务内的 DELETE+INSERT 更新，正确处理可见性。

7. **追加模式**：DEFAULT、IGNORE_DUPLICATES 和 INSERT_DUPLICATES 三种模式适应不同场景。

---

## 5.11 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/common/enums/index_constraint_type.hpp` | 约束类型枚举 |
| `src/include/duckdb/common/types/conflict_manager.hpp` | ConflictManager 定义 |
| `src/common/types/conflict_manager.cpp` | ConflictManager 实现 |
| `src/storage/table_index_list.cpp` | VerifyForeignKey 实现 |
| `src/execution/index/bound_index.cpp` | BoundIndex 约束方法 |
| `src/include/duckdb/execution/index/art/art_operator.hpp` | 插入冲突检测 |
| `src/execution/index/art/art_builder.cpp` | 批量构建冲突检测 |
| `src/execution/operator/schema/physical_create_art_index.cpp` | NOT NULL 检查 |
