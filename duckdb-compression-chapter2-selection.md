# DuckDB压缩系统深度解析（二）：压缩选择机制

## 引言

上一章介绍了DuckDB压缩系统的架构和接口设计。本章将深入探讨DuckDB如何从多个候选算法中选择最优的压缩方案。这个选择过程涉及三阶段评估流程、评分系统、强制压缩配置以及有效性列优化等多个方面。

## 1. 压缩选择流程概览

### 1.1 整体流程

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        压缩选择三阶段流程                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  阶段1: 初始化 (InitAnalyze)                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ • 获取物理类型对应的所有候选压缩函数                              │   │
│  │ • 为每个候选算法调用 init_analyze 创建分析状态                    │   │
│  │ • 应用强制压缩配置（如果有）                                      │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              v                                          │
│  阶段2: 分析 (Analyze)                                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ • 按顺序扫描所有数据段                                            │   │
│  │ • 对每个向量调用所有候选算法的 analyze 函数                       │   │
│  │ • analyze 返回 false 的算法被淘汰出局                             │   │
│  │ • 收集统计信息用于后续评分                                        │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                              │                                          │
│                              v                                          │
│  阶段3: 选择 (DetectBestCompressionMethod)                              │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ • 调用每个幸存算法的 final_analyze 获取评分                       │   │
│  │ • 选择评分最低的算法作为最优方案                                  │   │
│  │ • 如果是强制压缩，直接选择指定算法                                │   │
│  │ • 返回选中的压缩函数和分析状态                                    │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 核心数据结构

```cpp
// 分析结果封装
struct CheckpointAnalyzeResult {
    unique_ptr<AnalyzeState> analyze_state;  // 分析状态
    optional_ptr<CompressionFunction> function;  // 选中的压缩函数
};

// 检查点器主类
class ColumnDataCheckpointer {
    vector<reference<ColumnCheckpointState>> checkpoint_states;
    StorageManager &storage_manager;
    const RowGroup &row_group;
    Vector intermediate;
    ColumnCheckpointInfo &checkpoint_info;

    // 每列的候选压缩函数
    vector<vector<optional_ptr<CompressionFunction>>> compression_functions;
    // 每列每个算法的分析状态
    vector<vector<unique_ptr<AnalyzeState>>> analyze_states;
};
```

## 2. 阶段一：初始化分析

### 2.1 获取候选压缩函数

在检查点器构造时，系统为每列获取所有候选压缩函数：

```cpp
// src/storage/table/column_data_checkpointer.cpp

ColumnDataCheckpointer::ColumnDataCheckpointer(
    vector<reference<ColumnCheckpointState>> &checkpoint_states,
    StorageManager &storage_manager,
    const RowGroup &row_group,
    ColumnCheckpointInfo &checkpoint_info)
    : checkpoint_states(checkpoint_states),
      storage_manager(storage_manager),
      row_group(row_group),
      intermediate(CreateIntermediateVector(checkpoint_states)),
      checkpoint_info(checkpoint_info) {

    auto &db = storage_manager.GetDatabase();
    auto &config = DBConfig::GetConfig(db);

    compression_functions.resize(checkpoint_states.size());

    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &col_data = checkpoint_states[i].get().original_column;

        // 根据列的物理类型获取所有可用的压缩函数
        auto to_add = config.GetCompressionFunctions(
            col_data.type.InternalType());

        auto &functions = compression_functions[i];
        for (auto &func : to_add) {
            functions.push_back(&func.get());
        }
    }
}
```

### 2.2 处理强制压缩配置

系统支持两级强制压缩配置：

```cpp
CompressionType ForceCompression(
    StorageManager &storage_manager,
    vector<optional_ptr<CompressionFunction>> &compression_functions,
    CompressionType compression_type) {

    // 检查强制压缩类型是否在候选列表中
    bool found = false;
    for (idx_t i = 0; i < compression_functions.size(); i++) {
        auto &compression_function = *compression_functions[i];
        if (compression_function.type == compression_type) {
            found = true;
            break;
        }
    }

    if (!found) {
        return CompressionType::COMPRESSION_AUTO;
    }

    // 清除其他压缩方法，只保留强制方法和Uncompressed作为回退
    for (idx_t i = 0; i < compression_functions.size(); i++) {
        auto &compression_function = *compression_functions[i];
        if (compression_function.type == CompressionType::COMPRESSION_UNCOMPRESSED) {
            continue;  // 保留Uncompressed作为回退选项
        }
        if (compression_function.type != compression_type) {
            compression_functions[i] = nullptr;  // 清除其他方法
        }
    }
    return compression_type;
}
```

### 2.3 初始化分析状态

```cpp
void ColumnDataCheckpointer::InitAnalyze() {
    analyze_states.resize(checkpoint_states.size());

    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &functions = compression_functions[i];
        auto &states = analyze_states[i];
        auto &checkpoint_state = checkpoint_states[i];
        auto &coldata = checkpoint_state.get().GetResultColumn();

        states.resize(functions.size());

        for (idx_t j = 0; j < functions.size(); j++) {
            auto &func = functions[j];
            if (!func) {
                continue;  // 跳过已被清除的函数
            }
            // 调用每个算法的初始化分析函数
            states[j] = func->init_analyze(coldata, coldata.type.InternalType());
        }
    }
}
```

## 3. 阶段二：数据扫描与分析

### 3.1 段扫描机制

系统按顺序扫描所有数据段，并将每个向量提交给所有候选算法进行分析：

```cpp
void ColumnDataCheckpointer::ScanSegments(
    const std::function<void(Vector &, idx_t)> &callback) {

    Vector scan_vector(intermediate.GetType(), nullptr);
    auto &first_state = checkpoint_states[0];
    auto &col_data = first_state.get().original_column;

    // 遍历所有段节点
    for (auto &segment_node : col_data.data.SegmentNodes()) {
        auto &segment = segment_node.GetNode();
        ColumnScanState scan_state(nullptr);
        scan_state.current = segment_node;
        segment.InitializeScan(scan_state);

        // 每次扫描一个向量大小的数据
        for (idx_t base_row_index = 0;
             base_row_index < segment.count;
             base_row_index += STANDARD_VECTOR_SIZE) {

            scan_vector.Reference(intermediate);

            idx_t count = MinValue<idx_t>(
                segment.count - base_row_index,
                STANDARD_VECTOR_SIZE);

            scan_state.offset_in_column =
                segment_node.GetRowStart() + base_row_index;

            col_data.CheckpointScan(segment, scan_state, count, scan_vector);

            // 调用回调处理扫描到的数据
            callback(scan_vector, count);
        }
    }
}
```

### 3.2 分析回调逻辑

在选择阶段，分析回调会将数据提交给所有候选算法：

```cpp
// 在 DetectBestCompressionMethod 中
ScanSegments([&](Vector &scan_vector, idx_t count) {
    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &functions = compression_functions[i];
        auto &states = analyze_states[i];

        for (idx_t j = 0; j < functions.size(); j++) {
            auto &state = states[j];
            auto &func = functions[j];

            if (!state) {
                continue;  // 已被淘汰
            }

            // 调用 analyze 函数
            // 返回 false 表示该算法不适合此数据
            if (!func->analyze(*state, scan_vector, count)) {
                state = nullptr;   // 淘汰该算法
                func = nullptr;
            }
        }
    }
});
```

### 3.3 分析函数的淘汰机制

每个压缩算法可以通过`analyze`函数返回`false`来主动退出竞争。常见的退出场景：

```cpp
// 示例：FSST分析函数可能的退出逻辑
bool FSSTAnalyze(AnalyzeState &state, Vector &input, idx_t count) {
    auto &fsst_state = state.Cast<FSSTAnalyzeState>();

    // 如果字符串过长，退出
    if (HasVeryLongStrings(input, count)) {
        return false;
    }

    // 如果数据熵过高，退出
    if (fsst_state.entropy > FSST_MAX_ENTROPY) {
        return false;
    }

    // 正常分析
    AnalyzeStrings(fsst_state, input, count);
    return true;
}

// 示例：Dictionary分析函数
bool DictionaryAnalyze(AnalyzeState &state, Vector &input, idx_t count) {
    auto &dict_state = state.Cast<DictionaryAnalyzeState>();

    // 字典过大，退出
    if (dict_state.dictionary_size > MAX_DICTIONARY_SIZE) {
        return false;
    }

    // 添加新值到字典
    AddToDictionary(dict_state, input, count);
    return true;
}
```

## 4. 阶段三：最优算法选择

### 4.1 完整选择流程

```cpp
vector<CheckpointAnalyzeResult>
ColumnDataCheckpointer::DetectBestCompressionMethod() {
    D_ASSERT(!compression_functions.empty());

    auto &db = storage_manager.GetDatabase();
    auto &config = DBConfig::GetConfig(db);

    vector<CompressionType> forced_methods(
        checkpoint_states.size(),
        CompressionType::COMPRESSION_AUTO);

    // 处理强制压缩配置
    auto compression_type = checkpoint_info.GetCompressionType();
    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &functions = compression_functions[i];

        // 优先使用列级强制配置
        if (compression_type != CompressionType::COMPRESSION_AUTO) {
            forced_methods[i] = ForceCompression(
                storage_manager, functions, compression_type);
        }
        // 其次使用全局强制配置
        if (compression_type == CompressionType::COMPRESSION_AUTO &&
            config.options.force_compression != CompressionType::COMPRESSION_AUTO) {
            forced_methods[i] = ForceCompression(
                storage_manager, functions, config.options.force_compression);
        }
    }

    // 初始化分析状态
    InitAnalyze();

    // 扫描数据进行分析
    ScanSegments([&](Vector &scan_vector, idx_t count) {
        // ... 分析逻辑（见上节）
    });

    // 选择最优算法
    vector<CheckpointAnalyzeResult> result;
    result.resize(checkpoint_states.size());

    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &functions = compression_functions[i];
        auto &states = analyze_states[i];
        auto &forced_method = forced_methods[i];

        unique_ptr<AnalyzeState> chosen_state;
        idx_t best_score = NumericLimits<idx_t>::Maximum();
        idx_t compression_idx = DConstants::INVALID_INDEX;

        D_ASSERT(functions.size() == states.size());

        for (idx_t j = 0; j < functions.size(); j++) {
            auto &function = functions[j];
            auto &state = states[j];

            if (!state) {
                continue;  // 已被淘汰
            }

            // 检查是否是强制方法
            bool forced_method_found = function->type == forced_method;

            // 调用 final_analyze 获取评分
            auto score = function->final_analyze(*state);

            // INVALID_INDEX 表示该算法不可用
            if (score == DConstants::INVALID_INDEX) {
                continue;
            }

            // 选择评分更低的算法，或者如果是强制方法直接选择
            if (score < best_score || forced_method_found) {
                compression_idx = j;
                best_score = score;
                chosen_state = std::move(state);
            }

            // 找到强制方法后直接结束
            if (forced_method_found) {
                break;
            }
        }

        auto &checkpoint_state = checkpoint_states[i];
        auto &col_data = checkpoint_state.get().GetResultColumn();

        if (!chosen_state) {
            throw FatalException(
                "No suitable compression/storage method found for type %s",
                col_data.type.ToString());
        }

        auto &best_function = *functions[compression_idx];
        result[i] = CheckpointAnalyzeResult(
            std::move(chosen_state), best_function);
    }

    return result;
}
```

### 4.2 评分系统设计

评分系统的设计原则：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           评分系统设计                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  评分值含义:                                                             │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │ • 理想情况：精确的压缩后字节数                                  │     │
│  │ • 实际情况：可以是合理的估算值                                  │     │
│  │ • 越小越好：评分低表示压缩效果好                               │     │
│  │ • INVALID_INDEX：表示该算法不可用                              │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  各算法评分计算示例:                                                     │
│                                                                         │
│  RLE:                                                                   │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │ score = (sizeof(rle_count_t) + sizeof(T)) × seen_count        │     │
│  │ 即：(2字节计数 + 值大小) × 游程数                              │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  Bitpacking:                                                            │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │ score = (bit_width × count) / 8 + metadata_size               │     │
│  │ 即：位宽 × 值数量 / 8 + 元数据开销                             │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  Dictionary:                                                            │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │ score = dictionary_size + index_size × count                  │     │
│  │ 即：字典大小 + 索引大小 × 值数量                               │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  Uncompressed:                                                          │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │ score = sizeof(T) × count                                     │     │
│  │ 即：原始大小，作为基准                                         │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 4.3 RLE评分实现

```cpp
// src/storage/compression/rle.cpp

template <class T>
idx_t RLEFinalAnalyze(AnalyzeState &state) {
    auto &rle_state = state.template Cast<RLEAnalyzeState<T>>();

    // 压缩后大小 = 游程数 × 每个游程的存储大小
    // 每个游程存储：值(sizeof(T)) + 计数(2字节)
    return (sizeof(rle_count_t) + sizeof(T)) * rle_state.state.seen_count;
}
```

**评分分析**：
- 如果数据高度重复，游程数少，评分低，RLE胜出
- 如果数据几乎不重复，游程数接近值数量，评分高，RLE被淘汰

## 5. 有效性列优化

### 5.1 ValidityCoveredByBasedata机制

DuckDB的每个列实际包含两个子列：数据列和有效性列（NULL位图）。系统会检查是否可以优化有效性列的压缩：

```cpp
bool ColumnDataCheckpointer::ValidityCoveredByBasedata(
    vector<CheckpointAnalyzeResult> &result) {

    // 只在有两列（数据列+有效性列）时适用
    if (result.size() != 2) {
        return false;
    }

    auto &base = result[0];
    D_ASSERT(base.function);

    // 检查基础压缩函数是否声明不需要单独的有效性压缩
    return base.function->validity == CompressionValidity::NO_VALIDITY_REQUIRED;
}
```

### 5.2 有效性优化应用

```cpp
void ColumnDataCheckpointer::WriteToDisk() {
    auto analyze_result = DetectBestCompressionMethod();

    // 检查是否可以优化有效性列
    if (ValidityCoveredByBasedata(analyze_result)) {
        D_ASSERT(analyze_result.size() == 2);
        auto &validity = analyze_result[1];

        auto &db = storage_manager.GetDatabase();
        auto &config = DBConfig::GetConfig(db);

        // 将有效性列的压缩改为 COMPRESSION_EMPTY
        // 这是一个 no-op，只保存一个空段
        validity.function = config.GetCompressionFunction(
            CompressionType::COMPRESSION_EMPTY,
            PhysicalType::BIT);
    }

    // 继续压缩流程...
}
```

### 5.3 CompressionValidity的作用

```cpp
enum class CompressionValidity : uint8_t {
    REQUIRES_VALIDITY,      // 需要单独压缩有效性列
    NO_VALIDITY_REQUIRED    // 压缩函数内部处理有效性
};
```

**典型场景**：
- RLE可以在游程中记录NULL信息，可设置`NO_VALIDITY_REQUIRED`
- Bitpacking通常只处理非NULL值，需要`REQUIRES_VALIDITY`
- Dictionary可以用特殊字典索引表示NULL，可设置`NO_VALIDITY_REQUIRED`

## 6. 强制压缩配置

### 6.1 配置层级

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         强制压缩配置层级                                 │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  优先级从高到低:                                                         │
│                                                                         │
│  1. 列级配置 (checkpoint_info.GetCompressionType())                     │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ • 在 CREATE TABLE 时指定                                     │    │
│     │ • 例如: column_name TYPE USING compression_type              │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              v                                          │
│  2. 全局配置 (config.options.force_compression)                         │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ • 通过 SET force_compression='type' 设置                     │    │
│     │ • 影响所有后续的压缩操作                                     │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                              │                                          │
│                              v                                          │
│  3. 自动选择 (COMPRESSION_AUTO)                                         │
│     ┌─────────────────────────────────────────────────────────────┐    │
│     │ • 默认行为                                                   │    │
│     │ • 根据数据特征自动选择最优算法                               │    │
│     └─────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.2 强制压缩的实现细节

```cpp
CompressionType ForceCompression(
    StorageManager &storage_manager,
    vector<optional_ptr<CompressionFunction>> &compression_functions,
    CompressionType compression_type) {

    // 1. 验证强制类型是否可用
    bool found = false;
    for (idx_t i = 0; i < compression_functions.size(); i++) {
        auto &compression_function = *compression_functions[i];
        if (compression_function.type == compression_type) {
            found = true;
            break;
        }
    }

    // 类型不可用则退回自动选择
    if (!found) {
        return CompressionType::COMPRESSION_AUTO;
    }

    // 2. 清除其他压缩方法
    for (idx_t i = 0; i < compression_functions.size(); i++) {
        auto &compression_function = *compression_functions[i];

        // 始终保留 Uncompressed 作为回退
        if (compression_function.type ==
            CompressionType::COMPRESSION_UNCOMPRESSED) {
            continue;
        }

        // 清除非强制类型
        if (compression_function.type != compression_type) {
            compression_functions[i] = nullptr;
        }
    }

    return compression_type;
}
```

### 6.3 使用示例

```sql
-- 全局强制使用特定压缩
SET force_compression='zstd';

-- 查看当前设置
SELECT current_setting('force_compression');

-- 恢复自动选择
SET force_compression='auto';
```

## 7. 禁用压缩方法

### 7.1 禁用机制

系统支持禁用特定的压缩方法：

```cpp
// CompressionFunctionSet 中的禁用逻辑
void CompressionFunctionSet::SetDisabledCompressionMethods(
    const vector<CompressionType> &methods) {

    ResetDisabledMethods();
    for (auto &method : methods) {
        auto idx = GetCompressionIndex(method);
        is_disabled[idx] = true;
    }
}

// 获取压缩函数时检查禁用状态
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

        if (!EmitCompressionFunction(entry.type)) {
            continue;
        }

        result.push_back(entry);
    }
    return result;
}
```

### 7.2 配置接口

```cpp
// DBConfig 提供的接口
void DBConfig::SetDisabledCompressionMethods(
    const vector<CompressionType> &methods) {
    compression_functions->SetDisabledCompressionMethods(methods);
}

vector<CompressionType> DBConfig::GetDisabledCompressionMethods() const {
    return compression_functions->GetDisabledCompressionMethods();
}
```

## 8. 完整压缩流程

### 8.1 WriteToDisk实现

```cpp
void ColumnDataCheckpointer::WriteToDisk() {
    // 1. 选择最优压缩方法
    auto analyze_result = DetectBestCompressionMethod();

    // 2. 优化有效性列压缩
    if (ValidityCoveredByBasedata(analyze_result)) {
        D_ASSERT(analyze_result.size() == 2);
        auto &validity = analyze_result[1];
        auto &db = storage_manager.GetDatabase();
        auto &config = DBConfig::GetConfig(db);
        validity.function = config.GetCompressionFunction(
            CompressionType::COMPRESSION_EMPTY, PhysicalType::BIT);
    }

    // 3. 初始化压缩状态
    D_ASSERT(analyze_result.size() == checkpoint_states.size());
    vector<ColumnDataCheckpointData> checkpoint_data(checkpoint_states.size());
    vector<unique_ptr<CompressionState>> compression_states(
        checkpoint_states.size());

    for (idx_t i = 0; i < analyze_result.size(); i++) {
        auto &analyze_state = analyze_result[i].analyze_state;
        auto &function = analyze_result[i].function;

        auto &checkpoint_state = checkpoint_states[i];
        auto &col_data = checkpoint_state.get().GetResultColumn();

        checkpoint_data[i] = ColumnDataCheckpointData(
            checkpoint_state, col_data, col_data.GetDatabase(),
            row_group, storage_manager);

        // 调用 init_compression，传入分析状态
        compression_states[i] = function->init_compression(
            checkpoint_data[i], std::move(analyze_state));
    }

    // 4. 扫描数据进行压缩
    ScanSegments([&](Vector &scan_vector, idx_t count) {
        for (idx_t i = 0; i < checkpoint_states.size(); i++) {
            auto &function = analyze_result[i].function;
            auto &compression_state = compression_states[i];

            function->compress(*compression_state, scan_vector, count);
        }
    });

    // 5. 完成压缩
    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &function = analyze_result[i].function;
        auto &compression_state = compression_states[i];

        function->compress_finalize(*compression_state);
    }

    // 6. 释放旧段
    DropSegments();
}
```

### 8.2 检查点入口

```cpp
void ColumnDataCheckpointer::Checkpoint() {
    // 检查是否有任何变更
    for (idx_t i = 0; i < checkpoint_states.size(); i++) {
        auto &state = checkpoint_states[i];
        auto &col_data = state.get().original_column;
        if (col_data.HasChanges()) {
            has_changes = true;
            break;
        }
    }

    if (!has_changes) {
        // 没有变更，只需标记块为已检查点
        CheckpointBlockIdMarker marker(storage_manager.GetBlockManager());
        for (idx_t i = 0; i < checkpoint_states.size(); i++) {
            auto &state = checkpoint_states[i];
            auto &col_data = state.get().original_column;
            col_data.VisitBlockIds(marker);
        }
        return;
    }

    // 有变更，执行完整的压缩流程
    WriteToDisk();
}
```

## 9. 源码文件索引

| 文件路径 | 内容描述 |
|---------|---------|
| `src/storage/table/column_data_checkpointer.cpp` | 压缩选择核心逻辑 |
| `src/storage/table/column_data_checkpointer.hpp` | 检查点器头文件 |
| `src/function/compression_config.cpp` | 压缩函数配置与禁用 |
| `src/main/config.cpp` | 全局压缩配置 |

## 小结

本章详细分析了DuckDB的压缩选择机制：

1. **三阶段流程**：初始化→分析→选择，逐步筛选最优算法
2. **评分系统**：基于压缩后字节数估算，越小越好
3. **淘汰机制**：analyze返回false或final_analyze返回INVALID_INDEX时淘汰
4. **有效性优化**：当基础压缩可处理NULL时，有效性列使用EMPTY压缩
5. **强制配置**：支持列级和全局两级强制压缩配置
6. **禁用机制**：支持禁用特定压缩算法

下一章将深入分析整数压缩算法：Bitpacking和RLE的实现细节。
