# 第四章：扩展加载流程

## 概述

扩展加载是 DuckDB 扩展系统的核心流程，负责将已安装的扩展文件加载到数据库运行时。本章将深入分析动态库加载机制、加载状态管理、元数据验证以及自动加载机制的实现细节。

## 动态库加载机制

### 跨平台动态库封装

DuckDB 在 `src/include/duckdb/common/dl.hpp` 中提供了跨平台的动态库加载封装：

```cpp
// POSIX 系统使用 dlfcn.h
#ifndef _WIN32
#include <dlfcn.h>
#else
// Windows 定义兼容的宏
#define RTLD_NOW   0
#define RTLD_LOCAL 0
#endif

namespace duckdb {

#ifdef _WIN32
// Windows 平台实现
inline void *dlopen(const char *file, int mode) {
    if (!file) {
        return (void *)GetModuleHandle(nullptr);
    }
    auto fpath = WindowsUtil::UTF8ToUnicode(file);
    return (void *)LoadLibraryW(fpath.c_str());
}

inline void *dlsym(void *handle, const char *name) {
    D_ASSERT(handle);
    return (void *)GetProcAddress((HINSTANCE)handle, name);
}

inline std::string GetDLError(void) {
    return LocalFileSystem::GetLastErrorAsString();
}

inline void dlclose(void *handle) {
    D_ASSERT(handle);
    FreeLibrary((HINSTANCE)handle);
}
#else
// POSIX 平台使用系统 dlfcn.h
inline std::string GetDLError(void) {
    return dlerror();
}
#endif

} // namespace duckdb
```

这个封装层确保了相同的加载代码可以在 Linux、macOS 和 Windows 上运行。

### 动态库加载调用

扩展的实际加载发生在 `extension_load.cpp` 中：

```cpp
auto lib_hdl = dlopen(dopen_from.c_str(), RTLD_NOW | RTLD_LOCAL);
if (!lib_hdl) {
    throw IOException("Extension \"%s\" could not be loaded: %s",
                      filename, GetDLError());
}
```

加载标志说明：
- **RTLD_NOW**：立即解析所有未定义符号，而不是延迟解析
- **RTLD_LOCAL**：扩展中的符号不对其他动态加载的库可见

### 符号解析函数

加载动态库后，需要从中获取入口函数：

```cpp
template <class T>
static T LoadFunctionFromDLL(void *dll, const string &function_name,
                             const string &filename) {
    auto function = dlsym(dll, function_name.c_str());
    if (!function) {
        throw IOException("File \"%s\" did not contain function \"%s\": %s",
                          filename, function_name, GetDLError());
    }
    return (T)function;
}

template <class T>
static T TryLoadFunctionFromDLL(void *dll, const string &function_name,
                                const string &filename) {
    auto function = dlsym(dll, function_name.c_str());
    if (!function) {
        return nullptr;  // 不抛出异常，返回空
    }
    return (T)function;
}
```

`LoadFunctionFromDLL` 在找不到符号时抛出异常，而 `TryLoadFunctionFromDLL` 返回 nullptr，允许调用者处理缺失情况。

## 入口函数类型定义

DuckDB 支持两种 ABI 类型的扩展入口：

```cpp
// C++ 入口函数类型
typedef void (*ext_init_fun_t)(ExtensionLoader &);

// C API 入口函数类型
typedef bool (*ext_init_c_api_fun_t)(duckdb_extension_info info,
                                      duckdb_extension_access *access);
```

## ExtensionManager 加载管理

### 加载状态数据结构

`ExtensionManager` 使用以下数据结构管理扩展状态：

```cpp
class ExtensionInfo {
public:
    ExtensionInfo() : is_loaded(false) {}

    //! 扩展是否已加载
    bool is_loaded;
    //! 扩展安装信息
    unique_ptr<ExtensionInstallInfo> install_info;
    //! 加载锁（每个扩展独立）
    mutex lock;
};

class ExtensionActiveLoad {
public:
    ExtensionActiveLoad(DatabaseInstance &db, ExtensionInfo &info,
                        string extension_name);

    //! 完成加载
    void FinishLoad(ExtensionInstallInfo &install_info);
    //! 加载失败
    void LoadFail(const ErrorData &error);

private:
    DatabaseInstance &db;
    lock_guard<mutex> load_lock;  // RAII 锁
    ExtensionInfo &info;
    string extension_name;
};
```

### BeginLoad 加载开始

`BeginLoad` 方法处理并发加载的同步问题：

```cpp
unique_ptr<ExtensionActiveLoad> ExtensionManager::BeginLoad(const string &name) {
    auto extension_name = ExtensionHelper::GetExtensionName(name);

    unique_lock<mutex> extension_list_lock(lock);

    optional_ptr<ExtensionInfo> info;
    auto entry = loaded_extensions_info.find(extension_name);

    if (entry == loaded_extensions_info.end()) {
        // 创建新条目
        auto extension_info = make_uniq<ExtensionInfo>();
        info = extension_info.get();
        loaded_extensions_info.emplace(extension_name, std::move(extension_info));
    } else {
        // 已有条目
        if (entry->second->is_loaded) {
            // 已加载，直接返回
            return nullptr;
        }
        info = entry->second.get();
    }
    extension_list_lock.unlock();

    // 创建 ExtensionActiveLoad，同时获取扩展专属锁
    auto result = make_uniq<ExtensionActiveLoad>(db, *info, extension_name);

    // 双重检查：另一个线程可能已经完成加载
    if (info->is_loaded) {
        return nullptr;
    }

    // 触发加载开始回调
    auto &callbacks = DBConfig::GetConfig(db).extension_callbacks;
    for (auto &callback : callbacks) {
        callback->OnBeginExtensionLoad(db, extension_name);
    }

    return result;
}
```

这个设计实现了：
1. **双重检查锁定**：避免重复加载
2. **细粒度锁**：每个扩展有独立的锁，不同扩展可以并发加载
3. **回调支持**：允许注册加载事件监听器

### 加载流程图

```
                    BeginLoad()
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
    扩展未注册                   扩展已注册
           │                         │
           ▼                         ▼
    创建 ExtensionInfo          检查 is_loaded
           │                         │
           │              ┌──────────┴──────────┐
           │              ▼                     ▼
           │         已加载                  未加载
           │              │                     │
           │              ▼                     │
           │       返回 nullptr                 │
           │                                    │
           └──────────────┬─────────────────────┘
                          ▼
              创建 ExtensionActiveLoad
              （获取扩展专属锁）
                          │
                          ▼
                 再次检查 is_loaded
                          │
           ┌──────────────┴──────────────┐
           ▼                             ▼
       已加载                         未加载
           │                             │
           ▼                             ▼
    返回 nullptr               触发 OnBeginExtensionLoad
                                         │
                                         ▼
                              返回 ExtensionActiveLoad
```

## InitialLoad 初始加载

### 路径解析逻辑

`InitialLoad` 首先需要确定扩展文件的位置：

```cpp
ExtensionInitResult ExtensionHelper::InitialLoad(DatabaseInstance &db,
                                                  FileSystem &fs,
                                                  const string &extension) {
    string error;
    ExtensionInitResult result;

    if (!TryInitialLoad(db, fs, extension, result, error)) {
        auto &config = DBConfig::GetConfig(db);
        if (!config.options.autoinstall_known_extensions ||
            !ExtensionHelper::AllowAutoInstall(extension)) {
            throw IOException(error);
        }
        // 尝试自动安装
        ExtensionInstallOptions options;
        ExtensionHelper::InstallExtension(db, fs, extension, options);
        // 重试加载
        if (!TryInitialLoad(db, fs, extension, result, error)) {
            throw IOException(error);
        }
    }
    return result;
}
```

### TryInitialLoad 实现

```cpp
bool ExtensionHelper::TryInitialLoad(DatabaseInstance &db, FileSystem &fs,
                                      const string &extension,
                                      ExtensionInitResult &result,
                                      string &error) {
    // 1. 权限检查
    if (!db.config.options.enable_external_access) {
        throw PermissionException(
            "Loading external extensions is disabled through configuration");
    }

    auto filename = fs.ConvertSeparators(extension);
    bool direct_load;

    // 2. 路径判断
    if (!ExtensionHelper::IsFullPath(extension)) {
        direct_load = false;
        string extension_name = ApplyExtensionAlias(extension);

        // 构建搜索目录列表
        vector<string> search_directories;
        if (!db.config.options.extension_directory.empty()) {
            search_directories.push_back(db.config.options.extension_directory);
        }
        if (!db.config.options.extension_directories.empty()) {
            for (const auto &dir : db.config.options.extension_directories) {
                search_directories.push_back(dir);
            }
        }
        if (search_directories.empty()) {
            for (const auto &path : ExtensionHelper::DefaultExtensionFolders(fs)) {
                search_directories.push_back(path);
            }
        }

        // 在目录中查找扩展
        bool found = false;
        for (const auto &directory : search_directories) {
            filename = ComputeLocalExtensionPath(directory, extension_name);
            if (fs.FileExists(filename)) {
                found = true;
                break;
            }
        }

        if (!found) {
            filename = ComputeLocalExtensionPath(search_directories[0],
                                                  extension_name);
        }
    } else {
        direct_load = true;
        filename = fs.ExpandPath(filename);
    }

    // 3. 文件存在性检查
    if (!fs.FileExists(filename)) {
        string message;
        bool exact_match = ExtensionHelper::CreateSuggestions(extension, message);
        if (exact_match) {
            message += "\nInstall it first using \"INSTALL " + extension + "\".";
        }
        error = StringUtil::Format("Extension \"%s\" not found.\n%s",
                                   filename, message);
        return false;
    }

    // 4. 打开文件并解析元数据
    auto handle = fs.OpenFile(filename, FileFlags::FILE_FLAGS_READ);
    auto parsed_metadata = ParseExtensionMetaData(*handle);

    // 5. 元数据验证
    auto metadata_mismatch_error = parsed_metadata.GetInvalidMetadataError();

    // 6. 签名验证
    if (!db.config.options.allow_unsigned_extensions) {
        bool signature_valid;
        if (parsed_metadata.AppearsValid()) {
            signature_valid = CheckExtensionSignature(
                *handle, parsed_metadata,
                db.config.options.allow_community_extensions);
        } else {
            signature_valid = false;
        }

        if (!metadata_mismatch_error.empty()) {
            throw InvalidInputException(metadata_mismatch_error);
        }

        if (!signature_valid) {
            throw IOException(db.config.error_manager->FormatException(
                ErrorType::UNSIGNED_EXTENSION, filename));
        }
    }

    // 7. 加载动态库
    auto lib_hdl = dlopen(filename.c_str(), RTLD_NOW | RTLD_LOCAL);
    if (!lib_hdl) {
        throw IOException("Extension \"%s\" could not be loaded: %s",
                          filename, GetDLError());
    }

    // 8. 填充加载结果
    auto lowercase_extension_name = StringUtil::Lower(filebase);
    result.filebase = lowercase_extension_name;
    result.filename = filename;
    result.lib_hdl = lib_hdl;
    result.abi_type = parsed_metadata.abi_type;

    // 9. 读取安装信息
    if (!direct_load) {
        auto info_file_name = filename + ".info";
        result.install_info = ExtensionInstallInfo::TryReadInfoFile(
            fs, info_file_name, lowercase_extension_name);
    } else {
        result.install_info = make_uniq<ExtensionInstallInfo>();
        result.install_info->mode = ExtensionInstallMode::NOT_INSTALLED;
        result.install_info->full_path = filename;
        result.install_info->version = parsed_metadata.extension_version;
    }

    return true;
}
```

### ExtensionInitResult 结构

```cpp
struct ExtensionInitResult {
    //! 扩展文件基名（小写）
    string filebase;
    //! 完整文件路径
    string filename;
    //! 动态库句柄
    void *lib_hdl;
    //! ABI 类型
    ExtensionABIType abi_type;
    //! 安装信息
    unique_ptr<ExtensionInstallInfo> install_info;
};
```

## LoadExternalExtension 完整加载

### 加载入口

```cpp
void ExtensionHelper::LoadExternalExtension(DatabaseInstance &db,
                                             FileSystem &fs,
                                             const string &extension) {
    auto &manager = ExtensionManager::Get(db);
    auto info = manager.BeginLoad(extension);
    if (!info) {
        return;  // 已加载
    }
    try {
        LoadExternalExtensionInternal(db, fs, extension, *info);
    } catch (std::exception &ex) {
        ErrorData error(ex);
        info->LoadFail(error);
        throw;
    }
}
```

### 分 ABI 类型加载

```cpp
void ExtensionHelper::LoadExternalExtensionInternal(DatabaseInstance &db,
                                                     FileSystem &fs,
                                                     const string &extension,
                                                     ExtensionActiveLoad &info) {
    auto extension_init_result = InitialLoad(db, fs, extension);

    // C++ ABI 扩展
    if (extension_init_result.abi_type == ExtensionABIType::CPP) {
        // 构造入口函数名：<extension_name>_duckdb_cpp_init
        auto init_fun_name = extension_init_result.filebase + "_duckdb_cpp_init";

        ext_init_fun_t init_fun = TryLoadFunctionFromDLL<ext_init_fun_t>(
            extension_init_result.lib_hdl, init_fun_name,
            extension_init_result.filename);

        if (!init_fun) {
            throw IOException(
                "Extension '%s' did not contain the expected entrypoint '%s'",
                extension, init_fun_name);
        }

        try {
            ExtensionLoader loader(info);
            (*init_fun)(loader);        // 调用扩展初始化
            loader.FinalizeLoad();       // 完成加载
        } catch (std::exception &e) {
            ErrorData error(e);
            throw InvalidInputException(
                "Initialization function \"%s\" threw an exception: \"%s\"",
                init_fun_name, error.RawMessage());
        }

        info.FinishLoad(*extension_init_result.install_info);
        return;
    }

    // C ABI 扩展
    if (extension_init_result.abi_type == ExtensionABIType::C_STRUCT ||
        extension_init_result.abi_type == ExtensionABIType::C_STRUCT_UNSTABLE) {
        // 构造入口函数名：<extension_name>_init_c_api
        auto init_fun_name = extension_init_result.filebase + "_init_c_api";

        ext_init_c_api_fun_t init_fun_capi = TryLoadFunctionFromDLL<
            ext_init_c_api_fun_t>(
                extension_init_result.lib_hdl, init_fun_name,
                extension_init_result.filename);

        if (!init_fun_capi) {
            throw IOException("File \"%s\" did not contain function \"%s\"",
                              extension_init_result.filename, init_fun_name);
        }

        // 创建加载状态
        DuckDBExtensionLoadState load_state(db, extension_init_result);

        // 创建访问结构
        auto access = ExtensionAccess::CreateAccessStruct();

        // 调用 C API 初始化函数
        auto result = (*init_fun_capi)(load_state.ToCStruct(), &access);

        // 检查错误
        if (load_state.has_error) {
            load_state.error_data.Throw(
                "An error was thrown during initialization of '" +
                extension + "': ");
        }

        if (result == false) {
            throw FatalException(
                "Extension '%s' failed to initialize but did not return an error",
                extension);
        }

        info.FinishLoad(*extension_init_result.install_info);
        return;
    }

    throw IOException("Unknown ABI type for extension '%s'", extension);
}
```

## C API 加载支持

### DuckDBExtensionLoadState 结构

```cpp
struct DuckDBExtensionLoadState {
    explicit DuckDBExtensionLoadState(DatabaseInstance &db_p,
                                       ExtensionInitResult &init_result_p)
        : db(db_p), init_result(init_result_p), database_data(nullptr) {}

    //! 数据库实例引用
    DatabaseInstance &db;

    //! 初始化结果
    ExtensionInitResult &init_result;

    //! 传递给扩展的数据库包装器
    unique_ptr<DatabaseWrapper> database_data;

    //! API 函数指针结构
    duckdb_ext_api_v1 api_struct;

    //! 错误标志
    bool has_error = false;
    ErrorData error_data;

    //! 转换为 C API 指针
    duckdb_extension_info ToCStruct() {
        return reinterpret_cast<duckdb_extension_info>(this);
    }

    //! 从 C API 指针获取引用
    static DuckDBExtensionLoadState &Get(duckdb_extension_info info) {
        D_ASSERT(info);
        return *reinterpret_cast<DuckDBExtensionLoadState *>(info);
    }
};
```

### ExtensionAccess 回调函数

```cpp
struct ExtensionAccess {
    //! 创建访问结构
    static duckdb_extension_access CreateAccessStruct() {
        return {SetError, GetDatabase, GetAPI};
    }

    //! 扩展报告错误
    static void SetError(duckdb_extension_info info, const char *error) {
        auto &load_state = DuckDBExtensionLoadState::Get(info);
        load_state.has_error = true;
        load_state.error_data = error
            ? ErrorData(error)
            : ErrorData(ExceptionType::UNKNOWN_TYPE,
                        "Extension error without message");
    }

    //! 获取数据库实例
    static duckdb_database GetDatabase(duckdb_extension_info info) {
        auto &load_state = DuckDBExtensionLoadState::Get(info);
        try {
            load_state.database_data = make_uniq<DatabaseWrapper>();
            load_state.database_data->database =
                make_shared_ptr<DuckDB>(load_state.db);
            return reinterpret_cast<duckdb_database>(
                load_state.database_data.get());
        } catch (std::exception &ex) {
            load_state.has_error = true;
            load_state.error_data = ErrorData(ex);
            return nullptr;
        }
    }

    //! 获取 API 结构
    static const void *GetAPI(duckdb_extension_info info, const char *version) {
        string version_string = version;
        auto &load_state = DuckDBExtensionLoadState::Get(info);

        if (load_state.init_result.abi_type == ExtensionABIType::C_STRUCT) {
            // 验证 API 版本兼容性
            idx_t major, minor, patch;
            auto parsed = VersioningUtils::ParseSemver(version_string,
                                                        major, minor, patch);

            if (!parsed || !VersioningUtils::IsSupportedCAPIVersion(
                    major, minor, patch)) {
                load_state.has_error = true;
                load_state.error_data = ErrorData(
                    ExceptionType::UNKNOWN_TYPE,
                    "Unsupported C API version: " + version_string);
                return nullptr;
            }
        }
        // C_STRUCT_UNSTABLE 不检查版本

        load_state.api_struct = load_state.db.GetExtensionAPIV1();
        return &load_state.api_struct;
    }
};
```

## 元数据验证

### 解析扩展元数据

```cpp
ParsedExtensionMetaData ExtensionHelper::ParseExtensionMetaData(
    const char *metadata) noexcept {
    ParsedExtensionMetaData result;

    // 读取 8 个 32 字节字段
    vector<string> metadata_field;
    for (idx_t i = 0; i < 8; i++) {
        string field = string(metadata + i * 32, 32);
        metadata_field.emplace_back(field);
    }

    // 反转顺序（footer 是反向存储的）
    std::reverse(metadata_field.begin(), metadata_field.end());

    // 解析魔数
    result.magic_value = FilterZeroAtEnd(metadata_field[0]);
    if (!result.AppearsValid()) {
        return result;
    }

    // 解析平台
    result.platform = FilterZeroAtEnd(metadata_field[1]);

    // 解析扩展版本
    result.extension_version = FilterZeroAtEnd(metadata_field[3]);

    // 解析 ABI 类型
    auto extension_abi_metadata = FilterZeroAtEnd(metadata_field[4]);

    if (extension_abi_metadata == "C_STRUCT") {
        result.abi_type = ExtensionABIType::C_STRUCT;
        result.duckdb_capi_version = FilterZeroAtEnd(metadata_field[2]);
    } else if (extension_abi_metadata == "C_STRUCT_UNSTABLE") {
        result.abi_type = ExtensionABIType::C_STRUCT_UNSTABLE;
        result.duckdb_version = FilterZeroAtEnd(metadata_field[2]);
    } else if (extension_abi_metadata == "CPP" || extension_abi_metadata.empty()) {
        result.abi_type = ExtensionABIType::CPP;
        result.duckdb_version = FilterZeroAtEnd(metadata_field[2]);
    } else {
        result.abi_type = ExtensionABIType::UNKNOWN;
    }

    // 提取签名
    result.signature = string(metadata,
        ParsedExtensionMetaData::FOOTER_SIZE -
        ParsedExtensionMetaData::SIGNATURE_SIZE);

    return result;
}
```

### 签名验证

```cpp
bool ExtensionHelper::CheckExtensionSignature(
    FileHandle &handle,
    ParsedExtensionMetaData &parsed_metadata,
    const bool allow_community_extensions) {

    auto signature_offset = handle.GetFileSize() -
        ParsedExtensionMetaData::SIGNATURE_SIZE;

    // 分块计算 SHA256
    const idx_t maxLenChunks = 1024ULL * 1024ULL;
    const idx_t numChunks = (signature_offset + maxLenChunks - 1) / maxLenChunks;
    vector<string> hash_chunks(numChunks);

    // 多线程并行计算哈希
#ifndef DUCKDB_NO_THREADS
    vector<std::thread> threads;
    for (idx_t i = 0; i < numChunks; i++) {
        threads.emplace_back(ComputeSHA256FileSegment, &handle,
                             splits[i], splits[i + 1], &hash_chunks[i]);
    }
    for (auto &thread : threads) {
        thread.join();
    }
#else
    for (idx_t i = 0; i < numChunks; i++) {
        ComputeSHA256FileSegment(&handle, splits[i], splits[i + 1],
                                  &hash_chunks[i]);
    }
#endif

    // 合并哈希并计算最终哈希
    string hash_concatenation;
    for (auto &hash_chunk : hash_chunks) {
        hash_concatenation += hash_chunk;
    }

    string two_level_hash;
    ComputeSHA256String(hash_concatenation, &two_level_hash);

    // 读取签名
    handle.Read((void *)parsed_metadata.signature.data(),
                parsed_metadata.signature.size(), signature_offset);

    // 验证签名
    for (auto &key : ExtensionHelper::GetPublicKeys(allow_community_extensions)) {
        if (duckdb_mbedtls::MbedTlsWrapper::IsValidSha256Signature(
                key, parsed_metadata.signature, two_level_hash)) {
            return true;
        }
    }

    return false;
}
```

## 自动加载机制

### AutoLoadExtension

```cpp
void ExtensionHelper::AutoLoadExtension(DatabaseInstance &db,
                                         const string &extension_name) {
    if (db.ExtensionIsLoaded(extension_name)) {
        return;  // 避免重复下载
    }

    auto &dbconfig = DBConfig::GetConfig(db);
    try {
        auto fs = FileSystem::CreateLocal();

#ifndef DUCKDB_WASM
        if (dbconfig.options.autoinstall_known_extensions) {
            // 获取自动安装仓库
            auto repository_url = GetAutoInstallExtensionsRepository(
                dbconfig.options);
            auto autoinstall_repo = ExtensionRepository::GetRepositoryByUrl(
                repository_url);

            ExtensionInstallOptions options;
            options.repository = autoinstall_repo;
            ExtensionHelper::InstallExtension(db, *fs, extension_name, options);
        }
#endif
        ExtensionHelper::LoadExternalExtension(db, *fs, extension_name);
        DUCKDB_LOG_INFO(db, "Loaded extension '%s'", extension_name);
    } catch (std::exception &e) {
        ErrorData error(e);
        throw AutoloadException(extension_name, error.RawMessage());
    }
}
```

### TryAutoLoadExtension

提供不抛出异常的版本：

```cpp
bool ExtensionHelper::TryAutoLoadExtension(DatabaseInstance &instance,
                                            const string &extension_name) noexcept {
    if (instance.ExtensionIsLoaded(extension_name)) {
        return true;
    }

    auto &dbconfig = DBConfig::GetConfig(instance);
    try {
        auto &fs = FileSystem::GetFileSystem(instance);
        if (dbconfig.options.autoinstall_known_extensions) {
            auto repository_url = GetAutoInstallExtensionsRepository(
                dbconfig.options);
            auto autoinstall_repo = ExtensionRepository::GetRepositoryByUrl(
                repository_url);
            ExtensionInstallOptions options;
            options.repository = autoinstall_repo;
            ExtensionHelper::InstallExtension(instance, fs, extension_name,
                                               options);
        }
        ExtensionHelper::LoadExternalExtension(instance, fs, extension_name);
        return true;
    } catch (...) {
        return false;
    }
}
```

### CanAutoloadExtension

判断扩展是否可以自动加载：

```cpp
bool ExtensionHelper::CanAutoloadExtension(const string &ext_name) {
#ifdef DUCKDB_DISABLE_EXTENSION_LOAD
    return false;
#endif

    if (ext_name.empty()) {
        return false;
    }

    // 检查是否在自动加载列表中
    for (const auto &ext : AUTOLOADABLE_EXTENSIONS) {
        if (ext_name == ext) {
            return true;
        }
    }
    return false;
}
```

### AllowAutoInstall

限制哪些扩展可以自动安装：

```cpp
static const char *const auto_install[] = {
    "motherduck", "postgres_scanner", "mysql_scanner",
    "sqlite_scanner", "delta", "iceberg", "uc_catalog",
    "ui", "ducklake", nullptr
};

bool ExtensionHelper::AllowAutoInstall(const string &extension) {
    auto extension_name = ApplyExtensionAlias(extension);
    for (idx_t i = 0; auto_install[i]; i++) {
        if (extension_name == auto_install[i]) {
            return true;
        }
    }
    return false;
}
```

## 加载回调系统

### 回调注册

```cpp
// 在 DBConfig 中注册回调
class ExtensionCallback {
public:
    virtual void OnBeginExtensionLoad(DatabaseInstance &db,
                                       const string &extension_name) = 0;
    virtual void OnExtensionLoaded(DatabaseInstance &db,
                                    const string &extension_name) = 0;
    virtual void OnExtensionLoadFail(DatabaseInstance &db,
                                      const string &extension_name,
                                      const ErrorData &error) = 0;
};
```

### 回调触发

在 `ExtensionActiveLoad` 中触发回调：

```cpp
void ExtensionActiveLoad::FinishLoad(ExtensionInstallInfo &install_info) {
    info.is_loaded = true;
    info.install_info = make_uniq<ExtensionInstallInfo>(install_info);

    auto &callbacks = DBConfig::GetConfig(db).extension_callbacks;
    for (auto &callback : callbacks) {
        callback->OnExtensionLoaded(db, extension_name);
    }
    DUCKDB_LOG_INFO(db, extension_name);
}

void ExtensionActiveLoad::LoadFail(const ErrorData &error) {
    auto &callbacks = DBConfig::GetConfig(db).extension_callbacks;
    for (auto &callback : callbacks) {
        callback->OnExtensionLoadFail(db, extension_name, error);
    }
    DUCKDB_LOG_INFO(db, "Failed to load extension '%s': %s",
                    extension_name, error.Message());
}
```

## 完整加载流程图

```
                          LOAD 'extension'
                                │
                                ▼
                    ExtensionManager::BeginLoad()
                                │
                      ┌─────────┴─────────┐
                      ▼                   ▼
                  已加载             创建 ActiveLoad
                      │                   │
                      ▼                   ▼
                   返回              InitialLoad()
                                         │
                                         ▼
                              ┌──────────┴──────────┐
                              ▼                     ▼
                         短名称                  完整路径
                              │                     │
                              ▼                     ▼
                      搜索扩展目录           展开路径
                              │                     │
                              └──────────┬──────────┘
                                         ▼
                               文件存在性检查
                                         │
                           ┌─────────────┴─────────────┐
                           ▼                           ▼
                      不存在                        存在
                           │                           │
                           ▼                           ▼
                   抛出 IOException            解析元数据
                                                      │
                                                      ▼
                                              签名验证
                                                      │
                                   ┌──────────────────┴──────────────────┐
                                   ▼                                     ▼
                              无效签名                               有效签名
                                   │                                     │
                                   ▼                                     ▼
                         抛出 IOException                          dlopen()
                                                                        │
                                                                        ▼
                                                          ┌─────────────┴─────────────┐
                                                          ▼                           ▼
                                                       C++ ABI                     C ABI
                                                          │                           │
                                                          ▼                           ▼
                                               查找 *_duckdb_cpp_init    查找 *_init_c_api
                                                          │                           │
                                                          ▼                           ▼
                                               ExtensionLoader               ExtensionAccess
                                                          │                           │
                                                          ▼                           ▼
                                               (*init_fun)(loader)      (*init_fun_capi)()
                                                          │                           │
                                                          └─────────────┬─────────────┘
                                                                        ▼
                                                              FinishLoad()
                                                                        │
                                                                        ▼
                                                              触发回调通知
                                                                        │
                                                                        ▼
                                                                    完成
```

## 小结

本章详细分析了 DuckDB 扩展的加载流程：

1. **跨平台封装**：通过统一的 dlopen/dlsym 接口支持 Windows 和 POSIX 系统
2. **并发安全**：ExtensionManager 使用双重检查锁定避免重复加载
3. **ABI 分发**：根据扩展的 ABI 类型选择正确的入口函数和初始化方式
4. **安全验证**：元数据解析和签名验证确保扩展的完整性和来源可信
5. **自动加载**：支持按需自动安装和加载扩展
6. **回调系统**：允许监听扩展加载事件

这套机制确保了扩展加载的安全性、可靠性和灵活性。
