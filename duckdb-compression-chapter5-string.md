# DuckDB压缩系统深度解析（五）：字符串压缩算法

## 引言

字符串是数据库中最常见也最具挑战性的数据类型之一。字符串长度可变、内容多样，传统的数值压缩技术难以直接应用。DuckDB实现了三种专门针对字符串的压缩算法：Dictionary（字典压缩）、FSST（快速静态符号表）和Dict-FSST（混合压缩）。本章将深入分析这些算法的原理和实现。

## 1. Dictionary（字典压缩）

### 1.1 算法原理

字典压缩的核心思想是为所有唯一字符串建立字典，然后用小的整数索引替代原始字符串：

```
原始数据: ["apple", "banana", "apple", "cherry", "banana", "apple"]

字典:
  0 -> "apple"
  1 -> "banana"
  2 -> "cherry"

压缩后: [0, 1, 0, 2, 1, 0]

空间节省:
  原始: 5+6+5+6+6+5 = 33字节
  压缩: 字典(17字节) + 索引(6个×2位) ≈ 19字节
```

### 1.2 数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                      Dictionary压缩段布局                           │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (20 bytes)                                            │  │
│  │ ┌─────────────────────────────────────────────────────────┐ │  │
│  │ │ dict_size (4 bytes): 字典当前大小                        │ │  │
│  │ │ dict_end (4 bytes): 字典结束位置                         │ │  │
│  │ │ index_buffer_offset (4 bytes): 索引缓冲区偏移            │ │  │
│  │ │ index_buffer_count (4 bytes): 索引缓冲区条目数           │ │  │
│  │ │ bitpacking_width (4 bytes): 位压缩宽度                   │ │  │
│  │ └─────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Selection Buffer (位压缩)                                    │  │
│  │ 元组索引 -> 索引缓冲区索引的映射                              │  │
│  │ 每个值使用 bitpacking_width 位                               │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Index Buffer (uint32_t数组)                                  │  │
│  │ 字符串索引 -> 字典中的偏移量                                  │  │
│  │ 字符串长度 = index_buffer[i] - index_buffer[i-1]             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Dictionary (原始字节)                                        │  │
│  │ 从块末尾向前增长存储                                          │  │
│  │ 不包含终止符，通过索引缓冲区计算长度                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 1.3 头部结构

```cpp
// src/include/duckdb/storage/compression/dictionary/common.hpp

typedef struct {
    uint32_t dict_size;           // 字典当前大小
    uint32_t dict_end;            // 字典结束位置
    uint32_t index_buffer_offset; // 索引缓冲区起始偏移
    uint32_t index_buffer_count;  // 唯一字符串数量
    uint32_t bitpacking_width;    // 选择缓冲区的位宽
} dictionary_compression_header_t;
```

### 1.4 压缩流程

```cpp
void DictionaryCompressionCompressState::AddNewString(string_t str) {
    // 1. 更新统计信息
    UncompressedStringStorage::UpdateStringStats(current_segment->stats, str);

    // 2. 在字典末尾存储字符串（从块末尾向前增长）
    current_dictionary.size += str.GetSize();
    auto dict_pos = current_end_ptr - current_dictionary.size;
    memcpy(dict_pos, str.GetData(), str.GetSize());

    // 3. 添加偏移量到索引缓冲区
    index_buffer.push_back(current_dictionary.size);

    // 4. 添加映射到选择缓冲区
    selection_buffer.push_back(index_buffer.size() - 1);

    // 5. 更新字符串映射用于重复检测
    current_string_map.Insert(str);

    // 6. 更新位压缩宽度
    current_width = BitpackingPrimitives::MinimumBitWidth(index_buffer.size() - 1);
    current_segment->count++;
}
```

### 1.5 空间计算

```cpp
idx_t DictionaryCompression::RequiredSpace(
    idx_t current_count,      // 元组数量
    idx_t index_count,        // 唯一字符串数量
    idx_t dict_size,          // 字典字节数
    bitpacking_width_t packing_width  // 位宽
) {
    idx_t base_space = DICTIONARY_HEADER_SIZE + dict_size;
    idx_t selection_buffer_space =
        BitpackingPrimitives::GetRequiredSize(current_count, packing_width);
    idx_t index_buffer_space = index_count * sizeof(uint32_t);

    return base_space + index_buffer_space + selection_buffer_space;
}

bool DictionaryCompression::HasEnoughSpace(
    idx_t current_count,
    idx_t index_count,
    idx_t dict_size,
    bitpacking_width_t packing_width,
    const idx_t block_size
) {
    idx_t required = RequiredSpace(current_count, index_count, dict_size, packing_width);
    return required <= block_size;
}
```

### 1.6 解压流程

```cpp
uint16_t CompressedStringScanState::GetStringLength(sel_t index) {
    if (index == 0) {
        return 0;  // NULL值
    } else {
        // 长度 = 相邻偏移量之差
        return index_buffer_ptr[index] - index_buffer_ptr[index - 1];
    }
}

string_t CompressedStringScanState::FetchStringFromDict(
    int32_t dict_offset, uint16_t string_len) {

    if (dict_offset == 0) {
        return string_t(nullptr, 0);  // NULL
    }

    // 从块末尾读取
    auto dict_end = baseptr + dict.end;
    auto dict_pos = dict_end - dict_offset;
    auto str_ptr = char_ptr_cast(dict_pos);
    return string_t(str_ptr, string_len);
}

void CompressedStringScanState::ScanToFlatVector(
    Vector &result, idx_t result_offset,
    idx_t start, idx_t scan_count) {

    // 对齐到位压缩算法组大小
    idx_t start_offset = start % BITPACKING_ALGORITHM_GROUP_SIZE;
    idx_t decompress_count = RoundUpToAlgorithmGroupSize(scan_count + start_offset);

    // 解压选择缓冲区
    data_ptr_t src = &base_data[((start - start_offset) * current_width) / 8];
    BitpackingPrimitives::UnPackBuffer<sel_t>(
        sel_vec->data(), src, decompress_count, current_width);

    // 从字典获取字符串
    for (idx_t i = 0; i < scan_count; i++) {
        auto string_number = sel_vec->get_index(i + start_offset);
        auto dict_offset = index_buffer_ptr[string_number];
        auto str_len = GetStringLength(string_number);
        result_data[result_offset + i] = FetchStringFromDict(dict_offset, str_len);
    }
}
```

### 1.7 最小压缩比

```cpp
static constexpr float MINIMUM_COMPRESSION_RATIO = 1.2F;
```

只有当未压缩大小 ≥ 压缩大小 × 1.2 时，才认为压缩有效。

## 2. FSST（快速静态符号表）

### 2.1 算法原理

FSST将可变长度的字节序列（1-8字节）映射到单字节编码：

```
核心思想：
┌─────────────────────────────────────────────────────────────────────┐
│ 符号表: 255个符号 (编码0-254)，编码255用作转义字节                    │
│                                                                     │
│ 示例：                                                               │
│   原始: "the quick brown fox"                                       │
│   符号表: 'th'->1, 'e '->2, 'qu'->3, 'ick'->4, ...                  │
│   编码: [1, 2, 3, 4, ' ', 'b', 'r', 'o', 'w', 'n', 2, ...]          │
│                                                                     │
│ 转义机制：                                                           │
│   遇到不在符号表中的字节时，使用 [255, 原始字节] 表示                  │
│                                                                     │
│ 关键特性：                                                           │
│   - 相同字符串 → 相同压缩结果（支持索引兼容）                         │
│   - 支持随机访问解压（无需解压整个块）                               │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 符号表结构

```cpp
// third_party/fsst/fsst.h

typedef struct {
    unsigned long long version;       // 版本标识
    unsigned char zeroTerminated;     // 零终止符处理标志
    unsigned char len[255];           // 每个符号的长度 (1-8字节)
    unsigned long long symbol[255];   // 符号值 (小端序)
} duckdb_fsst_decoder_t;

// 序列化最大大小
#define FSST_MAXHEADER (8+1+8+2048+1)  // ~2057字节，通常~800-1200字节
```

### 2.3 头部结构

```cpp
// src/storage/compression/fsst.cpp

typedef struct {
    uint32_t dict_size;                 // 压缩字典大小
    uint32_t dict_end;                  // 结束位置
    uint32_t bitpacking_width;          // 长度位压缩宽度
    uint32_t fsst_symbol_table_offset;  // FSST符号表偏移
} fsst_compression_header_t;
```

### 2.4 数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                        FSST压缩段布局                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (fsst_compression_header_t)                           │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Bitpacked Offsets                                            │  │
│  │ Delta编码的字符串长度（位压缩）                               │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ FSST Symbol Table (~800-1200 bytes)                          │  │
│  │ 序列化的 duckdb_fsst_decoder_t                               │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ FSST Compressed Data                                         │  │
│  │ 可变长度的压缩字符串                                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 2.5 采样策略

FSST使用25%采样来构建符号表：

```cpp
bool FSSTStorage::StringAnalyze(AnalyzeState &state_p, Vector &input, idx_t count) {
    auto &state = state_p.Cast<FSSTAnalyzeState>();

    // 采样策略: 25%的数据
    static constexpr double ANALYSIS_SAMPLE_SIZE = 0.25;
    bool sample_selected = !state.have_valid_row ||
                          state.random_engine.NextRandom() < ANALYSIS_SAMPLE_SIZE;

    for (idx_t i = 0; i < count; i++) {
        if (!valid[idx]) continue;

        // 检查字符串大小限制
        if (string_size >= StringUncompressed::GetStringBlockLimit(block_size)) {
            return false;
        }

        if (!sample_selected) continue;

        if (string_size > 0) {
            state.have_valid_row = true;
            state.fsst_strings.push_back(string);
            state.fsst_string_total_size += string_size;
        } else {
            state.empty_strings++;
        }
    }
    return true;
}
```

### 2.6 符号表构建

```cpp
idx_t FSSTStorage::StringFinalAnalyze(AnalyzeState &state_p) {
    auto &state = state_p.Cast<FSSTAnalyzeState>();

    // 从采样字符串创建编码器
    state.fsst_encoder = duckdb_fsst_create(
        string_count,
        &fsst_string_sizes[0],
        &fsst_string_ptrs[0],
        0  // 非零终止
    );

    // 测试压缩以估算大小
    auto res = duckdb_fsst_compress(
        state.fsst_encoder,
        string_count,
        &fsst_string_sizes[0],
        &fsst_string_ptrs[0],
        output_buffer_size,
        compressed_buffer.get(),
        &compressed_sizes[0],
        &compressed_ptrs[0]
    );

    // 计算估算大小
    size_t compressed_dict_size = 0;
    size_t max_compressed_string_length = 0;
    for (auto &size : compressed_sizes) {
        compressed_dict_size += size;
        max_compressed_string_length = MaxValue(max_compressed_string_length, size);
    }

    auto minimum_width = BitpackingPrimitives::MinimumBitWidth(max_compressed_string_length);
    auto bitpacked_offsets_size =
        BitpackingPrimitives::GetRequiredSize(string_count + empty_strings, minimum_width);

    // 考虑采样率外推
    auto estimated_base_size = double(bitpacked_offsets_size + compressed_dict_size) *
                              (1 / ANALYSIS_SAMPLE_SIZE);
    auto num_blocks = estimated_base_size /
                     double(block_size - sizeof(duckdb_fsst_decoder_t));
    auto symtable_size = num_blocks * sizeof(duckdb_fsst_decoder_t);
    auto estimated_size = estimated_base_size + symtable_size;

    // 应用最小压缩比
    static constexpr double MINIMUM_COMPRESSION_RATIO = 1.2;
    return LossyNumericCast<idx_t>(estimated_size * MINIMUM_COMPRESSION_RATIO);
}
```

### 2.7 解压API

```cpp
// 快速内联解压
inline size_t duckdb_fsst_decompress(
    duckdb_fsst_decoder_t *decoder,
    size_t lenIn,              // 压缩长度
    const unsigned char *strIn, // 压缩数据
    size_t size,               // 输出缓冲区大小
    unsigned char *output      // 输出缓冲区
)
```

## 3. Dict-FSST（混合压缩）

### 3.1 算法原理

Dict-FSST结合了Dictionary和FSST的优点，根据数据特征自适应选择压缩模式：

```
设计动机：
┌─────────────────────────────────────────────────────────────────────┐
│ Dictionary: 适合低基数数据（大量重复）                               │
│ FSST: 适合高基数数据（唯一值多）                                     │
│                                                                     │
│ Dict-FSST: 自适应选择                                               │
│   - 字典小于4KB: 纯Dictionary模式                                   │
│   - 字典大于4KB: 对字典应用FSST编码                                  │
│   - 全唯一值: 纯FSST模式（省略选择缓冲区）                           │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 三种模式

```cpp
// src/include/duckdb/storage/compression/dict_fsst/common.hpp

enum class DictFSSTMode : uint8_t {
    DICTIONARY = 0,    // 纯字典（低基数）
    DICT_FSST = 1,     // 字典 + FSST编码
    FSST_ONLY = 2,     // 纯FSST（全唯一值）
    COUNT
};
```

模式选择流程：

```
数据分析
    │
    ├─> 字典 < 4KB? ─yes─> DICTIONARY模式
    │       保持原样，不使用FSST
    │
    ├─> 字典 >= 4KB?
    │       │
    │       ├─> 全唯一值? ─yes─> FSST_ONLY模式
    │       │       省略选择缓冲区，直接FSST压缩
    │       │
    │       └─> 有重复值? ─yes─> DICT_FSST模式
    │               对字典进行FSST编码，保留选择缓冲区
    │
    └─> 压缩不划算? ─> 回退到Uncompressed
```

### 3.3 头部结构

```cpp
typedef struct {
    uint32_t dict_size;               // 字典大小
    uint32_t dict_count;              // 唯一字符串数量
    DictFSSTMode mode;                // 使用的模式
    uint8_t string_lengths_width;     // 字符串长度位宽
    uint8_t dictionary_indices_width; // 选择缓冲区位宽
    uint32_t symbol_table_size;       // FSST符号表大小（0表示未编码）
} dict_fsst_compression_header_t;
```

### 3.4 数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                     Dict-FSST压缩段布局                             │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (dict_fsst_compression_header_t)                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Aligned Dictionary                                           │  │
│  │ 原始或FSST编码的字典字符串                                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ FSST Symbol Table (可选, 仅DICT_FSST/FSST_ONLY模式)          │  │
│  │ ~800-1200字节                                                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ String Lengths (位压缩)                                      │  │
│  │ 每个唯一字符串的长度（或FSST编码后的长度）                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Dictionary Indices (位压缩)                                  │  │
│  │ 元组 -> 字典索引映射                                          │  │
│  │ FSST_ONLY模式时省略                                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 3.5 状态机管理

```cpp
struct DictFSSTCompressionState {
    // 编码决策状态机
    DictionaryAppendState append_state = DictionaryAppendState::REGULAR;

    // 常规阶段：积累字典
    PrimitiveDictionary<string_t> current_string_map;
    vector<uint32_t> string_lengths;
    idx_t dict_count = 0;

    // 转换触发阈值：4KB字典数据
    static constexpr idx_t DICTIONARY_ENCODE_THRESHOLD = 4096;

    // 编码阶段：准备FSST编码
    vector<string_t> dictionary_encoding_buffer;  // 等待编码的字符串
    idx_t to_encode_string_sum = 0;               // 保守大小估算
    unsafe_unique_array<unsigned char> fsst_serialized_symbol_table;
    void *encoder = nullptr;
};
```

### 3.6 编码决策逻辑

```cpp
DictionaryAppendState DictFSSTCompressionState::TryEncode() {
    // 检查字典是否足够大以从FSST获益
    if (dictionary_offset < DICTIONARY_ENCODE_THRESHOLD) {
        return DictionaryAppendState::NOT_ENCODED;  // 保持纯字典模式
    }

    // 确定模式
    DictionaryAppendState new_state;
    if (!analyze->contains_nulls && AllUnique()) {
        // 全唯一值：使用FSST_ONLY（省略选择缓冲区）
        new_state = DictionaryAppendState::ENCODED_ALL_UNIQUE;
    } else {
        // 混合：使用DICT_FSST
        new_state = DictionaryAppendState::ENCODED;
    }

    // 从当前字典字符串创建FSST编码器
    encoder = reinterpret_cast<void *>(
        duckdb_fsst_create(string_count, fsst_string_sizes.data(),
                          fsst_string_ptrs.data(), 0)
    );

    // 测试压缩
    auto res = duckdb_fsst_compress(
        fsst_encoder,
        string_count,
        fsst_string_sizes.data(),
        fsst_string_ptrs.data(),
        dictionary_offset,
        dictionary_start,
        compressed_sizes.data(),
        compressed_ptrs.data()
    );

    // 验证压缩效率
    if (res != string_count) {
        return DictionaryAppendState::NOT_ENCODED;
    }

    // 检查压缩后字典是否确实更小
    idx_t new_size = 0;
    for (idx_t i = 0; i < string_count; i++) {
        new_size += compressed_sizes[i];
    }
    if (new_size + DICTIONARY_ENCODE_THRESHOLD > dictionary_offset) {
        // 压缩无益，回退
        return DictionaryAppendState::NOT_ENCODED;
    }

    // 导出符号表
    symbol_table_size = duckdb_fsst_export(fsst_encoder,
                                           fsst_serialized_symbol_table.get());

    // 验证最终大小适合块
    idx_t required_space = sizeof(dict_fsst_compression_header_t) +
                          new_size +           // 压缩字典
                          symbol_table_size +
                          string_lengths_space +
                          dictionary_indices_space;

    if (required_space > info.GetBlockSize()) {
        return DictionaryAppendState::NOT_ENCODED;
    }

    return new_state;
}
```

### 3.7 字符串添加逻辑

```cpp
bool DictFSSTCompressionState::CompressInternal(
    UnifiedVectorFormat &vector_format,
    const string_t &str,
    bool is_null,
    EncodedInput &encoded_input,
    idx_t i, idx_t count,
    bool fail_on_no_space
) {
    if (is_null) {
        dictionary_indices.push_back(0);  // NULL始终映射到索引0
        tuple_count++;
        return true;
    }

    // 尝试在字典中查找字符串
    const auto &entry = current_string_map.Lookup(str);
    if (!entry.IsEmpty()) {
        // 找到重复
        dictionary_indices.push_back(entry.index + 1);
        tuple_count++;
        return true;
    }

    // 新字符串：检查是否需要触发编码
    if (append_state == DictionaryAppendState::REGULAR &&
        dictionary_offset + str.GetSize() > DICTIONARY_ENCODE_THRESHOLD) {

        auto new_state = TryEncode();
        append_state = new_state;

        if (IsEncoded(append_state)) {
            // 切换到编码模式 - 缓冲此字符串以便后续编码
            if (append_state == DictionaryAppendState::ENCODED_ALL_UNIQUE) {
                encoded_input.offset = i;
            }
            // 添加到编码缓冲区用于批量FSST压缩
            dictionary_encoding_buffer.push_back(str);
            to_encode_string_sum += str.GetSize() * 2 + 7;  // 保守估算
        }
    }

    // 添加到字典
    if (append_state == DictionaryAppendState::REGULAR) {
        // 存储未压缩
        current_string_map.Insert(str);
        string_lengths.push_back(str.GetSize());
        dictionary_offset += str.GetSize();
        dict_count++;
    } else {
        // 在编码模式，仍添加到映射但存储为副本
        auto copy = uncompressed_dictionary_copy.AddBlob(str);
        current_string_map.Insert(copy);
    }

    dictionary_indices.push_back(dict_count - 1);
    tuple_count++;
    return true;
}
```

### 3.8 解压时的模式检测

```cpp
void CompressedStringScanState::Initialize(bool initialize_dictionary) {
    // 加载头部
    auto header_ptr = reinterpret_cast<dict_fsst_compression_header_t *>(baseptr);
    mode = header_ptr->mode;
    dict_count = header_ptr->dict_count;
    auto symbol_table_size = header_ptr->symbol_table_size;

    // 计算偏移
    auto dictionary_dest = AlignValue<idx_t>(DICTIONARY_HEADER_SIZE);
    auto symbol_table_dest = AlignValue<idx_t>(dictionary_dest + dictionary_size);
    auto string_lengths_dest = AlignValue<idx_t>(symbol_table_dest + symbol_table_size);
    auto dictionary_indices_dest = AlignValue<idx_t>(
        string_lengths_dest + string_lengths_space);

    dict_ptr = baseptr + dictionary_dest;
    string_lengths_ptr = baseptr + string_lengths_dest;
    dictionary_indices_ptr = baseptr + dictionary_indices_dest;

    // 如需要则导入FSST解码器
    switch (mode) {
    case DictFSSTMode::FSST_ONLY:
    case DictFSSTMode::DICT_FSST: {
        decoder = new duckdb_fsst_decoder_t;
        duckdb_fsst_import(decoder, baseptr + symbol_table_dest);
        break;
    }
    default:
        break;
    }

    // 加载字符串长度
    BitpackingPrimitives::UnPackBuffer<uint32_t>(
        string_lengths.data(),
        string_lengths_ptr,
        dict_count,
        string_lengths_width
    );
}

string_t CompressedStringScanState::FetchStringFromDict(
    Vector &result, uint32_t dict_offset, idx_t dict_idx) {

    if (dict_idx == 0) {
        return string_t(nullptr, 0);  // NULL
    }

    uint32_t string_len = string_lengths[dict_idx];
    auto str_ptr = char_ptr_cast(dict_ptr + dict_offset);

    switch (mode) {
    case DictFSSTMode::FSST_ONLY:
    case DictFSSTMode::DICT_FSST: {
        if (string_len == 0) return string_t(nullptr, 0);

        if (all_values_inlined) {
            // 解压到内联存储（最多12字节）
            return FSSTPrimitives::DecompressInlinedValue(decoder, str_ptr, string_len);
        } else {
            // 解压到向量缓冲区
            return FSSTPrimitives::DecompressValue(
                decoder, StringVector::GetStringBuffer(result), str_ptr, string_len);
        }
    }
    default:
        // 纯字典模式：无需解压
        return string_t(str_ptr, string_len);
    }
}
```

### 3.9 存储版本要求

```cpp
unique_ptr<AnalyzeState> DictFSSTCompressionStorage::StringInitAnalyze(
    ColumnData &col_data,
    PhysicalType type
) {
    auto &storage_manager = col_data.GetStorageManager();
    if (storage_manager.GetStorageVersion() < 5) {
        // Dict-FSST在v5之前未引入
        return nullptr;
    }

    CompressionInfo info(col_data.GetBlockManager());
    return make_uniq<DictFSSTAnalyzeState>(info);
}
```

## 4. 算法比较

### 4.1 特性对比

| 特性 | Dictionary | FSST | Dict-FSST |
|------|-----------|------|-----------|
| **最佳场景** | 低基数 | 高基数 | 混合基数 |
| **存储版本** | < 5 | < 5 | ≥ 5 |
| **最小压缩比** | 1.2x | 1.2x | 1.2x（估算）|
| **随机访问** | 是 | 是 | 是 |
| **符号表** | 无 | 每段 | 每段（如编码）|
| **编码决策** | 静态 | 静态 | 动态（4KB阈值）|
| **采样策略** | 无 | 25% | 完全分析 |
| **模式选择** | N/A | N/A | 自适应 |

### 4.2 压缩效果示例

```
场景1：低基数数据（10个唯一值，100万行）
  Dictionary: 极佳（只存10个字符串 + 20位/行索引）
  FSST: 一般（无重复利用）
  Dict-FSST: 选择DICTIONARY模式

场景2：高基数数据（90万唯一值，100万行）
  Dictionary: 差（字典太大）
  FSST: 好（符号表压缩）
  Dict-FSST: 选择FSST_ONLY模式

场景3：混合数据（1000唯一值，各重复1000次）
  Dictionary: 好
  FSST: 一般
  Dict-FSST: 选择DICT_FSST模式（字典用FSST编码）
```

### 4.3 选择建议

```
存储版本 < 5:
    低基数 → Dictionary
    高基数 → FSST
    系统自动选择

存储版本 ≥ 5:
    所有情况 → Dict-FSST（自动选择最优模式）
    - 字典 < 4KB → DICTIONARY模式
    - 全唯一 → FSST_ONLY模式
    - 其他 → DICT_FSST模式
```

## 5. 源码文件索引

| 算法 | 文件路径 | 描述 |
|------|---------|------|
| Dictionary | `src/storage/compression/dictionary/compression.cpp` | 压缩实现 |
| Dictionary | `src/storage/compression/dictionary/decompression.cpp` | 解压实现 |
| Dictionary | `src/include/.../dictionary/common.hpp` | 公共定义 |
| FSST | `src/storage/compression/fsst.cpp` | 完整实现 |
| FSST | `third_party/fsst/fsst.h` | FSST库头文件 |
| Dict-FSST | `src/storage/compression/dict_fsst/compression.cpp` | 压缩实现 |
| Dict-FSST | `src/storage/compression/dict_fsst/decompression.cpp` | 解压实现 |
| Dict-FSST | `src/include/.../dict_fsst/common.hpp` | 公共定义 |

## 小结

本章深入分析了DuckDB的三种字符串压缩算法：

1. **Dictionary**：
   - 经典的字典压缩方案
   - 三缓冲区结构：字典、索引缓冲区、选择缓冲区
   - 适合低基数数据，存储版本<5使用

2. **FSST**：
   - 基于符号表的字节序列压缩
   - 25%采样策略构建符号表
   - 适合高基数数据，存储版本<5使用

3. **Dict-FSST**：
   - 混合方案，自适应选择模式
   - 4KB阈值触发FSST编码决策
   - 三种模式：DICTIONARY、DICT_FSST、FSST_ONLY
   - 存储版本≥5的默认字符串压缩算法

关键设计：
- 所有算法都支持随机访问解压
- 使用位压缩优化空间效率
- 最小压缩比1.2x防止无效压缩

下一章将介绍位图压缩算法：Roaring Bitmap。
