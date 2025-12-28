# DuckDB 扩展系统深度解析（一）：架构概述

## 引言

DuckDB 的扩展系统是其模块化架构的核心，通过扩展机制可以在不修改核心代码的情况下添加新的函数、数据类型、文件格式支持等功能。本章将深入分析扩展系统的整体架构、核心抽象以及 ABI 兼容性设计。

## 1. 扩展系统设计理念

### 1.1 模块化与可插拔架构

DuckDB 采用模块化设计，将功能划分为核心引擎和可选扩展两部分：

```
┌─────────────────────────────────────────────────────────────────────┐
│                         DuckDB 应用层                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│
│  │   Parquet   │  │    JSON     │  │     ICU     │  │   Spatial   ││
│  │   Extension │  │  Extension  │  │  Extension  │  │  Extension  ││
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘│
│         │                │                │                │        │
│         └────────────────┴────────────────┴────────────────┘        │
│                                 │                                    │
│                    ┌────────────┴────────────┐                      │
│                    │    ExtensionLoader      │                      │
│                    │   (注册接口层)           │                      │
│                    └────────────┬────────────┘                      │
│                                 │                                    │
├─────────────────────────────────┼────────────────────────────────────┤
│                                 │                                    │
│  ┌──────────────────────────────┴──────────────────────────────────┐│
│  │                      DuckDB 核心引擎                             ││
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐            ││
│  │  │ Catalog │  │ Executor│  │ Storage │  │ Planner │            ││
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘            ││
│  └──────────────────────────────────────────────────────────────────┘│
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 扩展加载模式

DuckDB 支持两种扩展加载模式：

| 模式 | 特点 | 适用场景 |
|-----|------|---------|
| **静态链接** | 编译时链接到主程序 | 核心功能、嵌入式部署 |
| **动态加载** | 运行时加载 .duckdb_extension | 可选功能、按需加载 |

```cpp
// 静态链接的扩展在编译时直接包含
#ifdef BUILD_PARQUET_EXTENSION
    ParquetExtension::Load(loader);
#endif

// 动态加载的扩展在运行时按需加载
INSTALL parquet;
LOAD parquet;
```

### 1.3 扩展分类

DuckDB 扩展按来源和信任级别分类：

```
┌─────────────────────────────────────────────────────────────────────┐
│                          扩展分类                                    │
├─────────────────┬──────────────────┬────────────────────────────────┤
│                 │                  │                                │
│    核心扩展      │    官方扩展       │      社区扩展                   │
│  Core Extensions│ Official Extensions│ Community Extensions         │
│                 │                  │                                │
│  - core_functions│  - parquet      │  - spatial                    │
│  - autocomplete │  - json          │  - excel                      │
│                 │  - icu           │  - mysql                      │
│                 │  - httpfs        │  - postgres                   │
│                 │  - fts           │  - sqlite                     │
│                 │                  │                                │
│    随主程序编译   │   官方仓库分发     │    社区仓库分发                 │
│    最高信任级别   │   签名验证        │    可选信任                    │
└─────────────────┴──────────────────┴────────────────────────────────┘
```

## 2. Extension 基类设计

### 2.1 Extension 抽象类

所有扩展的基类定义了扩展的核心接口：

```cpp
// src/include/duckdb/main/extension.hpp

class Extension {
public:
    DUCKDB_API virtual ~Extension();

    //! 加载扩展，注册函数、类型等
    DUCKDB_API virtual void Load(ExtensionLoader &loader) = 0;

    //! 返回扩展名称
    DUCKDB_API virtual std::string Name() = 0;

    //! 返回扩展版本（可选）
    DUCKDB_API virtual std::string Version() const {
        return "";
    }

    //! 默认版本（DuckDB 版本）
    DUCKDB_API static const char *DefaultVersion();
};
```

### 2.2 扩展生命周期

```
┌────────────────────────────────────────────────────────────────────┐
│                       扩展生命周期                                  │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐       │
│   │ INSTALL │───▶│  LOAD   │───▶│ ACTIVE  │───▶│ UNLOAD  │       │
│   │ (可选)   │    │         │    │         │    │ (重启)  │       │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘       │
│        │              │              │              │              │
│        ▼              ▼              ▼              ▼              │
│   下载扩展文件     解析元数据      注册到 Catalog   清理资源         │
│   写入本地目录     验证签名        可被查询使用     (当前不支持)      │
│   记录安装信息     调用 Load()                                      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### 2.3 扩展实现示例

```cpp
// extension/core_functions/core_functions_extension.cpp

class CoreFunctionsExtension : public Extension {
public:
    void Load(ExtensionLoader &loader) override {
        // 注册所有核心函数
        FunctionList::RegisterExtensionFunctions(
            loader, CoreFunctionList::GetFunctionList());
    }

    std::string Name() override {
        return "core_functions";
    }

    std::string Version() const override {
#ifdef EXT_VERSION_CORE_FUNCTIONS
        return EXT_VERSION_CORE_FUNCTIONS;
#else
        return "";
#endif
    }
};
```

## 3. ExtensionABIType：ABI 兼容性

### 3.1 ABI 类型枚举

DuckDB 支持多种扩展 ABI 类型，以平衡功能与兼容性：

```cpp
enum class ExtensionABIType : uint8_t {
    UNKNOWN = 0,

    //! C++ ABI：版本需要精确匹配
    //! 使用 C++ 类和虚函数，依赖编译器 ABI
    CPP = 1,

    //! C ABI：使用 duckdb_ext_api_v1 结构
    //! 版本需要相等或更高（向后兼容）
    C_STRUCT = 2,

    //! C ABI 不稳定版本：包含"不稳定"函数
    //! 版本需要精确匹配
    C_STRUCT_UNSTABLE = 3
};
```

### 3.2 ABI 类型对比

| ABI 类型 | 版本要求 | 兼容性 | 性能 | 适用场景 |
|---------|---------|--------|-----|---------|
| CPP | 精确匹配 | 最低 | 最高 | 核心扩展、高性能需求 |
| C_STRUCT | >= 最低版本 | 最高 | 中等 | 跨版本分发、第三方扩展 |
| C_STRUCT_UNSTABLE | 精确匹配 | 中等 | 中等 | 使用新 API 的开发版 |

### 3.3 版本兼容性检查

```cpp
struct VersioningUtils {
    //! 解析语义化版本 v{major}.{minor}.{patch}
    static bool ParseSemver(string &semver,
                            idx_t &major_out,
                            idx_t &minor_out,
                            idx_t &patch_out);

    //! 检查 C API 版本是否兼容
    static bool IsSupportedCAPIVersion(string &capi_version_string);
    static bool IsSupportedCAPIVersion(idx_t major, idx_t minor, idx_t patch);
};
```

版本兼容性规则：

```
C_STRUCT 兼容性规则：
- 主版本号必须相同
- 扩展版本 <= DuckDB 版本

示例：
DuckDB v1.2.0 可加载：
  ✅ 扩展 v1.2.0
  ✅ 扩展 v1.1.0
  ✅ 扩展 v1.0.0
  ❌ 扩展 v1.3.0 (太新)
  ❌ 扩展 v0.9.0 (主版本不同)
```

## 4. 扩展元数据结构

### 4.1 ParsedExtensionMetaData

每个扩展文件末尾包含 512 字节的元数据 Footer：

```cpp
struct ParsedExtensionMetaData {
    static constexpr const idx_t FOOTER_SIZE = 512;
    static constexpr const idx_t SIGNATURE_SIZE = 256;
    static constexpr const char *EXPECTED_MAGIC_VALUE = {
        "4\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"};

    //! 魔数，用于验证文件格式
    string magic_value;

    //! ABI 类型
    ExtensionABIType abi_type;

    //! 目标平台（如 linux_amd64）
    string platform;

    //! DuckDB 版本（CPP/C_STRUCT_UNSTABLE）
    string duckdb_version;

    //! C API 版本（C_STRUCT）
    string duckdb_capi_version;

    //! 扩展版本
    string extension_version;

    //! 数字签名
    string signature;

    //! 扩展 ABI 元数据
    string extension_abi_metadata;

    //! 验证元数据是否有效
    bool AppearsValid() {
        return magic_value == EXPECTED_MAGIC_VALUE;
    }

    //! 获取元数据不匹配的错误描述
    string GetInvalidMetadataError();
};
```

### 4.2 元数据布局

```
┌────────────────────────────────────────────────────────────────────┐
│                    扩展文件结构                                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                      扩展二进制代码                           │ │
│  │                      (动态库内容)                             │ │
│  │                           ...                                 │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐ │
│  │                   元数据 Footer (512 字节)                    │ │
│  ├──────────────────────────────────────────────────────────────┤ │
│  │  Magic Value (32 bytes)     │ "4\0\0\0..."                   │ │
│  ├─────────────────────────────┼────────────────────────────────┤ │
│  │  ABI Type (1 byte)          │ CPP/C_STRUCT/C_STRUCT_UNSTABLE │ │
│  ├─────────────────────────────┼────────────────────────────────┤ │
│  │  Platform (variable)        │ "linux_amd64"                  │ │
│  ├─────────────────────────────┼────────────────────────────────┤ │
│  │  DuckDB Version (variable)  │ "v1.2.0"                       │ │
│  ├─────────────────────────────┼────────────────────────────────┤ │
│  │  Extension Version (var)    │ "v1.0.0"                       │ │
│  ├─────────────────────────────┼────────────────────────────────┤ │
│  │  Signature (256 bytes)      │ RSA 签名                       │ │
│  └──────────────────────────────────────────────────────────────┘ │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### 4.3 元数据解析

```cpp
// src/main/extension/extension_helper.cpp

ParsedExtensionMetaData ExtensionHelper::ParseExtensionMetaData(
    FileHandle &handle) {

    // 读取文件末尾 512 字节
    auto file_size = handle.GetFileSize();
    auto metadata_offset = file_size - ParsedExtensionMetaData::FOOTER_SIZE;

    char metadata[ParsedExtensionMetaData::FOOTER_SIZE];
    handle.Read(metadata, ParsedExtensionMetaData::FOOTER_SIZE, metadata_offset);

    return ParseExtensionMetaData(metadata);
}

ParsedExtensionMetaData ExtensionHelper::ParseExtensionMetaData(
    const char *metadata) noexcept {

    ParsedExtensionMetaData result;

    // 解析魔数
    result.magic_value = string(metadata, 32);

    // 解析 ABI 类型
    result.abi_type = static_cast<ExtensionABIType>(metadata[32]);

    // 解析其他字段...

    return result;
}
```

## 5. ExtensionManager 管理器

### 5.1 ExtensionManager 结构

ExtensionManager 跟踪已加载扩展的状态：

```cpp
// src/include/duckdb/main/extension_manager.hpp

class ExtensionInfo {
public:
    ExtensionInfo();

    //! 保护并发访问的锁
    mutex lock;
    //! 是否已加载
    atomic<bool> is_loaded;
    //! 安装信息
    unique_ptr<ExtensionInstallInfo> install_info;
    //! 加载信息（如描述）
    unique_ptr<ExtensionLoadedInfo> load_info;
};

class ExtensionManager {
public:
    explicit ExtensionManager(DatabaseInstance &db);

    //! 检查扩展是否已加载
    DUCKDB_API bool ExtensionIsLoaded(const string &name);

    //! 获取所有已加载扩展名称
    DUCKDB_API vector<string> GetExtensions();

    //! 获取扩展信息
    DUCKDB_API optional_ptr<ExtensionInfo> GetExtensionInfo(const string &name);

    //! 开始加载扩展
    DUCKDB_API unique_ptr<ExtensionActiveLoad> BeginLoad(const string &extension);

    //! 获取数据库实例的 ExtensionManager
    DUCKDB_API static ExtensionManager &Get(DatabaseInstance &db);
    DUCKDB_API static ExtensionManager &Get(ClientContext &context);

private:
    DatabaseInstance &db;
    mutex lock;
    //! 已加载扩展映射：名称 -> ExtensionInfo
    unordered_map<string, unique_ptr<ExtensionInfo>> loaded_extensions_info;
};
```

### 5.2 ExtensionActiveLoad 活跃加载

管理扩展加载过程中的状态：

```cpp
class ExtensionActiveLoad {
public:
    ExtensionActiveLoad(DatabaseInstance &db,
                        ExtensionInfo &info,
                        string extension_name);

    DatabaseInstance &db;
    //! 持有加载锁，防止并发加载
    unique_lock<mutex> load_lock;
    ExtensionInfo &info;
    string extension_name;

public:
    //! 完成加载
    void FinishLoad(ExtensionInstallInfo &install_info);
    //! 加载失败
    void LoadFail(const ErrorData &error);
};
```

### 5.3 扩展状态查询

```sql
-- 查询已安装的扩展
SELECT * FROM duckdb_extensions();

-- 输出示例：
-- ┌──────────────────┬─────────┬───────────┬─────────────┐
-- │  extension_name  │ loaded  │ installed │   version   │
-- ├──────────────────┼─────────┼───────────┼─────────────┤
-- │ core_functions   │ true    │ true      │ v1.2.0      │
-- │ parquet          │ false   │ true      │ v1.2.0      │
-- │ json             │ true    │ true      │ v1.2.0      │
-- └──────────────────┴─────────┴───────────┴─────────────┘
```

## 6. ExtensionLoader 加载上下文

### 6.1 ExtensionLoader 接口

ExtensionLoader 是扩展注册功能的主要接口：

```cpp
// src/include/duckdb/main/extension/extension_loader.hpp

class ExtensionLoader {
public:
    explicit ExtensionLoader(ExtensionActiveLoad &load_info);
    ExtensionLoader(DatabaseInstance &db, const string &extension_name);

    //! 获取数据库实例
    DUCKDB_API DatabaseInstance &GetDatabaseInstance();

    //! 设置扩展描述
    DUCKDB_API void SetDescription(const string &description);

    //! 注册函数
    DUCKDB_API void RegisterFunction(ScalarFunction function);
    DUCKDB_API void RegisterFunction(AggregateFunction function);
    DUCKDB_API void RegisterFunction(TableFunction function);
    DUCKDB_API void RegisterFunction(PragmaFunction function);
    DUCKDB_API void RegisterFunction(CopyFunction function);

    //! 注册类型
    DUCKDB_API void RegisterType(string type_name, LogicalType type,
                                 bind_logical_type_function_t bind_function = nullptr);

    //! 注册类型转换
    DUCKDB_API void RegisterCastFunction(const LogicalType &source,
                                         const LogicalType &target,
                                         BoundCastInfo function,
                                         int64_t implicit_cast_cost = -1);

    //! 添加函数重载
    DUCKDB_API void AddFunctionOverload(ScalarFunction function);

private:
    DatabaseInstance &db;
    string extension_name;
    string extension_description;
    optional_ptr<ExtensionInfo> extension_info;
};
```

### 6.2 C++ 扩展入口宏

```cpp
//! 定义 C++ 扩展入口点的宏
#define DUCKDB_CPP_EXTENSION_ENTRY(EXTENSION_NAME, LOADER_NAME)        \
    DUCKDB_EXTENSION_API void EXTENSION_NAME##_duckdb_cpp_init(        \
        duckdb::ExtensionLoader &LOADER_NAME)

// 使用示例：
DUCKDB_CPP_EXTENSION_ENTRY(my_extension, loader) {
    loader.SetDescription("My custom extension");
    loader.RegisterFunction(MyScalarFunction());
    loader.RegisterFunction(MyTableFunction());
}
```

## 7. 扩展与 Catalog 集成

### 7.1 注册到系统 Catalog

扩展注册的函数和类型存储在系统 Catalog 中：

```cpp
// src/main/extension/extension_loader.cpp

void ExtensionLoader::RegisterFunction(ScalarFunction function) {
    ScalarFunctionSet set(function.name);
    set.AddFunction(std::move(function));
    RegisterFunction(std::move(set));
}

void ExtensionLoader::RegisterFunction(ScalarFunctionSet function) {
    CreateScalarFunctionInfo info(std::move(function));
    // 冲突时合并重载
    info.on_conflict = OnCreateConflict::ALTER_ON_CONFLICT;
    RegisterFunction(std::move(info));
}

void ExtensionLoader::RegisterFunction(CreateScalarFunctionInfo function) {
    D_ASSERT(!function.functions.name.empty());
    auto &system_catalog = Catalog::GetSystemCatalog(db);
    auto data = CatalogTransaction::GetSystemTransaction(db);
    system_catalog.CreateFunction(data, function);
}
```

### 7.2 冲突处理策略

```cpp
enum class OnCreateConflict : uint8_t {
    ERROR_ON_CONFLICT,      // 冲突时抛错
    IGNORE_ON_CONFLICT,     // 冲突时忽略
    REPLACE_ON_CONFLICT,    // 冲突时替换
    ALTER_ON_CONFLICT       // 冲突时合并（添加重载）
};

// 扩展默认使用 ALTER_ON_CONFLICT
// 允许多个扩展为同一函数添加不同类型的重载
```

## 8. 总结

### 8.1 架构特点

1. **模块化设计**：核心引擎与扩展解耦，按需加载
2. **多 ABI 支持**：C++ ABI 高性能，C ABI 高兼容性
3. **安全机制**：签名验证、版本检查、信任级别
4. **统一接口**：ExtensionLoader 提供一致的注册 API

### 8.2 关键组件关系

```
┌─────────────────────────────────────────────────────────────────────┐
│                        扩展系统组件关系                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐         ┌──────────────────┐                     │
│  │  Extension   │────────▶│ ExtensionLoader  │                     │
│  │  (抽象基类)   │  Load() │  (注册接口)       │                     │
│  └──────────────┘         └────────┬─────────┘                     │
│                                    │                                │
│                                    ▼                                │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                         Catalog                               │  │
│  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐         │  │
│  │  │Functions│  │  Types  │  │  Casts  │  │Collation│         │  │
│  │  └─────────┘  └─────────┘  └─────────┘  └─────────┘         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                    ▲                                │
│                                    │                                │
│  ┌──────────────┐         ┌───────┴────────┐                       │
│  │ExtensionHelper│────────▶│ExtensionManager│                       │
│  │ (安装/加载)   │         │  (状态跟踪)     │                       │
│  └──────────────┘         └────────────────┘                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

下一章将深入分析 ExtensionLoader 的注册接口，详解如何注册函数、类型和转换规则。
