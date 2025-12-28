# DuckDB 类型系统深度解析 - 第五章：向量化存储模型

## 引言

向量化执行是 DuckDB 高性能的核心。与传统的逐行处理（Volcano 模型）不同，DuckDB 以固定大小的向量（默认 2048 行）为单位处理数据，充分利用 CPU 缓存和 SIMD 指令。本章将深入分析 DuckDB 的向量存储模型，包括 Vector 类、向量类型、有效性掩码以及各种辅助结构。

## Vector 类概述

`Vector` 是 DuckDB 向量化执行的核心数据结构，表示一列特定类型的值：

```cpp
// vector.hpp
class Vector {
protected:
    VectorType vector_type;           // 向量类型（FLAT、CONSTANT 等）
    LogicalType type;                 // 逻辑数据类型
    data_ptr_t data;                  // 指向实际数据的指针
    ValidityMask validity;            // 有效性掩码（NULL 标记）
    buffer_ptr<VectorBuffer> buffer;  // 主数据缓冲区
    buffer_ptr<VectorBuffer> auxiliary; // 辅助缓冲区（如字符串堆）

public:
    // 构造函数
    explicit Vector(LogicalType type, idx_t capacity = STANDARD_VECTOR_SIZE);
    explicit Vector(const Value &value);  // 从单个值创建常量向量
    Vector(LogicalType type, data_ptr_t dataptr);  // 引用外部数据
    Vector(LogicalType type, bool create_data, bool initialize_to_zero,
           idx_t capacity = STANDARD_VECTOR_SIZE);

    // 访问器
    VectorType GetVectorType() const { return vector_type; }
    const LogicalType &GetType() const { return type; }
    data_ptr_t GetData() const { return data; }

    // 操作方法
    void Flatten(idx_t count);                    // 展平向量
    void ToUnifiedFormat(idx_t count, UnifiedVectorFormat &data);  // 统一格式
    Value GetValue(idx_t index) const;            // 获取单个值
    void SetValue(idx_t index, const Value &val); // 设置单个值
};
```

## 向量类型（VectorType）

DuckDB 支持多种向量类型，每种类型针对特定场景优化：

```cpp
enum class VectorType : uint8_t {
    FLAT_VECTOR,       // 标准未压缩向量
    FSST_VECTOR,       // FSST 压缩的字符串向量
    CONSTANT_VECTOR,   // 常量向量（所有行相同值）
    DICTIONARY_VECTOR, // 字典向量（选择向量 + 基础向量）
    SEQUENCE_VECTOR    // 序列向量（等差数列）
};
```

### FLAT_VECTOR

最基本的向量类型，数据连续存储在内存中：

```cpp
struct FlatVector {
    static inline data_ptr_t GetData(Vector &vector) {
        return ConstantVector::GetData(vector);
    }

    template <class T>
    static inline T *GetData(Vector &vector) {
        return ConstantVector::GetData<T>(vector);
    }

    static inline ValidityMask &Validity(Vector &vector) {
        VerifyFlatVector(vector);
        return vector.validity;
    }

    static inline bool IsNull(const Vector &vector, idx_t idx) {
        D_ASSERT(vector.GetVectorType() == VectorType::FLAT_VECTOR);
        return !vector.validity.RowIsValid(idx);
    }

    static void SetNull(Vector &vector, idx_t idx, bool is_null);
};
```

内存布局：

```
FLAT_VECTOR 内存布局：
┌────────────────────────────────────────────────────────────┐
│  数据数组: [val0, val1, val2, ..., valN]                   │
│  大小: count * sizeof(T)                                   │
└────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────┐
│  有效性掩码: [bit0, bit1, bit2, ..., bitN]                 │
│  大小: ceil(count / 64) * 8 bytes                          │
└────────────────────────────────────────────────────────────┘
```

### CONSTANT_VECTOR

所有行具有相同值的向量，仅存储一个值：

```cpp
struct ConstantVector {
    static inline const_data_ptr_t GetData(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR ||
                 vector.GetVectorType() == VectorType::FLAT_VECTOR);
        return vector.data;
    }

    template <class T>
    static inline const T *GetData(const Vector &vector) {
        VerifyVectorType<T>(vector);
        return reinterpret_cast<const T *>(GetData(vector));
    }

    static inline bool IsNull(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR);
        return !vector.validity.RowIsValid(0);
    }

    static void SetNull(Vector &vector, bool is_null);

    // 零选择向量，用于将常量映射到所有行
    static const sel_t ZERO_VECTOR[STANDARD_VECTOR_SIZE];

    static const SelectionVector *ZeroSelectionVector(idx_t count,
                                                       SelectionVector &owned_sel);
};
```

内存布局：

```
CONSTANT_VECTOR 内存布局：
┌─────────────────────┐
│  单个值: val        │
│  大小: sizeof(T)    │
└─────────────────────┘
┌─────────────────────┐
│  有效性: 1 bit      │
└─────────────────────┘
```

### DICTIONARY_VECTOR

通过选择向量引用另一个向量的元素：

```cpp
struct DictionaryVector {
    static inline const SelectionVector &SelVector(const Vector &vector) {
        VerifyDictionary(vector);
        return vector.buffer->Cast<DictionaryBuffer>().GetSelVector();
    }

    static inline const Vector &Child(const Vector &vector) {
        VerifyDictionary(vector);
        return vector.auxiliary->Cast<VectorChildBuffer>().data;
    }

    static inline optional_idx DictionarySize(const Vector &vector) {
        VerifyDictionary(vector);
        const auto &child_buffer = vector.auxiliary->Cast<VectorChildBuffer>();
        if (child_buffer.size.IsValid()) {
            return child_buffer.size;
        }
        return vector.buffer->Cast<DictionaryBuffer>().GetDictionarySize();
    }
};
```

内存布局：

```
DICTIONARY_VECTOR 内存布局：

主向量（选择向量）：
┌────────────────────────────────────────────┐
│  sel: [0, 1, 0, 2, 1, 0, ...]              │
│  指向子向量的索引                           │
└────────────────────────────────────────────┘
                    ↓
子向量（实际数据）：
┌────────────────────────────────────────────┐
│  data: [A, B, C]                           │
│  只存储唯一值                               │
└────────────────────────────────────────────┘

结果：[A, B, A, C, B, A, ...]
```

### SEQUENCE_VECTOR

表示等差数列，仅存储起始值和增量：

```cpp
struct SequenceVector {
    static void GetSequence(const Vector &vector, int64_t &start,
                            int64_t &increment, int64_t &sequence_count) {
        D_ASSERT(vector.GetVectorType() == VectorType::SEQUENCE_VECTOR);
        auto data = reinterpret_cast<int64_t *>(vector.buffer->GetData());
        start = data[0];
        increment = data[1];
        sequence_count = data[2];
    }
};
```

内存布局：

```
SEQUENCE_VECTOR 内存布局：
┌─────────────────────────────────────────┐
│  start = 0, increment = 1, count = 100  │
│  表示: [0, 1, 2, 3, ..., 99]            │
│  大小: 3 * sizeof(int64_t) = 24 bytes   │
└─────────────────────────────────────────┘
```

### FSST_VECTOR

使用 FSST（Fast Static Symbol Table）压缩的字符串向量：

```cpp
struct FSSTVector {
    static inline const_data_ptr_t GetCompressedData(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::FSST_VECTOR);
        return vector.data;
    }

    static inline ValidityMask &Validity(Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::FSST_VECTOR);
        return vector.validity;
    }

    // 解压缩 FSST 向量到 FLAT 向量
    static void DecompressVector(const Vector &src, Vector &dst,
                                  idx_t src_offset, idx_t dst_offset,
                                  idx_t copy_count, const SelectionVector *sel);

    static string_t AddCompressedString(Vector &vector, string_t data);
    static void RegisterDecoder(Vector &vector,
                                 buffer_ptr<void> &duckdb_fsst_decoder,
                                 const idx_t string_block_limit);
    static void *GetDecoder(const Vector &vector);
};
```

## 有效性掩码（ValidityMask）

有效性掩码用于高效地标记 NULL 值：

```cpp
class ValidityMask {
    validity_t *validity;  // 位向量，1 = 有效，0 = NULL
    idx_t count;           // 当前元素数量

public:
    // 检查某行是否有效
    inline bool RowIsValid(idx_t row_idx) const {
        if (!validity) {
            return true;  // 无掩码时全部有效
        }
        idx_t entry_idx = row_idx / 64;
        idx_t bit_idx = row_idx % 64;
        return (validity[entry_idx] & (validity_t(1) << bit_idx)) != 0;
    }

    // 设置某行的有效性
    inline void SetInvalid(idx_t row_idx) {
        EnsureWritable();
        idx_t entry_idx = row_idx / 64;
        idx_t bit_idx = row_idx % 64;
        validity[entry_idx] &= ~(validity_t(1) << bit_idx);
    }

    inline void SetValid(idx_t row_idx) {
        EnsureWritable();
        idx_t entry_idx = row_idx / 64;
        idx_t bit_idx = row_idx % 64;
        validity[entry_idx] |= (validity_t(1) << bit_idx);
    }

    // 检查是否全部有效
    bool AllValid() const { return !validity; }

    // 初始化为全部有效
    void SetAllValid(idx_t count) {
        if (validity) {
            Reset();
        }
        this->count = count;
    }

    // 初始化为全部无效
    void SetAllInvalid(idx_t count);

    // 合并两个有效性掩码
    void Combine(const ValidityMask &other, idx_t count);
};
```

内存优化：
- 当所有值都有效时，`validity` 指针为 `nullptr`，节省内存
- 每个 bit 表示一个值的有效性，64 个值仅需 8 字节

## UnifiedVectorFormat

`UnifiedVectorFormat` 提供了统一的向量访问接口，屏蔽不同向量类型的差异：

```cpp
struct UnifiedVectorFormat {
    const SelectionVector *sel;   // 选择向量
    data_ptr_t data;              // 数据指针
    ValidityMask validity;        // 有效性掩码
    SelectionVector owned_sel;    // 自有选择向量（用于常量）
    PhysicalType physical_type;   // 物理类型

    template <class T>
    inline const T *GetData() const {
        VerifyVectorType<T>();
        return reinterpret_cast<const T *>(data);
    }
};
```

使用方法：

```cpp
// 将任意向量转换为统一格式
UnifiedVectorFormat format;
vector.ToUnifiedFormat(count, format);

// 统一访问数据
auto data = format.GetData<int32_t>();
for (idx_t i = 0; i < count; i++) {
    idx_t idx = format.sel->get_index(i);
    if (format.validity.RowIsValid(idx)) {
        // 访问有效值
        int32_t value = data[idx];
    }
}
```

不同向量类型的转换：

```
FLAT_VECTOR:
  sel = IncrementalSelectionVector (0, 1, 2, ...)
  data = vector.data
  validity = vector.validity

CONSTANT_VECTOR:
  sel = ZeroSelectionVector (0, 0, 0, ...)
  data = vector.data
  validity = vector.validity

DICTIONARY_VECTOR:
  sel = DictionaryVector::SelVector(vector)
  data = Child(vector).data
  validity = Child(vector).validity
```

## 嵌套类型向量

### ListVector

LIST 向量存储 `list_entry_t` 数组，并通过子向量存储实际元素：

```cpp
struct ListVector {
    // 获取 list_entry_t 数据
    static inline const list_entry_t *GetData(const Vector &v) {
        if (v.GetVectorType() == VectorType::DICTIONARY_VECTOR) {
            auto &child = DictionaryVector::Child(v);
            return GetData(child);
        }
        return FlatVector::GetData<const list_entry_t>(v);
    }

    // 获取子向量（存储所有列表元素）
    static const Vector &GetEntry(const Vector &vector);
    static Vector &GetEntry(Vector &vector);

    // 获取子向量的总大小
    static idx_t GetListSize(const Vector &vector);

    // 设置子向量大小
    static void SetListSize(Vector &vec, idx_t size);

    // 获取子向量容量
    static idx_t GetListCapacity(const Vector &vector);

    // 扩展子向量容量
    static void Reserve(Vector &vec, idx_t required_capacity);

    // 追加元素
    static void Append(Vector &target, const Vector &source,
                       idx_t source_size, idx_t source_offset = 0);

    // 添加单个值
    static void PushBack(Vector &target, const Value &insert);
};
```

内存布局：

```
LIST<INTEGER> 向量存储：
[[1,2], [3], [4,5,6]]

主向量（list_entry_t 数组）：
┌────────────────────────────────────────────┐
│ [0]: offset=0, length=2                    │
│ [1]: offset=2, length=1                    │
│ [2]: offset=3, length=3                    │
└────────────────────────────────────────────┘
              ↓ 引用
子向量（INTEGER 数组）：
┌────────────────────────────────────────────┐
│ [0]=1, [1]=2, [2]=3, [3]=4, [4]=5, [5]=6   │
└────────────────────────────────────────────┘
```

### StructVector

STRUCT 向量没有自己的数据，由多个子向量组成：

```cpp
struct StructVector {
    // 获取所有子向量
    static const vector<unique_ptr<Vector>> &GetEntries(const Vector &vector);
    static vector<unique_ptr<Vector>> &GetEntries(Vector &vector);
};
```

内存布局：

```
STRUCT(a INTEGER, b VARCHAR) 向量存储：
[{a: 1, b: 'x'}, {a: 2, b: 'y'}, {a: 3, b: 'z'}]

主向量：无数据，仅包含子向量引用

子向量 0 (a - INTEGER)：
┌────────────────────┐
│ [0]=1, [1]=2, [2]=3│
└────────────────────┘

子向量 1 (b - VARCHAR)：
┌────────────────────┐
│ [0]='x', [1]='y', [2]='z'│
└────────────────────┘
```

### UnionVector

UNION 向量存储标签向量和多个成员向量：

```cpp
struct UnionVector {
    // 获取标签向量
    static const Vector &GetTags(const Vector &v);
    static Vector &GetTags(Vector &v);

    // 尝试获取指定索引的标签
    static bool TryGetTag(const Vector &vector, idx_t index, union_tag_t &tag);

    // 获取成员向量
    static const Vector &GetMember(const Vector &vector, idx_t member_index);
    static Vector &GetMember(Vector &vector, idx_t member_index);

    // 设置所有行为特定成员
    static void SetToMember(Vector &vector, union_tag_t tag,
                             Vector &member_vector, idx_t count,
                             bool keep_tags_for_null);

    // 验证 UNION 有效性
    static UnionInvalidReason CheckUnionValidity(Vector &vector, idx_t count,
                                                  const SelectionVector &sel);
};

enum class UnionInvalidReason : uint8_t {
    VALID,
    TAG_OUT_OF_RANGE,    // 标签超出范围
    NO_MEMBERS,          // 没有成员
    VALIDITY_OVERLAP,    // 多个成员同时有效
    TAG_MISMATCH,        // 标签与有效成员不匹配
    NULL_TAG             // 标签为 NULL
};
```

UNION 的不变式：
1. 每行只有一个成员向量可以非 NULL
2. 标签向量的有效性与 UNION 向量一致
3. 有效的 UNION 不能有 NULL 标签
4. 对于所有标签，0 <= tag < 成员数量

### ArrayVector

ARRAY 向量类似 LIST，但大小固定：

```cpp
struct ArrayVector {
    // 获取子向量
    static const Vector &GetEntry(const Vector &vector);
    static Vector &GetEntry(Vector &vector);

    // 获取子向量总大小
    static idx_t GetTotalSize(const Vector &vector);
};
```

内存布局：

```
ARRAY(INTEGER, 3) 向量存储：
[[1,2,3], [4,5,6]]

主向量：无数据

子向量（INTEGER 数组）：
┌────────────────────────────────────────────┐
│ [0]=1, [1]=2, [2]=3, [3]=4, [4]=5, [5]=6   │
└────────────────────────────────────────────┘

Row 0 = 子向量[0:3]
Row 1 = 子向量[3:6]
（通过 row_index * array_size 计算偏移）
```

### MapVector

MAP 向量在物理上是 LIST<STRUCT<key, value>>：

```cpp
struct MapVector {
    // 获取键向量
    static const Vector &GetKeys(const Vector &vector);
    static Vector &GetKeys(Vector &vector);

    // 获取值向量
    static const Vector &GetValues(const Vector &vector);
    static Vector &GetValues(Vector &vector);

    // 验证 MAP 有效性
    static MapInvalidReason CheckMapValidity(Vector &map, idx_t count,
                                              const SelectionVector &sel);
};

enum class MapInvalidReason : uint8_t {
    VALID,
    NULL_KEY,       // 键为 NULL
    DUPLICATE_KEY,  // 重复键
    NOT_ALIGNED,    // 键值未对齐
    INVALID_PARAMS  // 参数无效
};
```

## StringVector

字符串向量需要特殊处理，因为字符串长度可变：

```cpp
struct StringVector {
    // 添加字符串到字符串堆
    static string_t AddString(Vector &vector, const char *data, idx_t len);
    static string_t AddString(Vector &vector, string_t data);
    static string_t AddString(Vector &vector, const string &data);

    // 添加二进制数据（无需 UTF8 验证）
    static string_t AddStringOrBlob(Vector &vector, const char *data, idx_t len);

    // 分配空字符串并返回可写指针
    static string_t EmptyString(Vector &vector, idx_t len);

    // 获取字符串缓冲区
    static VectorStringBuffer &GetStringBuffer(Vector &vector);

    // 添加缓冲区引用
    static void AddBuffer(Vector &vector, buffer_ptr<VectorBuffer> buffer);

    // 添加堆引用（共享字符串内存）
    static void AddHeapReference(Vector &vector, Vector &other);
};
```

字符串存储：
- 短字符串（≤12 字节）内联在 `string_t` 中
- 长字符串存储在辅助缓冲区（字符串堆）中

## 向量缓冲区（VectorBuffer）

`VectorBuffer` 是向量数据的基础存储单元：

```cpp
enum class VectorBufferType : uint8_t {
    STANDARD_BUFFER,       // 标准数据缓冲区
    DICTIONARY_BUFFER,     // 字典缓冲区
    VECTOR_CHILD_BUFFER,   // 子向量缓冲区
    STRING_BUFFER,         // 字符串堆
    FSST_BUFFER,           // FSST 压缩缓冲区
    STRUCT_BUFFER,         // 结构体缓冲区
    LIST_BUFFER,           // 列表缓冲区
    MANAGED_BUFFER,        // 托管缓冲区
    OPAQUE_BUFFER,         // 不透明缓冲区
    ARRAY_BUFFER           // 数组缓冲区
};

class VectorBuffer {
    VectorBufferType buffer_type;
    // ...
};
```

### VectorChildBuffer

存储子向量的缓冲区：

```cpp
class VectorChildBuffer : public VectorBuffer {
public:
    explicit VectorChildBuffer(Vector vector)
        : VectorBuffer(VectorBufferType::VECTOR_CHILD_BUFFER),
          data(std::move(vector)),
          cached_hashes(LogicalType::HASH, nullptr) {}

    Vector data;                 // 子向量
    optional_idx size;           // 字典大小
    string id;                   // 字典 ID
    mutex cached_hashes_lock;    // 哈希缓存锁
    Vector cached_hashes;        // 缓存的哈希值
};
```

## 向量大小常量

```cpp
// vector_size.hpp
constexpr idx_t STANDARD_VECTOR_SIZE = 2048;
```

所有向量操作都基于这个标准大小进行优化。

## 向量操作

### 展平（Flatten）

将任意向量类型转换为 FLAT_VECTOR：

```cpp
void Vector::Flatten(idx_t count) {
    switch (GetVectorType()) {
    case VectorType::FLAT_VECTOR:
        // 已经是 FLAT，无需操作
        break;
    case VectorType::CONSTANT_VECTOR:
        // 复制常量值到所有位置
        // ...
        break;
    case VectorType::DICTIONARY_VECTOR:
        // 根据选择向量重新排列数据
        // ...
        break;
    case VectorType::SEQUENCE_VECTOR:
        // 展开序列到数组
        // ...
        break;
    }
    vector_type = VectorType::FLAT_VECTOR;
}
```

### 切片（Slice）

创建向量的子视图：

```cpp
// 范围切片
void Vector::Slice(const Vector &other, idx_t offset, idx_t end);

// 选择向量切片
void Vector::Slice(const Vector &other, const SelectionVector &sel, idx_t count);

// 就地切片（转为字典向量）
void Vector::Slice(const SelectionVector &sel, idx_t count);
```

### 引用（Reference）

创建对另一个向量的引用：

```cpp
void Vector::Reference(const Vector &other);
void Vector::Reference(const Value &value);  // 引用常量值
```

## 小结

本章详细介绍了 DuckDB 的向量化存储模型：

1. **Vector 类**：向量化执行的核心，包含类型、数据指针、有效性掩码和缓冲区
2. **向量类型**：
   - FLAT_VECTOR：连续存储的标准向量
   - CONSTANT_VECTOR：单值常量向量
   - DICTIONARY_VECTOR：通过选择向量引用的压缩向量
   - SEQUENCE_VECTOR：等差数列表示
   - FSST_VECTOR：压缩字符串向量
3. **有效性掩码**：高效的 NULL 值标记，每 64 个值仅需 8 字节
4. **UnifiedVectorFormat**：统一的向量访问接口
5. **嵌套类型向量**：ListVector、StructVector、UnionVector、ArrayVector、MapVector
6. **字符串向量**：短字符串内联，长字符串存储在堆中

向量化存储的关键优势：
- 批量处理减少函数调用开销
- 连续内存布局利用 CPU 缓存
- 统一格式简化操作符实现
- 延迟物化减少内存拷贝

下一章将探讨 DataChunk 和批量处理机制。
