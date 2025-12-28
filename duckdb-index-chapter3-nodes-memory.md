# DuckDB 索引系统深度解析 - 第三章：ART 节点类型与内存管理

本章深入分析 DuckDB ART 索引的节点类型层次结构和内存管理机制，包括自适应节点、前缀压缩、叶子节点设计以及 FixedSizeAllocator 内存分配器。

## 3.1 节点类型体系

### 3.1.1 NType 枚举定义

DuckDB 定义了 10 种节点类型，通过 `NType` 枚举表示：

```cpp
// src/include/duckdb/execution/index/art/node.hpp
enum class NType : uint8_t {
    PREFIX = 1,           // 前缀节点
    LEAF = 2,             // 叶子节点（已弃用）
    NODE_4 = 3,           // 4叉节点
    NODE_16 = 4,          // 16叉节点
    NODE_48 = 5,          // 48叉节点
    NODE_256 = 6,         // 256叉节点
    LEAF_INLINED = 7,     // 内联叶子
    NODE_7_LEAF = 8,      // 7子叶子节点
    NODE_15_LEAF = 9,     // 15子叶子节点
    NODE_256_LEAF = 10    // 256子叶子节点
};
```

这些节点类型可分为三大类：

```
┌─────────────────────────────────────────────────────────────────┐
│                      ART 节点类型分类                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  内部节点 (Internal Nodes)                               │     │
│  │  ├── Node4   (1-4 子节点)                                │     │
│  │  ├── Node16  (5-16 子节点)                               │     │
│  │  ├── Node48  (17-48 子节点)                              │     │
│  │  └── Node256 (49-256 子节点)                             │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  前缀节点 (Prefix Node)                                  │     │
│  │  └── PREFIX: 存储公共前缀字节序列                         │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐     │
│  │  叶子节点 (Leaf Nodes)                                   │     │
│  │  ├── LEAF_INLINED: 行ID内联到指针中                       │     │
│  │  ├── LEAF: 已弃用的链表叶子                               │     │
│  │  ├── NODE_7_LEAF: 7子叶子节点                            │     │
│  │  ├── NODE_15_LEAF: 15子叶子节点                          │     │
│  │  └── NODE_256_LEAF: 256子叶子节点                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3.1.2 GateStatus 门控状态

`GateStatus` 用于标识嵌套 ART 的边界：

```cpp
enum class GateStatus : uint8_t {
    GATE_NOT_SET = 0,  // 不在嵌套 ART 中
    GATE_SET = 1,      // 进入嵌套 ART（用于重复键处理）
};
```

当 ART 索引允许重复键时（非唯一索引），相同的键可能对应多个行 ID。此时使用嵌套 ART 结构：外层 ART 以原始键为索引，内层 ART 以行 ID 为键。`GATE_SET` 标记表示进入了内层嵌套结构。

---

## 3.2 Node 包装类

### 3.2.1 Node 类设计

`Node` 类继承自 `IndexPointer`，作为所有 ART 节点的统一指针接口：

```cpp
// src/include/duckdb/execution/index/art/node.hpp
class Node : public IndexPointer {
public:
    //! 门控标志位（元数据最高位）
    static constexpr uint8_t AND_GATE = 0x80;        // 二进制: 1000-0000
    static constexpr idx_t AND_ROW_ID = 0x00FFFFFFFFFFFFFF;

    //! 静态工厂方法
    static void New(ART &art, Node &node, const NType type);
    static void FreeNode(ART &art, Node &node);
    static void FreeTree(ART &art, Node &node);

    //! 获取分配器
    static FixedSizeAllocator &GetAllocator(const ART &art, const NType type);
    static uint8_t GetAllocatorIdx(const NType type);

    //! 节点引用访问
    template <class NODE>
    static inline NODE &Ref(const ART &art, const Node ptr, const NType type);

    //! 子节点操作
    void ReplaceChild(const ART &art, const uint8_t byte, const Node child) const;
    static void InsertChild(ART &art, Node &node, const uint8_t byte, const Node child);
    static void DeleteChild(ART &art, Node &node, Node &prefix, const uint8_t byte,
                            const GateStatus status, const ARTKey &row_id);

    //! 子节点查询
    const unsafe_optional_ptr<Node> GetChild(ART &art, const uint8_t byte) const;
    const unsafe_optional_ptr<Node> GetNextChild(ART &art, uint8_t &byte) const;

    //! 类型查询
    inline NType GetType() const {
        return NType(GetMetadata() & ~AND_GATE);
    }

    //! 门控状态
    inline GateStatus GetGateStatus() const {
        return (GetMetadata() & AND_GATE) == 0 ?
               GateStatus::GATE_NOT_SET : GateStatus::GATE_SET;
    }
    inline void SetGateStatus(const GateStatus status);

    //! 行ID访问（用于 LEAF_INLINED）
    inline row_t GetRowId() const {
        return UnsafeNumericCast<row_t>(Get() & AND_ROW_ID);
    }
    inline void SetRowId(const row_t row_id);
};
```

### 3.2.2 节点类型判断

```cpp
//! 判断是否为内部节点（Node4/16/48/256）
bool Node::IsNode() const;

//! 判断是否为叶子节点类型（Node7Leaf/Node15Leaf/Node256Leaf）
bool Node::IsLeafNode() const;

//! 判断是否为任意叶子类型
bool Node::IsAnyLeaf() const;
```

### 3.2.3 NodeHandle 句柄类

`NodeHandle` 提供对节点的安全访问，自动处理缓冲区的加载和脏标记：

```cpp
template <class T>
class NodeHandle {
public:
    NodeHandle(ART &art, const Node node)
        : handle(Node::GetAllocator(art, node.GetType()).GetHandle(node)),
          n(handle.GetRef<T>()) {
        handle.MarkModified();  // 标记为已修改
    }

    T &Get() { return n; }

private:
    SegmentHandle handle;
    T &n;
};
```

---

## 3.3 内部节点实现

### 3.3.1 BaseNode 模板类

`Node4` 和 `Node16` 共享相同的结构，通过 `BaseNode` 模板实现：

```cpp
// src/include/duckdb/execution/index/art/base_node.hpp
template <uint8_t CAPACITY, NType TYPE>
class BaseNode {
private:
    uint8_t count;           // 当前子节点数量
    uint8_t key[CAPACITY];   // 键字节数组（有序）
    Node children[CAPACITY]; // 子节点指针数组

public:
    //! 创建新节点
    static NodeHandle<BaseNode> New(ART &art, Node &node) {
        node = Node::GetAllocator(art, TYPE).New();
        node.SetMetadata(static_cast<uint8_t>(TYPE));

        NodeHandle<BaseNode> handle(art, node);
        auto &n = handle.Get();
        n.count = 0;
        for (uint8_t i = 0; i < CAPACITY; i++) {
            n.key[i] = 0;
            n.children[i].Clear();
        }
        return handle;
    }

    //! 查找子节点（线性扫描）
    static unsafe_optional_ptr<Node> GetChild(BaseNode &n, const uint8_t byte,
                                               const bool unsafe = false) {
        for (uint8_t i = 0; i < n.count; i++) {
            if (n.key[i] == byte) {
                return &n.children[i];
            }
        }
        return nullptr;
    }

    //! 查找下一个子节点（大于等于指定字节）
    static unsafe_optional_ptr<Node> GetNextChild(BaseNode &n, uint8_t &byte) {
        for (uint8_t i = 0; i < n.count; i++) {
            if (n.key[i] >= byte) {
                byte = n.key[i];
                return &n.children[i];
            }
        }
        return nullptr;
    }
};
```

### 3.3.2 Node4 实现

`Node4` 是最小的内部节点，存储 1-4 个子节点：

```cpp
class Node4 : public BaseNode<4, NType::NODE_4> {
public:
    static constexpr uint8_t CAPACITY = 4;

    //! 插入子节点
    static void InsertChild(ART &art, Node &node, const uint8_t byte, const Node child);

    //! 删除子节点
    static void DeleteChild(ART &art, Node &node, Node &prefix,
                            const uint8_t byte, const GateStatus status);

private:
    //! 从 Node16 收缩
    static void ShrinkNode16(ART &art, Node &node4, Node &node16);
};
```

**内存布局**：
```
Node4 (约 40 字节):
┌─────────┬────────────────────────┬─────────────────────────────────┐
│ count   │ key[4]                 │ children[4]                     │
│ (1B)    │ (4B)                   │ (4 × 8B = 32B)                  │
└─────────┴────────────────────────┴─────────────────────────────────┘
```

### 3.3.3 Node16 实现

`Node16` 存储 5-16 个子节点，可利用 SIMD 加速查找：

```cpp
class Node16 : public BaseNode<16, NType::NODE_16> {
public:
    static constexpr uint8_t CAPACITY = 16;

    //! 插入子节点
    static void InsertChild(ART &art, Node &node, const uint8_t byte, const Node child);

    //! 删除子节点
    static void DeleteChild(ART &art, Node &node, const uint8_t byte);

private:
    //! 从 Node4 扩展
    static void GrowNode4(ART &art, Node &node16, Node &node4);

    //! 从 Node48 收缩
    static void ShrinkNode48(ART &art, Node &node16, Node &node48);
};
```

**内存布局**：
```
Node16 (约 144 字节):
┌─────────┬────────────────────────┬─────────────────────────────────┐
│ count   │ key[16]                │ children[16]                    │
│ (1B)    │ (16B)                  │ (16 × 8B = 128B)                │
└─────────┴────────────────────────┴─────────────────────────────────┘
```

### 3.3.4 Node48 实现

`Node48` 使用间接索引策略，存储 17-48 个子节点：

```cpp
// src/include/duckdb/execution/index/art/node48.hpp
class Node48 {
public:
    static constexpr uint8_t CAPACITY = 48;
    static constexpr uint8_t EMPTY_MARKER = 48;      // 空槽标记
    static constexpr uint8_t SHRINK_THRESHOLD = 12;  // 收缩阈值

private:
    uint8_t count;                      // 当前子节点数
    uint8_t child_index[256];           // 键字节 → children 数组索引
    Node children[CAPACITY];            // 子节点数组（最多48个）

public:
    static NodeHandle<Node48> New(ART &art, Node &node) {
        node = Node::GetAllocator(art, NODE_48).New();
        node.SetMetadata(static_cast<uint8_t>(NODE_48));

        NodeHandle<Node48> handle(art, node);
        auto &n = handle.Get();
        n.count = 0;
        // 初始化所有索引为 EMPTY_MARKER
        for (uint16_t i = 0; i < 256; i++) {
            n.child_index[i] = EMPTY_MARKER;
        }
        for (uint8_t i = 0; i < CAPACITY; i++) {
            n.children[i].Clear();
        }
        return handle;
    }

    template <class NODE>
    static unsafe_optional_ptr<Node> GetChild(NODE &n, const uint8_t byte,
                                               const bool unsafe = false) {
        if (n.child_index[byte] != EMPTY_MARKER) {
            return &n.children[n.child_index[byte]];
        }
        return nullptr;
    }
};
```

**内存布局与查找过程**：
```
Node48 (~650 字节):
┌─────────┬──────────────────────────┬─────────────────────────────────┐
│ count   │ child_index[256]         │ children[48]                    │
│ (1B)    │ (256B)                   │ (48 × 8B = 384B)                │
└─────────┴──────────────────────────┴─────────────────────────────────┘

查找过程:
  键字节 0x41 ('A')
       ↓
  child_index[0x41] = 5
       ↓
  children[5] → 子节点指针
```

### 3.3.5 Node256 实现

`Node256` 使用直接索引，支持全部 256 个子节点：

```cpp
// src/include/duckdb/execution/index/art/node256.hpp
class Node256 {
public:
    static constexpr uint16_t CAPACITY = 256;
    static constexpr uint8_t SHRINK_THRESHOLD = 36;  // 收缩阈值

private:
    uint16_t count;             // 当前子节点数
    Node children[CAPACITY];    // 直接索引的子节点数组

public:
    template <class NODE>
    static unsafe_optional_ptr<Node> GetChild(NODE &n, const uint8_t byte,
                                               const bool unsafe = false) {
        if (unsafe) {
            return &n.children[byte];
        }
        if (n.children[byte].HasMetadata()) {
            return &n.children[byte];
        }
        return nullptr;
    }
};
```

**内存布局**：
```
Node256 (~2050 字节):
┌───────────┬─────────────────────────────────────────────────────────┐
│ count     │ children[256]                                           │
│ (2B)      │ (256 × 8B = 2048B)                                      │
└───────────┴─────────────────────────────────────────────────────────┘

直接索引:
  键字节 0x41 ('A') → children[0x41] → 子节点指针
```

### 3.3.6 节点扩展与收缩

节点类型根据子节点数量动态调整：

```
节点扩展 (Grow):
  Node4 (count > 4)   → Node16
  Node16 (count > 16) → Node48
  Node48 (count > 48) → Node256

节点收缩 (Shrink):
  Node256 (count ≤ 36) → Node48
  Node48 (count < 12)  → Node16
  Node16 (count ≤ 4)   → Node4

┌─────────────────────────────────────────────────────────────────┐
│                      节点类型转换图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│     Insert          Insert          Insert                       │
│   ┌────────→      ┌────────→      ┌────────→                    │
│   │    >4         │   >16         │   >48                        │
│ ┌─┴──┐         ┌──┴──┐         ┌──┴──┐         ┌───────┐        │
│ │Node4│         │Node16│        │Node48│        │Node256│        │
│ └─┬──┘         └──┬──┘         └──┬──┘         └───────┘        │
│   │    ≤4         │   <12         │   ≤36                        │
│   └←───────      └←───────      └←───────                       │
│     Delete          Delete          Delete                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3.4 前缀节点（Prefix）

### 3.4.1 前缀压缩原理

前缀压缩是 ART 的核心优化技术，将单分支路径压缩为单个节点：

```
未压缩的 Trie:              压缩后的 ART:
     [root]                    [root]
        |                         |
       'h'                   [prefix: "hel"]
        |                         |
       'e'                    ┌───┴───┐
        |                    'l'      'p'
       'l'                    |        |
      / \                  [leaf]   [leaf]
    'l'  'p'               "hello"  "help"
     |    |
    'o' [leaf]
     |  "help"
  [leaf]
  "hello"
```

### 3.4.2 Prefix 类设计

```cpp
// src/include/duckdb/execution/index/art/prefix.hpp
class Prefix {
public:
    static constexpr NType PREFIX = NType::PREFIX;
    static constexpr uint8_t ROW_ID_SIZE = sizeof(row_t);   // 8 字节
    static constexpr uint8_t ROW_ID_COUNT = ROW_ID_SIZE - 1; // 7
    static constexpr uint8_t DEPRECATED_COUNT = 15;
    static constexpr uint8_t METADATA_SIZE = sizeof(Node) + 1;

    data_ptr_t data;   // 前缀字节数据指针
    Node *ptr;         // 子节点指针
    bool in_memory;    // 是否在内存中

public:
    //! 创建新的前缀节点链
    static void New(ART &art, reference<Node> &ref, const ARTKey &key,
                    const idx_t depth, idx_t count);

    //! 连接前缀节点: parent → prev_node4 → child
    static void Concat(ART &art, Node &parent, Node &node4, const Node child,
                       uint8_t byte, const GateStatus node4_status,
                       const GateStatus status);

    //! 从前缀移除 pos 个字节
    static void Reduce(ART &art, Node &node, const idx_t pos);

    //! 在 pos 位置分裂前缀
    static GateStatus Split(ART &art, reference<Node> &node,
                            Node &child, const uint8_t pos);

    //! 获取前缀长度
    static inline uint8_t Count(const ART &art) {
        return art.prefix_count;
    }

    //! 获取指定位置的字节
    static uint8_t GetByte(const ART &art, const Node &node, const uint8_t pos);
};
```

### 3.4.3 前缀节点内存布局

```
Prefix 节点内存布局:
┌─────────────────────────────────────────────────────────────────┐
│ data[0..count-1]   │ data[count]  │ child (Node)               │
│ 前缀字节            │ 字节计数      │ 子节点指针                  │
│ (N 字节)            │ (1 字节)      │ (8 字节)                   │
└─────────────────────────────────────────────────────────────────┘

示例: 前缀 "hel" (count = 3)
┌─────┬─────┬─────┬───────┬────────────────────────────┐
│ 'h' │ 'e' │ 'l' │   3   │ → 下一个节点               │
└─────┴─────┴─────┴───────┴────────────────────────────┘
```

### 3.4.4 前缀操作

**前缀分裂**：当插入新键与现有前缀部分匹配时：

```
插入 "help" 到包含 "hello" 的树:

原始:                          分裂后:
[prefix: "hello"]             [prefix: "hel"]
      |                             |
   [leaf]                       [Node4]
                                /     \
                              'l'     'p'
                               |       |
                         [prefix:"o"] [leaf]
                               |      "help"
                            [leaf]
                            "hello"
```

**前缀归约（Reduce）**：删除操作可能导致节点只有单个子节点，此时将子节点路径合并到前缀：

```cpp
static void Prefix::Reduce(ART &art, Node &node, const idx_t pos) {
    // 移除前 pos 个字节，保留剩余前缀
    // 如果前缀变空，则释放节点并用子节点替换
}
```

---

## 3.5 叶子节点

### 3.5.1 叶子节点类型

DuckDB 支持三种叶子节点策略：

```cpp
// src/include/duckdb/execution/index/art/leaf.hpp
class Leaf {
public:
    static constexpr NType LEAF = NType::LEAF;
    static constexpr NType INLINED = NType::LEAF_INLINED;
    static constexpr uint8_t LEAF_SIZE = 4;  // 已弃用

private:
    uint8_t count;            // 已弃用
    row_t row_ids[LEAF_SIZE]; // 已弃用
    Node ptr;                 // 已弃用

public:
    //! 将行ID内联到节点指针中
    static void New(Node &node, const row_t row_id);

    //! 合并两个内联叶子
    static void MergeInlined(ArenaAllocator &arena, ART &art,
                             Node &left, Node &right,
                             GateStatus status, idx_t depth);

    //! 转换为嵌套叶子
    static void TransformToNested(ART &art, Node &node);
};
```

### 3.5.2 LEAF_INLINED（内联叶子）

最常见且最高效的叶子类型，将行 ID 直接编码到 `Node` 指针中：

```
64-bit Node 指针布局 (LEAF_INLINED):
┌────────────────┬─────────────────────────────────────────────────┐
│ Bits 56-63     │ Bits 0-55                                       │
│ NType (7)      │ Row ID (56 bits)                                │
│ + Gate bit     │ 可表示最多 2^56 行                               │
└────────────────┴─────────────────────────────────────────────────┘

创建内联叶子:
static void Leaf::New(Node &node, const row_t row_id) {
    node.SetMetadata(static_cast<uint8_t>(NType::LEAF_INLINED));
    node.SetRowId(row_id);
}
```

**优势**：
- 零额外内存分配
- 直接从指针提取行 ID
- 缓存友好

### 3.5.3 嵌套叶子（Nested Leaves）

当非唯一索引中相同键对应多个行 ID 时，使用嵌套 ART：

```
非唯一索引的嵌套结构:

外层 ART (按索引键):          内层 ART (按行ID):
     [root]                    ┌──────────────┐
       |                       │ GATE_SET     │
   [prefix: key]               │ 标记嵌套入口  │
       |                       └──────────────┘
   [GATE_SET] ─────────────→        |
                                 [root]
                                /   |   \
                           [row1] [row2] [row3]
                           内联叶子节点
```

嵌套叶子使用三种节点类型存储行 ID 的最后一个字节：

```cpp
// Node7Leaf, Node15Leaf, Node256Leaf
// 类似于 Node4, Node16, Node256，但专门用于嵌套叶子
enum class NType : uint8_t {
    NODE_7_LEAF = 8,     // 最多 7 个行ID
    NODE_15_LEAF = 9,    // 最多 15 个行ID
    NODE_256_LEAF = 10,  // 最多 256 个行ID
};
```

### 3.5.4 已弃用的 LEAF 类型

旧版本使用链表存储多个行 ID，现已弃用：

```cpp
// 已弃用的 Leaf 结构
class Leaf {
    uint8_t count;
    row_t row_ids[4];  // 最多4个行ID
    Node ptr;          // 链接到下一个 Leaf 节点
};
```

保留这些方法是为了读取旧格式的持久化数据：

```cpp
static void DeprecatedFree(ART &art, Node &node);
static bool DeprecatedGetRowIds(ART &art, const Node &node,
                                 set<row_t> &row_ids, const idx_t max_count);
static void DeprecatedVacuum(ART &art, Node &node);
```

---

## 3.6 IndexPointer 编码

### 3.6.1 64 位编码结构

`IndexPointer` 是 ART 内存管理的核心，用 64 位整数编码所有寻址信息：

```cpp
// src/include/duckdb/execution/index/index_pointer.hpp
class IndexPointer {
public:
    static constexpr idx_t SHIFT_OFFSET = 32;
    static constexpr idx_t SHIFT_METADATA = 56;
    static constexpr idx_t AND_OFFSET = 0x0000000000FFFFFF;
    static constexpr idx_t AND_BUFFER_ID = 0x00000000FFFFFFFF;
    static constexpr idx_t AND_METADATA = 0xFF00000000000000;

private:
    idx_t data;  // 64位编码

public:
    //! 从缓冲区ID和偏移构造
    IndexPointer(const uint32_t buffer_id, const uint32_t offset) : data(0) {
        auto shifted_offset = UnsafeNumericCast<idx_t>(offset) << SHIFT_OFFSET;
        data += shifted_offset;
        data += buffer_id;
    }

    //! 获取元数据（节点类型 + 门控位）
    inline uint8_t GetMetadata() const {
        return data >> SHIFT_METADATA;
    }

    //! 获取段内偏移
    inline idx_t GetOffset() const {
        auto offset = data >> SHIFT_OFFSET;
        return offset & AND_OFFSET;
    }

    //! 获取缓冲区ID
    inline idx_t GetBufferId() const {
        return data & AND_BUFFER_ID;
    }
};
```

### 3.6.2 位域布局

```
IndexPointer 64位布局:
┌─────────────────────────────────────────────────────────────────┐
│ Bits 56-63      │ Bits 32-55        │ Bits 0-31                 │
│ (8 bits)        │ (24 bits)         │ (32 bits)                 │
├─────────────────┼───────────────────┼───────────────────────────┤
│ Metadata        │ Offset            │ Buffer ID                 │
│ • NType (7位)   │ 段内偏移           │ 缓冲区标识                 │
│ • Gate (1位)    │ (最多16M段)        │ (最多4G缓冲区)             │
└─────────────────┴───────────────────┴───────────────────────────┘

元数据字节:
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ G │ T6│ T5│ T4│ T3│ T2│ T1│ T0│
└───┴───┴───┴───┴───┴───┴───┴───┘
  │   └───────────┬───────────┘
  │               └── NType (1-10)
  └── Gate 位 (0 或 1)
```

### 3.6.3 寻址能力

```
理论容量:
• Buffer ID: 32 bits → 4,294,967,296 缓冲区
• Offset: 24 bits → 16,777,216 段/缓冲区
• 总计: 最多 ~72 PB 索引数据（理论值）

实际限制:
• 单个缓冲区大小受 BufferManager 限制
• 段大小根据节点类型固定
```

---

## 3.7 FixedSizeAllocator 内存分配器

### 3.7.1 分配器设计

`FixedSizeAllocator` 为每种节点类型提供固定大小的内存段分配：

```cpp
// src/include/duckdb/execution/index/fixed_size_allocator.hpp
class FixedSizeAllocator {
public:
    static constexpr uint8_t VACUUM_THRESHOLD = 10;  // 10% 阈值触发回收

    BlockManager &block_manager;
    BufferManager &buffer_manager;

private:
    MemoryTag memory_tag;
    idx_t segment_size;                 // 单个段的大小
    idx_t bitmask_count;                // 位图数量
    idx_t bitmask_offset;               // 有效载荷起始偏移
    idx_t available_segments_per_buffer; // 每缓冲区可用段数
    idx_t total_segment_count;          // 总段数

    unordered_map<idx_t, unique_ptr<FixedSizeBuffer>> buffers;
    unordered_set<idx_t> buffers_with_free_space;
    optional_idx buffer_with_free_space;  // 缓存的空闲缓冲区
    unordered_set<idx_t> vacuum_buffers;  // 待回收缓冲区

public:
    //! 分配新段，返回 IndexPointer
    IndexPointer New();

    //! 释放段
    void Free(const IndexPointer ptr);

    //! 获取段句柄
    inline SegmentHandle GetHandle(const IndexPointer ptr);

    //! 获取段数据指针（已弃用）
    template <class T>
    inline unsafe_optional_ptr<T> Get(const IndexPointer ptr, const bool dirty = true);

    //! 初始化回收操作
    bool InitializeVacuum();

    //! 完成回收操作
    void FinalizeVacuum();

    //! 判断是否需要回收
    inline bool NeedsVacuum(const IndexPointer ptr) const;

    //! 执行回收
    IndexPointer VacuumPointer(const IndexPointer ptr);
};
```

### 3.7.2 缓冲区内存布局

每个 `FixedSizeBuffer` 包含位图头和数据段：

```
FixedSizeBuffer 布局:
┌─────────────────────────────────────────────────────────────────┐
│                     Bitmask Header                               │
│  ┌─────────┬─────────┬─────────┬─────────┐                      │
│  │ mask[0] │ mask[1] │ mask[2] │ ...     │  validity_t 数组     │
│  └─────────┴─────────┴─────────┴─────────┘  标记段是否已分配     │
├─────────────────────────────────────────────────────────────────┤
│                     Segments (有效载荷)                          │
│  ┌─────────┬─────────┬─────────┬─────────┬─────────┐           │
│  │ Seg[0]  │ Seg[1]  │ Seg[2]  │ Seg[3]  │ ...     │           │
│  │ Node4   │ Node4   │ (free)  │ Node4   │         │           │
│  └─────────┴─────────┴─────────┴─────────┴─────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### 3.7.3 分配流程

```cpp
IndexPointer FixedSizeAllocator::New() {
    // 1. 查找有空闲空间的缓冲区
    if (buffers_with_free_space.empty()) {
        // 创建新缓冲区
        auto buffer_id = GetAvailableBufferId();
        buffers[buffer_id] = make_unique<FixedSizeBuffer>(...);
        buffers_with_free_space.insert(buffer_id);
    }

    // 2. 从缓存的缓冲区分配
    auto buffer_id = *buffer_with_free_space;
    auto &buffer = buffers[buffer_id];

    // 3. 在位图中查找空闲段
    auto offset = buffer->Allocate();

    // 4. 更新统计
    total_segment_count++;

    // 5. 返回 IndexPointer
    return IndexPointer(buffer_id, offset);
}
```

### 3.7.4 ART 使用的 9 个分配器

```cpp
// ART 类中定义
static constexpr uint8_t ALLOCATOR_COUNT = 9;

// 9 种分配器对应不同节点类型
shared_ptr<array<unique_ptr<FixedSizeAllocator>, ALLOCATOR_COUNT>> allocators;

// 分配器索引映射:
// 0 → PREFIX
// 1 → LEAF (已弃用)
// 2 → NODE_4
// 3 → NODE_16
// 4 → NODE_48
// 5 → NODE_256
// 6 → NODE_7_LEAF
// 7 → NODE_15_LEAF
// 8 → NODE_256_LEAF
```

### 3.7.5 Vacuum 回收机制

当缓冲区碎片化严重时，执行 Vacuum 压缩：

```cpp
bool FixedSizeAllocator::InitializeVacuum() {
    // 计算可回收空间
    idx_t free_count = 0;
    for (auto &[id, buffer] : buffers) {
        free_count += buffer->GetFreeCount();
    }

    // 检查是否超过阈值 (10%)
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

IndexPointer FixedSizeAllocator::VacuumPointer(const IndexPointer ptr) {
    if (!NeedsVacuum(ptr)) {
        return ptr;
    }
    // 将数据复制到新位置
    auto new_ptr = New();
    auto src = Get(ptr, false);
    auto dst = Get(new_ptr, true);
    memcpy(dst, src, segment_size);
    Free(ptr);
    return new_ptr;
}
```

---

## 3.8 节点内存分配总结

### 3.8.1 各节点类型大小

```
┌─────────────────┬────────────────┬─────────────────────────────────┐
│ 节点类型         │ 近似大小        │ 主要组成                         │
├─────────────────┼────────────────┼─────────────────────────────────┤
│ PREFIX          │ 可变            │ 前缀字节 + count + child ptr    │
│ LEAF_INLINED    │ 0 (内联)        │ 无额外分配                       │
│ LEAF            │ ~44B (弃用)     │ count + 4×row_id + ptr          │
│ Node4           │ ~40B            │ count + 4×key + 4×child         │
│ Node16          │ ~144B           │ count + 16×key + 16×child       │
│ Node48          │ ~650B           │ count + 256×index + 48×child    │
│ Node256         │ ~2050B          │ count + 256×child               │
│ Node7Leaf       │ ~8B             │ 7×byte                          │
│ Node15Leaf      │ ~16B            │ 15×byte                         │
│ Node256Leaf     │ ~32B            │ 256位位图                        │
└─────────────────┴────────────────┴─────────────────────────────────┘
```

### 3.8.2 内存分配架构图

```
┌─────────────────────────────────────────────────────────────────────┐
│                        ART Index                                     │
│                          │                                           │
│                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              allocators[9] 数组                               │    │
│  ├─────────┬─────────┬─────────┬─────────┬─────────┬──────────┤    │
│  │PREFIX   │LEAF     │NODE_4   │NODE_16  │NODE_48  │NODE_256  │    │
│  │Alloc    │Alloc    │Alloc    │Alloc    │Alloc    │Alloc     │    │
│  │[0]      │[1]      │[2]      │[3]      │[4]      │[5]       │    │
│  └────┬────┴────┬────┴────┬────┴────┬────┴────┬────┴────┬─────┘    │
│       │         │         │         │         │         │           │
│       ▼         ▼         ▼         ▼         ▼         ▼           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              FixedSizeBuffer 缓冲区                          │    │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐                        │    │
│  │  │Buffer 0 │ │Buffer 1 │ │Buffer N │                        │    │
│  │  │┌──────┐ │ │┌──────┐ │ │┌──────┐ │                        │    │
│  │  ││Bitmap│ │ ││Bitmap│ │ ││Bitmap│ │                        │    │
│  │  │├──────┤ │ │├──────┤ │ │├──────┤ │                        │    │
│  │  ││Segs  │ │ ││Segs  │ │ ││Segs  │ │                        │    │
│  │  │└──────┘ │ │└──────┘ │ │└──────┘ │                        │    │
│  │  └─────────┘ └─────────┘ └─────────┘                        │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                          │                                           │
│                          ▼                                           │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                   BufferManager                              │    │
│  │                   (持久化支持)                                │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3.9 本章小结

本章详细分析了 DuckDB ART 索引的节点类型体系和内存管理机制：

1. **节点类型层次**：10 种节点类型分为内部节点（Node4/16/48/256）、前缀节点（PREFIX）和叶子节点（LEAF_INLINED 及嵌套叶子）。

2. **自适应节点策略**：根据子节点数量动态选择节点类型，平衡内存效率和访问性能。Node4/Node16 使用线性扫描，Node48 使用间接索引，Node256 使用直接索引。

3. **前缀压缩**：通过 Prefix 节点压缩单分支路径，显著减少树高度和内存占用。支持分裂、归约、连接等操作。

4. **叶子节点优化**：LEAF_INLINED 将行 ID 直接内联到指针中，零额外分配。嵌套叶子用于处理重复键场景。

5. **IndexPointer 编码**：64 位指针编码元数据（8位）、偏移（24位）和缓冲区 ID（32位），支持大规模索引。

6. **FixedSizeAllocator**：为每种节点类型提供专用分配器，使用位图管理空闲段，支持 Vacuum 回收碎片空间。

下一章将深入分析索引的核心操作——构建、插入、删除、合并和维护流程。

---

## 3.10 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/execution/index/art/node.hpp` | Node 包装类定义 |
| `src/include/duckdb/execution/index/art/base_node.hpp` | BaseNode 模板 (Node4/Node16) |
| `src/include/duckdb/execution/index/art/node48.hpp` | Node48 定义 |
| `src/include/duckdb/execution/index/art/node256.hpp` | Node256 定义 |
| `src/include/duckdb/execution/index/art/prefix.hpp` | Prefix 节点定义 |
| `src/include/duckdb/execution/index/art/leaf.hpp` | Leaf 节点定义 |
| `src/include/duckdb/execution/index/index_pointer.hpp` | IndexPointer 编码 |
| `src/include/duckdb/execution/index/fixed_size_allocator.hpp` | 内存分配器定义 |
| `src/execution/index/fixed_size_allocator.cpp` | 分配器实现 |
| `src/execution/index/art/node.cpp` | Node 实现 |
| `src/execution/index/art/prefix.cpp` | Prefix 实现 |
| `src/execution/index/art/leaf.cpp` | Leaf 实现 |
