# DuckDB 存储引擎深度解析 - 第四章：压缩算法

## 4.1 概述

压缩是 DuckDB 存储引擎的核心特性之一。通过智能的压缩算法选择，DuckDB 可以显著减少存储空间占用，同时保持甚至提升查询性能。本章将深入剖析 DuckDB 的压缩框架、各种压缩算法的实现原理以及自动压缩选择机制。

### 4.1.1 压缩类型总览

```cpp
enum class CompressionType : uint8_t {
    COMPRESSION_AUTO = 0,           // 自动选择
    COMPRESSION_UNCOMPRESSED = 1,   // 不压缩
    COMPRESSION_CONSTANT = 2,       // 常量压缩（内部使用）
    COMPRESSION_RLE = 3,            // 游程编码
    COMPRESSION_DICTIONARY = 4,     // 字典压缩
    COMPRESSION_PFOR_DELTA = 5,     // PFOR-DELTA（已弃用）
    COMPRESSION_BITPACKING = 6,     // 位打包
    COMPRESSION_FSST = 7,           // 快速静态符号表
    COMPRESSION_CHIMP = 8,          // Chimp（浮点）
    COMPRESSION_PATAS = 9,          // Patas（浮点）
    COMPRESSION_ALP = 10,           // ALP（浮点）
    COMPRESSION_ALPRD = 11,         // ALP Real Double（浮点）
    COMPRESSION_ZSTD = 12,          // ZSTD 通用压缩
    COMPRESSION_ROARING = 13,       // Roaring Bitmap
    COMPRESSION_EMPTY = 14,         // 空值压缩（内部使用）
    COMPRESSION_DICT_FSST = 15,     // 字典 + FSST 组合
};
```

### 4.1.2 压缩算法适用性

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     压缩算法适用性矩阵                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  数据类型        适用算法                                                │
│  ───────────────────────────────────────────────────────────────────    │
│  整数类型        Constant, Uncompressed, RLE, Bitpacking                │
│  (INT8-INT64)                                                           │
│                                                                         │
│  浮点类型        Constant, Uncompressed, Chimp, Patas, ALP, ALPRD       │
│  (FLOAT/DOUBLE)                                                         │
│                                                                         │
│  字符串          Uncompressed, Dictionary, FSST, Dict+FSST              │
│  (VARCHAR)                                                              │
│                                                                         │
│  布尔类型        Constant, Uncompressed, Roaring                        │
│  (BOOL/BIT)                                                             │
│                                                                         │
│  有效性掩码      Empty, Uncompressed, Roaring                           │
│  (Validity)                                                             │
│                                                                         │
│  通用后备        ZSTD                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 4.2 压缩函数框架

### 4.2.1 CompressionFunction 接口

CompressionFunction 是所有压缩算法的统一接口，定义在 `src/include/duckdb/function/compression_function.hpp`：

```cpp
class CompressionFunction {
public:
    //! 压缩类型
    CompressionType type;

    //! 支持的物理类型
    PhysicalType data_type;

    //===--- 分析阶段（选择最佳压缩） ---===//

    //! 初始化分析状态
    compression_init_analyze_t init_analyze;

    //! 分析数据（逐向量调用）
    //! 返回 false 表示放弃此压缩方法
    compression_analyze_t analyze;

    //! 完成分析，返回预估压缩大小
    compression_final_analyze_t final_analyze;

    //===--- 压缩阶段 ---===//

    //! 初始化压缩状态
    compression_init_compression_t init_compression;

    //! 压缩数据（逐向量调用）
    compression_compress_data_t compress;

    //! 完成压缩
    compression_compress_finalize_t compress_finalize;

    //===--- 扫描阶段 ---===//

    //! 初始化预取状态
    compression_init_prefetch_t init_prefetch;

    //! 初始化段扫描状态
    compression_init_segment_scan_t init_scan;

    //! 扫描完整向量
    compression_scan_vector_t scan_vector;

    //! 扫描部分数据
    compression_scan_partial_t scan_partial;

    //! 带选择向量的扫描
    compression_select_t select;

    //! 带过滤器的扫描
    compression_filter_t filter;

    //! 获取单行
    compression_fetch_row_t fetch_row;

    //! 跳过数据
    compression_skip_t skip;

    //===--- 追加阶段（可选） ---===//

    //! 初始化段
    compression_init_segment_t init_segment;

    //! 初始化追加状态
    compression_init_append_t init_append;

    //! 追加数据
    compression_append_t append;

    //! 完成追加
    compression_finalize_append_t finalize_append;

    //! 回滚追加
    compression_revert_append_t revert_append;

    //===--- 序列化（可选） ---===//

    //! 序列化段状态
    compression_serialize_state_t serialize_state;

    //! 反序列化段状态
    compression_deserialize_state_t deserialize_state;

    //! 遍历块 ID
    compression_visit_block_ids_t visit_block_ids;
};
```

### 4.2.2 压缩选择流程

压缩选择是一个三阶段过程：

```
┌─────────────────────────────────────────────────────────────────────┐
│                     压缩选择流程                                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  阶段 1: 初始化分析                                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ for each compression_type:                                    │  │
│  │     if type.supports(physical_type):                          │  │
│  │         state[type] = type.init_analyze()                     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                            │                                        │
│                            ▼                                        │
│  阶段 2: 逐向量分析                                                  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ for each vector in row_group:                                 │  │
│  │     for each active compression_type:                         │  │
│  │         if !type.analyze(state[type], vector):                │  │
│  │             state[type] = INVALID  // 此算法不适用             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                            │                                        │
│                            ▼                                        │
│  阶段 3: 选择最佳压缩                                               │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ best_score = MAX                                              │  │
│  │ for each active compression_type:                             │  │
│  │     score = type.final_analyze(state[type])                   │  │
│  │     if score < best_score:                                    │  │
│  │         best_score = score                                    │  │
│  │         best_type = type                                      │  │
│  │ return best_type                                              │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2.3 CompressionFunctionSet 管理

```cpp
struct CompressionFunctionSet {
    static constexpr idx_t COMPRESSION_TYPE_COUNT = 15;
    static constexpr idx_t PHYSICAL_TYPE_COUNT = 19;

private:
    mutex lock;
    atomic<bool> is_disabled[COMPRESSION_TYPE_COUNT];  // 禁用的压缩方法
    atomic<bool> is_loaded[PHYSICAL_TYPE_COUNT];       // 已加载的类型
    vector<vector<CompressionFunction>> functions;     // 压缩函数表

public:
    // 获取指定物理类型的所有可用压缩函数
    vector<reference<CompressionFunction>> GetCompressionFunctions(PhysicalType type);

    // 获取特定压缩类型的函数
    optional_ptr<CompressionFunction> GetCompressionFunction(
        CompressionType type, PhysicalType physical_type);
};
```

内置压缩方法注册：
```cpp
static const DefaultCompressionMethod internal_compression_methods[] = {
    {CompressionType::COMPRESSION_CONSTANT, ConstantFun::GetFunction, ConstantFun::TypeIsSupported},
    {CompressionType::COMPRESSION_UNCOMPRESSED, UncompressedFun::GetFunction, UncompressedFun::TypeIsSupported},
    {CompressionType::COMPRESSION_RLE, RLEFun::GetFunction, RLEFun::TypeIsSupported},
    {CompressionType::COMPRESSION_BITPACKING, BitpackingFun::GetFunction, BitpackingFun::TypeIsSupported},
    {CompressionType::COMPRESSION_DICTIONARY, DictionaryCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_CHIMP, ChimpCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_PATAS, PatasCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_ALP, AlpCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_ALPRD, AlpRDCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_FSST, FSSTFun::GetFunction, ...},
    {CompressionType::COMPRESSION_ZSTD, ZSTDFun::GetFunction, ...},
    {CompressionType::COMPRESSION_ROARING, RoaringCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_EMPTY, EmptyValidityCompressionFun::GetFunction, ...},
    {CompressionType::COMPRESSION_DICT_FSST, DictFSSTCompressionFun::GetFunction, ...},
};
```

---

## 4.3 常量压缩 (Constant Compression)

### 4.3.1 原理

常量压缩是最高效的压缩方式，适用于整个段中所有值相同的情况：

```
原始数据: [42, 42, 42, 42, ..., 42]  (N 个值)
压缩后:   value = 42                  (1 个值 + 统计信息)

压缩率: N : 1
```

### 4.3.2 实现

常量压缩不需要存储实际数据，只需要存储统计信息中的最小值：

```cpp
template <class T>
void ConstantScanFunction(ColumnSegment &segment, ColumnScanState &state,
                           idx_t scan_count, Vector &result) {
    auto &nstats = segment.stats.statistics;

    // 从统计信息获取常量值
    auto data = FlatVector::GetData<T>(result);
    data[0] = NumericStats::GetMin<T>(nstats);

    // 设置为常量向量
    result.SetVectorType(VectorType::CONSTANT_VECTOR);
}
```

### 4.3.3 适用场景

- 默认值填充的列
- 分区列（同一分区内值相同）
- 大量重复数据

---

## 4.4 游程编码 (RLE - Run Length Encoding)

### 4.4.1 原理

RLE 将连续重复的值编码为 `(value, count)` 对：

```
原始数据: [1, 1, 1, 2, 2, 3, 3, 3, 3, 3]
压缩后:   [(1, 3), (2, 2), (3, 5)]

存储格式:
+------------------+------------------+
|   Values Array   |   Counts Array   |
| [1, 2, 3, ...]   | [3, 2, 5, ...]   |
+------------------+------------------+
```

### 4.4.2 数据结构

```cpp
using rle_count_t = uint16_t;  // 最大计数 65535

struct RLEConstants {
    static constexpr const idx_t RLE_HEADER_SIZE = sizeof(uint64_t);
};

// 段布局
// +----------+----------------+----------------+
// | Header   | Values[N]      | Counts[N]      |
// | 8 bytes  | N * sizeof(T)  | N * 2 bytes    |
// +----------+----------------+----------------+
```

### 4.4.3 分析阶段

```cpp
template <class T>
struct RLEState {
    idx_t seen_count;        // RLE 对的数量
    T last_value;            // 上一个值
    rle_count_t last_seen_count;  // 当前连续计数

    template <class OP>
    void Update(const T *data, ValidityMask &validity, idx_t idx) {
        if (validity.RowIsValid(idx)) {
            if (last_value == data[idx]) {
                // 相同值，增加计数
                last_seen_count++;
            } else {
                // 不同值，刷新之前的 RLE 对
                if (last_seen_count > 0) {
                    Flush<OP>();
                    seen_count++;
                }
                last_value = data[idx];
                last_seen_count = 1;
            }
        } else {
            // NULL 值也增加计数
            last_seen_count++;
        }

        // 计数达到上限时刷新
        if (last_seen_count == NumericLimits<rle_count_t>::Maximum()) {
            Flush<OP>();
            last_seen_count = 0;
            seen_count++;
        }
    }
};

template <class T>
idx_t RLEFinalAnalyze(AnalyzeState &state) {
    auto &rle_state = state.Cast<RLEAnalyzeState<T>>();
    // 预估压缩大小 = (值大小 + 计数大小) × RLE对数量
    return (sizeof(rle_count_t) + sizeof(T)) * rle_state.state.seen_count;
}
```

### 4.4.4 适用场景

- 排序列（ORDER BY 后的数据）
- 低基数列（如状态码、分类）
- 有规律重复的数据

---

## 4.5 位打包 (Bitpacking)

### 4.5.1 原理

位打包根据实际值的范围，使用最少的位数存储每个值：

```
原始数据: [0, 1, 2, 3, 4, 5, 6, 7]  (64-bit integers)
值范围: 0-7 只需要 3 bits

普通存储: 8 × 64 bits = 512 bits
位打包后: 8 × 3 bits = 24 bits

压缩率: ~21x
```

### 4.5.2 位打包模式

```cpp
enum class BitpackingMode : uint8_t {
    AUTO,           // 自动选择最佳模式
    CONSTANT,       // 常量模式（所有值相同）
    CONSTANT_DELTA, // 常量增量（如序列）
    DELTA_FOR,      // 增量 + Frame of Reference
    FOR             // Frame of Reference
};
```

### 4.5.3 Frame of Reference (FOR)

FOR 通过减去最小值来减少值的范围：

```
原始数据: [1000, 1001, 1002, 1003]
最小值 (FOR): 1000
调整后: [0, 1, 2, 3]  只需要 2 bits

存储: FOR(1000) + 位打包后的 [0, 1, 2, 3]
```

### 4.5.4 Delta 编码

Delta 编码存储相邻值的差：

```
原始数据: [100, 103, 105, 110, 112]
Delta:    [100, 3, 2, 5, 2]

结合 FOR:
Delta 范围: 2-5
Delta FOR: 2
调整后: [0, 1, 0, 3, 0]  只需要 2 bits
```

### 4.5.5 实现

```cpp
template <class T, class T_S = typename MakeSigned<T>::type>
struct BitpackingState {
    T compression_buffer[BITPACKING_METADATA_GROUP_SIZE + 1];
    T_S delta_buffer[BITPACKING_METADATA_GROUP_SIZE];
    bool compression_buffer_validity[BITPACKING_METADATA_GROUP_SIZE];

    T minimum, maximum, min_max_diff;
    T_S minimum_delta, maximum_delta, min_max_delta_diff;

    bool can_do_delta;
    bool can_do_for;

    template <class OP>
    bool Flush() {
        if (compression_buffer_idx == 0) {
            return true;
        }

        // 1. 检查常量模式
        if (all_invalid || maximum == minimum) {
            OP::WriteConstant(maximum, compression_buffer_idx, data_ptr, all_invalid);
            return true;
        }

        // 2. 计算 FOR 统计
        CalculateFORStats();  // min_max_diff

        // 3. 计算 Delta 统计
        CalculateDeltaStats();  // min_max_delta_diff

        // 4. 选择最佳模式
        if (can_do_delta) {
            // 检查常量增量
            if (minimum_delta == maximum_delta) {
                OP::WriteConstantDelta(minimum_delta, ...);
                return true;
            }

            // 比较 Delta-FOR vs FOR
            auto delta_width = BitpackingPrimitives::MinimumBitWidth(min_max_delta_diff);
            auto regular_width = BitpackingPrimitives::MinimumBitWidth(min_max_diff);

            if (delta_width < regular_width) {
                // 使用 Delta-FOR
                SubtractFrameOfReference(delta_buffer, minimum_delta);
                OP::WriteDeltaFor(compression_buffer, ..., delta_width, ...);
                return true;
            }
        }

        if (can_do_for) {
            // 使用 FOR
            auto width = BitpackingPrimitives::MinimumBitWidth(min_max_diff);
            SubtractFrameOfReference(compression_buffer, minimum);
            OP::WriteFor(compression_buffer, ..., width, minimum, ...);
            return true;
        }

        return false;  // 无法压缩
    }
};
```

### 4.5.6 位宽计算

```cpp
class BitpackingPrimitives {
public:
    static constexpr idx_t BITPACKING_ALGORITHM_GROUP_SIZE = 32;

    static bitpacking_width_t MinimumBitWidth(T value) {
        if (value == 0) {
            return 0;
        }
        return sizeof(T) * 8 - CountZeros<T>::Leading(value);
    }
};
```

---

## 4.6 字典压缩 (Dictionary Compression)

### 4.6.1 原理

字典压缩将重复字符串映射到整数 ID：

```
原始数据: ["apple", "banana", "apple", "cherry", "banana", "apple"]

字典:
  0 -> "apple"
  1 -> "banana"
  2 -> "cherry"

压缩后: [0, 1, 0, 2, 1, 0]  (使用位打包存储)
```

### 4.6.2 段布局

```
+------------------------------------------------------+
|                  Header                              |
|   +----------------------------------------------+   |
|   |   dictionary_compression_header_t  header    |   |
|   +----------------------------------------------+   |
+------------------------------------------------------+
|             Selection Buffer                         |
|   +------------------------------------+             |
|   |   uint16_t index_buffer_idx[]      |             |
|   +------------------------------------+             |
|      tuple index -> dictionary index                 |
+------------------------------------------------------+
|               Index Buffer                           |
|   +------------------------------------+             |
|   |   uint16_t  dictionary_offset[]    |             |
|   +------------------------------------+             |
|  dictionary_index -> offset in dictionary            |
+------------------------------------------------------+
|                Dictionary                            |
|   +------------------------------------+             |
|   |   uint8_t *raw_string_data         |             |
|   +------------------------------------+             |
|      raw string data without lengths                 |
+------------------------------------------------------+
```

### 4.6.3 分析阶段

```cpp
bool DictionaryCompressionStorage::StringAnalyze(AnalyzeState &state_p,
                                                  Vector &input, idx_t count) {
    auto &state = state_p.Cast<DictionaryCompressionAnalyzeState>();
    return state.analyze_state->UpdateState(input, count);
}

idx_t DictionaryCompressionStorage::StringFinalAnalyze(AnalyzeState &state_p) {
    auto &state = state_p.Cast<DictionaryCompressionAnalyzeState>();

    // 计算所需位宽
    auto width = BitpackingPrimitives::MinimumBitWidth(state.current_unique_count + 1);

    // 计算所需空间
    auto req_space = DictionaryCompression::RequiredSpace(
        state.current_tuple_count,
        state.current_unique_count,
        state.current_dict_size,
        width);

    // 返回压缩后的预估大小（乘以最小压缩比）
    return LossyNumericCast<idx_t>(
        DictionaryCompression::MINIMUM_COMPRESSION_RATIO * float(total_space));
}
```

### 4.6.4 扫描优化

字典压缩支持返回字典向量，避免字符串复制：

```cpp
template <bool ALLOW_DICT_VECTORS>
void StringScanPartial(ColumnSegment &segment, ColumnScanState &state,
                       idx_t scan_count, Vector &result, idx_t result_offset) {
    auto &scan_state = state.scan_state->Cast<CompressedStringScanState>();

    if (!ALLOW_DICT_VECTORS || scan_count != STANDARD_VECTOR_SIZE) {
        // 常规扫描：复制字符串
        scan_state.ScanToFlatVector(result, result_offset, start, scan_count);
    } else {
        // 优化扫描：返回字典向量
        scan_state.ScanToDictionaryVector(segment, result, result_offset,
                                           start, scan_count);
    }
}
```

---

## 4.7 FSST (Fast Static Symbol Table)

### 4.7.1 原理

FSST 是一种字符串压缩算法，通过学习常见字节序列构建符号表：

```
训练数据: ["http://example.com", "http://test.com", ...]

符号表:
  Symbol 0: "http://"
  Symbol 1: ".com"
  Symbol 2: "example"
  ...

压缩: "http://example.com" -> [0, 2, 1]
```

### 4.7.2 实现特点

```cpp
struct FSSTStorage {
    static constexpr double MINIMUM_COMPRESSION_RATIO = 1.2;
    static constexpr double ANALYSIS_SAMPLE_SIZE = 0.25;  // 采样 25% 进行训练

    struct FSSTAnalyzeState : public AnalyzeState {
        duckdb_fsst_encoder_t *fsst_encoder = nullptr;
        idx_t count;
        StringHeap fsst_string_heap;
        vector<string_t> fsst_strings;
        size_t fsst_string_total_size;
        RandomEngine random_engine;
    };
};
```

分析阶段使用随机采样训练符号表：
```cpp
bool FSSTStorage::StringAnalyze(AnalyzeState &state_p, Vector &input, idx_t count) {
    auto &state = state_p.Cast<FSSTAnalyzeState>();

    // 随机采样决定是否包含此向量
    bool sample_selected = !state.have_valid_row ||
                           state.random_engine.NextRandom() < ANALYSIS_SAMPLE_SIZE;

    for (idx_t i = 0; i < count; i++) {
        if (!vdata.validity.RowIsValid(idx)) continue;

        // 检查字符串大小限制
        auto string_size = data[idx].GetSize();
        if (string_size >= StringUncompressed::GetStringBlockLimit(...)) {
            return false;  // 字符串太大，放弃 FSST
        }

        if (sample_selected && string_size > 0) {
            // 收集训练样本
            state.fsst_strings.push_back(data[idx]);
            state.fsst_string_total_size += string_size;
        }
    }
    return true;
}
```

### 4.7.3 段头结构

```cpp
typedef struct {
    uint32_t dict_size;                // 字典大小
    uint32_t dict_end;                 // 字典结束位置
    uint32_t bitpacking_width;         // 位打包宽度
    uint32_t fsst_symbol_table_offset; // 符号表偏移
} fsst_compression_header_t;
```

---

## 4.8 Dictionary + FSST 组合

### 4.8.1 原理

Dict+FSST 结合了两种压缩方法的优势：
1. 使用字典消除重复字符串
2. 使用 FSST 压缩字典本身

```
原始数据: ["http://api.example.com/users",
           "http://api.example.com/posts",
           "http://api.example.com/users",
           ...]

步骤1 - 字典化:
  字典: ["http://api.example.com/users", "http://api.example.com/posts"]
  索引: [0, 1, 0, ...]

步骤2 - FSST 压缩字典:
  符号表: {"http://api.example.com/", ...}
  压缩字典: [符号序列...]
```

### 4.8.2 适用场景

- 高重复率的长字符串
- URL、文件路径等有共同前缀的数据
- 日志消息

---

## 4.9 浮点压缩算法

### 4.9.1 ALP (Adaptive Lossless floating Point)

ALP 通过将浮点数转换为整数来实现压缩：

```
原理:
1. 找到合适的缩放因子 (10^e)
2. 将浮点数乘以缩放因子转换为整数
3. 使用整数压缩算法（如位打包）

示例:
原始: [3.14, 3.15, 3.16]
缩放因子: 100
整数化: [314, 315, 316]
压缩: 位打包
```

```cpp
template <>
CompressionFunction GetAlpRDFunction<double>(PhysicalType data_type) {
    return CompressionFunction(
        CompressionType::COMPRESSION_ALPRD, data_type,
        AlpRDInitAnalyze<double>,
        AlpRDAnalyze<double>,
        AlpRDFinalAnalyze<double>,
        AlpRDInitCompression<double>,
        AlpRDCompress<double>,
        AlpRDFinalizeCompress<double>,
        AlpRDInitScan<double>,
        AlpRDScan<double>,
        AlpRDScanPartial<double>,
        AlpRDFetchRow<double>,
        AlpRDSkip<double>);
}
```

### 4.9.2 Chimp 和 Patas

这些是基于 XOR 的浮点压缩算法：

**Chimp**：
- 利用连续浮点值的位模式相似性
- 存储与前一个值的 XOR 结果
- 对 XOR 结果进行位打包

**Patas**：
- Chimp 的改进版本
- 更好的处理不规则数据
- 支持更多特殊情况

适用场景：
- 时序数据（温度、股价等）
- 传感器数据
- 科学计算数据

---

## 4.10 Roaring Bitmap 压缩

### 4.10.1 原理

Roaring Bitmap 用于压缩位图数据（布尔值、有效性掩码）：

```
将 32 位整数空间分成 65536 个容器，每个容器 65536 个位

容器类型:
1. Array Container: 稀疏数据，存储实际位置列表
2. Bitmap Container: 密集数据，存储完整位图
3. Run Container: 连续范围，存储 (start, length) 对

自动选择最优容器类型
```

### 4.10.2 适用场景

- 有效性掩码（NULL 值标记）
- 布尔列
- 稀疏位图数据

---

## 4.11 ZSTD 通用压缩

### 4.11.1 原理

ZSTD 是 Facebook 开发的通用压缩算法，作为后备方案：

```cpp
// ZSTD 压缩
static void Compress(CompressionState &state_p, Vector &scan_vector, idx_t count) {
    auto &state = state_p.Cast<ZSTDCompressionState>();

    // 序列化向量数据
    // ...

    // 使用 ZSTD 压缩
    auto compressed_size = ZSTD_compress(
        dst, dst_capacity,
        src, src_size,
        compression_level);
}
```

### 4.11.2 特点

- 压缩率高
- 压缩/解压速度适中
- 适用于无法使用专用压缩的数据

---

## 4.12 空值压缩 (Empty Validity)

### 4.12.1 原理

当有效性掩码全为 true（无 NULL 值）时，可以完全省略：

```cpp
void EmptyValidityCompressionFun::GetFunction(...) {
    // 空实现 - 不需要存储任何数据
}

void ConstantFillFunctionValidity(ColumnSegment &segment, Vector &result,
                                   idx_t start_idx, idx_t count) {
    auto &stats = segment.stats.statistics;
    if (stats.CanHaveNull()) {
        // 只有可能有 NULL 时才需要设置
        auto &mask = FlatVector::Validity(result);
        for (idx_t i = 0; i < count; i++) {
            mask.SetInvalid(start_idx + i);
        }
    }
    // 否则：所有值都有效，无需操作
}
```

---

## 4.13 压缩选择策略

### 4.13.1 选择算法

```cpp
CompressionType SelectCompression(ColumnData &col_data, const LogicalType &type) {
    auto &compression_set = DBConfig::GetConfig(col_data.GetDatabase())
                                .GetCompressionFunctions();

    // 1. 获取该类型支持的所有压缩函数
    auto functions = compression_set.GetCompressionFunctions(type.InternalType());

    // 2. 初始化所有候选分析状态
    vector<unique_ptr<AnalyzeState>> states;
    for (auto &function : functions) {
        auto state = function.get().init_analyze(col_data, type.InternalType());
        if (state) {
            states.push_back(std::move(state));
        }
    }

    // 3. 遍历数据进行分析
    for (auto &vector : data_vectors) {
        for (idx_t i = states.size(); i > 0; i--) {
            auto &function = functions[i - 1];
            if (!function.get().analyze(*states[i - 1], vector, count)) {
                // 此压缩方法不适用
                states.erase(states.begin() + i - 1);
                functions.erase(functions.begin() + i - 1);
            }
        }
    }

    // 4. 选择最佳压缩
    idx_t best_score = DConstants::INVALID_INDEX;
    CompressionType best_type = CompressionType::COMPRESSION_UNCOMPRESSED;

    for (idx_t i = 0; i < states.size(); i++) {
        auto score = functions[i].get().final_analyze(*states[i]);
        if (score < best_score) {
            best_score = score;
            best_type = functions[i].get().type;
        }
    }

    return best_type;
}
```

### 4.13.2 优先级和禁用

用户可以禁用特定压缩方法：
```sql
-- 禁用 RLE 压缩
SET disabled_compression_methods = 'rle';

-- 禁用多个
SET disabled_compression_methods = 'rle,bitpacking';
```

```cpp
void CompressionFunctionSet::SetDisabledCompressionMethods(
    const vector<CompressionType> &methods) {
    ResetDisabledMethods();
    for (auto &method : methods) {
        auto idx = GetCompressionIndex(method);
        is_disabled[idx] = true;
    }
}
```

---

## 4.14 小结

DuckDB 的压缩系统具有以下特点：

### 核心设计原则

1. **自动选择**
   - 分析阶段评估所有候选压缩
   - 基于预估压缩大小选择最佳算法
   - 无需用户手动配置

2. **类型特化**
   - 整数：Bitpacking、RLE
   - 浮点：ALP、Chimp、Patas
   - 字符串：Dictionary、FSST、Dict+FSST
   - 布尔：Roaring Bitmap

3. **分段独立**
   - 每个 ColumnSegment 独立选择压缩
   - 同一列可能使用不同压缩方法
   - 适应数据分布变化

4. **扫描优化**
   - 字典向量避免字符串复制
   - 常量向量快速返回
   - 支持带过滤器的扫描

### 压缩算法对比

| 算法 | 适用类型 | 压缩率 | 解压速度 | 适用场景 |
|------|----------|--------|----------|----------|
| Constant | 所有 | 极高 | 极快 | 单一值 |
| RLE | 整数 | 高 | 快 | 排序/低基数 |
| Bitpacking | 整数 | 中-高 | 极快 | 范围有限 |
| Dictionary | 字符串 | 高 | 快 | 重复字符串 |
| FSST | 字符串 | 中-高 | 中 | 有规律字符串 |
| ALP/ALPRD | 浮点 | 中-高 | 快 | 可整数化浮点 |
| Chimp/Patas | 浮点 | 中 | 快 | 时序数据 |
| Roaring | 布尔 | 高 | 快 | 稀疏位图 |
| ZSTD | 所有 | 高 | 中 | 通用后备 |

### 关键源文件

| 组件 | 头文件 | 实现文件 |
|------|--------|----------|
| CompressionFunction | `function/compression_function.hpp` | - |
| CompressionFunctionSet | `function/compression_function.hpp` | `function/compression_config.cpp` |
| RLE | - | `storage/compression/rle.cpp` |
| Bitpacking | `storage/compression/bitpacking.hpp` | `storage/compression/bitpacking.cpp` |
| Dictionary | `storage/compression/dictionary/*.hpp` | `storage/compression/dictionary_compression.cpp` |
| FSST | - | `storage/compression/fsst.cpp` |
| ALP/ALPRD | `storage/compression/alp*/*.hpp` | `storage/compression/alp*.cpp` |
| Chimp | `storage/compression/chimp/*.hpp` | `storage/compression/chimp/*.cpp` |
| Patas | `storage/compression/patas/*.hpp` | `storage/compression/patas.cpp` |
| ZSTD | - | `storage/compression/zstd.cpp` |
| Roaring | - | `storage/compression/roaring.cpp` |

---

下一章，我们将深入探讨 DuckDB 的统计系统，包括段统计、表统计和基数估计。
