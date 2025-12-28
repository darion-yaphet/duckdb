# DuckDB 执行引擎深度解析：第二章 向量化数据结构

向量化执行的核心在于高效的数据结构设计。DuckDB 通过精心设计的 `Vector`、`DataChunk`、`SelectionVector` 和 `ValidityMask` 等数据结构，实现了高性能的批量数据处理。本章将深入剖析这些核心数据结构的设计原理和实现细节。

---

## 2.1 Vector：向量化的基石

`Vector` 是 DuckDB 向量化执行的核心数据结构，它表示一列同类型数据的批量集合。

### 2.1.1 Vector 核心结构

```cpp
// src/include/duckdb/common/types/vector.hpp

class Vector {
protected:
    //! 向量类型：决定数据如何存储和访问
    VectorType vector_type;

    //! 逻辑类型：表示数据的语义类型（如 INTEGER, VARCHAR）
    LogicalType type;

    //! 数据指针：指向实际数据存储
    data_ptr_t data;

    //! 有效性掩码：标记每个位置是否为 NULL
    ValidityMask validity;

    //! 主缓冲区：持有数据的内存
    buffer_ptr<VectorBuffer> buffer;

    //! 辅助缓冲区：用于存储额外数据（如字符串堆）
    buffer_ptr<VectorBuffer> auxiliary;
};
```

**结构可视化**：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Vector                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  vector_type: VectorType (FLAT, CONSTANT, DICTIONARY, ...)      │
│  type: LogicalType (INTEGER, VARCHAR, STRUCT, ...)              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ data ──────────────────────────────────────────────┐    │    │
│  │                                                     ↓    │    │
│  │  ┌──────┬──────┬──────┬──────┬──────┬─────┬──────┐      │    │
│  │  │ v[0] │ v[1] │ v[2] │ v[3] │ v[4] │ ... │v[2047]│      │    │
│  │  └──────┴──────┴──────┴──────┴──────┴─────┴──────┘      │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ validity: ValidityMask                                  │    │
│  │  ┌────────────────────────────────────────────────┐     │    │
│  │  │ 1111111111111111111111111111111111111111101...  │     │    │
│  │  │ (位图: 1=有效, 0=NULL)                          │     │    │
│  │  └────────────────────────────────────────────────┘     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  buffer: VectorBuffer (数据所有权)                               │
│  auxiliary: VectorBuffer (辅助数据，如字符串堆)                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 2.1.2 VectorType：向量存储类型

DuckDB 定义了多种向量类型，每种类型有不同的存储和访问方式：

```cpp
// src/include/duckdb/common/enums/vector_type.hpp

enum class VectorType : uint8_t {
    FLAT_VECTOR,       // 扁平向量：标准未压缩存储
    FSST_VECTOR,       // FSST 向量：字符串 FSST 压缩
    CONSTANT_VECTOR,   // 常量向量：所有值相同
    DICTIONARY_VECTOR, // 字典向量：通过索引引用另一个向量
    SEQUENCE_VECTOR    // 序列向量：等差数列
};
```

**各类型详解**：

#### 1. FLAT_VECTOR（扁平向量）

最基本的向量类型，数据连续存储在内存中：

```
FLAT_VECTOR: [10, 20, 30, 40, 50, 60, 70, 80, ...]
              ↑
             data 指针直接指向连续数据

访问方式: data[i] (O(1) 直接访问)
内存占用: N × sizeof(T)
适用场景: 大多数情况，数据各不相同
```

```cpp
// 获取 FLAT_VECTOR 的数据
template <class T>
static inline T *GetData(Vector &vector) {
    D_ASSERT(vector.GetVectorType() == VectorType::FLAT_VECTOR);
    return reinterpret_cast<T *>(vector.data);
}

// 使用示例
auto data = FlatVector::GetData<int32_t>(vector);
for (idx_t i = 0; i < count; i++) {
    result[i] = data[i] * 2;  // 直接访问
}
```

#### 2. CONSTANT_VECTOR（常量向量）

所有位置都是同一个值，只存储一次：

```
CONSTANT_VECTOR:
  data[0] = 42

  逻辑视图: [42, 42, 42, 42, 42, 42, 42, 42, ...]
             ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑
            都引用 data[0]

内存占用: sizeof(T) (只存储一个值)
适用场景: 常量表达式，如 WHERE x = 100 中的 100
```

```cpp
// 常量向量操作
static inline const_data_ptr_t GetData(const Vector &vector) {
    D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR);
    return vector.data;  // 返回单一值的指针
}

static inline bool IsNull(const Vector &vector) {
    D_ASSERT(vector.GetVectorType() == VectorType::CONSTANT_VECTOR);
    return !vector.validity.RowIsValid(0);  // 只检查第一个位置
}

// 将值引用为常量
static void Reference(Vector &vector, Vector &source, idx_t position, idx_t count) {
    // 将 source[position] 作为常量引用
    vector.SetVectorType(VectorType::CONSTANT_VECTOR);
    // ...
}
```

#### 3. DICTIONARY_VECTOR（字典向量）

通过选择向量（索引数组）引用另一个向量：

```
DICTIONARY_VECTOR:

  Child Vector (被引用的向量):
    ["apple", "banana", "cherry", "date"]
       [0]       [1]       [2]      [3]

  Selection Vector (索引数组):
    [1, 2, 0, 2, 1, 3, 0, 1, ...]

  逻辑视图:
    ["banana", "cherry", "apple", "cherry", "banana", "date", "apple", "banana", ...]
        ↑          ↑        ↑         ↑         ↑        ↑        ↑        ↑
       [1]        [2]      [0]       [2]       [1]      [3]      [0]      [1]

优势:
- 压缩重复值
- 避免数据拷贝
- 实现过滤时只修改索引
```

```cpp
// 字典向量操作
struct DictionaryVector {
    // 获取选择向量
    static inline const SelectionVector &SelVector(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::DICTIONARY_VECTOR);
        return vector.buffer->Cast<DictionaryBuffer>().GetSelVector();
    }

    // 获取子向量（实际数据）
    static inline const Vector &Child(const Vector &vector) {
        D_ASSERT(vector.GetVectorType() == VectorType::DICTIONARY_VECTOR);
        return vector.auxiliary->Cast<VectorChildBuffer>().data;
    }
};

// 访问字典向量元素
auto &sel = DictionaryVector::SelVector(vector);
auto &child = DictionaryVector::Child(vector);
auto child_data = FlatVector::GetData<string_t>(child);
for (idx_t i = 0; i < count; i++) {
    auto value = child_data[sel.get_index(i)];  // 间接访问
}
```

#### 4. SEQUENCE_VECTOR（序列向量）

存储等差数列，只需起始值和增量：

```
SEQUENCE_VECTOR:
  start = 100
  increment = 5

  逻辑视图: [100, 105, 110, 115, 120, 125, ...]
             ↑    ↑    ↑    ↑    ↑    ↑
            100  100+5 100+10 ...

内存占用: 只需 3 个 int64_t (start, increment, count)
适用场景: ROW_NUMBER(), ROWID 等序列生成
```

```cpp
// 序列向量操作
struct SequenceVector {
    static void GetSequence(const Vector &vector,
                           int64_t &start,
                           int64_t &increment,
                           int64_t &sequence_count) {
        D_ASSERT(vector.GetVectorType() == VectorType::SEQUENCE_VECTOR);
        auto data = reinterpret_cast<int64_t *>(vector.buffer->GetData());
        start = data[0];
        increment = data[1];
        sequence_count = data[2];
    }
};

// 创建序列向量
void Vector::Sequence(int64_t start, int64_t increment, idx_t count) {
    SetVectorType(VectorType::SEQUENCE_VECTOR);
    // 存储 start, increment, count
}
```

#### 5. FSST_VECTOR（FSST压缩向量）

字符串数据使用 FSST 算法压缩存储：

```
FSST_VECTOR:
  压缩字符串数据 + FSST 解码器

  解压后才能读取实际字符串值
  适用于重复模式多的字符串数据
```

### 2.1.3 向量类型转换

不同类型的向量可以统一转换为标准格式进行处理：

```cpp
// UnifiedVectorFormat: 统一访问格式
struct UnifiedVectorFormat {
    const SelectionVector *sel;  // 选择向量
    data_ptr_t data;             // 数据指针
    ValidityMask validity;       // 有效性掩码
    PhysicalType physical_type;  // 物理类型
};

// 将任意向量转换为统一格式
void Vector::ToUnifiedFormat(idx_t count, UnifiedVectorFormat &data) {
    switch (GetVectorType()) {
    case VectorType::FLAT_VECTOR:
        // 扁平向量：使用增量选择向量
        data.sel = FlatVector::IncrementalSelectionVector();
        data.data = this->data;
        data.validity = this->validity;
        break;
    case VectorType::CONSTANT_VECTOR:
        // 常量向量：使用全零选择向量
        data.sel = ConstantVector::ZeroSelectionVector();
        data.data = this->data;
        data.validity = this->validity;
        break;
    case VectorType::DICTIONARY_VECTOR:
        // 字典向量：使用字典的选择向量
        data.sel = &DictionaryVector::SelVector(*this);
        data.data = DictionaryVector::Child(*this).data;
        // ...
        break;
    // ...
    }
}
```

**统一访问模式**：

```cpp
// 使用 UnifiedVectorFormat 处理任意类型向量
void ProcessAnyVector(Vector &input, idx_t count) {
    UnifiedVectorFormat format;
    input.ToUnifiedFormat(count, format);

    auto data = UnifiedVectorFormat::GetData<int32_t>(format);
    for (idx_t i = 0; i < count; i++) {
        auto idx = format.sel->get_index(i);
        if (format.validity.RowIsValid(idx)) {
            // 处理 data[idx]
        }
    }
}
```

### 2.1.4 Flatten：向量扁平化

将任何类型的向量转换为 FLAT_VECTOR：

```cpp
// 扁平化操作
void Vector::Flatten(idx_t count) {
    switch (GetVectorType()) {
    case VectorType::FLAT_VECTOR:
        // 已经是扁平向量，无需操作
        break;
    case VectorType::CONSTANT_VECTOR:
        // 常量向量：复制值到所有位置
        // 把单个值展开成 count 个副本
        break;
    case VectorType::DICTIONARY_VECTOR: {
        // 字典向量：按选择向量重新排列数据
        auto &sel = DictionaryVector::SelVector(*this);
        auto &child = DictionaryVector::Child(*this);
        // 按 sel 从 child 拷贝数据到新的扁平数组
        break;
    }
    // ...
    }
    SetVectorType(VectorType::FLAT_VECTOR);
}
```

---

## 2.2 DataChunk：批量数据容器

`DataChunk` 是一个包含多个 `Vector` 的容器，表示一个"行组"的列式数据。

### 2.2.1 DataChunk 结构

```cpp
// src/include/duckdb/common/types/data_chunk.hpp

class DataChunk {
public:
    //! 列向量数组
    vector<Vector> data;

private:
    //! 当前有效行数 (0 ~ 2048)
    idx_t count;
    //! 容量
    idx_t capacity;
    //! 初始容量（用于 Reset）
    idx_t initial_capacity;
    //! 向量缓存
    vector<VectorCache> vector_caches;
};
```

**可视化结构**：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              DataChunk                                       │
│                                                                              │
│  count = 1847                    (当前行数)                                  │
│  capacity = 2048                 (最大容量 = STANDARD_VECTOR_SIZE)          │
│  ColumnCount() = 4               (列数)                                      │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ data[0]: Vector (INTEGER)                                             │  │
│  │ ┌───┬───┬───┬───┬───┬─────┬───────┐                                   │  │
│  │ │ 1 │ 2 │ 3 │ 4 │ 5 │ ... │ 1847  │                                   │  │
│  │ └───┴───┴───┴───┴───┴─────┴───────┘                                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ data[1]: Vector (VARCHAR)                                             │  │
│  │ ┌─────┬─────┬─────┬─────┬─────┬─────┬─────────┐                       │  │
│  │ │"foo"│"bar"│"baz"│ ... │ ... │ ... │  ...    │                       │  │
│  │ └─────┴─────┴─────┴─────┴─────┴─────┴─────────┘                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ data[2]: Vector (DOUBLE)                                              │  │
│  │ ┌─────┬─────┬─────┬─────┬─────┬─────┬─────────┐                       │  │
│  │ │1.5  │2.7  │3.14 │ ... │ ... │ ... │  ...    │                       │  │
│  │ └─────┴─────┴─────┴─────┴─────┴─────┴─────────┘                       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ data[3]: Vector (BOOLEAN)                                             │  │
│  │ ┌───┬───┬───┬───┬───┬─────┬───────┐                                   │  │
│  │ │ T │ F │ T │ T │ F │ ... │  ...  │                                   │  │
│  │ └───┴───┴───┴───┴───┴─────┴───────┘                                   │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

                    列式存储布局
        ─────────────────────────────────────
        Row 0:  data[0][0], data[1][0], data[2][0], data[3][0]
        Row 1:  data[0][1], data[1][1], data[2][1], data[3][1]
        Row 2:  data[0][2], data[1][2], data[2][2], data[3][2]
        ...
```

### 2.2.2 DataChunk 初始化

```cpp
// 标准初始化方式
void DataChunk::Initialize(Allocator &allocator,
                           const vector<LogicalType> &types,
                           idx_t capacity = STANDARD_VECTOR_SIZE) {
    D_ASSERT(data.empty());
    this->capacity = capacity;
    this->initial_capacity = capacity;
    this->count = 0;

    // 为每一列创建 Vector
    for (auto &type : types) {
        VectorCache cache(allocator, type, capacity);
        data.emplace_back(cache);
        vector_caches.push_back(std::move(cache));
    }
}

// 使用示例
DataChunk chunk;
vector<LogicalType> types = {
    LogicalType::INTEGER,
    LogicalType::VARCHAR,
    LogicalType::DOUBLE
};
chunk.Initialize(allocator, types);
```

### 2.2.3 常用操作

```cpp
// 获取基本信息
idx_t size() const { return count; }                 // 当前行数
idx_t ColumnCount() const { return data.size(); }    // 列数
idx_t GetCapacity() const { return capacity; }       // 容量

// 设置行数
void SetCardinality(idx_t count_p) {
    D_ASSERT(count_p <= capacity);
    this->count = count_p;
}

// 重置（清空数据，保留内存）
void Reset() {
    count = 0;
    capacity = initial_capacity;
    for (idx_t i = 0; i < ColumnCount(); i++) {
        data[i].ResetFromCache(vector_caches[i]);
    }
}

// 引用另一个 DataChunk
void Reference(DataChunk &chunk) {
    count = chunk.count;
    capacity = chunk.capacity;
    for (idx_t i = 0; i < chunk.ColumnCount(); i++) {
        data[i].Reference(chunk.data[i]);
    }
}

// 追加数据
void Append(const DataChunk &other, bool resize = false,
            SelectionVector *sel = nullptr, idx_t count = 0);

// 扁平化所有向量
void Flatten() {
    for (auto &v : data) {
        v.Flatten(count);
    }
}

// 切片操作
void Slice(const SelectionVector &sel_vector, idx_t count) {
    for (auto &v : data) {
        v.Slice(sel_vector, count);
    }
    SetCardinality(count);
}
```

### 2.2.4 DataChunk 在执行中的流转

```cpp
// Pipeline 执行中 DataChunk 的使用
class PipelineExecutor {
    // 中间结果 DataChunk（每个算子一个）
    vector<unique_ptr<DataChunk>> intermediate_chunks;
    // 最终输出 DataChunk
    DataChunk final_chunk;

    OperatorResultType Execute(DataChunk &input, DataChunk &result) {
        // input 是上一个算子的输出
        // result 是当前算子的输出

        for (idx_t i = 0; i < operators.size(); i++) {
            auto &current_chunk = *intermediate_chunks[i];
            current_chunk.Reset();  // 清空准备接收数据

            // 执行算子
            auto result = operators[i].Execute(context,
                                               prev_chunk,
                                               current_chunk,
                                               gstate,
                                               state);
            // current_chunk 现在包含算子输出
        }

        return OperatorResultType::NEED_MORE_INPUT;
    }
};
```

---

## 2.3 SelectionVector：选择向量

`SelectionVector` 是向量化执行中处理控制流的关键数据结构。它存储一个索引数组，指示哪些行是"有效的"或"被选中的"。

### 2.3.1 SelectionVector 结构

```cpp
// src/include/duckdb/common/types/selection_vector.hpp

struct SelectionVector {
    // 索引数组指针
    sel_t *sel_vector;
    // 可选的拥有数据的缓冲区
    buffer_ptr<SelectionData> selection_data;

    // 核心操作
    void set_index(idx_t idx, idx_t loc) {
        sel_vector[idx] = (sel_t)loc;
    }

    idx_t get_index(idx_t idx) const {
        return sel_vector ? sel_vector[idx] : idx;
    }
};
```

**可视化**：

```
SelectionVector:
  sel_vector: [3, 7, 2, 5, 9, 1, 8, ...]
               ↓  ↓  ↓  ↓  ↓  ↓  ↓
  表示选择原数据的第 3, 7, 2, 5, 9, 1, 8, ... 行

原数据:
  Index:  [0]   [1]   [2]   [3]   [4]   [5]   [6]   [7]   [8]   [9]
  Value:  "A"   "B"   "C"   "D"   "E"   "F"   "G"   "H"   "I"   "J"

通过 SelectionVector 访问:
  sel[0]=3 → "D"
  sel[1]=7 → "H"
  sel[2]=2 → "C"
  sel[3]=5 → "F"
  sel[4]=9 → "J"
  sel[5]=1 → "B"
  sel[6]=8 → "I"
  ...
```

### 2.3.2 SelectionVector 的核心作用

#### 1. 过滤操作（避免数据拷贝）

```cpp
// 传统方式：拷贝数据
vector<int> Filter(vector<int> &input, Predicate &pred) {
    vector<int> result;
    for (auto &val : input) {
        if (pred(val)) {
            result.push_back(val);  // 数据拷贝
        }
    }
    return result;
}

// 向量化方式：使用 SelectionVector
idx_t Filter(Vector &input, SelectionVector &sel,
             Predicate &pred, idx_t count) {
    idx_t result_count = 0;
    auto data = FlatVector::GetData<int32_t>(input);

    for (idx_t i = 0; i < count; i++) {
        if (pred(data[i])) {
            sel.set_index(result_count++, i);  // 只记录索引
        }
    }
    return result_count;  // 返回选中的行数
}

// 零拷贝！原数据不变，只是记录了哪些行满足条件
```

**示例：WHERE age > 30**

```
原数据 (Vector):
  Index: [0] [1] [2] [3] [4] [5] [6] [7]
  Age:    25  35  28  42  31  29  55  38

过滤条件: age > 30

SelectionVector (过滤后):
  [1, 3, 4, 6, 7]
   ↓  ↓  ↓  ↓  ↓
  35 42 31 55 38  (这些行满足条件)

选中行数: 5
原数据: 不变
```

#### 2. 字典向量压缩

```cpp
// 通过 SelectionVector 创建字典向量
void Vector::Dictionary(idx_t dictionary_size,
                        const SelectionVector &sel,
                        idx_t count) {
    // 将当前向量转换为字典向量
    // sel 作为索引，当前数据作为字典
}
```

#### 3. Slice 操作

```cpp
// 使用 SelectionVector 对 DataChunk 切片
void DataChunk::Slice(const SelectionVector &sel_vector, idx_t count) {
    for (auto &v : data) {
        v.Slice(sel_vector, count);
    }
    SetCardinality(count);
}

// 示例
SelectionVector sel(3);
sel.set_index(0, 5);
sel.set_index(1, 2);
sel.set_index(2, 8);
chunk.Slice(sel, 3);
// chunk 现在只包含原来的第 5, 2, 8 行
```

### 2.3.3 特殊的 SelectionVector

#### 增量选择向量（用于 FLAT_VECTOR）

```cpp
// 0, 1, 2, 3, 4, 5, ... 的增量序列
// 表示"选择所有行，按顺序"
static const SelectionVector *IncrementalSelectionVector() {
    static sel_t incremental[STANDARD_VECTOR_SIZE];
    static SelectionVector vec(incremental);
    static bool initialized = false;
    if (!initialized) {
        for (idx_t i = 0; i < STANDARD_VECTOR_SIZE; i++) {
            incremental[i] = i;
        }
        initialized = true;
    }
    return &vec;
}
```

#### 零选择向量（用于 CONSTANT_VECTOR）

```cpp
// 全部是 0 的选择向量
// 表示"所有行都引用第 0 个位置"
static const sel_t ZERO_VECTOR[STANDARD_VECTOR_SIZE] = {0, 0, 0, ...};

static const SelectionVector *ZeroSelectionVector() {
    static SelectionVector vec((sel_t *)ZERO_VECTOR);
    return &vec;
}
```

### 2.3.4 SelectionVector 优化

```cpp
// 可选的 SelectionVector 包装器
// 当 sel 为 nullptr 时，假设是增量选择
class OptionalSelection {
    SelectionVector *sel;
    SelectionVector vec;

public:
    void Append(idx_t &count, const idx_t idx) {
        if (sel) {
            sel->set_index(count, idx);
        }
        ++count;  // 无论是否有选择向量都增加计数
    }
};
```

---

## 2.4 ValidityMask：空值处理

`ValidityMask` 是一个位图，用于高效表示 NULL 值。

### 2.4.1 ValidityMask 结构

```cpp
// src/include/duckdb/common/types/validity_mask.hpp

template <typename V>
struct TemplatedValidityMask {
    static constexpr idx_t BITS_PER_VALUE = sizeof(V) * 8;  // 64
    static constexpr idx_t STANDARD_ENTRY_COUNT =
        (STANDARD_VECTOR_SIZE + 63) / 64;  // 32

    // 位图数据：nullptr 表示全部有效
    V *validity_mask;
    // 拥有数据的缓冲区
    buffer_ptr<ValidityBuffer> validity_data;
    // 容量
    idx_t capacity;
};

using validity_t = uint64_t;
struct ValidityMask : public TemplatedValidityMask<validity_t> { ... };
```

**关键设计：AllValid 优化**

```cpp
// 当所有值都有效时，validity_mask 为 nullptr
// 这避免了为全有效数据分配和检查位图

inline bool AllValid() const {
    return !validity_mask;  // nullptr = 全部有效
}

inline bool RowIsValid(idx_t row_idx) const {
    if (!validity_mask) {
        return true;  // 快速路径：全部有效
    }
    return RowIsValidUnsafe(row_idx);  // 慢速路径：检查位图
}
```

### 2.4.2 位图操作

```
ValidityMask 存储结构:

validity_mask[0]: bits 0-63    (第 0-63 行)
validity_mask[1]: bits 64-127  (第 64-127 行)
...
validity_mask[31]: bits 1984-2047 (第 1984-2047 行)

每个 bit: 1 = 有效, 0 = NULL

例如 validity_mask[0] = 0xFFFFFFFFFFFFFFFE
表示第 0 行是 NULL，第 1-63 行都有效
```

```cpp
// 获取 entry 索引和 bit 位置
static inline void GetEntryIndex(idx_t row_idx,
                                 idx_t &entry_idx,
                                 idx_t &idx_in_entry) {
    entry_idx = row_idx / BITS_PER_VALUE;     // row_idx / 64
    idx_in_entry = row_idx % BITS_PER_VALUE;  // row_idx % 64
}

// 检查某行是否有效
inline bool RowIsValidUnsafe(idx_t row_idx) const {
    idx_t entry_idx, idx_in_entry;
    GetEntryIndex(row_idx, entry_idx, idx_in_entry);
    auto entry = validity_mask[entry_idx];
    return (entry >> idx_in_entry) & 1;
}

// 设置某行为无效 (NULL)
inline void SetInvalidUnsafe(idx_t row_idx) {
    idx_t entry_idx, idx_in_entry;
    GetEntryIndex(row_idx, entry_idx, idx_in_entry);
    validity_mask[entry_idx] &= ~(1ULL << idx_in_entry);
}

// 设置某行为有效
inline void SetValidUnsafe(idx_t row_idx) {
    idx_t entry_idx, idx_in_entry;
    GetEntryIndex(row_idx, entry_idx, idx_in_entry);
    validity_mask[entry_idx] |= (1ULL << idx_in_entry);
}
```

### 2.4.3 批量操作

```cpp
// 设置所有行为 NULL
inline void SetAllInvalid(idx_t count) {
    EnsureWritable();
    auto entry_count = EntryCount(count);
    for (idx_t i = 0; i < entry_count; i++) {
        validity_mask[i] = 0;
    }
}

// 设置所有行为有效
inline void SetAllValid(idx_t count) {
    EnsureWritable();
    auto entry_count = EntryCount(count);
    for (idx_t i = 0; i < entry_count; i++) {
        validity_mask[i] = MAX_ENTRY;  // 0xFFFFFFFFFFFFFFFF
    }
}

// 统计有效行数
idx_t CountValid(const idx_t count) const {
    if (AllValid()) {
        return count;  // 快速路径
    }

    idx_t valid = 0;
    auto entry_count = EntryCount(count);
    for (idx_t i = 0; i < entry_count; i++) {
        auto entry = validity_mask[i];
        if (entry == MAX_ENTRY) {
            valid += BITS_PER_VALUE;  // 整个 entry 都有效
        } else {
            // Kernighan 算法：统计 1 的个数
            while (entry) {
                entry &= (entry - 1);
                ++valid;
            }
        }
    }
    return valid;
}
```

### 2.4.4 NULL 处理模式

```cpp
// 向量化函数中的 NULL 处理示例
void AddWithNulls(Vector &left, Vector &right, Vector &result, idx_t count) {
    UnifiedVectorFormat left_format, right_format;
    left.ToUnifiedFormat(count, left_format);
    right.ToUnifiedFormat(count, right_format);

    auto ldata = UnifiedVectorFormat::GetData<int32_t>(left_format);
    auto rdata = UnifiedVectorFormat::GetData<int32_t>(right_format);
    auto result_data = FlatVector::GetData<int32_t>(result);
    auto &result_validity = FlatVector::Validity(result);

    for (idx_t i = 0; i < count; i++) {
        auto lidx = left_format.sel->get_index(i);
        auto ridx = right_format.sel->get_index(i);

        // NULL 传播：任一操作数为 NULL，结果也为 NULL
        if (!left_format.validity.RowIsValid(lidx) ||
            !right_format.validity.RowIsValid(ridx)) {
            result_validity.SetInvalid(i);
        } else {
            result_data[i] = ldata[lidx] + rdata[ridx];
        }
    }
}
```

---

## 2.5 字符串处理

字符串是变长数据类型，需要特殊处理。DuckDB 使用 `string_t` 实现了内联短字符串优化。

### 2.5.1 string_t 结构

```cpp
// src/include/duckdb/common/types/string_type.hpp

struct string_t {
    static constexpr idx_t PREFIX_BYTES = 4;     // 前缀长度
    static constexpr idx_t INLINE_BYTES = 12;    // 内联容量
    static constexpr idx_t HEADER_SIZE =
        sizeof(uint32_t) + PREFIX_BYTES;         // 8 bytes

    union {
        struct {
            uint32_t length;
            char inlined[INLINE_BYTES];  // 短字符串内联存储
        } inlined;

        struct {
            uint32_t length;
            char prefix[PREFIX_BYTES];    // 前 4 个字符（用于快速比较）
            char *ptr;                    // 指向实际数据
        } pointer;
    } value;

    bool IsInlined() const {
        return value.inlined.length <= INLINE_LENGTH;
    }
};
```

**内存布局**：

```
string_t 大小: 16 bytes

短字符串 (≤12 bytes):
┌───────────────────────────────────────────────────────────────┐
│ length (4B) │              inlined data (12B)                 │
│     8       │ 'H' 'e' 'l' 'l' 'o' ' ' 'B' 'o' 'b' 0  0  0     │
└───────────────────────────────────────────────────────────────┘
  直接存储，无需额外内存分配

长字符串 (>12 bytes):
┌───────────────────────────────────────────────────────────────┐
│ length (4B) │ prefix (4B) │        pointer (8B)               │
│     25      │ 'T' 'h' 'i' │ ──────────────────┐               │
└─────────────────────────────────────────────│─────────────────┘
                                              ↓
                              ┌────────────────────────────────────┐
                              │ "This is a longer string..."      │
                              │ (存储在 StringHeap 中)              │
                              └────────────────────────────────────┘
```

### 2.5.2 string_t 的优势

```cpp
// 1. 短字符串无需堆分配
string_t short_str("Hello");  // 5 bytes < 12, 内联存储

// 2. 快速比较（先比较前缀）
bool FastCompare(const string_t &a, const string_t &b) {
    if (a.GetSize() != b.GetSize()) return false;
    // 先比较前 4 个字符
    if (memcmp(a.GetPrefix(), b.GetPrefix(), PREFIX_LENGTH) != 0) {
        return false;  // 快速返回
    }
    // 前缀相同，再比较完整内容
    return memcmp(a.GetData(), b.GetData(), a.GetSize()) == 0;
}

// 3. 统一的访问接口
const char *GetData() const {
    return IsInlined() ? value.inlined.inlined : value.pointer.ptr;
}

idx_t GetSize() const {
    return value.inlined.length;
}
```

### 2.5.3 StringVector 和 StringHeap

```cpp
// 字符串向量操作
struct StringVector {
    // 添加字符串到向量的堆中
    static string_t AddString(Vector &vector, const char *data, idx_t len) {
        if (len <= string_t::INLINE_LENGTH) {
            // 短字符串：直接内联
            return string_t(data, len);
        }
        // 长字符串：添加到堆
        auto &buffer = GetStringBuffer(vector);
        auto result = buffer.AddBlob(data, len);
        return string_t((const char *)result, len);
    }

    // 分配空字符串空间
    static string_t EmptyString(Vector &vector, idx_t len) {
        auto &buffer = GetStringBuffer(vector);
        return buffer.EmptyString(len);
    }

    // 添加堆引用
    static void AddHeapReference(Vector &vector, Vector &other);
};
```

---

## 2.6 复杂类型向量

DuckDB 支持嵌套的复杂类型：LIST、STRUCT、MAP 等。

### 2.6.1 LIST 向量

```cpp
// LIST 向量结构
// 主向量存储 list_entry_t (offset, length)
// 子向量存储实际元素

struct list_entry_t {
    uint64_t offset;   // 在子向量中的起始位置
    uint64_t length;   // 列表长度
};
```

**可视化**：

```
LIST 向量表示: [[1,2], [3], [4,5,6]]

主向量 (list_entry_t[]):
  [0]: offset=0, length=2   → 子向量[0:2]  = [1,2]
  [1]: offset=2, length=1   → 子向量[2:3]  = [3]
  [2]: offset=3, length=3   → 子向量[3:6]  = [4,5,6]

子向量 (INTEGER[]):
  [0] [1] [2] [3] [4] [5]
   1   2   3   4   5   6
```

```cpp
struct ListVector {
    // 获取子向量
    static Vector &GetEntry(Vector &vector);
    // 获取子向量大小
    static idx_t GetListSize(const Vector &vector);
    // 设置子向量大小
    static void SetListSize(Vector &vec, idx_t size);
};

// 使用示例
auto &entry = ListVector::GetEntry(list_vec);
auto list_data = FlatVector::GetData<list_entry_t>(list_vec);
auto child_data = FlatVector::GetData<int32_t>(entry);

for (idx_t i = 0; i < count; i++) {
    auto &list = list_data[i];
    for (idx_t j = 0; j < list.length; j++) {
        auto value = child_data[list.offset + j];
        // 处理列表元素
    }
}
```

### 2.6.2 STRUCT 向量

```cpp
// STRUCT 向量：每个字段是一个子向量
struct StructVector {
    static vector<unique_ptr<Vector>> &GetEntries(Vector &vector);
};
```

**可视化**：

```
STRUCT 向量表示: [{a:1, b:"x"}, {a:2, b:"y"}, {a:3, b:"z"}]

主向量 (STRUCT):
  ├── 子向量 0 (a: INTEGER): [1, 2, 3]
  └── 子向量 1 (b: VARCHAR): ["x", "y", "z"]

所有子向量具有相同的行数
```

### 2.6.3 MAP 向量

```cpp
// MAP 基于 LIST<STRUCT{key, value}> 实现
struct MapVector {
    static const Vector &GetKeys(const Vector &vector);
    static const Vector &GetValues(const Vector &vector);
};
```

---

## 2.7 本章小结

本章详细介绍了 DuckDB 向量化执行的核心数据结构：

1. **Vector**：向量化执行的基石，支持多种存储类型（FLAT、CONSTANT、DICTIONARY、SEQUENCE、FSST），通过 `ToUnifiedFormat` 提供统一访问接口。

2. **DataChunk**：批量数据容器，包含多个 Vector，每批最多 2048 行，是算子间数据传递的基本单位。

3. **SelectionVector**：选择向量，通过索引数组实现零拷贝的过滤和切片操作，是向量化执行处理控制流的关键。

4. **ValidityMask**：位图表示 NULL 值，使用 `AllValid` 优化避免不必要的空值检查。

5. **string_t**：智能字符串类型，短字符串（≤12字节）内联存储，长字符串存储指针和前缀，实现高效的字符串操作。

6. **复杂类型**：LIST、STRUCT、MAP 等嵌套类型通过子向量实现，保持列式存储的优势。

这些数据结构的设计充分考虑了：
- **缓存友好**：连续内存访问、适当的批量大小
- **零拷贝**：引用语义、SelectionVector 避免数据移动
- **空间效率**：常量向量、字典向量、内联字符串
- **统一接口**：UnifiedVectorFormat 屏蔽存储差异

下一章我们将探讨如何使用这些数据结构实现高效的表达式执行。

---

## 源码参考

| 文件 | 描述 |
|------|------|
| `src/include/duckdb/common/types/vector.hpp` | Vector 类定义 |
| `src/common/types/vector.cpp` | Vector 实现 (107KB) |
| `src/include/duckdb/common/types/data_chunk.hpp` | DataChunk 定义 |
| `src/common/types/data_chunk.cpp` | DataChunk 实现 |
| `src/include/duckdb/common/types/selection_vector.hpp` | SelectionVector 定义 |
| `src/include/duckdb/common/types/validity_mask.hpp` | ValidityMask 定义 |
| `src/include/duckdb/common/types/string_type.hpp` | string_t 定义 |
| `src/include/duckdb/common/enums/vector_type.hpp` | VectorType 枚举 |
