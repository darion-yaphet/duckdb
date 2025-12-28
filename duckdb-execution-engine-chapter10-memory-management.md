# DuckDB 执行引擎深度解析：第十章 - 执行时内存管理

## 10.1 章节概述

高效的内存管理是数据库系统性能的关键。本章深入分析 DuckDB 执行时的内存管理机制，包括算子状态管理、ColumnDataCollection 中间结果存储、Spilling 机制、内存预算控制以及查询监控与性能分析。

```
┌────────────────────────────────────────────────────────────────────┐
│                    执行时内存管理架构                               │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                     BufferManager                            │ │
│   │  • 内存分配与回收                                            │ │
│   │  • 缓冲区池管理                                              │ │
│   │  • 临时文件管理                                              │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│          ┌─────────────────────┼─────────────────────┐            │
│          ▼                     ▼                     ▼            │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐    │
│   │OperatorState│       │ColumnData  │       │   Spilling  │    │
│   │             │       │ Collection │       │             │    │
│   │ • Global    │       │             │       │ • 磁盘溢出  │    │
│   │ • Local     │       │ • 中间结果  │       │ • 数据恢复  │    │
│   │ • Source    │       │ • 分段存储  │       │ • 分区管理  │    │
│   │ • Sink      │       │ • 迭代访问  │       │             │    │
│   └─────────────┘       └─────────────┘       └─────────────┘    │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                     QueryProfiler                            │ │
│   │  • 执行统计收集                                              │ │
│   │  • EXPLAIN ANALYZE                                           │ │
│   │  • 性能瓶颈分析                                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 10.2 算子状态管理

### 10.2.1 状态类型层次

DuckDB 的算子状态分为四类，分别服务于不同的执行角色：

```cpp
// src/include/duckdb/execution/physical_operator_states.hpp

// ===== 基础算子状态（中间算子使用）=====
class OperatorState {
public:
    virtual ~OperatorState() {}

    //! 算子完成时调用
    virtual void Finalize(const PhysicalOperator &op, ExecutionContext &context) {}

    template <class TARGET>
    TARGET &Cast() {
        DynamicCastCheck<TARGET>(this);
        return reinterpret_cast<TARGET &>(*this);
    }
};

// ===== 全局算子状态 =====
class GlobalOperatorState {
public:
    virtual ~GlobalOperatorState() {}

    //! 返回最大线程数
    virtual idx_t MaxThreads(idx_t source_max_threads) {
        return source_max_threads;
    }
};

// ===== Sink 全局状态 =====
class GlobalSinkState : public StateWithBlockableTasks {
public:
    GlobalSinkState() : state(SinkFinalizeType::READY) {}
    virtual ~GlobalSinkState() {}

    SinkFinalizeType state;

    virtual idx_t MaxThreads(idx_t source_max_threads) {
        return source_max_threads;
    }
};

// ===== Sink 本地状态 =====
class LocalSinkState {
public:
    virtual ~LocalSinkState() {}

    //! Source 分区信息
    SourcePartitionInfo partition_info;
};

// ===== Source 全局状态 =====
class GlobalSourceState : public StateWithBlockableTasks {
public:
    virtual ~GlobalSourceState() {}

    //! 返回最大并行度
    virtual idx_t MaxThreads() {
        return 1;
    }
};

// ===== Source 本地状态 =====
class LocalSourceState {
public:
    virtual ~LocalSourceState() {}
};
```

### 10.2.2 状态生命周期

```
┌────────────────────────────────────────────────────────────────────┐
│                      状态生命周期                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Pipeline 执行阶段                                                │
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │ 1. 初始化阶段                                                │ │
│   │    • GetGlobalSinkState() → 创建 GlobalSinkState            │ │
│   │    • GetGlobalSourceState() → 创建 GlobalSourceState        │ │
│   │    • GetLocalSinkState() → 每个线程创建 LocalSinkState      │ │
│   │    • GetLocalSourceState() → 每个线程创建 LocalSourceState  │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │ 2. Sink 阶段（数据流入）                                     │ │
│   │    • Sink(chunk, input) → 使用 GlobalSinkState + LocalSink  │ │
│   │    • 数据写入本地状态                                        │ │
│   │    • 可能触发 Spilling                                       │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │ 3. Combine 阶段（本地状态合并）                              │ │
│   │    • Combine(input) → 合并 LocalSinkState 到 GlobalSinkState│ │
│   │    • 每个线程完成后调用                                      │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │ 4. Finalize 阶段（全局完成）                                 │ │
│   │    • Finalize(input) → 完成 GlobalSinkState 构建            │ │
│   │    • 可能触发归并、排序等操作                                │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │ 5. Source 阶段（数据流出）                                   │ │
│   │    • GetData(chunk, input) → 从 GlobalSourceState 读取数据  │ │
│   │    • 支持并行扫描                                            │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │ 6. 清理阶段                                                  │ │
│   │    • 析构函数释放所有状态                                    │ │
│   │    • 释放临时文件                                            │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.2.3 状态输入结构

```cpp
// 用于 Sink 操作的输入
struct OperatorSinkInput {
    GlobalSinkState &global_state;
    LocalSinkState &local_state;
    InterruptState &interrupt_state;
};

// 用于 Source 操作的输入
struct OperatorSourceInput {
    GlobalSourceState &global_state;
    LocalSourceState &local_state;
    InterruptState &interrupt_state;
};

// 用于 Combine 操作的输入
struct OperatorSinkCombineInput {
    GlobalSinkState &global_state;
    LocalSinkState &local_state;
    InterruptState &interrupt_state;
};

// 用于 Finalize 操作的输入
struct OperatorSinkFinalizeInput {
    GlobalSinkState &global_state;
    InterruptState &interrupt_state;
};
```

---

## 10.3 ColumnDataCollection：中间结果存储

### 10.3.1 类设计

```cpp
// src/include/duckdb/common/types/column/column_data_collection.hpp

class ColumnDataCollection {
public:
    //! 内存分配器构造
    DUCKDB_API ColumnDataCollection(Allocator &allocator, vector<LogicalType> types);

    //! 缓冲区管理器构造（支持 Spilling）
    DUCKDB_API ColumnDataCollection(BufferManager &buffer_manager, vector<LogicalType> types,
                                     ColumnDataCollectionLifetime lifetime = ColumnDataCollectionLifetime::REGULAR);

    //! 上下文构造
    DUCKDB_API ColumnDataCollection(ClientContext &context, vector<LogicalType> types,
                                     ColumnDataAllocatorType type = ColumnDataAllocatorType::BUFFER_MANAGER_ALLOCATOR,
                                     ColumnDataCollectionLifetime lifetime = ColumnDataCollectionLifetime::REGULAR);

    //! 继承分配器（共享块）
    DUCKDB_API ColumnDataCollection(ColumnDataCollection &parent);

public:
    //! 获取列类型
    vector<LogicalType> &Types() { return types; }

    //! 获取行数
    const idx_t &Count() const { return count; }

    //! 获取列数
    idx_t ColumnCount() const { return types.size(); }

    //! 获取大小（字节）
    idx_t SizeInBytes() const;
    idx_t AllocationSize() const;

    //===--------------------------------------------------------------------===//
    // 追加操作
    //===--------------------------------------------------------------------===//
    //! 初始化追加状态
    DUCKDB_API void InitializeAppend(ColumnDataAppendState &state);
    //! 追加数据块
    DUCKDB_API void Append(ColumnDataAppendState &state, DataChunk &new_chunk);
    //! 直接追加（内部初始化状态）
    DUCKDB_API void Append(DataChunk &new_chunk);
    //! 合并另一个 Collection
    DUCKDB_API void Combine(ColumnDataCollection &other);

    //===--------------------------------------------------------------------===//
    // 扫描操作
    //===--------------------------------------------------------------------===//
    //! 初始化扫描块
    DUCKDB_API void InitializeScanChunk(DataChunk &chunk) const;
    //! 初始化扫描状态
    DUCKDB_API void InitializeScan(ColumnDataScanState &state,
                                    ColumnDataScanProperties properties = ColumnDataScanProperties::ALLOW_ZERO_COPY) const;
    //! 初始化并行扫描
    DUCKDB_API void InitializeScan(ColumnDataParallelScanState &state,
                                    ColumnDataScanProperties properties = ColumnDataScanProperties::ALLOW_ZERO_COPY) const;
    //! 扫描数据
    DUCKDB_API bool Scan(ColumnDataScanState &state, DataChunk &result) const;
    DUCKDB_API bool Scan(ColumnDataParallelScanState &state, ColumnDataLocalScanState &lstate, DataChunk &result) const;

    //===--------------------------------------------------------------------===//
    // 随机访问
    //===--------------------------------------------------------------------===//
    //! 获取数据块数量
    DUCKDB_API idx_t ChunkCount() const;
    //! 获取指定数据块
    DUCKDB_API void FetchChunk(idx_t chunk_idx, DataChunk &result) const;

    //! 迭代器访问
    DUCKDB_API ColumnDataChunkIterationHelper Chunks() const;
    DUCKDB_API ColumnDataRowIterationHelper Rows() const;

    //! 重置
    DUCKDB_API void Reset();

private:
    //! 分配器
    buffer_ptr<ColumnDataAllocator> allocator;
    //! 列类型
    vector<LogicalType> types;
    //! 行数
    idx_t count;
    //! 数据段
    vector<unique_ptr<ColumnDataCollectionSegment>> segments;
    //! 复制函数
    vector<ColumnDataCopyFunction> copy_functions;
    //! 是否完成追加
    bool finished_append;
};
```

### 10.3.2 内存布局

```
┌────────────────────────────────────────────────────────────────────┐
│                  ColumnDataCollection 内存布局                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ColumnDataCollection                                             │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ types: [INT, VARCHAR, DOUBLE]                                ││
│   │ count: 50000                                                 ││
│   │ segments: [Segment1, Segment2, ...]                          ││
│   └──────────────────────────────────────────────────────────────┘│
│                     │                                              │
│                     ▼                                              │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │              Segment 1 (4096 rows)                           │ │
│   │  ┌─────────────────────────────────────────────────────────┐│ │
│   │  │ Block 1 (Column 0: INT)                                 ││ │
│   │  │  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐          ││ │
│   │  │  │ 1   │ 2   │ 3   │ 4   │ ... │ 2048│ ... │          ││ │
│   │  │  └─────┴─────┴─────┴─────┴─────┴─────┴─────┘          ││ │
│   │  └─────────────────────────────────────────────────────────┘│ │
│   │  ┌─────────────────────────────────────────────────────────┐│ │
│   │  │ Block 2 (Column 1: VARCHAR)                             ││ │
│   │  │  ┌──────────┬──────────┬──────────┬──────────┐         ││ │
│   │  │  │ "hello"  │ "world"  │ "foo"    │ ...      │         ││ │
│   │  │  └──────────┴──────────┴──────────┴──────────┘         ││ │
│   │  │  + StringHeap (存储非内联字符串)                        ││ │
│   │  └─────────────────────────────────────────────────────────┘│ │
│   │  ┌─────────────────────────────────────────────────────────┐│ │
│   │  │ Block 3 (Column 2: DOUBLE)                              ││ │
│   │  │  ┌─────┬─────┬─────┬─────┬─────┬─────┬─────┐          ││ │
│   │  │  │ 1.5 │ 2.7 │ 3.14│ ... │ ... │ ... │ ... │          ││ │
│   │  │  └─────┴─────┴─────┴─────┴─────┴─────┴─────┘          ││ │
│   │  └─────────────────────────────────────────────────────────┘│ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │              Segment 2 (4096 rows)                           │ │
│   │  ... (类似结构)                                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   特点:                                                           │
│   • 列式存储，每列独立的内存块                                     │
│   • 按段组织，每段固定行数                                         │
│   • 字符串使用独立的 StringHeap                                   │
│   • 支持零拷贝扫描（ALLOW_ZERO_COPY）                             │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.3.3 使用模式

```cpp
// 1. 创建 Collection
ColumnDataCollection collection(context, {LogicalType::INTEGER, LogicalType::VARCHAR});

// 2. 追加数据
ColumnDataAppendState append_state;
collection.InitializeAppend(append_state);
for (auto &chunk : input_chunks) {
    collection.Append(append_state, chunk);
}

// 3. 扫描数据
ColumnDataScanState scan_state;
collection.InitializeScan(scan_state);
DataChunk result;
collection.InitializeScanChunk(result);
while (collection.Scan(scan_state, result)) {
    // 处理 result
}

// 4. 并行扫描
ColumnDataParallelScanState parallel_state;
collection.InitializeScan(parallel_state);

// 每个线程
ColumnDataLocalScanState local_state;
DataChunk result;
while (collection.Scan(parallel_state, local_state, result)) {
    // 并行处理
}
```

---

## 10.4 Spilling 机制

### 10.4.1 概述

当内存不足时，DuckDB 将数据溢出到磁盘，这称为 Spilling。主要发生在以下算子：

- **HashJoin**: Build 端数据量过大
- **HashAggregate**: 分组数量过多
- **Sort**: 排序数据量过大
- **Window**: 窗口函数数据量过大

### 10.4.2 分配器类型

```cpp
enum class ColumnDataAllocatorType : uint8_t {
    //! 纯内存分配
    IN_MEMORY_ALLOCATOR,
    //! 缓冲区管理器分配（支持 Spilling）
    BUFFER_MANAGER_ALLOCATOR,
    //! 混合分配
    HYBRID_ALLOCATOR
};

enum class ColumnDataCollectionLifetime : uint8_t {
    //! 常规生命周期
    REGULAR,
    //! 长期存储（跨查询）
    LONG_TERM
};
```

### 10.4.3 外部 Hash Join Spilling

```
┌────────────────────────────────────────────────────────────────────┐
│                    External Hash Join Spilling                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Phase 1: 内存不足检测                                            │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  Build 端数据量超过内存预算                                  │ │
│   │  触发 External Hash Join 模式                                │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   Phase 2: Radix 分区                                             │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  Build 端数据按 Hash 值分区                                  │ │
│   │                                                              │ │
│   │  输入数据 ─────► Hash(key) % N ─────► Partition 0           │ │
│   │                              └─────► Partition 1           │ │
│   │                              └─────► ...                    │ │
│   │                              └─────► Partition N-1         │ │
│   │                                                              │ │
│   │  内存中保留: Partition 0                                     │ │
│   │  溢出到磁盘: Partition 1 ~ N-1                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   Phase 3: Probe 端分区                                           │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  Probe 端同样按 Hash 值分区                                  │ │
│   │                                                              │ │
│   │  Probe Partition 0 ────► Join with Build Partition 0        │ │
│   │  Probe Partition 1 ────► 溢出到磁盘                         │ │
│   │  ...                                                         │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   Phase 4: 递归处理溢出分区                                       │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  for each spilled partition:                                 │ │
│   │    1. 读取 Build Partition i 到内存                         │ │
│   │    2. 构建 Hash Table                                        │ │
│   │    3. 读取 Probe Partition i                                 │ │
│   │    4. 执行 Join                                              │ │
│   │    5. 输出结果                                               │ │
│   │                                                              │ │
│   │  如果单个分区仍然过大，递归分区                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.4.4 外部排序 Spilling

```
┌────────────────────────────────────────────────────────────────────┐
│                    External Sort Spilling                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Phase 1: 创建初始 Runs                                          │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                                                              │ │
│   │  内存缓冲区 (M bytes)                                       │ │
│   │  ┌────────────────────────────────────────────────────────┐│ │
│   │  │  读取数据直到缓冲区满                                   ││ │
│   │  │  内存排序                                               ││ │
│   │  │  写入临时文件 (Run 1)                                   ││ │
│   │  └────────────────────────────────────────────────────────┘│ │
│   │                                                              │ │
│   │  重复直到所有输入处理完毕:                                  │ │
│   │  Run 1 → temp_sort_1.duckdb                                │ │
│   │  Run 2 → temp_sort_2.duckdb                                │ │
│   │  ...                                                         │ │
│   │  Run K → temp_sort_K.duckdb                                │ │
│   │                                                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   Phase 2: K-way 归并                                             │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                                                              │ │
│   │  Run 1 ─────┐                                               │ │
│   │  Run 2 ─────┤                                               │ │
│   │  Run 3 ─────┼────► 优先队列 ────► 排序后输出                │ │
│   │  ...   ─────┤      (最小堆)                                 │ │
│   │  Run K ─────┘                                               │ │
│   │                                                              │ │
│   │  每次从每个 Run 的缓冲区取最小元素                          │ │
│   │  输出后从对应 Run 补充                                      │ │
│   │                                                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
│   多轮归并（如果 K 太大）:                                        │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │  Round 1: K Runs → K/F Runs (F = fan-in)                    │ │
│   │  Round 2: K/F Runs → K/F² Runs                              │ │
│   │  ...                                                         │ │
│   │  Final: 1 Sorted Run                                        │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.4.5 临时文件管理

```cpp
// 临时文件由 BufferManager 管理
class BufferManager {
public:
    //! 分配临时块
    shared_ptr<BlockHandle> Allocate(MemoryTag tag, idx_t block_size,
                                      bool can_destroy = true,
                                      optional_ptr<MemoryState> memory_state = nullptr);

    //! 注册小块（用于小型分配）
    shared_ptr<BlockHandle> RegisterSmallMemory(idx_t block_size);

    //! 固定块（加载到内存）
    BufferHandle Pin(shared_ptr<BlockHandle> &handle);

    //! 解除固定（允许驱逐）
    void Unpin(shared_ptr<BlockHandle> &handle);

    //! 驱逐块到磁盘
    bool Evict(idx_t size_to_free);
};
```

---

## 10.5 内存预算控制

### 10.5.1 内存限制配置

```sql
-- 设置最大内存使用
SET memory_limit = '4GB';

-- 设置线程数（影响内存使用）
SET threads = 4;

-- 设置临时目录
SET temp_directory = '/path/to/temp';
```

### 10.5.2 算子内存分配策略

```
┌────────────────────────────────────────────────────────────────────┐
│                     内存分配策略                                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   全局内存预算                                                     │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ Total Memory: 4GB                                            ││
│   └──────────────────────────────────────────────────────────────┘│
│                     │                                              │
│          ┌──────────┴──────────┐                                   │
│          ▼                     ▼                                   │
│   ┌─────────────┐       ┌─────────────┐                           │
│   │  Buffer Pool │       │ Temp Storage │                          │
│   │   (60%)     │       │   (40%)      │                          │
│   │   2.4GB     │       │   1.6GB      │                          │
│   └─────────────┘       └─────────────┘                           │
│          │                                                         │
│          ▼                                                         │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                    算子内存分配                              │ │
│   │                                                              │ │
│   │   Query 1                     Query 2                        │ │
│   │   ┌────────────────┐         ┌────────────────┐             │ │
│   │   │ HashJoin: 500MB│         │ Sort: 300MB    │             │ │
│   │   │ Aggregate: 200MB│        │ Window: 400MB  │             │ │
│   │   └────────────────┘         └────────────────┘             │ │
│   │                                                              │ │
│   │   当总使用超过阈值时:                                        │ │
│   │   1. 首先尝试释放未使用的块                                  │ │
│   │   2. 驱逐 LRU 块到磁盘                                       │ │
│   │   3. 触发算子 Spilling                                       │ │
│   │                                                              │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 10.5.3 内存压力响应

```cpp
// 内存压力检测和响应
class MemoryManager {
public:
    bool CheckMemoryPressure() {
        idx_t current_usage = GetCurrentUsage();
        idx_t limit = GetMemoryLimit();

        if (current_usage > limit * 0.9) {
            // 高内存压力
            return true;
        }
        return false;
    }

    void HandleMemoryPressure() {
        // 1. 尝试释放缓存
        buffer_manager.PurgeUnpinnedBlocks();

        // 2. 驱逐块到磁盘
        if (CheckMemoryPressure()) {
            buffer_manager.Evict(GetTargetFreeMemory());
        }

        // 3. 如果仍然不足，抛出 OutOfMemory 异常
        if (CheckMemoryPressure()) {
            throw OutOfMemoryException("Memory limit exceeded");
        }
    }
};
```

---

## 10.6 QueryProfiler：查询性能分析

### 10.6.1 性能数据收集

```cpp
class QueryProfiler {
public:
    //! 开始算子计时
    void StartOperator(const PhysicalOperator &op);
    //! 结束算子计时
    void EndOperator(const PhysicalOperator &op, optional_ptr<DataChunk> chunk);

    //! 获取 EXPLAIN ANALYZE 输出
    string ToExplainAnalyzeOutput(ExplainOutputType output_type);

    //! 获取详细统计
    const ProfilingNode &GetRoot() const { return *root; }

private:
    //! 执行树根节点
    unique_ptr<ProfilingNode> root;
    //! 当前正在执行的节点
    optional_ptr<ProfilingNode> current_node;
};
```

### 10.6.2 EXPLAIN ANALYZE 输出

```sql
EXPLAIN ANALYZE
SELECT c.name, SUM(o.amount)
FROM customers c
JOIN orders o ON c.id = o.customer_id
WHERE c.country = 'China'
GROUP BY c.name
ORDER BY SUM(o.amount) DESC
LIMIT 10;
```

输出示例：
```
┌────────────────────────────────────────────────────────────────────┐
│                         Query Profile                              │
├────────────────────────────────────────────────────────────────────┤
│ TopN                                                               │
│   Time: 0.5ms    Rows: 10       Memory: 4KB                       │
│   │                                                                │
│   └─ HashAggregate                                                │
│        Time: 25ms    Rows: 1000    Memory: 128KB                  │
│        Groups: 1000                                               │
│        │                                                           │
│        └─ HashJoin (INNER)                                        │
│             Time: 150ms   Rows: 50000   Memory: 8MB               │
│             Build: 100ms  Probe: 50ms                             │
│             │                                                      │
│             ├─ Seq Scan: customers [Build]                        │
│             │    Time: 30ms    Rows: 5000    Memory: 256KB        │
│             │    Filter: country = 'China'                        │
│             │    Filtered: 45000 rows                             │
│             │                                                      │
│             └─ Seq Scan: orders [Probe]                           │
│                  Time: 80ms    Rows: 500000   Memory: 1MB         │
│                                                                    │
│ Total Time: 285.5ms                                               │
│ Peak Memory: 9.4MB                                                │
└────────────────────────────────────────────────────────────────────┘
```

### 10.6.3 性能指标

```
┌────────────────────────────────────────────────────────────────────┐
│                    性能指标收集                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   每个算子收集的指标:                                              │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ • Time: 执行时间 (wall clock)                                ││
│   │ • Rows: 处理/输出的行数                                      ││
│   │ • Memory: 峰值内存使用                                       ││
│   │ • Cardinality: 估计基数 vs 实际基数                          ││
│   └──────────────────────────────────────────────────────────────┘│
│                                                                    │
│   特定算子的额外指标:                                              │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ HashJoin:                                                    ││
│   │   • Build Time / Probe Time                                  ││
│   │   • Hash Table Size                                          ││
│   │   • Perfect Hash Join (if used)                             ││
│   │                                                              ││
│   │ HashAggregate:                                               ││
│   │   • Number of Groups                                         ││
│   │   • Spill Partitions (if spilled)                           ││
│   │                                                              ││
│   │ Sort:                                                        ││
│   │   • Sort Runs                                                ││
│   │   • External Sort (if spilled)                              ││
│   │                                                              ││
│   │ TableScan:                                                   ││
│   │   • Segments Scanned                                         ││
│   │   • Segments Skipped (zonemap filter)                       ││
│   │   • Filter Selectivity                                       ││
│   └──────────────────────────────────────────────────────────────┘│
│                                                                    │
│   全局指标:                                                        │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ • Total Execution Time                                       ││
│   │ • Peak Memory Usage                                          ││
│   │ • Number of Pipelines                                        ││
│   │ • Parallelism Achieved                                       ││
│   │ • I/O Statistics (reads, writes)                             ││
│   └──────────────────────────────────────────────────────────────┘│
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 10.7 ProgressBar：执行进度

### 10.7.1 进度追踪

```cpp
class ProgressBar {
public:
    //! 更新进度
    void Update(bool final);

    //! 获取进度百分比
    double GetProgress() const;

    //! 获取已处理行数
    idx_t GetRowsProcessed() const;

    //! 获取预计剩余时间
    double GetEstimatedTimeRemaining() const;
};

class ProgressData {
public:
    //! 已完成的行数
    idx_t done;
    //! 总行数（估计）
    idx_t total;

    //! 计算进度百分比
    double GetPercentage() const {
        if (total == 0) return 100.0;
        return (double)done / total * 100.0;
    }
};
```

### 10.7.2 进度计算

```
┌────────────────────────────────────────────────────────────────────┐
│                       进度计算                                      │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   Pipeline 1: TableScan → Filter → HashJoin(Build)                │
│   Progress: rows_scanned / estimated_total_rows                    │
│   ════════════════════════════░░░░░░░░░ 60%                       │
│                                                                    │
│   Pipeline 2: TableScan → HashJoin(Probe) → Aggregate             │
│   Progress: rows_scanned / estimated_total_rows                    │
│   ═══════════════════════════════════░░ 85%                       │
│                                                                    │
│   Pipeline 3: Aggregate(Scan) → Sort                              │
│   Progress: rows_output / total_groups                             │
│   ══════════════════░░░░░░░░░░░░░░░░░░ 40%                       │
│                                                                    │
│   总体进度计算:                                                    │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ 方法 1: 加权平均                                             ││
│   │   total_progress = Σ(pipeline_progress * pipeline_weight)   ││
│   │   weight = estimated_cost / total_estimated_cost            ││
│   │                                                              ││
│   │ 方法 2: 完成 Pipeline 计数                                   ││
│   │   total_progress = completed_pipelines / total_pipelines    ││
│   └──────────────────────────────────────────────────────────────┘│
│                                                                    │
│   进度显示:                                                        │
│   ┌──────────────────────────────────────────────────────────────┐│
│   │ Executing query...                                           ││
│   │ [████████████████████░░░░░░░░░░░░░░░░░░░░] 52%              ││
│   │ Rows processed: 1.2M / 2.3M estimated                       ││
│   │ Time elapsed: 5.2s  |  ETA: 4.8s                            ││
│   └──────────────────────────────────────────────────────────────┘│
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 10.8 ThreadContext：线程上下文

### 10.8.1 线程本地数据

```cpp
class ThreadContext {
public:
    ThreadContext(ClientContext &context);

    //! 客户端上下文引用
    ClientContext &context;

    //! 线程本地性能数据
    ClientProfiler profiler;
};

class ExecutionContext {
public:
    ExecutionContext(ClientContext &client, ThreadContext &thread,
                     optional_ptr<Pipeline> pipeline);

    //! 客户端上下文
    ClientContext &client;
    //! 线程上下文
    ThreadContext &thread;
    //! 当前 Pipeline
    optional_ptr<Pipeline> pipeline;
};
```

### 10.8.2 上下文数据流

```
┌────────────────────────────────────────────────────────────────────┐
│                      上下文数据流                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                   ClientContext                              │ │
│   │  • 数据库连接                                                │ │
│   │  • 事务状态                                                  │ │
│   │  • 配置参数                                                  │ │
│   │  • Catalog 引用                                              │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│                                ▼                                   │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                   Executor                                   │ │
│   │  • Pipeline 列表                                             │ │
│   │  • 事件管理                                                  │ │
│   │  • 错误处理                                                  │ │
│   └────────────────────────────┬────────────────────────────────┘ │
│                                │                                   │
│          ┌─────────────────────┼─────────────────────┐            │
│          ▼                     ▼                     ▼            │
│   ┌─────────────┐       ┌─────────────┐       ┌─────────────┐    │
│   │ThreadContext│       │ThreadContext│       │ThreadContext│    │
│   │  Thread 1   │       │  Thread 2   │       │  Thread 3   │    │
│   │             │       │             │       │             │    │
│   │ • profiler  │       │ • profiler  │       │ • profiler  │    │
│   │ • temp_data │       │ • temp_data │       │ • temp_data │    │
│   └──────┬──────┘       └──────┬──────┘       └──────┬──────┘    │
│          │                     │                     │            │
│          ▼                     ▼                     ▼            │
│   ┌─────────────────────────────────────────────────────────────┐ │
│   │                 ExecutionContext                             │ │
│   │  传递给算子的 Execute/Sink/Source 方法                       │ │
│   │  包含: client + thread + pipeline 引用                       │ │
│   └─────────────────────────────────────────────────────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## 10.9 源文件索引

| 组件 | 文件路径 |
|------|----------|
| OperatorState | `src/include/duckdb/execution/physical_operator_states.hpp` |
| ColumnDataCollection | `src/include/duckdb/common/types/column/column_data_collection.hpp` |
| BufferManager | `src/include/duckdb/storage/buffer_manager.hpp` |
| QueryProfiler | `src/include/duckdb/main/query_profiler.hpp` |
| ProgressBar | `src/include/duckdb/main/progress_bar/progress_bar.hpp` |
| ThreadContext | `src/include/duckdb/parallel/thread_context.hpp` |
| ExecutionContext | `src/include/duckdb/execution/execution_context.hpp` |
| External Hash Join | `src/execution/operator/join/physical_hash_join.cpp` |

---

## 10.10 本章小结

本章深入分析了 DuckDB 执行时的内存管理机制：

1. **算子状态管理** 提供了清晰的状态层次结构
   - GlobalSinkState / LocalSinkState 用于 Sink 操作
   - GlobalSourceState / LocalSourceState 用于 Source 操作
   - OperatorState 用于中间算子

2. **ColumnDataCollection** 是通用的中间结果存储
   - 列式存储，按段组织
   - 支持追加、扫描、并行访问
   - 支持零拷贝优化

3. **Spilling 机制** 处理内存不足情况
   - External Hash Join 使用 Radix 分区
   - External Sort 使用多路归并
   - 临时文件由 BufferManager 管理

4. **内存预算控制** 限制查询内存使用
   - 可配置的内存限制
   - 内存压力检测和响应
   - LRU 块驱逐策略

5. **QueryProfiler** 收集执行统计
   - 每个算子的时间、行数、内存
   - EXPLAIN ANALYZE 输出
   - 性能瓶颈分析

6. **ProgressBar** 显示执行进度
   - Pipeline 级别进度追踪
   - 预计剩余时间

7. **ThreadContext** 管理线程本地数据
   - 线程本地性能收集
   - 执行上下文传递

DuckDB 的内存管理设计兼顾了性能和资源控制，通过 ColumnDataCollection 提供高效的中间结果存储，通过 Spilling 机制优雅地处理内存不足，通过 QueryProfiler 提供详细的性能分析能力。

---

## 系列总结

至此，DuckDB 执行引擎深度解析系列全部完成。本系列共 10 章，系统性地分析了 DuckDB 执行引擎的各个核心组件：

| 章节 | 主题 | 核心内容 |
|------|------|----------|
| 第一章 | 执行模型概述 | Push-based 模型、Pipeline 概念、向量化执行 |
| 第二章 | Pipeline 与调度 | MetaPipeline、事件系统、任务调度 |
| 第三章 | 向量化执行基础 | Vector、DataChunk、SelectionVector |
| 第四章 | 表达式执行器 | ExpressionExecutor、函数执行 |
| 第五章 | 扫描与过滤算子 | TableScan、Filter、Projection |
| 第六章 | Join 算子实现 | HashJoin、MergeJoin、动态过滤 |
| 第七章 | 聚合与分组算子 | HashAggregate、Window 函数 |
| 第八章 | 排序与 TopN 算子 | Sort、TopNHeap、外部排序 |
| 第九章 | 并行执行 | TaskScheduler、Morsel-Driven 并行 |
| 第十章 | 执行时内存管理 | 状态管理、Spilling、QueryProfiler |

通过本系列，读者应该能够深入理解 DuckDB 执行引擎的设计哲学和实现细节，为进一步研究和贡献 DuckDB 项目打下坚实基础。
