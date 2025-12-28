# DuckDB 存储引擎深度解析：第六章 持久化与恢复

## 6.1 概述

DuckDB 采用 **Write-Ahead Logging (WAL)** 结合 **Checkpoint** 的混合持久化策略，在保证事务持久性（Durability）的同时，实现高效的数据写入和快速的崩溃恢复。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DuckDB 持久化架构                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     事务提交流程                                  │  │
│  │                                                                  │  │
│  │   Transaction    ┌─────────────┐    ┌─────────────────────────┐  │  │
│  │   COMMIT ───────>│   WAL       │───>│  In-Memory State        │  │  │
│  │                  │  (追加写入)  │    │  (立即可见)              │  │  │
│  │                  └─────────────┘    └─────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                              │                                          │
│                              ▼ (定期触发)                               │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    Checkpoint 流程                                │  │
│  │                                                                  │  │
│  │   ┌─────────┐    ┌──────────────┐    ┌─────────────────────┐    │  │
│  │   │  WAL    │───>│ Checkpoint   │───>│ Database File       │    │  │
│  │   │  数据   │    │ Writer       │    │ (.duckdb)           │    │  │
│  │   └─────────┘    └──────────────┘    └─────────────────────┘    │  │
│  │                         │                                        │  │
│  │                         ▼                                        │  │
│  │                  ┌──────────────┐                                │  │
│  │                  │ Truncate WAL │                                │  │
│  │                  └──────────────┘                                │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                    崩溃恢复流程                                   │  │
│  │                                                                  │  │
│  │   Database    ┌──────────────┐    ┌─────────────────────────┐   │  │
│  │   Startup ───>│ Load Last    │───>│ Replay WAL              │   │  │
│  │               │ Checkpoint   │    │ (如果存在)              │   │  │
│  │               └──────────────┘    └─────────────────────────┘   │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 6.2 数据库文件结构

### 6.2.1 MainHeader

`MainHeader` 是数据库文件的第一个头部，在文件创建时只写入一次：

```cpp
// src/include/duckdb/storage/storage_info.hpp

class MainHeader {
public:
    static constexpr idx_t MAX_VERSION_SIZE = 32;
    static constexpr idx_t MAGIC_BYTE_SIZE = 4;
    static constexpr idx_t FLAG_COUNT = 4;
    static constexpr idx_t DB_IDENTIFIER_LEN = 16;

    //! 魔数应为 "DUCK"
    static const char MAGIC_BYTES[];

    //! 数据库的存储版本号
    uint64_t version_number;
    //! 数据库使用的标志位
    uint64_t flags[FLAG_COUNT];

    //! 用于标识加密数据库
    static constexpr uint64_t ENCRYPTED_DATABASE_FLAG = 1;
    //! 加密密钥长度
    static constexpr uint64_t DEFAULT_ENCRYPTION_KEY_LENGTH = 32;

    //! 唯一数据库标识符（用于 WAL 匹配验证）
    data_t db_identifier[DB_IDENTIFIER_LEN];
    //! 加密相关元数据
    data_t encryption_metadata[ENCRYPTION_METADATA_LEN];
};
```

### 6.2.2 DatabaseHeader

每个存储文件有两个 `DatabaseHeader`，用于实现原子性的检查点切换：

```cpp
// src/include/duckdb/storage/storage_info.hpp

struct DatabaseHeader {
    //! 迭代计数器，每次检查点加 1
    uint64_t iteration = 0;
    //! 指向初始元数据块的指针
    idx_t meta_block = 0;
    //! 指向空闲列表块的指针
    idx_t free_list = 0;
    //! 文件中的块数量
    uint64_t block_count = 0;
    //! 块分配大小
    idx_t block_alloc_size = 0;
    //! 向量大小
    idx_t vector_size = 0;
    //! 序列化兼容版本
    idx_t serialization_compatibility = 0;
};
```

**双头部设计**：
- 启动时读取两个 DatabaseHeader，选择 `iteration` 更大的作为活跃头部
- 检查点时更新另一个头部，然后增加 `iteration` 切换活跃头部
- 这确保了即使在写入头部时崩溃，也能恢复到上一个有效状态

```
数据库文件布局：
┌───────────────────────────────────────────────────────────────────┐
│ Offset 0                                                          │
├───────────────────────────────────────────────────────────────────┤
│                         MainHeader                                │
│   - Magic Bytes: "DUCK"                                           │
│   - Version Number                                                │
│   - Flags (加密标志等)                                            │
│   - DB Identifier                                                 │
├───────────────────────────────────────────────────────────────────┤
│ Offset 4096 (FILE_HEADER_SIZE)                                    │
├───────────────────────────────────────────────────────────────────┤
│                     DatabaseHeader #1                             │
│   - iteration: 5                                                  │
│   - meta_block: 指向 Checkpoint 5 的元数据                        │
│   - free_list: 空闲块列表                                         │
│   - block_count: 1000                                             │
├───────────────────────────────────────────────────────────────────┤
│                     DatabaseHeader #2                             │
│   - iteration: 6  (活跃)                                          │
│   - meta_block: 指向 Checkpoint 6 的元数据                        │
│   - free_list: 更新后的空闲块列表                                 │
│   - block_count: 1050                                             │
├───────────────────────────────────────────────────────────────────┤
│                        Data Blocks                                │
│   Block 0, Block 1, Block 2, ...                                  │
└───────────────────────────────────────────────────────────────────┘
```

---

## 6.3 Write-Ahead Log (WAL)

### 6.3.1 WAL 概述

WAL 是实现事务持久性的核心机制：

```cpp
// src/include/duckdb/storage/write_ahead_log.hpp

//! WriteAheadLog (WAL) 用于提供持久性。在提交事务之前，
//! 它将事务对数据库所做的更改写入日志。
//! 如果服务器崩溃或关闭，可以在启动时重放日志。
class WriteAheadLog {
public:
    explicit WriteAheadLog(StorageManager &storage_manager,
                           const string &wal_path,
                           idx_t wal_size = 0ULL,
                           WALInitState state = WALInitState::NO_WAL,
                           optional_idx checkpoint_iteration = optional_idx());

    //! 重放并初始化 WAL
    static unique_ptr<WriteAheadLog> Replay(QueryContext context,
                                            StorageManager &storage_manager,
                                            const string &wal_path);

    //! 获取自启动以来写入 WAL 的总字节数
    idx_t GetTotalWritten() const;

    //! WAL 是否已初始化
    bool Initialized() const;

    //! 初始化 WAL 文件
    BufferedFileWriter &Initialize();

protected:
    StorageManager &storage_manager;
    mutex wal_lock;
    unique_ptr<BufferedFileWriter> writer;
    string wal_path;
    atomic<WALInitState> init_state;
    optional_idx checkpoint_iteration;
};
```

### 6.3.2 WAL 条目类型

DuckDB WAL 支持多种条目类型：

```cpp
// src/include/duckdb/common/enums/wal_type.hpp

enum class WALType : uint8_t {
    INVALID = 0,

    // ===== Catalog 操作 =====
    CREATE_TABLE = 1,
    DROP_TABLE = 2,
    CREATE_SCHEMA = 3,
    DROP_SCHEMA = 4,
    CREATE_VIEW = 5,
    DROP_VIEW = 6,
    CREATE_SEQUENCE = 8,
    DROP_SEQUENCE = 9,
    SEQUENCE_VALUE = 10,
    CREATE_MACRO = 11,
    DROP_MACRO = 12,
    CREATE_TYPE = 13,
    DROP_TYPE = 14,
    ALTER_INFO = 20,
    CREATE_TABLE_MACRO = 21,
    DROP_TABLE_MACRO = 22,
    CREATE_INDEX = 23,
    DROP_INDEX = 24,

    // ===== 数据操作 =====
    USE_TABLE = 25,      // 设置当前操作的表
    INSERT_TUPLE = 26,   // 插入数据
    DELETE_TUPLE = 27,   // 删除数据
    UPDATE_TUPLE = 28,   // 更新数据
    ROW_GROUP_DATA = 29, // 批量行组数据

    // ===== 控制操作 =====
    WAL_VERSION = 98,    // WAL 版本信息
    CHECKPOINT = 99,     // 检查点标记
    WAL_FLUSH = 100      // 刷新标记
};
```

### 6.3.3 WAL 写入流程

WAL 采用带校验和的序列化格式：

```cpp
// src/storage/write_ahead_log.cpp

constexpr uint64_t WAL_VERSION_NUMBER = 2;
constexpr uint64_t WAL_ENCRYPTED_VERSION_NUMBER = 3;

class ChecksumWriter : public WriteStream {
public:
    explicit ChecksumWriter(WriteAheadLog &wal)
        : wal(wal), memory_stream(Allocator::Get(wal.GetDatabase())) {}

    void WriteData(const_data_ptr_t buffer, idx_t write_size) override {
        // 先缓冲到内存流
        memory_stream.WriteData(buffer, write_size);
    }

    void Flush() {
        if (!stream) {
            stream = wal.Initialize();
        }

        auto data = memory_stream.GetData();
        auto size = memory_stream.GetPosition();

        // 计算校验和
        auto checksum = Checksum(data, size);

        // 写入条目大小和校验和
        stream->Write<uint64_t>(size);
        stream->Write<uint64_t>(checksum);

        // 写入实际数据
        stream->WriteData(data, size);

        // 重置缓冲区
        memory_stream.Rewind();
    }

private:
    WriteAheadLog &wal;
    optional_ptr<WriteStream> stream;
    MemoryStream memory_stream;
};
```

### 6.3.4 WAL 序列化器

WAL 条目通过专门的序列化器写入：

```cpp
class WriteAheadLogSerializer {
public:
    WriteAheadLogSerializer(WriteAheadLog &wal, WALType wal_type)
        : checksum_writer(wal),
          serializer(checksum_writer, SerializationOptions(wal.GetDatabase()))
    {
        if (!wal.Initialized()) {
            wal.Initialize();
        }
        // 如果还没有写入头部，先写入头部
        wal.WriteHeader();

        serializer.Begin();
        serializer.WriteProperty(100, "wal_type", wal_type);
    }

    void End() {
        serializer.End();
        checksum_writer.Flush();
    }

    template <class T>
    void WriteProperty(const field_id_t field_id, const char *tag, const T &value) {
        serializer.WriteProperty(field_id, tag, value);
    }
};
```

### 6.3.5 WAL 操作示例

**写入头部**：

```cpp
void WriteAheadLog::WriteHeader() {
    D_ASSERT(writer);
    if (writer->GetFileSize() > 0) {
        // 已经写入过，无需重复
        return;
    }

    BinarySerializer serializer(*writer);
    serializer.Begin();
    serializer.WriteProperty(100, "wal_type", WALType::WAL_VERSION);

    // 写入版本号
    auto &catalog = database.GetCatalog().Cast<DuckCatalog>();
    auto version = catalog.GetIsEncrypted()
        ? WAL_ENCRYPTED_VERSION_NUMBER
        : WAL_VERSION_NUMBER;
    serializer.WriteProperty(101, "version", version);

    // 写入数据库标识符（用于验证 WAL 是否匹配数据库文件）
    auto db_identifier = block_manager.GetDBIdentifier();
    serializer.WriteList(102, "db_identifier", MainHeader::DB_IDENTIFIER_LEN,
        [&](Serializer::List &list, idx_t i) {
            list.WriteElement(db_identifier[i]);
        });

    // 写入检查点迭代次数
    serializer.WriteProperty(103, "checkpoint_iteration", checkpoint_iteration);
    serializer.End();
}
```

**写入插入操作**：

```cpp
void WriteAheadLog::WriteInsert(DataChunk &chunk) {
    D_ASSERT(chunk.size() > 0);
    chunk.Verify();

    WriteAheadLogSerializer serializer(*this, WALType::INSERT_TUPLE);
    serializer.WriteProperty(101, "chunk", chunk);
    serializer.End();
}
```

**写入检查点标记**：

```cpp
void WriteAheadLog::WriteCheckpoint(MetaBlockPointer meta_block) {
    WriteAheadLogSerializer serializer(*this, WALType::CHECKPOINT);
    serializer.WriteProperty(101, "meta_block", meta_block);
    serializer.End();
}
```

### 6.3.6 WAL 刷新

```cpp
void WriteAheadLog::Flush() {
    if (!writer) {
        return;
    }

    // 写入一个空条目作为结束标记
    WriteAheadLogSerializer serializer(*this, WALType::WAL_FLUSH);
    serializer.End();

    // 将所有更改刷新到磁盘
    writer->Sync();
    storage_manager.SetWALSize(writer->GetFileSize());
}
```

---

## 6.4 WAL 重放

### 6.4.1 重放状态

```cpp
// src/storage/wal_replay.cpp

class ReplayState {
public:
    ReplayState(AttachedDatabase &db, ClientContext &context)
        : db(db), context(context), catalog(db.GetCatalog()) {}

    AttachedDatabase &db;
    ClientContext &context;
    Catalog &catalog;

    //! 当前操作的表
    optional_ptr<TableCatalogEntry> current_table;
    //! 检查点元数据块指针
    MetaBlockPointer checkpoint_id;
    //! WAL 版本号
    idx_t wal_version = 1;
    //! 当前读取位置
    optional_idx current_position;
    //! 检查点位置
    optional_idx checkpoint_position;

    //! 待添加的索引
    vector<ReplayIndexInfo> replay_index_infos;
};
```

### 6.4.2 重放反序列化器

```cpp
class WriteAheadLogDeserializer {
public:
    WriteAheadLogDeserializer(ReplayState &state_p, BufferedFileReader &stream_p,
                              bool deserialize_only = false)
        : state(state_p), db(state.db), context(state.context),
          catalog(state.catalog), deserializer(stream_p),
          deserialize_only(deserialize_only)
    {
        deserializer.Set<Catalog &>(catalog);
    }

    static WriteAheadLogDeserializer GetEntryDeserializer(
        ReplayState &state_p, BufferedFileReader &stream,
        bool deserialize_only = false)
    {
        if (state_p.wal_version == 1) {
            // 旧版 WAL 没有校验和
            return WriteAheadLogDeserializer(state_p, stream, deserialize_only);
        }

        if (state_p.wal_version == 2) {
            // 读取大小和校验和
            auto size = stream.Read<uint64_t>();
            auto stored_checksum = stream.Read<uint64_t>();

            // 读取数据到缓冲区
            auto buffer = unique_ptr<data_t[]>(new data_t[size]);
            stream.ReadData(buffer.get(), size);

            // 验证校验和
            auto computed_checksum = Checksum(buffer.get(), size);
            if (stored_checksum != computed_checksum) {
                throw IOException("Corrupt WAL file: checksum mismatch");
            }

            return WriteAheadLogDeserializer(state_p, std::move(buffer), size,
                                             deserialize_only);
        }

        // 版本 3：加密 WAL
        // ... 解密逻辑
    }

    bool ReplayEntry() {
        deserializer.Begin();
        auto wal_type = deserializer.ReadProperty<WALType>(100, "wal_type");
        if (wal_type == WALType::WAL_FLUSH) {
            deserializer.End();
            return true;  // 到达刷新点
        }
        ReplayEntry(wal_type);
        deserializer.End();
        return false;
    }
};
```

### 6.4.3 重放入口

```cpp
unique_ptr<WriteAheadLog> WriteAheadLog::Replay(
    QueryContext context, StorageManager &storage_manager,
    const string &wal_path)
{
    auto &fs = FileSystem::Get(storage_manager.GetAttached());
    auto handle = fs.OpenFile(wal_path,
        FileFlags::FILE_FLAGS_READ | FileFlags::FILE_FLAGS_NULL_IF_NOT_EXISTS);

    if (!handle) {
        // WAL 不存在，创建空 WAL
        return make_uniq<WriteAheadLog>(storage_manager, wal_path);
    }

    auto wal_handle = ReplayInternal(context, storage_manager, std::move(handle));
    if (wal_handle) {
        return wal_handle;
    }

    // 返回 NULL 表示可以删除 WAL
    if (!storage_manager.GetAttached().IsReadOnly()) {
        fs.TryRemoveFile(wal_path);
    }
    return make_uniq<WriteAheadLog>(storage_manager, wal_path);
}
```

### 6.4.4 重放流程

```cpp
unique_ptr<WriteAheadLog> WriteAheadLog::ReplayInternal(
    QueryContext context, StorageManager &storage_manager,
    unique_ptr<FileHandle> handle, WALReplayState replay_state)
{
    auto &database = storage_manager.GetAttached();
    Connection con(database.GetDatabase());
    BufferedFileReader reader(FileSystem::Get(database), std::move(handle));

    if (reader.Finished()) {
        // WAL 为空，可以删除
        return nullptr;
    }

    con.BeginTransaction();

    // 第一遍：扫描 WAL 查找检查点标记
    ReplayState checkpoint_state(database, *con.context);
    try {
        while (true) {
            checkpoint_state.current_position = reader.CurrentOffset();
            auto deserializer = WriteAheadLogDeserializer::GetEntryDeserializer(
                checkpoint_state, reader, true);  // deserialize_only = true

            if (deserializer.ReplayEntry()) {
                if (reader.Finished()) {
                    break;
                }
            }
        }
    } catch (std::exception &ex) {
        // 忽略序列化异常（表示 WAL 被截断）
    }

    // 检查是否有检查点标记
    if (checkpoint_state.checkpoint_id.IsValid()) {
        bool checkpoint_was_successful =
            manager.IsCheckpointClean(checkpoint_state.checkpoint_id);
        if (checkpoint_was_successful) {
            // 检查点已成功，WAL 内容已经持久化
            return nullptr;
        }
        // 需要处理检查点 WAL 合并...
    }

    // 第二遍：实际重放 WAL
    ReplayState state(database, *con.context);
    reader.Reset();

    idx_t successful_offset = 0;
    bool all_succeeded = false;

    try {
        while (true) {
            auto deserializer = WriteAheadLogDeserializer::GetEntryDeserializer(
                state, reader);

            if (deserializer.ReplayEntry()) {
                con.Commit();

                // 提交待添加的索引
                for (auto &info : state.replay_index_infos) {
                    info.index_list.get().AddIndex(std::move(info.index));
                }
                state.replay_index_infos.clear();

                successful_offset = reader.CurrentOffset();

                if (reader.Finished()) {
                    all_succeeded = true;
                    break;
                }
                con.BeginTransaction();
            }
        }
    } catch (std::exception &ex) {
        con.Query("ROLLBACK");
        // 处理异常...
    }

    auto init_state = all_succeeded
        ? WALInitState::UNINITIALIZED
        : WALInitState::UNINITIALIZED_REQUIRES_TRUNCATE;

    return make_uniq<WriteAheadLog>(storage_manager, wal_path,
                                    successful_offset, init_state);
}
```

### 6.4.5 条目重放分发

```cpp
void WriteAheadLogDeserializer::ReplayEntry(WALType entry_type) {
    switch (entry_type) {
    case WALType::WAL_VERSION:
        ReplayVersion();
        break;
    case WALType::CREATE_TABLE:
        ReplayCreateTable();
        break;
    case WALType::DROP_TABLE:
        ReplayDropTable();
        break;
    case WALType::USE_TABLE:
        ReplayUseTable();
        break;
    case WALType::INSERT_TUPLE:
        ReplayInsert();
        break;
    case WALType::DELETE_TUPLE:
        ReplayDelete();
        break;
    case WALType::UPDATE_TUPLE:
        ReplayUpdate();
        break;
    case WALType::CHECKPOINT:
        ReplayCheckpoint();
        break;
    // ... 其他条目类型
    }
}
```

### 6.4.6 数据重放示例

**重放插入**：

```cpp
void WriteAheadLogDeserializer::ReplayInsert() {
    DataChunk chunk;
    deserializer.ReadObject(101, "chunk",
        [&](Deserializer &object) { chunk.Deserialize(object); });

    if (DeserializeOnly()) {
        return;
    }

    if (!state.current_table) {
        throw InternalException("Corrupt WAL: insert without table");
    }

    // 追加到当前表（跳过约束验证）
    vector<unique_ptr<BoundConstraint>> bound_constraints;
    auto &storage = state.current_table->GetStorage();
    storage.LocalWALAppend(*state.current_table, context, chunk, bound_constraints);
}
```

**重放删除**：

```cpp
void WriteAheadLogDeserializer::ReplayDelete() {
    DataChunk chunk;
    deserializer.ReadObject(101, "chunk",
        [&](Deserializer &object) { chunk.Deserialize(object); });

    if (DeserializeOnly()) {
        return;
    }

    D_ASSERT(chunk.ColumnCount() == 1 &&
             chunk.data[0].GetType() == LogicalType::ROW_TYPE);

    auto &row_identifiers = chunk.data[0];
    auto &storage = state.current_table->GetStorage();

    TableDeleteState delete_state;
    storage.Delete(delete_state, context, row_identifiers, chunk.size());
}
```

---

## 6.5 Checkpoint 机制

### 6.5.1 Checkpoint 类型

```cpp
// src/include/duckdb/common/enums/checkpoint_type.hpp

enum class CheckpointWALAction {
    DELETE_WAL,       // 检查点后删除 WAL（通常在关闭时）
    DONT_DELETE_WAL   // 保留 WAL
};

enum class CheckpointAction {
    CHECKPOINT_IF_REQUIRED,  // 仅在需要时执行检查点
    ALWAYS_CHECKPOINT        // 总是执行检查点
};

enum class CheckpointType {
    //! 完整检查点：包括清理已删除行和更新
    //! 只能在没有事务需要读取旧数据时运行
    FULL_CHECKPOINT,

    //! 并发检查点：写入已提交数据，但清理较少
    //! 可以在有活跃事务时运行
    CONCURRENT_CHECKPOINT,

    //! 仅执行清理（用于内存表）
    VACUUM_ONLY
};
```

### 6.5.2 CheckpointWriter

```cpp
// src/include/duckdb/storage/checkpoint_manager.hpp

class CheckpointWriter {
public:
    explicit CheckpointWriter(AttachedDatabase &db) : db(db) {}
    virtual ~CheckpointWriter() {}

    AttachedDatabase &db;

    virtual void CreateCheckpoint() = 0;
    virtual MetadataManager &GetMetadataManager() = 0;
    virtual MetadataWriter &GetMetadataWriter() = 0;
    virtual unique_ptr<TableDataWriter> GetTableDataWriter(TableCatalogEntry &table) = 0;

protected:
    virtual void WriteEntry(CatalogEntry &entry, Serializer &serializer);
    virtual void WriteSchema(SchemaCatalogEntry &schema, Serializer &serializer);
    virtual void WriteTable(TableCatalogEntry &table, Serializer &serializer) = 0;
    virtual void WriteView(ViewCatalogEntry &table, Serializer &serializer);
    virtual void WriteSequence(SequenceCatalogEntry &table, Serializer &serializer);
    virtual void WriteIndex(IndexCatalogEntry &index, Serializer &serializer);
    // ...
};
```

### 6.5.3 SingleFileCheckpointWriter

```cpp
class SingleFileCheckpointWriter final : public CheckpointWriter {
public:
    SingleFileCheckpointWriter(QueryContext context, AttachedDatabase &db,
                               BlockManager &block_manager, CheckpointOptions options);

    void CreateCheckpoint() override;

private:
    optional_ptr<ClientContext> context;
    //! 元数据写入器（schema 信息）
    unique_ptr<MetadataWriter> metadata_writer;
    //! 表数据写入器（DataPointer）
    unique_ptr<MetadataWriter> table_metadata_writer;
    //! 部分块管理器（跨检查点共享）
    PartialBlockManager partial_block_manager;
    //! 检查点选项
    CheckpointOptions options;
};
```

### 6.5.4 Checkpoint 执行流程

```cpp
// src/storage/checkpoint_manager.cpp

void SingleFileCheckpointWriter::CreateCheckpoint() {
    auto &storage_manager = db.GetStorageManager().Cast<SingleFileStorageManager>();
    if (storage_manager.InMemory()) {
        return;
    }

    auto &block_manager = GetBlockManager();
    auto &metadata_manager = GetMetadataManager();

    // 1. 设置写入器
    metadata_writer = make_uniq<MetadataWriter>(metadata_manager);
    table_metadata_writer = make_uniq<MetadataWriter>(metadata_manager);

    // 2. 获取第一个元数据块的 ID
    auto meta_block = metadata_writer->GetMetaBlockPointer();

    // 3. 写入检查点标记到 WAL
    // 用于崩溃恢复时判断检查点是否完成
    auto has_wal = storage_manager.WALStartCheckpoint(meta_block, options);

    // 4. 扫描所有 Schema
    vector<reference<SchemaCatalogEntry>> schemas;
    auto &catalog = Catalog::GetCatalog(db).Cast<DuckCatalog>();
    catalog.ScanSchemas([&](SchemaCatalogEntry &entry) {
        schemas.push_back(entry);
    });

    // 5. 收集所有 Catalog 条目
    catalog_entry_vector_t catalog_entries = GetCatalogEntries(schemas);

    // 6. 按依赖顺序排序
    auto &dependency_manager = *catalog.GetDependencyManager();
    dependency_manager.ReorderEntries(catalog_entries);

    // 7. 序列化所有条目
    BinarySerializer serializer(*metadata_writer, SerializationOptions(db));
    serializer.Begin();
    serializer.WriteList(100, "catalog_entries", catalog_entries.size(),
        [&](Serializer::List &list, idx_t i) {
            auto &entry = catalog_entries[i];
            list.WriteObject([&](Serializer &obj) {
                WriteEntry(entry.get(), obj);
            });
        });
    serializer.End();

    // 8. 刷新元数据
    metadata_writer->Flush();
    table_metadata_writer->Flush();

    // 9. 写入更新后的 DatabaseHeader
    DatabaseHeader header;
    header.meta_block = meta_block.block_pointer;
    header.block_alloc_size = block_manager.GetBlockAllocSize();
    header.vector_size = STANDARD_VECTOR_SIZE;
    block_manager.WriteHeader(context, header);

    // 10. 截断文件（移除未使用的块）
    block_manager.Truncate();

    // 11. 完成 WAL 检查点（截断 WAL）
    if (has_wal) {
        storage_manager.WALFinishCheckpoint();
    }

    // 12. 合并索引增量
    for (auto &entry_ref : catalog_entries) {
        auto &entry = entry_ref.get();
        if (entry.type != CatalogType::TABLE_ENTRY) {
            continue;
        }
        auto &table = entry.Cast<DuckTableEntry>();
        auto &storage = table.GetStorage();
        auto &index_list = storage.GetDataTableInfo()->GetIndexes();
        index_list.MergeCheckpointDeltas(options.transaction_id);
    }
}
```

### 6.5.5 表数据写入

```cpp
// src/include/duckdb/storage/checkpoint/table_data_writer.hpp

class TableDataWriter {
public:
    explicit TableDataWriter(TableCatalogEntry &table, QueryContext context);

    void WriteTableData(Serializer &metadata_serializer);

    virtual void WriteUnchangedTable(MetaBlockPointer pointer, idx_t total_rows) = 0;
    virtual void FinalizeTable(const TableStatistics &global_stats,
                               DataTableInfo &info,
                               RowGroupCollection &collection,
                               Serializer &serializer) = 0;
    virtual unique_ptr<RowGroupWriter> GetRowGroupWriter(RowGroup &row_group) = 0;

protected:
    DuckTableEntry &table;
    optional_ptr<ClientContext> context;
    //! 每个 RowGroup 的起始指针
    vector<RowGroupPointer> row_group_pointers;
};
```

---

## 6.6 CheckpointReader

### 6.6.1 加载检查点

```cpp
void CheckpointReader::LoadCheckpoint(CatalogTransaction transaction,
                                       MetadataReader &reader)
{
    BinaryDeserializer deserializer(reader);
    deserializer.Set<Catalog &>(catalog);
    deserializer.Begin();

    deserializer.ReadList(100, "catalog_entries",
        [&](Deserializer::List &list, idx_t i) {
            return list.ReadObject([&](Deserializer &obj) {
                ReadEntry(transaction, obj);
            });
        });

    deserializer.End();
    deserializer.Unset<Catalog>();
}
```

### 6.6.2 SingleFileCheckpointReader

```cpp
void SingleFileCheckpointReader::LoadFromStorage() {
    auto &block_manager = *storage.block_manager;
    auto &metadata_manager = GetMetadataManager();

    MetaBlockPointer meta_block(block_manager.GetMetaBlock(), 0);
    if (!meta_block.IsValid()) {
        // 存储为空
        return;
    }

    // 预取元数据块
    if (block_manager.Prefetch()) {
        auto metadata_blocks = metadata_manager.GetBlocks();
        auto &buffer_manager = BufferManager::GetBufferManager(storage.GetDatabase());
        buffer_manager.Prefetch(metadata_blocks);
    }

    // 创建 MetadataReader 并加载
    MetadataReader reader(metadata_manager, meta_block);
    auto transaction = CatalogTransaction::GetSystemTransaction(catalog.GetDatabase());
    LoadCheckpoint(transaction, reader);
}
```

### 6.6.3 读取表数据

```cpp
void CheckpointReader::ReadTableData(CatalogTransaction transaction,
                                     Deserializer &deserializer,
                                     BoundCreateTableInfo &bound_info)
{
    // 读取表指针
    auto table_pointer = deserializer.ReadProperty<MetaBlockPointer>(101, "table_pointer");
    auto total_rows = deserializer.ReadProperty<idx_t>(102, "total_rows");

    // 读取索引存储信息
    auto index_storage_infos = deserializer.ReadPropertyWithExplicitDefault<
        vector<IndexStorageInfo>>(104, "index_storage_infos", {});

    if (!index_storage_infos.empty()) {
        bound_info.indexes = index_storage_infos;
    }

    // 读取表数据
    auto &binary_deserializer = dynamic_cast<BinaryDeserializer &>(deserializer);
    auto &reader = dynamic_cast<MetadataReader &>(binary_deserializer.GetStream());

    MetadataReader table_data_reader(reader.GetMetadataManager(), table_pointer);
    TableDataReader data_reader(table_data_reader, bound_info, table_pointer);
    data_reader.ReadTableData();

    bound_info.data->total_rows = total_rows;
}
```

---

## 6.7 崩溃恢复

### 6.7.1 恢复流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        DuckDB 崩溃恢复流程                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Database Startup                                                       │
│        │                                                                │
│        ▼                                                                │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. 读取 MainHeader                                               │   │
│  │    - 验证魔数 "DUCK"                                             │   │
│  │    - 检查版本兼容性                                              │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. 读取两个 DatabaseHeader                                       │   │
│  │    - 选择 iteration 更大的作为活跃头部                           │   │
│  │    - 获取 meta_block 指针                                        │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. 加载 Checkpoint 数据                                          │   │
│  │    - 从 meta_block 开始读取元数据                                │   │
│  │    - 重建 Catalog（Schema、Table、View 等）                      │   │
│  │    - 加载表数据                                                  │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 4. 检查 WAL 是否存在                                             │   │
│  │    ┌────────────────────┐    ┌────────────────────────────┐     │   │
│  │    │ WAL 不存在         │    │ WAL 存在                    │     │   │
│  │    │ → 完成恢复         │    │ → 继续重放                  │     │   │
│  │    └────────────────────┘    └────────────────────────────┘     │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 5. 验证 WAL 与数据库匹配                                         │   │
│  │    - 比较 db_identifier                                         │   │
│  │    - 比较 checkpoint_iteration                                   │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 6. 扫描 WAL 查找 CHECKPOINT 标记                                 │   │
│  │    ┌────────────────────────┐  ┌─────────────────────────────┐  │   │
│  │    │ 无 CHECKPOINT 标记     │  │ 有 CHECKPOINT 标记          │  │   │
│  │    │ → 重放整个 WAL        │  │ → 检查检查点是否成功        │  │   │
│  │    └────────────────────────┘  └─────────────────────────────┘  │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 7. 重放 WAL 条目                                                 │   │
│  │    - 按顺序应用每个条目                                          │   │
│  │    - 遇到 WAL_FLUSH 时提交事务                                   │   │
│  │    - 处理截断的 WAL（忽略不完整条目）                            │   │
│  └────────────────────────────────┬────────────────────────────────┘   │
│                                   │                                     │
│                                   ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 8. 截断 WAL 到最后成功位置                                       │   │
│  │    - 移除不完整的条目                                            │   │
│  │    - 准备接受新事务                                              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.7.2 检查点中断恢复

当检查点过程中发生崩溃时，DuckDB 使用 **Checkpoint WAL** 机制来恢复：

```cpp
if (checkpoint_state.checkpoint_id.IsValid()) {
    // 有检查点标记，检查点过程中可能发生崩溃

    // 检查是否有 checkpoint.wal 文件
    auto checkpoint_wal = manager.GetCheckpointWALPath();
    checkpoint_handle = fs.OpenFile(checkpoint_wal, ...);

    bool checkpoint_was_successful =
        manager.IsCheckpointClean(checkpoint_state.checkpoint_id);

    if (!checkpoint_handle) {
        // 没有 checkpoint WAL
        if (checkpoint_was_successful) {
            // 检查点已成功完成，WAL 可以删除
            return nullptr;
        }
        // 需要重放 WAL
    } else {
        // 有 checkpoint WAL
        if (checkpoint_was_successful) {
            // 检查点成功，只需要重放 checkpoint WAL
            return ReplayInternal(context, storage_manager,
                                  std::move(checkpoint_handle),
                                  WALReplayState::CHECKPOINT_WAL);
        }
        // 检查点失败，需要合并两个 WAL 并重放
        // 1. 复制主 WAL 到 checkpoint_position
        // 2. 追加 checkpoint WAL 内容
        // 3. 重放合并后的 WAL
    }
}
```

---

## 6.8 加密支持

### 6.8.1 加密 WAL

DuckDB 支持加密的 WAL 文件：

```cpp
void ChecksumWriter::FlushEncrypted() {
    auto &catalog = wal.GetDatabase().GetCatalog().Cast<DuckCatalog>();
    auto encryption_key_id = catalog.GetEncryptionKeyId();

    auto data = memory_stream.GetData();
    auto size = memory_stream.GetPosition();

    // 计算校验和
    auto checksum = Checksum(data, size);

    // 获取加密密钥
    auto &keys = EncryptionKeyManager::Get(db.GetDatabase());
    auto encryption_state = db.GetDatabase().GetEncryptionUtil()->CreateEncryptionState(
        db.GetStorageManager().GetCipher(),
        MainHeader::DEFAULT_ENCRYPTION_KEY_LENGTH);

    // 生成随机 nonce
    EncryptionNonce nonce;
    EncryptionTag tag;
    encryption_state->GenerateRandomData(nonce.data(), nonce.size());

    // 写入大小和 nonce
    stream->Write<uint64_t>(size);
    stream->WriteData(nonce.data(), nonce.size());

    // 准备加密缓冲区：校验和 + 数据
    const idx_t ciphertext_size = size + sizeof(uint64_t);
    std::unique_ptr<uint8_t[]> temp_buf(new uint8_t[ciphertext_size]);
    memcpy(temp_buf.get(), &checksum, sizeof(checksum));
    memcpy(temp_buf.get() + sizeof(checksum), data, size);

    // 加密
    encryption_state->InitializeEncryption(nonce.data(), nonce.size(),
        keys.GetKey(encryption_key_id),
        MainHeader::DEFAULT_ENCRYPTION_KEY_LENGTH);
    encryption_state->Process(temp_buf.get(), ciphertext_size,
                             temp_buf.get(), ciphertext_size);

    // 计算 GCM 标签
    encryption_state->Finalize(temp_buf.get(), ciphertext_size,
                              tag.data(), tag.size());

    // 写入加密数据和标签
    stream->WriteData(temp_buf.get(), ciphertext_size);
    stream->WriteData(tag.data(), tag.size());

    memory_stream.Rewind();
}
```

---

## 6.9 总结

### 6.9.1 持久化机制对比

| 特性 | WAL | Checkpoint |
|------|-----|------------|
| 写入模式 | 追加写入 | 批量写入 |
| 写入时机 | 事务提交时 | 定期或手动触发 |
| 恢复速度 | 较慢（需要重放） | 快速（直接加载） |
| 空间效率 | 较低（累积日志） | 高效（压缩存储） |
| 并发影响 | 最小 | 需要协调 |

### 6.9.2 关键设计特点

1. **双头部设计**：通过两个 DatabaseHeader 实现原子性检查点切换
2. **校验和保护**：WAL 条目带校验和，确保数据完整性
3. **增量恢复**：只重放检查点之后的 WAL 条目
4. **并发检查点**：支持在有活跃事务时执行检查点
5. **加密支持**：可选的 WAL 和数据加密
6. **检查点 WAL**：处理检查点过程中的崩溃恢复

### 6.9.3 核心源文件索引

| 文件 | 功能 |
|------|------|
| `src/include/duckdb/storage/storage_info.hpp` | MainHeader/DatabaseHeader 定义 |
| `src/include/duckdb/storage/write_ahead_log.hpp` | WriteAheadLog 类定义 |
| `src/storage/write_ahead_log.cpp` | WAL 写入实现 |
| `src/storage/wal_replay.cpp` | WAL 重放实现 |
| `src/include/duckdb/storage/checkpoint_manager.hpp` | CheckpointWriter/Reader 定义 |
| `src/storage/checkpoint_manager.cpp` | Checkpoint 实现 |
| `src/include/duckdb/common/enums/wal_type.hpp` | WAL 条目类型枚举 |
| `src/include/duckdb/common/enums/checkpoint_type.hpp` | Checkpoint 类型枚举 |
| `src/include/duckdb/storage/checkpoint/table_data_writer.hpp` | 表数据写入器 |

---

**下一章预告**：第七章将探讨 DuckDB 的索引系统，包括 ART 索引的实现细节和索引在查询优化中的应用。
