# DuckDB 单文件存储实现机制深度解析

DuckDB 是一款高性能的嵌入式分析型数据库，其存储引擎采用了优雅的单文件设计。本文将深入分析 DuckDB 如何在单个文件中实现完整的数据库存储，包括文件布局、块管理、事务支持、WAL 日志和崩溃恢复等核心机制。

## 一、整体架构概览

DuckDB 存储引擎的核心组件层次如下：

```
StorageManager (顶层管理)
    ├── SingleFileBlockManager (块管理)
    ├── BufferManager (缓冲管理)
    ├── WriteAheadLog (WAL日志)
    └── CheckpointManager (检查点)
```

所有数据库内容都存储在单个 `.duckdb` 文件中，这种设计带来了部署简单、备份方便、原子性保证等优势。

## 二、文件布局结构

### 2.1 整体布局

DuckDB 数据库文件采用固定的布局结构：

```
┌─────────────────────────────────────────────────────────────┐
│  偏移 0:        MainHeader (4KB)                            │
│                 - 魔数 "DUCK" (4字节)                        │
│                 - 存储版本号                                 │
│                 - 标志位 (加密等)                            │
│                 - 数据库唯一标识符                           │
├─────────────────────────────────────────────────────────────┤
│  偏移 4KB:      DatabaseHeader H1 (4KB)                     │
│                 - iteration (迭代计数)                       │
│                 - meta_block (元数据块指针)                  │
│                 - free_list (空闲列表指针)                   │
│                 - block_count (块数量)                       │
├─────────────────────────────────────────────────────────────┤
│  偏移 8KB:      DatabaseHeader H2 (4KB)                     │
│                 - 与 H1 结构相同 (双头部机制)                │
├─────────────────────────────────────────────────────────────┤
│  偏移 12KB:     数据块区域 (BLOCK_START)                    │
│                 - Block 0, Block 1, Block 2, ...            │
│                 - 每块默认 256KB                             │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 关键常量定义

```cpp
// src/include/duckdb/storage/storage_info.hpp
constexpr static idx_t FILE_HEADER_SIZE = 4096U;           // 4KB
constexpr static idx_t DEFAULT_BLOCK_ALLOC_SIZE = 262144;  // 256KB
constexpr static idx_t DEFAULT_BLOCK_HEADER_SIZE = 8;      // 校验和
```

### 2.3 MainHeader 结构

MainHeader 是文件的第一个头部，只在数据库创建时写入一次：

```cpp
class MainHeader {
    static const char MAGIC_BYTES[] = "DUCK";  // 魔数标识
    uint64_t version_number;                    // 存储版本
    uint64_t flags[4];                          // 标志位(加密等)
    data_t library_git_desc[32];                // 创建时的版本描述
    data_t library_git_hash[32];                // Git 哈希
    data_t encryption_metadata[8];              // 加密元数据
    data_t db_identifier[16];                   // 数据库唯一标识
    data_t encrypted_canary[8];                 // 加密验证金丝雀
};
```

### 2.4 DatabaseHeader 结构

DatabaseHeader 记录数据库当前状态，每次检查点时更新：

```cpp
struct DatabaseHeader {
    uint64_t iteration;              // 检查点迭代计数
    idx_t meta_block;                // 元数据块指针
    idx_t free_list;                 // 空闲列表指针
    uint64_t block_count;            // 总块数
    idx_t block_alloc_size;          // 块分配大小
    idx_t vector_size;               // 向量大小
    idx_t serialization_compatibility; // 序列化兼容版本
};
```

## 三、双头部 (Double Header) 机制

这是保证**崩溃恢复**和**原子性**的核心设计。

### 3.1 工作原理

DuckDB 维护两个 DatabaseHeader (H1 和 H2)，交替写入：

```cpp
// src/include/duckdb/storage/single_file_block_manager.hpp
class SingleFileBlockManager {
    uint8_t active_header;  // 当前活动头部: 0 (H1) 或 1 (H2)
    // ...
};
```

**检查点写入流程：**

1. 当前 `active_header = 1` (H2 是活动的)
2. 写入新数据块到文件
3. 写入新的 DatabaseHeader 到 H1 (非活动位置)
4. `fsync()` 确保数据落盘
5. 切换：`active_header = 0` (H1 变为活动)

### 3.2 代码实现

```cpp
// src/storage/single_file_block_manager.cpp:1216-1221
void SingleFileBlockManager::WriteHeader(QueryContext context, DatabaseHeader header) {
    // ... 准备工作 ...

    // 写入到非活动 header 位置
    auto location = active_header == 1 ? Storage::FILE_HEADER_SIZE
                                       : Storage::FILE_HEADER_SIZE * 2;
    ChecksumAndWrite(context, header_buffer, location);

    // 切换活动 header
    active_header = 1 - active_header;

    // 确保 header 写入落盘
    handle->Sync();
}
```

### 3.3 崩溃恢复

启动时的恢复逻辑非常简单：

```cpp
// 读取两个 header
DatabaseHeader h1 = ReadHeader(FILE_HEADER_SIZE);
DatabaseHeader h2 = ReadHeader(FILE_HEADER_SIZE * 2);

// 选择 iteration 更大的作为活动 header
if (h1.iteration > h2.iteration) {
    active_header = 0;
    Initialize(h1);
} else {
    active_header = 1;
    Initialize(h2);
}
```

**崩溃场景分析：**

| 崩溃时机 | 结果 |
|---------|------|
| 写入数据块时崩溃 | 旧 header 仍有效，数据块被忽略 |
| 写入新 header 时崩溃 | 旧 header 仍有效 |
| 切换 active_header 后崩溃 | 新 header 已生效 |

## 四、块管理系统

### 4.1 块分配与释放

```cpp
// src/include/duckdb/storage/single_file_block_manager.hpp
class SingleFileBlockManager {
    set<block_id_t> free_list;          // 可立即重用的空闲块
    set<block_id_t> free_blocks_in_use; // 已释放但仍被引用的块
    set<block_id_t> newly_used_blocks;  // 新使用但未检查点的块
    unordered_set<block_id_t> modified_blocks;  // 等待下次检查点释放的块
    block_id_t max_block;               // 当前最大块ID
};
```

### 4.2 块分配逻辑

```cpp
block_id_t SingleFileBlockManager::GetFreeBlockIdInternal(FreeBlockType type) {
    block_id_t block;
    lock_guard<mutex> lock(block_lock);

    if (!free_list.empty()) {
        // 优先从空闲列表分配
        block = *free_list.begin();
        free_list.erase(free_list.begin());
    } else {
        // 空闲列表为空时，分配新块
        block = max_block++;
    }

    if (type == FreeBlockType::NEWLY_USED_BLOCK) {
        newly_used_blocks.insert(block);
    }
    return block;
}
```

### 4.3 块位置计算

```cpp
idx_t SingleFileBlockManager::GetBlockLocation(block_id_t block_id) const {
    return BLOCK_START + block_id * GetBlockAllocSize();
    // BLOCK_START = 3 * 4KB = 12KB
    // Block 0: 12KB
    // Block 1: 12KB + 256KB = 268KB
    // Block N: 12KB + N * 256KB
}
```

### 4.4 块生命周期

```
新分配 → newly_used_blocks
    ↓ (检查点后)
正常使用中
    ↓ (被修改/释放)
modified_blocks
    ↓ (下次检查点后)
free_list → 可重用
```

## 五、数据完整性保护

### 5.1 校验和机制

每个块都带有校验和，存储在块头部：

```cpp
void SingleFileBlockManager::ChecksumAndWrite(QueryContext context,
                                              FileBuffer &block,
                                              uint64_t location) {
    // 计算校验和
    uint64_t checksum = Checksum(block.buffer, block.Size());

    // 存储到块头部
    Store<uint64_t>(checksum, block.InternalBuffer());

    // 写入磁盘
    block.Write(context, *handle, location);
}

void SingleFileBlockManager::ReadAndChecksum(QueryContext context,
                                             FileBuffer &block,
                                             uint64_t location) {
    block.Read(context, *handle, location);

    uint64_t stored_checksum = Load<uint64_t>(block.InternalBuffer());
    uint64_t computed_checksum = Checksum(block.buffer, block.Size());

    if (stored_checksum != computed_checksum) {
        throw IOException("Corrupt database file: checksum mismatch");
    }
}
```

### 5.2 加密支持

DuckDB 支持 AES-GCM 加密：

```cpp
// MainHeader 中的加密标志
bool MainHeader::IsEncrypted() const {
    return flags[0] & ENCRYPTED_DATABASE_FLAG;
}

// 加密时的金丝雀验证
void EncryptCanary(MainHeader &header, const_data_ptr_t derived_key) {
    // 加密已知明文 "DUCKKEY"
    encryption_state->Process(CANARY, CANARY_BYTE_SIZE,
                              canary_buffer, CANARY_BYTE_SIZE);
    header.SetEncryptedCanary(canary_buffer);
}

// 打开时验证密钥正确性
bool DecryptCanary(MainHeader &header, data_ptr_t derived_key) {
    encryption_state->Process(header.GetEncryptedCanary(), ...);
    return memcmp(decrypted_canary, "DUCKKEY", 8) == 0;
}
```

## 六、WAL (Write-Ahead Log) 实现

### 6.1 WAL 文件格式

```
┌─────────────────────────────────────────────────────────────┐
│  WAL Header                                                  │
│  - WAL_VERSION 标记                                          │
│  - 版本号 (2=普通, 3=加密)                                    │
│  - 数据库标识符                                              │
│  - checkpoint_iteration                                      │
├─────────────────────────────────────────────────────────────┤
│  Entry: [size (8B)] [checksum (8B)] [type + data...]        │
├─────────────────────────────────────────────────────────────┤
│  Entry: [size] [checksum] [type + data...]                  │
├─────────────────────────────────────────────────────────────┤
│  WAL_FLUSH Entry (事务提交边界)                              │
├─────────────────────────────────────────────────────────────┤
│  ...                                                         │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 WAL 条目类型

```cpp
enum class WALType : uint8_t {
    // DDL 操作
    CREATE_TABLE = 1,    DROP_TABLE = 2,
    CREATE_SCHEMA = 3,   DROP_SCHEMA = 4,
    CREATE_VIEW = 5,     DROP_VIEW = 6,
    CREATE_INDEX = 23,   DROP_INDEX = 24,
    ALTER_INFO = 20,

    // DML 操作
    USE_TABLE = 25,      // 设置当前表
    INSERT_TUPLE = 26,   // 插入
    DELETE_TUPLE = 27,   // 删除
    UPDATE_TUPLE = 28,   // 更新
    ROW_GROUP_DATA = 29, // 行组数据

    // 控制标记
    WAL_VERSION = 98,    // WAL 版本头
    CHECKPOINT = 99,     // 检查点标记
    WAL_FLUSH = 100      // 事务提交
};
```

### 6.3 事务提交流程

```cpp
void WriteAheadLog::Flush() {
    // 写入 WAL_FLUSH 条目作为事务边界
    WriteAheadLogSerializer serializer(*this, WALType::WAL_FLUSH);
    serializer.End();

    // 同步到磁盘
    writer->Sync();
}
```

事务提交的 WAL 写入序列：

```
1. WriteSetTable(schema, table)     → USE_TABLE
2. WriteInsert(chunk)               → INSERT_TUPLE
   WriteDelete(chunk)               → DELETE_TUPLE
   WriteUpdate(chunk, columns)      → UPDATE_TUPLE
3. Flush()                          → WAL_FLUSH + fsync
```

### 6.4 WAL 重放

启动时的恢复流程：

```cpp
unique_ptr<WriteAheadLog> WriteAheadLog::Replay(...) {
    // 1. 第一遍扫描：查找检查点标记
    while (!reader.Finished()) {
        auto deserializer = GetEntryDeserializer(state, reader, true);
        deserializer.ReplayEntry();  // deserialize_only = true
    }

    // 2. 检查 checkpoint_iteration 是否匹配
    if (checkpoint_was_successful) {
        return nullptr;  // WAL 已被检查点包含，可删除
    }

    // 3. 第二遍扫描：实际重放
    reader.Reset();
    while (!reader.Finished()) {
        auto deserializer = GetEntryDeserializer(state, reader, false);
        if (deserializer.ReplayEntry()) {
            // WAL_FLUSH → 提交事务
            con.Commit();
            successful_offset = reader.CurrentOffset();
        }
    }
}
```

### 6.5 截断的 WAL 处理

```cpp
try {
    // 重放 WAL 条目
    deserializer.ReplayEntry();
} catch (SerializationException &ex) {
    // 序列化异常 = 截断的 WAL
    // 回滚未完成的事务
    con.Query("ROLLBACK");

    // 截断 WAL 到最后成功位置
    return make_uniq<WriteAheadLog>(..., successful_offset,
                                    WALInitState::UNINITIALIZED_REQUIRES_TRUNCATE);
}
```

## 七、检查点机制

### 7.1 检查点流程

```
WriteHeader() 检查点流程:
┌──────────────────────────────────────────┐
│ 1. 收集空闲列表块                         │
├──────────────────────────────────────────┤
│ 2. 将 modified_blocks 合并到 free_list   │
├──────────────────────────────────────────┤
│ 3. 写入空闲列表到专用块                   │
├──────────────────────────────────────────┤
│ 4. 递增 iteration_count                   │
├──────────────────────────────────────────┤
│ 5. fsync() - 确保数据块落盘               │
├──────────────────────────────────────────┤
│ 6. 写入新 DatabaseHeader 到非活动位置     │
├──────────────────────────────────────────┤
│ 7. 切换 active_header                     │
├──────────────────────────────────────────┤
│ 8. fsync() - 确保 header 落盘             │
├──────────────────────────────────────────┤
│ 9. 截断 WAL                               │
└──────────────────────────────────────────┘
```

### 7.2 WAL 与检查点协调

```cpp
// 检查点开始时写入标记
void WriteAheadLog::WriteCheckpoint(MetaBlockPointer meta_block) {
    WriteAheadLogSerializer serializer(*this, WALType::CHECKPOINT);
    serializer.WriteProperty(101, "meta_block", meta_block);
    serializer.End();
}
```

恢复时的协调逻辑：

| 场景 | checkpoint_iteration | 处理方式 |
|------|---------------------|---------|
| 检查点成功，WAL 未截断 | 匹配 | 跳过 WAL |
| 检查点失败 | 不匹配 | 完整重放 WAL |
| 有 checkpoint_wal | - | 合并后重放 |

## 八、文件空间回收

### 8.1 文件截断

当空闲块在文件末尾时，可以截断文件释放空间：

```cpp
void SingleFileBlockManager::Truncate() {
    idx_t blocks_to_truncate = 0;

    // 从末尾开始找连续的空闲块
    for (auto entry = free_list.rbegin(); entry != free_list.rend(); entry++) {
        if (*entry + 1 != max_block) break;
        blocks_to_truncate++;
        max_block--;
    }

    if (blocks_to_truncate > 0) {
        free_list.erase(free_list.lower_bound(max_block), free_list.end());
        handle->Truncate(BLOCK_START + max_block * GetBlockAllocSize());
    }
}
```

### 8.2 块释放给文件系统 (TRIM)

```cpp
void SingleFileBlockManager::TrimFreeBlocks(const set<block_id_t> &blocks) {
    for (auto itr = blocks.begin(); itr != blocks.end(); ++itr) {
        block_id_t first = *itr;
        block_id_t last = first;

        // 找到连续块范围
        for (++itr; itr != blocks.end() && (*itr == last + 1); ++itr) {
            last = *itr;
        }
        --itr;

        // TRIM 该范围
        handle->Trim(BLOCK_START + first * GetBlockAllocSize(),
                     (last - first + 1) * GetBlockAllocSize());
    }
}
```

## 九、数据存储模型

### 9.1 列式存储结构

```
DataTable (表)
  └── RowGroupCollection (行组集合)
        └── RowGroup (行组 - 一段连续行)
              └── ColumnData (列数据)
                    └── ColumnSegment (列段 - 压缩后的数据块)
```

### 9.2 压缩支持

DuckDB 支持多种压缩算法，根据数据特征自动选择：

- **Dictionary/FSST**: 字典压缩，适合字符串
- **Bitpacking**: 位打包，适合整数
- **RLE**: 游程编码
- **ALP/ALPRD**: 适合数值列
- **Chimp/Patas**: 适合浮点数
- **Roaring**: 位图压缩

## 十、总结

DuckDB 单文件存储的设计体现了以下核心思想：

| 设计目标 | 实现方式 |
|---------|---------|
| **原子性** | 双 DatabaseHeader 交替写入 |
| **持久性** | fsync + 校验和验证 |
| **一致性** | WAL 日志 + 检查点协调 |
| **空间效率** | free_list 管理 + 文件截断 + TRIM |
| **崩溃恢复** | iteration 计数选择有效 header |
| **安全性** | AES-GCM 加密 + 金丝雀验证 |

这种设计使得 DuckDB 在保持高性能的同时，实现了企业级的数据可靠性保证。单文件的设计也极大简化了部署和备份流程，使其成为嵌入式分析场景的理想选择。

## 参考源码

- `src/storage/single_file_block_manager.cpp` - 单文件块管理器
- `src/storage/write_ahead_log.cpp` - WAL 写入逻辑
- `src/storage/wal_replay.cpp` - WAL 重放逻辑
- `src/include/duckdb/storage/storage_info.hpp` - 存储常量和结构定义
