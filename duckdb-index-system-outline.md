# DuckDB 索引系统深度解析 - 系列大纲

本系列深入分析 DuckDB 的索引系统实现，重点剖析 ART (Adaptive Radix Tree) 索引的设计与实现。

## 系列总览

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        DuckDB 索引系统深度解析                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第一章：索引架构概述                                                │     │
│  │  • 索引抽象层次（Index → BoundIndex/UnboundIndex）                  │     │
│  │  • 索引类型系统与注册机制                                           │     │
│  │  • 索引与存储引擎的集成                                             │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第二章：ART 索引核心实现                                            │     │
│  │  • Adaptive Radix Tree 原理                                         │     │
│  │  • ART 类设计与核心接口                                              │     │
│  │  • 键编码与比较机制（ARTKey）                                        │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第三章：ART 节点类型与内存管理                                       │     │
│  │  • 节点类型层次（Node4/16/48/256）                                   │     │
│  │  • 前缀压缩与叶子节点                                                │     │
│  │  • FixedSizeAllocator 内存分配                                       │     │
│  │  • IndexPointer 编码机制                                             │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第四章：索引操作与维护                                               │     │
│  │  • 索引构建（ARTBuilder）                                            │     │
│  │  • 插入、删除、查找操作                                              │     │
│  │  • 索引合并（ARTMerger）                                             │     │
│  │  • 垃圾回收与真空操作                                                │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第五章：索引约束与完整性                                             │     │
│  │  • UNIQUE/PRIMARY KEY 约束实现                                       │     │
│  │  • FOREIGN KEY 约束验证                                              │     │
│  │  • 冲突检测与处理                                                    │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │  第六章：索引在查询中的应用                                           │     │
│  │  • 索引扫描状态机（IndexScanState）                                  │     │
│  │  • 物理计划生成（PhysicalCreateARTIndex）                            │     │
│  │  • 索引持久化与恢复                                                  │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 第一章：索引架构概述

### 1.1 索引抽象层次

```
┌─────────────────────────────────────────────────────────────────┐
│                        Index (抽象基类)                          │
│  • column_ids: 索引列                                           │
│  • IsBound(): 是否已绑定                                        │
│  • GetIndexType(): 索引类型                                     │
│  • GetConstraintType(): 约束类型                                │
├─────────────────────────────────────────────────────────────────┤
│                              ↓                                   │
│         ┌───────────────────┴───────────────────┐               │
│         ↓                                       ↓               │
│  ┌─────────────────┐                    ┌─────────────────┐     │
│  │   BoundIndex    │                    │  UnboundIndex   │     │
│  │ (内存中可用)     │                    │ (延迟绑定)       │     │
│  │                 │                    │                 │     │
│  │ • 表达式绑定    │                    │ • create_info   │     │
│  │ • 数据操作      │                    │ • storage_info  │     │
│  │ • 扫描支持      │                    │ • WAL 重放缓冲  │     │
│  └────────┬────────┘                    └─────────────────┘     │
│           ↓                                                      │
│    ┌─────────────┐                                              │
│    │     ART     │  ← 目前唯一的具体索引实现                     │
│    └─────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
```

**核心内容**：
- `Index` 基类设计：`src/include/duckdb/storage/index.hpp`
- `BoundIndex` 实现：`src/execution/index/bound_index.cpp`
- `UnboundIndex` 实现：`src/execution/index/unbound_index.cpp`
- 绑定状态机：UNBOUND → BINDING → BOUND

### 1.2 索引类型注册机制

```cpp
// 索引创建输入
struct CreateIndexInput {
    TableIOManager &table_io_manager;
    AttachedDatabase &db;
    IndexConstraintType constraint_type;
    const string &name;
    const vector<column_t> &column_ids;
    const vector<unique_ptr<Expression>> &unbound_expressions;
    const IndexStorageInfo &storage_info;
};

// 索引类型函数
typedef unique_ptr<BoundIndex> (*index_create_function_t)(CreateIndexInput &input);
typedef PhysicalOperator &(*index_plan_function_t)(PlanIndexInput &input);
```

**核心内容**：
- `IndexType` 定义：`src/include/duckdb/execution/index/index_type.hpp`
- `IndexTypeSet` 管理：`src/execution/index/index_type_set.cpp`
- 类型注册与查询流程

### 1.3 索引与存储集成

```
┌─────────────────────────────────────────────────────────────────┐
│                       DataTable                                  │
│                          ↓                                       │
│                   TableIndexList                                 │
│                          ↓                                       │
│              ┌──────────────────────┐                           │
│              │     IndexEntry       │                           │
│              │ • bind_state (原子)   │                           │
│              │ • index (智能指针)    │                           │
│              │ • deleted_rows        │  ← 事务处理               │
│              │ • added_data          │  ← 检查点处理              │
│              └──────────────────────┘                           │
└─────────────────────────────────────────────────────────────────┘
```

**核心内容**：
- `TableIndexList` 实现：`src/include/duckdb/storage/table/table_index_list.hpp`
- `IndexEntry` 结构与生命周期
- 目录集成：`DuckIndexEntry`

---

## 第二章：ART 索引核心实现

### 2.1 Adaptive Radix Tree 原理

```
传统 Trie vs Adaptive Radix Tree:

传统 Trie (固定 256 子节点):          ART (自适应节点大小):
     [root]                                [root]
    /  |  \  ... (256 slots)              /    \
   a   b   c                             a      b
   |   |   |                             |      |
  [n] [n] [n]  ← 大量空间浪费           [n4]   [n4]  ← 按需分配
                                        /|\     |
                                       ...     ...

节点类型自适应:
• Node4   → 1-4 个子节点
• Node16  → 5-16 个子节点
• Node48  → 17-48 个子节点
• Node256 → 49-256 个子节点
```

**核心内容**：
- Radix Tree 基本原理
- 自适应节点策略的优势
- 路径压缩与前缀共享
- 与 B+Tree 的对比分析

### 2.2 ART 类设计

```cpp
class ART : public BoundIndex {
public:
    // 常量定义
    static constexpr const char *TYPE_NAME = "ART";
    static constexpr uint8_t ALLOCATOR_COUNT = 9;   // 9种固定大小分配器
    static constexpr idx_t MAX_KEY_LEN = 8192;      // 最大键长度

    // 核心成员
    Node tree;                              // 树根节点
    shared_ptr<array<...>> allocators;      // 内存分配器数组
    bool owns_data;                         // 数据所有权
    uint8_t prefix_count;                   // 前缀计数

    // 静态工厂方法
    static PhysicalOperator &CreatePlan(PlanIndexInput &input);
    static unique_ptr<BoundIndex> Create(CreateIndexInput &input);

    // 核心操作
    ErrorData Append(IndexLock &l, DataChunk &chunk, Vector &row_ids);
    ErrorData Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids);
    void Delete(IndexLock &lock, DataChunk &entries, Vector &row_ids);
    bool Scan(IndexScanState &state, idx_t max_count, set<row_t> &row_ids);
};
```

**核心内容**：
- ART 类定义：`src/include/duckdb/execution/index/art/art.hpp`
- 主要接口分析
- 与 BoundIndex 的关系

### 2.3 键编码机制（ARTKey）

```cpp
class ARTKey {
    // 键数据存储
    unsafe_unique_array<data_t> data;
    idx_t len;

    // 键创建（类型特化）
    template <class T>
    static ARTKey CreateARTKey(ArenaAllocator &allocator, T value);

    // 键比较
    bool operator>(const ARTKey &other) const;
    bool ByteMatches(const ARTKey &other, idx_t depth) const;

    // 行ID提取
    row_t GetRowId() const;
};
```

**键编码规则**：
```
类型              编码方式
─────────────────────────────────────
整数              符号位翻转 + 大端序
浮点数            IEEE 754 变换
字符串            直接字节序列 + 终止符
复合键            各部分连接
```

**核心内容**：
- ARTKey 实现：`src/execution/index/art/art_key.cpp`
- 各类型的编码规则
- 键比较语义

---

## 第三章：ART 节点类型与内存管理

### 3.1 节点类型层次

```cpp
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

**节点结构示意**：
```
Node4 (最小节点):
┌────────────────────────────────┐
│ keys[4]    │ children[4]       │
│ [a,b,c,_]  │ [ptr,ptr,ptr,_]   │
└────────────────────────────────┘

Node16 (SIMD 优化):
┌────────────────────────────────┐
│ keys[16]   │ children[16]      │  ← 可用 SIMD 并行比较
└────────────────────────────────┘

Node48 (间接索引):
┌────────────────────────────────┐
│ child_index[256] │ children[48]│
│ (1字节索引)       │ (48个指针)  │
└────────────────────────────────┘

Node256 (直接索引):
┌────────────────────────────────┐
│ children[256]                  │  ← 直接按字节索引
└────────────────────────────────┘
```

**核心内容**：
- 节点类模板：`src/include/duckdb/execution/index/art/base_node.hpp`
- Node4/Node16 实现细节
- Node48/Node256 特化
- 节点扩展与收缩策略

### 3.2 前缀压缩与叶子节点

```
前缀压缩示例:

未压缩:                     压缩后:
    [root]                    [root]
      |                         |
     'h'                    [prefix: "hel"]
      |                         |
     'e'                       'l'
      |                        / \
     'l'                     'o'  'p'
     / \                      |    |
   'l'  'p'                [leaf] [leaf]
    |    |
   'o'  [leaf]
    |
  [leaf]
```

**叶子节点类型**：
```cpp
// 内联叶子（小行ID集合）
class InlinedLeaf {
    row_t row_ids[CAPACITY];  // 直接存储行ID
};

// 嵌套叶子（大行ID集合）
class NestedLeaf {
    Node child;  // 指向另一个 ART 子树
};
```

**核心内容**：
- `Prefix` 类：`src/execution/index/art/prefix.cpp`
- `Leaf` 类：`src/execution/index/art/leaf.cpp`
- 前缀合并与分裂

### 3.3 内存分配器

```cpp
class FixedSizeAllocator {
    // 分配器配置
    idx_t segment_size;           // 段大小
    vector<FixedSizeBuffer> buffers;  // 缓冲区列表

    // 分配与释放
    IndexPointer Allocate();
    void Free(IndexPointer ptr);

    // 持久化支持
    void SerializeToDisk(BlockManager &block_manager);
};
```

**IndexPointer 编码**：
```
┌─────────────────────────────────────────────────────────────┐
│ 64-bit IndexPointer                                         │
├────────────┬───────────────┬────────────────────────────────┤
│ Bits 0-7   │ Bits 8-23     │ Bits 24-63                     │
│ 元数据      │ 段内偏移       │ 缓冲区ID                        │
│ (节点类型)  │ (16 bits)     │ (40 bits)                      │
└────────────┴───────────────┴────────────────────────────────┘
```

**核心内容**：
- `FixedSizeAllocator`：`src/execution/index/fixed_size_allocator.cpp`
- `FixedSizeBuffer`：`src/execution/index/fixed_size_buffer.cpp`
- `IndexPointer`：`src/include/duckdb/execution/index/art/index_pointer.hpp`
- 与 BufferManager 的集成

---

## 第四章：索引操作与维护

### 4.1 索引构建

```
构建流程 (已排序数据):

输入: 排序的 (key, row_id) 对
      ↓
┌─────────────────────────────────────────┐
│           ARTBuilder                     │
│  • 使用栈式深度优先遍历                    │
│  • 处理公共前缀                           │
│  • 批量创建节点                           │
└─────────────────────────────────────────┘
      ↓
输出: 完整的 ART 树

冲突检测:
enum class ARTConflictType {
    NO_CONFLICT,           // 无冲突
    CONSTRAINT_CONFLICT,   // 约束冲突
    MERGE_CONFLICT         // 合并冲突
};
```

**核心内容**：
- `ARTBuilder`：`src/execution/index/art/art_builder.cpp`
- 批量构建 vs 迭代插入
- 排序数据优化

### 4.2 插入、删除、查找

```cpp
// ARTOperator 静态操作
class ARTOperator {
    // 查找
    static optional_ptr<const Node> Lookup(
        const Node node,
        const ARTKey &key,
        idx_t depth);

    // 插入
    static ARTConflictType Insert(
        ART &art,
        Node &node,
        const ARTKey &key,
        idx_t depth,
        const ARTKey &row_id);

    // 删除
    static void Delete(
        ART &art,
        Node &node,
        const ARTKey &key,
        idx_t depth,
        const ARTKey &row_id);
};
```

**插入流程**：
```
Insert(key, row_id):
  1. 从根节点开始遍历
  2. 在每个节点:
     a. 检查前缀匹配
     b. 查找对应的子节点
     c. 如不存在，创建新节点
     d. 如需要，扩展节点类型 (Node4 → Node16 → ...)
  3. 到达叶子位置，插入 row_id
  4. 检查唯一性约束
```

**核心内容**：
- `ARTOperator`：`src/include/duckdb/execution/index/art/art_operator.hpp`
- 遍历与修改逻辑
- 节点分裂与合并

### 4.3 索引合并

```
合并场景:
┌─────────────────┐     ┌─────────────────┐
│   本地 ART      │  +  │   全局 ART      │
│ (线程本地构建)  │     │ (主索引)        │
└────────┬────────┘     └────────┬────────┘
         │                       │
         └───────────┬───────────┘
                     ↓
              ┌─────────────────┐
              │   合并后 ART    │
              │ (ARTMerger)     │
              └─────────────────┘
```

**核心内容**：
- `ARTMerger`：`src/execution/index/art/art_merger.cpp`
- 并行构建与合并策略
- 冲突处理

### 4.4 垃圾回收与真空

```cpp
void ART::Vacuum(IndexLock &state) {
    // 遍历所有分配器
    for (auto &allocator : *allocators) {
        if (allocator) {
            // 压缩每个分配器
            allocator->Vacuum();
        }
    }
}
```

**核心内容**：
- Vacuum 触发条件
- 空间回收策略
- 与事务的协调

---

## 第五章：索引约束与完整性

### 5.1 约束类型

```cpp
enum class IndexConstraintType : uint8_t {
    NONE = 0,      // 普通索引（无约束）
    UNIQUE = 1,    // UNIQUE 约束
    PRIMARY = 2,   // PRIMARY KEY 约束
    FOREIGN = 3    // FOREIGN KEY 约束
};
```

### 5.2 UNIQUE/PRIMARY KEY 实现

```
唯一性检查流程:

INSERT INTO t VALUES (key, ...)
        ↓
┌───────────────────────────────────────┐
│  ART::Insert()                        │
│    ↓                                  │
│  ARTOperator::LookupInLeaf()          │
│    ↓                                  │
│  如果找到现有 row_id:                  │
│    → 检查是否同一事务                  │
│    → 返回 CONSTRAINT_CONFLICT         │
└───────────────────────────────────────┘
```

**核心内容**：
- 冲突检测机制
- 事务内的约束验证
- 错误报告

### 5.3 FOREIGN KEY 实现

```cpp
// TableIndexList 中的外键支持
optional_ptr<IndexEntry> FindForeignKeyIndex(
    const vector<PhysicalIndex> &fk_keys,
    ForeignKeyType fk_type);

void VerifyForeignKey(
    const vector<PhysicalIndex> &fk_keys,
    DataChunk &chunk,
    ConflictManager &manager);
```

**核心内容**：
- 外键索引查找
- 引用完整性验证
- 级联操作支持

---

## 第六章：索引在查询中的应用

### 6.1 索引扫描

```cpp
class IndexScanState {
    // 扫描范围
    ARTKey low_key, high_key;
    bool low_inclusive, high_inclusive;

    // 扫描状态
    ARTScanner scanner;

    // 结果收集
    set<row_t> result_row_ids;
};
```

**扫描模式**：
```
点查询:  key = value     → Lookup
范围查询: key > a AND key < b → Iterator 遍历
前缀查询: key LIKE 'abc%' → 前缀匹配遍历
```

**核心内容**：
- `IndexScanState`：`src/include/duckdb/execution/index/art/art.hpp`
- `ARTScanner`：`src/include/duckdb/execution/index/art/art_scanner.hpp`
- 迭代器实现

### 6.2 物理计划生成

```
CREATE INDEX 计划生成:

LogicalCreateIndex
       ↓
plan_art.cpp::CreatePlan()
       ↓
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  [TableScan]                                                 │
│       ↓                                                      │
│  [Projection: index_cols + ROW_TYPE]                         │
│       ↓                                                      │
│  [Filter: NOT NULL] (可选，非 ALTER 时)                       │
│       ↓                                                      │
│  [Order By: index_cols] (可选，单列非 VARCHAR)                │
│       ↓                                                      │
│  [PhysicalCreateARTIndex]                                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**核心内容**：
- `plan_art.cpp`：`src/execution/index/art/plan_art.cpp`
- `PhysicalCreateARTIndex`：物理算子实现
- 排序优化策略

### 6.3 索引持久化

```cpp
struct IndexStorageInfo {
    string name;
    idx_t root;                              // 根节点位置
    case_insensitive_map_t<Value> options;   // 索引选项
    vector<FixedSizeAllocatorInfo> allocator_infos;  // 分配器信息
    vector<vector<IndexBufferInfo>> buffers; // 缓冲区信息
    BlockPointer root_block_ptr;             // 磁盘块指针
};
```

**持久化流程**：
```
内存 ART
    ↓
SerializeToDisk()
    ↓
┌─────────────────────────────────────┐
│  1. 序列化分配器状态                 │
│  2. 写入缓冲区到块                   │
│  3. 记录块指针映射                   │
│  4. 更新目录条目                     │
└─────────────────────────────────────┘
    ↓
磁盘存储
```

**核心内容**：
- `IndexStorageInfo`：`src/include/duckdb/storage/index_storage_info.hpp`
- 检查点写入
- WAL 集成
- 恢复流程

---

## 核心源文件索引

### 基础架构

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/storage/index.hpp` | Index 抽象基类 |
| `src/include/duckdb/execution/index/bound_index.hpp` | BoundIndex 定义 |
| `src/include/duckdb/execution/index/unbound_index.hpp` | UnboundIndex 定义 |
| `src/include/duckdb/execution/index/index_type.hpp` | 索引类型系统 |
| `src/include/duckdb/storage/table/table_index_list.hpp` | 表索引列表 |

### ART 实现

| 文件 | 说明 |
|------|------|
| `src/execution/index/art/art.cpp` | ART 主类实现 |
| `src/execution/index/art/art_key.cpp` | ARTKey 键编码 |
| `src/execution/index/art/art_builder.cpp` | 索引构建器 |
| `src/execution/index/art/art_merger.cpp` | 索引合并器 |
| `src/include/duckdb/execution/index/art/art_operator.hpp` | 核心操作 |

### 节点类型

| 文件 | 说明 |
|------|------|
| `src/execution/index/art/node.cpp` | Node 包装器 |
| `src/execution/index/art/base_node.cpp` | BaseNode 模板 |
| `src/execution/index/art/node48.cpp` | Node48 实现 |
| `src/execution/index/art/node256.cpp` | Node256 实现 |
| `src/execution/index/art/prefix.cpp` | 前缀节点 |
| `src/execution/index/art/leaf.cpp` | 叶子节点 |

### 内存管理

| 文件 | 说明 |
|------|------|
| `src/execution/index/fixed_size_allocator.cpp` | 固定大小分配器 |
| `src/execution/index/fixed_size_buffer.cpp` | 缓冲区管理 |
| `src/include/duckdb/execution/index/art/index_pointer.hpp` | 指针编码 |

### 执行与目录

| 文件 | 说明 |
|------|------|
| `src/execution/index/art/plan_art.cpp` | 计划生成 |
| `src/execution/operator/schema/physical_create_art_index.cpp` | 物理算子 |
| `src/catalog/catalog_entry/duck_index_entry.cpp` | 目录条目 |

---

## 写作计划

| 章节 | 预计篇幅 | 核心难点 |
|------|----------|----------|
| 第一章：索引架构概述 | ~30KB | 抽象层次理解 |
| 第二章：ART 核心实现 | ~35KB | 算法原理 |
| 第三章：节点与内存 | ~40KB | 内存布局细节 |
| 第四章：操作与维护 | ~35KB | 并发与一致性 |
| 第五章：约束与完整性 | ~25KB | 事务集成 |
| 第六章：查询应用 | ~35KB | 执行流程 |

**总计**: 约 200KB，6 章

---

## 与其他系列的关联

```
┌─────────────────────────────────────────────────────────────────┐
│                        系列关联图                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  存储引擎篇                索引系统篇                            │
│  ┌─────────┐              ┌─────────────┐                       │
│  │RowGroup │──────────────│ TableIndex  │                       │
│  │ColumnData│             │   List      │                       │
│  └─────────┘              └─────────────┘                       │
│       ↑                          ↑                               │
│       │                          │                               │
│  ┌─────────┐              ┌─────────────┐                       │
│  │ Buffer  │──────────────│ FixedSize   │                       │
│  │ Manager │              │ Allocator   │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
│  事务系统篇                                                      │
│  ┌─────────┐              ┌─────────────┐                       │
│  │ MVCC    │──────────────│ Index       │                       │
│  │ 版本链   │              │ Constraint  │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
│  执行引擎篇                                                      │
│  ┌─────────┐              ┌─────────────┐                       │
│  │Physical │──────────────│ PhysicalART │                       │
│  │Operator │              │ CreateIndex │                       │
│  └─────────┘              └─────────────┘                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```
