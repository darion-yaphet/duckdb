# DuckDB 索引系统深度解析 - 第二章：ART 索引核心实现

本章深入分析 DuckDB 中 ART (Adaptive Radix Tree) 索引的核心设计与实现，包括算法原理、类设计和键编码机制。

## 2.1 Adaptive Radix Tree 原理

### 2.1.1 从 Trie 到 Radix Tree

**传统 Trie（字典树）**：
- 每个节点存储一个字符
- 每个节点最多有 256 个子节点（对于字节键）
- 空间浪费严重：稀疏节点也要分配 256 个指针槽位

**Radix Tree（压缩 Trie）**：
- 将单路径压缩为前缀
- 减少树的深度
- 但节点大小仍然固定

```
传统 Trie:                        Radix Tree (压缩后):
     [root]                            [root]
       |                                 |
      'h'                           [prefix: "hello"]
       |                                 |
      'e'                              [leaf]
       |
      'l'
       |
      'l'
       |
      'o'
       |
     [leaf]
```

### 2.1.2 ART 的自适应策略

ART (Adaptive Radix Tree) 的核心创新是**自适应节点大小**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ART 节点类型                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  子节点数量        节点类型         存储结构                                  │
│  ─────────────────────────────────────────────────────────────────          │
│                                                                              │
│    1-4           Node4          keys[4] + children[4]                       │
│                                 线性扫描                                     │
│                                                                              │
│    5-16          Node16         keys[16] + children[16]                     │
│                                 SIMD 并行比较                                │
│                                                                              │
│    17-48         Node48         child_index[256] + children[48]             │
│                                 间接索引                                     │
│                                                                              │
│    49-256        Node256        children[256]                               │
│                                 直接索引                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**自适应优势**：
- Node4/Node16：节省空间，适合稀疏分支
- Node48：平衡空间和查找速度
- Node256：O(1) 查找，适合密集分支

### 2.1.3 ART vs B+Tree 对比

| 特性 | ART | B+Tree |
|------|-----|--------|
| 查找复杂度 | O(k)，k 为键长度 | O(log n) |
| 缓存友好性 | 高（节点较小） | 中（节点较大） |
| 前缀共享 | 天然支持 | 不支持 |
| 范围查询 | 需要遍历 | 高效 |
| 更新开销 | 较低 | 中等 |
| 空间利用 | 自适应优化 | 固定结构 |

**DuckDB 选择 ART 的原因**：
1. 键长度通常较短，O(k) 接近常数
2. 自适应节点节省内存
3. 前缀压缩减少存储
4. 主要用于约束验证和点查询

## 2.2 ART 类设计

### 2.2.1 类定义

```cpp
// src/include/duckdb/execution/index/art/art.hpp:28-69
class ART : public BoundIndex {
public:
    friend class Leaf;

    //! 索引类型名
    static constexpr const char *TYPE_NAME = "ART";
    //! 固定大小分配器数量（9 种节点类型）
    static constexpr uint8_t ALLOCATOR_COUNT = 9;
    //! 已弃用版本的分配器数量
    static constexpr uint8_t DEPRECATED_ALLOCATOR_COUNT = ALLOCATOR_COUNT - 3;
    //! 最大键长度
    static constexpr idx_t MAX_KEY_LEN = 8192;

public:
    //! 树根节点
    Node tree = Node();

    //! 固定大小分配器数组
    //! 每种节点类型使用独立的分配器
    shared_ptr<array<unsafe_unique_ptr<FixedSizeAllocator>, ALLOCATOR_COUNT>> allocators;

    //! 是否拥有数据所有权
    bool owns_data;

    //! 是否需要验证键长度
    bool verify_max_key_len;

    //! 前缀节点中存储的字节数
    uint8_t prefix_count;
};
```

### 2.2.2 构造函数

```cpp
// src/execution/index/art/art.cpp:46-118
ART::ART(const string &name,
         const IndexConstraintType index_constraint_type,
         const vector<column_t> &column_ids,
         TableIOManager &table_io_manager,
         const vector<unique_ptr<Expression>> &unbound_expressions,
         AttachedDatabase &db,
         const shared_ptr<array<...>> &allocators_ptr,
         const IndexStorageInfo &info)
    : BoundIndex(name, ART::TYPE_NAME, index_constraint_type, ...),
      allocators(allocators_ptr),
      owns_data(false),
      verify_max_key_len(false) {

    // 1. 验证支持的类型
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

    // 2. 确定是否需要验证键长度
    if (types.size() > 1) {
        verify_max_key_len = true;  // 复合键
    } else if (types[0] == PhysicalType::VARCHAR) {
        verify_max_key_len = true;  // 变长字符串
    }

    // 3. 初始化分配器
    SetPrefixCount(info);
    if (!allocators) {
        owns_data = true;
        auto prefix_size = NumericCast<idx_t>(prefix_count) + Prefix::METADATA_SIZE;
        auto &block_manager = table_io_manager.GetIndexBlockManager();

        // 创建 9 个固定大小分配器
        array<unsafe_unique_ptr<FixedSizeAllocator>, ALLOCATOR_COUNT> allocator_array = {
            make_unsafe_uniq<FixedSizeAllocator>(prefix_size, block_manager),      // PREFIX
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Leaf), block_manager),     // LEAF
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node4), block_manager),    // NODE_4
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node16), block_manager),   // NODE_16
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node48), block_manager),   // NODE_48
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node256), block_manager),  // NODE_256
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node7Leaf), block_manager),   // NODE_7_LEAF
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node15Leaf), block_manager),  // NODE_15_LEAF
            make_unsafe_uniq<FixedSizeAllocator>(sizeof(Node256Leaf), block_manager), // NODE_256_LEAF
        };
        allocators = make_shared_ptr<...>(std::move(allocator_array));
    }

    // 4. 从存储信息恢复
    if (info.IsValid()) {
        if (info.root_block_ptr.IsValid()) {
            Deserialize(info.root_block_ptr);  // 旧格式
        } else {
            tree.Set(info.root);
            InitAllocators(info);  // 新格式
        }
    }
}
```

### 2.2.3 支持的数据类型

```cpp
// 支持的物理类型
PhysicalType::BOOL      // 1 字节
PhysicalType::INT8      // 1 字节
PhysicalType::INT16     // 2 字节
PhysicalType::INT32     // 4 字节
PhysicalType::INT64     // 8 字节
PhysicalType::INT128    // 16 字节
PhysicalType::UINT8     // 1 字节
PhysicalType::UINT16    // 2 字节
PhysicalType::UINT32    // 4 字节
PhysicalType::UINT64    // 8 字节
PhysicalType::UINT128   // 16 字节
PhysicalType::FLOAT     // 4 字节
PhysicalType::DOUBLE    // 8 字节
PhysicalType::VARCHAR   // 变长
```

### 2.2.4 工厂方法

```cpp
// 创建索引实例
static unique_ptr<BoundIndex> Create(CreateIndexInput &input) {
    auto art = make_uniq<ART>(
        input.name,
        input.constraint_type,
        input.column_ids,
        input.table_io_manager,
        input.unbound_expressions,
        input.db,
        nullptr,
        input.storage_info);
    return std::move(art);
}

// 创建物理计划
static PhysicalOperator &CreatePlan(PlanIndexInput &input);
```

## 2.3 ARTKey 键编码

### 2.3.1 ARTKey 类

```cpp
// src/include/duckdb/execution/index/art/art_key.hpp:20-85
class ARTKey {
public:
    ARTKey();
    ARTKey(data_ptr_t data, idx_t len);
    ARTKey(ArenaAllocator &allocator, idx_t len);

    idx_t len;        // 键长度
    data_ptr_t data;  // 键数据

public:
    // 创建键
    template <class T>
    static inline ARTKey CreateARTKey(ArenaAllocator &allocator, T value);

    static ARTKey CreateKey(ArenaAllocator &allocator, PhysicalType type, Value &value);

    // 操作符
    bool operator>(const ARTKey &key) const;
    bool operator>=(const ARTKey &key) const;
    bool operator==(const ARTKey &key) const;

    // 辅助方法
    inline bool ByteMatches(const ARTKey &other, idx_t depth) const;
    inline bool Empty() const { return len == 0; }

    void Concat(ArenaAllocator &allocator, const ARTKey &other);
    row_t GetRowId() const;
    idx_t GetMismatchPos(const ARTKey &other, const idx_t start) const;
    void VerifyKeyLength(const idx_t max_len) const;

private:
    template <class T>
    static inline data_ptr_t CreateData(ArenaAllocator &allocator, T value) {
        auto data = allocator.Allocate(sizeof(value));
        Radix::EncodeData<T>(data, value);  // 使用 Radix 编码
        return data;
    }
};
```

### 2.3.2 Radix 编码

ART 使用特殊的键编码确保字节序比较等同于值比较：

```cpp
// src/include/duckdb/common/radix.hpp

struct Radix {
    // 编码有符号整数
    template <class T>
    static void EncodeSigned(data_ptr_t dataptr, T value) {
        using UNSIGNED = typename MakeUnsigned<T>::type;
        UNSIGNED bytes;
        Store<T>(value, data_ptr_cast(&bytes));
        Store<UNSIGNED>(BSwapIfLE(bytes), dataptr);  // 大端序
        dataptr[0] = FlipSign(dataptr[0]);  // 翻转符号位
    }

    // 翻转符号位
    static inline uint8_t FlipSign(uint8_t key_byte) {
        return key_byte ^ 128;
    }

    // 编码无符号整数
    template <>
    inline void EncodeData(data_ptr_t dataptr, uint64_t value) {
        Store<uint64_t>(BSwapIfLE(value), dataptr);  // 仅大端序
    }

    // 编码浮点数
    static inline uint32_t EncodeFloat(float x) {
        if (x == 0) {
            return 1u << 31;  // 零
        }
        if (Value::IsNan(x)) {
            return UINT_MAX;  // NaN
        }
        if (x > FLT_MAX) {
            return UINT_MAX - 1;  // +∞
        }
        if (x < -FLT_MAX) {
            return 0;  // -∞
        }

        uint32_t buff = Load<uint32_t>(const_data_ptr_cast(&x));
        if ((buff & (1U << 31)) == 0) {
            buff |= (1U << 31);  // 正数：设置符号位
        } else {
            buff = ~buff;  // 负数：取反
        }
        return buff;
    }
};
```

### 2.3.3 各类型编码规则

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        类型编码规则                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  类型           编码方式                    示例                             │
│  ───────────────────────────────────────────────────────────────────        │
│                                                                              │
│  BOOL           0 或 1                     false → 0x00, true → 0x01        │
│                                                                              │
│  INT8           符号位翻转                  -128 → 0x00, 0 → 0x80, 127 → 0xFF │
│                                                                              │
│  INT16/32/64    符号位翻转 + 大端序         -1 → 0x7FFFFFFF...               │
│                                                                              │
│  UINT8/16/32/64 仅大端序                   256 → 0x0100                      │
│                                                                              │
│  FLOAT/DOUBLE   IEEE 754 变换              见下文                            │
│                                                                              │
│  VARCHAR        转义 + 终止符              见下文                            │
│                                                                              │
│  HUGEINT        高位有符号 + 低位无符号     分两部分编码                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**浮点数编码详解**：

```
IEEE 754 浮点数存储：
  符号位 | 指数 | 尾数

问题：直接字节比较不能得到正确的数值顺序
  例如：-1.0 的字节表示 > 1.0 的字节表示（因为符号位是 1）

解决方案：
  1. 正数：设置最高位（符号位变为 1）
  2. 负数：全部取反

编码后顺序：
  -∞ → 0x00000000
  -1.0 → 某个值
  0 → 0x80000000
  1.0 → 某个值
  +∞ → 0xFFFFFFFE
  NaN → 0xFFFFFFFF
```

### 2.3.4 字符串编码

```cpp
// src/execution/index/art/art_key.cpp:26-55
template <>
ARTKey ARTKey::CreateARTKey(ArenaAllocator &allocator, string_t value) {
    auto string_data = const_data_ptr_cast(value.GetData());
    auto string_len = value.GetSize();

    // 计算转义字符数量
    // \00 和 \01 需要转义
    idx_t escape_count = 0;
    for (idx_t i = 0; i < string_len; i++) {
        if (string_data[i] <= 1) {
            escape_count++;
        }
    }

    // 分配空间：原长度 + 转义字符 + 终止符
    idx_t key_len = string_len + escape_count + 1;
    auto key_data = allocator.Allocate(key_len);

    // 复制数据并添加转义
    idx_t pos = 0;
    for (idx_t i = 0; i < string_len; i++) {
        if (string_data[i] <= 1) {
            key_data[pos++] = '\01';  // 转义前缀
        }
        key_data[pos++] = string_data[i];
    }

    // 添加终止符
    key_data[pos] = '\0';
    return ARTKey(key_data, key_len);
}
```

**字符串转义规则**：

```
原始字符串        转义后
─────────────────────────
"hello"         "hello\0"
"ab\0cd"        "ab\01\0cd\0"
"x\01y"         "x\01\01y\0"

转义原因：
  \00 是终止符，需要区分字符串中的 \00
  \01 是转义前缀，需要自转义
```

### 2.3.5 复合键

```cpp
// src/execution/index/art/art.cpp:259-288
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
            keys[i].Concat(allocator, other_key);  // 连接键
            continue;
        }

        // 前一列为 NULL
        if (keys[i].Empty()) {
            continue;
        }

        // 当前列为 NULL，整个键设为 NULL
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

**复合键示例**：

```
CREATE INDEX idx ON t(a INT, b VARCHAR);

键值 (42, "hello") 的编码：
  ┌──────────────────────────────────────────────┐
  │  INT32 编码  │     VARCHAR 编码               │
  │  4 字节      │     6 字节                     │
  ├──────────────┼────────────────────────────────┤
  │  0x8000002A  │  'h' 'e' 'l' 'l' 'o' '\0'      │
  └──────────────┴────────────────────────────────┘
```

## 2.4 键生成

### 2.4.1 GenerateKeys 模板

```cpp
// src/execution/index/art/art.cpp:290-412
template <bool IS_NOT_NULL>
void GenerateKeysInternal(ArenaAllocator &allocator, DataChunk &input,
                          unsafe_vector<ARTKey> &keys) {
    // 处理第一列
    switch (input.data[0].GetType().InternalType()) {
    case PhysicalType::BOOL:
        TemplatedGenerateKeys<bool, IS_NOT_NULL>(allocator, input.data[0], input.size(), keys);
        break;
    case PhysicalType::INT32:
        TemplatedGenerateKeys<int32_t, IS_NOT_NULL>(allocator, input.data[0], input.size(), keys);
        break;
    // ... 其他类型
    case PhysicalType::VARCHAR:
        TemplatedGenerateKeys<string_t, IS_NOT_NULL>(allocator, input.data[0], input.size(), keys);
        break;
    }

    // 连接后续列（复合键）
    for (idx_t i = 1; i < input.ColumnCount(); i++) {
        switch (input.data[i].GetType().InternalType()) {
        case PhysicalType::INT32:
            ConcatenateKeys<int32_t, IS_NOT_NULL>(allocator, input.data[i], input.size(), keys);
            break;
        // ... 其他类型
        }
    }
}

template <>
void ART::GenerateKeys<>(ArenaAllocator &allocator, DataChunk &input,
                         unsafe_vector<ARTKey> &keys) {
    GenerateKeysInternal<false>(allocator, input, keys);

    // 验证键长度
    if (verify_max_key_len) {
        auto max_len = MAX_KEY_LEN * idx_t(prefix_count);
        for (idx_t i = 0; i < input.size(); i++) {
            keys[i].VerifyKeyLength(max_len);
        }
    }
}
```

### 2.4.2 键向量生成

```cpp
// src/execution/index/art/art.cpp:414-423
void ART::GenerateKeyVectors(ArenaAllocator &allocator, DataChunk &input,
                             Vector &row_ids,
                             unsafe_vector<ARTKey> &keys,
                             unsafe_vector<ARTKey> &row_id_keys) {
    // 生成索引键
    GenerateKeys<>(allocator, input, keys);

    // 生成行 ID 键
    DataChunk row_id_chunk;
    row_id_chunk.Initialize(Allocator::DefaultAllocator(),
                            vector<LogicalType> {LogicalType::ROW_TYPE},
                            input.size());
    row_id_chunk.data[0].Reference(row_ids);
    row_id_chunk.SetCardinality(input.size());
    GenerateKeys<>(allocator, row_id_chunk, row_id_keys);
}
```

## 2.5 核心操作接口

### 2.5.1 操作概览

```cpp
class ART : public BoundIndex {
public:
    // 插入操作
    ErrorData Append(IndexLock &l, DataChunk &chunk, Vector &row_ids) override;
    ErrorData Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids) override;

    // 删除操作
    void Delete(IndexLock &lock, DataChunk &entries, Vector &row_ids) override;
    void CommitDrop(IndexLock &index_lock) override;

    // 扫描操作
    unique_ptr<IndexScanState> TryInitializeScan(const Expression &expr,
                                                  const Expression &filter_expr);
    bool Scan(IndexScanState &state, idx_t max_count, set<row_t> &row_ids);

    // 构建操作
    ARTConflictType Build(unsafe_vector<ARTKey> &keys,
                          unsafe_vector<ARTKey> &row_ids,
                          const idx_t row_count);

    // 合并操作
    bool MergeIndexes(IndexLock &state, BoundIndex &other_index) override;

    // 维护操作
    void Vacuum(IndexLock &state) override;

    // 序列化
    IndexStorageInfo SerializeToDisk(QueryContext context, ...) override;
    IndexStorageInfo SerializeToWAL(...) override;
};
```

### 2.5.2 冲突类型

```cpp
// src/include/duckdb/execution/index/art/art.hpp:18
enum class ARTConflictType : uint8_t {
    NO_CONFLICT = 0,   // 无冲突
    CONSTRAINT = 1,    // 约束冲突（重复键）
    TRANSACTION = 2    // 事务冲突（写-写冲突）
};
```

### 2.5.3 处理结果

```cpp
// src/include/duckdb/execution/index/art/art.hpp:19
enum class ARTHandlingResult : uint8_t {
    CONTINUE = 0,  // 继续遍历子节点
    SKIP = 1,      // 跳过子节点
    YIELD = 2,     // 中断遍历
    NONE = 3       // 无操作
};
```

## 2.6 ARTOperator 核心操作

### 2.6.1 查找操作

```cpp
// src/include/duckdb/execution/index/art/art_operator.hpp:24-63
static unsafe_optional_ptr<const Node> Lookup(ART &art, const Node &node,
                                               const ARTKey &key, idx_t depth) {
    reference<const Node> ref(node);

    while (ref.get().HasMetadata()) {
        // 1. 到达叶子节点
        if (ref.get().IsAnyLeaf() || ref.get().GetGateStatus() == GateStatus::GATE_SET) {
            return unsafe_optional_ptr<const Node>(ref.get());
        }

        // 2. 遍历前缀
        if (ref.get().GetType() == NType::PREFIX) {
            Prefix prefix(art, ref.get());
            for (idx_t i = 0; i < prefix.data[Prefix::Count(art)]; i++) {
                if (prefix.data[i] != key[depth]) {
                    return nullptr;  // 前缀不匹配
                }
                depth++;
            }
            ref = *prefix.ptr;
            continue;
        }

        // 3. 查找子节点
        D_ASSERT(depth < key.len);
        auto child = ref.get().GetChild(art, key[depth]);

        if (!child) {
            return nullptr;  // 无匹配子节点
        }

        ref = *child;
        depth++;
    }

    return nullptr;
}
```

**查找流程图**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          Lookup 流程                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  从根节点开始                                                                │
│       │                                                                      │
│       ▼                                                                      │
│  ┌─────────────┐                                                            │
│  │ 检查节点类型 │                                                            │
│  └──────┬──────┘                                                            │
│         │                                                                    │
│    ┌────┴────┬────────────┬──────────────┐                                  │
│    ▼         ▼            ▼              ▼                                  │
│  叶子节点  前缀节点     内部节点      门节点                                   │
│    │         │            │              │                                  │
│    ▼         ▼            ▼              ▼                                  │
│  返回叶子  匹配前缀     查找子节点    进入嵌套ART                              │
│           每个字节       按key[depth]                                        │
│             │              │                                                │
│         ┌───┴───┐      ┌───┴───┐                                           │
│         ▼       ▼      ▼       ▼                                           │
│       匹配    不匹配  找到    未找到                                          │
│         │       │      │       │                                            │
│         ▼       ▼      ▼       ▼                                            │
│      继续遍历 返回null 继续遍历 返回null                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.6.2 插入操作

```cpp
// src/include/duckdb/execution/index/art/art_operator.hpp:122-216
static ARTConflictType Insert(ArenaAllocator &arena, ART &art, Node &node,
                               const ARTKey &key, idx_t depth,
                               const ARTKey &row_id, GateStatus status,
                               optional_ptr<ART> delete_art,
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
        Prefix::New(art, active_node_ref, active_key_ref.get(), depth, key.len);
        Leaf::New(active_node_ref, row_id.GetRowId());
        return ARTConflictType::NO_CONFLICT;
    }

    while (active_node_ref.get().HasMetadata()) {
        auto &active_node = active_node_ref.get();
        auto &active_key = active_key_ref.get();

        // 处理门节点
        if (status == GateStatus::GATE_NOT_SET &&
            active_node.GetGateStatus() == GateStatus::GATE_SET) {
            if (!art.IsUnique()) {
                active_key_ref = row_id;
                depth = 0;
                status = GateStatus::GATE_SET;
                continue;
            }
            return ARTConflictType::TRANSACTION;
        }

        const auto type = active_node.GetType();
        switch (type) {
        case NType::LEAF_INLINED:
            return InsertIntoInlined(arena, art, active_node, key, row_id,
                                     depth, status, delete_art, append_mode);

        case NType::LEAF:
            Leaf::TransformToNested(art, active_node);
            continue;

        case NType::NODE_7_LEAF:
        case NType::NODE_15_LEAF:
        case NType::NODE_256_LEAF:
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
                    InsertIntoPrefix(art, active_node_ref, active_key, row_id,
                                     i, depth, status);
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

### 2.6.3 删除操作

```cpp
// src/include/duckdb/execution/index/art/art_operator.hpp:220-328
static void Delete(ART &art, Node &node, const ARTKey &key, const ARTKey &row_id) {
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
        // 进入门节点
        if (status == GateStatus::GATE_NOT_SET &&
            current.get().GetGateStatus() == GateStatus::GATE_SET) {
            status = GateStatus::GATE_SET;
            current_key = row_id;
            depth = 0;
            continue;
        }

        const auto type = current.get().GetType();
        switch (type) {
        case NType::LEAF_INLINED: {
            if (current.get().GetRowId() != row_id.GetRowId()) {
                return;  // 不匹配
            }
            // 删除叶子节点
            if (!passed_node && parent.get().GetType() == NType::PREFIX) {
                Node::FreeTree(art, parent);
                return;
            }
            if (parent.get().GetType() == NType::PREFIX) {
                Node::DeleteChild(art, grandparent, greatgrandparent,
                                  current_key.get()[grandparent_depth], status, row_id);
                return;
            }
            Node::DeleteChild(art, parent, grandparent,
                              current_key.get()[parent_depth], status, row_id);
            return;
        }

        case NType::PREFIX: {
            // 更新父节点引用
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
                        return;  // 不匹配
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
            greatgrandparent = grandparent;
            grandparent = parent;
            parent = current;
            grandparent_depth = parent_depth;
            parent_depth = depth;

            auto child = current.get().GetChildMutable(art, current_key.get()[depth]);
            if (!child) {
                return;  // 无子节点
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

## 2.7 高层操作封装

### 2.7.1 Append 和 Insert

```cpp
// src/execution/index/art/art.cpp:455-529
ErrorData ART::Insert(IndexLock &l, DataChunk &chunk, Vector &row_ids,
                      IndexAppendInfo &info) {
    auto row_count = chunk.size();

    // 生成键
    ArenaAllocator arena(BufferAllocator::Get(db));
    unsafe_vector<ARTKey> keys(row_count);
    unsafe_vector<ARTKey> row_id_keys(row_count);
    GenerateKeyVectors(arena, chunk, row_ids, keys, row_id_keys);

    optional_ptr<ART> delete_art;
    if (info.delete_index) {
        delete_art = info.delete_index->Cast<ART>();
    }

    auto conflict_type = ARTConflictType::NO_CONFLICT;
    optional_idx conflict_idx;
    auto was_empty = !tree.HasMetadata();

    // 逐行插入
    for (idx_t i = 0; i < row_count; i++) {
        if (keys[i].Empty()) {
            continue;  // 跳过 NULL 键
        }

        conflict_type = ARTOperator::Insert(arena, *this, tree, keys[i], 0,
                                            row_id_keys[i], GateStatus::GATE_NOT_SET,
                                            delete_art, info.append_mode);

        if (conflict_type != ARTConflictType::NO_CONFLICT) {
            conflict_idx = i;
            break;
        }
    }

    // 回滚冲突前的插入
    if (conflict_type != ARTConflictType::NO_CONFLICT) {
        D_ASSERT(conflict_idx.IsValid());
        for (idx_t i = 0; i < conflict_idx.GetIndex(); i++) {
            if (keys[i].Empty()) {
                continue;
            }
            ARTOperator::Delete(*this, tree, keys[i], row_id_keys[i]);
        }
    }

    // 处理错误
    if (conflict_type == ARTConflictType::TRANSACTION) {
        auto msg = AppendRowError(chunk, conflict_idx.GetIndex());
        return ErrorData(TransactionException("write-write conflict on key: \"%s\"", msg));
    }

    if (conflict_type == ARTConflictType::CONSTRAINT) {
        auto msg = AppendRowError(chunk, conflict_idx.GetIndex());
        return ErrorData(ConstraintException(
            "PRIMARY KEY or UNIQUE constraint violation: duplicate key \"%s\"", msg));
    }

    return ErrorData();
}

ErrorData ART::Append(IndexLock &l, DataChunk &chunk, Vector &row_ids) {
    // 执行表达式
    DataChunk expr_chunk;
    expr_chunk.Initialize(Allocator::DefaultAllocator(), logical_types);
    ExecuteExpressions(chunk, expr_chunk);

    // 插入
    IndexAppendInfo info;
    return Insert(l, expr_chunk, row_ids, info);
}
```

### 2.7.2 Delete

```cpp
// src/execution/index/art/art.cpp:572-611
void ART::Delete(IndexLock &state, DataChunk &input, Vector &row_ids) {
    auto row_count = input.size();

    // 执行表达式
    DataChunk expr_chunk;
    expr_chunk.Initialize(Allocator::DefaultAllocator(), logical_types);
    ExecuteExpressions(input, expr_chunk);

    // 生成键
    ArenaAllocator allocator(BufferAllocator::Get(db));
    unsafe_vector<ARTKey> keys(row_count);
    unsafe_vector<ARTKey> row_id_keys(row_count);
    GenerateKeyVectors(allocator, expr_chunk, row_ids, keys, row_id_keys);

    // 逐行删除
    for (idx_t i = 0; i < row_count; i++) {
        if (keys[i].Empty()) {
            continue;
        }
        ARTOperator::Delete(*this, tree, keys[i], row_id_keys[i]);
    }

#ifdef DEBUG
    // 验证删除成功
    for (idx_t i = 0; i < row_count; i++) {
        if (keys[i].Empty()) {
            continue;
        }
        auto leaf = ARTOperator::Lookup(*this, tree, keys[i], 0);
        if (leaf) {
            auto contains_row_id = ARTOperator::LookupInLeaf(*this, *leaf, row_id_keys[i]);
            D_ASSERT(!contains_row_id);
        }
    }
#endif
}
```

### 2.7.3 CommitDrop

```cpp
// src/execution/index/art/art.cpp:565-570
void ART::CommitDrop(IndexLock &index_lock) {
    // 重置所有分配器
    for (auto &allocator : *allocators) {
        allocator->Reset();
    }
    // 清空树根
    tree.Clear();
}
```

## 2.8 小结

本章分析了 ART 索引的核心实现：

1. **ART 原理**：自适应节点大小，平衡空间和性能
2. **ART 类设计**：继承 BoundIndex，管理树根和分配器
3. **键编码**：Radix 编码确保字节序比较正确
4. **类型支持**：整数、浮点数、字符串等
5. **核心操作**：Lookup、Insert、Delete 的实现
6. **高层封装**：Append、Insert、Delete 的用户接口

下一章将深入分析 ART 的节点类型和内存管理机制。

---

## 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/execution/index/art/art.hpp` | ART 类定义 |
| `src/execution/index/art/art.cpp` | ART 类实现 |
| `src/include/duckdb/execution/index/art/art_key.hpp` | ARTKey 定义 |
| `src/execution/index/art/art_key.cpp` | ARTKey 实现 |
| `src/include/duckdb/common/radix.hpp` | Radix 编码 |
| `src/include/duckdb/execution/index/art/art_operator.hpp` | 核心操作 |
