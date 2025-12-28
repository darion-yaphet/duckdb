# DuckDB压缩系统深度解析（六）：位图压缩 - Roaring Bitmap

## 引言

有效性掩码（Validity Mask）用于标记列中的NULL值。传统上，每行需要1位来表示是否为NULL，对于一个有100万行的表，仅有效性掩码就需要约125KB。DuckDB实现了Roaring位图压缩来高效存储这些有效性信息，根据NULL值的分布特征自适应选择最优编码方式。

## 1. 有效性掩码基础

### 1.1 有效性掩码结构

```cpp
// 核心定义
using validity_t = uint64_t;  // 每个条目64位

struct ValidityMask {
    static constexpr idx_t BITS_PER_VALUE = 64;
    static constexpr idx_t STANDARD_VECTOR_SIZE = 1024;
    static constexpr idx_t STANDARD_ENTRY_COUNT = 16;  // 1024 / 64
    static constexpr idx_t STANDARD_MASK_SIZE = 128;   // 16 * 8 字节
};

// 2048值的Roaring容器使用:
// 2048 / 64 = 32个条目 = 256字节（未压缩时）
```

### 1.2 设计考量

```
问题:
┌─────────────────────────────────────────────────────────────────────┐
│ 场景1: 全有效（无NULL）                                              │
│   传统: 2048位 = 256字节                                             │
│   优化: 0字节（用标志位表示）                                         │
│                                                                     │
│ 场景2: 稀疏NULL（只有5个NULL）                                        │
│   传统: 256字节                                                      │
│   优化: 5 × 2字节 = 10字节（数组方式）                                │
│                                                                     │
│ 场景3: 连续NULL（前500个是NULL）                                      │
│   传统: 256字节                                                      │
│   优化: 4字节（游程编码）                                             │
│                                                                     │
│ 场景4: 均匀分布NULL                                                   │
│   可能无法压缩，使用位图                                              │
└─────────────────────────────────────────────────────────────────────┘
```

## 2. Roaring位图核心概念

### 2.1 关键常量

```cpp
// 核心大小
static constexpr idx_t ROARING_CONTAINER_SIZE = 2048;          // 每容器值数
static constexpr uint16_t COMPRESSED_SEGMENT_SIZE = 256;       // 段大小
static constexpr uint16_t COMPRESSED_SEGMENT_SHIFT_AMOUNT = 8; // 256 = 1 << 8
static constexpr uint16_t COMPRESSED_SEGMENT_COUNT = 8;        // 2048 / 256

// 容器类型选择阈值
static constexpr uint16_t COMPRESSED_ARRAY_THRESHOLD = 8;      // >=8项使用压缩
static constexpr uint16_t COMPRESSED_RUN_THRESHOLD = 4;        // >=4游程使用压缩

// 各容器类型的最大容量
static constexpr uint16_t MAX_RUN_IDX =
    (UNCOMPRESSED_SIZE - COMPRESSED_SEGMENT_COUNT) / (sizeof(uint8_t) * 2);
static constexpr uint16_t MAX_ARRAY_IDX =
    (UNCOMPRESSED_SIZE - COMPRESSED_SEGMENT_COUNT) / (sizeof(uint8_t) * 1);
static constexpr uint16_t BITSET_CONTAINER_SENTINEL_VALUE = MAX_ARRAY_IDX + 1;

// 元数据位宽
static constexpr uint16_t CONTAINER_TYPE_BITWIDTH = 2;         // 标志位
static constexpr uint16_t RUN_CONTAINER_SIZE_BITWIDTH = 7;     // 最多127游程
static constexpr uint16_t ARRAY_CONTAINER_SIZE_BITWIDTH = 8;   // 最多255项
```

### 2.2 三种容器类型

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Roaring三种容器类型                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. RUN CONTAINER（游程容器）                                           │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 用途: 编码连续的相同值序列（特别是连续NULL）                        │ │
│  │ 结构: [(start, length), (start, length), ...]                    │ │
│  │ 适用: 长连续NULL块或长连续有效块                                   │ │
│  │                                                                   │ │
│  │ 示例: 位置[0-99]是NULL, [200-299]是NULL                           │ │
│  │ 编码: [(0, 100), (200, 100)]                                      │ │
│  │ 大小: 2游程 × 4字节 = 8字节                                        │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  2. ARRAY CONTAINER（数组容器）                                         │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 用途: 编码稀疏的位（NULL或非NULL）                                 │ │
│  │ 结构: [idx0, idx1, idx2, ...]                                     │ │
│  │ 适用: 少量分散的NULL或少量分散的非NULL                             │ │
│  │                                                                   │ │
│  │ 关键优化: 选择较少的一方存储                                       │ │
│  │   - 5个NULL / 2043个有效 → 存储5个NULL位置                         │ │
│  │   - 2043个NULL / 5个有效 → 存储5个有效位置（反转标志）              │ │
│  │                                                                   │ │
│  │ 大小: 项数 × 2字节 (未压缩) 或 8字节头 + 项数 × 1字节 (压缩)       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  3. BITSET CONTAINER（位图容器）                                        │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 用途: 密集数据的回退选项                                          │ │
│  │ 结构: uint64_t bitset[32]  // 2048位                              │ │
│  │ 适用: 无法有效压缩的情况                                          │ │
│  │                                                                   │ │
│  │ 固定大小: 256字节                                                  │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

## 3. 容器类型选择算法

### 3.1 基于成本的选择

```cpp
ContainerMetadata CreateMetadata(uint16_t count, uint16_t array_null,
                                uint16_t array_non_null, uint16_t runs) {
    // 成本计算
    uint16_t null_array_cost =
        array_null < 8 ? array_null * 2      // 2字节/项（未压缩）
                       : 8 + (array_null * 1); // 8字节头 + 1字节/项（压缩）

    uint16_t non_null_array_cost =
        array_non_null < 8 ? array_non_null * 2
                           : 8 + (array_non_null * 1);

    uint16_t run_cost =
        runs < 4 ? runs * 4        // 4字节/游程（未压缩）
                 : 8 + (runs * 2); // 8字节头 + 2字节/游程（压缩）

    uint16_t bitset_cost =
        ((count + 63) / 64) * 8;   // 对齐到64位边界

    // 决策逻辑
    if (all_arrays_too_large && runs_too_large) {
        return BitsetContainer(count);
    }

    if (min(null_array_cost, non_null_array_cost) < run_cost) {
        return ArrayContainer(
            smaller_of_null_or_non_null_array,
            is_inverted);
    }

    return RunContainer(runs);
}
```

### 3.2 选择流程图

```
数据分析
    │
    ├─> 计算各种成本
    │   │
    │   ├─> null_array_cost:     存储NULL位置的成本
    │   ├─> non_null_array_cost: 存储有效位置的成本
    │   ├─> run_cost:            存储游程的成本
    │   └─> bitset_cost:         位图固定成本（256字节）
    │
    ├─> 检查数组/游程是否超出限制
    │   └─> 是 → 使用BITSET
    │
    ├─> 比较 min(array_cost) vs run_cost
    │   │
    │   ├─> array_cost 更小
    │   │   └─> 使用ARRAY（选择NULL或非NULL中较少的）
    │   │
    │   └─> run_cost 更小
    │       └─> 使用RUN
    │
    └─> 最终检查: 如果最优成本 > bitset_cost
            └─> 使用BITSET
```

## 4. 数据布局

### 4.1 段结构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        段内存布局                                        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  [0] uint64_t metadata_offset      (指向元数据段)                       │
│                                                                         │
│  ┌────────────── 容器数据区 ──────────────────────────────────────────┐ │
│  │                                                                     │ │
│  │  Container 0: [类型取决于元数据]                                    │ │
│  │    - 未压缩数组: uint16_t values[]                                  │ │
│  │    - 压缩数组:                                                      │ │
│  │       uint8_t counts[8]     // 每段计数                             │ │
│  │       uint8_t offsets[]     // 段内偏移                             │ │
│  │                                                                     │ │
│  │  Container 1:                                                       │ │
│  │    - 位图: uint64_t bitset[32]                                      │ │
│  │    - 游程: uint8_t counts[8], uint8_t pairs[]                       │ │
│  │                                                                     │ │
│  │  ... 更多容器 ...                                                   │ │
│  │                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│  [对齐填充到8字节边界]                                                   │
│                                                                         │
│  ┌────────────── 元数据区 ──────────────────────────────────────────┐   │
│  │                                                                     │ │
│  │  容器类型 (位压缩, 每个2位):                                        │ │
│  │    [is_run | is_inverted] ...                                      │ │
│  │                                                                     │ │
│  │  游程容器大小 (位压缩, 7位):                                        │ │
│  │    [size] ...                                                       │ │
│  │                                                                     │ │
│  │  数组/位图大小 (8位):                                               │ │
│  │    [size] ...                                                       │ │
│  │                                                                     │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.2 各容器类型布局

#### RUN容器布局

```
未压缩 (< 4游程):
┌─────────────────────────────────────────────────────────────┐
│ RunContainerRLEPair[0]  (4 bytes)                           │
│   uint16_t start: 游程起始位置                               │
│   uint16_t length: 游程长度                                  │
├─────────────────────────────────────────────────────────────┤
│ RunContainerRLEPair[1]  (4 bytes)                           │
├─────────────────────────────────────────────────────────────┤
│ ...                                                         │
└─────────────────────────────────────────────────────────────┘

压缩 (>= 4游程):
┌─────────────────────────────────────────────────────────────┐
│ uint8_t segment_counts[8]  (8 bytes)                        │
│ 每个256值段中的游程数                                        │
├─────────────────────────────────────────────────────────────┤
│ uint8_t run_data[runs*2]                                    │
│ (start_offset, end_offset) 对                               │
└─────────────────────────────────────────────────────────────┘
```

#### ARRAY容器布局

```
未压缩 (< 8项):
┌─────────────────────────────────────────────────────────────┐
│ uint16_t indices[]  (2 bytes/项)                            │
│ 直接存储完整16位索引                                         │
└─────────────────────────────────────────────────────────────┘

压缩 (>= 8项):
┌─────────────────────────────────────────────────────────────┐
│ uint8_t segment_counts[8]  (8 bytes)                        │
│ 每个256值段中的项数                                          │
├─────────────────────────────────────────────────────────────┤
│ uint8_t relative_offsets[]  (1 byte/项)                     │
│ 段内相对偏移                                                 │
└─────────────────────────────────────────────────────────────┘

示例:
容器 [2048 值]
|--段0--|--段1--|--段2--|...
 [0-255] [256-511] [512-767]

压缩数组容器存储索引: [10, 300, 520, 780, 1024, 1536]
segment_counts:   [1, 1, 2, 0, 0, 1, 1, 0]  // 每段计数
relative_offsets: [10, 44, 8, 12, 0, 20, 0]  // 段内偏移
```

#### BITSET容器布局

```
┌─────────────────────────────────────────────────────────────┐
│ uint64_t bitset[32]  (256 bytes)                            │
│ 2048位直接存储                                               │
│ 1 = 有效, 0 = NULL (或反转)                                  │
└─────────────────────────────────────────────────────────────┘
```

## 5. 压缩流程

### 5.1 分析阶段

```cpp
class RoaringAnalyzeState {
    // 位掩码表：预计算每个8位字节的统计
    BitmaskTableEntry bitmask_table[256];

    // 当前容器统计
    uint16_t one_count = 0;       // 设置为1的位数
    uint16_t zero_count = 0;      // 设置为0的位数
    uint16_t run_count = 0;       // 游程数（连续0序列）
    bool last_bit_set;            // 上一个位是否设置

    // 累积统计
    ContainerMetadataCollection metadata_collection;
    vector<ContainerMetadata> container_metadata;
};

// 关键方法：逐字节处理有效性掩码
void HandleByte(RoaringAnalyzeState &state, uint8_t byte_value) {
    // 使用预计算表进行快速分析
    auto bit_info = state.bitmask_table[byte_value];

    // 计算转换以检测游程
    state.run_count += bit_info.run_count +
                      (bit_info.first_bit_set == false &&
                       (!state.count || state.last_bit_set == true));

    state.one_count += bit_info.valid_count;
    state.zero_count += 8 - bit_info.valid_count;
    state.last_bit_set = bit_info.last_bit_set;
}
```

### 5.2 位掩码表优化

```cpp
struct BitmaskTableEntry {
    uint8_t first_bit_set : 1;   // 最左位是否为1
    uint8_t last_bit_set : 1;    // 最右位是否为1
    uint8_t valid_count : 6;     // 有多少个1 (0-8)
    uint8_t run_count;           // 有多少个1->0转换
};

// 初始化时构建一次
static unsafe_unique_array<BitmaskTableEntry> CreateBitmaskTable() {
    auto result = make_unsafe_uniq_array<BitmaskTableEntry>(256);

    for (uint16_t val = 0; val <= 255; val++) {
        bool previous_bit;
        auto &entry = result[val];
        entry.valid_count = 0;
        entry.run_count = 0;

        for (uint8_t i = 0; i < 8; i++) {
            const bool bit_set = val & (1 << i);
            if (!i) {
                entry.first_bit_set = bit_set;
            } else if (i == 7) {
                entry.last_bit_set = bit_set;
            }
            entry.valid_count += bit_set;

            // 计算从1到0的转换
            if (i && !bit_set && previous_bit == true) {
                entry.run_count++;
            }
            previous_bit = bit_set;
        }
    }
    return result;
}
```

**优化效果**：一次表查找代替8次位操作。

### 5.3 压缩阶段

```cpp
class ContainerCompressionState {
    // 缓冲追加状态
    uint16_t length = 0;          // 缓冲游程长度
    bool last_bit_set;            // 当前游程是1还是0

    // 容器状态
    uint16_t appended_count = 0;  // 容器中总值数
    uint16_t null_count = 0;      // NULL值计数

    // 三种追加路径
    append_func_t append_function;  // 指向: AppendBitset, AppendRun, 或 AppendArray
};

// 位图追加
void AppendBitset(ContainerCompressionState &state, bool null, uint16_t amount) {
    if (null) {
        SetInvalidRange(mask, state.appended_count, state.appended_count + amount);
    }
}

// 游程追加
void AppendRun(ContainerCompressionState &state, bool null, uint16_t amount) {
    if (null && run_idx < MAX_RUN_IDX) {
        auto &run = state.runs[run_idx];
        run.start = appended_count;
        // ... 设置游程长度
    }
}

// 数组追加
template <bool INVERTED>
void AppendToArray(ContainerCompressionState &state, bool null, uint16_t amount) {
    if (INVERTED != null) {
        auto index = state.appended_count;
        if (compressed) {
            compressed_array[idx] = segment_offset + i;
        } else {
            arrays[idx] = appended_count + i;
        }
    }
}
```

### 5.4 扫描/解压阶段

```cpp
class RoaringScanState {
    ContainerMetadataCollection metadata_collection;
    vector<ContainerMetadata> container_metadata;
    vector<idx_t> data_start_position;

    unique_ptr<ContainerScanState> current_container;
};

// 游程容器扫描：展开(start, length)对
void RunContainerScanState::ScanPartial(...) {
    while (!finished && result_idx < to_scan) {
        auto start_of_run = MaxValue(MinValue(run.start, ...), ...);
        auto run_end = run.start + 1 + run.length;

        // 标记整个游程为无效
        SetInvalidRange(result_mask, start, end);
    }
}

// 压缩数组容器扫描：解码段偏移
void CompressedArrayContainerScanState::LoadNextValue() {
    this->value = segment++;  // 获取段基址
    this->value += reinterpret_cast<uint8_t *>(this->data)[array_index];
}

// 位图容器扫描：直接复制（可能需要位移）
void BitsetContainerScanState::ScanPartial(...) {
    if (aligned_case) {
        ValidityUncompressed::AlignedScan(...);  // 快速路径
    } else {
        ValidityUncompressed::UnalignedScan(...); // 处理位对齐
    }
}
```

## 6. 特殊优化

### 6.1 Empty Validity优化

当列没有NULL值时，使用`EmptyValidityCompression`：

```cpp
class EmptyValidityCompression {
    static void Compress(CompressionState &state_p, Vector &scan_vector, idx_t count) {
        UnifiedVectorFormat format;
        scan_vector.ToUnifiedFormat(count, format);
        state.non_nulls += format.validity.CountValid(count);
        state.count += count;
    }

    static void FinalizeCompress(CompressionState &state_p) {
        if (state.non_nulls == state.count) {
            // 所有值都有效，标记统计并使用空段
            compressed_segment->stats.statistics.SetHasNoNullFast();
            checkpoint_state.FlushSegment(..., 0);  // 需要0字节！
        }
    }
};
```

**优势**：无NULL的列几乎不占空间。

### 6.2 未压缩有效性回退

对于分散的NULL模式：

```cpp
class ValidityUncompressed {
    // 预计算位掩码用于高效范围操作
    static const validity_t LOWER_MASKS[65];  // [0x0, 0x1, 0x3, ..., 0xFFFF...]
    static const validity_t UPPER_MASKS[65];  // [0x0, 0x8000..., 0xC000..., ...]

    // 优化扫描，处理非对齐位
    static void UnalignedScan(data_ptr_t input, idx_t input_size,
                              idx_t input_start, Vector &result,
                              idx_t result_offset, idx_t scan_count);
};
```

## 7. 实际示例

### 7.1 稀疏NULL场景

```cpp
// 输入: 2048个值，NULL分散在: [10, 50, 100, 200, 500]

RoaringAnalyzeState state;
RoaringStateAppender::AppendVector(state, input, 2048);

// 分析确定:
// - zero_count = 5 (NULL数)
// - one_count = 2043 (有效数)
// - run_count = 5 (每个NULL是独立游程)

// 成本比较:
//   - null_array: 5 * 2 = 10字节 (< 8, 使用未压缩)
//   - non_null_array: 8 + (2043 * 1) = 2051字节
//   - run_array: 5 * 4 = 20字节
//   - bitset: ((2048 + 63) / 64) * 8 = 256字节
//
// 获胜者: ARRAY_CONTAINER存储nulls (inverted=true), 10字节!
```

### 7.2 长连续NULL场景

```cpp
// 输入: 2048个值，模式:
// [200个NULL] [1000个有效] [500个NULL] [348个有效]

// 分析:
// - zero_count = 700
// - one_count = 1348
// - run_count = 2 (一段NULL，一段有效，一段NULL)

// 成本比较:
//   - null_array: 8 + 700 = 708字节
//   - non_null_array: 8 + 1348 = 1356字节
//   - run_array: 2 * 4 = 8字节
//   - bitset: 256字节
//
// 获胜者: RUN_CONTAINER, 8字节!
```

### 7.3 段内存布局示例

```cpp
// 压缩3个容器后的段:
// - Container 0: 2048值, ARRAY_CONTAINER存储null (10字节)
// - Container 1: 2048值, BITSET_CONTAINER (256字节)
// - Container 2: 512值, RUN_CONTAINER有2个游程 (8字节)

struct SegmentMemory {
    // 偏移 0
    uint64_t metadata_offset = 280;  // 指向元数据区

    // 偏移 8 - Container 0 (ARRAY)
    uint16_t array_indices[5] = {10, 50, 100, 200, 500};  // 10字节

    // 偏移 18 - 对齐填充
    uint8_t padding[6] = {0};  // 对齐到8字节边界 -> 偏移24

    // 偏移 24 - Container 1 (BITSET)
    uint64_t bitset[32];  // 256字节 -> 偏移280

    // 偏移 280 - Container 2 (RUN)
    uint8_t run_counts[8] = {1, 1, 0, 0, 0, 0, 0, 0};  // 8字节
    uint8_t run_data[4] = {0, 200, 200, 512};  // 4字节

    // 偏移 292 - 元数据
    // 容器类型 (位压缩):
    //   00 (array, not_inverted)
    //   00 (bitset, not_inverted)
    //   10 (run, not_inverted)
    // 打包 = 0b10_00_00 = 0x20 (1字节)
};
```

## 8. 性能特性

| 场景 | 容器类型 | 空间 | 速度 |
|------|---------|------|------|
| 全NULL (2048) | EMPTY | ~0字节 | 常量时间 |
| 无NULL | EMPTY | ~0字节 | 常量时间 |
| 稀疏NULL (<8) | ARRAY | 10-16字节 | O(n)扫描 |
| 长NULL游程 | RUN | 8-16字节 | O(游程数)展开 |
| 密集NULL (>50%) | BITSET | 256字节 | O(1)/64位 |
| 混合模式 | 按成本选择 | 变化 | 最优选择 |

## 9. 源码文件索引

| 文件路径 | 描述 |
|---------|------|
| `src/storage/compression/roaring/analyze.cpp` | 分析逻辑 |
| `src/storage/compression/roaring/compress.cpp` | 压缩逻辑 |
| `src/storage/compression/roaring/common.cpp` | 工具函数 |
| `src/storage/compression/roaring/metadata.cpp` | 元数据处理 |
| `src/storage/compression/roaring/scan.cpp` | 扫描/解压 |
| `src/storage/compression/empty_validity.cpp` | Empty优化 |
| `src/storage/compression/validity_uncompressed.cpp` | 未压缩回退 |

## 小结

本章深入分析了DuckDB的Roaring位图压缩：

1. **自适应策略**：根据数据特征在分析时选择容器类型
2. **三种容器**：
   - RUN: 连续NULL的游程编码
   - ARRAY: 稀疏NULL的索引数组
   - BITSET: 密集数据的位图回退
3. **反转编码**：数组容器选择NULL或非NULL中较少的存储
4. **段级编码**：压缩数组使用8字节段头减少偏移范围
5. **位掩码表**：预计算256项表实现O(1)游程检测
6. **Empty优化**：无NULL列几乎零成本

关键设计洞察：
- 成本驱动选择确保最优存储
- 元数据位压缩减少开销
- 支持随机访问解压

下一章将介绍通用压缩和存储引擎集成。
