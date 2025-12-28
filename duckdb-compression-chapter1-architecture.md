# DuckDB压缩系统深度解析（一）：架构与接口设计

## 引言

DuckDB实现了一套精密的自适应压缩系统，包含15种压缩算法，能够根据数据特征自动选择最优压缩方案。与传统数据库使用单一压缩算法不同，DuckDB为不同数据类型设计了专门的压缩策略：整数使用Bitpacking，浮点数使用Chimp/ALP，字符串使用Dictionary/FSST，位图使用Roaring。这种精细化的压缩策略使DuckDB在保持高查询性能的同时，实现了优秀的存储效率。

本章将深入剖析DuckDB压缩系统的整体架构和核心接口设计。

## 1. 压缩系统架构概览

### 1.1 三阶段处理模型

DuckDB的压缩系统采用经典的三阶段处理模型：

```
┌─────────────────────────────────────────────────────────────────────┐
│                        压缩处理流程                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │   分析阶段   │ -> │   压缩阶段   │ -> │   存储阶段   │         │
│  │   Analyze    │    │   Compress   │    │   Persist    │         │
│  └──────────────┘    └──────────────┘    └──────────────┘         │
│         │                   │                   │                  │
│         v                   v                   v                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐         │
│  │ 统计信息收集 │    │ 数据编码    │    │ 元数据写入   │         │
│  │ 评分计算     │    │ 缓冲区管理  │    │ 段持久化     │         │
│  │ 算法筛选     │    │ 段分割      │    │ 块分配       │         │
│  └──────────────┘    └──────────────┘    └──────────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**阶段一：分析（Analyze）**
- 扫描所有待压缩数据
- 收集统计信息（基数、值范围、重复模式等）
- 为每个候选算法计算评分
- 筛选出最优压缩算法

**阶段二：压缩（Compress）**
- 使用选定算法编码数据
- 管理压缩缓冲区
- 按需分割数据段

**阶段三：存储（Persist）**
- 将压缩数据写入磁盘块
- 更新元数据
- 完成段持久化

### 1.2 压缩类型枚举

DuckDB定义了15种压缩类型，覆盖不同的数据场景：

```cpp
// src/include/duckdb/common/enums/compression_type.hpp

enum class CompressionType : uint8_t {
    COMPRESSION_AUTO = 0,           // 自动选择
    COMPRESSION_UNCOMPRESSED = 1,   // 不压缩
    COMPRESSION_CONSTANT = 2,       // 常量压缩（内部使用）
    COMPRESSION_RLE = 3,            // 游程编码
    COMPRESSION_DICTIONARY = 4,     // 字典压缩
    COMPRESSION_PFOR_DELTA = 5,     // PFor Delta（遗留）
    COMPRESSION_BITPACKING = 6,     // 位压缩
    COMPRESSION_FSST = 7,           // 快速静态符号表
    COMPRESSION_CHIMP = 8,          // Chimp浮点压缩
    COMPRESSION_PATAS = 9,          // Patas浮点压缩
    COMPRESSION_ALP = 10,           // 自适应无损浮点
    COMPRESSION_ALPRD = 11,         // ALP + Roaring Delta
    COMPRESSION_ZSTD = 12,          // Zstandard通用压缩
    COMPRESSION_ROARING = 13,       // Roaring位图
    COMPRESSION_EMPTY = 14,         // 空有效性（内部使用）
    COMPRESSION_DICT_FSST = 15,     // Dictionary + FSST组合
    COMPRESSION_COUNT               // 类型计数
};
```

这些压缩类型可分为几个类别：

| 类别 | 压缩类型 | 适用数据类型 |
|------|---------|-------------|
| 整数压缩 | Bitpacking, RLE | INT8-INT64, UINT8-UINT64 |
| 浮点压缩 | Chimp, Patas, ALP, ALPRD | FLOAT, DOUBLE |
| 字符串压缩 | Dictionary, FSST, Dict-FSST | VARCHAR |
| 位图压缩 | Roaring, Empty | BIT (有效性) |
| 通用压缩 | ZSTD, Uncompressed, Constant | 所有类型 |

### 1.3 物理类型到压缩函数的映射

系统为19种物理类型分别维护可用的压缩函数列表：

```cpp
// src/function/compression_config.cpp

idx_t CompressionFunctionSet::GetCompressionIndex(PhysicalType physical_type) {
    switch (physical_type) {
    case PhysicalType::BOOL:     return 0;
    case PhysicalType::UINT8:    return 1;
    case PhysicalType::INT8:     return 2;
    case PhysicalType::UINT16:   return 3;
    case PhysicalType::INT16:    return 4;
    case PhysicalType::UINT32:   return 5;
    case PhysicalType::INT32:    return 6;
    case PhysicalType::UINT64:   return 7;
    case PhysicalType::INT64:    return 8;
    case PhysicalType::FLOAT:    return 9;
    case PhysicalType::DOUBLE:   return 10;
    case PhysicalType::INTERVAL: return 11;
    case PhysicalType::LIST:     return 12;
    case PhysicalType::STRUCT:   return 13;
    case PhysicalType::ARRAY:    return 14;
    case PhysicalType::VARCHAR:  return 15;
    case PhysicalType::UINT128:  return 16;
    case PhysicalType::INT128:   return 17;
    case PhysicalType::BIT:      return 18;
    default:
        throw InternalException("Unsupported physical type");
    }
}
```

## 2. 核心状态类设计

DuckDB为压缩的不同阶段定义了独立的状态类，实现关注点分离：

### 2.1 CompressionInfo - 压缩上下文信息

`CompressionInfo`封装了压缩所需的基础上下文信息：

```cpp
// src/include/duckdb/function/compression_function.hpp

class CompressionInfo {
public:
    explicit CompressionInfo(BlockManager &block_manager)
        : block_manager(block_manager) {}

    //! 段压缩刷新阈值（块大小的80%）
    idx_t GetCompactionFlushLimit() const {
        return block_manager.GetBlockSize() / 5 * 4;
    }

    //! 获取块大小
    idx_t GetBlockSize() const {
        return block_manager.GetBlockSize();
    }

    //! 获取块头大小
    idx_t GetBlockHeaderSize() const {
        return block_manager.GetBlockHeaderSize();
    }

    BlockManager &GetBlockManager() const {
        return block_manager;
    }

private:
    BlockManager &block_manager;
};
```

关键设计点：
- **压缩刷新阈值**：当段达到块大小的80%时触发刷新，留出缓冲空间
- **块大小感知**：压缩算法需要知道可用空间来进行合理分割

### 2.2 AnalyzeState - 分析阶段状态

`AnalyzeState`是所有分析状态类的基类：

```cpp
struct AnalyzeState {
    explicit AnalyzeState(const CompressionInfo &info) : info(info) {};
    virtual ~AnalyzeState() {}

    template <class TARGET>
    TARGET &Cast() {
        DynamicCastCheck<TARGET>(this);
        return reinterpret_cast<TARGET &>(*this);
    }

    CompressionInfo info;
};
```

每种压缩算法定义自己的派生状态类。以RLE为例：

```cpp
// src/storage/compression/rle.cpp

template <class T>
struct RLEAnalyzeState : public AnalyzeState {
    explicit RLEAnalyzeState(const CompressionInfo &info)
        : AnalyzeState(info) {}

    RLEState<T> state;  // RLE特有的统计状态
};

template <class T>
struct RLEState {
    idx_t seen_count;           // 已见的游程数
    T last_value;               // 上一个值
    rle_count_t last_seen_count; // 当前游程长度
    void *dataptr;              // 回调数据指针
    bool all_null = true;       // 是否全为NULL

    template <class OP>
    void Update(const T *data, ValidityMask &validity, idx_t idx) {
        // 更新RLE状态...
    }
};
```

### 2.3 CompressionState - 压缩阶段状态

`CompressionState`管理压缩过程中的运行时状态：

```cpp
struct CompressionState {
    explicit CompressionState(const CompressionInfo &info) : info(info) {};
    virtual ~CompressionState() {}

    template <class TARGET>
    TARGET &Cast() {
        DynamicCastCheck<TARGET>(this);
        return reinterpret_cast<TARGET &>(*this);
    }

    CompressionInfo info;
};
```

RLE的压缩状态示例：

```cpp
template <class T, bool WRITE_STATISTICS>
struct RLECompressState : public CompressionState {
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

    ColumnDataCheckpointData &checkpoint_data;  // 检查点数据
    CompressionFunction &function;              // 压缩函数引用
    unique_ptr<ColumnSegment> current_segment;  // 当前段
    BufferHandle handle;                        // 缓冲区句柄
    RLEState<T> state;                         // RLE状态
    idx_t entry_count = 0;                     // 条目计数
    idx_t max_rle_count;                       // 最大RLE条目数

    idx_t MaxRLECount() {
        auto entry_size = sizeof(T) + sizeof(rle_count_t);
        return AlignValueFloor(
            (info.GetBlockSize() - RLE_HEADER_SIZE) / entry_size);
    }
};
```

### 2.4 CompressedSegmentState - 持久化段状态

`CompressedSegmentState`保存压缩段的持久化元数据：

```cpp
struct CompressedSegmentState {
    virtual ~CompressedSegmentState() {}

    //! 用于PRAGMA storage_info的显示信息
    virtual string GetSegmentInfo() const {
        return "";
    }

    template <class TARGET>
    TARGET &Cast() {
        DynamicCastCheck<TARGET>(this);
        return reinterpret_cast<TARGET &>(*this);
    }
};
```

### 2.5 CompressionAppendState - 追加状态

用于事务性追加操作：

```cpp
struct CompressionAppendState {
    explicit CompressionAppendState(BufferHandle handle_p)
        : handle(std::move(handle_p)) {}
    virtual ~CompressionAppendState() {}

    BufferHandle handle;  // 内存缓冲区句柄
};
```

## 3. CompressionFunction - 核心接口抽象

`CompressionFunction`是压缩系统的核心抽象，定义了压缩算法必须实现的完整接口：

### 3.1 函数指针类型定义

```cpp
//===--------------------------------------------------------------------===//
// 分析阶段函数
//===--------------------------------------------------------------------===//
// 初始化分析状态
typedef unique_ptr<AnalyzeState> (*compression_init_analyze_t)(
    ColumnData &col_data, PhysicalType type);

// 分析单个向量，返回false表示放弃该压缩方法
typedef bool (*compression_analyze_t)(
    AnalyzeState &state, Vector &input, idx_t count);

// 完成分析，返回评分（字节数估算）
typedef idx_t (*compression_final_analyze_t)(AnalyzeState &state);

//===--------------------------------------------------------------------===//
// 压缩阶段函数
//===--------------------------------------------------------------------===//
// 初始化压缩状态
typedef unique_ptr<CompressionState> (*compression_init_compression_t)(
    ColumnDataCheckpointData &checkpoint_data,
    unique_ptr<AnalyzeState> state);

// 压缩单个向量
typedef void (*compression_compress_data_t)(
    CompressionState &state, Vector &scan_vector, idx_t count);

// 完成压缩
typedef void (*compression_compress_finalize_t)(CompressionState &state);

//===--------------------------------------------------------------------===//
// 解压/扫描阶段函数
//===--------------------------------------------------------------------===//
// 初始化预取状态
typedef void (*compression_init_prefetch_t)(
    ColumnSegment &segment, PrefetchState &prefetch_state);

// 初始化段扫描
typedef unique_ptr<SegmentScanState> (*compression_init_segment_scan_t)(
    const QueryContext &context, ColumnSegment &segment);

// 扫描完整向量（STANDARD_VECTOR_SIZE）
typedef void (*compression_scan_vector_t)(
    ColumnSegment &segment, ColumnScanState &state,
    idx_t scan_count, Vector &result);

// 扫描部分数据
typedef void (*compression_scan_partial_t)(
    ColumnSegment &segment, ColumnScanState &state,
    idx_t scan_count, Vector &result, idx_t result_offset);

// 选择扫描（带选择向量）
typedef void (*compression_select_t)(
    ColumnSegment &segment, ColumnScanState &state,
    idx_t vector_count, Vector &result,
    const SelectionVector &sel, idx_t sel_count);

// 过滤扫描
typedef void (*compression_filter_t)(
    ColumnSegment &segment, ColumnScanState &state,
    idx_t vector_count, Vector &result,
    SelectionVector &sel, idx_t &sel_count,
    const TableFilter &filter, TableFilterState &filter_state);

// 获取单行
typedef void (*compression_fetch_row_t)(
    ColumnSegment &segment, ColumnFetchState &state,
    row_t row_id, Vector &result, idx_t result_idx);

// 跳过行
typedef void (*compression_skip_t)(
    ColumnSegment &segment, ColumnScanState &state, idx_t skip_count);
```

### 3.2 CompressionFunction类定义

```cpp
class CompressionFunction {
public:
    CompressionFunction(
        CompressionType type,
        PhysicalType data_type,
        // 分析阶段
        compression_init_analyze_t init_analyze,
        compression_analyze_t analyze,
        compression_final_analyze_t final_analyze,
        // 压缩阶段
        compression_init_compression_t init_compression,
        compression_compress_data_t compress,
        compression_compress_finalize_t compress_finalize,
        // 扫描阶段
        compression_init_segment_scan_t init_scan,
        compression_scan_vector_t scan_vector,
        compression_scan_partial_t scan_partial,
        compression_fetch_row_t fetch_row,
        compression_skip_t skip,
        // 可选函数...
        compression_init_segment_t init_segment = nullptr,
        compression_init_append_t init_append = nullptr,
        compression_append_t append = nullptr,
        compression_finalize_append_t finalize_append = nullptr,
        compression_revert_append_t revert_append = nullptr,
        compression_serialize_state_t serialize_state = nullptr,
        compression_deserialize_state_t deserialize_state = nullptr,
        compression_visit_block_ids_t visit_block_ids = nullptr,
        compression_init_prefetch_t init_prefetch = nullptr,
        compression_select_t select = nullptr,
        compression_filter_t filter = nullptr
    );

    //! 压缩类型
    CompressionType type;
    //! 支持的数据物理类型
    PhysicalType data_type;

    // === 分析阶段函数 ===
    compression_init_analyze_t init_analyze;
    compression_analyze_t analyze;
    compression_final_analyze_t final_analyze;

    // === 压缩阶段函数 ===
    compression_init_compression_t init_compression;
    compression_compress_data_t compress;
    compression_compress_finalize_t compress_finalize;

    // === 扫描阶段函数 ===
    compression_init_prefetch_t init_prefetch;
    compression_init_segment_scan_t init_scan;
    compression_scan_vector_t scan_vector;
    compression_scan_partial_t scan_partial;
    compression_select_t select;
    compression_filter_t filter;
    compression_fetch_row_t fetch_row;
    compression_skip_t skip;

    // === 追加函数（可选）===
    compression_init_segment_t init_segment;
    compression_init_append_t init_append;
    compression_append_t append;
    compression_finalize_append_t finalize_append;
    compression_revert_append_t revert_append;

    // === 序列化函数（可选）===
    compression_serialize_state_t serialize_state;
    compression_deserialize_state_t deserialize_state;
    compression_visit_block_ids_t visit_block_ids;

    // === 段信息函数（可选）===
    compression_get_segment_info_t get_segment_info = nullptr;

    //! 有效性处理策略
    CompressionValidity validity = CompressionValidity::REQUIRES_VALIDITY;
};
```

### 3.3 CompressionValidity - 有效性处理策略

```cpp
enum class CompressionValidity : uint8_t {
    REQUIRES_VALIDITY,      // 需要单独压缩有效性列
    NO_VALIDITY_REQUIRED    // 压缩算法内部处理有效性
};
```

当压缩算法设置`NO_VALIDITY_REQUIRED`时，系统会自动将有效性列的压缩优化为`COMPRESSION_EMPTY`，节省存储空间。

## 4. 压缩函数注册机制

### 4.1 内部压缩方法数组

所有内置压缩算法通过静态数组注册：

```cpp
// src/function/compression_config.cpp

typedef CompressionFunction (*get_compression_function_t)(PhysicalType type);
typedef bool (*compression_supports_type_t)(const PhysicalType physical_type);

struct DefaultCompressionMethod {
    CompressionType type;
    get_compression_function_t get_function;
    compression_supports_type_t supports_type;
};

static const DefaultCompressionMethod internal_compression_methods[] = {
    {CompressionType::COMPRESSION_CONSTANT,
     ConstantFun::GetFunction, ConstantFun::TypeIsSupported},
    {CompressionType::COMPRESSION_UNCOMPRESSED,
     UncompressedFun::GetFunction, UncompressedFun::TypeIsSupported},
    {CompressionType::COMPRESSION_RLE,
     RLEFun::GetFunction, RLEFun::TypeIsSupported},
    {CompressionType::COMPRESSION_BITPACKING,
     BitpackingFun::GetFunction, BitpackingFun::TypeIsSupported},
    {CompressionType::COMPRESSION_DICTIONARY,
     DictionaryCompressionFun::GetFunction, DictionaryCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_CHIMP,
     ChimpCompressionFun::GetFunction, ChimpCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_PATAS,
     PatasCompressionFun::GetFunction, PatasCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_ALP,
     AlpCompressionFun::GetFunction, AlpCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_ALPRD,
     AlpRDCompressionFun::GetFunction, AlpRDCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_FSST,
     FSSTFun::GetFunction, FSSTFun::TypeIsSupported},
    {CompressionType::COMPRESSION_ZSTD,
     ZSTDFun::GetFunction, ZSTDFun::TypeIsSupported},
    {CompressionType::COMPRESSION_ROARING,
     RoaringCompressionFun::GetFunction, RoaringCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_EMPTY,
     EmptyValidityCompressionFun::GetFunction, EmptyValidityCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_DICT_FSST,
     DictFSSTCompressionFun::GetFunction, DictFSSTCompressionFun::TypeIsSupported},
    {CompressionType::COMPRESSION_AUTO, nullptr, nullptr}  // 终止标记
};
```

### 4.2 CompressionFunctionSet - 函数集合管理

```cpp
struct CompressionFunctionSet {
    static constexpr idx_t COMPRESSION_TYPE_COUNT = 15;
    static constexpr idx_t PHYSICAL_TYPE_COUNT = 19;

public:
    CompressionFunctionSet();

    //! 获取指定物理类型的所有可用压缩函数
    vector<reference<CompressionFunction>>
        GetCompressionFunctions(PhysicalType physical_type);

    //! 获取特定类型的压缩函数
    optional_ptr<CompressionFunction>
        GetCompressionFunction(CompressionType type, PhysicalType physical_type);

    //! 禁用指定的压缩方法
    void SetDisabledCompressionMethods(const vector<CompressionType> &methods);

    //! 获取被禁用的压缩方法列表
    vector<CompressionType> GetDisabledCompressionMethods() const;

private:
    mutex lock;
    atomic<bool> is_disabled[COMPRESSION_TYPE_COUNT];  // 禁用标记
    atomic<bool> is_loaded[PHYSICAL_TYPE_COUNT];       // 加载标记
    vector<vector<CompressionFunction>> functions;     // 函数集合

    void LoadCompressionFunctions(PhysicalType physical_type);
    static void TryLoadCompression(CompressionType type, PhysicalType physical_type,
                                   vector<CompressionFunction> &result);
};
```

### 4.3 延迟加载机制

压缩函数采用延迟加载策略，只有在首次需要时才加载：

```cpp
void CompressionFunctionSet::LoadCompressionFunctions(PhysicalType physical_type) {
    auto index = GetCompressionIndex(physical_type);
    auto &function_list = functions[index];

    if (is_loaded[index]) {
        return;  // 已加载
    }

    // 加锁进行加载
    lock_guard<mutex> guard(lock);

    // 双重检查
    if (is_loaded[index]) {
        return;
    }

    // 遍历所有注册的压缩方法
    for (idx_t i = 0; internal_compression_methods[i].get_function; i++) {
        TryLoadCompression(internal_compression_methods[i].type,
                          physical_type, function_list);
    }
    is_loaded[index] = true;
}

void CompressionFunctionSet::TryLoadCompression(
    CompressionType type,
    PhysicalType physical_type,
    vector<CompressionFunction> &result) {

    for (idx_t i = 0; internal_compression_methods[i].get_function; i++) {
        const auto &method = internal_compression_methods[i];
        if (method.type == type) {
            // 检查是否支持该物理类型
            if (!method.supports_type(physical_type)) {
                return;
            }
            // 创建压缩函数并加入列表
            result.push_back(method.get_function(physical_type));
            return;
        }
    }
    throw InternalException("Unsupported compression function type");
}
```

### 4.4 获取可用压缩函数

```cpp
vector<reference<CompressionFunction>>
CompressionFunctionSet::GetCompressionFunctions(PhysicalType physical_type) {
    LoadCompressionFunctions(physical_type);

    auto index = GetCompressionIndex(physical_type);
    auto &function_list = functions[index];

    vector<reference<CompressionFunction>> result;
    for (auto &entry : function_list) {
        auto compression_index = GetCompressionIndex(entry.type);

        // 跳过被禁用的方法
        if (is_disabled[compression_index]) {
            continue;
        }

        // 跳过不对外暴露的方法
        if (!EmitCompressionFunction(entry.type)) {
            continue;
        }

        result.push_back(entry);
    }
    return result;
}

// 控制哪些压缩类型对外可见
bool EmitCompressionFunction(CompressionType type) {
    switch (type) {
    case CompressionType::COMPRESSION_UNCOMPRESSED:
    case CompressionType::COMPRESSION_RLE:
    case CompressionType::COMPRESSION_BITPACKING:
    case CompressionType::COMPRESSION_DICTIONARY:
    case CompressionType::COMPRESSION_CHIMP:
    case CompressionType::COMPRESSION_PATAS:
    case CompressionType::COMPRESSION_ALP:
    case CompressionType::COMPRESSION_ALPRD:
    case CompressionType::COMPRESSION_FSST:
    case CompressionType::COMPRESSION_ZSTD:
    case CompressionType::COMPRESSION_ROARING:
    case CompressionType::COMPRESSION_DICT_FSST:
        return true;
    default:
        return false;  // CONSTANT, EMPTY等内部类型不对外暴露
    }
}
```

## 5. 压缩函数实现模式

### 5.1 工厂函数模式

每个压缩算法通过静态工厂类提供函数获取接口：

```cpp
// src/include/duckdb/function/compression/compression.hpp

struct RLEFun {
    static CompressionFunction GetFunction(PhysicalType type);
    static bool TypeIsSupported(const PhysicalType physical_type);
};

struct BitpackingFun {
    static CompressionFunction GetFunction(PhysicalType type);
    static bool TypeIsSupported(const PhysicalType physical_type);
};

struct DictionaryCompressionFun {
    static CompressionFunction GetFunction(PhysicalType type);
    static bool TypeIsSupported(const PhysicalType physical_type);
};
// ... 其他压缩算法
```

### 5.2 模板化实现

大多数压缩算法使用C++模板来支持多种数据类型：

```cpp
// RLE函数实现示例
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
    return true;
}

template <class T>
idx_t RLEFinalAnalyze(AnalyzeState &state) {
    auto &rle_state = state.template Cast<RLEAnalyzeState<T>>();
    // 返回预估的压缩后字节数
    return (sizeof(rle_count_t) + sizeof(T)) * rle_state.state.seen_count;
}
```

### 5.3 分析阶段评分策略

`final_analyze`函数返回评分，用于选择最优算法：

```cpp
// 评分含义：
// - 返回值 = 预估的压缩后字节数（越小越好）
// - 返回 DConstants::INVALID_INDEX = 跳过该算法

template <class T>
idx_t RLEFinalAnalyze(AnalyzeState &state) {
    auto &rle_state = state.template Cast<RLEAnalyzeState<T>>();
    // RLE压缩后大小 = 游程数 × (值大小 + 计数大小)
    return (sizeof(rle_count_t) + sizeof(T)) * rle_state.state.seen_count;
}
```

## 6. 完整调用流程

让我们通过一个完整的检查点流程来理解压缩系统的工作方式：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      Checkpoint压缩完整流程                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. ColumnDataCheckpointer::Checkpoint()                               │
│     │                                                                   │
│     ├─> 检查是否有变更                                                   │
│     │                                                                   │
│     └─> 2. WriteToDisk()                                               │
│            │                                                            │
│            ├─> 3. DetectBestCompressionMethod()                        │
│            │      │                                                     │
│            │      ├─> InitAnalyze()                                    │
│            │      │   为每个候选算法调用 init_analyze()                  │
│            │      │                                                     │
│            │      ├─> ScanSegments() + analyze()                       │
│            │      │   扫描数据，调用每个算法的 analyze()                 │
│            │      │                                                     │
│            │      └─> final_analyze()                                  │
│            │          计算评分，选择最优算法                             │
│            │                                                            │
│            ├─> ValidityCoveredByBasedata()                             │
│            │   检查是否可以优化有效性列压缩                              │
│            │                                                            │
│            ├─> init_compression()                                      │
│            │   初始化选定算法的压缩状态                                  │
│            │                                                            │
│            ├─> ScanSegments() + compress()                             │
│            │   扫描数据并压缩                                           │
│            │                                                            │
│            └─> compress_finalize()                                     │
│                完成压缩，刷新最后的段                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## 7. 源码文件索引

| 文件路径 | 内容描述 |
|---------|---------|
| `src/include/duckdb/function/compression_function.hpp` | 核心接口定义 |
| `src/include/duckdb/common/enums/compression_type.hpp` | 压缩类型枚举 |
| `src/include/duckdb/function/compression/compression.hpp` | 压缩函数工厂声明 |
| `src/function/compression_config.cpp` | 函数注册和配置 |
| `src/storage/table/column_data_checkpointer.cpp` | 检查点压缩流程 |

## 小结

本章深入分析了DuckDB压缩系统的架构设计：

1. **三阶段模型**：分析→压缩→存储的清晰流程划分
2. **状态类层次**：AnalyzeState、CompressionState、CompressedSegmentState各司其职
3. **CompressionFunction接口**：统一的函数指针抽象，支持算法热插拔
4. **注册机制**：静态数组+延迟加载的高效实现
5. **模板化实现**：利用C++模板支持多种数据类型

下一章将详细介绍压缩选择机制，包括评分系统、强制压缩和有效性优化等内容。
