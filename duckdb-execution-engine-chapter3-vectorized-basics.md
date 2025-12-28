# DuckDB 执行引擎深度解析 - 第三章：向量化执行基础

## 引言

向量化执行是 DuckDB 高性能的核心基石。与传统的逐行处理（tuple-at-a-time）模型不同，DuckDB 采用批量处理（vector-at-a-time）模型，每次处理一批数据（通常是 2048 行）。这种设计充分利用了现代 CPU 的特性，包括 SIMD 指令、缓存局部性和分支预测。

本章深入分析 DuckDB 向量化执行的基础数据结构，包括 Vector、DataChunk、SelectionVector、ValidityMask 等核心组件，以及它们如何协同工作实现高效的批量数据处理。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        向量化执行基础架构                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      DataChunk                                   │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐           │   │
│  │  │ Vector 0 │ │ Vector 1 │ │ Vector 2 │ │   ...    │           │   │
│  │  │ (col_a)  │ │ (col_b)  │ │ (col_c)  │ │          │           │   │
│  │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └──────────┘           │   │
│  │       │            │            │                               │   │
│  │       ▼            ▼            ▼                               │   │
│  │  ┌─────────────────────────────────────────────────────────┐   │   │
│  │  │                  count = 2048                            │   │   │
│  │  └─────────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         Vector                                   │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │ vector_type │  │    type     │  │         data            │  │   │
│  │  │  (FLAT/     │  │ (LogicalType│  │   (指向实际数据)         │  │   │
│  │  │ CONSTANT/   │  │   INT64/    │  │                         │  │   │
│  │  │ DICTIONARY) │  │  VARCHAR)   │  │  [v0, v1, v2, ...]      │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  │                                                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐│   │
│  │  │                     ValidityMask                             ││   │
│  │  │  [1, 1, 0, 1, 1, 1, 0, 1, ...]  (0=NULL, 1=Valid)           ││   │
│  │  └─────────────────────────────────────────────────────────────┘│   │
│  │                                                                  │   │
│  │  ┌─────────────────────────────────────────────────────────────┐│   │
│  │  │                    VectorBuffer                              ││   │
│  │  │  (管理数据的内存分配和生命周期)                               ││   │
│  │  └─────────────────────────────────────────────────────────────┘│   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    SelectionVector                               │   │
│  │  ┌───────────────────────────────────────────────────────────┐  │   │
│  │  │  [0, 2, 5, 7, 8, ...]  (选择有效行的索引)                   │  │   │
│  │  └───────────────────────────────────────────────────────────┘  │   │
│  │  用于稀疏表示，避免物化过滤后的数据                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3.1 Vector 类设计

`Vector` 是 DuckDB 向量化执行的核心数据结构，代表一列数据的一个批次。每个 Vector 包含最多 `STANDARD_VECTOR_SIZE`（2048）个元素。

### 3.1.1 Vector 核心结构

```cpp
// src/include/duckdb/common/types/vector.hpp

class Vector {
protected:
    //! 向量类型：FLAT、CONSTANT、DICTIONARY、SEQUENCE、FSST
    VectorType vector_type;

    //! 逻辑类型：INTEGER、VARCHAR、DATE 等
    LogicalType type;

    //! 指向实际数据的指针
    data_ptr_t data;

    //! NULL 值掩码
    ValidityMask validity;

    //! 数据缓冲区（管理内存生命周期）
    buffer_ptr<VectorBuffer> buffer;

    //! 辅助缓冲区（用于复杂类型如 STRING、LIST、STRUCT）
    buffer_ptr<VectorBuffer> auxiliary;

public:
    // 友元结构体，提供类型安全的访问方法
    friend struct ConstantVector;
    friend struct DictionaryVector;
    friend struct FlatVector;
    friend struct ListVector;
    friend struct StringVector;
    friend struct FSSTVector;
    // ...
};
```

### 3.1.2 Vector 构造方式

Vector 提供多种构造方式，适应不同场景：

```cpp
// 1. 基本构造：指定类型和容量
Vector::Vector(LogicalType type_p, bool create_data, bool initialize_to_zero, idx_t capacity)
    : vector_type(VectorType::FLAT_VECTOR), type(std::move(type_p)), data(nullptr),
      validity(capacity) {
    if (create_data) {
        Initialize(initialize_to_zero, capacity);
    }
}

// 2. 便捷构造：自动分配数据
Vector::Vector(LogicalType type_p, idx_t capacity)
    : Vector(std::move(type_p), true, false, capacity) {}

// 3. 外部数据构造：使用已有内存
Vector::Vector(LogicalType type_p, data_ptr_t dataptr)
    : vector_type(VectorType::FLAT_VECTOR), type(std::move(type_p)), data(dataptr) {}

// 4. 引用构造：引用另一个 Vector
Vector::Vector(Vector &other) : type(other.type) {
    Reference(other);
}

// 5. 切片构造：从另一个 Vector 切片
Vector::Vector(const Vector &other, const SelectionVector &sel, idx_t count)
    : type(other.type) {
    Slice(other, sel, count);
}

// 6. 常量构造：从单个值创建常量向量
Vector::Vector(const Value &value) : type(value.type()) {
    Reference(value);
}
```

### 3.1.3 Vector 初始化

```cpp
void Vector::Initialize(bool initialize_to_zero, idx_t capacity) {
    auxiliary.reset();
    validity.Reset();
    auto &type = GetType();
    auto internal_type = type.InternalType();

    // 处理复杂类型的辅助缓冲区
    if (internal_type == PhysicalType::STRUCT) {
        auto struct_buffer = make_uniq<VectorStructBuffer>(type, capacity);
        auxiliary = shared_ptr<VectorBuffer>(struct_buffer.release());
    } else if (internal_type == PhysicalType::LIST) {
        auto list_buffer = make_uniq<VectorListBuffer>(type, capacity);
        auxiliary = shared_ptr<VectorBuffer>(list_buffer.release());
    } else if (internal_type == PhysicalType::ARRAY) {
        auto array_buffer = make_uniq<VectorArrayBuffer>(type, capacity);
        auxiliary = shared_ptr<VectorBuffer>(array_buffer.release());
    }

    // 分配主数据缓冲区
    auto type_size = GetTypeIdSize(internal_type);
    if (type_size > 0) {
        buffer = VectorBuffer::CreateStandardVector(type, capacity);
        data = buffer->GetData();
        if (initialize_to_zero) {
            memset(data, 0, capacity * type_size);
        }
    }

    // 确保 ValidityMask 有足够容量
    if (capacity > validity.Capacity()) {
        validity.Resize(capacity);
    }
}
```

---

## 3.2 VectorType：向量类型

DuckDB 支持多种向量类型，每种类型针对特定场景优化存储和访问效率。

### 3.2.1 VectorType 枚举

```cpp
// src/include/duckdb/common/enums/vector_type.hpp

enum class VectorType : uint8_t {
    FLAT_VECTOR,       // 标准平面向量，每个元素独立存储
    FSST_VECTOR,       // FSST 压缩的字符串向量
    CONSTANT_VECTOR,   // 常量向量，所有元素相同
    DICTIONARY_VECTOR, // 字典向量，通过选择向量引用另一个向量
    SEQUENCE_VECTOR    // 序列向量，等差数列
};
```

### 3.2.2 各类型详解

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         VectorType 类型对比                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  FLAT_VECTOR（标准向量）                                                 │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  data:     [10, 20, 30, 40, 50, 60, 70, 80, ...]                  │ │
│  │  validity: [1,  1,  1,  0,  1,  1,  0,  1,  ...]                  │ │
│  │                      ↑ NULL          ↑ NULL                       │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│  • 最通用的格式，每个元素独立存储                                        │
│  • 支持任意 NULL 模式                                                   │
│                                                                         │
│  CONSTANT_VECTOR（常量向量）                                             │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  data: [42]  ← 只存储一个值                                        │ │
│  │  所有位置都返回这个值：42, 42, 42, 42, ...                          │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│  • 常用于常量表达式（如 WHERE col > 100）                                │
│  • 极大节省内存和处理时间                                                │
│                                                                         │
│  DICTIONARY_VECTOR（字典向量）                                           │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  child:  [A, B, C, D, E]  ← 原始数据                               │ │
│  │  sel:    [0, 2, 2, 4, 1]  ← 选择向量                               │ │
│  │  result: [A, C, C, E, B]  ← 逻辑视图                               │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│  • 用于 Filter 后的稀疏表示                                             │
│  • 避免数据复制，只记录索引                                              │
│                                                                         │
│  SEQUENCE_VECTOR（序列向量）                                             │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  start: 100, increment: 10                                        │ │
│  │  result: [100, 110, 120, 130, 140, ...]                           │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│  • 用于 row_number() 等窗口函数                                         │
│  • 只存储 start 和 increment                                            │
│                                                                         │
│  FSST_VECTOR（FSST 压缩向量）                                            │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │  compressed_data: [压缩后的字符串数据]                              │ │
│  │  symbol_table: [符号表]                                            │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│  • 用于字符串列的压缩存储                                                │
│  • 可以直接在压缩数据上进行某些操作                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2.3 类型转换与展平

当需要统一处理不同类型的向量时，可以通过 `Flatten()` 方法将其转换为 FLAT_VECTOR：

```cpp
// src/common/types/vector.cpp

void Vector::Flatten(idx_t count) {
    switch (GetVectorType()) {
    case VectorType::FLAT_VECTOR:
        // 已经是 FLAT，无需处理
        break;
    case VectorType::CONSTANT_VECTOR:
        // 将常量展开为 count 个相同的值
        Flatten(*ConstantVector::ZeroSelectionVector(), count);
        break;
    case VectorType::DICTIONARY_VECTOR: {
        // 根据选择向量物化数据
        auto &sel = DictionaryVector::SelVector(*this);
        Flatten(sel, count);
        break;
    }
    case VectorType::SEQUENCE_VECTOR: {
        // 生成序列数据
        int64_t start, increment;
        SequenceVector::GetSequence(*this, start, increment);
        // ... 生成 count 个元素
        break;
    }
    case VectorType::FSST_VECTOR: {
        // 解压 FSST 数据
        FSSTVector::DecompressVector(*this, ...);
        break;
    }
    }
}
```

---

## 3.3 DataChunk：数据块

`DataChunk` 是向量化执行的基本传输单元，包含多个 Vector（列），代表一个行批次的所有列数据。

### 3.3.1 DataChunk 结构

```cpp
// src/include/duckdb/common/types/data_chunk.hpp

class DataChunk {
public:
    //! 列向量数组
    vector<Vector> data;

private:
    //! 当前行数
    idx_t count;
    //! 容量上限
    idx_t capacity;

public:
    // 初始化指定类型的列
    DUCKDB_API void Initialize(Allocator &allocator, const vector<LogicalType> &types,
                               idx_t capacity = STANDARD_VECTOR_SIZE);

    // 获取/设置行数
    inline idx_t size() const { return count; }
    inline void SetCardinality(idx_t count_p);

    // 列访问
    inline idx_t ColumnCount() const { return data.size(); }
    inline Vector &operator[](idx_t idx) { return data[idx]; }
};
```

### 3.3.2 DataChunk 与 Vector 的关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DataChunk 结构                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  DataChunk { count = 2048, capacity = 2048 }                           │
│  │                                                                      │
│  ├─ data[0]: Vector(INTEGER)                                           │
│  │    ├─ vector_type: FLAT_VECTOR                                      │
│  │    ├─ data: [1, 2, 3, 4, ..., 2048]                                 │
│  │    └─ validity: [1, 1, 1, 0, ...]                                   │
│  │                                                                      │
│  ├─ data[1]: Vector(VARCHAR)                                           │
│  │    ├─ vector_type: DICTIONARY_VECTOR                                │
│  │    ├─ child → 原始字符串数据                                         │
│  │    └─ sel: [0, 1, 0, 2, ...]                                        │
│  │                                                                      │
│  ├─ data[2]: Vector(DOUBLE)                                            │
│  │    ├─ vector_type: CONSTANT_VECTOR                                  │
│  │    └─ data: [3.14]  (所有行都是这个值)                               │
│  │                                                                      │
│  └─ data[3]: Vector(DATE)                                              │
│       ├─ vector_type: FLAT_VECTOR                                      │
│       ├─ data: [19000, 19001, 19002, ...]                              │
│       └─ validity: [1, 1, 0, 1, ...]                                   │
│                                                                         │
│  注意：同一个 DataChunk 中的 Vector 可以有不同的 VectorType              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3.3 DataChunk 初始化

```cpp
// src/common/types/data_chunk.cpp

void DataChunk::Initialize(Allocator &allocator, const vector<LogicalType> &types,
                           idx_t capacity_p) {
    D_ASSERT(data.empty());
    D_ASSERT(capacity_p <= ArrayValue::MAX_ARRAY_SIZE);

    capacity = capacity_p;
    count = 0;

    // 为每种类型创建一个 Vector
    for (auto &type : types) {
        VectorCache cache(allocator, type, capacity);
        data.emplace_back(cache);
    }
}

void DataChunk::InitializeEmpty(const vector<LogicalType> &types) {
    capacity = STANDARD_VECTOR_SIZE;
    count = 0;

    // 创建空 Vector（不分配数据）
    for (auto &type : types) {
        data.emplace_back(type, nullptr);
    }
}
```

### 3.3.4 DataChunk 操作

```cpp
// 重置 DataChunk（保留内存分配）
void DataChunk::Reset() {
    count = 0;
    for (auto &vec : data) {
        vec.ResetFromCache(cache); // 重用已分配的缓冲区
    }
}

// 追加数据
void DataChunk::Append(DataChunk &other, bool resize, SelectionVector *sel,
                       idx_t sel_count) {
    idx_t new_count = count + (sel ? sel_count : other.size());

    // 必要时扩容
    if (new_count > capacity) {
        if (!resize) {
            throw InternalException("Cannot append to DataChunk");
        }
        for (auto &vec : data) {
            vec.Resize(count, new_count);
        }
        capacity = new_count;
    }

    // 复制数据
    for (idx_t i = 0; i < ColumnCount(); i++) {
        VectorOperations::Copy(other.data[i], data[i],
                               sel ? *sel : *FlatVector::IncrementalSelectionVector(),
                               sel ? sel_count : other.size(), 0, count);
    }
    count = new_count;
}

// 切片操作
void DataChunk::Slice(const SelectionVector &sel, idx_t count_p) {
    for (auto &vec : data) {
        vec.Slice(sel, count_p);
    }
    count = count_p;
}
```

---

## 3.4 SelectionVector：选择向量

`SelectionVector` 是一个索引数组，用于指定 Vector 中哪些行是有效的。它是实现延迟物化（late materialization）的关键组件。

### 3.4.1 SelectionVector 结构

```cpp
// src/include/duckdb/common/types/selection_vector.hpp

// 选择向量的元素类型（32位无符号整数）
using sel_t = uint32_t;

// 选择数据的存储
struct SelectionData {
    explicit SelectionData(idx_t count) {
        owned_data = make_unsafe_uniq_array_uninitialized<sel_t>(count);
    }
    unique_array_ptr<sel_t> owned_data;
};

// 选择向量
struct SelectionVector {
    //! 指向选择数据的指针
    sel_t *sel_vector;
    //! 拥有的选择数据（如果有）
    buffer_ptr<SelectionData> selection_data;

public:
    // 默认构造（未设置）
    SelectionVector() : sel_vector(nullptr) {}

    // 从已有指针构造
    explicit SelectionVector(sel_t *sel) : sel_vector(sel) {}

    // 分配新的选择向量
    explicit SelectionVector(idx_t count)
        : selection_data(make_buffer<SelectionData>(count)) {
        sel_vector = selection_data->owned_data.get();
    }

    // 获取索引
    inline idx_t get_index(idx_t idx) const {
        return sel_vector[idx];
    }

    // 设置索引
    inline void set_index(idx_t idx, idx_t loc) {
        sel_vector[idx] = UnsafeNumericCast<sel_t>(loc);
    }

    // 检查是否已设置
    inline bool IsSet() const {
        return sel_vector != nullptr;
    }
};
```

### 3.4.2 SelectionVector 应用场景

```
┌─────────────────────────────────────────────────────────────────────────┐
│               SelectionVector 使用场景                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景1：过滤操作 (Filter)                                                │
│  ─────────────────────────────────────────                              │
│  原始数据:   [10, 25, 5, 30, 15, 8, 22, 40]                             │
│  过滤条件:   value > 20                                                 │
│                                                                         │
│  传统方式（需要复制数据）:                                               │
│    新数据: [25, 30, 22, 40]  ← 需要复制                                 │
│                                                                         │
│  SelectionVector 方式（零复制）:                                         │
│    原始数据: [10, 25, 5, 30, 15, 8, 22, 40]  ← 保持不变                 │
│    sel:      [1, 3, 6, 7]                    ← 只记录索引               │
│    count:    4                                                          │
│                                                                         │
│  场景2：Join 操作                                                        │
│  ─────────────────                                                      │
│  左表:  [A, B, C, D, E]                                                 │
│  右表:  [X, Y, Z]                                                       │
│  匹配:  A-Y, B-X, D-Z, E-Y                                              │
│                                                                         │
│  左表 sel: [0, 1, 3, 4]  → 选择 A, B, D, E                              │
│  右表 sel: [1, 0, 2, 1]  → 选择 Y, X, Z, Y                              │
│                                                                         │
│  场景3：字典编码                                                         │
│  ─────────────────                                                      │
│  字典:  ["apple", "banana", "cherry"]                                   │
│  数据:  ["apple", "banana", "apple", "cherry", "apple"]                │
│                                                                         │
│  DICTIONARY_VECTOR:                                                     │
│    child: ["apple", "banana", "cherry"]                                │
│    sel:   [0, 1, 0, 2, 0]                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.4.3 SelectionVector 操作

```cpp
// 切片操作：组合两个选择向量
buffer_ptr<SelectionData> SelectionVector::Slice(const SelectionVector &sel,
                                                  idx_t count) const {
    auto result = make_buffer<SelectionData>(count);
    auto result_ptr = result->owned_data.get();
    // 组合索引：result[i] = this[sel[i]]
    for (idx_t i = 0; i < count; i++) {
        auto idx = sel.get_index(i);
        result_ptr[i] = this->get_index(idx);
    }
    return result;
}

// 创建增量选择向量 [0, 1, 2, 3, ...]
const SelectionVector *FlatVector::IncrementalSelectionVector() {
    static SelectionVector INCREMENTAL_SELECTION_VECTOR = []() {
        SelectionVector sel(STANDARD_VECTOR_SIZE);
        for (idx_t i = 0; i < STANDARD_VECTOR_SIZE; i++) {
            sel.set_index(i, i);
        }
        return sel;
    }();
    return &INCREMENTAL_SELECTION_VECTOR;
}
```

---

## 3.5 ValidityMask：NULL 值处理

`ValidityMask` 是一个位图，用于标记 Vector 中每个元素是否为 NULL。使用位图而非单独的标志可以极大节省内存并支持批量操作。

### 3.5.1 ValidityMask 结构

```cpp
// src/include/duckdb/common/types/validity_mask.hpp

// 每个 validity_t 存储 64 个布尔值
using validity_t = uint64_t;

template <typename V>
struct TemplatedValidityMask {
    //! 每个值中的位数
    static constexpr idx_t BITS_PER_VALUE = sizeof(V) * 8;  // 64
    //! 全1掩码（所有有效）
    static constexpr V MAX_ENTRY = V(~V(0));

    //! 指向 validity 数据的指针
    V *validity_mask;
    //! 拥有的数据缓冲区
    buffer_ptr<ValidityBuffer> validity_data;

public:
    // 默认构造：所有元素有效
    TemplatedValidityMask() : validity_mask(nullptr) {}

    // 指定容量构造
    explicit TemplatedValidityMask(idx_t count);

    // 检查单行是否有效
    inline bool RowIsValid(idx_t row_idx) const {
        if (!validity_mask) {
            return true;  // 无掩码表示全部有效
        }
        idx_t entry_idx = row_idx / BITS_PER_VALUE;
        idx_t bit_idx = row_idx % BITS_PER_VALUE;
        return (validity_mask[entry_idx] & (V(1) << bit_idx)) != 0;
    }

    // 检查所有元素是否有效
    inline bool AllValid() const {
        return !validity_mask;  // nullptr 表示全部有效
    }

    // 设置元素为无效 (NULL)
    inline void SetInvalid(idx_t row_idx) {
        EnsureWritable();
        idx_t entry_idx = row_idx / BITS_PER_VALUE;
        idx_t bit_idx = row_idx % BITS_PER_VALUE;
        validity_mask[entry_idx] &= ~(V(1) << bit_idx);
    }

    // 设置元素为有效
    inline void SetValid(idx_t row_idx) {
        // 只在已有掩码时才需要设置
        if (validity_mask) {
            idx_t entry_idx = row_idx / BITS_PER_VALUE;
            idx_t bit_idx = row_idx % BITS_PER_VALUE;
            validity_mask[entry_idx] |= (V(1) << bit_idx);
        }
    }
};

struct ValidityMask : public TemplatedValidityMask<validity_t> {
    // 继承所有方法，添加额外功能
};
```

### 3.5.2 ValidityMask 优化策略

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    ValidityMask 优化策略                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  优化1：延迟分配                                                         │
│  ────────────────                                                       │
│  validity_mask = nullptr  →  表示所有 2048 个元素都有效                  │
│                                                                         │
│  只在第一次遇到 NULL 时才分配内存：                                       │
│  void EnsureWritable() {                                                │
│      if (!validity_mask) {                                              │
│          Initialize(max_count);  // 分配并设置所有位为1                  │
│      }                                                                  │
│  }                                                                      │
│                                                                         │
│  优化2：位图压缩                                                         │
│  ────────────────                                                       │
│  2048 个元素只需要 2048 / 64 = 32 个 uint64_t = 256 字节               │
│  相比每个元素一个字节需要 2048 字节，节省 87.5%                          │
│                                                                         │
│  优化3：批量检查                                                         │
│  ────────────────                                                       │
│  // 快速检查连续 64 个元素是否全部有效                                   │
│  bool AllValid64(idx_t entry_idx) {                                     │
│      return validity_mask[entry_idx] == MAX_ENTRY;                      │
│  }                                                                      │
│                                                                         │
│  优化4：SIMD 友好                                                        │
│  ────────────────                                                       │
│  // 可以使用 SIMD 指令一次处理多个 validity 条目                         │
│  // 例如 AVX-256 可以一次处理 4 个 uint64_t = 256 个元素                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.5.3 ValidityMask 常用操作

```cpp
// 合并两个 ValidityMask（AND 操作）
void ValidityMask::Combine(const ValidityMask &other, idx_t count) {
    if (other.AllValid()) {
        return;  // other 全有效，结果不变
    }
    if (AllValid()) {
        // this 全有效，直接使用 other
        Initialize(other);
        return;
    }
    // 两者都有 NULL，需要合并
    auto entry_count = EntryCount(count);
    for (idx_t i = 0; i < entry_count; i++) {
        validity_mask[i] &= other.validity_mask[i];
    }
}

// 切片操作
void ValidityMask::Slice(const ValidityMask &other, idx_t offset, idx_t count) {
    if (other.AllValid()) {
        Reset();  // 源全有效，结果也全有效
        return;
    }

    EnsureWritable();
    // 复制并移位位图
    for (idx_t i = 0; i < count; i++) {
        Set(i, other.RowIsValid(offset + i));
    }
}

// 根据 SelectionVector 复制
void ValidityMask::CopySel(const ValidityMask &other, const SelectionVector &sel,
                           idx_t source_offset, idx_t target_offset, idx_t count) {
    if (other.AllValid()) {
        // 源全有效，设置目标范围为有效
        for (idx_t i = 0; i < count; i++) {
            SetValid(target_offset + i);
        }
        return;
    }

    // 逐个复制
    for (idx_t i = 0; i < count; i++) {
        auto source_idx = sel.get_index(source_offset + i);
        Set(target_offset + i, other.RowIsValid(source_idx));
    }
}
```

---

## 3.6 VectorBuffer：缓冲区管理

`VectorBuffer` 管理 Vector 的底层内存，不同类型的 Vector 使用不同的 Buffer 类型。

### 3.6.1 VectorBufferType 枚举

```cpp
// src/include/duckdb/common/types/vector_buffer.hpp

enum class VectorBufferType : uint8_t {
    STANDARD_BUFFER,     // 标准缓冲区，存储固定大小的数据
    DICTIONARY_BUFFER,   // 字典缓冲区，存储选择向量
    VECTOR_CHILD_BUFFER, // 向量子缓冲区，存储另一个 Vector
    STRING_BUFFER,       // 字符串缓冲区，存储字符串堆
    FSST_BUFFER,         // FSST 压缩缓冲区
    STRUCT_BUFFER,       // 结构体缓冲区，存储子向量
    LIST_BUFFER,         // 列表缓冲区
    MANAGED_BUFFER,      // 受管理缓冲区（由 BufferManager 管理）
    OPAQUE_BUFFER,       // 不透明缓冲区
    ARRAY_BUFFER         // 数组缓冲区
};
```

### 3.6.2 VectorBuffer 基类

```cpp
class VectorBuffer {
public:
    explicit VectorBuffer(VectorBufferType type) : buffer_type(type) {}

    // 分配指定大小的内存
    explicit VectorBuffer(idx_t data_size)
        : buffer_type(VectorBufferType::STANDARD_BUFFER) {
        if (data_size > 0) {
            data = Allocator::DefaultAllocator().Allocate(data_size);
        }
    }

    // 获取数据指针
    data_ptr_t GetData() { return data.get(); }

    // 创建工厂方法
    static buffer_ptr<VectorBuffer> CreateStandardVector(PhysicalType type,
                                                          idx_t capacity = STANDARD_VECTOR_SIZE);
    static buffer_ptr<VectorBuffer> CreateConstantVector(PhysicalType type);

protected:
    VectorBufferType buffer_type;
    unique_ptr<VectorAuxiliaryData> aux_data;
    AllocatedData data;
};
```

### 3.6.3 特化 Buffer 类型

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    VectorBuffer 类型层次                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  VectorBuffer (基类)                                                     │
│       │                                                                 │
│       ├── DictionaryBuffer                                              │
│       │   └── 存储 SelectionVector，用于 DICTIONARY_VECTOR              │
│       │       struct DictionaryBuffer : VectorBuffer {                  │
│       │           SelectionVector sel_vector;                           │
│       │           optional_idx dictionary_size;                         │
│       │           string dictionary_id;                                 │
│       │       };                                                        │
│       │                                                                 │
│       ├── VectorStringBuffer                                            │
│       │   └── 存储字符串堆，用于 VARCHAR 类型                            │
│       │       struct VectorStringBuffer : VectorBuffer {                │
│       │           StringHeap heap;                                      │
│       │           vector<buffer_ptr<VectorBuffer>> references;          │
│       │       };                                                        │
│       │                                                                 │
│       ├── VectorFSSTStringBuffer                                        │
│       │   └── 存储 FSST 压缩字符串                                       │
│       │       struct VectorFSSTStringBuffer : VectorStringBuffer {      │
│       │           buffer_ptr<void> duckdb_fsst_decoder;                 │
│       │           idx_t total_string_count;                             │
│       │           vector<unsigned char> decompress_buffer;              │
│       │       };                                                        │
│       │                                                                 │
│       ├── VectorStructBuffer                                            │
│       │   └── 存储结构体子向量                                           │
│       │       struct VectorStructBuffer : VectorBuffer {                │
│       │           vector<unique_ptr<Vector>> children;                  │
│       │       };                                                        │
│       │                                                                 │
│       ├── VectorListBuffer                                              │
│       │   └── 存储列表子向量                                             │
│       │       struct VectorListBuffer : VectorBuffer {                  │
│       │           unique_ptr<Vector> child;                             │
│       │           idx_t capacity, size;                                 │
│       │       };                                                        │
│       │                                                                 │
│       ├── VectorArrayBuffer                                             │
│       │   └── 存储固定大小数组                                           │
│       │       struct VectorArrayBuffer : VectorBuffer {                 │
│       │           unique_ptr<Vector> child;                             │
│       │           idx_t array_size, size;                               │
│       │       };                                                        │
│       │                                                                 │
│       └── ManagedVectorBuffer                                           │
│           └── 存储 BufferManager 管理的缓冲区句柄                        │
│               struct ManagedVectorBuffer : VectorBuffer {               │
│                   BufferHandle handle;                                  │
│               };                                                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3.7 UnifiedVectorFormat：统一向量格式

`UnifiedVectorFormat` 提供了一种统一的方式来访问任意类型的 Vector，无论其底层表示是 FLAT、CONSTANT 还是 DICTIONARY。

### 3.7.1 UnifiedVectorFormat 结构

```cpp
// src/include/duckdb/common/types/vector.hpp

struct UnifiedVectorFormat {
    //! 选择向量指针
    const SelectionVector *sel;
    //! 数据指针
    data_ptr_t data;
    //! NULL 掩码
    ValidityMask validity;
    //! 自有的选择向量（当需要构造时使用）
    SelectionVector owned_sel;
    //! 物理类型
    PhysicalType physical_type;

    // 获取类型化数据的便捷方法
    template <class T>
    static inline const T *GetData(const UnifiedVectorFormat &format) {
        return reinterpret_cast<const T *>(format.data);
    }

    template <class T>
    static inline T *GetDataNoConst(UnifiedVectorFormat &format) {
        return reinterpret_cast<T *>(format.data);
    }
};
```

### 3.7.2 ToUnifiedFormat 转换

```cpp
// src/common/types/vector.cpp

void Vector::ToUnifiedFormat(idx_t count, UnifiedVectorFormat &format) {
    format.physical_type = GetType().InternalType();

    switch (GetVectorType()) {
    case VectorType::DICTIONARY_VECTOR: {
        // 字典向量：使用其选择向量
        auto &sel = DictionaryVector::SelVector(*this);
        format.owned_sel.Initialize(sel);
        format.sel = &format.owned_sel;

        auto &child = DictionaryVector::Child(*this);
        // 递归获取子向量的统一格式
        if (child.GetVectorType() == VectorType::FLAT_VECTOR) {
            format.data = FlatVector::GetData(child);
            format.validity = FlatVector::Validity(child);
        } else {
            // 需要展平
            UnifiedVectorFormat child_format;
            child.ToUnifiedFormat(count, child_format);
            format.data = child_format.data;
            format.validity = child_format.validity;
        }
        break;
    }
    case VectorType::CONSTANT_VECTOR:
        // 常量向量：创建全零选择向量
        format.sel = ConstantVector::ZeroSelectionVector();
        format.data = ConstantVector::GetData(*this);
        format.validity.Slice(ConstantVector::Validity(*this), 0, 1);
        break;

    case VectorType::FSST_VECTOR:
        // FSST 向量：需要先解压
        Flatten(count);
        // fallthrough

    case VectorType::FLAT_VECTOR:
    default:
        // 平面向量：直接使用增量选择向量
        format.sel = FlatVector::IncrementalSelectionVector();
        format.data = FlatVector::GetData(*this);
        format.validity = FlatVector::Validity(*this);
        break;
    }
}
```

### 3.7.3 UnifiedVectorFormat 使用示例

```
┌─────────────────────────────────────────────────────────────────────────┐
│                  UnifiedVectorFormat 使用模式                            │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  问题：不同 VectorType 访问方式不同                                       │
│  ─────────────────────────────────                                      │
│                                                                         │
│  FLAT_VECTOR:                                                           │
│    auto data = FlatVector::GetData<int32_t>(vec);                       │
│    value = data[i];                                                     │
│                                                                         │
│  CONSTANT_VECTOR:                                                       │
│    auto data = ConstantVector::GetData<int32_t>(vec);                   │
│    value = data[0];  // 总是索引 0                                       │
│                                                                         │
│  DICTIONARY_VECTOR:                                                     │
│    auto &child = DictionaryVector::Child(vec);                          │
│    auto &sel = DictionaryVector::SelVector(vec);                        │
│    auto data = FlatVector::GetData<int32_t>(child);                     │
│    value = data[sel.get_index(i)];                                      │
│                                                                         │
│  解决方案：UnifiedVectorFormat                                           │
│  ─────────────────────────────                                          │
│                                                                         │
│  UnifiedVectorFormat format;                                            │
│  vec.ToUnifiedFormat(count, format);                                    │
│                                                                         │
│  // 现在可以用统一的方式访问任何类型的向量                                 │
│  auto data = UnifiedVectorFormat::GetData<int32_t>(format);             │
│  for (idx_t i = 0; i < count; i++) {                                    │
│      auto idx = format.sel->get_index(i);  // 获取实际索引               │
│      if (format.validity.RowIsValid(idx)) {                             │
│          int32_t value = data[idx];                                     │
│          // 处理有效值                                                   │
│      }                                                                  │
│  }                                                                      │
│                                                                         │
│  不同 VectorType 在 UnifiedVectorFormat 中的表现：                        │
│  ─────────────────────────────────────────────                          │
│  FLAT_VECTOR:       sel = [0,1,2,3,...], data = 原始数据                │
│  CONSTANT_VECTOR:   sel = [0,0,0,0,...], data = 单个值                  │
│  DICTIONARY_VECTOR: sel = 字典的选择向量, data = 子向量数据              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3.8 Helper 结构体

DuckDB 提供了一组 Helper 结构体，以类型安全的方式访问不同类型的 Vector。

### 3.8.1 FlatVector

```cpp
// src/include/duckdb/common/types/vector.hpp

struct FlatVector {
    // 验证向量类型
    static void VerifyFlatVector(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::FLAT_VECTOR);
    }

    // 获取数据指针
    template <class T>
    static inline T *GetData(Vector &vector) {
        VerifyFlatVector(vector);
        return reinterpret_cast<T *>(vector.data);
    }

    // 获取 ValidityMask
    static inline ValidityMask &Validity(Vector &vector) {
        VerifyFlatVector(vector);
        return vector.validity;
    }

    // 设置单个元素为 NULL
    static void SetNull(Vector &vector, idx_t idx, bool is_null) {
        VerifyFlatVector(vector);
        vector.validity.Set(idx, !is_null);
    }

    // 获取增量选择向量 [0, 1, 2, ...]
    static const SelectionVector *IncrementalSelectionVector();
};
```

### 3.8.2 ConstantVector

```cpp
struct ConstantVector {
    template <class T>
    static void VerifyVectorType(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR);
    }

    // 获取数据（总是只有一个值）
    template <class T>
    static inline const T *GetData(const Vector &vector) {
        VerifyVectorType<T>(vector);
        return reinterpret_cast<const T *>(vector.data);
    }

    // 检查常量是否为 NULL
    static inline bool IsNull(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR);
        return !vector.validity.RowIsValid(0);
    }

    // 设置为 NULL
    static inline void SetNull(Vector &vector, bool is_null) {
        D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR);
        vector.validity.Set(0, !is_null);
    }

    // 获取全零选择向量（用于展开常量）
    static const SelectionVector *ZeroSelectionVector();
};
```

### 3.8.3 DictionaryVector

```cpp
struct DictionaryVector {
    static void VerifyDictionary(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::DICTIONARY_VECTOR);
    }

    // 获取子向量
    static inline const Vector &Child(const Vector &vector) {
        VerifyDictionary(vector);
        auto &buffer = vector.auxiliary->Cast<VectorChildBuffer>();
        return buffer.data;
    }

    // 获取选择向量
    static inline const SelectionVector &SelVector(const Vector &vector) {
        VerifyDictionary(vector);
        auto &buffer = vector.buffer->Cast<DictionaryBuffer>();
        return buffer.GetSelVector();
    }

    // 获取字典大小
    static inline optional_idx DictionarySize(const Vector &vector) {
        VerifyDictionary(vector);
        auto &buffer = vector.buffer->Cast<DictionaryBuffer>();
        return buffer.GetDictionarySize();
    }
};
```

### 3.8.4 ListVector

```cpp
struct ListVector {
    // 获取列表数据
    static inline const list_entry_t *GetData(const Vector &v) {
        if (v.GetVectorType() == VectorType::DICTIONARY_VECTOR) {
            auto &child = DictionaryVector::Child(v);
            return GetData(child);
        }
        return FlatVector::GetData<list_entry_t>(v);
    }

    // 获取列表子向量
    static const Vector &GetEntry(const Vector &vector);
    static Vector &GetEntry(Vector &vector);

    // 获取列表总大小
    static idx_t GetListSize(const Vector &vector);

    // 设置列表大小
    static void SetListSize(Vector &vec, idx_t size);

    // 预留空间
    static void Reserve(Vector &vec, idx_t required_capacity);

    // 追加数据
    static void Append(Vector &target, const Vector &source,
                       idx_t source_size, idx_t source_offset = 0);
};

// list_entry_t 结构
struct list_entry_t {
    uint64_t offset;  // 子向量中的起始位置
    uint64_t length;  // 列表长度
};
```

---

## 3.9 Vector Operations：向量操作

DuckDB 提供了丰富的向量操作函数，这些函数针对批量处理进行了优化。

### 3.9.1 VectorOperations 概览

```cpp
// src/include/duckdb/common/vector_operations/vector_operations.hpp

struct VectorOperations {
    // 复制操作
    static void Copy(const Vector &source, Vector &target,
                     idx_t source_count, idx_t source_offset, idx_t target_offset);
    static void Copy(const Vector &source, Vector &target,
                     const SelectionVector &sel, idx_t source_count,
                     idx_t source_offset, idx_t target_offset);

    // 生成序列
    static void GenerateSequence(Vector &result, idx_t count, int64_t start, int64_t increment);
    static void GenerateSequence(Vector &result, idx_t count, const SelectionVector &sel,
                                 int64_t start, int64_t increment);

    // 哈希操作
    static void Hash(Vector &input, Vector &hashes, idx_t count);
    static void CombineHash(Vector &hashes, Vector &input, idx_t count);

    // 类型转换
    static bool TryCast(CastFunctionSet &set, GetCastFunctionInput &input,
                        Vector &source, Vector &result, idx_t count,
                        CastParameters &parameters, optional_ptr<Vector> result_vector);

    // 存储序列化
    static void WriteToStorage(Vector &source, idx_t count, data_ptr_t target);
    static void ReadFromStorage(data_ptr_t source, idx_t count, Vector &result);
};
```

### 3.9.2 Copy 操作实现

```cpp
// src/common/vector_operations/vector_copy.cpp

void VectorOperations::Copy(const Vector &source_p, Vector &target,
                            const SelectionVector &sel_p, idx_t source_count,
                            idx_t source_offset, idx_t target_offset, idx_t copy_count) {
    SelectionVector owned_sel;
    const SelectionVector *sel = &sel_p;
    const Vector *source = &source_p;
    bool finished = false;

    // 解开嵌套的字典向量
    while (!finished) {
        switch (source->GetVectorType()) {
        case VectorType::DICTIONARY_VECTOR: {
            auto &child = DictionaryVector::Child(*source);
            auto &dict_sel = DictionaryVector::SelVector(*source);
            // 合并选择向量
            auto new_buffer = dict_sel.Slice(*sel, source_count);
            owned_sel.Initialize(new_buffer);
            sel = &owned_sel;
            source = &child;
            break;
        }
        case VectorType::SEQUENCE_VECTOR: {
            // 生成序列然后复制
            int64_t start, increment;
            SequenceVector::GetSequence(*source, start, increment);
            Vector seq(source->GetType());
            VectorOperations::GenerateSequence(seq, source_count, *sel, start, increment);
            VectorOperations::Copy(seq, target, *sel, source_count, source_offset, target_offset);
            return;
        }
        case VectorType::CONSTANT_VECTOR:
            // 使用全零选择向量
            sel = ConstantVector::ZeroSelectionVector(copy_count, owned_sel);
            finished = true;
            break;
        case VectorType::FLAT_VECTOR:
        case VectorType::FSST_VECTOR:
            finished = true;
            break;
        }
    }

    // 复制 NULL 掩码
    auto &tmask = FlatVector::Validity(target);
    if (source->GetVectorType() == VectorType::CONSTANT_VECTOR) {
        const bool valid = !ConstantVector::IsNull(*source);
        for (idx_t i = 0; i < copy_count; i++) {
            tmask.Set(target_offset + i, valid);
        }
    } else {
        auto &smask = FlatVector::Validity(*source);
        tmask.CopySel(smask, *sel, source_offset, target_offset, copy_count);
    }

    // 复制数据（按类型特化）
    switch (source->GetType().InternalType()) {
    case PhysicalType::INT32:
        TemplatedCopy<int32_t>(*source, *sel, target, source_offset, target_offset, copy_count);
        break;
    case PhysicalType::INT64:
        TemplatedCopy<int64_t>(*source, *sel, target, source_offset, target_offset, copy_count);
        break;
    case PhysicalType::VARCHAR:
        // 字符串需要特殊处理
        CopyStrings(*source, *sel, target, source_offset, target_offset, copy_count);
        break;
    // ... 其他类型
    }
}
```

### 3.9.3 模板化的批量操作

```cpp
// 模板化复制
template <class T>
void TemplatedCopy(const Vector &source, const SelectionVector &sel, Vector &target,
                   idx_t source_offset, idx_t target_offset, idx_t copy_count) {
    auto ldata = FlatVector::GetData<T>(source);
    auto tdata = FlatVector::GetData<T>(target);
    for (idx_t i = 0; i < copy_count; i++) {
        auto source_idx = sel.get_index(source_offset + i);
        tdata[target_offset + i] = ldata[source_idx];
    }
}

// 模板化比较
template <class T, class OP>
void TemplatedCompare(Vector &left, Vector &right, Vector &result, idx_t count) {
    UnifiedVectorFormat left_data, right_data;
    left.ToUnifiedFormat(count, left_data);
    right.ToUnifiedFormat(count, right_data);

    auto ldata = UnifiedVectorFormat::GetData<T>(left_data);
    auto rdata = UnifiedVectorFormat::GetData<T>(right_data);
    auto result_data = FlatVector::GetData<bool>(result);

    for (idx_t i = 0; i < count; i++) {
        auto left_idx = left_data.sel->get_index(i);
        auto right_idx = right_data.sel->get_index(i);

        if (!left_data.validity.RowIsValid(left_idx) ||
            !right_data.validity.RowIsValid(right_idx)) {
            FlatVector::SetNull(result, i, true);
        } else {
            result_data[i] = OP::Operation(ldata[left_idx], rdata[right_idx]);
        }
    }
}
```

---

## 3.10 性能优化技术

### 3.10.1 向量化 vs 逐行处理

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    向量化 vs 逐行处理性能对比                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  逐行处理 (Tuple-at-a-time)                                              │
│  ─────────────────────────                                              │
│  while (hasNext()) {                                                    │
│      tuple = getNext();         // 函数调用开销                         │
│      result = process(tuple);   // 虚函数调用                           │
│      emit(result);              // 输出开销                             │
│  }                                                                      │
│                                                                         │
│  问题：                                                                  │
│  • 每行一次函数调用 → 函数调用开销                                        │
│  • 每行一次虚函数调用 → 分支预测失败                                      │
│  • 缓存未充分利用 → 内存访问延迟                                         │
│                                                                         │
│  向量化处理 (Vector-at-a-time)                                           │
│  ─────────────────────────────                                          │
│  while (hasNextChunk()) {                                               │
│      chunk = getNextChunk();    // 每 2048 行一次调用                   │
│      result = process(chunk);   // 批量处理                             │
│      emit(result);                                                      │
│  }                                                                      │
│                                                                         │
│  process(chunk):                                                        │
│      for (i = 0; i < chunk.size(); i++) {                              │
│          // 简单循环，编译器可以优化                                     │
│          result[i] = input[i] + constant;                              │
│      }                                                                  │
│                                                                         │
│  优势：                                                                  │
│  • 函数调用开销分摊到 2048 行                                            │
│  • 循环可以被编译器向量化 (SIMD)                                         │
│  • 数据在缓存中，减少内存访问延迟                                        │
│  • 分支预测更准确（紧凑循环）                                            │
│                                                                         │
│  性能提升：通常 5-10x，某些场景可达 50x                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.10.2 SIMD 优化

```cpp
// DuckDB 在多处使用 SIMD 优化
// 例如：位图操作

// 批量检查 ValidityMask
bool ValidityMask::CheckAllValid(idx_t start, idx_t count) {
    // 检查整个 uint64_t 块
    idx_t start_entry = start / BITS_PER_VALUE;
    idx_t end_entry = (start + count - 1) / BITS_PER_VALUE;

    for (idx_t entry = start_entry; entry <= end_entry; entry++) {
        if (validity_mask[entry] != MAX_ENTRY) {
            return false;
        }
    }
    return true;
}

// 批量设置 ValidityMask
void ValidityMask::SetAllValid(idx_t count) {
    if (!validity_mask) {
        return;  // 已经全部有效
    }
    idx_t entry_count = EntryCount(count);
    // 使用 memset 批量设置（编译器会优化为 SIMD）
    memset(validity_mask, 0xFF, entry_count * sizeof(validity_t));
}
```

### 3.10.3 延迟物化

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        延迟物化策略                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景：SELECT a, b FROM t WHERE c > 100                                 │
│                                                                         │
│  早期物化 (Early Materialization)                                        │
│  ────────────────────────────────                                       │
│  1. 扫描所有列 (a, b, c)                                                │
│  2. 过滤 c > 100，复制满足条件的行                                       │
│  3. 投影选择 (a, b)                                                     │
│                                                                         │
│  延迟物化 (Late Materialization)                                         │
│  ────────────────────────────────                                       │
│  1. 只扫描 c 列                                                         │
│  2. 过滤 c > 100，获得 SelectionVector                                  │
│  3. 用 SelectionVector 扫描 (a, b) 列                                   │
│                                                                         │
│  DuckDB 实现：                                                           │
│  ────────────                                                           │
│  // Filter 操作不复制数据，只更新 SelectionVector                        │
│  void PhysicalFilter::Execute(DataChunk &input) {                       │
│      // 评估过滤条件                                                     │
│      SelectionVector sel(STANDARD_VECTOR_SIZE);                         │
│      idx_t result_count = 0;                                            │
│      for (idx_t i = 0; i < input.size(); i++) {                        │
│          if (EvaluateCondition(input, i)) {                             │
│              sel.set_index(result_count++, i);                          │
│          }                                                              │
│      }                                                                  │
│      // 不复制数据，只切片                                               │
│      input.Slice(sel, result_count);                                    │
│  }                                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3.11 总结

本章深入分析了 DuckDB 向量化执行的基础数据结构：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    向量化执行核心组件总结                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  组件                    职责                       关键特性             │
│  ────────────────────────────────────────────────────────────────────── │
│  Vector                 列数据容器                 多种 VectorType 优化 │
│  VectorType             向量存储类型               FLAT/CONST/DICT/...  │
│  DataChunk              行批次容器                 包含多个 Vector      │
│  SelectionVector        索引向量                   延迟物化支持         │
│  ValidityMask           NULL 值掩码                位图压缩             │
│  VectorBuffer           内存管理                   多种 Buffer 类型     │
│  UnifiedVectorFormat    统一访问接口               类型无关访问         │
│                                                                         │
│  设计原则：                                                              │
│  ────────                                                               │
│  1. 批量处理：每次处理 2048 行，分摊函数调用开销                          │
│  2. 列式存储：相同类型数据连续存储，缓存友好                              │
│  3. 延迟物化：使用 SelectionVector 避免不必要的数据复制                  │
│  4. NULL 优化：位图压缩 + 延迟分配                                       │
│  5. 类型特化：模板化实现，编译时类型检查                                  │
│  6. 内存效率：引用计数 + 智能指针管理生命周期                            │
│                                                                         │
│  性能影响：                                                              │
│  ────────                                                               │
│  • 向量化处理比逐行处理快 5-10x                                          │
│  • SelectionVector 避免 90%+ 的数据复制                                 │
│  • 位图 ValidityMask 节省 87.5% 的 NULL 标记空间                        │
│  • UnifiedVectorFormat 简化代码同时保持性能                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 核心源文件索引

| 组件 | 主要文件 |
|------|----------|
| Vector | `src/include/duckdb/common/types/vector.hpp` |
| Vector 实现 | `src/common/types/vector.cpp` |
| DataChunk | `src/include/duckdb/common/types/data_chunk.hpp` |
| SelectionVector | `src/include/duckdb/common/types/selection_vector.hpp` |
| ValidityMask | `src/include/duckdb/common/types/validity_mask.hpp` |
| VectorBuffer | `src/include/duckdb/common/types/vector_buffer.hpp` |
| VectorType | `src/include/duckdb/common/enums/vector_type.hpp` |
| VectorOperations | `src/common/vector_operations/*.cpp` |

---

## 下一章预告

第四章将深入分析 **表达式执行器（ExpressionExecutor）**，探讨 DuckDB 如何在向量化环境中高效执行标量函数、聚合函数和条件表达式。
