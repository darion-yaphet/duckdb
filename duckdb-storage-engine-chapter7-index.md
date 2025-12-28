# DuckDB 存储引擎深度解析：第七章 索引系统

## 7.1 索引系统概述

DuckDB 的索引系统采用 **ART (Adaptive Radix Tree)** 作为核心数据结构，这是一种针对内存优化的基数树变体。与传统的 B+ 树相比，ART 具有更好的缓存局部性和空间效率。

```
┌─────────────────────────────────────────────────────────────────┐
│                    DuckDB 索引系统架构                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  IndexCatalogEntry                       │   │
│  │           (索引目录条目 - 元数据管理)                     │   │
│  └─────────────────────────────────────────────────────────┘   │
│                            │                                    │
│                            ▼                                    │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                       Index (基类)                        │   │
│  │              column_ids, table_io_manager                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                    ┌───────┴───────┐                           │
│                    ▼               ▼                           │
│  ┌──────────────────────┐  ┌──────────────────────┐           │
│  │    UnboundIndex      │  │     BoundIndex       │           │
│  │  (未绑定索引-WAL回放) │  │   (已绑定索引-运行时) │           │
│  └──────────────────────┘  └──────────────────────┘           │
│                                     │                          │
│                                     ▼                          │
│              ┌──────────────────────────────────────────┐     │
│              │                   ART                     │     │
│              │     (Adaptive Radix Tree - 自适应基数树)   │     │
│              │                                          │     │
│              │  ┌─────────────────────────────────────┐ │     │
│              │  │         节点类型 (NType)             │ │     │
│              │  ├─────────────────────────────────────┤ │     │
│              │  │ Node4   │ Node16  │ Node48 │ Node256│ │     │
│              │  │ Prefix  │ Leaf    │ LeafNodes       │ │     │
│              │  └─────────────────────────────────────┘ │     │
│              │                                          │     │
│              │  ┌─────────────────────────────────────┐ │     │
│              │  │      FixedSizeAllocator (×9)        │ │     │
│              │  │         (节点内存分配器)             │ │     │
│              │  └─────────────────────────────────────┘ │     │
│              └──────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.1.1 核心设计理念

ART 索引的核心设计理念：

1. **自适应节点大小**：根据子节点数量动态选择节点类型 (4/16/48/256)
2. **路径压缩**：使用 Prefix 节点压缩单路径
3. **内联叶子**：单行 ID 直接内联到指针中，无需额外分配
4. **Gate 机制**：支持重复键值（用于非唯一索引）

### 7.1.2 支持的数据类型

```cpp
// src/execution/index/art/art.cpp:54-74
for (idx_t i = 0; i < types.size(); i++) {
    switch (types[i]) {
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
        break;
    default:
        throw InvalidTypeException(logical_types[i], "Invalid type for index key.");
    }
}
```

---

## 7.2 索引类层次结构

### 7.2.1 Index 基类

`Index` 是所有索引的抽象基类：

```cpp
// src/include/duckdb/storage/index.hpp
class Index {
protected:
    //! 索引列的列 ID
    vector<column_t> column_ids;
    //! 列 ID 集合（用于快速查找）
    unordered_set<column_t> column_id_set;

public:
    //! 表 I/O 管理器引用
    TableIOManager &table_io_manager;
    //! 所属数据库
    AttachedDatabase &db;

    //! 是否已绑定
    virtual bool IsBound() const = 0;
    //! 获取索引类型名称
    virtual const string &GetIndexType() const = 0;
    //! 获取索引名称
    virtual const string &GetIndexName() const = 0;
    //! 获取约束类型
    virtual IndexConstraintType GetConstraintType() const = 0;

    //! 约束类型判断
    bool IsUnique() const;
    bool IsPrimary() const;
    bool IsForeign() const;

    //! 提交删除操作
    virtual void CommitDrop() = 0;
};
```

### 7.2.2 BoundIndex 已绑定索引

`BoundIndex` 代表运行时的索引，包含完整的操作接口：

```cpp
// src/include/duckdb/execution/index/bound_index.hpp
class BoundIndex : public Index {
public:
    //! 物理类型列表
    vector<PhysicalType> types;
    //! 逻辑类型列表
    vector<LogicalType> logical_types;
    //! 索引名称
    string name;
    //! 索引类型 (如 "ART")
    string index_type;
    //! 约束类型
    IndexConstraintType index_constraint_type;
    //! 未绑定的索引表达式
    vector<unique_ptr<Expression>> unbound_expressions;

    //! 核心操作接口
    virtual ErrorData Append(IndexLock &l, DataChunk &chunk, Vector &row_ids) = 0;
    virtual void Delete(IndexLock &state, DataChunk &entries, Vector &row_ids) = 0;
    virtual ErrorData Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids) = 0;
    virtual bool MergeIndexes(IndexLock &state, BoundIndex &other_index) = 0;
    virtual void Vacuum(IndexLock &l) = 0;
    virtual idx_t GetInMemorySize(IndexLock &state) = 0;

protected:
    //! 索引互斥锁
    mutex lock;
    //! 表达式执行器
    ExpressionExecutor executor;
};
```

### 7.2.3 UnboundIndex 未绑定索引

`UnboundIndex` 用于 WAL 回放期间缓冲索引操作：

```cpp
// src/include/duckdb/execution/index/unbound_index.hpp
enum class BufferedIndexReplay : uint8_t {
    INSERT_ENTRY = 0,
    DEL_ENTRY = 1
};

struct BufferedIndexReplays {
    vector<ReplayRange> ranges;              // 操作区间
    unique_ptr<ColumnDataCollection> buffered_inserts;  // 缓冲的插入
    unique_ptr<ColumnDataCollection> buffered_deletes;  // 缓冲的删除
};

class UnboundIndex final : public Index {
private:
    unique_ptr<CreateInfo> create_info;      // 创建信息
    IndexStorageInfo storage_info;           // 存储信息
    BufferedIndexReplays buffered_replays;   // 缓冲的回放操作
    vector<StorageIndex> mapped_column_ids;  // 列映射

public:
    bool IsBound() const override { return false; }
    void BufferChunk(DataChunk &chunk, Vector &row_ids,
                     const vector<StorageIndex> &ids,
                     BufferedIndexReplay replay_type);
};
```

### 7.2.4 IndexCatalogEntry 索引目录条目

```cpp
// src/include/duckdb/catalog/catalog_entry/index_catalog_entry.hpp
class IndexCatalogEntry : public StandardEntry {
public:
    string sql;                             // CREATE INDEX SQL 语句
    case_insensitive_map_t<Value> options;  // 索引选项

    string index_type;                      // 索引类型 (ART, B+-tree, etc.)
    IndexConstraintType index_constraint_type;
    vector<column_t> column_ids;            // 索引列 ID
    vector<unique_ptr<ParsedExpression>> expressions;  // 索引表达式
};
```

---

## 7.3 ART 自适应基数树

### 7.3.1 ART 核心结构

```cpp
// src/include/duckdb/execution/index/art/art.hpp
class ART : public BoundIndex {
public:
    static constexpr const char *TYPE_NAME = "ART";
    static constexpr uint8_t ALLOCATOR_COUNT = 9;    // 9 种节点类型的分配器
    static constexpr idx_t MAX_KEY_LEN = 8192;       // 最大键长度

    //! 树的根节点
    Node tree = Node();

    //! 固定大小分配器数组 (每种节点类型一个)
    shared_ptr<array<unsafe_unique_ptr<FixedSizeAllocator>, ALLOCATOR_COUNT>> allocators;

    //! 是否拥有数据
    bool owns_data;
    //! 是否验证最大键长度
    bool verify_max_key_len;
    //! 前缀计数
    uint8_t prefix_count;
};
```

### 7.3.2 节点类型枚举

ART 使用 10 种不同的节点类型：

```cpp
// src/include/duckdb/execution/index/art/node.hpp
enum class NType : uint8_t {
    PREFIX = 1,         // 前缀节点（路径压缩）
    LEAF = 2,           // 叶子节点（已弃用）
    NODE_4 = 3,         // 4 个子节点
    NODE_16 = 4,        // 16 个子节点
    NODE_48 = 5,        // 48 个子节点
    NODE_256 = 6,       // 256 个子节点
    LEAF_INLINED = 7,   // 内联叶子（row_id 直接存储在指针中）
    NODE_7_LEAF = 8,    // 7 字节叶子节点
    NODE_15_LEAF = 9,   // 15 字节叶子节点
    NODE_256_LEAF = 10, // 256 位掩码叶子节点
};

enum class GateStatus : uint8_t {
    GATE_NOT_SET = 0,   // Gate 未设置
    GATE_SET = 1,       // Gate 已设置（处理重复键）
};
```

### 7.3.3 节点类型选择策略

```
┌─────────────────────────────────────────────────────────────────┐
│                    ART 节点类型选择                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  子节点数量          节点类型         内存占用                    │
│  ───────────────────────────────────────────────────           │
│      1-4            Node4           较小 (排序数组)             │
│      5-16           Node16          中等 (排序数组)             │
│     17-48           Node48          中等 (索引数组+子节点数组)    │
│     49-256          Node256         较大 (直接索引数组)          │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │               自动扩容和收缩                              │  │
│  │                                                          │  │
│  │    Node4 ←→ Node16 ←→ Node48 ←→ Node256                 │  │
│  │        4       16      48       256                      │  │
│  │                                                          │  │
│  │  扩容阈值: 当前容量已满                                    │  │
│  │  收缩阈值: Node48→Node16 at 12, Node256→Node48 at 36     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.4 ART 节点详解

### 7.4.1 IndexPointer 基础指针

所有 ART 节点使用 64 位紧凑指针：

```cpp
// src/include/duckdb/execution/index/index_pointer.hpp
class IndexPointer {
public:
    //! 位布局
    static constexpr idx_t SHIFT_OFFSET = 32;
    static constexpr idx_t SHIFT_METADATA = 56;
    static constexpr idx_t AND_OFFSET = 0x0000000000FFFFFF;
    static constexpr idx_t AND_BUFFER_ID = 0x00000000FFFFFFFF;
    static constexpr idx_t AND_METADATA = 0xFF00000000000000;

private:
    //! 64 位数据布局:
    //! [0-7: metadata (节点类型 + gate 状态),
    //!  8-31: offset (段内偏移),
    //!  32-63: buffer_id (缓冲区 ID)]
    idx_t data;

public:
    inline uint8_t GetMetadata() const;
    inline idx_t GetOffset() const;
    inline idx_t GetBufferId() const;
};
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   IndexPointer 位布局 (64 bits)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  63      56 55      32 31                                 0     │
│  ┌────────┬───────────┬─────────────────────────────────────┐  │
│  │metadata│  offset   │              buffer_id              │  │
│  │ 8 bits │  24 bits  │               32 bits               │  │
│  └────────┴───────────┴─────────────────────────────────────┘  │
│                                                                 │
│  metadata: [7: gate_status, 0-6: node_type (NType)]            │
│  offset:   段在缓冲区内的偏移                                    │
│  buffer_id: FixedSizeAllocator 中的缓冲区标识                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4.2 Node 类

`Node` 继承自 `IndexPointer`，添加了 ART 特定功能：

```cpp
// src/include/duckdb/execution/index/art/node.hpp
class Node : public IndexPointer {
public:
    static constexpr uint8_t AND_GATE = 0x80;           // Gate 位掩码
    static constexpr idx_t AND_ROW_ID = 0x00FFFFFFFFFFFFFF;  // Row ID 掩码

    //! 节点操作
    static void New(ART &art, Node &node, const NType type);
    static void FreeNode(ART &art, Node &node);
    static void FreeTree(ART &art, Node &node);

    //! 获取分配器
    static FixedSizeAllocator &GetAllocator(const ART &art, const NType type);

    //! 子节点操作
    void ReplaceChild(const ART &art, const uint8_t byte, const Node child) const;
    static void InsertChild(ART &art, Node &node, const uint8_t byte, const Node child);
    static void DeleteChild(ART &art, Node &node, Node &prefix, const uint8_t byte,
                            const GateStatus status, const ARTKey &row_id);

    //! 查找子节点
    const unsafe_optional_ptr<Node> GetChild(ART &art, const uint8_t byte) const;
    const unsafe_optional_ptr<Node> GetNextChild(ART &art, uint8_t &byte) const;

    //! 类型判断
    inline NType GetType() const { return NType(GetMetadata() & ~AND_GATE); }
    bool IsNode() const;       // Node4/16/48/256
    bool IsLeafNode() const;   // Node7Leaf/15Leaf/256Leaf
    bool IsAnyLeaf() const;    // 任意叶子类型

    //! Gate 状态管理
    inline GateStatus GetGateStatus() const;
    inline void SetGateStatus(const GateStatus status);

    //! Row ID 访问（用于 LEAF_INLINED）
    inline row_t GetRowId() const { return UnsafeNumericCast<row_t>(Get() & AND_ROW_ID); }
    inline void SetRowId(const row_t row_id);
};
```

### 7.4.3 BaseNode 模板

Node4 和 Node16 使用相同的模板基类：

```cpp
// src/include/duckdb/execution/index/art/base_node.hpp
template <uint8_t CAPACITY, NType TYPE>
class BaseNode {
private:
    uint8_t count;              // 当前子节点数量
    uint8_t key[CAPACITY];      // 键字节数组（排序）
    Node children[CAPACITY];    // 子节点数组

public:
    static NodeHandle<BaseNode> New(ART &art, Node &node);

    //! 子节点操作（线性查找，排序维护）
    static void ReplaceChild(BaseNode &n, const uint8_t byte, const Node child);
    static unsafe_optional_ptr<Node> GetChild(BaseNode &n, const uint8_t byte);
    static unsafe_optional_ptr<Node> GetNextChild(BaseNode &n, uint8_t &byte);
};

//! Node4: 4 个子节点
class Node4 : public BaseNode<4, NType::NODE_4> {
    static constexpr uint8_t CAPACITY = 4;
    static void InsertChild(ART &art, Node &node, const uint8_t byte, const Node child);
    static void DeleteChild(ART &art, Node &node, Node &prefix, const uint8_t byte,
                            const GateStatus status);
};

//! Node16: 16 个子节点
class Node16 : public BaseNode<16, NType::NODE_16> {
    static constexpr uint8_t CAPACITY = 16;
};
```

### 7.4.4 Node48

Node48 使用间接索引优化查找：

```cpp
// src/include/duckdb/execution/index/art/node48.hpp
class Node48 {
public:
    static constexpr uint8_t CAPACITY = 48;
    static constexpr uint8_t EMPTY_MARKER = 48;     // 空位标记
    static constexpr uint8_t SHRINK_THRESHOLD = 12; // 收缩阈值

private:
    uint8_t count;
    uint8_t child_index[256];    // 键字节 → children 索引的映射
    Node children[48];           // 子节点数组

public:
    template <class NODE>
    static unsafe_optional_ptr<Node> GetChild(NODE &n, const uint8_t byte) {
        if (n.child_index[byte] != EMPTY_MARKER) {
            return &n.children[n.child_index[byte]];
        }
        return nullptr;
    }
};
```

```
┌─────────────────────────────────────────────────────────────────┐
│                      Node48 结构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  child_index[256]:  键字节到子节点索引的映射                      │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬─────────┬───┐              │
│  │ 48│ 48│ 0 │ 48│ 1 │ 48│ 2 │ 48│  ...    │ 48│              │
│  └───┴───┴───┴───┴───┴───┴───┴───┴─────────┴───┘              │
│    0   1   2   3   4   5   6   7          255                  │
│            │       │       │                                   │
│            └───────┼───────┼───────────────────────┐           │
│                    │       │                       │           │
│  children[48]:     ▼       ▼                       ▼           │
│  ┌─────┬─────┬─────┬─────┬─────┬─────────────┬─────┐          │
│  │Node │Node │Node │ ... │ ... │    ...      │ ... │          │
│  └─────┴─────┴─────┴─────┴─────┴─────────────┴─────┘          │
│    [0]   [1]   [2]                            [47]             │
│                                                                 │
│  查找 key=4: child_index[4]=1 → children[1]                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.4.5 Node256

Node256 使用直接索引，查找时间 O(1)：

```cpp
// src/include/duckdb/execution/index/art/node256.hpp
class Node256 {
public:
    static constexpr uint16_t CAPACITY = 256;
    static constexpr uint8_t SHRINK_THRESHOLD = 36;  // 收缩阈值

private:
    uint16_t count;
    Node children[256];    // 直接使用键字节作为索引

public:
    template <class NODE>
    static unsafe_optional_ptr<Node> GetChild(NODE &n, const uint8_t byte) {
        if (n.children[byte].HasMetadata()) {
            return &n.children[byte];
        }
        return nullptr;
    }
};
```

### 7.4.6 Prefix 前缀节点

Prefix 用于路径压缩，减少树高度：

```cpp
// src/include/duckdb/execution/index/art/prefix.hpp
class Prefix {
public:
    static constexpr uint8_t ROW_ID_SIZE = sizeof(row_t);    // 8 bytes
    static constexpr uint8_t ROW_ID_COUNT = ROW_ID_SIZE - 1; // 7 bytes
    static constexpr uint8_t DEPRECATED_COUNT = 15;
    static constexpr uint8_t METADATA_SIZE = sizeof(Node) + 1;  // 9 bytes

    data_ptr_t data;    // 前缀字节数据
    Node *ptr;          // 子节点指针
    bool in_memory;

public:
    //! 创建新的前缀链
    static void New(ART &art, reference<Node> &ref, const ARTKey &key,
                    const idx_t depth, idx_t count);

    //! 拼接: parent -> prev_node4 -> child
    static void Concat(ART &art, Node &parent, Node &node4, const Node child,
                       uint8_t byte, const GateStatus node4_status, const GateStatus status);

    //! 删除前 pos 个字节
    static void Reduce(ART &art, Node &node, const idx_t pos);

    //! 在 pos 位置分裂
    static GateStatus Split(ART &art, reference<Node> &node, Node &child, const uint8_t pos);
};
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    路径压缩示例                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  未压缩:                         压缩后:                        │
│                                                                 │
│     [a]                          [abc]                          │
│      │                             │                            │
│     [b]                         [Node4]                         │
│      │                          /   \                           │
│     [c]                      [d]     [e]                        │
│      │                        │       │                         │
│   [Node4]                    ...     ...                        │
│   /    \                                                        │
│ [d]    [e]                   Prefix 节点存储 "abc"              │
│  │      │                    减少树的深度                        │
│ ...    ...                                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.5 叶子节点系统

### 7.5.1 Leaf 叶子类型

DuckDB 使用多种叶子表示方式处理不同场景：

```cpp
// src/include/duckdb/execution/index/art/leaf.hpp
class Leaf {
public:
    static constexpr NType LEAF = NType::LEAF;          // 已弃用
    static constexpr NType INLINED = NType::LEAF_INLINED;

    //! 内联单个 row_id 到 Node 指针中
    static void New(Node &node, const row_t row_id);

    //! 合并两个内联叶子（创建嵌套 ART）
    static void MergeInlined(ArenaAllocator &arena, ART &art,
                              Node &left, Node &right,
                              GateStatus status, idx_t depth);

    //! 转换为嵌套叶子 / 已弃用叶子
    static void TransformToNested(ART &art, Node &node);
    static void TransformToDeprecated(ART &art, Node &node);
};
```

### 7.5.2 叶子节点类型层次

```
┌─────────────────────────────────────────────────────────────────┐
│                    叶子节点类型                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. LEAF_INLINED (内联叶子)                                     │
│     - Row ID 直接存储在 Node 指针的低 56 位中                    │
│     - 适用于唯一键，无额外内存分配                                │
│     - 最高效的存储方式                                          │
│                                                                 │
│  2. 嵌套叶子 (Gate 机制)                                        │
│     - 处理重复键值（非唯一索引）                                  │
│     - 使用 row_id 作为键构建嵌套 ART                             │
│     - Gate 标记指示进入嵌套结构                                  │
│                                                                 │
│  3. Node7Leaf / Node15Leaf / Node256Leaf                       │
│     - 用于嵌套 ART 的最后一级                                    │
│     - 存储 row_id 的最后一个字节                                 │
│                                                                 │
│  Node7Leaf:   存储 7 个排序字节                                  │
│  Node15Leaf:  存储 15 个排序字节                                 │
│  Node256Leaf: 使用 256 位掩码                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.5.3 BaseLeaf 模板

```cpp
// src/include/duckdb/execution/index/art/base_leaf.hpp
template <uint8_t CAPACITY, NType TYPE>
class BaseLeaf {
private:
    uint8_t count;
    uint8_t key[CAPACITY];   // 存储的字节（row_id 的最后部分）

public:
    bool HasByte(const uint8_t byte) const;
    array_ptr<uint8_t> GetBytes();
    bool GetNextByte(uint8_t &byte) const;
};

//! Node7Leaf: 7 个字节
class Node7Leaf : public BaseLeaf<7, NType::NODE_7_LEAF> {
    static constexpr uint8_t CAPACITY = 7;
    static void InsertByte(ART &art, Node &node, const uint8_t byte);
    static void DeleteByte(ART &art, Node &node, Node &prefix,
                           const uint8_t byte, const ARTKey &row_id);
};

//! Node15Leaf: 15 个字节
class Node15Leaf : public BaseLeaf<15, NType::NODE_15_LEAF> {
    static constexpr uint8_t CAPACITY = 15;
};
```

### 7.5.4 Node256Leaf

```cpp
// src/include/duckdb/execution/index/art/node256_leaf.hpp
class Node256Leaf {
public:
    static constexpr uint16_t CAPACITY = 256;

private:
    uint16_t count;
    validity_t mask[256 / sizeof(validity_t)];  // 256 位掩码

public:
    static void InsertByte(ART &art, Node &node, const uint8_t byte);
    static void DeleteByte(ART &art, Node &node, const uint8_t byte);
    bool HasByte(const uint8_t byte);
    bool GetNextByte(uint8_t &byte);
};
```

---

## 7.6 ARTKey 键编码

### 7.6.1 ARTKey 结构

```cpp
// src/include/duckdb/execution/index/art/art_key.hpp
class ARTKey {
public:
    idx_t len;        // 键长度
    data_ptr_t data;  // 键数据

    //! 从值创建键
    template <class T>
    static ARTKey CreateARTKey(ArenaAllocator &allocator, T value) {
        auto data = allocator.Allocate(sizeof(value));
        Radix::EncodeData<T>(data, value);  // 使用基数编码
        return ARTKey(data, sizeof(value));
    }

    //! 拼接多列键
    void Concat(ArenaAllocator &allocator, const ARTKey &other);

    //! 获取不匹配位置
    idx_t GetMismatchPos(const ARTKey &other, const idx_t start) const;

    //! 提取 row_id
    row_t GetRowId() const;
};
```

### 7.6.2 基数编码

ART 使用基数编码确保字节序比较等价于数值比较：

```cpp
// src/common/radix.hpp
template <class T>
static inline void EncodeData(data_ptr_t data, T value) {
    // 整数: 翻转符号位
    // 浮点数: 处理 IEEE 754 表示
    // 字符串: 直接复制字节
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    基数编码示例                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INT32 编码:                                                    │
│  ───────────                                                    │
│  原值: -100   →  编码: 0x7FFFFF9C  (翻转符号位)                  │
│  原值: 0      →  编码: 0x80000000                               │
│  原值: 100    →  编码: 0x80000064                               │
│                                                                 │
│  编码后的字节序比较 = 数值比较:                                   │
│  0x7FFFFF9C < 0x80000000 < 0x80000064                          │
│  即: -100 < 0 < 100                                             │
│                                                                 │
│  VARCHAR 编码:                                                  │
│  ─────────────                                                  │
│  "abc" → [0x61, 0x62, 0x63] (直接使用 UTF-8 字节)               │
│                                                                 │
│  复合键编码:                                                     │
│  ─────────────                                                  │
│  (INT32: 10, VARCHAR: "a") → [INT32编码] + [VARCHAR编码]        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.6.3 键生成流程

```cpp
// src/execution/index/art/art.cpp
template <bool IS_NOT_NULL>
void GenerateKeysInternal(ArenaAllocator &allocator, DataChunk &input,
                          unsafe_vector<ARTKey> &keys) {
    // 生成第一列的键
    switch (input.data[0].GetType().InternalType()) {
    case PhysicalType::INT32:
        TemplatedGenerateKeys<int32_t, IS_NOT_NULL>(allocator, input.data[0], count, keys);
        break;
    case PhysicalType::VARCHAR:
        TemplatedGenerateKeys<string_t, IS_NOT_NULL>(allocator, input.data[0], count, keys);
        break;
    // ... 其他类型
    }

    // 拼接后续列（复合键）
    for (idx_t i = 1; i < input.ColumnCount(); i++) {
        ConcatenateKeys<T, IS_NOT_NULL>(allocator, input.data[i], count, keys);
    }
}
```

---

## 7.7 FixedSizeAllocator 内存分配

### 7.7.1 分配器设计

ART 为每种节点类型维护独立的固定大小分配器：

```cpp
// src/include/duckdb/execution/index/fixed_size_allocator.hpp
class FixedSizeAllocator {
public:
    static constexpr uint8_t VACUUM_THRESHOLD = 10;  // 10% 清理阈值

    BlockManager &block_manager;
    BufferManager &buffer_manager;

private:
    MemoryTag memory_tag;
    idx_t segment_size;                // 段大小（固定）

    idx_t bitmask_count;               // 位图计数
    idx_t bitmask_offset;              // 有效载荷起始偏移
    idx_t available_segments_per_buffer;  // 每个缓冲区的段数

    idx_t total_segment_count;         // 总分配段数

    unordered_map<idx_t, unique_ptr<FixedSizeBuffer>> buffers;
    unordered_set<idx_t> buffers_with_free_space;
    optional_idx buffer_with_free_space;

    unordered_set<idx_t> vacuum_buffers;  // 待清理的缓冲区

public:
    //! 分配新段
    IndexPointer New();
    //! 释放段
    void Free(const IndexPointer ptr);

    //! 获取段句柄
    SegmentHandle GetHandle(const IndexPointer ptr);

    //! 内存统计
    idx_t GetInMemorySize() const;
    idx_t GetSegmentCount() const;

    //! 清理操作
    bool InitializeVacuum();
    void FinalizeVacuum();
    IndexPointer VacuumPointer(const IndexPointer ptr);

    //! 合并另一个分配器
    void Merge(FixedSizeAllocator &other);

    //! 序列化支持
    FixedSizeAllocatorInfo GetInfo() const;
    void SerializeBuffers(PartialBlockManager &partial_block_manager);
};
```

### 7.7.2 ART 分配器配置

```cpp
// src/execution/index/art/art.cpp:89-99
// 9 种分配器，每种对应一种节点类型
array<unsafe_unique_ptr<FixedSizeAllocator>, ALLOCATOR_COUNT> allocator_array = {
    make_unsafe_uniq<FixedSizeAllocator>(prefix_size, block_manager),     // PREFIX
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Leaf), block_manager),    // LEAF (deprecated)
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node4), block_manager),   // NODE_4
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node16), block_manager),  // NODE_16
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node48), block_manager),  // NODE_48
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node256), block_manager), // NODE_256
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node7Leaf), block_manager),   // NODE_7_LEAF
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node15Leaf), block_manager),  // NODE_15_LEAF
    make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node256Leaf), block_manager), // NODE_256_LEAF
};
```

```
┌─────────────────────────────────────────────────────────────────┐
│               FixedSizeBuffer 内存布局                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┬──────────────────────────────────────────┐    │
│  │   Bitmask   │              Segments                    │    │
│  │  (位图)     │           (固定大小的段)                   │    │
│  └─────────────┴──────────────────────────────────────────┘    │
│  │<-bitmask_offset->│                                          │
│                                                                 │
│  Bitmask: 每一位标记对应段是否已分配                             │
│  Segments: 连续的固定大小内存块                                  │
│                                                                 │
│  段计算:                                                        │
│  offset = ptr.GetOffset() * segment_size + bitmask_offset      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7.8 索引操作

### 7.8.1 Insert 插入操作

```cpp
// src/execution/index/art/art.cpp
ErrorData ART::Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids,
                      IndexAppendInfo &info) {
    ArenaAllocator arena(BufferAllocator::Get(db));
    unsafe_vector<ARTKey> keys(row_count);
    unsafe_vector<ARTKey> row_id_keys(row_count);
    GenerateKeyVectors(arena, chunk, row_ids, keys, row_id_keys);

    for (idx_t i = 0; i < row_count; i++) {
        if (keys[i].Empty()) continue;  // 跳过 NULL

        conflict_type = ARTOperator::Insert(arena, *this, tree, keys[i], 0,
                                            row_id_keys[i], GateStatus::GATE_NOT_SET,
                                            delete_art, info.append_mode);
        if (conflict_type != ARTConflictType::NO_CONFLICT) {
            // 冲突处理：回滚之前的插入
            for (idx_t j = 0; j < i; j++) {
                ARTOperator::Delete(*this, tree, keys[j], row_id_keys[j]);
            }
            break;
        }
    }

    // 返回冲突错误
    if (conflict_type == ARTConflictType::CONSTRAINT) {
        return ErrorData(ConstraintException(
            "PRIMARY KEY or UNIQUE constraint violation: duplicate key"));
    }
    return ErrorData();
}
```

### 7.8.2 Delete 删除操作

```cpp
void ART::Delete(IndexLock &state, DataChunk &input, Vector &row_ids) {
    DataChunk expr_chunk;
    expr_chunk.Initialize(Allocator::DefaultAllocator(), logical_types);
    ExecuteExpressions(input, expr_chunk);

    ArenaAllocator allocator(BufferAllocator::Get(db));
    unsafe_vector<ARTKey> keys(row_count);
    unsafe_vector<ARTKey> row_id_keys(row_count);
    GenerateKeyVectors(allocator, expr_chunk, row_ids, keys, row_id_keys);

    for (idx_t i = 0; i < row_count; i++) {
        if (keys[i].Empty()) continue;
        ARTOperator::Delete(*this, tree, keys[i], row_id_keys[i]);
    }
}
```

### 7.8.3 Scan 扫描操作

```cpp
bool ART::Scan(IndexScanState &state, const idx_t max_count, set<row_t> &row_ids) {
    auto &scan_state = state.Cast<ARTIndexScanState>();
    auto key = ARTKey::CreateKey(arena_allocator, types[0], scan_state.values[0]);

    lock_guard<mutex> l(lock);

    if (scan_state.values[1].IsNull()) {
        // 单谓词扫描
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
        }
    }

    // 范围扫描
    return SearchCloseRange(key, upper_bound, left_equal, right_equal, max_count, row_ids);
}
```

### 7.8.4 Iterator 迭代器

```cpp
// src/include/duckdb/execution/index/art/iterator.hpp
class Iterator {
public:
    IteratorKey current_key;  // 当前键路径

    //! 扫描直到 upper_bound
    bool Scan(const ARTKey &upper_bound, const idx_t max_count,
              set<row_t> &row_ids, const bool equal);

    //! 找到最小值
    void FindMinimum(const Node &node);

    //! 找到下界
    bool LowerBound(const Node &node, const ARTKey &key, const bool equal);

private:
    ART &art;
    stack<IteratorEntry> nodes;     // 节点栈
    Node last_leaf = Node();        // 上次访问的叶子
    GateStatus status;              // Gate 状态
    uint8_t nested_depth = 0;       // 嵌套深度

    bool Next();
    void PopNode();
};
```

---

## 7.9 索引持久化

### 7.9.1 序列化到磁盘

```cpp
IndexStorageInfo ART::SerializeToDisk(QueryContext context,
                                       const case_insensitive_map_t<Value> &options) {
    lock_guard<mutex> guard(lock);

    // 检查是否需要转换为旧版存储格式
    auto v1_0_0_option = options.find("v1_0_0_storage");
    bool v1_0_0_storage = v1_0_0_option == options.end() ||
                          v1_0_0_option->second != Value(false);

    auto info = PrepareSerialize(options, v1_0_0_storage);

    // 写入部分块
    WritePartialBlocks(context, v1_0_0_storage);

    // 收集分配器信息
    for (idx_t i = 0; i < allocator_count; i++) {
        info.allocator_infos.push_back((*allocators)[i]->GetInfo());
    }

    return info;
}
```

### 7.9.2 序列化到 WAL

```cpp
IndexStorageInfo ART::SerializeToWAL(const case_insensitive_map_t<Value> &options) {
    auto info = PrepareSerialize(options, v1_0_0_storage);

    // 获取缓冲区信息用于 WAL
    for (idx_t i = 0; i < allocator_count; i++) {
        info.buffers.push_back((*allocators)[i]->InitSerializationToWAL());
    }

    for (idx_t i = 0; i < allocator_count; i++) {
        info.allocator_infos.push_back((*allocators)[i]->GetInfo());
    }

    return info;
}
```

### 7.9.3 反序列化

```cpp
void ART::Deserialize(const BlockPointer &pointer) {
    auto &metadata_manager = table_io_manager.GetMetadataManager();
    MetadataReader reader(metadata_manager, pointer);

    // 读取根节点
    tree = reader.Read<Node>();

    // 读取各个分配器的状态
    for (idx_t i = 0; i < DEPRECATED_ALLOCATOR_COUNT; i++) {
        (*allocators)[i]->Deserialize(metadata_manager, reader.Read<BlockPointer>());
    }
}
```

---

## 7.10 约束验证

### 7.10.1 约束类型

```cpp
enum class IndexConstraintType : uint8_t {
    NONE,           // 无约束
    UNIQUE,         // 唯一约束
    PRIMARY,        // 主键约束
    FOREIGN         // 外键约束
};
```

### 7.10.2 VerifyConstraint 验证

```cpp
void ART::VerifyConstraint(DataChunk &chunk, IndexAppendInfo &info,
                           ConflictManager &manager) {
    lock_guard<mutex> l(lock);

    ArenaAllocator arena_allocator(BufferAllocator::Get(db));
    unsafe_vector<ARTKey> keys(expr_chunk.size());
    GenerateKeys<>(arena_allocator, expr_chunk, keys);

    for (idx_t i = 0; i < chunk.size(); i++) {
        if (keys[i].Empty()) {
            if (manager.AddNull(i)) {
                conflict_idx = i;
            }
            continue;
        }

        // 查找是否存在
        auto leaf = ARTOperator::Lookup(*this, tree, keys[i], 0);
        if (!leaf) continue;

        // 验证叶子节点
        VerifyLeaf(*leaf, keys[i], delete_art, manager, conflict_idx, i);
    }

    manager.FinishLookup();

    if (conflict_idx.IsValid()) {
        auto key_name = GenerateErrorKeyName(chunk, conflict_idx.GetIndex());
        throw ConstraintException(GenerateConstraintErrorMessage(
            manager.GetVerifyExistenceType(), key_name));
    }
}
```

### 7.10.3 冲突类型

```cpp
enum class ARTConflictType : uint8_t {
    NO_CONFLICT = 0,    // 无冲突
    CONSTRAINT = 1,     // 约束冲突（重复键）
    TRANSACTION = 2     // 事务冲突（写-写冲突）
};
```

---

## 7.11 索引合并与清理

### 7.11.1 MergeIndexes 合并

```cpp
bool ART::MergeIndexes(IndexLock &state, BoundIndex &other_index) {
    auto &other_art = other_index.Cast<ART>();
    if (!other_art.tree.HasMetadata()) return true;

    if (other_art.owns_data) {
        if (tree.HasMetadata()) {
            // 调整 other 的 buffer ID
            unsafe_vector<idx_t> upper_bounds;
            InitializeMergeUpperBounds(upper_bounds);
            other_art.InitializeMerge(other_art.tree, upper_bounds);
        }

        // 合并分配器存储
        for (idx_t i = 0; i < allocators->size(); i++) {
            (*allocators)[i]->Merge(*(*other_art.allocators)[i]);
        }
    }

    // 合并 ART 结构
    if (tree.HasMetadata()) {
        ArenaAllocator arena(Allocator::Get(db));
        ARTMerger merger(arena, *this);
        merger.Init(tree, other_art.tree);
        return merger.Merge() == ARTConflictType::NO_CONFLICT;
    }

    tree = other_art.tree;
    other_art.tree.Clear();
    return true;
}
```

### 7.11.2 Vacuum 清理

```cpp
void ART::Vacuum(IndexLock &state) {
    if (!tree.HasMetadata()) {
        for (auto &allocator : *allocators) {
            allocator->Reset();
        }
        return;
    }

    // 检查哪些分配器需要清理
    unordered_set<uint8_t> indexes;
    InitializeVacuum(indexes);

    if (indexes.empty()) return;

    // 遍历树进行清理
    auto handler = [&](Node &node) {
        const auto type = node.GetType();
        const auto idx = Node::GetAllocatorIdx(type);
        auto &allocator = Node::GetAllocator(art, type);

        if (indexes.find(idx) != indexes.end() && allocator.NeedsVacuum(node)) {
            const auto status = node.GetGateStatus();
            node = allocator.VacuumPointer(node);
            node.SetMetadata(static_cast<uint8_t>(type));
            node.SetGateStatus(status);
        }
        return ARTHandlingResult::CONTINUE;
    };

    ARTScanner<ARTScanHandling::EMPLACE, Node> scanner(*this, handler, tree);
    scanner.Scan(handler);

    FinalizeVacuum(indexes);
}
```

---

## 7.12 源文件索引

| 组件 | 文件路径 |
|------|----------|
| Index 基类 | `src/include/duckdb/storage/index.hpp` |
| BoundIndex | `src/include/duckdb/execution/index/bound_index.hpp` |
| UnboundIndex | `src/include/duckdb/execution/index/unbound_index.hpp` |
| ART 主类 | `src/include/duckdb/execution/index/art/art.hpp` |
| Node 类 | `src/include/duckdb/execution/index/art/node.hpp` |
| BaseNode/Node4/Node16 | `src/include/duckdb/execution/index/art/base_node.hpp` |
| Node48 | `src/include/duckdb/execution/index/art/node48.hpp` |
| Node256 | `src/include/duckdb/execution/index/art/node256.hpp` |
| Leaf | `src/include/duckdb/execution/index/art/leaf.hpp` |
| BaseLeaf | `src/include/duckdb/execution/index/art/base_leaf.hpp` |
| Node256Leaf | `src/include/duckdb/execution/index/art/node256_leaf.hpp` |
| Prefix | `src/include/duckdb/execution/index/art/prefix.hpp` |
| ARTKey | `src/include/duckdb/execution/index/art/art_key.hpp` |
| Iterator | `src/include/duckdb/execution/index/art/iterator.hpp` |
| IndexPointer | `src/include/duckdb/execution/index/index_pointer.hpp` |
| FixedSizeAllocator | `src/include/duckdb/execution/index/fixed_size_allocator.hpp` |
| ART 实现 | `src/execution/index/art/art.cpp` |

---

## 7.13 总结

DuckDB 的索引系统基于 ART (Adaptive Radix Tree) 实现，具有以下核心特点：

```
┌─────────────────────────────────────────────────────────────────┐
│                    ART 索引核心特性                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 自适应节点大小                                               │
│     Node4 → Node16 → Node48 → Node256                          │
│     根据子节点数量自动扩容/收缩                                  │
│                                                                 │
│  2. 路径压缩                                                    │
│     Prefix 节点压缩单路径，减少树高度                            │
│                                                                 │
│  3. 内联叶子                                                    │
│     LEAF_INLINED: row_id 直接存储在 64 位指针中                 │
│     零额外内存分配                                              │
│                                                                 │
│  4. Gate 机制                                                   │
│     支持非唯一索引的重复键                                       │
│     使用嵌套 ART 存储相同键的多个 row_id                         │
│                                                                 │
│  5. 固定大小分配器                                               │
│     9 种节点类型各自独立的 FixedSizeAllocator                   │
│     高效的内存管理和 Vacuum 清理                                 │
│                                                                 │
│  6. 持久化支持                                                  │
│     支持序列化到磁盘和 WAL                                       │
│     向后兼容 v1.0.0 存储格式                                     │
│                                                                 │
│  7. 约束验证                                                    │
│     PRIMARY KEY / UNIQUE / FOREIGN KEY 支持                    │
│     事务级冲突检测                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

ART 索引相比传统 B+ 树的优势：

| 特性 | ART | B+ 树 |
|------|-----|-------|
| 缓存效率 | 高（自适应节点） | 中等 |
| 空间效率 | 高（路径压缩+内联叶子） | 中等 |
| 查找复杂度 | O(k) 键长度 | O(log n) |
| 范围扫描 | 支持 | 支持 |
| 前缀匹配 | 天然支持 | 需要额外处理 |
| 内存碎片 | 低（固定大小分配） | 可能较高 |
