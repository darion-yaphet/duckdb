# DuckDB 索引系统深度解析 - 第四章：索引操作与维护

本章深入分析 DuckDB ART 索引的核心操作实现，包括索引构建、查找、插入、删除、合并以及垃圾回收机制。

## 4.1 ARTOperator 核心操作类

### 4.1.1 操作类设计

`ARTOperator` 是一个静态工具类，提供 ART 索引的三大核心操作：

```cpp
// src/include/duckdb/execution/index/art/art_operator.hpp
class ARTOperator {
public:
    //! 查找：返回匹配键的叶子节点指针
    static unsafe_optional_ptr<const Node> Lookup(
        ART &art, const Node &node, const ARTKey &key, idx_t depth);

    //! 叶子内查找：检查行ID是否存在于叶子中
    static bool LookupInLeaf(ART &art, const Node &node, const ARTKey &rowid);

    //! 插入：将键和行ID插入到节点中
    static ARTConflictType Insert(
        ArenaAllocator &arena, ART &art, Node &node,
        const ARTKey &key, idx_t depth, const ARTKey &row_id,
        GateStatus status, optional_ptr<ART> delete_art,
        const IndexAppendMode append_mode);

    //! 删除：从树中删除键和行ID
    static void Delete(ART &art, Node &node,
                       const ARTKey &key, const ARTKey &row_id);

private:
    static ARTConflictType InsertIntoInlined(...);
    static void InsertIntoNode(...);
    static void InsertIntoPrefix(...);
};
```

### 4.1.2 冲突类型定义

```cpp
enum class ARTConflictType : uint8_t {
    NO_CONFLICT = 0,    // 无冲突，操作成功
    CONSTRAINT = 1,     // 约束冲突（唯一性违反）
    TRANSACTION = 2,    // 事务冲突
};
```

---

## 4.2 查找操作（Lookup）

### 4.2.1 基本查找算法

`Lookup` 实现从根节点到叶子的遍历，返回匹配键的叶子节点：

```cpp
static unsafe_optional_ptr<const Node> Lookup(
    ART &art, const Node &node, const ARTKey &key, idx_t depth) {

    reference<const Node> ref(node);

    while (ref.get().HasMetadata()) {
        // 1. 到达叶子或门控节点
        if (ref.get().IsAnyLeaf() ||
            ref.get().GetGateStatus() == GateStatus::GATE_SET) {
            return unsafe_optional_ptr<const Node>(ref.get());
        }

        // 2. 遍历前缀节点
        if (ref.get().GetType() == NType::PREFIX) {
            Prefix prefix(art, ref.get());
            for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
                if (prefix.data[i] != key[depth]) {
                    // 前缀不匹配
                    return nullptr;
                }
                depth++;
            }
            ref = *prefix.ptr;
            continue;
        }

        // 3. 在内部节点中查找子节点
        D_ASSERT(depth < key.len);
        auto child = ref.get().GetChild(art, key[depth]);

        if (!child) {
            // 没有对应的子节点
            return nullptr;
        }

        // 4. 继续在子节点中查找
        ref = *child;
        depth++;
    }

    return nullptr;
}
```

### 4.2.2 查找流程图

```
Lookup(key = "hello"):

     [root]
        │
        ▼
   ┌─────────────────┐
   │ PREFIX: "hel"   │  depth: 0 → 3
   └────────┬────────┘
            │ 匹配成功
            ▼
   ┌─────────────────┐
   │    Node4        │  key[3] = 'l' ?
   │  ┌─┬─┬─┬─┐     │
   │  │l│p│ │ │     │
   └──┴─┴─┴─┴─┴─────┘
            │ 找到 'l'
            ▼
   ┌─────────────────┐
   │ PREFIX: "o"     │  depth: 4 → 5
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │ LEAF_INLINED    │  ← 返回此节点
   │ row_id: 42      │
   └─────────────────┘
```

### 4.2.3 叶子内查找（LookupInLeaf）

当叶子可能包含多个行 ID（嵌套叶子）时，需要在叶子内部继续查找：

```cpp
static bool LookupInLeaf(ART &art, const Node &node, const ARTKey &rowid) {
    reference<const Node> ref(node);
    idx_t depth = 0;

    while (ref.get().HasMetadata()) {
        const auto type = ref.get().GetType();
        switch (type) {
        case NType::LEAF_INLINED:
            // 直接比较行ID
            return ref.get().GetRowId() == rowid.GetRowId();

        case NType::NODE_7_LEAF:
        case NType::NODE_15_LEAF:
        case NType::NODE_256_LEAF:
            // 叶子节点：检查最后一个字节
            return ref.get().HasByte(art, rowid[Prefix::ROW_ID_COUNT]);

        case NType::NODE_4:
        case NType::NODE_16:
        case NType::NODE_48:
        case NType::NODE_256:
            // 内部节点：继续查找
            auto child = ref.get().GetChild(art, rowid[depth]);
            if (child) {
                ref = *child;
                depth++;
                continue;
            }
            return false;

        case NType::PREFIX:
            // 前缀匹配
            Prefix prefix(art, ref.get());
            for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
                if (prefix.data[i] != rowid[depth]) {
                    return false;
                }
                depth++;
            }
            ref = *prefix.ptr;
        }
    }
    return false;
}
```

---

## 4.3 插入操作（Insert）

### 4.3.1 插入算法概述

插入操作遍历树直到找到合适的位置，可能触发以下情况：

1. **空树**：创建前缀链 + 内联叶子
2. **到达内联叶子**：合并或报告冲突
3. **前缀不匹配**：分裂前缀
4. **子节点不存在**：插入新子节点
5. **进入门控**：切换到嵌套 ART

```cpp
static ARTConflictType Insert(
    ArenaAllocator &arena, ART &art, Node &node,
    const ARTKey &key, idx_t depth, const ARTKey &row_id,
    GateStatus status, optional_ptr<ART> delete_art,
    const IndexAppendMode append_mode) {

    reference<Node> active_node_ref(node);
    reference<const ARTKey> active_key_ref(key);

    // 空树快速路径
    if (!node.HasMetadata()) {
        D_ASSERT(depth == 0);
        if (status == GateStatus::GATE_SET) {
            Leaf::New(node, row_id.GetRowId());
            return ARTConflictType::NO_CONFLICT;
        }
        Prefix::New(art, active_node_ref, key, depth, key.len);
        Leaf::New(active_node_ref, row_id.GetRowId());
        return ARTConflictType::NO_CONFLICT;
    }

    while (active_node_ref.get().HasMetadata()) {
        auto &active_node = active_node_ref.get();
        auto &active_key = active_key_ref.get();

        // 处理门控节点
        if (status == GateStatus::GATE_NOT_SET &&
            active_node.GetGateStatus() == GateStatus::GATE_SET) {
            if (!art.IsUnique()) {
                // 进入嵌套 ART
                active_key_ref = row_id;
                depth = 0;
                status = GateStatus::GATE_SET;
                continue;
            }
            // 唯一索引事务冲突
            return ARTConflictType::TRANSACTION;
        }

        switch (active_node.GetType()) {
        case NType::LEAF_INLINED:
            return InsertIntoInlined(arena, art, active_node, key,
                                     row_id, depth, status, delete_art, append_mode);

        case NType::LEAF:
            // 转换已弃用的叶子
            Leaf::TransformToNested(art, active_node);
            continue;

        case NType::NODE_7_LEAF:
        case NType::NODE_15_LEAF:
        case NType::NODE_256_LEAF:
            // 行ID唯一，直接插入
            Node::InsertChild(art, active_node, active_key[Prefix::ROW_ID_COUNT]);
            return ARTConflictType::NO_CONFLICT;

        case NType::NODE_4:
        case NType::NODE_16:
        case NType::NODE_48:
        case NType::NODE_256: {
            auto child = active_node.GetChildMutable(art, active_key[depth]);
            if (child) {
                active_node_ref = *child;
                depth++;
                continue;
            }
            InsertIntoNode(art, active_node, key, row_id, depth, status);
            return ARTConflictType::NO_CONFLICT;
        }

        case NType::PREFIX: {
            Prefix prefix(art, active_node, true);
            for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
                if (prefix.data[i] != active_key[depth]) {
                    // 前缀不匹配，需要分裂
                    InsertIntoPrefix(art, active_node_ref, active_key,
                                     row_id, i, depth, status);
                    return ARTConflictType::NO_CONFLICT;
                }
                depth++;
            }
            active_node_ref = *prefix.ptr;
            continue;
        }
        }
    }
    throw InternalException("node without metadata in ARTOperator::Insert");
}
```

### 4.3.2 插入到内联叶子

当插入位置已有内联叶子时：

```cpp
static ARTConflictType InsertIntoInlined(
    ArenaAllocator &arena, ART &art, Node &node,
    const ARTKey &key, const ARTKey &row_id, const idx_t depth,
    const GateStatus status, optional_ptr<ART> delete_art,
    const IndexAppendMode append_mode) {

    Node row_id_node;
    Leaf::New(row_id_node, row_id.GetRowId());

    // 非唯一索引：合并两个叶子
    if (!art.IsUnique() || append_mode == IndexAppendMode::INSERT_DUPLICATES) {
        Leaf::MergeInlined(arena, art, node, row_id_node, status, depth);
        return ARTConflictType::NO_CONFLICT;
    }

    // 唯一索引：检查是否为事务更新
    if (!delete_art) {
        if (append_mode == IndexAppendMode::IGNORE_DUPLICATES) {
            return ARTConflictType::NO_CONFLICT;
        }
        return ARTConflictType::CONSTRAINT;
    }

    // 在删除索引中查找
    auto delete_leaf = Lookup(*delete_art, delete_art->tree, key, 0);
    if (!delete_leaf) {
        return ARTConflictType::CONSTRAINT;
    }

    // 检查行ID是否匹配（同一事务的DELETE+INSERT）
    auto deleted_row_id = delete_leaf->GetRowId();
    auto this_row_id = node.GetRowId();
    if (deleted_row_id != this_row_id) {
        return ARTConflictType::CONSTRAINT;
    }

    // 允许事务内更新
    Leaf::MergeInlined(arena, art, node, row_id_node, status, depth);
    return ARTConflictType::NO_CONFLICT;
}
```

### 4.3.3 插入到内部节点

```cpp
static void InsertIntoNode(
    ART &art, Node &node, const ARTKey &key,
    const ARTKey &row_id, const idx_t depth, const GateStatus status) {

    if (status == GateStatus::GATE_SET) {
        // 嵌套ART内部：直接创建内联叶子
        Node row_id_node;
        Leaf::New(row_id_node, row_id.GetRowId());
        Node::InsertChild(art, node, row_id[depth], row_id_node);
        return;
    }

    Node leaf;
    reference<Node> leaf_ref(leaf);

    // 创建剩余键的前缀
    if (depth + 1 < key.len) {
        auto count = key.len - depth - 1;
        Prefix::New(art, leaf_ref, key, depth + 1, count);
    }

    // 创建并插入内联叶子
    Leaf::New(leaf_ref, row_id.GetRowId());
    Node::InsertChild(art, node, key[depth], leaf);
}
```

### 4.3.4 前缀分裂

当新键与现有前缀部分匹配时：

```cpp
static void InsertIntoPrefix(
    ART &art, reference<Node> &node_ref, const ARTKey &key,
    const ARTKey &row_id, const idx_t pos, const idx_t depth,
    const GateStatus status) {

    const auto cast_pos = UnsafeNumericCast<uint8_t>(pos);
    const auto byte = Prefix::GetByte(art, node_ref, cast_pos);

    // 分裂前缀
    Node child;
    const auto split_status = Prefix::Split(art, node_ref, child, cast_pos);

    // 创建新的 Node4
    Node4::New(art, node_ref);
    node_ref.get().SetGateStatus(split_status);

    // 插入原有子节点和新节点
    Node4::InsertChild(art, node_ref, byte, child);
    InsertIntoNode(art, node_ref, key, row_id, depth, status);
}
```

**分裂示例**：

```
插入 "help" 到 prefix "hello":

原始:                          分裂后:
┌─────────────────┐           ┌─────────────────┐
│ PREFIX: "hello" │           │ PREFIX: "hel"   │
└────────┬────────┘           └────────┬────────┘
         │                             │
    [LEAF: r1]                    ┌────┴────┐
                                  │  Node4  │
                                  │ 'l' 'p' │
                                  └──┬───┬──┘
                                     │   │
                            ┌────────┘   └────────┐
                            ▼                     ▼
                    ┌───────────────┐     ┌───────────────┐
                    │ PREFIX: "o"   │     │ LEAF: r2      │
                    └───────┬───────┘     │ (help)        │
                            │             └───────────────┘
                     [LEAF: r1]
                     (hello)
```

---

## 4.4 删除操作（Delete）

### 4.4.1 删除算法概述

删除操作需要处理节点合并和前缀压缩：

```cpp
static void Delete(ART &art, Node &node,
                   const ARTKey &key, const ARTKey &row_id) {
    Node empty;
    reference<Node> greatgrandparent(empty);
    reference<Node> grandparent(empty);
    reference<Node> parent(node);
    reference<Node> current(node);
    reference<const ARTKey> current_key(key);

    idx_t grandparent_depth = 0;
    idx_t parent_depth = 0;
    idx_t depth = 0;
    auto status = GateStatus::GATE_NOT_SET;
    auto passed_node = false;

    while (current.get().HasMetadata()) {
        // 进入门控
        if (status == GateStatus::GATE_NOT_SET &&
            current.get().GetGateStatus() == GateStatus::GATE_SET) {
            status = GateStatus::GATE_SET;
            current_key = row_id;
            depth = 0;
            continue;
        }

        switch (current.get().GetType()) {
        case NType::LEAF_INLINED: {
            if (current.get().GetRowId() != row_id.GetRowId()) {
                return;  // 行ID不匹配
            }
            // 删除节点，可能触发压缩
            if (!passed_node && parent.get().GetType() == NType::PREFIX) {
                Node::FreeTree(art, parent);
                return;
            }
            Node::DeleteChild(art, grandparent, greatgrandparent,
                              current_key.get()[grandparent_depth], status, row_id);
            return;
        }

        case NType::PREFIX: {
            // 更新祖先引用
            greatgrandparent = grandparent;
            grandparent = parent;
            parent = current;
            grandparent_depth = parent_depth;
            parent_depth = depth;

            // 遍历前缀链
            while (current.get().GetType() == NType::PREFIX) {
                Prefix prefix(art, current, true);
                for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
                    if (prefix.data[i] != current_key.get()[depth]) {
                        return;  // 键不存在
                    }
                    depth++;
                }
                current = *prefix.ptr;
                if (current.get().GetGateStatus() == GateStatus::GATE_SET) {
                    break;
                }
            }
            break;
        }

        case NType::NODE_4:
        case NType::NODE_16:
        case NType::NODE_48:
        case NType::NODE_256: {
            passed_node = true;
            // 更新祖先引用
            greatgrandparent = grandparent;
            grandparent = parent;
            parent = current;
            grandparent_depth = parent_depth;
            parent_depth = depth;

            auto child = current.get().GetChildMutable(art, current_key.get()[depth]);
            if (!child) {
                return;  // 键不存在
            }
            current = *child;
            depth++;
            break;
        }

        case NType::NODE_7_LEAF:
        case NType::NODE_15_LEAF:
        case NType::NODE_256_LEAF: {
            const auto byte = current_key.get()[depth];
            if (current.get().HasByte(art, byte)) {
                Node::DeleteChild(art, current, parent, byte, status, row_id);
            }
            return;
        }
        }
    }
}
```

### 4.4.2 删除后的节点压缩

删除可能导致节点只有一个子节点，需要与前缀合并：

```
删除 "help" 后:

删除前:                        删除后（压缩）:
┌─────────────────┐           ┌─────────────────┐
│ PREFIX: "hel"   │           │ PREFIX: "hello" │
└────────┬────────┘           └────────┬────────┘
         │                             │
    ┌────┴────┐                  [LEAF: r1]
    │  Node4  │
    │ 'l' 'p' │
    └──┬───┬──┘
       │   │
       │   └──→ [LEAF: r2] ← 被删除
       │
       ▼
┌───────────────┐
│ PREFIX: "o"   │
└───────┬───────┘
        │
  [LEAF: r1]
```

---

## 4.5 索引构建（ARTBuilder）

### 4.5.1 ARTBuilder 设计

`ARTBuilder` 用于从已排序的键值对批量构建 ART：

```cpp
// src/include/duckdb/execution/index/art/art_builder.hpp
class ARTBuilder {
public:
    ARTBuilder(ArenaAllocator &arena, ART &art,
               const unsafe_vector<ARTKey> &keys,
               const unsafe_vector<ARTKey> &row_ids)
        : arena(arena), art(art), keys(keys), row_ids(row_ids) {}

    //! 初始化构建器
    void Init(Node &node, const idx_t end) {
        s.emplace(node, 0, end, 0);
    }

    //! 执行构建
    ARTConflictType Build();

private:
    struct NodeEntry {
        Node &node;
        idx_t start;   // 键范围起始
        idx_t end;     // 键范围结束
        idx_t depth;   // 当前深度
    };

    ArenaAllocator &arena;
    ART &art;
    const unsafe_vector<ARTKey> &keys;
    const unsafe_vector<ARTKey> &row_ids;
    stack<NodeEntry> s;
};
```

### 4.5.2 构建算法

```cpp
// src/execution/index/art/art_builder.cpp
ARTConflictType ARTBuilder::Build() {
    while (!s.empty()) {
        auto entry = s.top();
        s.pop();

        auto &start = keys[entry.start];
        auto &end = keys[entry.end];

        // 1. 计算公共前缀长度
        auto prefix_depth = entry.depth;
        while (start.len != entry.depth && start.ByteMatches(end, entry.depth)) {
            entry.depth++;
        }

        // 2. 到达叶子：所有字节都匹配
        if (start.len == entry.depth) {
            auto row_id_count = entry.end - entry.start + 1;

            // 唯一索引检查
            if (art.IsUnique() && row_id_count != 1) {
                return ARTConflictType::CONSTRAINT;
            }

            // 创建前缀
            reference<Node> ref(entry.node);
            auto count = UnsafeNumericCast<uint8_t>(start.len - prefix_depth);
            Prefix::New(art, ref, start, prefix_depth, count);

            // 单行：内联叶子
            if (row_id_count == 1) {
                Leaf::New(ref, row_ids[entry.start].GetRowId());
                continue;
            }

            // 多行：创建嵌套 ART
            for (idx_t i = entry.start; i < entry.start + row_id_count; i++) {
                ARTOperator::Insert(arena, art, ref, row_ids[i], 0,
                                    row_ids[i], GateStatus::GATE_SET,
                                    nullptr, IndexAppendMode::DEFAULT);
            }
            ref.get().SetGateStatus(GateStatus::GATE_SET);
            continue;
        }

        // 3. 创建公共前缀
        reference<Node> ref(entry.node);
        auto prefix_length = entry.depth - prefix_depth;
        Prefix::New(art, ref, start, prefix_depth, prefix_length);

        // 4. 找到所有不同的子节点字节
        vector<idx_t> child_offsets;
        child_offsets.emplace_back(entry.start);
        for (idx_t i = entry.start + 1; i <= entry.end; i++) {
            if (keys[i - 1].data[entry.depth] != keys[i].data[entry.depth]) {
                child_offsets.emplace_back(i);
            }
        }

        // 5. 创建内部节点并递归构建子树
        Node::New(art, ref, Node::GetNodeType(child_offsets.size()));
        auto start_offset = child_offsets[0];

        for (idx_t i = 1; i <= child_offsets.size(); i++) {
            auto child_byte = keys[start_offset].data[entry.depth];
            Node::InsertChild(art, ref, child_byte);
            auto child = ref.get().GetChildMutable(art, child_byte, true);
            auto end_offset = i != child_offsets.size()
                              ? child_offsets[i] - 1 : entry.end;
            s.emplace(*child, start_offset, end_offset, entry.depth + 1);
            start_offset = end_offset + 1;
        }
    }

    return ARTConflictType::NO_CONFLICT;
}
```

### 4.5.3 构建流程图

```
输入: 已排序的键
["hello", "help", "helper", "world"]

构建过程:
┌─────────────────────────────────────────────────────────────────────┐
│ Step 1: 处理 ["hello", "help", "helper", "world"]                   │
│   公共前缀: "" (空)                                                  │
│   分裂字节: 'h'(0-2), 'w'(3)                                        │
│   创建 Node4: ['h', 'w']                                            │
├─────────────────────────────────────────────────────────────────────┤
│ Step 2: 处理 ['h'] → ["hello", "help", "helper"]                    │
│   公共前缀: "hel"                                                    │
│   分裂字节: 'l'(0), 'p'(1-2)                                        │
│   创建 PREFIX "hel" + Node4 ['l', 'p']                              │
├─────────────────────────────────────────────────────────────────────┤
│ Step 3: 处理 ['l'] → ["hello"]                                      │
│   创建 PREFIX "lo" + LEAF                                           │
├─────────────────────────────────────────────────────────────────────┤
│ Step 4: 处理 ['p'] → ["help", "helper"]                             │
│   公共前缀: ""                                                       │
│   分裂字节: 后续继续...                                              │
└─────────────────────────────────────────────────────────────────────┘

最终树结构:
            [Node4: h,w]
           /            \
    [PREFIX: "hel"]   [PREFIX: "world"]
          |                  |
    [Node4: l,p]          [LEAF]
       /      \
  [PREFIX:o]  [PREFIX: ""]
     |           |
   [LEAF]    [Node4: ...]
   hello        ...
```

---

## 4.6 索引合并（ARTMerger）

### 4.6.1 ARTMerger 设计

`ARTMerger` 用于合并两个 ART 索引，常用于并行构建后的索引合并：

```cpp
// src/include/duckdb/execution/index/art/art_merger.hpp
class ARTMerger {
public:
    ARTMerger(ArenaAllocator &arena, ART &art)
        : arena(arena), art(art) {}

    //! 初始化合并
    void Init(Node &left, Node &right);

    //! 执行合并
    ARTConflictType Merge();

private:
    struct NodeEntry {
        Node &left;       // 左节点（合并目标）
        Node &right;      // 右节点（被合并）
        GateStatus status;
        idx_t depth;
    };

    ArenaAllocator &arena;
    ART &art;
    stack<NodeEntry> s;

    void Emplace(Node &left, Node &right, const GateStatus parent_status, const idx_t depth);
    ARTConflictType MergeNodeAndInlined(NodeEntry &entry);
    void MergeLeaves(NodeEntry &entry);
    void MergeNodes(NodeEntry &entry);
    void MergeNodeAndPrefix(Node &node, Node &prefix, ...);
    void MergePrefixes(NodeEntry &entry);
};
```

### 4.6.2 合并算法

```cpp
// src/execution/index/art/art_merger.cpp
ARTConflictType ARTMerger::Merge() {
    while (!s.empty()) {
        auto entry = s.top();
        s.pop();

        const auto left_type = entry.left.GetType();
        const auto right_type = entry.right.GetType();

        // 唯一性约束检查
        const auto duplicate_key =
            right_type == NType::LEAF_INLINED ||
            entry.right.GetGateStatus() == GateStatus::GATE_SET;
        if (art.IsUnique() && duplicate_key) {
            return ARTConflictType::CONSTRAINT;
        }

        // 情况1: 两个内联叶子
        if (left_type == NType::LEAF_INLINED) {
            D_ASSERT(right_type == NType::LEAF_INLINED);
            Leaf::MergeInlined(arena, art, entry.left, entry.right,
                               entry.status, entry.depth);
            continue;
        }

        // 情况2: 节点 + 内联叶子
        if (right_type == NType::LEAF_INLINED) {
            auto result = MergeNodeAndInlined(entry);
            if (result != ARTConflictType::NO_CONFLICT) {
                return result;
            }
            continue;
        }

        // 情况3: 两个叶子节点
        if (entry.right.IsLeafNode()) {
            D_ASSERT(entry.left.IsLeafNode());
            MergeLeaves(entry);
            continue;
        }

        // 情况4: 两个内部节点
        if (entry.left.IsNode() && entry.right.IsNode()) {
            MergeNodes(entry);
            continue;
        }

        // 情况5: 前缀相关
        D_ASSERT(right_type == NType::PREFIX);
        if (left_type == NType::PREFIX) {
            MergePrefixes(entry);
            continue;
        }
        MergeNodeAndPrefix(entry.left, entry.right, entry.status, entry.depth);
    }

    return ARTConflictType::NO_CONFLICT;
}
```

### 4.6.3 节点合并策略

**合并两个内部节点**：

```cpp
void ARTMerger::MergeNodes(NodeEntry &entry) {
    // 将较小的节点合并到较大的节点
    if (entry.left.GetType() < entry.right.GetType()) {
        swap(entry.left, entry.right);
    }

    // 提取右节点的所有子节点
    auto children = ExtractChildren(entry.right);
    Node::FreeNode(art, entry.right);

    // 合并子节点
    vector<idx_t> remaining;
    for (idx_t i = 0; i < children.bytes.size(); i++) {
        const auto byte = children.bytes[i];
        auto child = entry.left.GetChildMutable(art, byte);

        if (!child) {
            // 左节点无此子节点，直接插入
            Node::InsertChild(art, entry.left, byte, children.children[i]);
            continue;
        }
        // 两边都有，需要递归合并
        remaining.emplace_back(i);
    }

    // 递归处理需要合并的子节点
    for (idx_t i = 0; i < remaining.size(); i++) {
        const auto byte = children.bytes[remaining[i]];
        auto &right_child = children.children[remaining[i]];
        auto child = entry.left.GetChildMutable(art, byte);
        Emplace(*child, right_child, entry.status, entry.depth + 1);
    }
}
```

### 4.6.4 合并示意图

```
合并两个 ART:

左树 (Local):              右树 (Global):
    [Node4]                    [Node4]
     'a','b'                   'b','c'
    /     \                   /     \
[LEAF1] [LEAF2]          [LEAF3] [LEAF4]

                    ↓ 合并

              [Node4]
             'a','b','c'
            /    |    \
      [LEAF1] [merged] [LEAF4]
                 |
            ┌────┴────┐
            合并 LEAF2 和 LEAF3
```

---

## 4.7 Vacuum 垃圾回收

### 4.7.1 触发条件

当分配器碎片化程度超过 10% 时触发 Vacuum：

```cpp
// FixedSizeAllocator
static constexpr uint8_t VACUUM_THRESHOLD = 10;

bool FixedSizeAllocator::InitializeVacuum() {
    idx_t free_count = 0;
    for (auto &[id, buffer] : buffers) {
        free_count += buffer->GetFreeCount();
    }

    auto total_capacity = buffers.size() * available_segments_per_buffer;
    if (free_count * 100 / total_capacity < VACUUM_THRESHOLD) {
        return false;  // 不需要回收
    }

    // 标记待回收的缓冲区
    for (auto &[id, buffer] : buffers) {
        if (buffer->CanVacuum()) {
            vacuum_buffers.insert(id);
        }
    }
    return !vacuum_buffers.empty();
}
```

### 4.7.2 ART Vacuum 实现

```cpp
// ART::Vacuum
void ART::Vacuum(IndexLock &state) {
    // 1. 初始化所有分配器的回收状态
    bool needs_vacuum = false;
    for (auto &allocator : *allocators) {
        if (allocator && allocator->InitializeVacuum()) {
            needs_vacuum = true;
        }
    }

    if (!needs_vacuum) {
        return;
    }

    // 2. 遍历整棵树，迁移需要回收的节点
    VacuumTree(tree);

    // 3. 完成回收，释放空闲缓冲区
    for (auto &allocator : *allocators) {
        if (allocator) {
            allocator->FinalizeVacuum();
        }
    }
}

void ART::VacuumTree(Node &node) {
    // 递归遍历并迁移节点
    if (!node.HasMetadata()) {
        return;
    }

    // 检查是否需要迁移
    auto &allocator = Node::GetAllocator(*this, node.GetType());
    if (allocator.NeedsVacuum(node)) {
        node = allocator.VacuumPointer(node);
    }

    // 递归处理子节点
    // ... (根据节点类型遍历子节点)
}
```

### 4.7.3 Vacuum 流程图

```
Vacuum 流程:

┌─────────────────────────────────────────────────────────────────────┐
│ Phase 1: 初始化                                                      │
│   • 统计每个分配器的空闲段数量                                        │
│   • 判断是否超过 10% 阈值                                            │
│   • 标记需要回收的缓冲区                                              │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 2: 节点迁移                                                    │
│   遍历树中每个节点:                                                   │
│   • 检查节点是否在待回收缓冲区中                                       │
│   • 如果是，分配新位置并复制数据                                       │
│   • 更新父节点的指针                                                  │
│   • 释放旧位置                                                       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Phase 3: 完成回收                                                    │
│   • 释放完全空闲的缓冲区                                              │
│   • 更新分配器元数据                                                  │
│   • 清除 vacuum_buffers 集合                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4.8 索引创建物理算子

### 4.8.1 PhysicalCreateARTIndex

`PhysicalCreateARTIndex` 是创建索引的物理执行算子：

```cpp
// src/execution/operator/schema/physical_create_art_index.cpp
class PhysicalCreateARTIndex : public PhysicalOperator {
public:
    PhysicalCreateARTIndex(/* params */);

    //! 获取全局 Sink 状态
    unique_ptr<GlobalSinkState> GetGlobalSinkState(ClientContext &context) const;

    //! 获取线程本地 Sink 状态
    unique_ptr<LocalSinkState> GetLocalSinkState(ExecutionContext &context) const;

    //! 处理数据块
    SinkResultType Sink(ExecutionContext &context, DataChunk &chunk,
                        OperatorSinkInput &input) const;

    //! 合并本地索引到全局索引
    SinkCombineResultType Combine(ExecutionContext &context,
                                  OperatorSinkCombineInput &input) const;

    //! 完成索引创建
    SinkFinalizeType Finalize(Pipeline &pipeline, Event &event,
                              ClientContext &context,
                              OperatorSinkFinalizeInput &input) const;
};
```

### 4.8.2 并行构建流程

```cpp
// 全局状态
class CreateARTIndexGlobalSinkState : public GlobalSinkState {
public:
    unique_ptr<BoundIndex> global_index;
};

// 本地状态
class CreateARTIndexLocalSinkState : public LocalSinkState {
public:
    unique_ptr<BoundIndex> local_index;
    ArenaAllocator arena_allocator;
    DataChunk key_chunk;
    unsafe_vector<ARTKey> keys;
    unsafe_vector<ARTKey> row_ids;
};

// 排序构建
SinkResultType PhysicalCreateARTIndex::SinkSorted(OperatorSinkInput &input) const {
    auto &l_state = input.local_state.Cast<CreateARTIndexLocalSinkState>();

    // 为当前 chunk 构建临时 ART
    auto art = make_uniq<ART>(...);
    if (art->Build(l_state.keys, l_state.row_ids, l_state.key_chunk.size())
        != ARTConflictType::NO_CONFLICT) {
        throw ConstraintException("Data contains duplicates on indexed column(s)");
    }

    // 合并到本地索引
    if (!l_state.local_index->MergeIndexes(*art)) {
        throw ConstraintException("Data contains duplicates on indexed column(s)");
    }

    return SinkResultType::NEED_MORE_INPUT;
}

// 合并本地索引到全局索引
SinkCombineResultType PhysicalCreateARTIndex::Combine(...) const {
    auto &g_state = input.global_state.Cast<CreateARTIndexGlobalSinkState>();
    auto &l_state = input.local_state.Cast<CreateARTIndexLocalSinkState>();

    if (!g_state.global_index->MergeIndexes(*l_state.local_index)) {
        throw ConstraintException("Data contains duplicates on indexed column(s)");
    }

    return SinkCombineResultType::FINISHED;
}

// 完成构建
SinkFinalizeType PhysicalCreateARTIndex::Finalize(...) const {
    auto &state = input.global_state.Cast<CreateARTIndexGlobalSinkState>();

    // 清理和验证
    state.global_index->Vacuum();
    state.global_index->Verify();
    state.global_index->VerifyAllocations();

    // 添加到存储层
    storage.AddIndex(std::move(state.global_index));

    return SinkFinalizeType::READY;
}
```

### 4.8.3 并行构建架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                       CREATE INDEX 并行执行                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Thread 1              Thread 2              Thread 3               │
│   ┌───────────┐        ┌───────────┐        ┌───────────┐           │
│   │Local ART 1│        │Local ART 2│        │Local ART 3│           │
│   │ (排序构建) │        │ (排序构建) │        │ (排序构建) │           │
│   └─────┬─────┘        └─────┬─────┘        └─────┬─────┘           │
│         │                    │                    │                  │
│         └─────────────┬──────┴────────────────────┘                  │
│                       │ Combine (串行合并)                            │
│                       ▼                                              │
│               ┌───────────────┐                                      │
│               │  Global ART   │                                      │
│               │   (最终索引)   │                                      │
│               └───────┬───────┘                                      │
│                       │ Finalize                                     │
│                       ▼                                              │
│               ┌───────────────┐                                      │
│               │ Vacuum + Verify│                                     │
│               │ AddIndex()     │                                     │
│               └───────────────┘                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 4.9 本章小结

本章详细分析了 DuckDB ART 索引的核心操作：

1. **查找操作**：从根节点遍历到叶子，处理前缀匹配和子节点查找。`LookupInLeaf` 支持嵌套叶子的行 ID 查找。

2. **插入操作**：处理空树、内联叶子、前缀分裂和内部节点插入等场景。支持事务内的 DELETE+INSERT 更新。

3. **删除操作**：维护祖先引用链以支持节点压缩。删除后可能触发 Node4 到前缀的合并。

4. **索引构建（ARTBuilder）**：使用栈式深度优先遍历从已排序键构建 ART，批量处理公共前缀。

5. **索引合并（ARTMerger）**：支持并行构建后的索引合并，处理内联叶子、叶子节点、内部节点和前缀的各种合并场景。

6. **Vacuum 回收**：当碎片超过 10% 阈值时触发，遍历树迁移节点到新位置，释放空闲缓冲区。

7. **物理算子**：`PhysicalCreateARTIndex` 实现并行索引构建，每个线程构建本地 ART 后合并到全局索引。

---

## 4.10 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/execution/index/art/art_operator.hpp` | 核心操作实现 |
| `src/include/duckdb/execution/index/art/art_builder.hpp` | 索引构建器定义 |
| `src/execution/index/art/art_builder.cpp` | 索引构建器实现 |
| `src/include/duckdb/execution/index/art/art_merger.hpp` | 索引合并器定义 |
| `src/execution/index/art/art_merger.cpp` | 索引合并器实现 |
| `src/execution/index/art/art.cpp` | ART 主类实现 |
| `src/execution/operator/schema/physical_create_art_index.cpp` | 物理算子实现 |
| `src/execution/index/fixed_size_allocator.cpp` | 分配器和 Vacuum |
