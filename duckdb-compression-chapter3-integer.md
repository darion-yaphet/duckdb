# DuckDB压缩系统深度解析（三）：整数压缩算法 - Bitpacking与RLE

## 引言

整数是数据库中最常见的数据类型之一。DuckDB为整数压缩实现了两种高效算法：Bitpacking（位压缩）和RLE（游程编码）。这两种算法各有特点：Bitpacking适用于值范围较小的数据，而RLE适用于重复值较多的数据。本章将深入分析这两种算法的实现细节。

## 1. RLE（游程编码）压缩

### 1.1 算法原理

RLE（Run-Length Encoding，游程编码）是一种简单但高效的压缩算法，其核心思想是将连续重复的值编码为(值, 计数)对：

```
原始数据: [5, 5, 5, 5, 8, 8, 3, 3, 3]
RLE编码:  [(5, 4), (8, 2), (3, 3)]
           ↓
          3个游程替代9个值
```

### 1.2 核心数据结构

```cpp
// src/storage/compression/rle.cpp

// 游程计数类型：uint16_t，最大值65535
using rle_count_t = uint16_t;

// RLE状态管理
template <class T>
struct RLEState {
    idx_t seen_count;           // 已见的游程数
    T last_value;               // 上一个值
    rle_count_t last_seen_count; // 当前游程长度
    void *dataptr;              // 回调数据指针
    bool all_null = true;       // 是否全为NULL

    template <class OP>
    void Flush() {
        // 将当前游程写入
        OP::template Operation<T>(last_value, last_seen_count, dataptr, all_null);
    }

    template <class OP = EmptyRLEWriter>
    void Update(const T *data, ValidityMask &validity, idx_t idx) {
        if (validity.RowIsValid(idx)) {
            if (all_null) {
                // 首个有效值
                last_value = data[idx];
                seen_count++;
                last_seen_count++;
                all_null = false;
            } else if (last_value == data[idx]) {
                // 值相同，增加计数
                last_seen_count++;
            } else {
                // 值不同，刷新当前游程
                if (last_seen_count > 0) {
                    Flush<OP>();
                    seen_count++;
                }
                last_value = data[idx];
                last_seen_count = 1;
            }
        } else {
            // NULL值：仅增加计数（NULL被视为与上一个值相同）
            last_seen_count++;
        }

        // 达到计数上限时强制刷新
        if (last_seen_count == NumericLimits<rle_count_t>::Maximum()) {
            Flush<OP>();
            last_seen_count = 0;
            seen_count++;
        }
    }
};
```

### 1.3 段数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                         RLE压缩段布局                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Header (8 bytes)                                             │  │
│  │ ┌─────────────────────────────────────────────────────────┐ │  │
│  │ │ rle_count_offset: uint64_t                              │ │  │
│  │ │ (计数数组在段中的偏移位置)                               │ │  │
│  │ └─────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Values Array (sizeof(T) × entry_count)                       │  │
│  │ ┌─────┬─────┬─────┬─────┬─────┐                             │  │
│  │ │  V0 │  V1 │  V2 │ ... │  Vn │  每个游程的值               │  │
│  │ └─────┴─────┴─────┴─────┴─────┘                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ [Padding for alignment]                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │ Counts Array (2 bytes × entry_count)                         │  │
│  │ ┌─────┬─────┬─────┬─────┬─────┐                             │  │
│  │ │  C0 │  C1 │  C2 │ ... │  Cn │  每个游程的长度             │  │
│  │ └─────┴─────┴─────┴─────┴─────┘                             │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 1.4 分析阶段

```cpp
template <class T>
struct RLEAnalyzeState : public AnalyzeState {
    explicit RLEAnalyzeState(const CompressionInfo &info)
        : AnalyzeState(info) {}
    RLEState<T> state;
};

template <class T>
unique_ptr<AnalyzeState> RLEInitAnalyze(ColumnData &col_data, PhysicalType type) {
    CompressionInfo info(col_data.GetBlockManager());
    return make_uniq<RLEAnalyzeState<T>>(info);
}

template <class T>
bool RLEAnalyze(AnalyzeState &state, Vector &input, idx_t count) {
    auto &rle_state = state.template Cast<RLEAnalyzeState<T>>();
    UnifiedVectorFormat vdata;
    input.ToUnifiedFormat(count, vdata);

    auto data = UnifiedVectorFormat::GetData<T>(vdata);
    for (idx_t i = 0; i < count; i++) {
        auto idx = vdata.sel->get_index(i);
        rle_state.state.Update(data, vdata.validity, idx);
    }
    return true;  // RLE始终可用
}

template <class T>
idx_t RLEFinalAnalyze(AnalyzeState &state) {
    auto &rle_state = state.template Cast<RLEAnalyzeState<T>>();
    // 评分 = 每个游程的存储大小 × 游程数
    return (sizeof(rle_count_t) + sizeof(T)) * rle_state.state.seen_count;
}
```

### 1.5 压缩阶段

```cpp
template <class T, bool WRITE_STATISTICS>
struct RLECompressState : public CompressionState {
    // RLE写入器
    struct RLEWriter {
        template <class VALUE_TYPE>
        static void Operation(VALUE_TYPE value, rle_count_t count,
                             void *dataptr, bool is_null) {
            auto state = reinterpret_cast<RLECompressState *>(dataptr);
            state->WriteValue(value, count, is_null);
        }
    };

    idx_t MaxRLECount() {
        auto entry_size = sizeof(T) + sizeof(rle_count_t);
        return AlignValueFloor(
            (info.GetBlockSize() - RLE_HEADER_SIZE) / entry_size);
    }

    RLECompressState(ColumnDataCheckpointData &checkpoint_data_p,
                     const CompressionInfo &info)
        : CompressionState(info),
          checkpoint_data(checkpoint_data_p),
          function(checkpoint_data.GetCompressionFunction(
              CompressionType::COMPRESSION_RLE)) {
        CreateEmptySegment();
        state.dataptr = (void *)this;
        max_rle_count = MaxRLECount();
    }

    void WriteValue(T value, rle_count_t count, bool is_null) {
        auto handle_ptr = handle.Ptr() + RLE_HEADER_SIZE;
        auto data_pointer = reinterpret_cast<T *>(handle_ptr);
        auto index_pointer = reinterpret_cast<rle_count_t *>(
            handle_ptr + max_rle_count * sizeof(T));

        // 写入值和计数
        data_pointer[entry_count] = value;
        index_pointer[entry_count] = count;
        entry_count++;

        // 更新统计信息
        if (WRITE_STATISTICS) {
            if (!is_null) {
                current_segment->stats.statistics.SetHasNoNullFast();
                current_segment->stats.statistics.UpdateNumericStats<T>(value);
            } else {
                current_segment->stats.statistics.SetHasNullFast();
            }
        }
        current_segment->count += count;

        // 达到最大条目数时刷新段
        if (entry_count == max_rle_count) {
            FlushSegment();
            CreateEmptySegment();
            entry_count = 0;
        }
    }

    void FlushSegment() {
        // 压缩段：将计数数组移到值数组之后
        idx_t counts_size = sizeof(rle_count_t) * entry_count;
        idx_t original_rle_offset = RLE_HEADER_SIZE + max_rle_count * sizeof(T);
        idx_t minimal_rle_offset = RLE_HEADER_SIZE + sizeof(T) * entry_count;
        idx_t aligned_rle_offset = AlignValue(minimal_rle_offset);
        idx_t total_segment_size = aligned_rle_offset + counts_size;

        auto data_ptr = handle.Ptr();
        memmove(data_ptr + aligned_rle_offset,
                data_ptr + original_rle_offset, counts_size);

        // 存储计数数组偏移
        Store<uint64_t>(aligned_rle_offset, data_ptr);
        handle.Destroy();

        auto &state = checkpoint_data.GetCheckpointState();
        state.FlushSegment(std::move(current_segment), std::move(handle),
                          total_segment_size);
    }
};
```

### 1.6 扫描阶段

```cpp
template <class T>
struct RLEScanState : public SegmentScanState {
    explicit RLEScanState(ColumnSegment &segment) {
        auto &buffer_manager = BufferManager::GetBufferManager(segment.db);
        handle = buffer_manager.Pin(segment.block);
        entry_pos = 0;
        position_in_entry = 0;
        rle_count_offset = Load<uint64_t>(handle.Ptr() + segment.GetBlockOffset());
    }

    void Skip(ColumnSegment &segment, idx_t skip_count) {
        auto data = handle.Ptr() + segment.GetBlockOffset();
        auto index_pointer = reinterpret_cast<rle_count_t *>(
            data + rle_count_offset);

        while (skip_count > 0) {
            rle_count_t run_end = index_pointer[entry_pos];
            idx_t skip_amount = MinValue<idx_t>(
                skip_count, run_end - position_in_entry);

            skip_count -= skip_amount;
            position_in_entry += skip_amount;

            if (position_in_entry >= run_end) {
                entry_pos++;
                position_in_entry = 0;
            }
        }
    }

    BufferHandle handle;
    idx_t entry_pos;
    idx_t position_in_entry;
    uint32_t rle_count_offset;
};

// 常量向量优化
template <class T>
static void RLEScanConstant(RLEScanState<T> &scan_state,
                            rle_count_t *index_pointer,
                            T *data_pointer,
                            idx_t scan_count,
                            Vector &result) {
    // 如果整个向量在同一个游程内，返回常量向量
    result.SetVectorType(VectorType::CONSTANT_VECTOR);
    auto result_data = ConstantVector::GetData<T>(result);
    result_data[0] = data_pointer[scan_state.entry_pos];
    scan_state.position_in_entry += scan_count;
}

template <class T, bool ENTIRE_VECTOR>
void RLEScanPartialInternal(ColumnSegment &segment,
                            ColumnScanState &state,
                            idx_t scan_count,
                            Vector &result,
                            idx_t result_offset) {
    auto &scan_state = state.scan_state->Cast<RLEScanState<T>>();

    auto data = scan_state.handle.Ptr() + segment.GetBlockOffset();
    auto data_pointer = reinterpret_cast<T *>(data + RLE_HEADER_SIZE);
    auto index_pointer = reinterpret_cast<rle_count_t *>(
        data + scan_state.rle_count_offset);

    // 检查是否可以返回常量向量
    if (CanEmitConstantVector<ENTIRE_VECTOR>(
            scan_state.position_in_entry,
            index_pointer[scan_state.entry_pos],
            scan_count)) {
        RLEScanConstant<T>(scan_state, index_pointer, data_pointer,
                          scan_count, result);
        return;
    }

    // 普通扫描：展开RLE编码
    auto result_data = FlatVector::GetData<T>(result);
    result.SetVectorType(VectorType::FLAT_VECTOR);

    idx_t result_end = result_offset + scan_count;
    while (result_offset < result_end) {
        rle_count_t run_end = index_pointer[scan_state.entry_pos];
        idx_t run_count = run_end - scan_state.position_in_entry;
        T element = data_pointer[scan_state.entry_pos];

        idx_t remaining = result_end - result_offset;
        if (run_count > remaining) {
            // 当前游程足够覆盖剩余扫描
            for (idx_t i = 0; i < remaining; i++) {
                result_data[result_offset + i] = element;
            }
            scan_state.position_in_entry += remaining;
            break;
        }

        // 填充当前游程的所有值
        for (idx_t i = 0; i < run_count; i++) {
            result_data[result_offset + i] = element;
        }

        result_offset += run_count;
        scan_state.entry_pos++;
        scan_state.position_in_entry = 0;
    }
}
```

### 1.7 过滤优化

RLE实现了一个重要的过滤优化：对所有游程值预先应用过滤器，避免对每个值重复过滤：

```cpp
template <class T>
void RLEFilter(ColumnSegment &segment, ColumnScanState &state,
               idx_t vector_count, Vector &result,
               SelectionVector &sel, idx_t &sel_count,
               const TableFilter &filter, TableFilterState &filter_state) {
    auto &scan_state = state.scan_state->Cast<RLEScanState<T>>();

    auto data = scan_state.handle.Ptr() + segment.GetBlockOffset();
    auto data_pointer = reinterpret_cast<T *>(data + RLE_HEADER_SIZE);
    auto index_pointer = reinterpret_cast<rle_count_t *>(
        data + scan_state.rle_count_offset);

    auto total_run_count =
        (scan_state.rle_count_offset - RLE_HEADER_SIZE) / sizeof(T);

    if (!scan_state.matching_runs) {
        // 首次过滤：对所有游程值批量应用过滤器
        scan_state.matching_runs =
            make_unsafe_uniq_array<bool>(total_run_count);
        memset(scan_state.matching_runs.get(), 0,
               sizeof(bool) * total_run_count);

        // 创建包含所有游程值的向量
        Vector run_vector(result.GetType(), data_ptr_cast(data_pointer));

        UnifiedVectorFormat run_format;
        run_vector.ToUnifiedFormat(total_run_count, run_format);

        SelectionVector run_matches;
        scan_state.matching_run_count = total_run_count;

        // 批量过滤
        ColumnSegment::FilterSelection(
            run_matches, run_vector, run_format,
            filter, filter_state, total_run_count,
            scan_state.matching_run_count);

        // 记录匹配的游程
        for (idx_t i = 0; i < scan_state.matching_run_count; i++) {
            auto idx = run_matches.get_index(i);
            scan_state.matching_runs[idx] = true;
        }
    }

    if (scan_state.matching_run_count == 0) {
        // 没有游程匹配，直接返回空结果
        sel_count = 0;
        return;
    }

    // 只扫描匹配的游程
    // ...
}
```

## 2. Bitpacking（位压缩）

### 2.1 算法原理

Bitpacking的核心思想是使用最小必要的位宽存储整数值。例如，如果一组值的范围是0-15，只需4位而非完整的32位来存储每个值：

```
原始数据 (32位/值): [3, 7, 2, 15, 8]
位宽计算: max=15, 需要4位
压缩后 (4位/值):   0011 0111 0010 1111 1000
                   ↓
                  20位替代160位
```

### 2.2 五种压缩模式

DuckDB的Bitpacking实现了五种模式，根据数据特征自动选择：

```cpp
// src/include/duckdb/storage/compression/bitpacking.hpp

enum class BitpackingMode : uint8_t {
    INVALID,        // 无效
    AUTO,           // 自动选择（默认）
    CONSTANT,       // 常量模式
    CONSTANT_DELTA, // 等差数列模式
    DELTA_FOR,      // 差分 + 参考帧
    FOR             // 纯参考帧模式
};
```

各模式的适用场景和存储结构：

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Bitpacking五种模式详解                             │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. CONSTANT（常量模式）                                                │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 适用: 所有值相同，或全为NULL                                       │ │
│  │ 存储: [元数据] + [常量值]                                          │ │
│  │ 大小: sizeof(T) + sizeof(metadata)                                │ │
│  │ 示例: [5, 5, 5, 5, 5] -> 存储一次5                                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  2. CONSTANT_DELTA（等差数列模式）                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 适用: 等差数列（每两个相邻值的差相同）                              │ │
│  │ 存储: [元数据] + [首值] + [公差]                                    │ │
│  │ 大小: 2×sizeof(T) + sizeof(metadata)                              │ │
│  │ 示例: [10, 20, 30, 40] -> 存储首值10和公差10                       │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  3. FOR（Frame of Reference，参考帧模式）                               │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 适用: 值范围较小，减去最小值后可用更少位表示                         │ │
│  │ 存储: [元数据] + [最小值] + [位宽] + [位压缩数据]                   │ │
│  │ 原理: 存储 (value - min_value)，需要 log2(max-min) 位              │ │
│  │ 示例: [1000, 1005, 1002] -> min=1000, 存储[0,5,2]用3位            │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  4. DELTA_FOR（差分 + 参考帧模式）                                      │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 适用: 差分后值范围比原值范围更小                                    │ │
│  │ 存储: [元数据] + [最小差值] + [位宽] + [偏移] + [位压缩差值]        │ │
│  │ 原理: 先差分，再FOR压缩差值                                        │ │
│  │ 示例: [100, 102, 105, 107] -> 差值[2,3,2], 存储差值                │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
│  5. AUTO（自动选择）                                                    │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │ 默认模式，根据数据特征自动选择最优模式                              │ │
│  │ 选择顺序:                                                          │ │
│  │   1. 检查是否全相同 -> CONSTANT                                    │ │
│  │   2. 检查是否等差数列 -> CONSTANT_DELTA                            │ │
│  │   3. 比较FOR vs DELTA_FOR的位宽 -> 选择更小的                      │ │
│  └──────────────────────────────────────────────────────────────────┘ │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.3 元数据编码

每个压缩组使用4字节元数据：

```cpp
typedef struct {
    BitpackingMode mode;  // 高8位：压缩模式
    uint32_t offset;      // 低24位：数据在段中的偏移
} bitpacking_metadata_t;

typedef uint32_t bitpacking_metadata_encoded_t;

static bitpacking_metadata_encoded_t EncodeMeta(bitpacking_metadata_t metadata) {
    D_ASSERT(metadata.offset <= 0x00FFFFFF);  // 最大24位
    bitpacking_metadata_encoded_t encoded_value = metadata.offset;
    encoded_value |= (uint8_t)metadata.mode << 24;
    return encoded_value;
}

static bitpacking_metadata_t DecodeMeta(bitpacking_metadata_encoded_t *encoded) {
    bitpacking_metadata_t metadata;
    metadata.mode = static_cast<BitpackingMode>((*encoded >> 24) & 0xFF);
    metadata.offset = *encoded & 0x00FFFFFF;
    return metadata;
}
```

### 2.4 分组机制

Bitpacking按固定大小分组处理数据：

```cpp
static constexpr const idx_t BITPACKING_METADATA_GROUP_SIZE =
    STANDARD_VECTOR_SIZE > 512 ? STANDARD_VECTOR_SIZE : 2048;

template <class T, class T_S = typename MakeSigned<T>::type>
struct BitpackingState {
    // 压缩缓冲区（额外1个位置用于差分编码）
    T compression_buffer_internal[BITPACKING_METADATA_GROUP_SIZE + 1];
    T *compression_buffer;
    T_S delta_buffer[BITPACKING_METADATA_GROUP_SIZE];
    bool compression_buffer_validity[BITPACKING_METADATA_GROUP_SIZE];
    idx_t compression_buffer_idx;
    idx_t total_size;

    // 统计信息
    T minimum;
    T maximum;
    T min_max_diff;
    T_S minimum_delta;
    T_S maximum_delta;
    T_S min_max_delta_diff;
    T_S delta_offset;
    bool all_valid;
    bool all_invalid;
    bool can_do_delta;
    bool can_do_for;

    BitpackingMode mode = BitpackingMode::AUTO;
};
```

### 2.5 模式选择逻辑

```cpp
template <class OP>
bool Flush() {
    if (compression_buffer_idx == 0) {
        return true;
    }

    // 1. 检查CONSTANT模式
    if ((all_invalid || maximum == minimum) &&
        (mode == BitpackingMode::AUTO || mode == BitpackingMode::CONSTANT)) {
        OP::WriteConstant(maximum, compression_buffer_idx, data_ptr, all_invalid);
        total_size += sizeof(T) + sizeof(bitpacking_metadata_encoded_t);
        return true;
    }

    // 计算FOR和Delta统计
    CalculateFORStats();
    CalculateDeltaStats();

    if (can_do_delta) {
        // 2. 检查CONSTANT_DELTA模式
        if (maximum_delta == minimum_delta &&
            mode != BitpackingMode::FOR &&
            mode != BitpackingMode::DELTA_FOR) {
            T frame_of_reference = compression_buffer[0];
            OP::WriteConstantDelta(maximum_delta, frame_of_reference,
                                  compression_buffer_idx, ...);
            total_size += sizeof(T) + sizeof(T) + sizeof(metadata);
            return true;
        }

        // 3. 比较DELTA_FOR vs FOR
        auto delta_bitwidth = BitpackingPrimitives::MinimumBitWidth<T, false>(
            static_cast<T>(min_max_delta_diff));
        auto regular_bitwidth = BitpackingPrimitives::MinimumBitWidth(min_max_diff);

        bool prefer_for = can_do_for && delta_bitwidth >= regular_bitwidth;

        if (!prefer_for && mode != BitpackingMode::FOR) {
            // 使用DELTA_FOR
            SubtractFrameOfReference(delta_buffer, minimum_delta);
            OP::WriteDeltaFor(delta_buffer, ..., delta_bitwidth, ...);
            total_size += sizeof(T) * 3 +
                BitpackingPrimitives::GetRequiredSize(count, delta_bitwidth);
            return true;
        }
    }

    // 4. 使用FOR模式
    if (can_do_for) {
        auto width = BitpackingPrimitives::MinimumBitWidth<T, false>(min_max_diff);
        SubtractFrameOfReference(compression_buffer, minimum);
        OP::WriteFor(compression_buffer, ..., width, minimum, ...);
        total_size += sizeof(T) + AlignValue(sizeof(bitpacking_width_t)) +
            BitpackingPrimitives::GetRequiredSize(count, width);
        return true;
    }

    return false;  // 无法压缩
}
```

### 2.6 FOR计算

```cpp
void CalculateFORStats() {
    // 检查是否可以安全计算 max - min
    can_do_for = TrySubtractOperator::Operation(maximum, minimum, min_max_diff);
}

template <class T_INNER>
void SubtractFrameOfReference(T_INNER *buffer, T_INNER frame_of_reference) {
    using T_U = typename MakeUnsigned<T_INNER>::type;

    for (idx_t i = 0; i < compression_buffer_idx; i++) {
        // 无符号减法避免溢出
        reinterpret_cast<T_U *>(buffer)[i] -= static_cast<T_U>(frame_of_reference);
    }
}
```

### 2.7 差分计算

```cpp
void CalculateDeltaStats() {
    // 限制：值不能超过有符号最大值
    if (maximum > static_cast<T>(NumericLimits<T_S>::Maximum())) {
        return;
    }

    // 至少需要2个值才能差分
    if (compression_buffer_idx < 2) {
        return;
    }

    // 当前不支持带NULL的差分
    if (!all_valid) {
        return;
    }

    // 计算差分值
    // compression_buffer[-1] 是前一组的最后一个值
    for (int64_t i = 0; i < static_cast<int64_t>(compression_buffer_idx); i++) {
        delta_buffer[i] = static_cast<T_S>(compression_buffer[i]) -
                          static_cast<T_S>(compression_buffer[i - 1]);
    }

    can_do_delta = true;

    // 找差分的最大最小值
    for (idx_t i = 1; i < compression_buffer_idx; i++) {
        maximum_delta = MaxValue<T_S>(maximum_delta, delta_buffer[i]);
        minimum_delta = MinValue<T_S>(minimum_delta, delta_buffer[i]);
    }

    // 第一个差分值设为minimum_delta以优化范围
    delta_buffer[0] = minimum_delta;

    can_do_delta = can_do_delta &&
        TrySubtractOperator::Operation(maximum_delta, minimum_delta, min_max_delta_diff);
    can_do_delta = can_do_delta &&
        TrySubtractOperator::Operation(
            static_cast<T_S>(compression_buffer[0]), minimum_delta, delta_offset);
}
```

### 2.8 段数据布局

```
┌────────────────────────────────────────────────────────────────────────┐
│                      Bitpacking压缩段布局                               │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ Header (8 bytes)                                                 │  │
│  │ metadata_offset: idx_t (元数据起始偏移)                          │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ Data Section (向上增长)                                          │  │
│  │ ┌─────────────────────────────────────────────────────────────┐ │  │
│  │ │ Group 0: [FOR值] [位宽] [位压缩数据...]                     │ │  │
│  │ │ Group 1: [常量值] (CONSTANT模式)                            │ │  │
│  │ │ Group 2: [首值] [公差] (CONSTANT_DELTA模式)                 │ │  │
│  │ │ Group 3: [FOR值] [位宽] [偏移] [位压缩数据...] (DELTA_FOR)  │ │  │
│  │ │ ...                                                         │ │  │
│  │ └─────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  [Padding for alignment]                                               │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ Metadata Section (向下增长)                                      │  │
│  │ ┌─────────────────────────────────────────────────────────────┐ │  │
│  │ │ Meta N-1: [mode:8位 | offset:24位]                          │ │  │
│  │ │ Meta N-2: [mode:8位 | offset:24位]                          │ │  │
│  │ │ ...                                                         │ │  │
│  │ │ Meta 0:   [mode:8位 | offset:24位]                          │ │  │
│  │ └─────────────────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### 2.9 位打包原语

DuckDB使用高效的位打包原语：

```cpp
struct BitpackingPrimitives {
    static constexpr idx_t BITPACKING_ALGORITHM_GROUP_SIZE = 32;
    static constexpr idx_t BITPACKING_HEADER_SIZE = sizeof(idx_t);

    // 计算所需的最小位宽
    template <class T, bool SIGNED = true>
    static bitpacking_width_t MinimumBitWidth(T value) {
        // 使用位操作快速计算
        if constexpr (SIGNED) {
            // 有符号需要额外1位
            return MinimumBitWidth<T, false>(
                value < 0 ? ~value : value) + 1;
        } else {
            // 无符号：计算最高位位置
            return sizeof(T) * 8 - CountLeadingZeros(value);
        }
    }

    // 计算存储所需字节数
    static idx_t GetRequiredSize(idx_t count, bitpacking_width_t width) {
        return (count * width + 7) / 8;  // 向上取整到字节
    }

    // 打包缓冲区
    template <class T, bool DELTA = false>
    static void PackBuffer(data_ptr_t dst, T *src, idx_t count,
                          bitpacking_width_t width) {
        // 使用SIMD优化的位打包
        // 按32个值一组处理
        for (idx_t i = 0; i < count; i += BITPACKING_ALGORITHM_GROUP_SIZE) {
            PackGroup(dst, src + i, width);
            dst += (BITPACKING_ALGORITHM_GROUP_SIZE * width) / 8;
        }
    }
};
```

## 3. 算法比较与选择

### 3.1 适用场景对比

| 特性 | RLE | Bitpacking |
|------|-----|------------|
| **最佳场景** | 大量连续重复值 | 值范围有限 |
| **数据特征** | 低基数、排序数据 | 均匀分布、趋势数据 |
| **压缩比** | 重复率高时极佳 | 稳定，取决于值范围 |
| **解压速度** | 常量向量优化快 | SIMD优化快 |
| **NULL处理** | 内嵌在游程中 | 需要单独有效性列 |

### 3.2 评分计算对比

```
RLE评分:
  score = (sizeof(T) + sizeof(rle_count_t)) × 游程数
        = (值大小 + 2) × 游程数

Bitpacking评分:
  score = 所有组的存储大小之和
        = Σ(组元数据 + 组数据)

示例（1000个INT32值）:
  全相同值: RLE = (4+2)×1 = 6字节, Bitpacking = 4+4 = 8字节
  1000个不同值: RLE = (4+2)×1000 = 6000字节
                Bitpacking(0-255范围) = 1000×8/8 + 元数据 ≈ 1020字节
```

### 3.3 选择决策树

```
数据分析
    │
    ├─> 全相同/全NULL? ─yes─> RLE (单游程) 或 Bitpacking CONSTANT
    │
    ├─> 大量连续重复? ─yes─> RLE
    │       (游程数 << 值数量)
    │
    ├─> 等差数列? ─yes─> Bitpacking CONSTANT_DELTA
    │
    ├─> 值范围小? ─yes─> Bitpacking FOR/DELTA_FOR
    │       (max-min < 2^16)
    │
    └─> 其他 ─> 比较RLE和Bitpacking评分，选择更小的
```

## 4. 源码文件索引

| 文件路径 | 内容描述 |
|---------|---------|
| `src/storage/compression/rle.cpp` | RLE完整实现 |
| `src/storage/compression/bitpacking.cpp` | Bitpacking完整实现 |
| `src/storage/compression/bitpacking_hugeint.cpp` | HugeInt特殊处理 |
| `src/include/duckdb/storage/compression/bitpacking.hpp` | Bitpacking模式定义 |
| `src/include/duckdb/common/bitpacking.hpp` | 位打包原语 |

## 小结

本章深入分析了DuckDB的两种整数压缩算法：

1. **RLE**：
   - 使用(值, 计数)对编码连续重复值
   - 支持常量向量优化，单游程时性能极佳
   - 过滤时可批量处理游程，提升效率

2. **Bitpacking**：
   - 五种模式自适应选择
   - CONSTANT/CONSTANT_DELTA用于特殊模式
   - FOR/DELTA_FOR用于值范围压缩
   - 使用SIMD优化的位打包原语

3. **选择策略**：
   - 系统自动计算两种算法的评分
   - 选择压缩后字节数更小的算法
   - 考虑数据的重复模式、值范围、差分特性

下一章将介绍浮点数压缩算法：Chimp、Patas和ALP。
