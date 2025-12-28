# DuckDB压缩系统深度解析 - 第七章：通用压缩与存储引擎集成

## 引言

前面六章详细介绍了DuckDB针对特定数据类型的专用压缩算法。本章将聚焦于三个重要主题：ZSTD通用压缩作为字符串的fallback方案、Constant和Uncompressed等基础压缩策略，以及压缩系统与存储引擎的集成机制。这些内容构成了压缩系统的基础设施层，确保系统在任何数据场景下都能正常工作。

## 7.1 ZSTD通用压缩

### 7.1.1 算法定位

ZSTD（Zstandard）是Facebook开发的通用压缩算法，DuckDB将其作为字符串列的备选压缩方案。与Dictionary、FSST等专用算法相比，ZSTD具有以下特点：

```
压缩算法选择层次：
┌─────────────────────────────────────────┐
│  Dict-FSST (低基数 + 模式重复)          │  ← 最优先
├─────────────────────────────────────────┤
│  Dictionary (低基数)                     │
├─────────────────────────────────────────┤
│  FSST (高基数但有模式)                   │
├─────────────────────────────────────────┤
│  ZSTD (通用压缩)                         │  ← 通用fallback
├─────────────────────────────────────────┤
│  Uncompressed (原始存储)                 │  ← 最终fallback
└─────────────────────────────────────────┘
```

### 7.1.2 使用限制

ZSTD有严格的使用条件：

```cpp
// src/storage/compression/zstd.cpp:142
unique_ptr<AnalyzeState> ZSTDStorage::StringInitAnalyze(ColumnData &col_data, PhysicalType type) {
    auto &storage = col_data.GetStorageManager();
    auto &block_manager = col_data.GetBlockManager();

    if (block_manager.InMemory()) {
        // 内存模式不使用ZSTD
        return nullptr;
    }
    if (storage.GetStorageVersion() < 4) {
        // 存储版本 < 4 时禁用ZSTD（兼容性）
        return nullptr;
    }

    return make_uniq<ZSTDAnalyzeState>(info, config);
}
```

### 7.1.3 数据布局

ZSTD采用向量粒度的压缩策略：

```
Segment 数据布局：
+=====================================================+
|                   Vector Metadata                    |
|  ┌─────────────────────────────────────────────┐    |
|  │  page_id_t page_ids[vector_count]            │    |
|  │  page_offset_t page_offsets[vector_count]    │    |
|  │  uncompressed_size_t uncompressed[vc]        │    |
|  │  compressed_size_t compressed[vc]            │    |
|  └─────────────────────────────────────────────┘    |
+=====================================================+
|                   Vector Data (多个)                 |
|  ┌─────────────────────────────────────────────┐    |
|  │  string_length_t lengths[ZSTD_VECTOR_SIZE]   │    |
|  │  [compressed_data...]                        │    |
|  └─────────────────────────────────────────────┘    |
+=====================================================+

ZSTD_VECTOR_SIZE = max(STANDARD_VECTOR_SIZE, 2048)
```

每个向量独立压缩，便于随机访问和并行解压。

### 7.1.4 评分机制

ZSTD的评分基于字符串平均长度设置惩罚因子：

```cpp
// src/storage/compression/zstd.cpp:193
idx_t ZSTDStorage::StringFinalAnalyze(AnalyzeState &state_p) {
    auto &state = state_p.Cast<ZSTDAnalyzeState>();

    double penalty;
    idx_t average_length = state.total_size / state.count;
    auto threshold = state.config.options.zstd_min_string_length;

    if (average_length >= threshold) {
        penalty = 1.0;  // 长字符串，正常评分
    } else {
        penalty = 1000.0;  // 短字符串，高惩罚
    }

    // 预估压缩后大小（假设50%压缩率）
    auto expected_compressed_size = (double)state.total_size / 2.0;

    idx_t estimated_size = 0;
    estimated_size += LossyNumericCast<idx_t>(expected_compressed_size);
    estimated_size += state.count * sizeof(string_length_t);  // 长度数组
    estimated_size += GetVectorMetadataSize(GetVectorCount(state.count));

    return LossyNumericCast<idx_t>((double)estimated_size * penalty);
}
```

短字符串使用ZSTD效率低下，因此设置1000倍惩罚使其几乎不会被选中。

### 7.1.5 流式压缩

ZSTD使用流式API处理大数据：

```cpp
// src/storage/compression/zstd.cpp:391
void ZSTDCompressionState::CompressString(const string_t &string, bool end_of_vector) {
    duckdb_zstd::ZSTD_inBuffer in_buffer = {
        string.GetData(),
        size_t(string.GetSize()),
        0  // pos
    };

    const auto end_mode = end_of_vector ?
        duckdb_zstd::ZSTD_e_end :     // 刷新并结束帧
        duckdb_zstd::ZSTD_e_continue; // 继续压缩

    while (true) {
        auto compress_result = duckdb_zstd::ZSTD_compressStream2(
            analyze_state->context,
            &out_buffer,
            &in_buffer,
            end_mode
        );

        if (compress_result == 0) {
            break;  // 压缩完成
        }

        // 输出缓冲区满，切换到新页面
        NewPage();
    }
}
```

### 7.1.6 跨页存储

压缩数据可能跨越多个页面，通过链表结构组织：

```
页面链表结构：
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Page 1      │    │ Page 2      │    │ Page 3      │
│ ┌─────────┐ │    │ ┌─────────┐ │    │ ┌─────────┐ │
│ │ Data    │ │    │ │ Data    │ │    │ │ Data    │ │
│ │  ...    │ │    │ │  ...    │ │    │ │  ...    │ │
│ ├─────────┤ │    │ ├─────────┤ │    │ ├─────────┤ │
│ │next_id=2│─┼────┼→│next_id=3│─┼────┼→│next_id=-1│ │
│ └─────────┘ │    │ └─────────┘ │    │ └─────────┘ │
└─────────────┘    └─────────────┘    └─────────────┘
```

解压时根据元数据定位到起始页面，然后顺序读取。

## 7.2 Numeric Constant压缩

### 7.2.1 设计理念

当一列中所有值相同时（常量列），无需存储实际数据，只需记录该常量值即可。DuckDB通过统计信息自动检测这种情况：

```cpp
// src/storage/compression/numeric_constant.cpp:33
template <class T>
void ConstantFillFunction(ColumnSegment &segment, Vector &result,
                          idx_t start_idx, idx_t count) {
    auto &nstats = segment.stats.statistics;
    auto data = FlatVector::GetData<T>(result);

    // 从统计信息中获取常量值（min == max）
    auto constant_value = NumericStats::GetMin<T>(nstats);

    for (idx_t i = 0; i < count; i++) {
        data[start_idx + i] = constant_value;
    }
}
```

### 7.2.2 支持的类型

Constant压缩支持所有定长数值类型：

```cpp
// src/storage/compression/numeric_constant.cpp:223
CompressionFunction ConstantFun::GetFunction(PhysicalType data_type) {
    switch (data_type) {
    case PhysicalType::BIT:
        return ConstantGetFunctionValidity(data_type);
    case PhysicalType::BOOL:
    case PhysicalType::INT8:
        return ConstantGetFunction<int8_t>(data_type);
    case PhysicalType::INT16:
        return ConstantGetFunction<int16_t>(data_type);
    // ... INT32, INT64, UINT8-UINT64
    case PhysicalType::INT128:
        return ConstantGetFunction<hugeint_t>(data_type);
    case PhysicalType::FLOAT:
        return ConstantGetFunction<float>(data_type);
    case PhysicalType::DOUBLE:
        return ConstantGetFunction<double>(data_type);
    default:
        throw InternalException("Unsupported type for ConstantUncompressed");
    }
}
```

### 7.2.3 扫描优化

常量列扫描时可以直接返回CONSTANT_VECTOR，避免逐元素填充：

```cpp
// src/storage/compression/numeric_constant.cpp:71
template <class T>
void ConstantScanFunction(ColumnSegment &segment, ColumnScanState &state,
                          idx_t scan_count, Vector &result) {
    auto &nstats = segment.stats.statistics;
    auto data = FlatVector::GetData<T>(result);

    // 只设置第一个元素
    data[0] = NumericStats::GetMin<T>(nstats);
    // 标记为常量向量，无需填充其他元素
    result.SetVectorType(VectorType::CONSTANT_VECTOR);
}
```

### 7.2.4 有效性常量优化

对于全NULL或全非NULL的有效性列，同样适用常量优化：

```cpp
// src/storage/compression/numeric_constant.cpp:58
void ConstantScanFunctionValidity(ColumnSegment &segment, ColumnScanState &state,
                                  idx_t scan_count, Vector &result) {
    auto &stats = segment.stats.statistics;

    if (stats.CanHaveNull()) {
        // 全部为NULL
        if (result.GetVectorType() == VectorType::CONSTANT_VECTOR) {
            ConstantVector::SetNull(result, true);
        } else {
            result.Flatten(scan_count);
            // 设置所有位为无效
            auto &mask = FlatVector::Validity(result);
            for (idx_t i = 0; i < scan_count; i++) {
                mask.SetInvalid(i);
            }
        }
    }
    // 否则全部有效，无需任何操作
}
```

### 7.2.5 过滤器优化

常量列配合过滤器可以直接返回结果，无需实际扫描：

```cpp
// src/storage/compression/numeric_constant.cpp:177
void ConstantFilterValidity(ColumnSegment &segment, ColumnScanState &state,
                            idx_t vector_count, Vector &result,
                            SelectionVector &sel, idx_t &sel_count,
                            const TableFilter &filter, TableFilterState &filter_state) {
    bool filters_nulls, filters_valid_values;
    ConstantFun::FiltersNullValues(result.GetType(), filter,
                                   filters_nulls, filters_valid_values, filter_state);

    auto &stats = segment.stats.statistics;
    if (stats.CanHaveNull()) {
        // 全是NULL
        if (filters_nulls) {
            sel_count = 0;  // 过滤器排除NULL，结果为空
            return;
        }
    } else {
        // 全是有效值
        if (filters_valid_values) {
            sel_count = 0;  // 过滤器排除有效值，结果为空
            return;
        }
    }
    // 否则正常扫描
    ConstantScanFunctionValidity(segment, state, vector_count, result);
}
```

## 7.3 Uncompressed压缩策略

Uncompressed（不压缩）是最基础的存储策略，作为所有压缩算法的最终fallback。DuckDB根据数据类型实现了三种Uncompressed变体：

```cpp
// src/storage/compression/uncompressed.cpp:6
CompressionFunction UncompressedFun::GetFunction(PhysicalType type) {
    switch (type) {
    case PhysicalType::BOOL:
    case PhysicalType::INT8:
    // ... 所有定长类型
    case PhysicalType::INTERVAL:
        return FixedSizeUncompressed::GetFunction(type);

    case PhysicalType::BIT:
        return ValidityUncompressed::GetFunction(type);

    case PhysicalType::VARCHAR:
        return StringUncompressed::GetFunction(type);

    default:
        throw InternalException("Unsupported type for Uncompressed");
    }
}
```

### 7.3.1 Fixed-Size Uncompressed

定长类型直接连续存储：

```
内存布局（以INT32为例）：
┌──────┬──────┬──────┬──────┬──────┬─────┐
│ val0 │ val1 │ val2 │ val3 │ val4 │ ... │
│ 4B   │ 4B   │ 4B   │ 4B   │ 4B   │     │
└──────┴──────┴──────┴──────┴──────┴─────┘
```

**扫描实现：**

```cpp
// src/storage/compression/fixed_size_uncompressed.cpp:169
template <class T>
void FixedSizeScan(ColumnSegment &segment, ColumnScanState &state,
                   idx_t scan_count, Vector &result) {
    auto &scan_state = state.scan_state->Cast<FixedSizeScanState>();
    auto start = state.GetPositionInSegment();

    auto data = scan_state.handle.Ptr() + segment.GetBlockOffset();
    auto source_data = data + start * sizeof(T);

    result.SetVectorType(VectorType::FLAT_VECTOR);
    // 直接设置数据指针，零拷贝
    FlatVector::SetData(result, source_data);
}
```

**追加实现：**

```cpp
// src/storage/compression/fixed_size_uncompressed.cpp:253
template <class T, class OP>
idx_t FixedSizeAppend(CompressionAppendState &append_state, ColumnSegment &segment,
                      SegmentStatistics &stats, UnifiedVectorFormat &data,
                      idx_t offset, idx_t count) {
    auto target_ptr = append_state.handle.Ptr();
    idx_t max_tuple_count = segment.SegmentSize() / sizeof(T);
    idx_t copy_count = MinValue<idx_t>(count, max_tuple_count - segment.count);

    // 调用特定类型的追加逻辑
    OP::template Append<T>(stats, target_ptr, segment.count, data, offset, copy_count);
    segment.count += copy_count;
    return copy_count;
}
```

**NULL值处理：**

```cpp
// src/storage/compression/fixed_size_uncompressed.cpp:205
struct StandardFixedSizeAppend {
    template <class T>
    static void Append(SegmentStatistics &stats, data_ptr_t target, idx_t target_offset,
                       UnifiedVectorFormat &adata, idx_t offset, idx_t count) {
        auto sdata = UnifiedVectorFormat::GetData<T>(adata);
        auto tdata = reinterpret_cast<T *>(target);

        if (!adata.validity.AllValid()) {
            for (idx_t i = 0; i < count; i++) {
                auto source_idx = adata.sel->get_index(offset + i);
                auto target_idx = target_offset + i;
                bool is_null = !adata.validity.RowIsValid(source_idx);
                if (!is_null) {
                    tdata[target_idx] = sdata[source_idx];
                    stats.statistics.UpdateNumericStats<T>(sdata[source_idx]);
                } else {
                    // NULL位置写入NullValue（仅用于调试）
                    tdata[target_idx] = NullValue<T>();
                }
            }
        } else {
            // 无NULL值，直接复制
            for (idx_t i = 0; i < count; i++) {
                auto source_idx = adata.sel->get_index(offset + i);
                auto target_idx = target_offset + i;
                tdata[target_idx] = sdata[source_idx];
            }
        }
    }
};
```

### 7.3.2 String Uncompressed

字符串的Uncompressed存储采用字典风格布局：

```
String Uncompressed 布局：
┌────────────────────────────────────────────────────────┐
│                    Segment Buffer                       │
│  ┌──────────────────────────────────────────────────┐  │
│  │ Dictionary Header (8 bytes)                       │  │
│  │  ├─ size: uint32_t (字典总大小)                   │  │
│  │  └─ end: uint32_t (字典结束位置)                  │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ Offset Array (int32_t[])                          │  │
│  │  累积偏移量，负值表示大字符串                      │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ (空闲区域)                                        │  │
│  ├──────────────────────────────────────────────────┤  │
│  │ String Data (从末尾向前生长)                      │  │
│  │  ├─ "world"                                       │  │
│  │  ├─ "hello"                                       │  │
│  │  └─ ...                                           │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**大字符串处理：**

超过阈值的字符串存储到溢出块：

```cpp
// src/storage/compression/string_uncompressed.cpp:317
void UncompressedStringStorage::WriteString(ColumnSegment &segment, string_t string,
                                            block_id_t &result_block, int32_t &result_offset) {
    auto &state = segment.GetSegmentState()->Cast<UncompressedStringSegmentState>();

    if (state.overflow_writer) {
        // 有磁盘写入器，写入溢出块
        state.overflow_writer->WriteString(state, string, result_block, result_offset);
    } else {
        // 内存模式，分配内存块
        WriteStringMemory(segment, string, result_block, result_offset);
    }
}
```

溢出块阈值：

```cpp
static constexpr idx_t BIG_STRING_MARKER_SIZE = sizeof(block_id_t) + sizeof(int32_t);

// 字符串长度 >= 块大小 - 一些保留空间 时触发溢出
idx_t GetStringBlockLimit(idx_t block_size) {
    return block_size - BIG_STRING_MARKER_SIZE - sizeof(uint32_t);
}
```

**扫描实现：**

```cpp
// src/storage/compression/string_uncompressed.cpp:91
void UncompressedStringStorage::StringScanPartial(ColumnSegment &segment,
                                                  ColumnScanState &state,
                                                  idx_t scan_count, Vector &result,
                                                  idx_t result_offset) {
    auto &scan_state = state.scan_state->Cast<StringScanState>();
    auto start = state.GetPositionInSegment();

    auto baseptr = scan_state.handle.Ptr() + segment.GetBlockOffset();
    auto dict_end = GetDictionaryEnd(segment, scan_state.handle);
    auto base_data = reinterpret_cast<int32_t *>(baseptr + DICTIONARY_HEADER_SIZE);
    auto result_data = FlatVector::GetData<string_t>(result);

    int32_t previous_offset = start > 0 ? base_data[start - 1] : 0;

    for (idx_t i = 0; i < scan_count; i++) {
        auto current_offset = base_data[start + i];
        // 使用std::abs因为负偏移表示大字符串
        auto string_length = std::abs(current_offset) - std::abs(previous_offset);

        result_data[result_offset + i] = FetchStringFromDict(
            segment, dict_end, result, baseptr, current_offset, string_length);
        previous_offset = base_data[start + i];
    }
}
```

### 7.3.3 Validity Uncompressed

有效性位图采用位打包存储：

```
Validity 布局：
┌───────────────────────────────────────┐
│ 64-bit validity entries               │
│ ┌───┬───┬───┬───┬───┬───┬───┬───┐    │
│ │v0 │v1 │...│v63│v64│...│...│...│    │
│ └───┴───┴───┴───┴───┴───┴───┴───┘    │
│  ↑ 每位表示一个值是否有效              │
└───────────────────────────────────────┘

位图优化：使用LOWER_MASKS和UPPER_MASKS加速位操作
```

**掩码常量表：**

```cpp
// src/storage/compression/validity_uncompressed.cpp:30
// LOWER_MASKS[n] = 最低n位全为1
const validity_t ValidityUncompressed::LOWER_MASKS[] = {
    0x0,              // 0位
    0x1,              // 1位
    0x3,              // 2位
    // ...
    0xffffffffffffffff // 64位
};

// UPPER_MASKS[n] = 最高n位全为1
const validity_t ValidityUncompressed::UPPER_MASKS[] = {
    0x0,                    // 0位
    0x8000000000000000,     // 1位
    0xc000000000000000,     // 2位
    // ...
    0xffffffffffffffff      // 64位
};
```

**对齐扫描优化：**

```cpp
// src/storage/compression/validity_uncompressed.cpp:374
void ValidityUncompressed::AlignedScan(data_ptr_t input, idx_t input_start,
                                       Vector &result, idx_t scan_count) {
    D_ASSERT(input_start % ValidityMask::BITS_PER_VALUE == 0);

    auto &result_mask = FlatVector::Validity(result);
    auto input_data = reinterpret_cast<validity_t *>(input);
    auto result_data = result_mask.GetData();

    idx_t start_offset = input_start / ValidityMask::BITS_PER_VALUE;
    idx_t entry_scan_count = (scan_count + 63) / 64;

    for (idx_t i = 0; i < entry_scan_count; i++) {
        auto input_entry = input_data[start_offset + i];
        if (!result_data && input_entry == ValidityMask::ValidityBuffer::MAX_ENTRY) {
            continue;  // 全有效，跳过
        }
        if (!result_data) {
            result_mask.Initialize();
            result_data = result_mask.GetData();
        }
        result_data[i] = input_entry;
    }
}
```

**非对齐扫描：**

当起始位置不对齐时，需要进行位移操作：

```cpp
// src/storage/compression/validity_uncompressed.cpp:222
void ValidityUncompressed::UnalignedScan(data_ptr_t input, idx_t input_size,
                                         idx_t input_start, Vector &result,
                                         idx_t result_offset, idx_t scan_count) {
    auto &result_mask = FlatVector::Validity(result);
    auto input_data = reinterpret_cast<validity_t *>(input);

    idx_t result_entry = result_offset / ValidityMask::BITS_PER_VALUE;
    idx_t result_idx = result_offset % ValidityMask::BITS_PER_VALUE;
    idx_t input_entry = input_start / ValidityMask::BITS_PER_VALUE;
    idx_t input_idx = input_start % ValidityMask::BITS_PER_VALUE;

    idx_t pos = 0;
    while (pos < scan_count) {
        validity_t input_mask = input_data[input_entry];

        if (result_idx < input_idx) {
            // 输入需要右移对齐
            auto shift = input_idx - result_idx;
            input_mask = input_mask >> shift;
            input_mask |= UPPER_MASKS[shift];  // 保护高位
        } else if (result_idx > input_idx) {
            // 输入需要左移对齐
            auto shift = result_idx - input_idx;
            input_mask = (input_mask & ~UPPER_MASKS[shift]) << shift;
            input_mask |= LOWER_MASKS[shift];  // 保护低位
        }

        // 合并到结果
        if (input_mask != ValidityMask::ValidityBuffer::MAX_ENTRY) {
            if (!result_data) {
                result_mask.Initialize();
                result_data = result_mask.GetData();
            }
            result_data[result_entry] &= input_mask;
        }

        // 移动到下一个entry
        // ...
    }
}
```

## 7.4 与存储引擎的集成

### 7.4.1 ColumnSegment架构

ColumnSegment是压缩数据的载体：

```cpp
// src/include/duckdb/storage/table/column_segment.hpp:41
class ColumnSegment : public SegmentBase<ColumnSegment> {
public:
    DatabaseInstance &db;
    LogicalType type;
    idx_t type_size;
    ColumnSegmentType segment_type;  // TRANSIENT or PERSISTENT
    SegmentStatistics stats;
    shared_ptr<BlockHandle> block;

private:
    reference<CompressionFunction> function;  // 压缩函数引用
    block_id_t block_id;
    idx_t offset;
    idx_t segment_size;
    unique_ptr<CompressedSegmentState> segment_state;  // 压缩状态
};
```

**创建持久化段：**

```cpp
// ColumnSegment::CreatePersistentSegment
static unique_ptr<ColumnSegment> CreatePersistentSegment(
    DatabaseInstance &db,
    BlockManager &block_manager,
    block_id_t id,
    idx_t offset,
    const LogicalType &type_p,
    idx_t count,
    CompressionType compression_type,
    BaseStatistics statistics,
    unique_ptr<ColumnSegmentState> segment_state);
```

**创建临时段：**

```cpp
// ColumnSegment::CreateTransientSegment
static unique_ptr<ColumnSegment> CreateTransientSegment(
    DatabaseInstance &db,
    CompressionFunction &function,
    const LogicalType &type,
    const idx_t segment_size,
    BlockManager &block_manager);
```

### 7.4.2 Checkpoint流程

数据持久化时触发压缩选择：

```
Checkpoint 压缩流程：
┌─────────────────────────────────────────────────────────┐
│                ColumnDataCheckpointer                    │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 1. InitAnalyze()                                   │  │
│  │    为每个候选压缩函数创建分析状态                   │  │
│  └───────────────────────────────────────────────────┘  │
│                          ↓                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 2. ScanSegments()                                  │  │
│  │    扫描所有数据段，调用各压缩函数的analyze         │  │
│  └───────────────────────────────────────────────────┘  │
│                          ↓                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 3. DetectBestCompressionMethod()                   │  │
│  │    调用final_analyze获取评分，选择最优算法         │  │
│  └───────────────────────────────────────────────────┘  │
│                          ↓                               │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 4. WriteToDisk()                                   │  │
│  │    使用选中的压缩函数压缩数据并写入磁盘            │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

**压缩函数选择：**

```cpp
// src/storage/table/column_data_checkpointer.cpp:160
vector<CheckpointAnalyzeResult> ColumnDataCheckpointer::DetectBestCompressionMethod() {
    // 检查是否有强制压缩配置
    auto compression_type = checkpoint_info.GetCompressionType();
    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &functions = compression_functions[i];
        if (compression_type != CompressionType::COMPRESSION_AUTO) {
            forced_methods[i] = ForceCompression(storage_manager, functions, compression_type);
        }
    }

    // 初始化分析状态
    InitAnalyze();

    // 扫描所有段并运行分析
    ScanSegments([&](Vector &scan_vector, idx_t count) {
        for (idx_t i = 0; i < checkpoint_states.size(); i++) {
            auto &functions = compression_functions[i];
            auto &states = analyze_states[i];

            for (idx_t j = 0; j < functions.size(); j++) {
                auto &state = states[j];
                auto &func = functions[j];

                if (!state) continue;

                // 分析失败则移除该候选
                if (!func->analyze(*state, scan_vector, count)) {
                    state = nullptr;
                    func = nullptr;
                }
            }
        }
    });

    // 选择评分最低（最优）的压缩方法
    // ...
}
```

### 7.4.3 扫描与解压

压缩数据的扫描遵循延迟解压原则：

```cpp
// ColumnSegment::Scan 简化流程
void ColumnSegment::Scan(ColumnScanState &state, idx_t scan_count,
                         Vector &result, idx_t result_offset,
                         ScanVectorType scan_type) {
    // 根据扫描类型选择方法
    if (scan_type == ScanVectorType::SCAN_FLAT_VECTOR) {
        // 调用压缩函数的scan
        function.get().scan(*state.scan_state, segment, state, scan_count, result);
    } else {
        // 部分扫描
        function.get().scan_partial(*state.scan_state, segment, state,
                                    scan_count, result, result_offset);
    }
}
```

**跳过优化：**

```cpp
// ColumnSegment::Skip
void ColumnSegment::Skip(ColumnScanState &state) {
    if (function.get().skip) {
        // 使用压缩函数特定的跳过逻辑
        function.get().skip(*this, state, state.offset_in_segment);
    }
    // 更新扫描位置
}
```

### 7.4.4 段状态序列化

持久化时需要保存压缩相关的元数据：

```cpp
// 序列化接口
unique_ptr<ColumnSegmentState> SerializeState(ColumnSegment &segment);

// 反序列化接口
unique_ptr<ColumnSegmentState> DeserializeState(Deserializer &deserializer);

// 示例：字符串溢出块序列化
void SerializedStringSegmentState::Serialize(Serializer &serializer) const {
    serializer.WriteProperty(1, "overflow_blocks", blocks);
}
```

### 7.4.5 块访问管理

压缩系统与BufferManager协作管理内存：

```cpp
// 扫描初始化时固定块
unique_ptr<SegmentScanState> FixedSizeInitScan(const QueryContext &context,
                                               ColumnSegment &segment) {
    auto result = make_uniq<FixedSizeScanState>();
    auto &buffer_manager = BufferManager::GetBufferManager(segment.db);

    // Pin块到内存
    result->handle = buffer_manager.Pin(context, segment.block);
    return std::move(result);
}

// 扫描完成后自动释放（RAII）
```

## 7.5 压缩系统设计总结

### 7.5.1 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│                      Application Layer                       │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │
│   │ SELECT  │  │ INSERT  │  │ UPDATE  │  │ COPY    │       │
│   └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘       │
└────────┼────────────┼────────────┼────────────┼─────────────┘
         │            │            │            │
┌────────▼────────────▼────────────▼────────────▼─────────────┐
│                     Storage Engine Layer                     │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                 ColumnDataCheckpointer               │   │
│   │  (压缩选择、扫描、写入协调)                          │   │
│   └─────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                    Compression Layer                         │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              CompressionFunctionSet                  │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐   │   │
│   │  │Bitpack  │ │ RLE     │ │Dict-FSST│ │ ZSTD    │   │   │
│   │  │ALP      │ │ ALPRD   │ │ Roaring │ │Constant │   │   │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────┐
│                      Block Layer                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │  BlockManager  +  BufferManager  +  ColumnSegment   │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### 7.5.2 类型到算法映射总结

| 物理类型 | 主要压缩算法 | 备选算法 | Fallback |
|---------|-------------|---------|----------|
| INT8-INT128 | Bitpacking, RLE | Constant | Uncompressed |
| FLOAT/DOUBLE | ALP, ALPRD | Bitpacking, Constant | Uncompressed |
| VARCHAR | Dict-FSST, Dictionary, FSST | ZSTD | Uncompressed |
| BIT (Validity) | Roaring, Empty | Constant | Uncompressed |
| LIST | - | - | Uncompressed |
| INTERVAL | - | - | Uncompressed |

### 7.5.3 设计原则回顾

1. **自适应选择**：基于数据特征自动选择最优算法
2. **类型特化**：针对不同类型实现专用算法
3. **Fallback保证**：Uncompressed确保任何情况都能工作
4. **延迟解压**：扫描时按需解压，减少内存开销
5. **零拷贝优化**：定长类型可直接返回数据指针
6. **向量化友好**：所有算法支持批量操作
7. **存储版本兼容**：新算法受版本控制，确保向后兼容

## 源码文件索引

| 文件 | 核心内容 |
|-----|---------|
| `src/storage/compression/zstd.cpp` | ZSTD流式压缩实现 |
| `src/storage/compression/numeric_constant.cpp` | 常量列优化 |
| `src/storage/compression/fixed_size_uncompressed.cpp` | 定长类型存储 |
| `src/storage/compression/string_uncompressed.cpp` | 字符串存储 |
| `src/storage/compression/validity_uncompressed.cpp` | 有效性位图 |
| `src/storage/compression/uncompressed.cpp` | Uncompressed调度 |
| `src/storage/table/column_segment.cpp` | 段管理 |
| `src/storage/table/column_data_checkpointer.cpp` | 压缩选择流程 |

## 系列总结

通过七章内容，我们完整剖析了DuckDB压缩系统：

1. **架构层面**：三阶段模型、CompressionFunction接口、函数注册机制
2. **选择机制**：评分系统、强制压缩、存储版本感知
3. **整数压缩**：Bitpacking五种模式、RLE游程编码
4. **浮点压缩**：Chimp XOR编码、ALP/ALPRD自适应无损压缩
5. **字符串压缩**：Dictionary、FSST、Dict-FSST三级方案
6. **位图压缩**：Roaring三种容器类型、Empty Validity优化
7. **通用策略**：ZSTD、Constant、Uncompressed基础设施

DuckDB的压缩系统体现了分析型数据库的核心优化思想：**理解数据特征，针对性地选择最高效的存储方式**。这种自适应设计使得DuckDB能够在保持高压缩比的同时，维持出色的查询性能。
