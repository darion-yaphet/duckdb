# 第九章：并发数据结构

## 9.1 概述

DuckDB 实现了多种并发数据结构，用于在多线程环境中高效、安全地共享数据。这些数据结构采用不同的同步策略：

1. **原子类型**：使用 C++ 标准库的 `std::atomic`，实现无锁原子操作
2. **无锁队列**：使用 moodycamel 的 ConcurrentQueue，实现高性能任务调度
3. **锁保护容器**：使用互斥锁保护标准容器，如 SegmentTree、TableIndexList
4. **混合策略**：结合原子操作和锁，如 BufferPool 的内存计数缓存

```
并发数据结构层次
================

                    高层并发容器
    ┌────────────────────┴────────────────────┐
    │                                          │
    ▼                                          ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ SegmentTree   │  │TableIndexList │  │  BufferPool   │
│(mutex+atomic) │  │   (mutex)     │  │(atomic cache) │
└───────────────┘  └───────────────┘  └───────────────┘
         │                  │                  │
         └──────────────────┼──────────────────┘
                            ▼
                    基础同步原语
    ┌─────────────┬─────────────┬─────────────┐
    │  atomic<T>  │  atomic_ptr │ConcurrentQueue
    │  (无锁)      │  (原子指针)  │  (无锁队列)  │
    └─────────────┴─────────────┴─────────────┘
```

## 9.2 原子类型封装

### 9.2.1 atomic.hpp

DuckDB 对标准库 `std::atomic` 进行了简单的命名空间导入：

```cpp
// src/include/duckdb/common/atomic.hpp
#pragma once
#include <atomic>

namespace duckdb {
using std::atomic;
}
```

使用原子类型的关键准则：

```cpp
// 在循环中使用 compare_exchange_weak（性能更好）
while (!value.compare_exchange_weak(expected, desired)) {
    // 重试
}

// 在非循环中使用 compare_exchange_strong（更可靠）
if (value.compare_exchange_strong(expected, desired)) {
    // 成功
}
```

### 9.2.2 atomic_ptr 线程安全指针

`atomic_ptr` 是一个模板类，封装了原子指针操作：

```cpp
// src/include/duckdb/common/atomic_ptr.hpp
template <class T, bool SAFE = true>
class atomic_ptr {
public:
    atomic_ptr() noexcept : ptr(nullptr) {}
    atomic_ptr(T *ptr_p) : ptr(ptr_p) {}              // 从指针构造
    atomic_ptr(T &ref) : ptr(&ref) {}                  // 从引用构造
    atomic_ptr(const unique_ptr<T> &ptr_p) : ptr(ptr_p.get()) {}
    atomic_ptr(const shared_ptr<T> &ptr_p) : ptr(ptr_p.get()) {}

    void CheckValid(const T *ptr) const {
        if (MemorySafety<SAFE>::ENABLED) {
            return;  // 安全模式下不检查
        }
        if (!ptr) {
            throw InternalException("Attempting to dereference a null pointer");
        }
    }

    T *GetPointer() {
        auto res = ptr.load();  // 原子加载
        CheckValid(res);
        return res;
    }

    void set(T &ref) {
        ptr = &ref;  // 原子存储
    }

    void reset() {
        ptr = nullptr;  // 原子设置为 null
    }

    bool operator==(const atomic_ptr<T> &rhs) const {
        return ptr.load() == rhs.ptr.load();  // 原子比较
    }

private:
    atomic<T *> ptr;  // 底层原子指针
};

// 非安全版本别名
template <typename T>
using unsafe_atomic_ptr = atomic_ptr<T, false>;
```

### 9.2.3 atomic_ptr 使用场景

```
atomic_ptr 线程安全访问
======================

Thread 1                              Thread 2
    │                                     │
    │  atomic_ptr<Node> current;          │
    │                                     │
    │  current.set(node1);  ◀────────────┼──── 原子存储
    │         │                           │
    │         ▼                           │
    │  ┌─────────────┐                    │
    │  │ ptr = node1 │◀───────────────────┼──── current.GetPointer()
    │  └─────────────┘                    │           │
    │         │                           │           ▼
    │  current.set(node2);                │      返回 node1 或 node2
    │         │                           │      （取决于读取时机）
    │         ▼                           │
    │  ┌─────────────┐                    │
    │  │ ptr = node2 │                    │
    │  └─────────────┘                    │

保证：
1. 指针读写是原子的
2. 不会看到部分更新的指针值
3. 但不保证指向对象的内容一致性
```

## 9.3 ConcurrentQueue 无锁队列

### 9.3.1 moodycamel ConcurrentQueue

DuckDB 使用 moodycamel 的 ConcurrentQueue 作为任务队列：

```cpp
// src/include/duckdb/parallel/concurrentqueue.hpp
#ifndef DUCKDB_NO_THREADS
#include "concurrentqueue.h"  // moodycamel 实现
#else
// 单线程环境的简化实现
namespace duckdb_moodycamel {

template <typename T>
class ConcurrentQueue {
private:
    std::queue<T, std::deque<T>> q;

public:
    template <typename U>
    bool enqueue(U &&item) {
        q.push(std::forward<U>(item));
        return true;
    }

    bool try_dequeue(T &item) {
        if (q.empty()) return false;
        item = std::move(q.front());
        q.pop();
        return true;
    }

    size_t size_approx() const {
        return q.size();
    }
};

} // namespace duckdb_moodycamel
#endif
```

### 9.3.2 ProducerToken 优化

```cpp
// moodycamel ProducerToken 减少锁争用
struct ProducerToken {
    template <typename T, typename Traits>
    explicit ProducerToken(ConcurrentQueue<T> &);

    bool valid() const { return true; }
};
```

ProducerToken 的作用：

```
无 ProducerToken                    有 ProducerToken
================                    =================

Producer 1 ─┐                      Producer 1 ─┬─ Token 1 ─┐
Producer 2 ─┼─▶ [全局队列锁]         Producer 2 ─┴─ Token 2 ─┼─▶ [无锁入队]
Producer 3 ─┘     争用激烈          Producer 3 ─── Token 3 ─┘     性能更好

机制：每个 Producer 有自己的 sub-queue，消费者可从多个 sub-queue 取任务
```

### 9.3.3 TaskScheduler 中的使用

```cpp
// src/include/duckdb/parallel/task_scheduler.hpp
class TaskScheduler {
private:
    //! 无锁任务队列
    duckdb_moodycamel::ConcurrentQueue<shared_ptr<Task>> queue;

    void ScheduleTask(ProducerToken &token, shared_ptr<Task> task) {
        queue.enqueue(token, std::move(task));
        semaphore.signal();
    }

    bool TryDequeueTask(shared_ptr<Task> &task) {
        return queue.try_dequeue(task);
    }
};
```

## 9.4 SegmentTree 分段树

### 9.4.1 结构定义

SegmentTree 用于管理列数据的分段，支持按行号快速定位：

```cpp
// src/include/duckdb/storage/table/segment_tree.hpp
template <class T>
struct SegmentNode {
    idx_t row_start;
    shared_ptr<T> node;
#ifndef DUCKDB_R_BUILD
    atomic<SegmentNode<T> *> next;  // 原子 next 指针
#else
    SegmentNode<T> *next;
#endif
    idx_t index;

public:
    optional_ptr<SegmentNode<T>> Next() const {
        return next.load();  // 原子读取
    }

    void SetNext(optional_ptr<SegmentNode<T>> next) {
        this->next = next.get();  // 原子写入
    }
};

template <class T, bool SUPPORTS_LAZY_LOADING = false>
class SegmentTree {
private:
    mutable vector<unique_ptr<SegmentNode<T>>> nodes;
    mutable mutex node_lock;
    mutable atomic<bool> finished_loading;
    idx_t base_row_id;

public:
    SegmentLock Lock() const {
        return SegmentLock(node_lock);
    }
};
```

### 9.4.2 SegmentLock RAII 封装

```cpp
// src/include/duckdb/storage/table/segment_lock.hpp
struct SegmentLock {
    explicit SegmentLock(mutex &lock) : lock(lock) {}

    // 禁用拷贝，启用移动
    SegmentLock(const SegmentLock &) = delete;
    SegmentLock(SegmentLock &&other) noexcept {
        std::swap(lock, other.lock);
    }

    void Release() {
        lock.unlock();
    }

private:
    unique_lock<mutex> lock;
};
```

### 9.4.3 并发访问模式

```
SegmentTree 并发访问
===================

读操作（需要锁）：
┌──────────────────────────────────────────────────────────────┐
│ optional_ptr<SegmentNode<T>> GetSegment(idx_t row_number) {  │
│     auto l = Lock();              // 获取锁                   │
│     return GetSegment(l, row_number);                        │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘

写操作（需要锁）：
┌──────────────────────────────────────────────────────────────┐
│ void AppendSegment(shared_ptr<T> segment) {                  │
│     auto l = Lock();              // 获取锁                   │
│     LoadAllSegments(l);           // 确保所有段加载           │
│     AppendSegmentInternal(l, std::move(segment));            │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘

遍历（无锁访问 next 指针）：
┌──────────────────────────────────────────────────────────────┐
│ optional_ptr<SegmentNode<T>> GetNextSegment(SegmentNode<T>&) │
│ {                                                            │
│     if (finished_loading) {       // 原子检查                 │
│         return node.Next();       // 原子读取 next            │
│     }                                                        │
│     auto l = Lock();              // 需要锁来加载更多         │
│     return GetNextSegment(l, node);                          │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘
```

### 9.4.4 二分查找定位

```cpp
bool TryGetSegmentIndex(SegmentLock &l, idx_t row_number, idx_t &result) const {
    // 按需加载段直到包含目标行
    while (nodes.empty() || row_number >= nodes.back()->GetRowEnd()) {
        if (!LoadNextSegment(l)) break;
    }

    if (nodes.empty()) return false;

    // 二分查找
    idx_t lower = 0;
    idx_t upper = nodes.size() - 1;
    while (lower <= upper) {
        idx_t index = (lower + upper) / 2;
        auto &entry = *nodes[index];
        if (row_number < entry.GetRowStart()) {
            upper = index - 1;
        } else if (row_number >= entry.GetRowEnd()) {
            lower = index + 1;
        } else {
            result = index;
            return true;
        }
    }
    return false;
}
```

## 9.5 TableIndexList 索引列表

### 9.5.1 线程安全索引管理

```cpp
// src/include/duckdb/storage/table/table_index_list.hpp
enum class IndexBindState : uint8_t {
    UNBOUND,   // 未绑定
    BINDING,   // 正在绑定
    BOUND      // 已绑定
};

struct IndexEntry {
    atomic<IndexBindState> bind_state;  // 原子绑定状态
    mutex lock;                          // 条目锁
    unique_ptr<Index> index;
    unique_ptr<BoundIndex> deleted_rows_in_use;
    unique_ptr<BoundIndex> added_data_during_checkpoint;
};

class TableIndexList {
private:
    mutex index_entries_lock;                       // 列表锁
    vector<unique_ptr<IndexEntry>> index_entries;   // 索引条目
    idx_t unbound_count = 0;                        // 未绑定计数

public:
    template <class T>
    void Scan(T &&callback) {
        lock_guard<mutex> lock(index_entries_lock);  // 获取列表锁
        for (auto &entry : index_entries) {
            if (callback(*entry->index)) {
                break;
            }
        }
    }

    bool Empty() {
        lock_guard<mutex> lock(index_entries_lock);
        return index_entries.empty();
    }

    idx_t Count() {
        lock_guard<mutex> lock(index_entries_lock);
        return index_entries.size();
    }
};
```

### 9.5.2 索引绑定状态机

```
IndexBindState 状态转换
======================

         ┌─────────────────────────────────────┐
         │                                      │
         ▼                                      │
    ┌─────────┐   BindIndexes()   ┌─────────┐  │  绑定成功
    │ UNBOUND │──────────────────▶│ BINDING │──┴─────────────▶ BOUND
    └─────────┘                   └─────────┘
                                       │
                                       │ 绑定失败
                                       ▼
                                  ┌─────────┐
                                  │ UNBOUND │
                                  └─────────┘

原子状态检查：
if (entry.bind_state.load() == IndexBindState::UNBOUND) {
    // 尝试绑定
    IndexBindState expected = IndexBindState::UNBOUND;
    if (entry.bind_state.compare_exchange_strong(expected, IndexBindState::BINDING)) {
        // 成功获取绑定权
        try {
            BindIndex(entry);
            entry.bind_state = IndexBindState::BOUND;
        } catch (...) {
            entry.bind_state = IndexBindState::UNBOUND;
            throw;
        }
    }
}
```

## 9.6 BufferPool 缓冲池

### 9.6.1 分布式内存计数

BufferPool 使用分布式计数器缓存减少原子操作争用：

```cpp
// src/include/duckdb/storage/buffer/buffer_pool.hpp
class BufferPool {
protected:
    struct MemoryUsage {
        //! 缓存数量和阈值
        static constexpr idx_t MEMORY_USAGE_CACHE_COUNT = 64;
        static constexpr idx_t MEMORY_USAGE_CACHE_THRESHOLD = 32 << 10;  // 32KB

        //! 全局内存计数器
        using MemoryUsageCounters = array<atomic<int64_t>, MEMORY_TAG_COUNT + 1>;
        MemoryUsageCounters memory_usage;

        //! 每 CPU/线程的缓存计数器
        array<MemoryUsageCounters, MEMORY_USAGE_CACHE_COUNT> memory_usage_caches;

        void UpdateUsedMemory(MemoryTag tag, int64_t size);

        idx_t GetUsedMemory(idx_t index, MemoryUsageCaches cache) {
            if (cache == MemoryUsageCaches::NO_FLUSH) {
                // 不刷新缓存，直接读取（可能不精确）
                auto used_memory = memory_usage[index].load(std::memory_order_relaxed);
                return used_memory > 0 ? static_cast<idx_t>(used_memory) : 0;
            }
            // 刷新所有缓存到全局计数器
            int64_t cached = 0;
            for (auto &cache : memory_usage_caches) {
                cached += cache[index].exchange(0, std::memory_order_relaxed);
            }
            auto used_memory = memory_usage[index].fetch_add(cached, std::memory_order_relaxed) + cached;
            return used_memory > 0 ? static_cast<idx_t>(used_memory) : 0;
        }
    };

    mutable MemoryUsage memory_usage;
    mutex limit_lock;                        // 限制修改锁
    atomic<idx_t> maximum_memory;            // 最大内存
    atomic<idx_t> allocator_bulk_deallocation_flush_threshold;
};
```

### 9.6.2 分布式计数器工作原理

```
分布式内存计数器
===============

                    ┌────────────────────────────────┐
                    │     Global Counter (atomic)    │
                    │     memory_usage[tag]          │
                    └───────────────┬────────────────┘
                                    │
            ┌───────────────────────┼───────────────────────┐
            │                       │                       │
            ▼                       ▼                       ▼
    ┌───────────────┐       ┌───────────────┐       ┌───────────────┐
    │  Cache[0]     │       │  Cache[1]     │       │  Cache[N]     │
    │  (CPU 0)      │       │  (CPU 1)      │       │  (CPU N)      │
    │  local += 4KB │       │  local += 8KB │       │  local += 2KB │
    └───────────────┘       └───────────────┘       └───────────────┘

更新流程：
1. Thread 在其 CPU 对应的 cache 中累加
2. 当 cache 值 > 32KB 时，原子更新到全局计数器
3. 读取时可选择是否刷新缓存

优势：
- 减少全局原子操作的争用
- 接受最多 64 * 32KB = 2MB 的统计误差
- 大幅提升高并发场景性能
```

### 9.6.3 内存更新实现

```cpp
void BufferPool::MemoryUsage::UpdateUsedMemory(MemoryTag tag, int64_t size) {
    // 获取当前 CPU/线程对应的缓存索引
    idx_t cache_idx = GetCurrentCPU() % MEMORY_USAGE_CACHE_COUNT;
    auto &cache = memory_usage_caches[cache_idx];

    // 原子累加到本地缓存
    auto new_value = cache[tag].fetch_add(size, std::memory_order_relaxed) + size;

    // 检查是否需要刷新到全局计数器
    if (std::abs(new_value) >= MEMORY_USAGE_CACHE_THRESHOLD) {
        auto cached = cache[tag].exchange(0, std::memory_order_relaxed);
        memory_usage[tag].fetch_add(cached, std::memory_order_relaxed);

        // 同时更新总计数
        auto total_cached = cache[TOTAL_MEMORY_USAGE_INDEX].exchange(0, std::memory_order_relaxed);
        memory_usage[TOTAL_MEMORY_USAGE_INDEX].fetch_add(total_cached, std::memory_order_relaxed);
    }
}
```

## 9.7 BlockHandle 块句柄

### 9.7.1 原子状态管理

BlockHandle 大量使用原子类型保护状态：

```cpp
// src/include/duckdb/storage/buffer/block_handle.hpp
enum class BlockState : uint8_t {
    BLOCK_UNLOADED = 0,
    BLOCK_LOADED = 1
};

class BlockHandle : public enable_shared_from_this<BlockHandle> {
private:
    mutex lock;                              // 块级别锁
    atomic<BlockState> state;                // 原子状态
    atomic<int32_t> readers;                 // 并发读者计数
    const block_id_t block_id;
    unique_ptr<FileBuffer> buffer;
    atomic<idx_t> eviction_seq_num;          // 驱逐序列号
    atomic<int64_t> lru_timestamp_msec;      // LRU 时间戳
    atomic<DestroyBufferUpon> destroy_buffer_upon;
    atomic<idx_t> memory_usage;              // 内存使用量
    atomic<idx_t> eviction_queue_idx;        // 驱逐队列索引

public:
    BlockLock GetLock() {
        return BlockLock(lock);
    }

    int32_t Readers() const {
        return readers;  // 原子读取
    }

    int32_t DecrementReaders() {
        return --readers;  // 原子递减
    }

    idx_t NextEvictionSequenceNumber() {
        return ++eviction_seq_num;  // 原子递增
    }

    bool CanUnload() const {
        // 无锁检查是否可卸载
        // 注意：结果可能在检查后立即过时
        return state == BlockState::BLOCK_LOADED && readers == 0;
    }
};
```

### 9.7.2 锁与原子的配合

```
BlockHandle 并发访问模式
=======================

高频读取（无锁）：
┌─────────────────────────────────────────┐
│ bool CanUnload() const {                │
│     return state == BLOCK_LOADED &&     │  ◀── 原子读取，无锁
│            readers == 0;                │
│ }                                       │
└─────────────────────────────────────────┘

加载操作（需要锁）：
┌─────────────────────────────────────────┐
│ BufferHandle Load(...) {                │
│     auto l = GetLock();                 │  ◀── 获取互斥锁
│     if (state == BLOCK_LOADED) {        │
│         readers++;                      │  ◀── 原子递增
│         return BufferHandle(...);       │
│     }                                   │
│     // 加载数据到 buffer                 │
│     state = BLOCK_LOADED;               │  ◀── 原子写入
│     readers++;                          │
│     return BufferHandle(...);           │
│ }                                       │
└─────────────────────────────────────────┘

卸载操作（需要锁）：
┌─────────────────────────────────────────┐
│ void Unload(BlockLock &) {              │
│     D_ASSERT(readers == 0);             │  ◀── 断言无读者
│     // 写回数据                          │
│     buffer.reset();                      │
│     state = BLOCK_UNLOADED;             │  ◀── 原子写入
│ }                                       │
└─────────────────────────────────────────┘
```

## 9.8 RowGroupCollection 行组集合

### 9.8.1 原子计数与锁保护

```cpp
// src/include/duckdb/storage/table/row_group_collection.hpp
class RowGroupCollection {
private:
    atomic<idx_t> total_rows;                    // 原子行数
    mutable mutex row_group_pointer_lock;        // 行组指针锁
    shared_ptr<RowGroupSegmentTree> owned_row_groups;
    atomic<idx_t> allocation_size;               // 分配大小

public:
    idx_t GetTotalRows() const {
        return total_rows.load();  // 原子读取
    }

    shared_ptr<RowGroupSegmentTree> GetRowGroups() const {
        lock_guard<mutex> lock(row_group_pointer_lock);  // 获取锁
        return owned_row_groups;
    }

    void SetRowGroups(shared_ptr<RowGroupSegmentTree> row_groups) {
        lock_guard<mutex> lock(row_group_pointer_lock);  // 获取锁
        owned_row_groups = std::move(row_groups);
    }
};
```

### 9.8.2 并行扫描状态

```cpp
// 并行扫描需要同步访问扫描状态
void RowGroupCollection::InitializeParallelScan(ParallelCollectionScanState &state) {
    auto row_groups = GetRowGroups();  // 安全获取行组
    auto l = row_groups->Lock();       // 锁定 SegmentTree
    state.current_row_group = row_groups->GetRootSegment(l);
    state.total_rows = total_rows.load();
    // ...
}

bool RowGroupCollection::NextParallelScan(ClientContext &context,
                                          ParallelCollectionScanState &state,
                                          CollectionScanState &scan_state) {
    auto row_groups = GetRowGroups();
    while (true) {
        // 原子获取并更新扫描位置
        // 使用 compare_exchange 确保无竞争
        // ...
    }
}
```

## 9.9 DataTableInfo 表信息

### 9.9.1 多层锁保护

```cpp
// src/include/duckdb/storage/table/data_table_info.hpp
struct DataTableInfo {
private:
    mutex name_lock;                    // 表名锁
    string schema;
    string table;
    TableIndexList indexes;             // 索引列表（自带锁）
    StorageLock checkpoint_lock;        // 检查点读写锁

public:
    string GetTableName() {
        lock_guard<mutex> lock(name_lock);
        return table;
    }

    void SetTableName(string name) {
        lock_guard<mutex> lock(name_lock);
        table = std::move(name);
    }

    unique_ptr<StorageLockKey> GetSharedLock() {
        return checkpoint_lock.GetSharedLock();
    }

    TableIndexList &GetIndexes() {
        return indexes;  // 索引列表有自己的锁
    }
};
```

### 9.9.2 锁层次结构

```
DataTableInfo 锁层次
===================

Level 0: name_lock
         │
         │ 保护：schema, table
         │
Level 1: checkpoint_lock (StorageLock)
         │
         │ 保护：检查点操作 vs 写入操作
         │
Level 2: indexes.index_entries_lock
         │
         │ 保护：索引列表修改
         │
Level 3: IndexEntry.lock
         │
         │ 保护：单个索引条目

规则：只能按层次顺序获取锁，不能跨层逆序
```

## 9.10 BufferEvictionNode 驱逐节点

### 9.10.1 弱引用避免循环

```cpp
// src/include/duckdb/storage/buffer/buffer_pool.hpp
struct BufferEvictionNode {
    weak_ptr<BlockHandle> handle;     // 弱引用，不延长生命周期
    idx_t handle_sequence_number;      // 序列号用于验证

    bool CanUnload(BlockHandle &handle_p) {
        // 检查 handle 是否仍然有效且可卸载
        return handle_p.EvictionSequenceNumber() == handle_sequence_number &&
               handle_p.CanUnload();
    }

    shared_ptr<BlockHandle> TryGetBlockHandle() {
        return handle.lock();  // 尝试提升为 shared_ptr
    }
};
```

### 9.10.2 驱逐队列设计

```
驱逐队列 + 弱引用
================

BufferPool
    │
    ▼
EvictionQueue (每种 FileBufferType 一个)
    │
    ├── BufferEvictionNode { weak_ptr<BlockHandle>, seq_num }
    ├── BufferEvictionNode { weak_ptr<BlockHandle>, seq_num }
    ├── BufferEvictionNode { weak_ptr<BlockHandle>, seq_num }  ◀── 可能已失效
    └── ...

驱逐流程：
1. 从队列取出 node
2. TryGetBlockHandle() 尝试获取 shared_ptr
   - 如果返回 nullptr：handle 已被释放，跳过
   - 如果返回有效：继续检查
3. CanUnload() 检查序列号和状态
   - 序列号不匹配：handle 被重用，跳过
   - 状态不可卸载：跳过
4. 执行驱逐

优势：
- 队列中的 weak_ptr 不会阻止 BlockHandle 释放
- 序列号防止 ABA 问题
- 惰性清理失效节点
```

## 9.11 同步策略对比

### 9.11.1 策略选择指南

```
同步策略选择
===========

┌──────────────────┬────────────────────┬──────────────────────┐
│      场景         │      策略          │      示例             │
├──────────────────┼────────────────────┼──────────────────────┤
│ 简单计数器        │ atomic<T>         │ readers, total_rows  │
├──────────────────┼────────────────────┼──────────────────────┤
│ 指针交换          │ atomic_ptr        │ SegmentNode.next     │
├──────────────────┼────────────────────┼──────────────────────┤
│ 高并发队列        │ ConcurrentQueue   │ TaskScheduler.queue  │
├──────────────────┼────────────────────┼──────────────────────┤
│ 复杂状态修改      │ mutex + state     │ BlockHandle.Load()   │
├──────────────────┼────────────────────┼──────────────────────┤
│ 读多写少          │ StorageLock       │ checkpoint_lock      │
├──────────────────┼────────────────────┼──────────────────────┤
│ 高频更新计数      │ 分布式缓存         │ BufferPool.MemoryUsage│
├──────────────────┼────────────────────┼──────────────────────┤
│ 容器修改          │ mutex + container │ SegmentTree, TableIndexList│
└──────────────────┴────────────────────┴──────────────────────┘
```

### 9.11.2 性能特征对比

```
性能特征比较
===========

延迟 (低 → 高)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
atomic   atomic_ptr   ConcurrentQueue    mutex      StorageLock
 无锁       无锁         多数无锁         阻塞锁      读写锁

吞吐量 (高 → 低)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
分布式缓存  ConcurrentQueue  atomic   StorageLock(读)  mutex
  最高          高            中          中           低

复杂度 (低 → 高)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
mutex    atomic    StorageLock   ConcurrentQueue   分布式缓存
 简单     简单        中等           复杂            复杂
```

## 9.12 小结

DuckDB 的并发数据结构设计遵循以下原则：

1. **分层同步**：从简单的 atomic 到复杂的分布式缓存，根据需求选择合适的同步策略

2. **无锁优先**：在高频路径上使用 atomic 和 ConcurrentQueue，减少锁争用

3. **读写分离**：使用 StorageLock 区分读写场景，允许并发读

4. **缓存友好**：BufferPool 的分布式计数器利用 CPU 本地缓存减少跨核通信

5. **弱引用协作**：使用 weak_ptr 避免循环引用，配合序列号防止 ABA 问题

6. **RAII 封装**：SegmentLock、BlockLock 等封装确保锁的正确释放

关键数据结构及其同步策略：

| 数据结构 | 同步策略 | 用途 |
|---------|---------|------|
| atomic/atomic_ptr | 无锁 | 简单状态、指针 |
| ConcurrentQueue | 无锁队列 | 任务调度 |
| SegmentTree | mutex + atomic | 段管理 |
| TableIndexList | mutex | 索引管理 |
| BufferPool | 分布式缓存 | 内存统计 |
| BlockHandle | mutex + atomic | 块状态管理 |
