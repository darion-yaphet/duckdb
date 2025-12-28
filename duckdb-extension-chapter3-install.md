# DuckDB 扩展系统深度解析（三）：扩展安装机制

## 引言

DuckDB 的扩展安装机制支持从官方仓库、社区仓库或自定义路径安装扩展。本章将深入分析安装流程、仓库系统、安装信息管理以及目录结构。

## 1. ExtensionHelper 安装接口

### 1.1 核心安装方法

```cpp
// src/include/duckdb/main/extension_helper.hpp

class ExtensionHelper {
public:
    //! 从 ClientContext 安装扩展
    static unique_ptr<ExtensionInstallInfo> InstallExtension(
        ClientContext &context,
        const string &extension,
        ExtensionInstallOptions &options);

    //! 从 DatabaseInstance 安装扩展
    static unique_ptr<ExtensionInstallInfo> InstallExtension(
        DatabaseInstance &db,
        FileSystem &fs,
        const string &extension,
        ExtensionInstallOptions &options);
};
```

### 1.2 安装选项

```cpp
struct ExtensionInstallOptions {
    //! 从指定仓库安装（覆盖默认仓库）
    optional_ptr<ExtensionRepository> repository;

    //! 安装特定版本
    string version;

    //! 强制覆盖已安装的扩展
    bool force_install = false;

    //! 使用 ETag 避免重复下载
    bool use_etags = false;

    //! 来源不匹配时抛出错误
    bool throw_on_origin_mismatch = false;
};
```

### 1.3 安装流程

```cpp
unique_ptr<ExtensionInstallInfo> ExtensionHelper::InstallExtension(
    ClientContext &context,
    const string &extension,
    ExtensionInstallOptions &options) {

#ifdef WASM_LOADABLE_EXTENSIONS
    // WASM 版本安装为空操作
    return nullptr;
#endif

    auto &db = DatabaseInstance::GetDatabase(context);
    auto &fs = FileSystem::GetFileSystem(context);

    // 获取安装目录
    string local_path = ExtensionDirectory(context);

    // 执行内部安装
    return InstallExtensionInternal(db, fs, local_path, extension, options, context);
}
```

## 2. 扩展仓库系统

### 2.1 ExtensionRepository 结构

```cpp
// src/include/duckdb/main/extension_install_info.hpp

struct ExtensionRepository {
    //! 官方仓库 URL
    static constexpr const char *CORE_REPOSITORY_URL =
        "http://extensions.duckdb.org";

    //! 每日构建仓库
    static constexpr const char *CORE_NIGHTLY_REPOSITORY_URL =
        "http://nightly-extensions.duckdb.org";

    //! 社区仓库
    static constexpr const char *COMMUNITY_REPOSITORY_URL =
        "http://community-extensions.duckdb.org";

    //! 调试仓库（本地构建）
    static constexpr const char *BUILD_DEBUG_REPOSITORY_PATH =
        "./build/debug/repository";
    static constexpr const char *BUILD_RELEASE_REPOSITORY_PATH =
        "./build/release/repository";

    //! 默认仓库
    static constexpr const char *DEFAULT_REPOSITORY_URL =
        CORE_REPOSITORY_URL;

    //! 仓库名称
    string name;
    //! 仓库路径/URL
    string path;

    ExtensionRepository();
    ExtensionRepository(const string &name, const string &url);

    //! 获取默认仓库
    static ExtensionRepository GetDefaultRepository(optional_ptr<DBConfig> config);
    static ExtensionRepository GetDefaultRepository(ClientContext &context);

    //! 获取核心仓库
    static ExtensionRepository GetCoreRepository();

    //! 从 URL 获取仓库
    static ExtensionRepository GetRepositoryByUrl(const string &url);

    //! 获取可读的仓库名称
    string ToReadableString();
};
```

### 2.2 仓库 URL 模板

```cpp
string ExtensionHelper::ExtensionUrlTemplate(
    optional_ptr<const DatabaseInstance> db,
    const ExtensionRepository &repository,
    const string &version) {

    string versioned_path;
    if (!version.empty()) {
        // 带版本的路径
        versioned_path = "/${NAME}/" + version +
                         "/${REVISION}/${PLATFORM}/${NAME}.duckdb_extension";
    } else {
        // 不带版本的路径
        versioned_path = "/${REVISION}/${PLATFORM}/${NAME}.duckdb_extension";
    }

#ifdef WASM_LOADABLE_EXTENSIONS
    versioned_path = versioned_path + ".wasm";
#else
    // 添加 .gz 压缩后缀
    versioned_path = versioned_path +
        CompressionExtensionFromType(FileCompressionType::GZIP);
#endif

    return repository.path + versioned_path;
}

string ExtensionHelper::ExtensionFinalizeUrlTemplate(
    const string &url_template,
    const string &extension_name) {

    auto url = StringUtil::Replace(url_template, "${REVISION}",
                                   GetVersionDirectoryName());
    url = StringUtil::Replace(url, "${PLATFORM}", DuckDB::Platform());
    url = StringUtil::Replace(url, "${NAME}", extension_name);
    return url;
}
```

### 2.3 URL 示例

```
模板: http://extensions.duckdb.org/${REVISION}/${PLATFORM}/${NAME}.duckdb_extension.gz

展开后:
http://extensions.duckdb.org/v1.2.0/linux_amd64/parquet.duckdb_extension.gz
http://extensions.duckdb.org/v1.2.0/osx_arm64/json.duckdb_extension.gz
http://extensions.duckdb.org/v1.2.0/windows_amd64/icu.duckdb_extension.gz
```

## 3. 安装信息管理

### 3.1 ExtensionInstallInfo 结构

```cpp
// src/include/duckdb/main/extension_install_info.hpp

enum class ExtensionInstallMode : uint8_t {
    UNKNOWN = 0,           // 安装信息缺失
    REPOSITORY = 1,        // 从仓库安装
    CUSTOM_PATH = 2,       // 从自定义路径安装
    STATICALLY_LINKED = 3, // 静态链接
    NOT_INSTALLED = 4      // 未安装（直接加载）
};

class ExtensionInstallInfo {
public:
    //! 安装模式
    ExtensionInstallMode mode = ExtensionInstallMode::UNKNOWN;

    //! 扩展来源路径（可选）
    string full_path;

    //! 仓库 URL（可选）
    string repository_url;

    //! 扩展版本（可选）
    string version;

    //! HTTP ETag（可选，用于缓存）
    string etag;

    //! 序列化安装信息
    void Serialize(Serializer &serializer) const;

    //! 尝试读取安装信息文件
    static unique_ptr<ExtensionInstallInfo> TryReadInfoFile(
        FileSystem &fs,
        const string &info_file_path,
        const string &extension_name);

    //! 反序列化安装信息
    static unique_ptr<ExtensionInstallInfo> Deserialize(
        Deserializer &deserializer);
};
```

### 3.2 安装信息文件

每个安装的扩展都有一个 `.info` 元数据文件：

```
扩展安装目录结构：
~/.duckdb/extensions/v1.2.0/linux_amd64/
├── parquet.duckdb_extension       # 扩展二进制
├── parquet.duckdb_extension.info  # 安装信息
├── json.duckdb_extension
├── json.duckdb_extension.info
└── icu.duckdb_extension
    icu.duckdb_extension.info
```

### 3.3 安装信息写入

```cpp
static void WriteExtensionMetadataFileToDisk(
    FileSystem &fs,
    const string &path,
    ExtensionInstallInfo &metadata) {

    auto file_writer = BufferedFileWriter(fs, path);
    BinarySerializer::Serialize(metadata, file_writer);
    file_writer.Sync();
}
```

## 4. 目录管理

### 4.1 扩展目录结构

```cpp
// 默认扩展目录
#ifdef _WIN32
#define DUCKDB_EXTENSION_DIRECTORIES "~\\.duckdb\\extensions"
#else
#define DUCKDB_EXTENSION_DIRECTORIES "~/.duckdb/extensions"
#endif

// 完整目录路径
// ~/.duckdb/extensions/{version}/{platform}/

vector<string> ExtensionHelper::GetExtensionDirectoryPath(
    DatabaseInstance &db, FileSystem &fs) {

    vector<string> extension_directories;
    auto &config = db.config;

    // 1. 用户配置的目录
    if (!config.options.extension_directory.empty()) {
        extension_directories.push_back(config.options.extension_directory);
    }

    // 2. 多目录配置
    if (!config.options.extension_directories.empty()) {
        for (const auto &dir : config.options.extension_directories) {
            extension_directories.push_back(dir);
        }
    }

    // 3. 默认目录
    if (extension_directories.empty()) {
        for (const auto &default_dir : DefaultExtensionFolders(fs)) {
            extension_directories.push_back(default_dir);
        }
    }

    // 添加版本和平台路径组件
    auto path_components = PathComponents();
    for (auto &extension_directory : extension_directories) {
        extension_directory = fs.ConvertSeparators(extension_directory);
        extension_directory = fs.ExpandPath(extension_directory);

        for (auto &path_ele : path_components) {
            extension_directory = fs.JoinPath(extension_directory, path_ele);
        }
    }

    return extension_directories;
}
```

### 4.2 路径组件

```cpp
const vector<string> ExtensionHelper::PathComponents() {
    return vector<string> {
        GetVersionDirectoryName(),  // 版本号或 commit hash
        DuckDB::Platform()          // 平台标识
    };
}

const string ExtensionHelper::GetVersionDirectoryName() {
#ifdef DUCKDB_WASM_VERSION
    return DUCKDB_QUOTE_DEFINE(DUCKDB_WASM_VERSION);
#endif
    if (IsRelease(DuckDB::LibraryVersion())) {
        return NormalizeVersionTag(DuckDB::LibraryVersion());
    } else {
        // 开发版本使用 git commit hash
        return DuckDB::SourceID();
    }
}
```

### 4.3 目录创建

```cpp
string ExtensionHelper::ExtensionDirectory(DatabaseInstance &db, FileSystem &fs) {
    auto extension_directories = GetExtensionDirectoryPath(db, fs);
    string extension_directory = extension_directories[0];

    if (!fs.DirectoryExists(extension_directory)) {
        string home_directory = fs.GetHomeDirectory();

        // 检查主目录是否存在
        if (extension_directory.rfind(home_directory, 0) == 0 &&
            !fs.DirectoryExists(home_directory)) {
            throw IOException(
                "Can't find the home directory at '%s'\n"
                "Specify a home directory using the SET "
                "home_directory='/path/to/dir' option.",
                home_directory);
        }

        // 递归创建目录
        fs.CreateDirectoriesRecursive(extension_directory);
    }

    D_ASSERT(fs.DirectoryExists(extension_directory));
    return extension_directory;
}
```

## 5. 安装流程详解

### 5.1 安装流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                       扩展安装流程                                   │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  INSTALL parquet;                                                    │
│       │                                                              │
│       ▼                                                              │
│  ┌─────────────────────┐                                            │
│  │ 1. 确定扩展来源      │                                            │
│  │   - 仓库 URL        │                                            │
│  │   - 本地路径        │                                            │
│  │   - 远程 URL        │                                            │
│  └──────────┬──────────┘                                            │
│             │                                                        │
│             ▼                                                        │
│  ┌─────────────────────┐                                            │
│  │ 2. 下载扩展文件      │                                            │
│  │   - HTTP GET 请求   │                                            │
│  │   - 本地文件读取    │                                            │
│  │   - GZIP 解压缩     │                                            │
│  └──────────┬──────────┘                                            │
│             │                                                        │
│             ▼                                                        │
│  ┌─────────────────────┐                                            │
│  │ 3. 验证扩展元数据    │                                            │
│  │   - 魔数检查        │                                            │
│  │   - 版本兼容性      │                                            │
│  │   - 平台匹配        │                                            │
│  └──────────┬──────────┘                                            │
│             │                                                        │
│             ▼                                                        │
│  ┌─────────────────────┐                                            │
│  │ 4. 写入本地目录      │                                            │
│  │   - 写入临时文件    │                                            │
│  │   - 写入 .info 文件 │                                            │
│  │   - 原子移动        │                                            │
│  └──────────┬──────────┘                                            │
│             │                                                        │
│             ▼                                                        │
│       安装完成                                                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.2 文件读取

```cpp
static unsafe_unique_array<data_t> ReadExtensionFileFromDisk(
    FileSystem &fs,
    const string &path,
    idx_t &file_size) {

    auto source_file = fs.OpenFile(path, FileFlags::FILE_FLAGS_READ);
    file_size = source_file->GetFileSize();

    auto in_buffer = make_unsafe_uniq_array<data_t>(file_size);
    source_file->Read(QueryContext(), in_buffer.get(), file_size);
    source_file->Close();

    return in_buffer;
}
```

### 5.3 解压与验证

```cpp
static void CheckExtensionMetadataOnInstall(
    DatabaseInstance &db,
    void *in_buffer,
    idx_t file_size,
    ExtensionInstallInfo &info,
    const string &extension_name) {

    // 检查文件大小
    if (file_size < ParsedExtensionMetaData::FOOTER_SIZE) {
        throw IOException(
            "Failed to install '%s', file too small!",
            extension_name);
    }

    // 解析元数据
    auto parsed_metadata = ExtensionHelper::ParseExtensionMetaData(
        static_cast<char *>(in_buffer) +
        (file_size - ParsedExtensionMetaData::FOOTER_SIZE));

    // 检查元数据有效性
    auto metadata_mismatch_error = parsed_metadata.GetInvalidMetadataError();

    if (!metadata_mismatch_error.empty() &&
        !DBConfig::GetSetting<AllowExtensionsMetadataMismatchSetting>(db)) {
        throw IOException(
            "Failed to install '%s'\n%s",
            extension_name, metadata_mismatch_error);
    }

    // 记录版本信息
    info.version = parsed_metadata.extension_version;
}
```

### 5.4 原子写入

```cpp
static void WriteExtensionFiles(
    FileSystem &fs,
    const string &temp_path,
    const string &local_extension_path,
    void *in_buffer,
    idx_t file_size,
    ExtensionInstallInfo &info) {

    // 1. 写入扩展到临时文件
    WriteExtensionFileToDisk(fs, temp_path, in_buffer, file_size);

    // 2. 写入元数据到临时文件
    auto metadata_tmp_path = temp_path + ".info";
    auto metadata_file_path = local_extension_path + ".info";
    WriteExtensionMetadataFileToDisk(fs, metadata_tmp_path, info);

    // 3. 原子移动（确保安装完整性）
    fs.MoveFile(metadata_tmp_path, metadata_file_path);
    fs.MoveFile(temp_path, local_extension_path);
}
```

## 6. 远程与本地安装

### 6.1 远程安装

```cpp
static unique_ptr<ExtensionInstallInfo> DirectInstallExtension(
    DatabaseInstance &db,
    FileSystem &fs,
    const string &path,
    const string &temp_path,
    const string &extension_name,
    const string &local_extension_path,
    ExtensionInstallOptions &options,
    optional_ptr<ClientContext> context) {

    string extension;
    string file;

    if (fs.IsRemoteFile(path, extension)) {
        file = path;

        // 自动加载 httpfs 扩展以支持 HTTPS
        if (context) {
            auto &db = DatabaseInstance::GetDatabase(*context);
            if (extension == "httpfs" &&
                !db.ExtensionIsLoaded("httpfs") &&
                db.config.options.autoload_known_extensions) {
                ExtensionHelper::AutoLoadExtension(*context, "httpfs");
            }
        }
    } else {
        file = fs.ConvertSeparators(path);
    }

    // 检查文件存在性
    bool exists = fs.FileExists(file);

    // 尝试不带 .gz 后缀
    if (!exists && StringUtil::EndsWith(file, ".gz")) {
        file = file.substr(0, file.size() - 3);
        exists = fs.FileExists(file);
    }

    if (!exists) {
        throw IOException("Failed to install extension \"%s\"", extension_name);
    }

    // 读取并处理文件
    idx_t file_size;
    auto in_buffer = ReadExtensionFileFromDisk(fs, file, file_size);

    // ... 解压、验证、写入 ...
}
```

### 6.2 仓库安装

```cpp
static unique_ptr<ExtensionInstallInfo> RepositoryInstallExtension(
    DatabaseInstance &db,
    FileSystem &fs,
    const string &extension_name,
    const string &temp_path,
    const string &local_extension_path,
    ExtensionInstallOptions &options,
    optional_ptr<ClientContext> context) {

    // 获取仓库配置
    ExtensionRepository repository;
    if (options.repository) {
        repository = *options.repository;
    } else {
        repository = ExtensionRepository::GetDefaultRepository(context);
    }

    // 构建下载 URL
    auto url_template = ExtensionUrlTemplate(&db, repository, options.version);
    auto url = ExtensionFinalizeUrlTemplate(url_template, extension_name);

    // 下载扩展
    // ... HTTP 请求处理 ...

    // 创建安装信息
    ExtensionInstallInfo info;
    info.mode = ExtensionInstallMode::REPOSITORY;
    info.repository_url = repository.path;

    return make_uniq<ExtensionInstallInfo>(std::move(info));
}
```

## 7. 错误处理与建议

### 7.1 扩展名建议

```cpp
bool ExtensionHelper::CreateSuggestions(
    const string &extension_name,
    string &message) {

    auto lowercase_extension_name = StringUtil::Lower(extension_name);
    vector<string> candidates;

    // 收集所有已知扩展名
    for (idx_t i = 0; i < DefaultExtensionCount(); i++) {
        candidates.emplace_back(GetDefaultExtension(i).name);
    }

    // 收集所有别名
    for (idx_t i = 0; i < ExtensionAliasCount(); i++) {
        candidates.emplace_back(GetExtensionAlias(i).alias);
    }

    // 使用 Jaro-Winkler 相似度查找最接近的扩展名
    auto closest_extensions = StringUtil::TopNJaroWinkler(
        candidates, lowercase_extension_name);

    message = StringUtil::CandidatesMessage(
        closest_extensions, "Candidate extensions");

    // 检查是否完全匹配
    for (auto &closest : closest_extensions) {
        if (closest == lowercase_extension_name) {
            message = "Extension \"" + extension_name +
                      "\" is an existing extension.\n";
            return true;
        }
    }

    return false;
}
```

### 7.2 安装帮助信息

```cpp
string ExtensionHelper::ExtensionInstallDocumentationLink(
    const string &extension_name) {

    auto components = PathComponents();

    string link = "https://duckdb.org/docs/stable/extensions/troubleshooting";

    if (components.size() >= 2) {
        link += "?version=" + components[0] +
                "&platform=" + components[1] +
                "&extension=" + extension_name;
    }

    return link;
}
```

## 8. 小结

本章详细分析了 DuckDB 的扩展安装机制：

1. **仓库系统**：支持官方、社区和自定义仓库
2. **安装信息**：ExtensionInstallInfo 记录安装来源和版本
3. **目录管理**：版本+平台的目录结构确保兼容性
4. **安装流程**：下载→验证→原子写入的完整流程
5. **错误处理**：智能建议和帮助文档链接

下一章将分析扩展的动态加载流程，包括符号解析、元数据验证和自动加载机制。
