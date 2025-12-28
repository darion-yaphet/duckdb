# 第九章：行数据与列数据集合

## 9.1 数据集合概述

DuckDB提供了两种主要的数据集合抽象，分别针对行存储和列存储场景优化：

1. **ColumnDataCollection**：列式存储集合，适合高效扫描和分析处理
2. **TupleDataCollection**：行式存储集合，适合哈希表、排序等需要随机访问的场景

```
┌─────────────────────────────────────────────────────────────────┐
│                     数据集合体系结构                              │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ColumnDataCollection │    │ TupleDataCollection │             │
│  │    (列式存储)        │    │    (行式存储)        │             │
│  └─────────┬───────────┘    └─────────┬───────────┘             │
│            │                          │                         │
│            ▼                          ▼                         │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ColumnDataSegment    │    │ TupleDataSegment    │             │
│  │  - ChunkMetaData    │    │  - TupleDataChunk   │             │
│  │  - VectorMetaData   │    │  - ChunkPart        │             │
│  └─────────────────────┘    └─────────────────────┘             │
│            │                          │                         │
│            ▼                          ▼                         │
│  ┌─────────────────────┐    ┌─────────────────────┐             │
│  │ ColumnDataAllocator │    │ TupleDataAllocator  │             │
│  │  - BufferManager    │    │  - BufferManager    │             │
│  │  - InMemory         │    │  - Block Management │             │
│  └─────────────────────┘    └─────────────────────┘             │
└─────────────────────────────────────────────────────────────────┘
```

## 9.2 ColumnDataCollection

### 9.2.1 类定义

```cpp
// src/include/duckdb/common/types/column/column_data_collection.hpp

class ColumnDataCollection {
public:
    // 构造函数
    ColumnDataCollection(Allocator &allocator, vector<LogicalType> types);
    ColumnDataCollection(BufferManager &buffer_manager, vector<LogicalType> types,
                         ColumnDataCollectionLifetime lifetime);
    ColumnDataCollection(ClientContext &context, vector<LogicalType> types,
                         ColumnDataAllocatorType type, ColumnDataCollectionLifetime lifetime);

    // 基本属性
    vector<LogicalType> &Types();
    const idx_t &Count() const;
    idx_t ColumnCount() const;
    idx_t SizeInBytes() const;

    // 追加操作
    void InitializeAppend(ColumnDataAppendState &state);
    void Append(ColumnDataAppendState &state, DataChunk &new_chunk);
    void Append(DataChunk &new_chunk);

    // 扫描操作
    void InitializeScan(ColumnDataScanState &state, ColumnDataScanProperties properties);
    void InitializeScan(ColumnDataScanState &state, vector<column_t> column_ids,
                        ColumnDataScanProperties properties);
    bool Scan(ColumnDataScanState &state, DataChunk &result) const;

    // 并行扫描
    void InitializeScan(ColumnDataParallelScanState &state, ColumnDataScanProperties properties);
    bool Scan(ColumnDataParallelScanState &state, ColumnDataLocalScanState &lstate,
              DataChunk &result) const;

    // 合并操作
    void Combine(ColumnDataCollection &other);

    // 迭代器
    ColumnDataChunkIterationHelper Chunks() const;
    ColumnDataRowIterationHelper Rows() const;
    ColumnDataRowCollection GetRows() const;

private:
    buffer_ptr<ColumnDataAllocator> allocator;
    vector<LogicalType> types;
    idx_t count;
    vector<unique_ptr<ColumnDataCollectionSegment>> segments;
    vector<ColumnDataCopyFunction> copy_functions;
    bool finished_append;
};
```

### 9.2.2 列数据段结构

```cpp
// src/include/duckdb/common/types/column/column_data_collection_segment.hpp

// 向量元数据
struct VectorMetaData {
    uint32_t block_id;      // 数据块ID
    uint32_t offset;        // 块内偏移
    uint16_t count;         // 条目数
    vector<SwizzleMetaData> swizzle_data;  // 字符串指针元数据

    VectorChildIndex child_index;  // 子向量索引（LIST/STRUCT）
    VectorDataIndex next_data;     // 下一个向量（LIST子数据）
};

// 块元数据
struct ChunkMetaData {
    vector<VectorDataIndex> vector_data;  // 向量数据索引
    unordered_set<uint32_t> block_ids;    // 引用的块ID
    uint16_t count;                        // 条目数
};

// 列数据段
class ColumnDataCollectionSegment {
public:
    shared_ptr<ColumnDataAllocator> allocator;
    vector<LogicalType> types;
    idx_t count;
    vector<ChunkMetaData> chunk_data;      // 块元数据
    vector<VectorMetaData> vector_data;    // 向量元数据
    vector<VectorDataIndex> child_indices; // 子索引
    shared_ptr<StringHeap> heap;           // 字符串堆

public:
    VectorDataIndex AllocateVector(const LogicalType &type, ChunkMetaData &chunk_data,
                                   ColumnDataAppendState &append_state);
    void ReadChunk(idx_t chunk_index, ChunkManagementState &state, DataChunk &chunk,
                   const vector<column_t> &column_ids);
    idx_t ReadVector(ChunkManagementState &state, VectorDataIndex vector_index,
                     Vector &result);
};
```

### 9.2.3 内存布局

```
ColumnDataCollection
┌─────────────────────────────────────────────────────────────────┐
│  Segment 0                                                      │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Chunk 0                                                 │   │
│  │  ┌──────────┬──────────┬──────────┬──────────┐           │   │
│  │  │ Vector 0 │ Vector 1 │ Vector 2 │   ...    │           │   │
│  │  │ (col 0)  │ (col 1)  │ (col 2)  │          │           │   │
│  │  └──────────┴──────────┴──────────┴──────────┘           │   │
│  ├──────────────────────────────────────────────────────────┤   │
│  │  Chunk 1                                                 │   │
│  │  ┌──────────┬──────────┬──────────┬──────────┐           │   │
│  │  │ Vector 0 │ Vector 1 │ Vector 2 │   ...    │           │   │
│  │  └──────────┴──────────┴──────────┴──────────┘           │   │
│  └──────────────────────────────────────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│  Segment 1                                                      │
│  ...                                                            │
└─────────────────────────────────────────────────────────────────┘
```

### 9.2.4 追加操作

```cpp
void ColumnDataCollection::Append(ColumnDataAppendState &state, DataChunk &new_chunk) {
    if (new_chunk.size() == 0) {
        return;
    }

    auto &segment = *segments.back();
    auto &chunk_data = segment.chunk_data.back();

    // 获取每列的追加位置
    for (idx_t col_idx = 0; col_idx < types.size(); col_idx++) {
        auto &vector_data = state.current_chunk_state.vector_data[col_idx];
        auto &target = segment.vector_data[vector_data.index];

        // 复制数据到目标位置
        auto &copy_func = copy_functions[col_idx];
        copy_func.function(state, segment, new_chunk.data[col_idx], target, col_idx);
    }

    // 更新计数
    chunk_data.count += new_chunk.size();
    segment.count += new_chunk.size();
    count += new_chunk.size();
}
```

### 9.2.5 扫描操作

```cpp
bool ColumnDataCollection::Scan(ColumnDataScanState &state, DataChunk &result) const {
    idx_t chunk_index, segment_index, row_index;
    if (!NextScanIndex(state, chunk_index, segment_index, row_index)) {
        return false;
    }

    auto &segment = *segments[segment_index];
    auto &chunk_data = segment.chunk_data[chunk_index];

    // 读取每列数据
    for (idx_t col_idx = 0; col_idx < state.column_ids.size(); col_idx++) {
        auto column_id = state.column_ids[col_idx];
        auto vector_index = chunk_data.vector_data[column_id];
        segment.ReadVector(state.chunk_state, vector_index, result.data[col_idx]);
    }

    result.SetCardinality(chunk_data.count);
    return true;
}
```

### 9.2.6 零拷贝扫描

ColumnDataCollection支持零拷贝扫描，直接引用底层数据而不复制：

```cpp
enum class ColumnDataScanProperties {
    ALLOW_ZERO_COPY,      // 允许零拷贝（直接引用）
    DISALLOW_ZERO_COPY    // 禁止零拷贝（必须复制）
};

// 零拷贝实现
idx_t ColumnDataCollectionSegment::ReadVector(ChunkManagementState &state,
                                               VectorDataIndex vector_index,
                                               Vector &result) {
    auto &vector_meta = vector_data[vector_index.index];

    // 获取数据指针
    auto handle = allocator->Pin(vector_meta.block_id);
    auto base_ptr = handle.Ptr() + vector_meta.offset;

    // 直接引用底层数据
    if (state.properties.AllowZeroCopy()) {
        result.Reference(base_ptr, vector_meta.count);
    } else {
        // 复制数据
        VectorOperations::Copy(source, result, vector_meta.count, 0, 0);
    }

    return vector_meta.count;
}
```

## 9.3 TupleDataCollection（行数据集合）

### 9.3.1 类定义

```cpp
// src/include/duckdb/common/types/row/tuple_data_collection.hpp

class TupleDataCollection {
public:
    TupleDataCollection(BufferManager &buffer_manager,
                        shared_ptr<TupleDataLayout> layout_ptr,
                        MemoryTag tag);

    // 布局访问
    shared_ptr<TupleDataLayout> GetLayoutPtr() const;
    const TupleDataLayout &GetLayout() const;
    idx_t TuplesPerBlock() const;

    // 基本属性
    const idx_t &Count() const;
    idx_t ChunkCount() const;
    idx_t SizeInBytes() const;

    // 追加操作
    void InitializeAppend(TupleDataAppendState &append_state,
                          TupleDataPinProperties properties);
    void Append(DataChunk &new_chunk, const SelectionVector &append_sel,
                idx_t append_count);
    void Append(TupleDataAppendState &append_state, DataChunk &new_chunk,
                const SelectionVector &append_sel, idx_t append_count);

    // 扫描操作
    void InitializeScan(TupleDataScanState &state, TupleDataPinProperties properties);
    bool Scan(TupleDataScanState &state, DataChunk &result);

    // Scatter/Gather操作
    void Scatter(TupleDataChunkState &chunk_state, const DataChunk &new_chunk,
                 const SelectionVector &append_sel, const idx_t append_count) const;
    void Gather(Vector &row_locations, const SelectionVector &scan_sel,
                const idx_t scan_count, DataChunk &result,
                const SelectionVector &target_sel) const;

    // 合并操作
    void Combine(TupleDataCollection &other);

private:
    DatabaseInstance &db;
    shared_ptr<TupleDataLayout> layout_ptr;
    const TupleDataLayout &layout;
    shared_ptr<TupleDataAllocator> allocator;
    idx_t count;
    idx_t data_size;
    unsafe_arena_vector<unsafe_arena_ptr<TupleDataSegment>> segments;
    unsafe_arena_vector<TupleDataScatterFunction> scatter_functions;
    unsafe_arena_vector<TupleDataGatherFunction> gather_functions;
};
```

### 9.3.2 TupleDataLayout（行布局）

```cpp
// src/include/duckdb/common/types/row/tuple_data_layout.hpp

class TupleDataLayout {
public:
    void Initialize(vector<LogicalType> types, Aggregates aggregates,
                    TupleDataValidityType validity_type);

    // 布局信息
    idx_t ColumnCount() const;
    const vector<LogicalType> &GetTypes() const;
    idx_t GetRowWidth() const;      // 行总宽度
    idx_t GetDataOffset() const;    // 数据起始偏移
    idx_t GetDataWidth() const;     // 数据区宽度
    idx_t GetAggrOffset() const;    // 聚合区起始偏移
    idx_t GetAggrWidth() const;     // 聚合区宽度
    const vector<idx_t> &GetOffsets() const;  // 各列偏移
    bool AllConstant() const;       // 是否全为定长列
    idx_t GetHeapSizeOffset() const;  // 堆大小偏移

private:
    vector<LogicalType> types;
    Aggregates aggregates;
    idx_t flag_width;   // 有效性标志宽度
    idx_t data_width;   // 数据宽度
    idx_t aggr_width;   // 聚合宽度
    idx_t row_width;    // 行总宽度
    vector<idx_t> offsets;  // 列偏移数组
    bool all_constant;
    idx_t heap_size_offset;
};
```

### 9.3.3 行格式内存布局

```
单行内存布局（Row Format）
┌─────────────────────────────────────────────────────────────────┐
│  Validity Mask  │  Column Data  │  Aggregate State  │  Heap Ptr │
│   (N bytes)     │  (variable)   │    (variable)     │ (8 bytes) │
└─────────────────────────────────────────────────────────────────┘

详细布局：
┌──────────────────────────────────────────────────────────────────────────┐
│ Byte 0-N: Validity Bits (每列1位，按8字节对齐)                             │
├──────────────────────────────────────────────────────────────────────────┤
│ Column 0: [offset_0] 固定大小数据 (如 int64: 8 bytes)                     │
├──────────────────────────────────────────────────────────────────────────┤
│ Column 1: [offset_1] 固定大小数据                                         │
├──────────────────────────────────────────────────────────────────────────┤
│ ...                                                                      │
├──────────────────────────────────────────────────────────────────────────┤
│ Variable Column: 指向堆的指针 + 长度                                      │
├──────────────────────────────────────────────────────────────────────────┤
│ Aggregate State 0: [aggr_offset_0] 聚合函数状态                           │
├──────────────────────────────────────────────────────────────────────────┤
│ Heap Size: 该行堆数据总大小                                               │
├──────────────────────────────────────────────────────────────────────────┤
│ Heap Pointer: 指向堆数据起始位置                                          │
└──────────────────────────────────────────────────────────────────────────┘

堆数据（变长数据）：
┌──────────────────────────────────────────────────────────────────────────┐
│ String 1 Data │ String 2 Data │ List Data │ Nested Struct Data │ ...    │
└──────────────────────────────────────────────────────────────────────────┘
```

### 9.3.4 TupleDataSegment

```cpp
// src/include/duckdb/common/types/row/tuple_data_segment.hpp

struct TupleDataChunkPart {
    uint32_t row_block_index;    // 行块索引
    uint32_t row_block_offset;   // 行块内偏移
    uint32_t heap_block_index;   // 堆块索引
    uint32_t heap_block_offset;  // 堆块内偏移
    data_ptr_t base_heap_ptr;    // 堆基址
    idx_t total_heap_size;       // 堆总大小
    uint32_t count;              // 元组数
    reference<mutex> lock;       // 锁
};

struct TupleDataChunk {
    ContinuousIdSet part_ids;       // 部分ID集合
    ContinuousIdSet row_block_ids;  // 行块ID集合
    ContinuousIdSet heap_block_ids; // 堆块ID集合
    idx_t count;                    // 元组总数
    reference<mutex> lock;
};

struct TupleDataSegment {
    shared_ptr<TupleDataAllocator> allocator;
    const TupleDataLayout &layout;
    unsafe_vector<unsafe_arena_ptr<TupleDataChunk>> chunks;
    unsafe_vector<unsafe_arena_ptr<TupleDataChunkPart>> chunk_parts;
    idx_t count;
    idx_t data_size;
    mutex pinned_handles_lock;
    unsafe_arena_vector<BufferHandle> pinned_row_handles;
    unsafe_arena_vector<BufferHandle> pinned_heap_handles;
};
```

### 9.3.5 Scatter操作（列到行）

Scatter将列式数据转换为行格式：

```cpp
typedef void (*tuple_data_scatter_function_t)(
    const Vector &source,
    const TupleDataVectorFormat &source_format,
    const SelectionVector &append_sel,
    const idx_t append_count,
    const TupleDataLayout &layout,
    const Vector &row_locations,
    Vector &heap_locations,
    const idx_t col_idx,
    const UnifiedVectorFormat &list_format,
    const vector<TupleDataScatterFunction> &child_functions);

// 使用示例
void TupleDataCollection::Scatter(TupleDataChunkState &chunk_state,
                                  const DataChunk &new_chunk,
                                  const SelectionVector &append_sel,
                                  const idx_t append_count) const {
    auto row_locations = FlatVector::GetData<data_ptr_t>(chunk_state.row_locations);

    for (idx_t col_idx = 0; col_idx < layout.ColumnCount(); col_idx++) {
        auto &source = new_chunk.data[col_idx];
        auto &scatter_func = scatter_functions[col_idx];

        scatter_func.function(source, chunk_state.vector_data[col_idx],
                              append_sel, append_count, layout,
                              chunk_state.row_locations, chunk_state.heap_locations,
                              col_idx, chunk_state.list_format, scatter_func.child_functions);
    }
}
```

### 9.3.6 Gather操作（行到列）

Gather将行格式数据转换为列式：

```cpp
typedef void (*tuple_data_gather_function_t)(
    const TupleDataLayout &layout,
    Vector &row_locations,
    const idx_t col_idx,
    const SelectionVector &scan_sel,
    const idx_t scan_count,
    Vector &target,
    const SelectionVector &target_sel,
    optional_ptr<Vector> list_vector,
    const vector<TupleDataGatherFunction> &child_functions);

// 使用示例
void TupleDataCollection::Gather(Vector &row_locations,
                                  const SelectionVector &scan_sel,
                                  const idx_t scan_count,
                                  DataChunk &result,
                                  const SelectionVector &target_sel) const {
    for (idx_t col_idx = 0; col_idx < layout.ColumnCount(); col_idx++) {
        auto &gather_func = gather_functions[col_idx];
        auto &target = result.data[col_idx];

        gather_func.function(layout, row_locations, col_idx,
                             scan_sel, scan_count, target, target_sel,
                             nullptr, gather_func.child_functions);
    }

    result.SetCardinality(scan_count);
}
```

## 9.4 RowDataCollection（遗留行数据集合）

### 9.4.1 类定义

```cpp
// src/include/duckdb/common/types/row/row_data_collection.hpp

struct RowDataBlock {
    shared_ptr<BlockHandle> block;
    idx_t capacity;
    const idx_t entry_size;
    idx_t count;
    idx_t byte_offset;
};

class RowDataCollection {
public:
    RowDataCollection(BufferManager &buffer_manager, idx_t block_capacity,
                      idx_t entry_size, bool keep_pinned = false);

    BufferManager &buffer_manager;
    idx_t count;
    idx_t block_capacity;
    idx_t entry_size;
    vector<unique_ptr<RowDataBlock>> blocks;
    vector<BufferHandle> pinned_blocks;
    const bool keep_pinned;

public:
    idx_t AppendToBlock(RowDataBlock &block, BufferHandle &handle,
                        vector<BlockAppendEntry> &append_entries,
                        idx_t remaining, idx_t entry_sizes[]);
    RowDataBlock &CreateBlock();
    vector<BufferHandle> Build(idx_t added_count, data_ptr_t key_locations[],
                               idx_t entry_sizes[], const SelectionVector *sel);
    void Merge(RowDataCollection &other);
    void Clear();
    idx_t SizeInBytes() const;

    static idx_t EntriesPerBlock(idx_t width, idx_t block_size);
};
```

### 9.4.2 块分配

```cpp
RowDataBlock &RowDataCollection::CreateBlock() {
    auto new_block = make_uniq<RowDataBlock>(
        MemoryTag::HASH_TABLE,
        buffer_manager,
        block_capacity,
        entry_size);
    auto &result = *new_block;
    blocks.push_back(std::move(new_block));
    return result;
}

vector<BufferHandle> RowDataCollection::Build(idx_t added_count,
                                               data_ptr_t key_locations[],
                                               idx_t entry_sizes[],
                                               const SelectionVector *sel) {
    vector<BufferHandle> handles;
    vector<BlockAppendEntry> append_entries;

    idx_t remaining = added_count;
    while (remaining > 0) {
        auto &block = (blocks.empty() || blocks.back()->count >= blocks.back()->capacity)
                      ? CreateBlock()
                      : *blocks.back();

        auto handle = buffer_manager.Pin(block.block);
        auto appended = AppendToBlock(block, handle, append_entries, remaining, entry_sizes);
        remaining -= appended;

        if (keep_pinned) {
            handles.push_back(std::move(handle));
        }
    }

    count += added_count;
    return handles;
}
```

## 9.5 BatchedDataCollection

### 9.5.1 批次数据集合

BatchedDataCollection用于按批次索引组织数据，适合并行写入场景：

```cpp
// src/include/duckdb/common/types/batched_data_collection.hpp

class BatchedDataCollection {
public:
    BatchedDataCollection(ClientContext &context, vector<LogicalType> types,
                          ColumnDataAllocatorType allocator_type,
                          ColumnDataCollectionLifetime lifetime);

    // 按批次追加
    void Append(DataChunk &input, idx_t batch_index);

    // 合并批次
    void Merge(BatchedDataCollection &other);

    // 按批次顺序扫描
    void InitializeScan(BatchedChunkScanState &state);
    void Scan(BatchedChunkScanState &state, DataChunk &output);

    // 获取合并后的集合
    unique_ptr<ColumnDataCollection> FetchCollection();

    idx_t Count() const;
    idx_t BatchCount() const;

private:
    ClientContext &context;
    vector<LogicalType> types;
    ColumnDataAllocatorType allocator_type;
    ColumnDataCollectionLifetime lifetime;
    map<idx_t, unique_ptr<ColumnDataCollection>> data;  // batch_index -> collection
};
```

### 9.5.2 批次扫描

```cpp
void BatchedDataCollection::InitializeScan(BatchedChunkScanState &state) {
    state.range.begin = data.begin();
    state.range.end = data.end();
    if (state.range.begin != state.range.end) {
        state.range.begin->second->InitializeScan(state.scan_state);
    }
}

void BatchedDataCollection::Scan(BatchedChunkScanState &state, DataChunk &output) {
    while (state.range.begin != state.range.end) {
        auto &collection = *state.range.begin->second;
        if (collection.Scan(state.scan_state, output)) {
            return;  // 成功扫描到数据
        }
        // 当前批次扫描完毕，移到下一批次
        ++state.range.begin;
        if (state.range.begin != state.range.end) {
            state.range.begin->second->InitializeScan(state.scan_state);
        }
    }
    output.SetCardinality(0);
}
```

## 9.6 ColumnDataRowCollection

### 9.6.1 行级访问封装

ColumnDataRowCollection提供对ColumnDataCollection的行级访问：

```cpp
// src/include/duckdb/common/types/column/column_data_collection.hpp

class ColumnDataRowCollection {
public:
    explicit ColumnDataRowCollection(const ColumnDataCollection &collection,
                                     ColumnDataScanProperties properties);

    Value GetValue(idx_t column, idx_t index) const;

    // STL容器接口
    bool empty() const;
    idx_t size() const;
    ColumnDataRow &operator[](idx_t i);
    const ColumnDataRow &operator[](idx_t i) const;

    // 迭代器
    vector<ColumnDataRow>::iterator begin();
    vector<ColumnDataRow>::iterator end();
    vector<ColumnDataRow>::const_iterator cbegin() const;
    vector<ColumnDataRow>::const_iterator cend() const;

private:
    vector<ColumnDataRow> rows;
    vector<unique_ptr<DataChunk>> chunks;
    ColumnDataScanState scan_state;
};
```

### 9.6.2 ColumnDataRow

```cpp
class ColumnDataRow {
public:
    ColumnDataRow(DataChunk &chunk, idx_t row_index, idx_t base_index);

    // 获取单个值
    Value GetValue(idx_t column) const;

    // 行索引
    idx_t row_index;

private:
    DataChunk &chunk;
    idx_t internal_row_index;
};
```

## 9.7 内存管理

### 9.7.1 ColumnDataAllocator

```cpp
enum class ColumnDataAllocatorType {
    IN_MEMORY_ALLOCATOR,      // 纯内存分配
    BUFFER_MANAGER_ALLOCATOR  // 通过BufferManager分配（可溢出到磁盘）
};

class ColumnDataAllocator {
public:
    virtual ~ColumnDataAllocator() = default;

    // 分配数据块
    virtual idx_t Allocate(idx_t size) = 0;

    // 获取分配类型
    virtual ColumnDataAllocatorType GetType() const = 0;

    // 获取BufferManager（如果可用）
    virtual BufferManager &GetBufferManager() = 0;

    // Pin/Unpin操作
    virtual BufferHandle Pin(idx_t block_id) = 0;
};
```

### 9.7.2 TupleDataAllocator

```cpp
class TupleDataAllocator {
public:
    TupleDataAllocator(BufferManager &buffer_manager, const TupleDataLayout &layout);

    // 分配行块
    void AllocateRowBlock(TupleDataPinState &pin_state, idx_t count);

    // 分配堆块
    void AllocateHeapBlock(TupleDataPinState &pin_state, idx_t size);

    // 构建块空间
    void Build(TupleDataSegment &segment, TupleDataPinState &pin_state,
               TupleDataChunkState &chunk_state, const idx_t append_offset,
               const idx_t append_count);

private:
    BufferManager &buffer_manager;
    const TupleDataLayout &layout;
    vector<TupleDataBlock> row_blocks;
    vector<TupleDataBlock> heap_blocks;
};
```

### 9.7.3 Pin属性

```cpp
enum class TupleDataPinProperties {
    UNPIN_AFTER_DONE,          // 操作完成后解除Pin
    KEEP_EVERYTHING_PINNED,    // 保持所有块Pin住
    ALREADY_PINNED             // 块已经被Pin
};

enum class ColumnDataScanProperties {
    ALLOW_ZERO_COPY,      // 允许零拷贝（直接引用）
    DISALLOW_ZERO_COPY    // 禁止零拷贝
};
```

## 9.8 并行操作

### 9.8.1 并行扫描状态

```cpp
// 列数据集合并行扫描
struct ColumnDataParallelScanState {
    mutex lock;
    ColumnDataScanState scan_state;
};

struct ColumnDataLocalScanState {
    ChunkManagementState current_chunk_state;
};

// 元组数据集合并行扫描
struct TupleDataParallelScanState {
    mutex lock;
    idx_t segment_index;
    idx_t chunk_index;
};

struct TupleDataLocalScanState {
    TupleDataPinState pin_state;
    TupleDataChunkState chunk_state;
};
```

### 9.8.2 并行追加

TupleDataCollection支持多线程并行追加：

```cpp
// 每个线程维护自己的追加状态
void ParallelAppend(TupleDataCollection &collection, DataChunk &chunk) {
    TupleDataAppendState append_state;
    collection.InitializeAppend(append_state, TupleDataPinProperties::UNPIN_AFTER_DONE);
    collection.Append(append_state, chunk);
}
```

## 9.9 使用场景对比

### 9.9.1 ColumnDataCollection适用场景

1. **物化查询结果**：存储最终查询结果供客户端消费
2. **窗口函数**：存储窗口分区数据
3. **子查询物化**：缓存子查询结果
4. **CTE递归**：存储递归CTE的中间结果

```cpp
// 物化查询结果示例
ColumnDataCollection result(context, result_types);
while (executor.Execute(chunk)) {
    result.Append(chunk);
}
// 结果可供多次扫描
```

### 9.9.2 TupleDataCollection适用场景

1. **哈希表构建**：JOIN和GROUP BY的哈希表存储
2. **排序**：外部排序的行存储
3. **聚合**：聚合函数状态存储
4. **分区**：按哈希或范围分区的数据存储

```cpp
// 哈希表构建示例
TupleDataLayout layout;
layout.Initialize(key_types, aggregates, TupleDataValidityType::CAN_HAVE_NULL_VALUES);

TupleDataCollection collection(buffer_manager, make_shared<TupleDataLayout>(layout), MemoryTag::HASH_TABLE);

// 追加数据
TupleDataAppendState append_state;
collection.InitializeAppend(append_state);
collection.Append(append_state, chunk);

// Scatter到行格式（用于哈希表探测）
collection.Scatter(chunk_state, chunk, sel, count);
```

## 9.10 性能优化

### 9.10.1 块重用

ColumnDataCollection支持块继承，避免重复分配：

```cpp
// 子集合继承父集合的块
ColumnDataCollection child(parent);  // 继承半填充的块
// parent不能再写入，child继续写入
```

### 9.10.2 预分配

TupleDataCollection支持预分配以减少重分配：

```cpp
// 根据预估大小初始化
TupleDataAppendState append_state;
collection.InitializeAppend(append_state, TupleDataPinProperties::KEEP_EVERYTHING_PINNED);
// 批量追加减少锁竞争
```

### 9.10.3 延迟物化

两种集合都支持延迟物化，只在需要时才读取数据：

```cpp
// 只扫描需要的列
vector<column_t> needed_columns = {0, 2, 5};
ColumnDataScanState state;
collection.InitializeScan(state, needed_columns, ColumnDataScanProperties::ALLOW_ZERO_COPY);
```

## 9.11 小结

DuckDB的行数据和列数据集合提供了灵活的数据存储抽象：

1. **ColumnDataCollection**：列式存储，优化顺序扫描和分析查询
2. **TupleDataCollection**：行式存储，优化随机访问和哈希操作
3. **BatchedDataCollection**：批次组织，支持并行写入和有序扫描

这些集合与BufferManager集成，支持内存溢出到磁盘，使DuckDB能够处理超出内存大小的数据集。通过Scatter/Gather操作，行列格式可以高效转换，满足不同操作符的需求。
