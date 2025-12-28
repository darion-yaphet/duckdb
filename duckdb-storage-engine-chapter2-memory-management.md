# DuckDB 存储引擎深度解析 - 第二章：内存管理

## 2.1 概述

DuckDB 的内存管理系统是存储引擎的核心组件之一，它负责管理数据库运行时的所有内存分配、缓存和换出操作。作为一款面向 OLAP 的嵌入式数据库，DuckDB 需要高效地处理可能超过物理内存大小的数据集，同时保证查询性能。

本章将深入剖析 DuckDB 内存管理的完整架构，包括：
- BufferManager：统一的内存管理接口
- BufferPool：内存池与驱逐策略
- BlockHandle 与 BufferHandle：内存块的抽象
- BlockAllocator：高性能块分配器
- 临时文件管理：内存不足时的磁盘换出
- 内存标签系统：精细化内存追踪

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DuckDB 内存管理架构                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    BufferManager                             │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │   │
│  │  │   Allocate   │  │     Pin      │  │      Unpin       │   │   │
│  │  └──────────────┘  └──────────────┘  └──────────────────┘   │   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                     │
│  ┌────────────────────────────▼────────────────────────────────┐   │
│  │                      BufferPool                              │   │
│  │  ┌─────────────┐  ┌─────────────────┐  ┌────────────────┐   │   │
│  │  │ MemoryUsage │  │ EvictionQueues  │  │MemoryTagCounters│  │   │
│  │  └─────────────┘  └─────────────────┘  └────────────────┘   │   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                     │
│  ┌────────────────────────────▼────────────────────────────────┐   │
│  │                    BlockHandle                               │   │
│  │  ┌────────┐  ┌────────┐  ┌───────────┐  ┌───────────────┐   │   │
│  │  │block_id│  │ state  │  │  buffer   │  │memory_charge  │   │   │
│  │  └────────┘  └────────┘  └───────────┘  └───────────────┘   │   │
│  └────────────────────────────┬────────────────────────────────┘   │
│                               │                                     │
│  ┌────────────────────────────▼────────────────────────────────┐   │
│  │                    FileBuffer                                │   │
│  │  ┌────────────┐  ┌────────────┐  ┌───────────────────────┐  │   │
│  │  │   BLOCK    │  │  MANAGED   │  │     TINY_BUFFER       │  │   │
│  │  └────────────┘  └────────────┘  └───────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │            TemporaryFileManager (磁盘换出)                    │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐    │   │
│  │  │ Write Buffer │  │ Read Buffer  │  │   Compression   │    │   │
│  │  └──────────────┘  └──────────────┘  └─────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2.2 BufferManager 架构

### 2.2.1 抽象基类设计

BufferManager 是 DuckDB 内存管理的统一接口，定义在 `src/include/duckdb/storage/buffer_manager.hpp` 中：

```cpp
class BufferManager {
    friend class BufferHandle;
    friend class BlockHandle;
    friend class BlockManager;

public:
    // 分配临时内存（指定大小）
    virtual shared_ptr<BlockHandle> AllocateTemporaryMemory(
        MemoryTag tag, idx_t block_size, bool can_destroy = true) = 0;

    // 分配块管理器对应大小的内存
    virtual shared_ptr<BlockHandle> AllocateMemory(
        MemoryTag tag, BlockManager *block_manager, bool can_destroy = true) = 0;

    // 分配并固定内存
    virtual BufferHandle Allocate(MemoryTag tag, idx_t block_size,
                                   bool can_destroy = true) = 0;

    // 固定内存块（Pin）
    virtual BufferHandle Pin(shared_ptr<BlockHandle> &handle) = 0;

    // 预取多个块
    virtual void Prefetch(vector<shared_ptr<BlockHandle>> &handles) = 0;

    // 释放固定（Unpin）
    virtual void Unpin(shared_ptr<BlockHandle> &handle) = 0;

    // 重新分配大小
    virtual void ReAllocate(shared_ptr<BlockHandle> &handle, idx_t block_size) = 0;

    // 内存使用查询
    virtual idx_t GetUsedMemory() const = 0;
    virtual idx_t GetMaxMemory() const = 0;
    virtual idx_t GetUsedSwap() const = 0;
    virtual optional_idx GetMaxSwap() const = 0;
};
```

BufferManager 采用 **Pin/Unpin** 语义管理内存：
- **Pin**：固定内存块，防止被驱逐
- **Unpin**：释放固定，允许内存块被驱逐

### 2.2.2 StandardBufferManager 实现

StandardBufferManager 是 BufferManager 的标准实现，位于 `src/include/duckdb/storage/standard_buffer_manager.hpp`：

```cpp
class StandardBufferManager : public BufferManager {
protected:
    // 数据库实例引用
    DatabaseInstance &db;

    // 共享的内存池
    BufferPool &buffer_pool;

    // 临时文件目录配置
    TemporaryFileData temporary_directory;

    // 临时块 ID 生成器（从 MAXIMUM_BLOCK 开始）
    atomic<block_id_t> temporary_id;

    // 缓冲区分配器
    Allocator buffer_allocator;

    // 临时块管理器
    unique_ptr<BlockManager> temp_block_manager;

    // 每种内存标签的驱逐数据统计
    atomic<idx_t> evicted_data_per_tag[MEMORY_TAG_COUNT];
};
```

关键特点：
1. **共享 BufferPool**：多个数据库实例可以共享同一个 BufferPool
2. **临时文件支持**：内存不足时可以将数据换出到磁盘
3. **内存标签追踪**：按类别统计内存使用和驱逐

### 2.2.3 内存分配流程

```cpp
BufferHandle StandardBufferManager::Allocate(MemoryTag tag, idx_t block_size,
                                              bool can_destroy) {
    // 1. 分配临时内存块
    auto block = AllocateTemporaryMemory(tag, block_size, can_destroy);

    // 2. 调试模式下用垃圾数据初始化
#ifdef DUCKDB_DEBUG_DESTROY_BLOCKS
    WriteGarbageIntoBuffer(*block);
#endif

    // 3. 固定并返回
    return Pin(block);
}

shared_ptr<BlockHandle> StandardBufferManager::RegisterMemory(
    MemoryTag tag, idx_t block_size, idx_t block_header_size, bool can_destroy) {

    auto alloc_size = GetAllocSize(block_size + block_header_size);

    // 尝试驱逐块以腾出空间，可能会得到一个可重用的缓冲区
    unique_ptr<FileBuffer> reusable_buffer;
    auto res = EvictBlocksOrThrow(tag, alloc_size, &reusable_buffer,
        "could not allocate block of size %s%s",
        StringUtil::BytesToHumanReadableString(alloc_size));

    // 创建新的缓冲区
    const auto file_buffer_type =
        tag == MemoryTag::EXTERNAL_FILE_CACHE
            ? FileBufferType::EXTERNAL_FILE
            : FileBufferType::MANAGED_BUFFER;

    auto buffer = ConstructManagedBuffer(block_size, block_header_size,
                                          std::move(reusable_buffer), file_buffer_type);

    // 确定驱逐时的处理方式
    const auto destroy_buffer_upon =
        can_destroy ? DestroyBufferUpon::EVICTION : DestroyBufferUpon::BLOCK;

    // 创建 BlockHandle
    return make_shared_ptr<BlockHandle>(
        *temp_block_manager, ++temporary_id, tag,
        std::move(buffer), destroy_buffer_upon,
        alloc_size, std::move(res));
}
```

### 2.2.4 Pin 操作详解

Pin 操作是访问内存块的核心方法：

```cpp
BufferHandle StandardBufferManager::Pin(const QueryContext &context,
                                         shared_ptr<BlockHandle> &handle) {
    BufferHandle buf;
    idx_t required_memory;

    {
        // 1. 锁定块
        auto lock = handle->GetLock();

        // 2. 检查是否已加载
        if (handle->GetState() == BlockState::BLOCK_LOADED) {
            // 已加载，增加读者计数并返回
            buf = handle->Load(context);
        }
        required_memory = handle->GetMemoryUsage();
    }

    if (buf.IsValid()) {
        return buf;  // 块已加载
    }

    // 3. 块未加载，需要驱逐其他块腾出空间
    unique_ptr<FileBuffer> reusable_buffer;
    auto reservation = EvictBlocksOrThrow(handle->GetMemoryTag(), required_memory,
        &reusable_buffer, "failed to pin block of size %s%s",
        StringUtil::BytesToHumanReadableString(required_memory));

    // 4. 再次检查（其他线程可能已加载）
    auto lock = handle->GetLock();
    if (handle->GetState() == BlockState::BLOCK_LOADED) {
        reservation.Resize(0);
        buf = handle->Load(context);
    } else {
        // 5. 真正加载块
        D_ASSERT(handle->Readers() == 0);
        buf = handle->Load(context, std::move(reusable_buffer));
        if (!buf.IsValid()) {
            reservation.Resize(0);
            return buf;  // 块被销毁
        }

        // 6. 转移内存预留
        auto &memory_charge = handle->GetMemoryCharge(lock);
        memory_charge = std::move(reservation);
    }

    return buf;
}
```

### 2.2.5 Unpin 与驱逐队列

```cpp
void StandardBufferManager::Unpin(shared_ptr<BlockHandle> &handle) {
    bool purge = false;
    {
        auto lock = handle->GetLock();

        // 小缓冲区不参与驱逐
        if (!handle->GetBuffer(lock) ||
            handle->GetBufferType() == FileBufferType::TINY_BUFFER) {
            return;
        }

        D_ASSERT(handle->Readers() > 0);
        auto new_readers = handle->DecrementReaders();

        if (new_readers == 0) {
            // 调试模式：验证无读者时的数据完整性
            VerifyZeroReaders(lock, handle);

            if (handle->MustAddToEvictionQueue()) {
                // 添加到驱逐队列
                purge = buffer_pool.AddToEvictionQueue(handle);
            } else {
                // 立即卸载
                handle->Unload(lock);
            }
        }
    }

    // 可能需要清理驱逐队列
    if (purge) {
        PurgeQueue(*handle);
    }
}
```

---

## 2.3 BufferPool 设计

### 2.3.1 核心数据结构

BufferPool 是跨数据库共享的内存池，定义在 `src/include/duckdb/storage/buffer/buffer_pool.hpp`：

```cpp
class BufferPool {
protected:
    // 内存限制锁
    mutex limit_lock;

    // 最大内存限制
    atomic<idx_t> maximum_memory;

    // 批量释放阈值
    atomic<idx_t> allocator_bulk_deallocation_flush_threshold;

    // 是否记录驱逐时间戳
    bool track_eviction_timestamps;

    // 驱逐队列（多个）
    vector<unique_ptr<EvictionQueue>> queues;

    // 临时内存管理器
    unique_ptr<TemporaryMemoryManager> temporary_memory_manager;

    // 内存使用统计（支持缓存优化）
    mutable MemoryUsage memory_usage;

    // 块分配器
    BlockAllocator &block_allocator;

    // 驱逐队列配置
    static constexpr idx_t EVICTION_QUEUE_TYPES = FILE_BUFFER_TYPE_COUNT - 1;
    static constexpr idx_t BLOCK_AND_EXTERNAL_FILE_QUEUE_SIZE = 1;
    static constexpr idx_t MANAGED_BUFFER_QUEUE_SIZE = 6;
    static constexpr idx_t TINY_BUFFER_QUEUE_SIZE = 1;
};
```

### 2.3.2 内存使用统计优化

为了减少多线程竞争，BufferPool 采用了 **CPU 本地缓存** 策略：

```cpp
struct MemoryUsage {
    // 缓存参数
    static constexpr idx_t MEMORY_USAGE_CACHE_COUNT = 64;
    static constexpr idx_t MEMORY_USAGE_CACHE_THRESHOLD = 32 << 10;  // 32KB
    static constexpr idx_t TOTAL_MEMORY_USAGE_INDEX = MEMORY_TAG_COUNT;

    using MemoryUsageCounters = array<atomic<int64_t>, MEMORY_TAG_COUNT + 1>;

    // 全局计数器
    MemoryUsageCounters memory_usage;

    // CPU 本地缓存（64 个槽位）
    array<MemoryUsageCounters, MEMORY_USAGE_CACHE_COUNT> memory_usage_caches;

    void UpdateUsedMemory(MemoryTag tag, int64_t size) {
        auto tag_idx = (idx_t)tag;

        if ((idx_t)AbsValue(size) < MEMORY_USAGE_CACHE_THRESHOLD) {
            // 小更新：使用 CPU 本地缓存
            auto cache_idx = (idx_t)TaskScheduler::GetEstimatedCPUId() %
                             MEMORY_USAGE_CACHE_COUNT;
            auto &cache = memory_usage_caches[cache_idx];

            // 更新本地缓存
            auto new_tag_size = cache[tag_idx].fetch_add(size) + size;

            // 超过阈值时刷新到全局计数器
            if ((idx_t)AbsValue(new_tag_size) >= MEMORY_USAGE_CACHE_THRESHOLD) {
                auto tag_size = cache[tag_idx].exchange(0);
                memory_usage[tag_idx].fetch_add(tag_size);
            }

            // 同样处理总计数
            auto new_total_size = cache[TOTAL_MEMORY_USAGE_INDEX].fetch_add(size) + size;
            if ((idx_t)AbsValue(new_total_size) >= MEMORY_USAGE_CACHE_THRESHOLD) {
                auto total_size = cache[TOTAL_MEMORY_USAGE_INDEX].exchange(0);
                memory_usage[TOTAL_MEMORY_USAGE_INDEX].fetch_add(total_size);
            }
        } else {
            // 大更新：直接更新全局计数器
            memory_usage[tag_idx].fetch_add(size);
            memory_usage[TOTAL_MEMORY_USAGE_INDEX].fetch_add(size);
        }
    }
};
```

这种设计的优势：
1. **减少原子操作竞争**：小更新在本地缓存完成
2. **统计精度可控**：最大误差为 64 * 32KB = 2MB
3. **支持按需刷新**：可以强制刷新获取精确值

### 2.3.3 驱逐队列与 LRU-like 策略

DuckDB 使用多个驱逐队列，按缓冲区类型分类：

```cpp
// 缓冲区类型到驱逐队列的映射
static idx_t FileBufferTypeToEvictionQueueTypeIdx(const FileBufferType &type) {
    switch (type) {
    case FileBufferType::BLOCK:
    case FileBufferType::EXTERNAL_FILE:
        return 0;  // 最先驱逐（释放代价低）
    case FileBufferType::MANAGED_BUFFER:
        return 1;  // 其次驱逐（需要写入临时文件）
    case FileBufferType::TINY_BUFFER:
        return 2;  // 最后驱逐（最后手段）
    default:
        throw InternalException("Unknown FileBufferType");
    }
}
```

驱逐队列的核心是 **并发队列**（使用 moodycamel 的 ConcurrentQueue）：

```cpp
struct EvictionQueue {
    // 并发队列
    eviction_queue_t q;

    // 插入计数（用于触发清理）
    atomic<idx_t> evict_queue_insertions;

    // 死节点计数
    atomic<idx_t> total_dead_nodes;

    // 清理锁
    mutex purge_lock;

    // 触发清理的插入间隔
    constexpr static idx_t INSERT_INTERVAL = 4096;

    bool AddToEvictionQueue(BufferEvictionNode &&node) {
        q.enqueue(std::move(node));
        return ++evict_queue_insertions % INSERT_INTERVAL == 0;
    }
};
```

### 2.3.4 驱逐节点与序列号

驱逐节点使用序列号机制判断有效性：

```cpp
struct BufferEvictionNode {
    weak_ptr<BlockHandle> handle;
    idx_t handle_sequence_number;

    bool CanUnload(BlockHandle &handle_p) {
        // 序列号不匹配说明块在入队后被使用过
        if (handle_sequence_number != handle_p.EvictionSequenceNumber()) {
            return false;
        }
        return handle_p.CanUnload();
    }

    shared_ptr<BlockHandle> TryGetBlockHandle() {
        auto handle_p = handle.lock();
        if (!handle_p) {
            return nullptr;  // BlockHandle 已销毁
        }
        if (!CanUnload(*handle_p)) {
            return nullptr;  // 块在入队后被使用
        }
        return handle_p;
    }
};
```

### 2.3.5 驱逐策略实现

```cpp
BufferPool::EvictionResult BufferPool::EvictBlocksInternal(
    EvictionQueue &queue, MemoryTag tag, idx_t extra_memory,
    idx_t memory_limit, unique_ptr<FileBuffer> *buffer) {

    TempBufferPoolReservation r(tag, *this, extra_memory);
    bool found = false;

    // 检查是否已有足够内存
    if (memory_usage.GetUsedMemory(MemoryUsageCaches::NO_FLUSH) <= memory_limit) {
        if (extra_memory > allocator_bulk_deallocation_flush_threshold) {
            block_allocator.FlushAll(extra_memory);
        }
        return {true, std::move(r)};
    }

    // 遍历可卸载的块
    queue.IterateUnloadableBlocks([&](BufferEvictionNode &,
        const shared_ptr<BlockHandle> &handle, BlockLock &lock) {

        // 检查是否可以重用缓冲区
        if (buffer && handle->GetBuffer(lock)->AllocSize() == extra_memory) {
            // 可以直接重用内存
            *buffer = handle->UnloadAndTakeBlock(lock);
            found = true;
            return false;  // 停止迭代
        }

        // 释放内存
        handle->Unload(lock);

        // 检查是否已足够
        if (memory_usage.GetUsedMemory(MemoryUsageCaches::NO_FLUSH) <= memory_limit) {
            found = true;
            return false;
        }

        return true;  // 继续迭代
    });

    if (!found) {
        r.Resize(0);
    }

    return {found, std::move(r)};
}
```

### 2.3.6 队列清理机制

驱逐队列会积累"死节点"（已失效的驱逐节点），需要定期清理：

```cpp
void EvictionQueue::Purge() {
    // 只允许一个线程清理
    unique_lock<mutex> guard(purge_lock, std::try_to_lock);
    if (!guard.owns_lock()) {
        return;
    }

    idx_t purge_size = INSERT_INTERVAL * PURGE_SIZE_MULTIPLIER;
    idx_t approx_q_size = q.size_approx();

    // 队列不够大时跳过清理
    if (approx_q_size < purge_size * EARLY_OUT_MULTIPLIER) {
        return;
    }

    // 迭代清理
    idx_t max_purges = approx_q_size / purge_size;
    while (max_purges != 0) {
        PurgeIteration(purge_size);

        approx_q_size = q.size_approx();
        if (approx_q_size < purge_size * EARLY_OUT_MULTIPLIER) {
            break;
        }

        // 检查死节点比例
        idx_t approx_dead_nodes = total_dead_nodes;
        idx_t approx_alive_nodes = approx_q_size - approx_dead_nodes;
        if (approx_alive_nodes * (ALIVE_NODE_MULTIPLIER - 1) > approx_dead_nodes) {
            break;
        }

        max_purges--;
    }
}

void EvictionQueue::PurgeIteration(const idx_t purge_size) {
    // 批量出队
    const idx_t actually_dequeued = q.try_dequeue_bulk(
        purge_nodes.begin(), purge_size);

    // 筛选存活节点
    idx_t alive_nodes = 0;
    for (idx_t i = 0; i < actually_dequeued; i++) {
        auto &node = purge_nodes[i];
        auto handle = node.TryGetBlockHandle();
        if (handle) {
            purge_nodes[alive_nodes++] = std::move(node);
        }
    }

    // 重新入队存活节点
    q.enqueue_bulk(purge_nodes.begin(), alive_nodes);

    // 更新死节点计数
    total_dead_nodes -= actually_dequeued - alive_nodes;
}
```

---

## 2.4 BlockHandle 与 BufferHandle

### 2.4.1 BlockHandle：内存块元数据

BlockHandle 管理一个内存块的完整生命周期：

```cpp
class BlockHandle : public enable_shared_from_this<BlockHandle> {
private:
    // 块级锁
    mutex lock;

    // 加载状态
    atomic<BlockState> state;  // BLOCK_UNLOADED 或 BLOCK_LOADED

    // 并发读者计数
    atomic<int32_t> readers;

    // 块 ID
    const block_id_t block_id;

    // 内存标签（用于分类统计）
    const MemoryTag tag;

    // 缓冲区类型
    const FileBufferType buffer_type;

    // 实际数据缓冲区
    unique_ptr<FileBuffer> buffer;

    // 驱逐序列号（用于判断驱逐节点有效性）
    atomic<idx_t> eviction_seq_num;

    // LRU 时间戳
    atomic<int64_t> lru_timestamp_msec;

    // 驱逐时的处理方式
    atomic<DestroyBufferUpon> destroy_buffer_upon;

    // 内存使用量
    atomic<idx_t> memory_usage;

    // 内存预留
    BufferPoolReservation memory_charge;

    // 是否包含内存指针（需要 swizzle）
    const char *unswizzled;

    // 驱逐队列索引
    atomic<idx_t> eviction_queue_idx;

public:
    BlockManager &block_manager;
};
```

### 2.4.2 块状态管理

```cpp
enum class BlockState : uint8_t {
    BLOCK_UNLOADED = 0,  // 未加载（可能在磁盘上）
    BLOCK_LOADED = 1     // 已加载到内存
};

// 判断是否可以卸载
bool BlockHandle::CanUnload() const {
    if (state == BlockState::BLOCK_UNLOADED) {
        return false;  // 已卸载
    }
    if (readers > 0) {
        return false;  // 有活跃读者
    }
    if (block_id >= MAXIMUM_BLOCK && MustWriteToTemporaryFile() &&
        !block_manager.buffer_manager.HasTemporaryDirectory()) {
        // 需要写入临时文件但没有临时目录
        return false;
    }
    return true;
}
```

### 2.4.3 块加载流程

```cpp
BufferHandle BlockHandle::Load(QueryContext context,
                                unique_ptr<FileBuffer> reusable_buffer) {
    if (state == BlockState::BLOCK_LOADED) {
        // 已加载，增加读者计数
        D_ASSERT(buffer);
        ++readers;
        return BufferHandle(shared_from_this(), buffer.get());
    }

    if (block_id < MAXIMUM_BLOCK) {
        // 持久化块：从磁盘读取
        auto block = AllocateBlock(block_manager, std::move(reusable_buffer), block_id);
        block_manager.Read(context, *block);
        buffer = std::move(block);
    } else {
        // 临时块
        if (MustWriteToTemporaryFile()) {
            // 从临时文件读取
            buffer = block_manager.buffer_manager.ReadTemporaryBuffer(
                QueryContext(), tag, *this, std::move(reusable_buffer));
        } else {
            // 块已销毁，无法恢复
            return BufferHandle();
        }
    }

    state = BlockState::BLOCK_LOADED;
    readers = 1;
    return BufferHandle(shared_from_this(), buffer.get());
}
```

### 2.4.4 块卸载流程

```cpp
unique_ptr<FileBuffer> BlockHandle::UnloadAndTakeBlock(BlockLock &lock) {
    VerifyMutex(lock);

    if (state == BlockState::BLOCK_UNLOADED) {
        return nullptr;
    }

    D_ASSERT(!unswizzled);  // 不能有 swizzle 指针
    D_ASSERT(CanUnload());

    if (block_id >= MAXIMUM_BLOCK && MustWriteToTemporaryFile()) {
        // 临时块需要写入临时文件
        block_manager.buffer_manager.WriteTemporaryBuffer(tag, block_id, *buffer);
    }

    // 释放内存预留
    memory_charge.Resize(0);
    state = BlockState::BLOCK_UNLOADED;

    return std::move(buffer);
}
```

### 2.4.5 BufferHandle：RAII 封装

BufferHandle 是访问内存块的 RAII 句柄：

```cpp
class BufferHandle {
public:
    BufferHandle();
    explicit BufferHandle(shared_ptr<BlockHandle> handle,
                          optional_ptr<FileBuffer> node);
    ~BufferHandle();

    // 禁用拷贝
    BufferHandle(const BufferHandle &other) = delete;
    BufferHandle &operator=(const BufferHandle &) = delete;

    // 允许移动
    BufferHandle(BufferHandle &&other) noexcept;
    BufferHandle &operator=(BufferHandle &&) noexcept;

    // 检查有效性
    bool IsValid() const;

    // 获取数据指针
    data_ptr_t Ptr() const {
        D_ASSERT(IsValid());
        return node->buffer;
    }

    // 获取文件缓冲区
    FileBuffer &GetFileBuffer();

    // 销毁句柄
    void Destroy();

    const shared_ptr<BlockHandle> &GetBlockHandle() const {
        return handle;
    }

private:
    shared_ptr<BlockHandle> handle;
    optional_ptr<FileBuffer> node;
};
```

BufferHandle 的析构函数会自动调用 Unpin，这是 RAII 模式的核心。

### 2.4.6 内存预留机制

BufferPoolReservation 用于追踪内存预留：

```cpp
struct BufferPoolReservation {
    MemoryTag tag;
    idx_t size {0};
    BufferPool &pool;

    BufferPoolReservation(MemoryTag tag, BufferPool &pool);
    ~BufferPoolReservation();

    // 禁用拷贝，允许移动
    BufferPoolReservation(const BufferPoolReservation &) = delete;
    BufferPoolReservation(BufferPoolReservation &&) noexcept;

    void Resize(idx_t new_size);
    void Merge(BufferPoolReservation src);
};
```

内存预留确保在分配内存前已经有足够空间，避免分配后再发现内存不足的情况。

---

## 2.5 BlockAllocator 块分配器

### 2.5.1 设计目标

BlockAllocator 是 DuckDB 的高性能内存分配器，专门用于固定大小块的分配：

```cpp
class BlockAllocator {
private:
    // 唯一标识符
    const hugeint_t uuid;

    // 回退分配器
    Allocator &allocator;

    // 块大小（必须是 2 的幂）
    const idx_t block_size;
    const idx_t block_size_div_shift;  // 用于快速除法

    // 虚拟内存区域
    const idx_t virtual_memory_size;
    const data_ptr_t virtual_memory_space;

    // 物理内存大小（可增长）
    mutex physical_memory_lock;
    atomic<idx_t> physical_memory_size;

    // 未触碰的块 ID 队列
    unsafe_unique_ptr<BlockQueue> untouched;

    // 已触碰的块 ID 队列
    unsafe_unique_ptr<BlockQueue> touched;
};
```

### 2.5.2 虚拟内存预留

BlockAllocator 使用操作系统的虚拟内存机制：

```cpp
static data_ptr_t AllocateVirtualMemory(const idx_t size) {
#if INTPTR_MAX == INT32_MAX
    return nullptr;  // 32 位系统禁用
#endif

#if defined(_WIN32)
    // Windows: MEM_RESERVE 预留虚拟地址空间
    return data_ptr_t(VirtualAlloc(nullptr, size, MEM_RESERVE, PAGE_NOACCESS));
#else
    // Unix: mmap 匿名映射
    const auto ptr = mmap(nullptr, size, PROT_READ | PROT_WRITE,
                          MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    return ptr == MAP_FAILED ? nullptr : data_ptr_cast(ptr);
#endif
}
```

这种设计的优势：
1. **地址空间预留**：预留大量虚拟地址空间，但不立即占用物理内存
2. **按需分配**：只有真正使用时才分配物理内存
3. **连续地址**：所有块在虚拟地址空间中连续，简化管理

### 2.5.3 线程本地状态

为了减少锁竞争，每个线程维护本地的块缓存：

```cpp
class BlockAllocatorThreadLocalState {
private:
    hugeint_t cached_uuid;
    optional_ptr<const BlockAllocator> block_allocator;

    static constexpr idx_t BATCH_SIZE = 128;
    static constexpr idx_t FREE_THRESHOLD = BATCH_SIZE * 2;

    // 未触碰的块（首次分配需要初始化物理内存）
    vector<uint32_t> untouched;

    // 已触碰的块（可直接重用）
    vector<uint32_t> touched;

public:
    data_ptr_t Allocate() {
        // 1. 优先使用本地已触碰的块
        auto pointer = TryAllocateFromLocal();
        if (pointer) {
            return pointer;
        }

        // 2. 尝试从全局队列批量获取
        if (TryGetBatch(touched, *block_allocator->touched) ||
            TryGetBatch(untouched, *block_allocator->untouched)) {
            pointer = TryAllocateFromLocal();
            D_ASSERT(pointer);
            return pointer;
        }

        // 3. 回退到系统分配器
        return block_allocator->allocator.AllocateData(block_allocator->block_size);
    }

    void Free(const data_ptr_t pointer) {
        touched.push_back(block_allocator->GetBlockID(pointer));

        // 达到阈值时批量归还
        if (touched.size() < FREE_THRESHOLD) {
            return;
        }

        std::sort(touched.begin(), touched.end());
        block_allocator->touched->q.enqueue_bulk(
            touched.end() - BATCH_SIZE, BATCH_SIZE);
        touched.resize(touched.size() - BATCH_SIZE);
    }
};
```

### 2.5.4 首次分配与释放

```cpp
static void OnFirstAllocation(const data_ptr_t pointer, const idx_t size) {
#if defined(_WIN32)
    // Windows: 提交物理内存
    VirtualAlloc(pointer, size, MEM_COMMIT, PAGE_READWRITE);
#elif defined(__APPLE__)
    // macOS: 无需特殊处理
#else
    // Linux: 预触碰内存页
    for (idx_t i = 0; i < size; i += 4096) {
        pointer[i] = 0;
    }
#endif
}

static void OnDeallocation(const data_ptr_t pointer, const idx_t size) {
#if defined(_WIN32)
    // Windows: 释放物理内存但保留虚拟地址
    VirtualFree(pointer, size, MEM_DECOMMIT);
#elif defined(__APPLE__)
    // macOS: 标记为可重用
    madvise(pointer, size, MADV_FREE_REUSABLE);
#else
    // Linux: 告知内核不再需要这些页
    madvise(pointer, size, MADV_DONTNEED);
#endif
}
```

### 2.5.5 批量释放与合并

```cpp
void BlockAllocator::FreeInternal(const idx_t extra_memory) const {
    auto count = DivBlockSize(extra_memory);
    unsafe_vector<uint32_t> to_free_buffer;
    to_free_buffer.resize(count);

    count = touched->q.try_dequeue_bulk(to_free_buffer.begin(), count);
    if (count == 0) {
        return;
    }
    to_free_buffer.resize(count);

    // 排序以便合并连续块
    std::sort(to_free_buffer.begin(), to_free_buffer.end());

    // 合并连续块并批量释放
    uint32_t block_id_start = to_free_buffer[0];
    for (idx_t i = 1; i < to_free_buffer.size(); i++) {
        const auto &previous_block_id = to_free_buffer[i - 1];
        const auto &current_block_id = to_free_buffer[i];

        if (previous_block_id == current_block_id - 1) {
            continue;  // 连续块，继续合并
        }

        // 释放之前的连续段
        FreeContiguousBlocks(block_id_start, previous_block_id);
        block_id_start = current_block_id;
    }

    // 释放最后一段
    FreeContiguousBlocks(block_id_start, to_free_buffer.back());

    // 将块 ID 移回未触碰队列
    untouched->q.enqueue_bulk(to_free_buffer.begin(), to_free_buffer.size());
}
```

---

## 2.6 临时文件管理

### 2.6.1 TemporaryFileManager 架构

当内存不足时，DuckDB 将数据换出到临时文件：

```cpp
class TemporaryFileManager {
private:
    DatabaseInstance &db;
    string temp_directory;

    // 管理锁
    mutex manager_lock;

    // 临时文件映射
    TemporaryFileMap files;

    // block_id -> 文件位置映射
    unordered_map<block_id_t, TemporaryFileIndex> used_blocks;

    // 按缓冲区大小分组的索引管理器
    unordered_map<TemporaryBufferSize, BlockIndexManager, EnumClassHash> index_managers;

    // 磁盘使用量
    atomic<idx_t> &size_on_disk;

    // 最大交换空间
    idx_t max_swap_space;

    // 压缩自适应（64 个实例减少竞争）
    static constexpr idx_t COMPRESSION_ADAPTIVITIES = 64;
    array<TemporaryFileCompressionAdaptivity, COMPRESSION_ADAPTIVITIES> compression_adaptivities;
};
```

### 2.6.2 临时缓冲区大小分级

DuckDB 支持多种临时缓冲区大小，以适应不同压缩率：

```cpp
enum class TemporaryBufferSize : uint64_t {
    INVALID = 0,
    S32K = 32768,     // 32KB
    S64K = 65536,     // 64KB
    S96K = 98304,     // 96KB
    S128K = 131072,   // 128KB
    S160K = 163840,   // 160KB
    S192K = 196608,   // 192KB
    S224K = 229376,   // 224KB
    DEFAULT = DEFAULT_BLOCK_ALLOC_SIZE,  // 256KB（不压缩）
};

static constexpr uint64_t TEMPORARY_BUFFER_SIZE_GRANULARITY = 32ULL * 1024ULL;
```

### 2.6.3 自适应压缩

DuckDB 使用 ZSTD 压缩临时文件，并动态调整压缩级别：

```cpp
class TemporaryFileCompressionAdaptivity {
private:
    static constexpr int64_t INITIAL_NS = 50000;
    static constexpr idx_t LEVELS = 6;
    static constexpr double DURATION_RATIO_THRESHOLD = 2.0;
    static constexpr double COMPRESSION_DEVIATION = 0.5;
    static constexpr int64_t WEIGHT = 16;

    RandomEngine random_engine;
    int64_t last_uncompressed_write_ns;
    int64_t last_compressed_writes_ns[LEVELS];

public:
    TemporaryCompressionLevel GetCompressionLevel() {
        // 找到写入时间最短的压缩级别
        idx_t min_compression_idx = 0;
        auto min_compressed_time = last_compressed_writes_ns[0];
        for (idx_t i = 1; i < LEVELS; i++) {
            if (last_compressed_writes_ns[i] < min_compressed_time) {
                min_compression_idx = i;
                min_compressed_time = last_compressed_writes_ns[i];
            }
        }

        auto level = IndexToLevel(min_compression_idx);
        double ratio = double(min_compressed_time) / double(last_uncompressed_write_ns);
        bool should_compress = ratio < DURATION_RATIO_THRESHOLD;

        // 偶尔随机偏离，探索其他选项
        bool should_deviate = random_engine.NextRandom() < COMPRESSION_DEVIATION;

        // ... 决策逻辑
        return result;
    }

    void Update(TemporaryCompressionLevel level, int64_t time_before_ns) {
        auto duration = GetCurrentTimeNanos() - time_before_ns;
        auto &last_write_ns = level == TemporaryCompressionLevel::UNCOMPRESSED
            ? last_uncompressed_write_ns
            : last_compressed_writes_ns[LevelToIndex(level)];

        // 指数移动平均
        last_write_ns = (last_write_ns * (WEIGHT - 1) + duration) / WEIGHT;
    }
};
```

压缩级别范围：
```cpp
enum class TemporaryCompressionLevel : int {
    ZSTD_MINUS_FIVE = -5,   // 最快压缩
    ZSTD_MINUS_THREE = -3,
    ZSTD_MINUS_ONE = -1,
    UNCOMPRESSED = 0,       // 不压缩
    ZSTD_ONE = 1,
    ZSTD_THREE = 3,
    ZSTD_FIVE = 5,          // 最高压缩率
};
```

### 2.6.4 写入临时缓冲区

```cpp
idx_t TemporaryFileManager::WriteTemporaryBuffer(block_id_t block_id, FileBuffer &buffer) {
    D_ASSERT(buffer.AllocSize() == BufferManager::GetBufferManager(db).GetBlockAllocSize());

    auto header_size = buffer.GetHeaderSize();
    const auto adaptivity_idx = TaskScheduler::GetEstimatedCPUId() % COMPRESSION_ADAPTIVITIES;
    auto &compression_adaptivity = compression_adaptivities[adaptivity_idx];

    // 1. 尝试压缩
    const auto time_before_ns = TemporaryFileCompressionAdaptivity::GetCurrentTimeNanos();
    AllocatedData compressed_buffer;
    const auto compression_result = CompressBuffer(compression_adaptivity, buffer, compressed_buffer);

    TemporaryFileIndex index;
    optional_ptr<TemporaryFileHandle> handle;

    {
        TemporaryFileManagerLock lock(manager_lock);

        // 2. 查找可用的现有文件
        for (auto &entry : files.GetMapForSize(compression_result.size)) {
            index = entry.second->TryGetBlockIndex(header_size);
            if (index.IsValid()) {
                handle = entry.second.get();
                break;
            }
        }

        // 3. 没有可用文件时创建新文件
        if (!handle) {
            auto &size = compression_result.size;
            const TemporaryFileIdentifier identifier(db, size,
                index_managers[size].GetNewBlockIndex(size), IsEncrypted());
            auto &new_file = files.CreateFile(identifier);
            index = new_file.TryGetBlockIndex(header_size);
            handle = &new_file;
        }

        // 4. 记录块位置
        used_blocks[block_id] = index;
    }

    // 5. 写入文件（可能加密）
    handle->WriteTemporaryBuffer(buffer, index.block_index.GetIndex(), compressed_buffer);

    // 6. 更新压缩统计
    compression_adaptivity.Update(compression_result.level, time_before_ns);

    return static_cast<idx_t>(compression_result.size);
}
```

### 2.6.5 读取临时缓冲区

```cpp
unique_ptr<FileBuffer> TemporaryFileManager::ReadTemporaryBuffer(
    QueryContext context, block_id_t id, unique_ptr<FileBuffer> reusable_buffer) {

    TemporaryFileIndex index;
    optional_ptr<TemporaryFileHandle> handle;

    {
        TemporaryFileManagerLock lock(manager_lock);
        index = GetTempBlockIndex(lock, id);
        handle = GetFileHandle(lock, index.identifier);
    }

    // 读取并解压
    auto buffer = handle->ReadTemporaryBuffer(context, index, std::move(reusable_buffer));

    {
        // 删除块记录
        TemporaryFileManagerLock lock(manager_lock);
        EraseUsedBlock(lock, id, *handle, index);
    }

    return buffer;
}
```

### 2.6.6 TemporaryFileHandle 实现

```cpp
unique_ptr<FileBuffer> TemporaryFileHandle::ReadTemporaryBuffer(
    QueryContext context, const TemporaryFileIndex &index_in_file,
    unique_ptr<FileBuffer> reusable_buffer) const {

    auto &buffer_manager = BufferManager::GetBufferManager(db);
    auto block_index = index_in_file.block_index.GetIndex();
    auto block_header_size = index_in_file.block_header_size.GetIndex();

    // 分配缓冲区
    auto buffer = buffer_manager.ConstructManagedBuffer(
        buffer_manager.GetBlockAllocSize() - block_header_size,
        block_header_size, std::move(reusable_buffer));

    bool is_uncompressed = identifier.size == TemporaryBufferSize::DEFAULT;

    AllocatedData compressed_buffer;
    data_ptr_t read_buffer;
    idx_t read_size;

    if (is_uncompressed) {
        read_buffer = buffer->InternalBuffer();
        read_size = buffer->AllocSize();
    } else {
        // 压缩数据：先读入压缩缓冲区
        compressed_buffer = Allocator::Get(db).Allocate(
            TemporaryBufferSizeToSize(identifier.size));
        read_buffer = compressed_buffer.get();
        read_size = compressed_buffer.GetSize();
    }

    idx_t read_position = GetPositionInFile(block_index);

    if (IsEncrypted()) {
        // 读取加密元数据并解密
        uint8_t encryption_metadata[DEFAULT_ENCRYPTED_BUFFER_HEADER_SIZE];
        handle->Read(context, encryption_metadata, DEFAULT_ENCRYPTED_BUFFER_HEADER_SIZE, read_position);
        handle->Read(context, read_buffer, read_size,
                     read_position + DEFAULT_ENCRYPTED_BUFFER_HEADER_SIZE);
        EncryptionEngine::DecryptTemporaryBuffer(db, read_buffer, read_size, encryption_metadata);
    } else {
        handle->Read(context, read_buffer, read_size, read_position);
    }

    if (is_uncompressed) {
        return buffer;
    }

    // 解压
    const auto compressed_size = Load<idx_t>(compressed_buffer.get());
    duckdb_zstd::ZSTD_decompress(
        buffer->InternalBuffer(), buffer->AllocSize(),
        compressed_buffer.get() + sizeof(idx_t), compressed_size);

    return buffer;
}
```

---

## 2.7 TemporaryMemoryManager

### 2.7.1 并发内存协调

TemporaryMemoryManager 协调多个查询的临时内存使用：

```cpp
class TemporaryMemoryManager {
private:
    // 最小预留保证
    static constexpr idx_t MINIMUM_RESERVATION_PER_STATE_PER_THREAD =
        512ULL * DEFAULT_BLOCK_ALLOC_SIZE;  // 每线程 128MB
    static constexpr idx_t MINIMUM_RESERVATION_MEMORY_LIMIT_DIVISOR = 16ULL;

    // 最大预留比例
    static constexpr double MAXIMUM_MEMORY_LIMIT_RATIO = 0.9;
    static constexpr double MAXIMUM_FREE_MEMORY_RATIO = 0.9;

    mutex lock;

    // 配置参数
    idx_t memory_limit = DConstants::INVALID_INDEX;
    bool has_temporary_directory = false;
    idx_t num_threads = DConstants::INVALID_INDEX;
    idx_t num_connections = DConstants::INVALID_INDEX;
    idx_t query_max_memory = DConstants::INVALID_INDEX;

    // 活跃状态
    reference_set_t<TemporaryMemoryState> active_states;
    idx_t reservation;      // 总预留
    idx_t remaining_size;   // 总剩余需求
};
```

### 2.7.2 TemporaryMemoryState

每个需要临时内存的操作创建一个状态：

```cpp
class TemporaryMemoryState {
private:
    TemporaryMemoryManager &temporary_memory_manager;

    atomic<idx_t> remaining_size;      // 剩余需求
    atomic<idx_t> minimum_reservation; // 最小预留
    atomic<idx_t> reservation;         // 当前预留
    atomic<idx_t> materialization_penalty; // 物化惩罚权重

public:
    // 设置剩余需求并更新预留
    void SetRemainingSizeAndUpdateReservation(ClientContext &context,
                                               idx_t new_remaining_size);

    // 获取预留量
    idx_t GetReservation() const;

    // 设置物化惩罚
    void SetMaterializationPenalty(idx_t new_materialization_penalty);
};
```

---

## 2.8 内存标签系统

### 2.8.1 标签定义

DuckDB 使用内存标签进行精细化追踪：

```cpp
enum class MemoryTag : uint8_t {
    BASE_TABLE = 0,          // 基表数据
    HASH_TABLE = 1,          // 哈希表（Join、Aggregate）
    PARQUET_READER = 2,      // Parquet 读取器
    CSV_READER = 3,          // CSV 读取器
    ORDER_BY = 4,            // 排序
    ART_INDEX = 5,           // ART 索引
    COLUMN_DATA = 6,         // 列数据
    METADATA = 7,            // 元数据
    OVERFLOW_STRINGS = 8,    // 溢出字符串
    IN_MEMORY_TABLE = 9,     // 内存表
    ALLOCATOR = 10,          // 分配器
    EXTENSION = 11,          // 扩展
    TRANSACTION = 12,        // 事务
    EXTERNAL_FILE_CACHE = 13,// 外部文件缓存
    WINDOW = 14              // 窗口函数
};

static constexpr const idx_t MEMORY_TAG_COUNT = 15;
```

### 2.8.2 内存信息查询

```cpp
vector<MemoryInformation> StandardBufferManager::GetMemoryUsageInfo() const {
    vector<MemoryInformation> result;
    for (idx_t k = 0; k < MEMORY_TAG_COUNT; k++) {
        MemoryInformation info;
        info.tag = MemoryTag(k);
        info.size = buffer_pool.memory_usage.GetUsedMemory(
            MemoryTag(k), BufferPool::MemoryUsageCaches::FLUSH);
        info.evicted_data = evicted_data_per_tag[k].load();
        result.push_back(info);
    }
    return result;
}
```

这允许用户查询每种类型的内存使用情况：
```sql
SELECT * FROM duckdb_memory();
```

---

## 2.9 FileBuffer 类型

### 2.9.1 缓冲区类型分类

```cpp
enum class FileBufferType : uint8_t {
    BLOCK = 1,           // 磁盘块（持久化数据）
    MANAGED_BUFFER = 2,  // 托管缓冲区（临时数据）
    TINY_BUFFER = 3,     // 小缓冲区（小于块大小）
    EXTERNAL_FILE = 4    // 外部文件缓存
};
```

不同类型的驱逐策略：
- **BLOCK/EXTERNAL_FILE**：最先驱逐（释放代价低，只需释放内存）
- **MANAGED_BUFFER**：其次驱逐（可能需要写入临时文件）
- **TINY_BUFFER**：最后驱逐（通常太小，驱逐收益有限）

### 2.9.2 DestroyBufferUpon 策略

```cpp
enum class DestroyBufferUpon : uint8_t {
    BLOCK,     // 销毁块时（写入临时文件）
    EVICTION,  // 驱逐时销毁（不保留）
    UNPIN      // Unpin 时立即销毁
};
```

---

## 2.10 预取机制

### 2.10.1 批量读取优化

StandardBufferManager 支持批量预取以提高 I/O 效率：

```cpp
void StandardBufferManager::Prefetch(vector<shared_ptr<BlockHandle>> &handles) {
    // 1. 筛选需要加载的块
    map<block_id_t, idx_t> to_be_loaded;
    for (idx_t block_idx = 0; block_idx < handles.size(); block_idx++) {
        auto &handle = handles[block_idx];
        if (handle->GetState() != BlockState::BLOCK_LOADED) {
            to_be_loaded.insert(make_pair(handle->BlockId(), block_idx));
        }
    }

    if (to_be_loaded.empty()) {
        return;
    }

    // 2. 识别连续块范围并批量读取
    block_id_t first_block = -1;
    block_id_t previous_block_id = -1;

    for (auto &entry : to_be_loaded) {
        if (previous_block_id < 0) {
            first_block = entry.first;
            previous_block_id = first_block;
        } else if (previous_block_id + 1 == entry.first) {
            // 连续块
            previous_block_id = entry.first;
        } else {
            // 不连续，执行之前的批量读取
            BatchRead(handles, to_be_loaded, first_block, previous_block_id);
            first_block = entry.first;
            previous_block_id = entry.first;
        }
    }

    // 3. 最后一批
    BatchRead(handles, to_be_loaded, first_block, previous_block_id);
}
```

### 2.10.2 批量读取实现

```cpp
void StandardBufferManager::BatchRead(
    vector<shared_ptr<BlockHandle>> &handles,
    const map<block_id_t, idx_t> &load_map,
    block_id_t first_block, block_id_t last_block) {

    auto &block_manager = handles[0]->block_manager;
    idx_t block_count = NumericCast<idx_t>(last_block - first_block + 1);

    // 分配批量读取缓冲区
    auto total_block_size = block_count * block_manager.GetBlockAllocSize();
    auto batch_memory = RegisterMemory(MemoryTag::BASE_TABLE, total_block_size, 0, true);
    auto intermediate_buffer = Pin(batch_memory);

    // 执行批量读取
    block_manager.ReadBlocks(intermediate_buffer.GetFileBuffer(),
                             first_block, block_count);

    // 分配到各个块
    for (idx_t block_idx = 0; block_idx < block_count; block_idx++) {
        block_id_t block_id = first_block + NumericCast<block_id_t>(block_idx);
        auto entry = load_map.find(block_id);
        auto &handle = handles[entry->second];

        // 为块预留内存
        idx_t required_memory = handle->GetMemoryUsage();
        unique_ptr<FileBuffer> reusable_buffer;
        auto reservation = EvictBlocksOrThrow(
            handle->GetMemoryTag(), required_memory, &reusable_buffer,
            "failed to pin block of size %s%s",
            StringUtil::BytesToHumanReadableString(required_memory));

        // 从批量缓冲区加载
        {
            auto lock = handle->GetLock();
            if (handle->GetState() == BlockState::BLOCK_LOADED) {
                reservation.Resize(0);
                continue;
            }
            auto block_ptr = intermediate_buffer.GetFileBuffer().InternalBuffer() +
                             block_idx * block_manager.GetBlockAllocSize();
            handle->LoadFromBuffer(lock, block_ptr,
                                   std::move(reusable_buffer), std::move(reservation));
        }
    }
}
```

---

## 2.11 内存限制与配置

### 2.11.1 设置内存限制

```cpp
void BufferPool::SetLimit(idx_t limit, const char *exception_postscript) {
    lock_guard<mutex> l_lock(limit_lock);

    // 尝试驱逐以满足新限制
    if (!EvictBlocks(MemoryTag::EXTENSION, 0, limit).success) {
        throw OutOfMemoryException(
            "Failed to change memory limit to %lld: "
            "could not free up enough memory for the new limit%s",
            limit, exception_postscript);
    }

    idx_t old_limit = maximum_memory;
    maximum_memory = limit;

    // 再次尝试
    if (!EvictBlocks(MemoryTag::EXTENSION, 0, limit).success) {
        maximum_memory = old_limit;
        throw OutOfMemoryException(...);
    }

    block_allocator.FlushAll();
}
```

### 2.11.2 交换空间限制

```cpp
void TemporaryFileManager::SetMaxSwapSpace(optional_idx limit) {
    idx_t new_limit;
    if (limit.IsValid()) {
        new_limit = limit.GetIndex();
    } else {
        // 默认使用 90% 的可用磁盘空间
        new_limit = GetDefaultMax(temp_directory);
    }

    auto current_size_on_disk = GetTotalUsedSpaceInBytes();
    if (current_size_on_disk > new_limit) {
        throw OutOfMemoryException(
            "failed to adjust the 'max_temp_directory_size', "
            "currently used space (%s) exceeds the new limit (%s)",
            StringUtil::BytesToHumanReadableString(current_size_on_disk),
            StringUtil::BytesToHumanReadableString(new_limit));
    }

    max_swap_space = new_limit;
}
```

---

## 2.12 小结

DuckDB 的内存管理系统是一个精心设计的多层架构：

### 核心设计原则

1. **分层抽象**
   - BufferManager：统一接口
   - BufferPool：共享资源池
   - BlockHandle：块级管理
   - FileBuffer：底层存储

2. **Pin/Unpin 语义**
   - RAII 风格的资源管理
   - 自动的生命周期追踪
   - 并发安全的引用计数

3. **高效驱逐策略**
   - 多队列按类型优先级驱逐
   - 序列号机制避免 ABA 问题
   - 定期清理死节点

4. **性能优化**
   - CPU 本地缓存减少竞争
   - BlockAllocator 虚拟内存池
   - 批量预取与 I/O 合并

5. **弹性扩展**
   - 临时文件换出
   - 自适应压缩
   - 可配置的内存和交换空间限制

6. **可观测性**
   - 内存标签分类统计
   - 驱逐数据追踪
   - 详细的内存使用信息

### 关键源文件

| 组件 | 头文件 | 实现文件 |
|------|--------|----------|
| BufferManager | `storage/buffer_manager.hpp` | - |
| StandardBufferManager | `storage/standard_buffer_manager.hpp` | `storage/standard_buffer_manager.cpp` |
| BufferPool | `storage/buffer/buffer_pool.hpp` | `storage/buffer/buffer_pool.cpp` |
| BlockHandle | `storage/buffer/block_handle.hpp` | `storage/buffer/block_handle.cpp` |
| BufferHandle | `storage/buffer/buffer_handle.hpp` | - |
| BlockAllocator | `storage/block_allocator.hpp` | `storage/block_allocator.cpp` |
| TemporaryFileManager | `storage/temporary_file_manager.hpp` | `storage/temporary_file_manager.cpp` |
| TemporaryMemoryManager | `storage/temporary_memory_manager.hpp` | `storage/temporary_memory_manager.cpp` |
| MemoryTag | `common/enums/memory_tag.hpp` | - |

---

下一章，我们将深入探讨 DuckDB 的数据组织结构，包括 DataTable、RowGroup 和 ColumnData 的设计与实现。
