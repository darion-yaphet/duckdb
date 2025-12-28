# DuckDB 存储引擎深度解析：第一章 - 存储架构总览

## 概述

DuckDB 采用单文件存储架构（Single-File Storage），将所有数据库对象（表、索引、元数据）存储在一个 `.duckdb` 文件中。这种设计简化了部署和备份，同时通过精心设计的内存管理和块分配策略实现了高性能的分析型工作负载。

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DuckDB 存储架构总览                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        应用层 / SQL 引擎                             │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                       StorageManager                                │   │
│   │   ┌───────────────┐  ┌───────────────┐  ┌───────────────────────┐   │   │
│   │   │ BlockManager  │  │ TableIOManager│  │ WriteAheadLog (WAL)  │   │   │
│   │   └───────────────┘  └───────────────┘  └───────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        BufferManager                                │   │
│   │   ┌───────────────────────────────────────────────────────────┐     │   │
│   │   │  BufferPool (LRU 缓存)                                    │     │   │
│   │   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐         │     │   │
│   │   │   │ Block 0 │ │ Block 1 │ │ Block 2 │ │  ...    │         │     │   │
│   │   │   └─────────┘ └─────────┘ └─────────┘ └─────────┘         │     │   │
│   │   └───────────────────────────────────────────────────────────┘     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                     │                                       │
│                                     ▼                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                 SingleFileBlockManager                              │   │
│   │   ┌────────────────────────────────────────────────────────────┐    │   │
│   │   │  .duckdb 文件                                              │    │   │
│   │   │  ┌──────────┬──────────┬──────────┬────────────────────┐   │    │   │
│   │   │  │MainHeader│DBHeader 1│DBHeader 2│  Data Blocks ...   │   │    │   │
│   │   │  │  4KB     │   4KB    │   4KB    │   256KB each       │   │    │   │
│   │   │  └──────────┴──────────┴──────────┴────────────────────┘   │    │   │
│   │   └────────────────────────────────────────────────────────────┘    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1.1 存储系统设计理念

### 1.1.1 单文件存储架构

DuckDB 选择单文件存储有以下优势：

1. **简单部署**：一个文件就是一个完整的数据库，便于复制、备份和迁移
2. **原子性保证**：通过双头设计和 WAL 实现事务原子性
3. **减少文件系统开销**：避免多文件管理的复杂性
4. **便于嵌入式使用**：适合作为嵌入式数据库集成到应用中

```cpp
// 数据库路径处理
StorageManager::StorageManager(AttachedDatabase &db, string path_p, const AttachOptions &options)
    : db(db), path(std::move(path_p)), read_only(options.access_mode == AccessMode::READ_ONLY) {
    if (path.empty()) {
        path = IN_MEMORY_PATH;  // 内存模式：":memory:"
        return;
    }
    auto &fs = FileSystem::Get(db);
    path = fs.ExpandPath(path);  // 展开路径（处理 ~、环境变量等）
}
```

### 1.1.2 列式存储优势

DuckDB 采用列式存储格式，特别适合分析型（OLAP）工作负载：

- **高压缩比**：同类型数据连续存储，压缩效果更好
- **向量化执行**：便于 SIMD 指令并行处理
- **列裁剪**：只读取查询需要的列
- **延迟物化**：减少不必要的数据移动

### 1.1.3 与 SQLite 的对比

| 特性 | DuckDB | SQLite |
|------|--------|--------|
| 存储格式 | 列式 | 行式 |
| 目标工作负载 | OLAP（分析） | OLTP（事务） |
| 文件数量 | 单文件 + WAL | 单文件 + WAL |
| 块大小 | 256KB（可配置） | 4KB（页） |
| 压缩 | 多种算法 | 无内置压缩 |
| 向量化 | 是 | 否 |

---

## 1.2 核心组件概览

### 1.2.1 StorageManager

`StorageManager` 是存储层的入口点，负责协调所有存储相关的操作：

```cpp
// src/include/duckdb/storage/storage_manager.hpp

class StorageManager {
public:
    StorageManager(AttachedDatabase &db, string path, const AttachOptions &options);

    // 初始化数据库（创建新数据库或加载已有数据库）
    void Initialize(QueryContext context);

    // 获取 WAL
    optional_ptr<WriteAheadLog> GetWAL();

    // 获取数据库路径
    string GetDBPath() const { return path; }

    // 是否为内存数据库
    bool InMemory() const;

    // 检查点相关
    virtual void CreateCheckpoint(QueryContext context, CheckpointOptions options) = 0;
    virtual bool AutomaticCheckpoint(idx_t estimated_wal_bytes) = 0;

    // 获取数据库大小信息
    virtual DatabaseSize GetDatabaseSize() = 0;

    // 获取块管理器
    virtual BlockManager &GetBlockManager() = 0;

protected:
    AttachedDatabase &db;           // 关联的数据库
    string path;                    // 数据库文件路径
    string wal_path;                // WAL 文件路径
    unique_ptr<WriteAheadLog> wal;  // WAL 实例
    mutex wal_lock;                 // WAL 锁
    bool read_only;                 // 只读模式
    bool load_complete = false;     // 加载完成标志
    optional_idx storage_version;   // 存储版本
    atomic<idx_t> wal_size;         // WAL 大小
    StorageOptions storage_options; // 存储选项
};
```

### 1.2.2 SingleFileStorageManager

`SingleFileStorageManager` 是单文件存储的具体实现：

```cpp
// src/include/duckdb/storage/storage_manager.hpp

class SingleFileStorageManager : public StorageManager {
public:
    SingleFileStorageManager(AttachedDatabase &db, string path, const AttachOptions &options);

    // 块管理器（管理数据块的读写）
    unique_ptr<BlockManager> block_manager;

    // 表 I/O 管理器
    unique_ptr<TableIOManager> table_io_manager;

    // 创建检查点
    void CreateCheckpoint(QueryContext context, CheckpointOptions options) override;

    // 获取数据库大小
    DatabaseSize GetDatabaseSize() override;

protected:
    // 加载数据库
    void LoadDatabase(QueryContext context) override;
};
```

### 1.2.3 TableIOManager

`TableIOManager` 为每个表提供 I/O 管理服务：

```cpp
// src/include/duckdb/storage/table_io_manager.hpp

class TableIOManager {
public:
    virtual ~TableIOManager() {}

    // 获取索引数据的块管理器
    virtual BlockManager &GetIndexBlockManager() = 0;

    // 获取行数据的块管理器
    virtual BlockManager &GetBlockManagerForRowData() = 0;

    // 获取元数据管理器
    virtual MetadataManager &GetMetadataManager() = 0;

    // 获取 RowGroup 大小
    virtual idx_t GetRowGroupSize() const = 0;
};

// 单文件实现
class SingleFileTableIOManager : public TableIOManager {
public:
    explicit SingleFileTableIOManager(BlockManager &block_manager, idx_t row_group_size)
        : block_manager(block_manager), row_group_size(row_group_size) {}

    BlockManager &block_manager;
    idx_t row_group_size;

    BlockManager &GetIndexBlockManager() override { return block_manager; }
    BlockManager &GetBlockManagerForRowData() override { return block_manager; }
    MetadataManager &GetMetadataManager() override {
        return block_manager.GetMetadataManager();
    }
    idx_t GetRowGroupSize() const override { return row_group_size; }
};
```

### 1.2.4 SingleFileBlockManager

`SingleFileBlockManager` 管理单文件中的块分配和读写：

```cpp
// src/include/duckdb/storage/single_file_block_manager.hpp

class SingleFileBlockManager : public BlockManager {
    // 块数据开始位置（跳过三个文件头）
    static constexpr uint64_t BLOCK_START = Storage::FILE_HEADER_SIZE * 3;

public:
    SingleFileBlockManager(AttachedDatabase &db, const string &path,
                           const StorageManagerOptions &options);

    // 创建新数据库
    void CreateNewDatabase(QueryContext context);

    // 加载已有数据库
    void LoadExistingDatabase(QueryContext context);

    // 块操作
    unique_ptr<Block> CreateBlock(block_id_t block_id, FileBuffer *source_buffer) override;
    block_id_t GetFreeBlockId() override;
    void Read(QueryContext context, Block &block) override;
    void Write(QueryContext context, FileBuffer &buffer, block_id_t block_id) override;

    // 写入文件头
    void WriteHeader(QueryContext context, DatabaseHeader header) override;

    // 文件同步
    void FileSync() override;

    // 统计信息
    idx_t TotalBlocks() override;
    idx_t FreeBlocks() override;

private:
    AttachedDatabase &db;
    uint8_t active_header;               // 当前活动的头（0 或 1）
    string path;                         // 文件路径
    unique_ptr<FileHandle> handle;       // 文件句柄
    FileBuffer header_buffer;            // 头缓冲区

    set<block_id_t> free_list;           // 空闲块列表
    set<block_id_t> free_blocks_in_use;  // 正在使用的空闲块
    set<block_id_t> newly_used_blocks;   // 新使用的块
    unordered_map<block_id_t, uint32_t> multi_use_blocks;  // 多引用块
    unordered_set<block_id_t> modified_blocks;  // 已修改块

    idx_t meta_block;                    // 元数据块 ID
    block_id_t max_block;                // 最大块 ID
    idx_t free_list_id;                  // 空闲列表块 ID
    uint64_t iteration_count;            // 检查点迭代计数

    StorageManagerOptions options;
    mutex block_lock;
};
```

---

## 1.3 存储常量与配置

### 1.3.1 核心常量定义

```cpp
// src/include/duckdb/storage/storage_info.hpp

// 默认 RowGroup 大小：122,880 行
#define DEFAULT_ROW_GROUP_SIZE 122880ULL

// 无效块 ID
#define INVALID_BLOCK (-1)

// 最大块 ID
#define MAXIMUM_BLOCK 4611686018427388000LL

// 默认块分配大小：256 KB
#define DEFAULT_BLOCK_ALLOC_SIZE 262144ULL

// 默认块头大小：8 字节（用于校验和）
#define DEFAULT_BLOCK_HEADER_STORAGE_SIZE 8ULL

// 加密块头大小：40 字节
#define DEFAULT_ENCRYPTION_BLOCK_HEADER_SIZE 40ULL

struct Storage {
    // 扇区大小（Direct I/O 使用）
    constexpr static idx_t SECTOR_SIZE = 4096U;

    // 文件头大小：4 KB
    constexpr static idx_t FILE_HEADER_SIZE = 4096U;

    // 最大 RowGroup 大小：1GB
    constexpr static const idx_t MAX_ROW_GROUP_SIZE = 1ULL << 30ULL;

    // 最小块分配大小：16 KB
    constexpr static idx_t MIN_BLOCK_ALLOC_SIZE = 16384ULL;

    // 最大块分配大小：256 KB
    constexpr static idx_t MAX_BLOCK_ALLOC_SIZE = 262144ULL;

    // 默认块头大小
    constexpr static idx_t DEFAULT_BLOCK_HEADER_SIZE = sizeof(idx_t);

    // 最大块头大小：128 字节
    constexpr static idx_t MAX_BLOCK_HEADER_SIZE = 128ULL;

    // 默认可用块大小 = 分配大小 - 头大小
    constexpr static idx_t DEFAULT_BLOCK_SIZE =
        DEFAULT_BLOCK_ALLOC_SIZE - DEFAULT_BLOCK_HEADER_SIZE;
};
```

### 1.3.2 常量关系图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        存储常量关系                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  文件结构:                                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ FILE_HEADER_SIZE = 4KB                                                │  │
│  │ ┌─────────────┬─────────────┬─────────────┐                           │  │
│  │ │ MainHeader  │ DBHeader 1  │ DBHeader 2  │  BLOCK_START = 12KB      │  │
│  │ │   4 KB      │    4 KB     │    4 KB     │                           │  │
│  │ └─────────────┴─────────────┴─────────────┘                           │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  块结构:                                                                     │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ DEFAULT_BLOCK_ALLOC_SIZE = 256 KB                                     │  │
│  │ ┌──────────────────┬─────────────────────────────────────────────────┐│  │
│  │ │ Block Header     │              Block Data                         ││  │
│  │ │ 8 bytes (checksum│            256 KB - 8 bytes                     ││  │
│  │ │ or 40 bytes enc) │                                                 ││  │
│  │ └──────────────────┴─────────────────────────────────────────────────┘│  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
│  RowGroup 结构:                                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ DEFAULT_ROW_GROUP_SIZE = 122,880 行                                   │  │
│  │ = 60 × STANDARD_VECTOR_SIZE (2,048)                                  │  │
│  │                                                                       │  │
│  │ 约束: ROW_GROUP_SIZE % VECTOR_SIZE == 0                              │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3.3 配置参数

存储选项可以在 ATTACH 时配置：

```cpp
// src/storage/storage_manager.cpp

void StorageOptions::Initialize(const unordered_map<string, Value> &options) {
    for (auto &entry : options) {
        if (entry.first == "block_size") {
            // 块分配大小（用户称为 block_size，实际是 block_alloc_size）
            block_alloc_size = entry.second.GetValue<uint64_t>();
        } else if (entry.first == "encryption_key") {
            // 加密密钥
            user_key = make_shared_ptr<string>(StringValue::Get(entry.second));
            block_header_size = DEFAULT_ENCRYPTION_BLOCK_HEADER_SIZE;
            encryption = true;
        } else if (entry.first == "encryption_cipher") {
            // 加密算法：GCM 或 CTR
            encryption_cipher = EncryptionTypes::StringToCipher(entry.second.ToString());
        } else if (entry.first == "row_group_size") {
            // RowGroup 大小
            row_group_size = entry.second.GetValue<uint64_t>();
        } else if (entry.first == "storage_version") {
            // 存储版本
            storage_version = SerializationCompatibility::FromString(...);
        } else if (entry.first == "compress") {
            // 内存压缩（仅内存数据库）
            compress_in_memory = entry.second.GetValue<bool>()
                ? CompressInMemory::COMPRESS : CompressInMemory::DO_NOT_COMPRESS;
        }
    }
}
```

---

## 1.4 文件格式设计

### 1.4.1 文件布局

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          DuckDB 文件布局                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  偏移量        内容                     大小                                 │
│  ─────────────────────────────────────────────────────────────────────────  │
│  0x0000       MainHeader               4 KB                                 │
│              ┌─────────────────────────────────────────────────────────┐    │
│              │ [Block Header: 8 bytes]                                 │    │
│              │ Magic: "DUCK" (4 bytes)                                 │    │
│              │ Version Number (8 bytes)                                │    │
│              │ Flags[4] (32 bytes)                                     │    │
│              │ Library Version (32 bytes)                              │    │
│              │ Git Hash (32 bytes)                                     │    │
│              │ Encryption Metadata (8 bytes)                           │    │
│              │ DB Identifier (16 bytes)                                │    │
│              │ Encrypted Canary (8 bytes)                              │    │
│              └─────────────────────────────────────────────────────────┘    │
│                                                                             │
│  0x1000       DatabaseHeader 1         4 KB                                 │
│              ┌─────────────────────────────────────────────────────────┐    │
│              │ [Block Header: 8 bytes]                                 │    │
│              │ Iteration Count (8 bytes)                               │    │
│              │ Meta Block Pointer (8 bytes)                            │    │
│              │ Free List Pointer (8 bytes)                             │    │
│              │ Block Count (8 bytes)                                   │    │
│              │ Block Alloc Size (8 bytes)                              │    │
│              │ Vector Size (8 bytes)                                   │    │
│              │ Serialization Compatibility (8 bytes)                   │    │
│              └─────────────────────────────────────────────────────────┘    │
│                                                                             │
│  0x2000       DatabaseHeader 2         4 KB (备份头)                        │
│                                                                             │
│  0x3000       Data Block 0             256 KB                               │
│  0x43000      Data Block 1             256 KB                               │
│  ...          ...                      ...                                  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.4.2 MainHeader 详解

`MainHeader` 是数据库文件的主头，只写入一次：

```cpp
// src/include/duckdb/storage/storage_info.hpp

class MainHeader {
public:
    static constexpr idx_t MAX_VERSION_SIZE = 32;
    static constexpr idx_t MAGIC_BYTE_SIZE = 4;
    static constexpr idx_t MAGIC_BYTE_OFFSET = Storage::DEFAULT_BLOCK_HEADER_SIZE;
    static constexpr idx_t FLAG_COUNT = 4;

    // 魔数："DUCK"
    static const char MAGIC_BYTES[];

    // 加密验证："DUCKKEY"
    static const char CANARY[];

    // 加密标志位
    static constexpr uint64_t ENCRYPTED_DATABASE_FLAG = 1;

    // 存储版本号
    uint64_t version_number;

    // 标志位数组
    uint64_t flags[FLAG_COUNT];

    // 是否加密
    bool IsEncrypted() const {
        return flags[0] & ENCRYPTED_DATABASE_FLAG;
    }

    // 序列化/反序列化
    void Write(WriteStream &ser);
    static MainHeader Read(ReadStream &source);

private:
    data_t library_git_desc[MAX_VERSION_SIZE];   // 库版本描述
    data_t library_git_hash[MAX_VERSION_SIZE];   // Git 提交哈希
    data_t encryption_metadata[8];               // 加密元数据
    data_t db_identifier[16];                    // 数据库唯一标识
    data_t encrypted_canary[8];                  // 加密验证密文
};
```

### 1.4.3 DatabaseHeader 详解

`DatabaseHeader` 存储数据库的当前状态，每次检查点更新：

```cpp
// src/include/duckdb/storage/storage_info.hpp

struct DatabaseHeader {
    // 检查点迭代计数（每次检查点 +1）
    uint64_t iteration = 0;

    // 元数据块指针
    idx_t meta_block = 0;

    // 空闲列表块指针
    idx_t free_list = 0;

    // 文件中的总块数
    uint64_t block_count = 0;

    // 块分配大小
    idx_t block_alloc_size = 0;

    // 向量大小（编译时常量，用于兼容性检查）
    idx_t vector_size = 0;

    // 序列化兼容版本
    idx_t serialization_compatibility = 0;

    void Write(WriteStream &ser);
    static DatabaseHeader Read(const MainHeader &header, ReadStream &source);
};
```

### 1.4.4 双头设计

DuckDB 使用两个 `DatabaseHeader`（H1 和 H2）实现原子检查点：

```
检查点流程:
1. 当前活动头为 H1 (iteration = N)
2. 写入新数据块到文件
3. 更新 H2 (iteration = N+1, 指向新元数据)
4. 写入 H2 并 fsync
5. 切换活动头为 H2

恢复流程:
1. 读取 H1 和 H2
2. 选择 iteration 较大的头作为活动头
3. 如果两个头 iteration 相同，选择 H1（先写入的）

原子性保证:
- 如果检查点中途崩溃，较旧的头仍然有效
- 文件系统 fsync 保证头的原子写入
```

```cpp
// 加载时选择活动头
void SingleFileBlockManager::LoadExistingDatabase(QueryContext context) {
    // 读取两个头
    auto header1 = ReadDatabaseHeader(1);
    auto header2 = ReadDatabaseHeader(2);

    // 选择 iteration 较大的头
    if (header1.iteration >= header2.iteration) {
        active_header = 0;
        Initialize(header1, ...);
    } else {
        active_header = 1;
        Initialize(header2, ...);
    }
}

// 写入头时交替使用
void SingleFileBlockManager::WriteHeader(QueryContext context, DatabaseHeader header) {
    // 增加迭代计数
    header.iteration = iteration_count++;

    // 写入非活动头
    uint8_t new_active = active_header == 0 ? 1 : 0;
    WriteHeaderToSlot(context, header, new_active);

    // 文件同步
    FileSync();

    // 切换活动头
    active_header = new_active;
}
```

### 1.4.5 加密支持

DuckDB 支持数据库加密，使用 AES-GCM 或 AES-CTR：

```cpp
// 加密选项
struct EncryptionOptions {
    bool encryption_enabled = false;
    bool additional_authenticated_data = false;
    string derived_key_id;
    EncryptionTypes::KeyDerivationFunction kdf = EncryptionTypes::KeyDerivationFunction::SHA256;
    uint32_t key_length = MainHeader::DEFAULT_ENCRYPTION_KEY_LENGTH;  // 32 bytes
    shared_ptr<string> user_key;
};

// 加密时的头部变化
// 块头从 8 字节扩展到 40 字节：
// - Checksum: 8 bytes
// - IV/Nonce: 16 bytes
// - Tag (GCM): 16 bytes

// Canary 加密验证
void EncryptCanary(MainHeader &main_header, ...) {
    // 使用派生密钥加密已知明文 "DUCKKEY"
    // 解密时验证是否匹配，用于检测错误密钥
    encryption_state->Process(
        reinterpret_cast<const_data_ptr_t>(MainHeader::CANARY),
        MainHeader::CANARY_BYTE_SIZE,
        canary_buffer,
        MainHeader::CANARY_BYTE_SIZE
    );
    main_header.SetEncryptedCanary(canary_buffer);
}
```

---

## 1.5 Block 与 BlockPointer

### 1.5.1 Block 类

`Block` 继承自 `FileBuffer`，表示磁盘上的数据块：

```cpp
// src/include/duckdb/storage/block.hpp

class Block : public FileBuffer {
public:
    Block(BlockAllocator &allocator, const block_id_t id,
          const idx_t block_size, const idx_t block_header_size);

    block_id_t id;  // 块 ID
};
```

### 1.5.2 BlockPointer

用于引用块内特定位置：

```cpp
// src/include/duckdb/storage/block.hpp

struct BlockPointer {
    block_id_t block_id;   // 块 ID
    uint32_t offset;       // 块内偏移

    bool IsValid() const {
        return block_id != INVALID_BLOCK;
    }
};

// 元数据块指针（用于 Catalog 等元数据）
struct MetaBlockPointer {
    idx_t block_pointer;   // 包含块 ID 和块索引
    uint32_t offset;       // 块内偏移

    block_id_t GetBlockId() const;
    uint32_t GetBlockIndex() const;

    bool IsValid() const {
        return block_pointer != DConstants::INVALID_INDEX;
    }
};
```

### 1.5.3 块位置计算

```cpp
// 计算块在文件中的位置
idx_t SingleFileBlockManager::GetBlockLocation(block_id_t block_id) const {
    // 跳过三个头，每个块 block_alloc_size
    return BLOCK_START + block_id * block_alloc_size;
    // BLOCK_START = FILE_HEADER_SIZE * 3 = 12 KB
}
```

---

## 1.6 数据库生命周期

### 1.6.1 创建新数据库

```cpp
void SingleFileBlockManager::CreateNewDatabase(QueryContext context) {
    auto &fs = FileSystem::Get(db);

    // 1. 创建文件
    auto flags = GetFileFlags(true);  // create_new = true
    handle = fs.OpenFile(path, flags);

    // 2. 生成数据库标识符
    GenerateDBIdentifier(options.db_identifier);

    // 3. 构造 MainHeader
    auto main_header = ConstructMainHeader(GetVersionNumber());
    if (options.encryption_options.encryption_enabled) {
        main_header.SetEncrypted();
        StoreEncryptionMetadata(main_header);
        StoreEncryptedCanary(db, main_header, options.encryption_options.derived_key_id);
    }
    StoreDBIdentifier(main_header, options.db_identifier);

    // 4. 写入 MainHeader
    SerializeHeaderStructure(main_header, header_buffer.buffer);
    ChecksumAndWrite(context, header_buffer, 0);

    // 5. 初始化 DatabaseHeader
    DatabaseHeader db_header;
    db_header.block_alloc_size = block_alloc_size;
    db_header.vector_size = STANDARD_VECTOR_SIZE;
    db_header.serialization_compatibility = options.storage_version.GetIndex();

    // 6. 写入两个 DatabaseHeader
    SerializeHeaderStructure(db_header, header_buffer.buffer);
    ChecksumAndWrite(context, header_buffer, Storage::FILE_HEADER_SIZE);
    ChecksumAndWrite(context, header_buffer, Storage::FILE_HEADER_SIZE * 2);

    // 7. 初始化状态
    max_block = 0;
    iteration_count = 0;
    active_header = 0;
}
```

### 1.6.2 加载已有数据库

```cpp
void SingleFileBlockManager::LoadExistingDatabase(QueryContext context) {
    auto &fs = FileSystem::Get(db);

    // 1. 打开文件
    auto flags = GetFileFlags(false);
    handle = fs.OpenFile(path, flags);

    // 2. 读取并验证 MainHeader
    MainHeader::CheckMagicBytes(context, *handle);
    ReadAndChecksum(context, header_buffer, 0);
    auto main_header = DeserializeMainHeader(header_buffer.buffer);

    // 3. 处理加密
    if (main_header.IsEncrypted()) {
        CheckAndAddEncryptionKey(main_header);
    }

    // 4. 读取两个 DatabaseHeader，选择较新的
    ReadAndChecksum(context, header_buffer, Storage::FILE_HEADER_SIZE);
    auto header1 = DeserializeDatabaseHeader(main_header, header_buffer.buffer);

    ReadAndChecksum(context, header_buffer, Storage::FILE_HEADER_SIZE * 2);
    auto header2 = DeserializeDatabaseHeader(main_header, header_buffer.buffer);

    // 5. 选择活动头
    if (header1.iteration >= header2.iteration) {
        active_header = 0;
        Initialize(header1, options.block_alloc_size);
    } else {
        active_header = 1;
        Initialize(header2, options.block_alloc_size);
    }

    // 6. 加载空闲列表
    LoadFreeList(context);
}
```

### 1.6.3 完整加载流程

```cpp
void SingleFileStorageManager::LoadDatabase(QueryContext context) {
    if (InMemory()) {
        // 内存模式
        block_manager = make_uniq<InMemoryBlockManager>(...);
        table_io_manager = make_uniq<SingleFileTableIOManager>(...);
        return;
    }

    auto &fs = FileSystem::Get(db);

    if (!read_only && !fs.FileExists(path)) {
        // 创建新数据库
        auto sf_block_manager = make_uniq<SingleFileBlockManager>(...);
        sf_block_manager->CreateNewDatabase(context);
        block_manager = std::move(sf_block_manager);

        // 初始化 WAL
        wal = make_uniq<WriteAheadLog>(*this, wal_path);
    } else {
        // 加载已有数据库
        auto sf_block_manager = make_uniq<SingleFileBlockManager>(...);
        sf_block_manager->LoadExistingDatabase(context);
        block_manager = std::move(sf_block_manager);

        // 加载检查点
        auto checkpoint_reader = SingleFileCheckpointReader(*this);
        checkpoint_reader.LoadFromStorage();

        // 回放 WAL
        wal = WriteAheadLog::Replay(context, *this, wal_path);
    }

    table_io_manager = make_uniq<SingleFileTableIOManager>(...);
    load_complete = true;
}
```

---

## 1.7 存储扩展接口

### 1.7.1 StorageExtension

DuckDB 支持通过扩展自定义存储后端：

```cpp
// src/include/duckdb/storage/storage_extension.hpp

class StorageExtension {
public:
    // Attach 函数：创建自定义 Catalog
    attach_function_t attach;

    // 创建事务管理器
    create_transaction_manager_t create_transaction_manager;

    // 扩展信息
    shared_ptr<StorageExtensionInfo> storage_info;

    virtual ~StorageExtension() {}

    // 检查点钩子
    virtual void OnCheckpointStart(AttachedDatabase &db, CheckpointOptions options) {}
    virtual void OnCheckpointEnd(AttachedDatabase &db, CheckpointOptions options) {}
};

// 函数类型定义
typedef unique_ptr<Catalog> (*attach_function_t)(
    optional_ptr<StorageExtensionInfo> storage_info,
    ClientContext &context,
    AttachedDatabase &db,
    const string &name,
    AttachInfo &info,
    AttachOptions &options
);

typedef unique_ptr<TransactionManager> (*create_transaction_manager_t)(
    optional_ptr<StorageExtensionInfo> storage_info,
    AttachedDatabase &db,
    Catalog &catalog
);
```

### 1.7.2 扩展用例

存储扩展可用于：

1. **远程存储**：将数据存储在 S3、GCS 等云存储
2. **分布式存储**：跨多节点分布数据
3. **特殊格式**：读写 Parquet、CSV 等外部格式
4. **只读数据源**：连接外部数据库

```sql
-- 示例：使用存储扩展附加远程数据库
ATTACH 's3://bucket/database.duckdb' AS remote_db (TYPE S3_STORAGE);
```

---

## 1.8 版本兼容性

### 1.8.1 版本号管理

```cpp
// src/include/duckdb/storage/storage_info.hpp

// 当前版本号
extern const uint64_t VERSION_NUMBER;

// 可读取的版本范围
extern const uint64_t VERSION_NUMBER_LOWER;
extern const uint64_t VERSION_NUMBER_UPPER;

// 版本号到 DuckDB 版本的映射
string GetDuckDBVersions(const idx_t version_number);

// 版本兼容性检查
if (header.version_number < VERSION_NUMBER_LOWER ||
    header.version_number > VERSION_NUMBER_UPPER) {
    throw IOException(
        "Trying to read a database file with version number %lld, "
        "but we can only read versions between %lld and %lld.",
        header.version_number, VERSION_NUMBER_LOWER, VERSION_NUMBER_UPPER
    );
}
```

### 1.8.2 序列化兼容性

```cpp
// DatabaseHeader 中的序列化兼容版本
idx_t serialization_compatibility = 0;

// 序列化版本控制
// 版本 1: v0.x
// 版本 2: v1.0.0
// 版本 3: v1.1.0
// 版本 4: v1.2.0 (支持大 RowGroup)
// ...

// 根据版本选择序列化格式
if (serialization_version < 4) {
    // 旧格式
} else {
    // 新格式
}
```

---

## 1.9 源文件索引

| 文件 | 功能 |
|------|------|
| `src/include/duckdb/storage/storage_manager.hpp` | StorageManager 类定义 |
| `src/storage/storage_manager.cpp` | StorageManager 实现 |
| `src/include/duckdb/storage/single_file_block_manager.hpp` | SingleFileBlockManager 定义 |
| `src/storage/single_file_block_manager.cpp` | SingleFileBlockManager 实现 |
| `src/include/duckdb/storage/storage_info.hpp` | 存储常量和头结构 |
| `src/storage/storage_info.cpp` | 版本信息实现 |
| `src/include/duckdb/storage/table_io_manager.hpp` | TableIOManager 接口 |
| `src/include/duckdb/storage/block.hpp` | Block 和 BlockPointer |
| `src/include/duckdb/storage/storage_extension.hpp` | 存储扩展接口 |
| `src/include/duckdb/storage/storage_options.hpp` | 存储选项 |

---

## 1.10 总结

本章介绍了 DuckDB 存储引擎的整体架构：

1. **单文件架构**：所有数据存储在一个 `.duckdb` 文件中，简化部署和管理
2. **分层设计**：StorageManager → BlockManager → 文件系统的清晰层次
3. **双头设计**：通过两个 DatabaseHeader 实现原子检查点
4. **加密支持**：可选的 AES-GCM/CTR 全库加密
5. **可扩展性**：StorageExtension 接口支持自定义存储后端

下一章我们将深入探讨 BufferManager 和内存管理机制。
