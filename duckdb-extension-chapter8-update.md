# 第八章：扩展更新机制

## 概述

DuckDB 提供了扩展更新机制，允许用户更新已安装的扩展到最新版本。本章将分析更新接口、状态标记、ETag 优化和版本比较的实现。

## UpdateExtension 接口

### 单扩展更新

```cpp
ExtensionUpdateResult ExtensionHelper::UpdateExtension(ClientContext &context,
                                                        const string &extension_name) {
    auto &fs = FileSystem::GetFileSystem(context);
    DatabaseInstance &db = DatabaseInstance::GetDatabase(context);
    auto ext_directory = ExtensionHelper::ExtensionDirectory(db, fs);

    // 构建扩展文件路径
    auto full_extension_path = fs.JoinPath(ext_directory,
        extension_name + ".duckdb_extension");

    // 执行更新
    auto update_result = UpdateExtensionInternal(context, db, fs,
        full_extension_path, extension_name);

    // 处理错误情况
    if (update_result.tag == ExtensionUpdateResultTag::NOT_INSTALLED) {
        throw InvalidInputException(
            "Failed to update '%s', the extension is not installed!",
            extension_name);
    } else if (update_result.tag == ExtensionUpdateResultTag::UNKNOWN) {
        throw InternalException(
            "Failed to update '%s', an unknown error occurred",
            extension_name);
    }

    return update_result;
}
```

### 批量更新

```cpp
vector<ExtensionUpdateResult> ExtensionHelper::UpdateExtensions(ClientContext &context) {
    auto &fs = FileSystem::GetFileSystem(context);
    vector<ExtensionUpdateResult> result;
    DatabaseInstance &db = DatabaseInstance::GetDatabase(context);

#ifndef WASM_LOADABLE_EXTENSIONS
    case_insensitive_set_t seen_extensions;

    // 扫描扩展目录
    auto ext_directory = ExtensionHelper::ExtensionDirectory(db, fs);
    fs.ListFiles(ext_directory, [&](const string &path, bool is_directory) {
        // 只处理扩展文件
        if (!StringUtil::EndsWith(path, ".duckdb_extension")) {
            return;
        }

        // 提取扩展名
        auto extension_file_name = StringUtil::GetFileName(path);
        auto extension_name = StringUtil::Split(extension_file_name, ".")[0];

        seen_extensions.insert(extension_name);

        // 更新扩展
        result.push_back(UpdateExtensionInternal(context, db, fs,
            fs.JoinPath(ext_directory, path), extension_name));
    });
#endif

    return result;
}
```

## ExtensionUpdateResult 结构

### 更新结果定义

```cpp
struct ExtensionUpdateResult {
    //! 扩展名称
    string extension_name;
    //! 更新结果标记
    ExtensionUpdateResultTag tag;
    //! 之前的版本
    string prev_version;
    //! 安装后的版本
    string installed_version;
    //! 来源仓库
    string repository;
};
```

### 更新状态枚举

```cpp
enum class ExtensionUpdateResultTag {
    //! 未知状态
    UNKNOWN,
    //! 扩展未安装
    NOT_INSTALLED,
    //! 缺少安装信息文件
    MISSING_INSTALL_INFO,
    //! 不是从仓库安装的
    NOT_A_REPOSITORY,
    //! 无可用更新
    NO_UPDATE_AVAILABLE,
    //! 已更新到新版本
    UPDATED,
    //! 重新下载（版本未知时）
    REDOWNLOADED
};
```

## 内部更新流程

### UpdateExtensionInternal 实现

```cpp
static ExtensionUpdateResult UpdateExtensionInternal(
    ClientContext &context,
    DatabaseInstance &db,
    FileSystem &fs,
    const string &full_extension_path,
    const string &extension_name) {

    ExtensionUpdateResult result;
    result.extension_name = extension_name;

    // 1. 检查扩展是否存在
    if (!fs.FileExists(full_extension_path)) {
        result.tag = ExtensionUpdateResultTag::NOT_INSTALLED;
        return result;
    }

    // 2. 检查 .info 文件
    const string info_file_path = full_extension_path + ".info";
    if (!fs.FileExists(info_file_path)) {
        result.tag = ExtensionUpdateResultTag::MISSING_INSTALL_INFO;
        return result;
    }

    // 3. 解析当前版本
    auto ext_binary_handle = fs.OpenFile(full_extension_path,
        FileOpenFlags::FILE_FLAGS_READ);
    auto parsed_metadata = ExtensionHelper::ParseExtensionMetaData(*ext_binary_handle);

    if (!parsed_metadata.AppearsValid() &&
        !DBConfig::GetSetting<AllowExtensionsMetadataMismatchSetting>(context)) {
        throw IOException(
            "Failed to update '%s', the metadata appears invalid! "
            "Try 'FORCE INSTALL %s'",
            extension_name, extension_name);
    }

    result.prev_version = parsed_metadata.AppearsValid()
        ? parsed_metadata.extension_version : "";

    // 4. 读取安装信息
    auto extension_install_info = ExtensionInstallInfo::TryReadInfoFile(
        fs, info_file_path, extension_name);

    // 检查安装模式
    if (extension_install_info->mode == ExtensionInstallMode::UNKNOWN) {
        result.tag = ExtensionUpdateResultTag::MISSING_INSTALL_INFO;
        return result;
    }

    if (extension_install_info->mode != ExtensionInstallMode::REPOSITORY) {
        result.tag = ExtensionUpdateResultTag::NOT_A_REPOSITORY;
        result.installed_version = result.prev_version;
        return result;
    }

    // 5. 获取仓库信息
    auto repository_from_info = ExtensionRepository::GetRepositoryByUrl(
        extension_install_info->repository_url);
    result.repository = repository_from_info.ToReadableString();

    // 6. 强制安装（使用 ETag 优化）
    ExtensionInstallOptions options;
    options.repository = repository_from_info;
    options.force_install = true;
    options.use_etags = true;

    unique_ptr<ExtensionInstallInfo> install_result;
    try {
        install_result = ExtensionHelper::InstallExtension(context,
            extension_name, options);
    } catch (std::exception &e) {
        ErrorData error(e);
        error.Throw("Extension updating failed for '" + extension_name + "': ");
    }

    result.installed_version = install_result->version;

    // 7. 确定更新结果
    if (result.installed_version.empty()) {
        result.tag = ExtensionUpdateResultTag::REDOWNLOADED;
    } else if (result.installed_version != result.prev_version) {
        result.tag = ExtensionUpdateResultTag::UPDATED;
    } else {
        result.tag = ExtensionUpdateResultTag::NO_UPDATE_AVAILABLE;
    }

    return result;
}
```

## ETag 优化

### HTTP ETag 机制

ETag（Entity Tag）是 HTTP 协议的一部分，用于缓存验证：

```
                    首次下载
                        │
                        ▼
                 GET /extension.so
                        │
                        ▼
                HTTP 200 OK
                ETag: "abc123"
                        │
                        ▼
                保存 ETag 到 .info 文件
                        │
                        │
                    更新请求
                        │
                        ▼
            GET /extension.so
            If-None-Match: "abc123"
                        │
           ┌────────────┴────────────┐
           ▼                         ▼
    304 Not Modified              200 OK
    (无需下载)                   (新版本)
           │                         │
           ▼                         ▼
    NO_UPDATE_AVAILABLE       UPDATED/REDOWNLOADED
```

### ETag 在安装选项中的使用

```cpp
struct ExtensionInstallOptions {
    //! 是否强制安装
    bool force_install = false;
    //! 是否使用 ETag 优化
    bool use_etags = false;
    //! 目标仓库
    ExtensionRepository repository;
};
```

### 安装时的 ETag 处理

```cpp
unique_ptr<ExtensionInstallInfo> ExtensionHelper::InstallExtensionInternal(
    ..., ExtensionInstallOptions &options, ...) {

    // 读取已有的 ETag
    string current_etag;
    if (options.use_etags && fs.FileExists(info_file_path)) {
        auto existing_info = ExtensionInstallInfo::TryReadInfoFile(
            fs, info_file_path, extension_name);
        current_etag = existing_info->etag;
    }

    // 发起带 If-None-Match 的请求
    HTTPRequest request;
    request.url = extension_url;
    if (!current_etag.empty()) {
        request.headers["If-None-Match"] = current_etag;
    }

    auto response = http_client.Execute(request);

    if (response.status_code == 304) {
        // 未修改，无需下载
        return existing_info;
    }

    if (response.status_code == 200) {
        // 新版本，保存文件和新 ETag
        fs.WriteFile(extension_path, response.body);

        auto install_info = make_uniq<ExtensionInstallInfo>();
        install_info->etag = response.GetHeader("ETag");
        install_info->version = parsed_version;
        install_info->WriteInfoFile(fs, info_file_path);

        return install_info;
    }

    throw IOException("Failed to download extension: HTTP %d",
        response.status_code);
}
```

## 版本比较

### 语义化版本解析

```cpp
bool VersioningUtils::ParseSemver(const string &version,
                                   idx_t &major, idx_t &minor, idx_t &patch) {
    // 移除 'v' 前缀
    string ver = version;
    if (StringUtil::StartsWith(ver, "v")) {
        ver = ver.substr(1);
    }

    // 分割版本号
    auto parts = StringUtil::Split(ver, '.');
    if (parts.size() < 2) {
        return false;
    }

    try {
        major = std::stoull(parts[0]);
        minor = std::stoull(parts[1]);
        patch = parts.size() > 2 ? std::stoull(parts[2]) : 0;
        return true;
    } catch (...) {
        return false;
    }
}
```

### 版本比较逻辑

```cpp
int VersioningUtils::CompareVersions(const string &v1, const string &v2) {
    idx_t major1, minor1, patch1;
    idx_t major2, minor2, patch2;

    if (!ParseSemver(v1, major1, minor1, patch1) ||
        !ParseSemver(v2, major2, minor2, patch2)) {
        // 无法解析，进行字符串比较
        return v1.compare(v2);
    }

    // 主版本比较
    if (major1 != major2) {
        return major1 < major2 ? -1 : 1;
    }

    // 次版本比较
    if (minor1 != minor2) {
        return minor1 < minor2 ? -1 : 1;
    }

    // 补丁版本比较
    if (patch1 != patch2) {
        return patch1 < patch2 ? -1 : 1;
    }

    return 0;  // 版本相同
}
```

## SQL 接口

### UPDATE EXTENSIONS 语句

```sql
-- 更新单个扩展
UPDATE EXTENSIONS httpfs;

-- 更新所有扩展
UPDATE EXTENSIONS;
```

### 查看更新结果

```sql
-- 更新并返回结果
FROM update_extensions();

-- 返回结果示例：
-- ┌───────────────┬────────────────────┬──────────────┬───────────────────┬────────────────────┐
-- │ extension_name│        tag         │ prev_version │ installed_version │     repository     │
-- ├───────────────┼────────────────────┼──────────────┼───────────────────┼────────────────────┤
-- │ httpfs        │ UPDATED            │ v1.1.0       │ v1.2.0            │ core               │
-- │ json          │ NO_UPDATE_AVAILABLE│ v1.2.0       │ v1.2.0            │ core               │
-- │ parquet       │ UPDATED            │ v1.1.0       │ v1.2.0            │ core               │
-- └───────────────┴────────────────────┴──────────────┴───────────────────┴────────────────────┘
```

## 安装信息文件

### .info 文件格式

每个扩展旁边都有一个 `.duckdb_extension.info` 文件：

```
扩展目录结构：
extensions/
└── v1.2.0/
    └── linux_amd64/
        ├── httpfs.duckdb_extension      # 扩展二进制
        ├── httpfs.duckdb_extension.info  # 安装信息
        ├── json.duckdb_extension
        ├── json.duckdb_extension.info
        └── ...
```

### ExtensionInstallInfo 结构

```cpp
struct ExtensionInstallInfo {
    //! 安装模式
    ExtensionInstallMode mode = ExtensionInstallMode::UNKNOWN;
    //! 扩展版本
    string version;
    //! 仓库 URL
    string repository_url;
    //! HTTP ETag
    string etag;
    //! 完整路径（直接加载时）
    string full_path;

    //! 写入信息文件
    void WriteInfoFile(FileSystem &fs, const string &path);

    //! 读取信息文件
    static unique_ptr<ExtensionInstallInfo> TryReadInfoFile(
        FileSystem &fs, const string &path, const string &extension_name);
};
```

### 安装模式

```cpp
enum class ExtensionInstallMode : uint8_t {
    //! 未知模式
    UNKNOWN = 0,
    //! 从仓库安装
    REPOSITORY = 1,
    //! 从自定义路径安装
    CUSTOM_PATH = 2,
    //! 未安装（直接加载）
    NOT_INSTALLED = 3,
    //! 静态链接
    STATICALLY_LINKED = 4
};
```

## 更新流程图

```
                    UPDATE EXTENSIONS
                          │
                          ▼
                 ExtensionHelper::UpdateExtensions()
                          │
                          ▼
                 扫描扩展目录
                          │
                          ▼
            ┌─────────────┴─────────────┐
            ▼                           ▼
    找到 *.duckdb_extension         继续扫描...
            │
            ▼
    UpdateExtensionInternal()
            │
            ▼
    检查 .info 文件存在？
            │
    ┌───────┴───────┐
    ▼               ▼
  不存在          存在
    │               │
    ▼               ▼
  MISSING_    读取安装信息
  INSTALL_INFO
                    │
                    ▼
            mode == REPOSITORY?
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
        否                    是
         │                     │
         ▼                     ▼
  NOT_A_REPOSITORY    InstallExtension()
                      (force=true, use_etags=true)
                               │
                               ▼
                    HTTP GET + If-None-Match
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
         304 Not Modified   200 OK         其他错误
              │                │                │
              ▼                ▼                ▼
    NO_UPDATE_AVAILABLE  下载并保存      抛出异常
                               │
                               ▼
                      比较版本
                               │
                  ┌────────────┴────────────┐
                  ▼                         ▼
          版本相同                    版本不同
                  │                         │
                  ▼                         ▼
      NO_UPDATE_AVAILABLE             UPDATED
```

## 错误处理

### 常见错误情况

| 错误类型 | 原因 | 处理方式 |
|---------|------|---------|
| NOT_INSTALLED | 扩展未安装 | 提示先安装 |
| MISSING_INSTALL_INFO | 缺少 .info 文件 | 提示 FORCE INSTALL |
| NOT_A_REPOSITORY | 非仓库安装 | 返回当前版本 |
| 网络错误 | 无法连接仓库 | 抛出异常 |
| 元数据无效 | 扩展文件损坏 | 提示 FORCE INSTALL |

### 恢复策略

```sql
-- 当更新失败时，使用 FORCE INSTALL 重新安装
FORCE INSTALL httpfs;

-- 从指定仓库重新安装
INSTALL httpfs FROM 'http://extensions.duckdb.org';
```

## 小结

本章详细分析了 DuckDB 扩展的更新机制：

1. **更新接口**：支持单扩展和批量更新
2. **状态追踪**：`ExtensionUpdateResult` 提供详细的更新结果
3. **ETag 优化**：避免重复下载未更改的扩展
4. **版本比较**：语义化版本解析和比较
5. **信息文件**：持久化安装信息支持更新检测

这套机制确保了扩展更新的高效性和可靠性。
