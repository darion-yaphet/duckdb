# 第七章：扩展安全模型

## 概述

DuckDB 的扩展系统支持动态加载第三方代码，这带来了安全风险。为了确保用户安全，DuckDB 实现了多层安全机制，包括数字签名验证、信任级别管理和加载策略控制。本章将深入分析这些安全机制的实现。

## 扩展签名机制

### 签名结构

每个扩展文件的末尾包含一个 512 字节的元数据 footer，其中最后 256 字节是数字签名：

```
扩展二进制文件布局：
┌─────────────────────────────────┐
│         扩展代码和数据           │
│            (可变长度)            │
├─────────────────────────────────┤
│      元数据字段 (256 字节)       │
│   - 魔数 (32 字节)              │
│   - 平台 (32 字节)              │
│   - 版本信息 (32 字节)           │
│   - ABI 类型 (32 字节)          │
│   - 扩展版本 (32 字节)           │
│   - 保留字段 (96 字节)           │
├─────────────────────────────────┤
│      数字签名 (256 字节)         │
│        RSA-2048 签名            │
└─────────────────────────────────┘
```

### 签名算法

DuckDB 使用 RSA-2048 + SHA-256 签名：

1. **分块哈希**：将扩展文件（不含签名部分）分成 1MB 的块
2. **并行计算**：多线程计算每个块的 SHA-256 哈希
3. **二级哈希**：将所有块哈希连接后再次计算 SHA-256
4. **签名验证**：使用公钥验证签名与二级哈希是否匹配

```cpp
bool ExtensionHelper::CheckExtensionSignature(
    FileHandle &handle,
    ParsedExtensionMetaData &parsed_metadata,
    const bool allow_community_extensions) {

    auto signature_offset = handle.GetFileSize() -
        ParsedExtensionMetaData::SIGNATURE_SIZE;

    // 分块计算
    const idx_t maxLenChunks = 1024ULL * 1024ULL;  // 1MB
    const idx_t numChunks = (signature_offset + maxLenChunks - 1) / maxLenChunks;
    vector<string> hash_chunks(numChunks);

    // 并行计算每块的哈希
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

    // 合并哈希
    string hash_concatenation;
    for (auto &hash_chunk : hash_chunks) {
        hash_concatenation += hash_chunk;
    }

    // 二级哈希
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

### 公钥管理

DuckDB 内置了两组公钥：

```cpp
// 官方公钥（约 20 个）
static const char *const public_keys[] = {
    R"(
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA6aZuHUa1cLR9YDDYaEfi
UDbWY8m2t7b71S+k1ZkXfHqu+5drAxm+dIDzdOHOKZSIdwnJbT3sSqwFoG6PlXF3
...
-----END PUBLIC KEY-----
)",
    // ... 更多公钥
    nullptr
};

// 社区公钥（约 20 个）
static const char *const community_public_keys[] = {
    R"(
-----BEGIN PUBLIC KEY-----
MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAtXl28loGwAH3ZGQXXgJQ
3omhIEiUb3z9Petjl+jmdtEQnMNUFEZiXkfJB02UFWBL1OoKKnjiGhcr5oGiIZKR
...
-----END PUBLIC KEY-----
)",
    // ... 更多公钥
    nullptr
};
```

### 获取公钥

```cpp
const vector<string> ExtensionHelper::GetPublicKeys(bool allow_community_extensions) {
    vector<string> keys;

    // 始终包含官方公钥
    for (idx_t i = 0; public_keys[i]; i++) {
        keys.emplace_back(public_keys[i]);
    }

    // 如果允许社区扩展，添加社区公钥
    if (allow_community_extensions) {
        for (idx_t i = 0; community_public_keys[i]; i++) {
            keys.emplace_back(community_public_keys[i]);
        }
    }

    return keys;
}
```

## 信任级别

### 扩展来源分类

```
                    扩展信任级别
                         │
          ┌──────────────┼──────────────┐
          ▼              ▼              ▼
      官方扩展        社区扩展       未签名扩展
          │              │              │
          ▼              ▼              ▼
    官方公钥验证   社区公钥验证     无法验证
          │              │              │
          ▼              ▼              ▼
    默认允许加载   需启用配置      需显式允许
```

### 信任策略配置

| 配置选项 | 默认值 | 说明 |
|---------|-------|------|
| `allow_unsigned_extensions` | false | 允许加载未签名扩展 |
| `allow_community_extensions` | true | 允许加载社区签名扩展 |
| `allow_extensions_metadata_mismatch` | false | 允许元数据不匹配 |
| `enable_external_access` | true | 允许外部访问（含扩展加载） |

### 加载决策流程

```cpp
bool ExtensionHelper::TryInitialLoad(...) {
    // 1. 检查外部访问权限
    if (!db.config.options.enable_external_access) {
        throw PermissionException(
            "Loading external extensions is disabled through configuration");
    }

    // 2. 解析元数据
    auto parsed_metadata = ParseExtensionMetaData(*handle);
    auto metadata_mismatch_error = parsed_metadata.GetInvalidMetadataError();

    // 3. 签名验证
    if (!db.config.options.allow_unsigned_extensions) {
        bool signature_valid;

        if (parsed_metadata.AppearsValid()) {
            signature_valid = CheckExtensionSignature(
                *handle, parsed_metadata,
                db.config.options.allow_community_extensions);
        } else {
            signature_valid = false;
        }

        // 元数据不匹配时抛出错误
        if (!metadata_mismatch_error.empty()) {
            throw InvalidInputException(metadata_mismatch_error);
        }

        // 签名无效时抛出错误
        if (!signature_valid) {
            throw IOException(db.config.error_manager->FormatException(
                ErrorType::UNSIGNED_EXTENSION, filename));
        }
    } else if (!DBConfig::GetSetting<AllowExtensionsMetadataMismatchSetting>(db)) {
        // 允许未签名但不允许元数据不匹配
        if (!metadata_mismatch_error.empty()) {
            throw InvalidInputException(metadata_mismatch_error);
        }
    }

    // 4. 继续加载...
}
```

## 元数据验证

### 解析元数据

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

    return result;
}
```

### 魔数验证

```cpp
// 有效的魔数值
static constexpr const char *DUCKDB_EXTENSION_MAGIC = "4duck5db";

bool ParsedExtensionMetaData::AppearsValid() const {
    return magic_value == DUCKDB_EXTENSION_MAGIC;
}
```

### 元数据不匹配检测

```cpp
string ParsedExtensionMetaData::GetInvalidMetadataError() const {
    // 检查魔数
    if (!AppearsValid()) {
        return "Invalid extension magic value";
    }

    // 检查平台
    string current_platform = DuckDB::Platform();
    if (platform != current_platform) {
        return StringUtil::Format(
            "Extension platform mismatch: expected '%s' but got '%s'",
            current_platform, platform);
    }

    // 检查版本（C++ ABI 需要精确匹配）
    if (abi_type == ExtensionABIType::CPP) {
        string current_version = ExtensionHelper::GetVersionDirectoryName();
        if (duckdb_version != current_version) {
            return StringUtil::Format(
                "Extension version mismatch: expected '%s' but got '%s'",
                current_version, duckdb_version);
        }
    }

    return "";  // 验证通过
}
```

## 版本安全

### ABI 兼容性

| ABI 类型 | 版本要求 | 风险 |
|---------|---------|------|
| CPP | 精确匹配 | 版本不匹配会导致崩溃 |
| C_STRUCT | 向后兼容 | 可使用旧版本扩展 |
| C_STRUCT_UNSTABLE | 精确匹配 | 使用不稳定 API |

### 版本混淆攻击防护

为防止攻击者用旧版本扩展利用已修复的漏洞，DuckDB 实施以下保护：

1. **版本字段验证**：元数据中的版本必须与当前 DuckDB 版本匹配（C++ ABI）
2. **平台验证**：平台字符串必须与当前平台匹配
3. **签名覆盖范围**：签名覆盖整个扩展文件，包括元数据

```cpp
// 版本目录名生成
string ExtensionHelper::GetVersionDirectoryName() {
#ifdef DUCKDB_VERSION
    // 发布版本：使用版本号
    return DUCKDB_VERSION;  // 如 "v1.2.0"
#else
    // 开发版本：使用 git commit hash
    return DUCKDB_SOURCE_ID;  // 如 "abc123..."
#endif
}
```

## 安全加载策略

### 禁用扩展加载

编译时可完全禁用扩展加载：

```cpp
#ifdef DUCKDB_DISABLE_EXTENSION_LOAD
bool ExtensionHelper::TryInitialLoad(...) {
    throw PermissionException(
        "Loading external extensions is disabled through a compile time flag");
}
#endif
```

### 运行时禁用

```sql
-- 禁用外部访问（包括扩展加载）
SET enable_external_access = false;
```

### 仅允许官方扩展

```sql
-- 禁用社区扩展
SET allow_community_extensions = false;
```

## 仓库安全

### 仓库信任级别

```cpp
// 扩展仓库定义
struct ExtensionRepository {
    string path;
    RepositoryType type;

    enum class RepositoryType {
        CORE,       // 官方核心仓库
        COMMUNITY,  // 社区仓库
        CUSTOM      // 自定义仓库
    };
};
```

### 官方仓库

```cpp
static constexpr const char *CORE_REPOSITORY_URL =
    "http://extensions.duckdb.org";
static constexpr const char *NIGHTLY_REPOSITORY_URL =
    "http://nightly-extensions.duckdb.org";
static constexpr const char *COMMUNITY_REPOSITORY_URL =
    "http://community-extensions.duckdb.org";
```

### 仓库验证

从官方仓库下载的扩展会进行签名验证，确保：

1. 扩展由官方构建流程产生
2. 扩展在传输过程中未被篡改
3. 扩展适用于当前平台和版本

## 安全最佳实践

### 生产环境配置

```sql
-- 推荐的生产环境安全配置

-- 禁用未签名扩展
SET allow_unsigned_extensions = false;

-- 仅允许官方扩展
SET allow_community_extensions = false;

-- 禁止元数据不匹配
SET allow_extensions_metadata_mismatch = false;

-- 限制自动安装
SET autoinstall_known_extensions = false;
```

### 开发环境配置

```sql
-- 开发环境可以放宽限制

-- 允许加载本地开发的未签名扩展
SET allow_unsigned_extensions = true;

-- 允许元数据不匹配（用于测试）
SET allow_extensions_metadata_mismatch = true;
```

### 扩展审计

```sql
-- 查看已加载的扩展
SELECT * FROM duckdb_extensions();

-- 查看扩展安装信息
SELECT * FROM duckdb_extensions()
WHERE installed AND loaded;
```

## 安全验证流程图

```
                    LOAD 'extension'
                          │
                          ▼
                 enable_external_access?
                          │
           ┌──────────────┴──────────────┐
           ▼                             ▼
         false                         true
           │                             │
           ▼                             ▼
    PermissionException            解析元数据
                                        │
                                        ▼
                              AppearsValid()?
                                        │
                 ┌──────────────────────┴──────────────────────┐
                 ▼                                             ▼
               false                                         true
                 │                                             │
                 ▼                                             ▼
    InvalidInputException                           allow_unsigned_extensions?
                                                              │
                                     ┌────────────────────────┴────────────────────────┐
                                     ▼                                                 ▼
                                   true                                              false
                                     │                                                 │
                                     ▼                                                 ▼
                              跳过签名验证                                      验证签名
                                     │                                                 │
                                     │                      ┌──────────────────────────┴──────────────────────────┐
                                     │                      ▼                                                     ▼
                                     │               签名有效                                               签名无效
                                     │                      │                                                     │
                                     │                      ▼                                                     ▼
                                     │               验证平台和版本                                     UNSIGNED_EXTENSION
                                     │                      │                                                  错误
                                     │           ┌──────────┴──────────┐
                                     │           ▼                     ▼
                                     │        匹配                  不匹配
                                     │           │                     │
                                     │           │                     ▼
                                     │           │            元数据不匹配错误
                                     │           │
                                     └─────────┬─┘
                                               ▼
                                        加载动态库
                                               │
                                               ▼
                                        调用入口函数
                                               │
                                               ▼
                                           加载完成
```

## 小结

本章详细分析了 DuckDB 扩展的安全模型：

1. **数字签名**：使用 RSA-2048 + SHA-256，支持并行计算提高验证效率
2. **公钥管理**：内置官方和社区两组公钥，可配置是否信任社区扩展
3. **元数据验证**：检查魔数、平台和版本，确保扩展适用于当前环境
4. **信任级别**：官方扩展、社区扩展和未签名扩展三个级别
5. **配置灵活性**：支持生产环境严格配置和开发环境宽松配置

这套安全机制在保护用户安全的同时，也为扩展开发和分发提供了便利。
