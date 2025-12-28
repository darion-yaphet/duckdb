# DuckDB压缩系统深度解析（四）：浮点数压缩算法

## 引言

浮点数压缩是数据库压缩中的一个难题。与整数不同，浮点数的IEEE 754表示使得传统的位压缩技术效果不佳。DuckDB实现了四种专门针对浮点数的压缩算法：Chimp、Patas、ALP和ALPRD。本章将深入分析这些算法的原理和实现。

## 1. 算法概览

| 算法 | 核心原理 | 适用场景 | 状态 |
|------|---------|---------|------|
| Chimp | XOR + 环形缓冲区 | 时序数据 | 已废弃（仅解压） |
| Patas | XOR + 字节对齐 | 时序数据 | 已废弃（仅解压） |
| ALP | 十进制编码 + FOR | 金融/科学数据 | 活跃 |
| ALPRD | 分裂编码 + 字典 | 通用场景 | 活跃 |

## 2. Chimp压缩算法

### 2.1 算法原理

Chimp是一种基于XOR的压缩算法，利用相邻浮点数之间的相似性进行压缩：

```
核心思想：
┌─────────────────────────────────────────────────────────────────────┐
│ 如果两个浮点数相似，它们的二进制表示的XOR结果会有很多前导零和尾随零 │
│                                                                     │
│ 示例：                                                               │
│   A = 3.14159 = 0x40_09_21_F9_FE_76_C8_B4                           │
│   B = 3.14160 = 0x40_09_21_FA_5E_35_3F_7D                           │
│   A XOR B     = 0x00_00_00_03_A0_43_F7_C9                           │
│                 ↑↑↑↑ 4字节前导零                                    │
│                                                                     │
│ 只需存储非零部分，加上位置信息，即可大幅压缩                          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 环形缓冲区

Chimp使用128元素的环形缓冲区存储历史值，用于查找最佳参考值：

```cpp
// src/include/duckdb/storage/compression/chimp/algorithm/ring_buffer.hpp

template <class CHIMP_TYPE>
class RingBuffer {
    static constexpr uint8_t RING_SIZE = 128;

    uint64_t buffer[RING_SIZE];     // 存储历史值
    uint64_t indices[INDICES_SIZE]; // 哈希表快速查找
    uint64_t index;                 // 当前位置

    // 使用LSB哈希查找参考值
    uint64_t Key(CHIMP_TYPE value) {
        return value & (INDICES_SIZE - 1);  // 取低位作为哈希键
    }

    uint64_t IndexOf(uint64_t key) {
        return indices[key];
    }

    void Insert(CHIMP_TYPE value) {
        buffer[index & (RING_SIZE - 1)] = value;
        indices[Key(value)] = index;
        index++;
    }
};
```

### 2.3 四种标志位

每个压缩值使用2位标志表示压缩模式：

```cpp
enum class Flags : uint8_t {
    VALUE_IDENTICAL = 0,          // XOR结果为0，值完全相同
    TRAILING_EXCEEDS_THRESHOLD = 1, // 尾随零超过阈值
    LEADING_ZERO_EQUALITY = 2,    // 前导零与上一个相同
    LEADING_ZERO_LOAD = 3         // 新的前导零值
};
```

对应的存储格式：

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Chimp四种压缩模式                                  │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  Flag 0: VALUE_IDENTICAL（值相同）                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 存储: [7位索引]                                                   │ │
│  │ 原理: XOR=0，只存储参考值在环形缓冲区中的索引                      │ │
│  │ 大小: 7位                                                         │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  Flag 1: TRAILING_EXCEEDS_THRESHOLD（高尾随零）                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 存储: [7位索引] + [3位前导零] + [6位有效位数] + [有效位]           │ │
│  │ 原理: 右移去除尾随零后存储                                        │ │
│  │ 大小: 16位元数据 + 有效位                                         │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  Flag 2: LEADING_ZERO_EQUALITY（前导零相同）                            │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 存储: [有效位]                                                    │ │
│  │ 原理: 复用上一个值的前导零信息                                    │ │
│  │ 大小: (64 - 前导零) 位                                            │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  Flag 3: LEADING_ZERO_LOAD（新前导零）                                  │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 存储: [3位前导零] + [有效位]                                       │ │
│  │ 原理: 存储新的前导零计数和有效位                                  │ │
│  │ 大小: 3位 + (64 - 前导零) 位                                      │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.4 压缩流程

```cpp
static void CompressValue(CHIMP_TYPE in, State &state) {
    // 1. 从环形缓冲区查找参考值
    auto key = state.ring_buffer.Key(in);
    const CHIMP_TYPE reference_index = state.ring_buffer.IndexOf(key);
    CHIMP_TYPE reference_value = state.ring_buffer.Value(reference_index);

    // 2. XOR计算
    CHIMP_TYPE xor_result = in ^ reference_value;

    // 3. 计算前导零和尾随零
    uint32_t trailing_zeros = CountZeros<CHIMP_TYPE>::Trailing(xor_result);
    uint8_t leading_zeros = CountZeros<CHIMP_TYPE>::Leading(xor_result);

    // 4. 根据XOR结果选择压缩模式
    if (xor_result == 0) {
        // 模式0：完全相同
        state.flag_buffer.Insert(Flags::VALUE_IDENTICAL);
        state.output.WriteValue<uint8_t, 7>(reference_index);
    }
    else if (trailing_zeros > THRESHOLD) {
        // 模式1：高尾随零
        state.flag_buffer.Insert(Flags::TRAILING_EXCEEDS_THRESHOLD);
        // 存储索引、前导零、有效位数和XOR结果
        state.packed_data.Insert(reference_index, leading_zeros, significant_bits);
        state.output.WriteValue(xor_result >> trailing_zeros, significant_bits);
    }
    else if (leading_zeros == state.previous_leading_zeros) {
        // 模式2：前导零相同
        state.flag_buffer.Insert(Flags::LEADING_ZERO_EQUALITY);
        state.output.WriteValue(xor_result, 64 - leading_zeros);
    }
    else {
        // 模式3：新前导零
        state.flag_buffer.Insert(Flags::LEADING_ZERO_LOAD);
        state.leading_zero_buffer.Insert(leading_zeros);
        state.output.WriteValue(xor_result, 64 - leading_zeros);
        state.previous_leading_zeros = leading_zeros;
    }

    // 5. 更新环形缓冲区
    state.ring_buffer.Insert(in);
}
```

### 2.5 数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                      Chimp压缩块布局                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (4 bytes)                                             │  │
│  │ 元数据大小信息                                                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ XOR Data (可变长度)                                          │  │
│  │ - 第一个值: 64位完整存储                                      │  │
│  │ - 后续值: XOR结果的有效位 (可变位数)                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Flag Buffer (每4个值1字节 = 256字节/1024值)                   │  │
│  │ 2位/值的压缩模式标志                                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Leading Zero Buffer (最多384字节)                            │  │
│  │ 3位/值的前导零编码（仅LEADING_ZERO_LOAD模式）                 │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Packed Data Buffer (TRAILING_EXCEEDS_THRESHOLD专用)          │  │
│  │ [7位索引][3位前导零][6位有效位数]                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## 3. Patas压缩算法

### 3.1 算法原理

Patas是Chimp的简化版本，主要区别是使用**字节对齐**存储而非位对齐：

```cpp
// Chimp: 位对齐，最大压缩但复杂
state.output.WriteValue<CHIMP_TYPE>(xor_result, significant_bits);

// Patas: 字节对齐，简单但略大
const uint8_t significant_bytes = (significant_bits >> 3) + ((significant_bits & 7) != 0);
state.byte_writer.WriteValue<EXACT_TYPE>(xor_result >> trailing_zeros, significant_bytes);
```

### 3.2 压缩流程

```cpp
static void StoreCompressed(EXACT_TYPE value, State &state) {
    // 1. 查找参考值
    auto key = state.ring_buffer.Key(value);
    uint64_t reference_index = state.ring_buffer.IndexOf(key);

    // 2. 验证参考值有效性
    const bool difference_too_big =
        ((state.ring_buffer.Size() + 1) - reference_index) >= 128;
    if (difference_too_big) {
        reference_index = state.ring_buffer.Size();
    }

    // 3. XOR操作
    EXACT_TYPE xor_result = value ^ state.ring_buffer.Value(reference_index);

    // 4. 分析位模式
    const uint8_t trailing_zero = CountZeros<EXACT_TYPE>::Trailing(xor_result);
    const uint8_t leading_zero = CountZeros<EXACT_TYPE>::Leading(xor_result);
    const bool is_equal = (xor_result == 0);

    // 5. 计算有效字节数（字节对齐）
    const uint8_t significant_bits = !is_equal *
        (EXACT_TYPE_BITSIZE - trailing_zero - leading_zero);
    const uint8_t significant_bytes =
        (significant_bits >> 3) + ((significant_bits & 7) != 0);

    // 6. 字节对齐写入
    state.byte_writer.WriteValue<EXACT_TYPE>(
        xor_result >> (trailing_zero - is_equal),
        significant_bytes * 8  // 字节对齐
    );

    // 7. 存储元数据
    state.ring_buffer.Insert(value);
    state.UpdateMetadata(trailing_zero, significant_bytes,
                        state.ring_buffer.Size() - reference_index);
}
```

### 3.3 与Chimp的对比

| 特性 | Chimp | Patas |
|------|-------|-------|
| 对齐方式 | 位对齐 | 字节对齐 |
| Flag缓冲区 | 有（2位/值） | 无（内嵌元数据） |
| Leading Zero缓冲区 | 有（3位/值） | 无 |
| 压缩率 | 更高 | 略低 |
| 解压速度 | 较慢 | 更快 |
| 实现复杂度 | 高 | 低 |

## 4. ALP压缩算法

### 4.1 算法原理

ALP（Adaptive Lossless Floating-Point）采用完全不同的策略：将浮点数转换为整数，然后使用FOR（Frame of Reference）压缩：

```
核心思想：
┌─────────────────────────────────────────────────────────────────────┐
│ 很多浮点数可以表示为：value = integer × 10^exponent               │
│                                                                     │
│ 示例：3.14159 ≈ 314159 × 10^(-5)                                   │
│       编码为整数: 314159                                            │
│       存储: exponent=5, factor=0, 然后对整数进行FOR压缩             │
│                                                                     │
│ 对于无法精确编码的值，作为"异常"存储原始浮点数                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.2 编码/解码机制

```cpp
struct AlpEncodingIndices {
    uint8_t exponent;  // 10的幂次（0-18 for double）
    uint8_t factor;    // 分数因子（0-20 for double）
};

// 浮点数 → 整数
static int64_t EncodeValue(T value, AlpEncodingIndices indices) {
    // 乘以10^exponent，除以10^factor
    T tmp_encoded = value * EXP_ARR[indices.exponent] * FRAC_ARR[indices.factor];
    return NumberToInt64(tmp_encoded);
}

// 整数 → 浮点数
static T DecodeValue(int64_t encoded_value, AlpEncodingIndices indices) {
    // 逆操作
    T decoded = (T)encoded_value * FACT_ARR[indices.factor] *
                FRAC_ARR[indices.exponent];
    return decoded;
}

// 魔数技术：移除小数部分
// Double: 2^51 + 2^52 = 6755399441055744
static int64_t NumberToInt64(T n) {
    T tmp = n + MAGIC_NUMBER - MAGIC_NUMBER;  // 截断小数
    return LossyNumericCast<int64_t>(tmp);
}
```

### 4.3 两级采样策略

ALP使用两级采样来选择最优的exponent/factor组合：

```
┌────────────────────────────────────────────────────────────────────────┐
│                         ALP两级采样策略                                 │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  第一级：段级采样                                                       │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • 从整个Row Group中采样多个向量                                   │ │
│  │ • 对每个采样向量，尝试所有exponent/factor组合                     │ │
│  │ • 统计每种组合被选为最优的次数                                    │ │
│  │ • 保留出现频率最高的Top-K组合（K=5）                              │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                              │                                         │
│                              v                                         │
│  第二级：向量级选择                                                     │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ • 对每个实际向量，只测试Top-K组合                                 │ │
│  │ • 选择压缩效果最好的组合                                          │ │
│  │ • 大幅减少组合搜索空间（从400+减少到5）                           │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 4.4 压缩流程

```cpp
static void Compress(const T *input_vector, idx_t n_values,
                    AlpEncodingIndices indices, CompressionData &data) {
    // 1. 编码所有值
    uint16_t exceptions_idx = 0;
    for (idx_t i = 0; i < n_values; i++) {
        // 尝试编码
        int64_t encoded = EncodeValue(input_vector[i], indices);
        T decoded = DecodeValue(encoded, indices);

        data.encoded_integers[i] = encoded;

        // 检查是否可逆（无损）
        bool is_exception = (decoded != input_vector[i]);

        if (is_exception) {
            data.exceptions_positions[exceptions_idx] = i;
            exceptions_idx++;
        }
    }
    data.exceptions_count = exceptions_idx;

    // 2. 保存异常值的原始浮点数
    for (idx_t i = 0; i < exceptions_idx; i++) {
        idx_t pos = data.exceptions_positions[i];
        data.exceptions[i] = input_vector[pos];
        // 用非异常值替换，便于FOR计算
        data.encoded_integers[pos] = find_non_exception_value();
    }

    // 3. 应用FOR（Frame of Reference）
    int64_t min_value = *min_element(data.encoded_integers, ...);
    data.frame_of_reference = min_value;

    for (idx_t i = 0; i < n_values; i++) {
        data.encoded_integers[i] -= min_value;
    }

    // 4. 位压缩到最小宽度
    int64_t max_value = *max_element(data.encoded_integers, ...);
    data.bit_width = ceil(log2(max_value + 1));

    BitpackingPrimitives::PackBuffer(data.values_encoded, ..., data.bit_width);
}
```

### 4.5 解压流程

```cpp
static void Decompress(uint8_t *for_encoded, T *output, idx_t count,
                      AlpEncodingIndices indices,
                      uint16_t exceptions_count, T *exceptions,
                      const uint16_t *exceptions_positions,
                      uint64_t frame_of_reference, uint8_t bit_width) {

    // 1. 位解压到int64
    uint64_t encoded_integers[ALP_VECTOR_SIZE];
    BitpackingPrimitives::UnPackBuffer<uint64_t>(
        encoded_integers, for_encoded, count, bit_width);

    // 2. 逆FOR
    for (idx_t i = 0; i < count; i++) {
        encoded_integers[i] += frame_of_reference;
    }

    // 3. 解码为浮点数
    for (idx_t i = 0; i < count; i++) {
        output[i] = DecodeValue((int64_t)encoded_integers[i], indices);
    }

    // 4. 恢复异常值
    for (idx_t i = 0; i < exceptions_count; i++) {
        output[exceptions_positions[i]] = exceptions[i];
    }
}
```

### 4.6 数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                    ALP压缩块布局（每向量1024值）                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (4 bytes)                                             │  │
│  │ 元数据指针                                                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Compressed Data                                              │  │
│  │ ┌─────────────────────────────────────────────────────────┐ │  │
│  │ │ Exponent (1 byte): 0-18                                 │ │  │
│  │ │ Factor (1 byte): 0-20                                   │ │  │
│  │ │ FOR Base (8 bytes): 最小值                              │ │  │
│  │ │ Bit Width (1 byte): 每值位数                            │ │  │
│  │ │ Bit-Packed FOR Values (可变)                            │ │  │
│  │ │ Exceptions Count (2 bytes)                              │ │  │
│  │ │ Exception Positions (2 bytes × count)                   │ │  │
│  │ │ Exception Values (8 bytes × count)                      │ │  │
│  │ └─────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Metadata Section (向下增长)                                  │  │
│  │ 每向量的偏移信息                                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

## 5. ALPRD压缩算法

### 5.1 算法原理

ALPRD将浮点数的二进制表示**分裂**为高位（left）和低位（right）两部分，对高位使用字典压缩：

```
核心思想：
┌─────────────────────────────────────────────────────────────────────┐
│ 浮点数的高位（符号位、指数、高位尾数）在同类数据中往往重复性高          │
│                                                                     │
│ 64位Double分裂示例（切割点=16）：                                    │
│ ┌──────────────────────────────────────────────────────────────┐   │
│ │  High Part (48 bits)          │  Low Part (16 bits)          │   │
│ │  符号 + 指数 + 高位尾数        │  低位尾数                     │   │
│ └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ 对High Part建立字典（最多8个条目），用3位索引替代48位               │
│ Low Part直接位压缩存储                                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 切割点选择

```cpp
static double FindBestDictionary(const vector<EXACT_TYPE> &values,
                                CompressionData &compression_data) {
    uint8_t best_right_bit_width = 0;
    double best_dict_size = INT32_MAX;

    // 尝试切割点1-16
    for (idx_t cutting_pos = 1; cutting_pos <= 16; cutting_pos++) {
        // 分裂值
        // left_part = value >> cutting_pos
        // right_part = value & ((1 << cutting_pos) - 1)

        // 为left_part建立字典
        double estimated_size = BuildLeftPartsDictionary(
            values, cutting_pos, compression_data);

        if (estimated_size < best_dict_size) {
            best_dict_size = estimated_size;
            best_right_bit_width = cutting_pos;
        }
    }

    compression_data.right_bit_width = best_right_bit_width;
}
```

### 5.3 字典构建

```cpp
template <bool PERSIST_DICT>
static double BuildLeftPartsDictionary(const vector<EXACT_TYPE> &values,
                                       uint8_t right_bit_width,
                                       CompressionData &data) {
    unordered_map<EXACT_TYPE, int32_t> left_parts_hash;

    // 统计每个left_part的出现次数
    for (auto &value : values) {
        auto left_part = value >> right_bit_width;
        left_parts_hash[left_part]++;
    }

    // 按频率排序（最常见的在前）
    vector<AlpRDLeftPartInfo> sorted;
    for (auto &pair : left_parts_hash) {
        sorted.emplace_back(pair.second, pair.first);  // (count, value)
    }
    sort(sorted.begin(), sorted.end(),
         [](const auto &a, const auto &b) { return a.count > b.count; });

    // 保留Top-8进入字典，其余作为异常
    uint32_t exceptions_count = 0;
    for (idx_t i = 8; i < sorted.size(); i++) {
        exceptions_count += sorted[i].count;
    }

    // 计算字典位宽
    uint64_t dict_size = min(8UL, sorted.size());
    uint8_t left_bit_width = ceil(log2(dict_size));  // 0-3位

    if (PERSIST_DICT) {
        for (idx_t i = 0; i < dict_size; i++) {
            data.left_parts_dict[i] = sorted[i].hash;
            data.left_parts_dict_map[sorted[i].hash] = i;
        }
    }

    // 估算压缩大小
    double estimated = right_bit_width + left_bit_width +
                      (exceptions_count * EXCEPTION_OVERHEAD) / values.size();
    return estimated;
}
```

### 5.4 编码流程

```cpp
// 对每个值：
for (idx_t i = 0; i < count; i++) {
    EXACT_TYPE value = input[i];

    // 分裂
    EXACT_TYPE right_part = value & ((1ULL << right_bit_width) - 1);
    EXACT_TYPE left_part = value >> right_bit_width;

    // 查字典
    auto dict_entry = data.left_parts_dict_map.find(left_part);

    if (dict_entry != end()) {
        // 在字典中：用小索引编码
        uint16_t dict_index = dict_entry->second;
        left_parts_encoded[i] = dict_index;  // 0-7 (3位)
    } else {
        // 不在字典中：作为异常
        exceptions[exceptions_idx] = value;
        exceptions_positions[exceptions_idx] = i;
        exceptions_idx++;
    }

    right_parts[i] = right_part;
}

// 分别位压缩两部分
BitpackingPrimitives::PackBuffer(right_parts_encoded, ..., right_bit_width);
BitpackingPrimitives::PackBuffer(left_parts_encoded, ..., left_bit_width);
```

### 5.5 数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                      ALPRD压缩块布局                                │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (7 bytes)                                             │  │
│  │ ┌─────────────────────────────────────────────────────────┐ │  │
│  │ │ Metadata Pointer (4 bytes)                              │ │  │
│  │ │ Right Bit Width (1 byte): 1-16                          │ │  │
│  │ │ Left Bit Width (1 byte): 0-3                            │ │  │
│  │ │ Dictionary Size (1 byte): 0-8                           │ │  │
│  │ └─────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Dictionary (2-16 bytes)                                      │  │
│  │ left_parts_dict[0..dict_size-1]                              │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Bit-Packed Right Parts (right_bit_width × 1024 / 8)          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Bit-Packed Left Parts (left_bit_width × 1024 / 8)            │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Exceptions                                                   │  │
│  │ ┌─────────────────────────────────────────────────────────┐ │  │
│  │ │ Exception Count (2 bytes)                               │ │  │
│  │ │ Exception Values (2 bytes × count)                      │ │  │
│  │ │ Exception Positions (2 bytes × count)                   │ │  │
│  │ └─────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 5.6 压缩效果示例

```
输入: 3.14159 = 0x400921F9FE76C8B4

切割点 = 8:
  Right Part (8 bits): 0xB4 = 180
  Left Part (56 bits): 0x400921F9FE76C8

假设 left_part 在字典中排第2（共5个不同值）:
  Dictionary Index: 2 (需要 ceil(log2(5)) = 3 位)
  Right Part: 8 位

总计: 3 + 8 = 11 位 vs 原始 64 位
压缩率: 64 / 11 ≈ 5.8x
```

## 6. 算法比较

### 6.1 特性对比

| 特性 | Chimp | Patas | ALP | ALPRD |
|------|-------|-------|-----|-------|
| 核心原理 | XOR | XOR | 十进制编码 | 分裂+字典 |
| 历史缓冲区 | 128值 | 128值 | 无 | 无 |
| 位/字节对齐 | 位 | 字节 | 位 | 位 |
| 采样策略 | 无 | 无 | 两级 | 隐式 |
| 典型压缩率 | 2-4x | 1.5-3x | 3-8x | 5-15x |
| 解压速度 | 中等 | 快 | 快 | 快 |
| 最佳场景 | 时序数据 | 时序数据 | 金融数据 | 通用 |
| 当前状态 | 废弃 | 废弃 | 活跃 | 活跃 |

### 6.2 选择建议

```
数据分析
    │
    ├─> 时序数据，值变化小? ─yes─> Chimp/Patas (如果仍支持)
    │
    ├─> 金融数据，固定小数位? ─yes─> ALP
    │       (如价格: 12.34, 56.78)
    │
    ├─> 科学数据，指数相似? ─yes─> ALPRD
    │       (如: 1.23e10, 4.56e10)
    │
    └─> 其他 ─> ALP/ALPRD自动选择
```

## 7. 源码文件索引

| 算法 | 文件路径 | 描述 |
|------|---------|------|
| Chimp | `src/storage/compression/chimp/` | 实现文件 |
| Chimp | `src/include/.../chimp/algorithm/chimp128.hpp` | 核心算法 |
| Chimp | `src/include/.../chimp/algorithm/ring_buffer.hpp` | 环形缓冲区 |
| Patas | `src/storage/compression/patas.cpp` | 实现文件 |
| Patas | `src/include/.../patas/algorithm/patas.hpp` | 核心算法 |
| ALP | `src/storage/compression/alp/alp.cpp` | 实现文件 |
| ALP | `src/include/.../alp/algorithm/alp.hpp` | 核心算法 |
| ALPRD | `src/storage/compression/alprd.cpp` | 实现文件 |
| ALPRD | `src/include/.../alprd/algorithm/alprd.hpp` | 核心算法 |

## 小结

本章深入分析了DuckDB的四种浮点压缩算法：

1. **Chimp**：基于XOR的压缩，使用环形缓冲区查找相似值，适合时序数据
2. **Patas**：Chimp的简化版，字节对齐提高解压速度
3. **ALP**：将浮点数编码为整数，使用FOR压缩，适合固定小数位数据
4. **ALPRD**：分裂浮点数，高位用字典压缩，适合通用场景

当前DuckDB推荐使用ALP和ALPRD，它们提供更好的压缩率和更广泛的适用性。

下一章将介绍字符串压缩算法：Dictionary、FSST和Dict-FSST。
