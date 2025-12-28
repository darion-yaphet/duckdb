# 第六章：C API 扩展接口

## 概述

DuckDB 提供了稳定的 C API 接口，允许使用 C 或其他支持 C FFI 的语言开发扩展。与 C++ ABI 扩展不同，C API 扩展具有向后兼容性，可以在不同版本的 DuckDB 之间使用。本章将深入分析 C API 的结构设计、函数分类、版本管理和使用方法。

## duckdb_ext_api_v1 结构

### 设计理念

C API 采用函数指针表的设计，所有 DuckDB API 函数都作为函数指针存储在一个结构体中：

```cpp
// extension_api.hpp
typedef struct {
    // v1.2.0 - 初始版本
    duckdb_state (*duckdb_open)(const char *path, duckdb_database *out_database);
    duckdb_state (*duckdb_open_ext)(const char *path, duckdb_database *out_database,
                                    duckdb_config config, char **out_error);
    void (*duckdb_close)(duckdb_database *database);
    // ... 数百个函数指针
} duckdb_ext_api_v1;
```

这种设计的优势：
1. **ABI 稳定性**：函数指针表的布局在编译时确定，不依赖 C++ ABI
2. **向后兼容**：新版本只在末尾添加函数，旧扩展仍可使用前面的函数
3. **跨语言支持**：任何支持 C FFI 的语言都可以使用

### API 结构创建

```cpp
inline duckdb_ext_api_v1 CreateAPIv1() {
    duckdb_ext_api_v1 result;

    // 数据库连接
    result.duckdb_open = duckdb_open;
    result.duckdb_open_ext = duckdb_open_ext;
    result.duckdb_close = duckdb_close;
    result.duckdb_connect = duckdb_connect;
    result.duckdb_disconnect = duckdb_disconnect;

    // 查询执行
    result.duckdb_query = duckdb_query;
    result.duckdb_prepare = duckdb_prepare;
    result.duckdb_execute_prepared = duckdb_execute_prepared;

    // 类型系统
    result.duckdb_create_logical_type = duckdb_create_logical_type;
    result.duckdb_destroy_logical_type = duckdb_destroy_logical_type;

    // ... 填充所有函数指针

    return result;
}
```

## API 函数分类

### 数据库连接 API

```cpp
// 打开数据库
duckdb_state (*duckdb_open)(const char *path, duckdb_database *out_database);
duckdb_state (*duckdb_open_ext)(const char *path, duckdb_database *out_database,
                                duckdb_config config, char **out_error);
void (*duckdb_close)(duckdb_database *database);

// 创建连接
duckdb_state (*duckdb_connect)(duckdb_database database,
                               duckdb_connection *out_connection);
void (*duckdb_disconnect)(duckdb_connection *connection);
void (*duckdb_interrupt)(duckdb_connection connection);

// 查询进度
duckdb_query_progress_type (*duckdb_query_progress)(duckdb_connection connection);

// 版本信息
const char *(*duckdb_library_version)();
```

### 配置 API

```cpp
// 配置创建与销毁
duckdb_state (*duckdb_create_config)(duckdb_config *out_config);
void (*duckdb_destroy_config)(duckdb_config *config);

// 配置选项
size_t (*duckdb_config_count)();
duckdb_state (*duckdb_get_config_flag)(size_t index, const char **out_name,
                                        const char **out_description);
duckdb_state (*duckdb_set_config)(duckdb_config config, const char *name,
                                   const char *option);

// 自定义配置选项
duckdb_config_option (*duckdb_create_config_option)();
void (*duckdb_config_option_set_name)(duckdb_config_option option, const char *name);
void (*duckdb_config_option_set_type)(duckdb_config_option option,
                                       duckdb_logical_type type);
duckdb_state (*duckdb_register_config_option)(duckdb_connection connection,
                                               duckdb_config_option option);
```

### 查询执行 API

```cpp
// 直接查询
duckdb_state (*duckdb_query)(duckdb_connection connection, const char *query,
                              duckdb_result *out_result);

// 预处理语句
duckdb_state (*duckdb_prepare)(duckdb_connection connection, const char *query,
                                duckdb_prepared_statement *out_prepared_statement);
void (*duckdb_destroy_prepare)(duckdb_prepared_statement *prepared_statement);
const char *(*duckdb_prepare_error)(duckdb_prepared_statement prepared_statement);

// 参数绑定
duckdb_state (*duckdb_bind_boolean)(duckdb_prepared_statement prepared_statement,
                                     idx_t param_idx, bool val);
duckdb_state (*duckdb_bind_int64)(duckdb_prepared_statement prepared_statement,
                                   idx_t param_idx, int64_t val);
duckdb_state (*duckdb_bind_double)(duckdb_prepared_statement prepared_statement,
                                    idx_t param_idx, double val);
duckdb_state (*duckdb_bind_varchar)(duckdb_prepared_statement prepared_statement,
                                     idx_t param_idx, const char *val);
duckdb_state (*duckdb_bind_null)(duckdb_prepared_statement prepared_statement,
                                  idx_t param_idx);

// 执行预处理语句
duckdb_state (*duckdb_execute_prepared)(duckdb_prepared_statement prepared_statement,
                                         duckdb_result *out_result);

// 异步执行（Pending Result）
duckdb_state (*duckdb_pending_prepared)(duckdb_prepared_statement prepared_statement,
                                         duckdb_pending_result *out_result);
duckdb_pending_state (*duckdb_pending_execute_task)(duckdb_pending_result pending_result);
duckdb_state (*duckdb_execute_pending)(duckdb_pending_result pending_result,
                                        duckdb_result *out_result);
```

### 结果处理 API

```cpp
// 结果元数据
idx_t (*duckdb_column_count)(duckdb_result *result);
const char *(*duckdb_column_name)(duckdb_result *result, idx_t col);
duckdb_type (*duckdb_column_type)(duckdb_result *result, idx_t col);
duckdb_logical_type (*duckdb_column_logical_type)(duckdb_result *result, idx_t col);

// 行数与状态
idx_t (*duckdb_rows_changed)(duckdb_result *result);
const char *(*duckdb_result_error)(duckdb_result *result);
duckdb_result_type (*duckdb_result_return_type)(duckdb_result result);

// 结果销毁
void (*duckdb_destroy_result)(duckdb_result *result);

// 数据获取（新 API - DataChunk）
duckdb_data_chunk (*duckdb_fetch_chunk)(duckdb_result result);
idx_t (*duckdb_data_chunk_get_size)(duckdb_data_chunk chunk);
duckdb_vector (*duckdb_data_chunk_get_vector)(duckdb_data_chunk chunk, idx_t col_idx);
```

### 类型系统 API

```cpp
// 基础类型创建
duckdb_logical_type (*duckdb_create_logical_type)(duckdb_type type);
void (*duckdb_destroy_logical_type)(duckdb_logical_type *type);

// 复合类型创建
duckdb_logical_type (*duckdb_create_list_type)(duckdb_logical_type type);
duckdb_logical_type (*duckdb_create_array_type)(duckdb_logical_type type,
                                                 idx_t array_size);
duckdb_logical_type (*duckdb_create_map_type)(duckdb_logical_type key_type,
                                               duckdb_logical_type value_type);
duckdb_logical_type (*duckdb_create_struct_type)(duckdb_logical_type *member_types,
                                                  const char **member_names,
                                                  idx_t member_count);
duckdb_logical_type (*duckdb_create_enum_type)(const char **member_names,
                                                idx_t member_count);
duckdb_logical_type (*duckdb_create_decimal_type)(uint8_t width, uint8_t scale);

// 类型信息查询
duckdb_type (*duckdb_get_type_id)(duckdb_logical_type type);
char *(*duckdb_logical_type_get_alias)(duckdb_logical_type type);
uint8_t (*duckdb_decimal_width)(duckdb_logical_type type);
uint8_t (*duckdb_decimal_scale)(duckdb_logical_type type);
idx_t (*duckdb_struct_type_child_count)(duckdb_logical_type type);
char *(*duckdb_struct_type_child_name)(duckdb_logical_type type, idx_t index);
```

### 值操作 API

```cpp
// 值创建
duckdb_value (*duckdb_create_bool)(bool input);
duckdb_value (*duckdb_create_int64)(int64_t val);
duckdb_value (*duckdb_create_double)(double input);
duckdb_value (*duckdb_create_varchar)(const char *text);
duckdb_value (*duckdb_create_blob)(const uint8_t *data, idx_t length);
duckdb_value (*duckdb_create_null_value)();

// 复合值创建
duckdb_value (*duckdb_create_list_value)(duckdb_logical_type type,
                                          duckdb_value *values, idx_t value_count);
duckdb_value (*duckdb_create_struct_value)(duckdb_logical_type type,
                                            duckdb_value *values);
duckdb_value (*duckdb_create_map_value)(duckdb_logical_type map_type,
                                         duckdb_value *keys, duckdb_value *values,
                                         idx_t entry_count);

// 值读取
bool (*duckdb_get_bool)(duckdb_value val);
int64_t (*duckdb_get_int64)(duckdb_value val);
double (*duckdb_get_double)(duckdb_value val);
char *(*duckdb_get_varchar)(duckdb_value value);
duckdb_blob (*duckdb_get_blob)(duckdb_value val);
bool (*duckdb_is_null_value)(duckdb_value value);

// 值销毁
void (*duckdb_destroy_value)(duckdb_value *value);
```

### Vector 操作 API

```cpp
// Vector 数据访问
void *(*duckdb_vector_get_data)(duckdb_vector vector);
uint64_t *(*duckdb_vector_get_validity)(duckdb_vector vector);
duckdb_logical_type (*duckdb_vector_get_column_type)(duckdb_vector vector);

// 有效性操作
bool (*duckdb_validity_row_is_valid)(uint64_t *validity, idx_t row);
void (*duckdb_validity_set_row_validity)(uint64_t *validity, idx_t row, bool valid);
void (*duckdb_vector_ensure_validity_writable)(duckdb_vector vector);

// 字符串赋值
void (*duckdb_vector_assign_string_element)(duckdb_vector vector, idx_t index,
                                             const char *str);
void (*duckdb_vector_assign_string_element_len)(duckdb_vector vector, idx_t index,
                                                 const char *str, idx_t str_len);

// 嵌套类型访问
duckdb_vector (*duckdb_list_vector_get_child)(duckdb_vector vector);
duckdb_vector (*duckdb_struct_vector_get_child)(duckdb_vector vector, idx_t index);
duckdb_vector (*duckdb_array_vector_get_child)(duckdb_vector vector);
idx_t (*duckdb_list_vector_get_size)(duckdb_vector vector);
duckdb_state (*duckdb_list_vector_set_size)(duckdb_vector vector, idx_t size);
```

### DataChunk API

```cpp
// 创建与销毁
duckdb_data_chunk (*duckdb_create_data_chunk)(duckdb_logical_type *types,
                                               idx_t column_count);
void (*duckdb_destroy_data_chunk)(duckdb_data_chunk *chunk);
void (*duckdb_data_chunk_reset)(duckdb_data_chunk chunk);

// 访问
idx_t (*duckdb_data_chunk_get_column_count)(duckdb_data_chunk chunk);
duckdb_vector (*duckdb_data_chunk_get_vector)(duckdb_data_chunk chunk, idx_t col_idx);
idx_t (*duckdb_data_chunk_get_size)(duckdb_data_chunk chunk);
void (*duckdb_data_chunk_set_size)(duckdb_data_chunk chunk, idx_t size);
```

## 扩展开发 API

### 标量函数注册

```cpp
// 创建标量函数
duckdb_scalar_function (*duckdb_create_scalar_function)();
void (*duckdb_destroy_scalar_function)(duckdb_scalar_function *scalar_function);

// 设置函数属性
void (*duckdb_scalar_function_set_name)(duckdb_scalar_function scalar_function,
                                         const char *name);
void (*duckdb_scalar_function_add_parameter)(duckdb_scalar_function scalar_function,
                                              duckdb_logical_type type);
void (*duckdb_scalar_function_set_return_type)(duckdb_scalar_function scalar_function,
                                                duckdb_logical_type type);
void (*duckdb_scalar_function_set_varargs)(duckdb_scalar_function scalar_function,
                                            duckdb_logical_type type);

// 设置回调
void (*duckdb_scalar_function_set_function)(duckdb_scalar_function scalar_function,
                                             duckdb_scalar_function_t function);
void (*duckdb_scalar_function_set_extra_info)(duckdb_scalar_function scalar_function,
                                               void *extra_info,
                                               duckdb_delete_callback_t destroy);
void (*duckdb_scalar_function_set_bind)(duckdb_scalar_function scalar_function,
                                         duckdb_scalar_function_bind_t bind);

// 注册
duckdb_state (*duckdb_register_scalar_function)(duckdb_connection con,
                                                 duckdb_scalar_function scalar_function);

// 函数集（重载）
duckdb_scalar_function_set (*duckdb_create_scalar_function_set)(const char *name);
duckdb_state (*duckdb_add_scalar_function_to_set)(duckdb_scalar_function_set set,
                                                   duckdb_scalar_function function);
duckdb_state (*duckdb_register_scalar_function_set)(duckdb_connection con,
                                                     duckdb_scalar_function_set set);
```

### 聚合函数注册

```cpp
// 创建聚合函数
duckdb_aggregate_function (*duckdb_create_aggregate_function)();
void (*duckdb_destroy_aggregate_function)(duckdb_aggregate_function *aggregate_function);

// 设置属性
void (*duckdb_aggregate_function_set_name)(duckdb_aggregate_function aggregate_function,
                                            const char *name);
void (*duckdb_aggregate_function_add_parameter)(duckdb_aggregate_function aggregate_function,
                                                 duckdb_logical_type type);
void (*duckdb_aggregate_function_set_return_type)(duckdb_aggregate_function aggregate_function,
                                                   duckdb_logical_type type);

// 设置回调（聚合需要多个回调）
void (*duckdb_aggregate_function_set_functions)(
    duckdb_aggregate_function aggregate_function,
    duckdb_aggregate_state_size state_size,      // 状态大小
    duckdb_aggregate_init_t state_init,          // 状态初始化
    duckdb_aggregate_update_t update,            // 更新
    duckdb_aggregate_combine_t combine,          // 合并
    duckdb_aggregate_finalize_t finalize);       // 最终化

void (*duckdb_aggregate_function_set_destructor)(
    duckdb_aggregate_function aggregate_function,
    duckdb_aggregate_destroy_t destroy);

// 注册
duckdb_state (*duckdb_register_aggregate_function)(
    duckdb_connection con,
    duckdb_aggregate_function aggregate_function);
```

### 表函数注册

```cpp
// 创建表函数
duckdb_table_function (*duckdb_create_table_function)();
void (*duckdb_destroy_table_function)(duckdb_table_function *table_function);

// 设置属性
void (*duckdb_table_function_set_name)(duckdb_table_function table_function,
                                        const char *name);
void (*duckdb_table_function_add_parameter)(duckdb_table_function table_function,
                                             duckdb_logical_type type);
void (*duckdb_table_function_add_named_parameter)(duckdb_table_function table_function,
                                                   const char *name,
                                                   duckdb_logical_type type);

// 设置回调
void (*duckdb_table_function_set_bind)(duckdb_table_function table_function,
                                        duckdb_table_function_bind_t bind);
void (*duckdb_table_function_set_init)(duckdb_table_function table_function,
                                        duckdb_table_function_init_t init);
void (*duckdb_table_function_set_local_init)(duckdb_table_function table_function,
                                              duckdb_table_function_init_t init);
void (*duckdb_table_function_set_function)(duckdb_table_function table_function,
                                            duckdb_table_function_t function);

// 投影下推
void (*duckdb_table_function_supports_projection_pushdown)(
    duckdb_table_function table_function, bool pushdown);

// 注册
duckdb_state (*duckdb_register_table_function)(duckdb_connection con,
                                                duckdb_table_function function);
```

### 类型转换函数注册

```cpp
// 创建转换函数
duckdb_cast_function (*duckdb_create_cast_function)();
void (*duckdb_destroy_cast_function)(duckdb_cast_function *cast_function);

// 设置属性
void (*duckdb_cast_function_set_source_type)(duckdb_cast_function cast_function,
                                              duckdb_logical_type source_type);
void (*duckdb_cast_function_set_target_type)(duckdb_cast_function cast_function,
                                              duckdb_logical_type target_type);
void (*duckdb_cast_function_set_implicit_cast_cost)(duckdb_cast_function cast_function,
                                                     int64_t cost);
void (*duckdb_cast_function_set_function)(duckdb_cast_function cast_function,
                                           duckdb_cast_function_t function);

// 注册
duckdb_state (*duckdb_register_cast_function)(duckdb_connection con,
                                               duckdb_cast_function cast_function);
```

### Copy 函数注册

```cpp
// 创建 Copy 函数
duckdb_copy_function (*duckdb_create_copy_function)();
void (*duckdb_copy_function_set_name)(duckdb_copy_function copy_function,
                                       const char *name);

// 设置回调
void (*duckdb_copy_function_set_bind)(duckdb_copy_function copy_function,
                                       duckdb_copy_function_bind_t bind);
void (*duckdb_copy_function_set_global_init)(duckdb_copy_function copy_function,
                                              duckdb_copy_function_global_init_t init);
void (*duckdb_copy_function_set_sink)(duckdb_copy_function copy_function,
                                       duckdb_copy_function_sink_t function);
void (*duckdb_copy_function_set_finalize)(duckdb_copy_function copy_function,
                                           duckdb_copy_function_finalize_t finalize);
void (*duckdb_copy_function_set_copy_from_function)(duckdb_copy_function copy_function,
                                                     duckdb_table_function table_function);

// 注册
duckdb_state (*duckdb_register_copy_function)(duckdb_connection connection,
                                               duckdb_copy_function copy_function);
```

## C 扩展入口

### 入口函数规范

```cpp
// C 扩展入口函数签名
typedef bool (*ext_init_c_api_fun_t)(duckdb_extension_info info,
                                      duckdb_extension_access *access);

// 访问结构
typedef struct {
    void (*set_error)(duckdb_extension_info info, const char *error);
    duckdb_database (*get_database)(duckdb_extension_info info);
    const void *(*get_api)(duckdb_extension_info info, const char *version);
} duckdb_extension_access;
```

### 扩展实现示例

```c
#include "duckdb.h"

// 标量函数实现
void my_add_function(duckdb_function_info info, duckdb_data_chunk input,
                     duckdb_vector output) {
    duckdb_ext_api_v1 *api = (duckdb_ext_api_v1 *)duckdb_function_get_extra_info(info);

    idx_t count = api->duckdb_data_chunk_get_size(input);
    duckdb_vector left = api->duckdb_data_chunk_get_vector(input, 0);
    duckdb_vector right = api->duckdb_data_chunk_get_vector(input, 1);

    int64_t *left_data = (int64_t *)api->duckdb_vector_get_data(left);
    int64_t *right_data = (int64_t *)api->duckdb_vector_get_data(right);
    int64_t *result_data = (int64_t *)api->duckdb_vector_get_data(output);

    for (idx_t i = 0; i < count; i++) {
        result_data[i] = left_data[i] + right_data[i];
    }
}

// 入口函数
bool my_extension_init_c_api(duckdb_extension_info info,
                              duckdb_extension_access *access) {
    // 获取 API
    duckdb_ext_api_v1 *api = (duckdb_ext_api_v1 *)access->get_api(info, "v1.2.0");
    if (!api) {
        access->set_error(info, "Failed to get DuckDB C API");
        return false;
    }

    // 获取数据库连接
    duckdb_database db = access->get_database(info);
    duckdb_connection conn;
    if (api->duckdb_connect(db, &conn) != DuckDBSuccess) {
        access->set_error(info, "Failed to create connection");
        return false;
    }

    // 创建标量函数
    duckdb_scalar_function func = api->duckdb_create_scalar_function();
    api->duckdb_scalar_function_set_name(func, "my_add");

    // 添加参数
    duckdb_logical_type bigint_type = api->duckdb_create_logical_type(DUCKDB_TYPE_BIGINT);
    api->duckdb_scalar_function_add_parameter(func, bigint_type);
    api->duckdb_scalar_function_add_parameter(func, bigint_type);
    api->duckdb_scalar_function_set_return_type(func, bigint_type);
    api->duckdb_destroy_logical_type(&bigint_type);

    // 设置函数实现
    api->duckdb_scalar_function_set_function(func, my_add_function);
    api->duckdb_scalar_function_set_extra_info(func, api, NULL);

    // 注册函数
    if (api->duckdb_register_scalar_function(conn, func) != DuckDBSuccess) {
        api->duckdb_destroy_scalar_function(&func);
        api->duckdb_disconnect(&conn);
        access->set_error(info, "Failed to register function");
        return false;
    }

    api->duckdb_destroy_scalar_function(&func);
    api->duckdb_disconnect(&conn);

    return true;
}
```

## API 版本管理

### 版本检查

```cpp
// extension_load.cpp
static const void *GetAPI(duckdb_extension_info info, const char *version) {
    string version_string = version;
    auto &load_state = DuckDBExtensionLoadState::Get(info);

    if (load_state.init_result.abi_type == ExtensionABIType::C_STRUCT) {
        // 解析请求的版本
        idx_t major, minor, patch;
        auto parsed = VersioningUtils::ParseSemver(version_string,
                                                    major, minor, patch);

        // 检查版本兼容性
        if (!parsed || !VersioningUtils::IsSupportedCAPIVersion(
                major, minor, patch)) {
            load_state.has_error = true;
            load_state.error_data = ErrorData(
                ExceptionType::UNKNOWN_TYPE,
                "Unsupported C API version: " + version_string);
            return nullptr;
        }
    } else if (load_state.init_result.abi_type ==
               ExtensionABIType::C_STRUCT_UNSTABLE) {
        // C_STRUCT_UNSTABLE 不检查版本
        // 扩展与 DuckDB 版本 1:1 绑定
    }

    // 返回 API 结构
    load_state.api_struct = load_state.db.GetExtensionAPIV1();
    return &load_state.api_struct;
}
```

### 稳定 vs 不稳定 API

| 特性 | C_STRUCT (稳定) | C_STRUCT_UNSTABLE |
|-----|----------------|-------------------|
| 版本检查 | 语义化版本检查 | 无检查 |
| 向后兼容 | 是 | 否 |
| 适用场景 | 发布版扩展 | 开发测试 |
| API 可用性 | 仅稳定 API | 全部 API |

## 内存管理

### 分配与释放

```cpp
// 内存分配（对齐到 DuckDB 内部要求）
void *(*duckdb_malloc)(size_t size);
void (*duckdb_free)(void *ptr);
```

### 资源销毁模式

```cpp
// 所有创建的资源都需要销毁
void (*duckdb_destroy_value)(duckdb_value *value);
void (*duckdb_destroy_logical_type)(duckdb_logical_type *type);
void (*duckdb_destroy_result)(duckdb_result *result);
void (*duckdb_destroy_data_chunk)(duckdb_data_chunk *chunk);
void (*duckdb_destroy_scalar_function)(duckdb_scalar_function *scalar_function);
void (*duckdb_destroy_table_function)(duckdb_table_function *table_function);
// ...
```

### 回调销毁

```cpp
// 为 extra_info 设置销毁回调
typedef void (*duckdb_delete_callback_t)(void *);

void (*duckdb_scalar_function_set_extra_info)(
    duckdb_scalar_function scalar_function,
    void *extra_info,
    duckdb_delete_callback_t destroy);  // 当函数被销毁时调用
```

## 错误处理

### 扩展初始化错误

```cpp
// 在入口函数中报告错误
access->set_error(info, "Error message");
return false;
```

### 函数执行错误

```cpp
// 在函数实现中报告错误
void (*duckdb_scalar_function_set_error)(duckdb_function_info info,
                                          const char *error);
void (*duckdb_aggregate_function_set_error)(duckdb_function_info info,
                                             const char *error);
void (*duckdb_function_set_error)(duckdb_function_info info, const char *error);
```

### 绑定阶段错误

```cpp
void (*duckdb_bind_set_error)(duckdb_bind_info info, const char *error);
void (*duckdb_init_set_error)(duckdb_init_info info, const char *error);
```

## 文件系统 API

```cpp
// 获取文件系统
duckdb_file_system (*duckdb_client_context_get_file_system)(
    duckdb_client_context context);
void (*duckdb_destroy_file_system)(duckdb_file_system *file_system);

// 文件操作
duckdb_state (*duckdb_file_system_open)(duckdb_file_system file_system,
                                         const char *path,
                                         duckdb_file_open_options options,
                                         duckdb_file_handle *out_file);

// 文件句柄操作
int64_t (*duckdb_file_handle_read)(duckdb_file_handle file_handle,
                                    void *buffer, int64_t size);
int64_t (*duckdb_file_handle_write)(duckdb_file_handle file_handle,
                                     const void *buffer, int64_t size);
duckdb_state (*duckdb_file_handle_seek)(duckdb_file_handle file_handle,
                                         int64_t position);
int64_t (*duckdb_file_handle_tell)(duckdb_file_handle file_handle);
duckdb_state (*duckdb_file_handle_sync)(duckdb_file_handle file_handle);
int64_t (*duckdb_file_handle_size)(duckdb_file_handle file_handle);
```

## Appender API

```cpp
// 创建 Appender
duckdb_state (*duckdb_appender_create)(duckdb_connection connection,
                                        const char *schema, const char *table,
                                        duckdb_appender *out_appender);

// 添加数据
duckdb_state (*duckdb_append_bool)(duckdb_appender appender, bool value);
duckdb_state (*duckdb_append_int64)(duckdb_appender appender, int64_t value);
duckdb_state (*duckdb_append_double)(duckdb_appender appender, double value);
duckdb_state (*duckdb_append_varchar)(duckdb_appender appender, const char *val);
duckdb_state (*duckdb_append_null)(duckdb_appender appender);

// 行结束
duckdb_state (*duckdb_appender_end_row)(duckdb_appender appender);

// 批量添加
duckdb_state (*duckdb_append_data_chunk)(duckdb_appender appender,
                                          duckdb_data_chunk chunk);

// 刷新与关闭
duckdb_state (*duckdb_appender_flush)(duckdb_appender appender);
duckdb_state (*duckdb_appender_close)(duckdb_appender appender);
duckdb_state (*duckdb_appender_destroy)(duckdb_appender *appender);
```

## API 分类总结

| 类别 | 函数数量 | 用途 |
|-----|---------|------|
| 数据库连接 | ~10 | 打开/关闭数据库和连接 |
| 配置管理 | ~15 | 配置选项管理 |
| 查询执行 | ~30 | SQL 执行和预处理语句 |
| 结果处理 | ~20 | 结果元数据和数据访问 |
| 类型系统 | ~40 | 类型创建和查询 |
| 值操作 | ~60 | 值创建、读取和销毁 |
| Vector/DataChunk | ~30 | 向量化数据操作 |
| 标量函数 | ~15 | 标量函数注册 |
| 聚合函数 | ~15 | 聚合函数注册 |
| 表函数 | ~20 | 表函数注册 |
| 类型转换 | ~10 | 转换函数注册 |
| Copy 函数 | ~25 | Copy 函数注册 |
| Appender | ~25 | 批量数据插入 |
| 文件系统 | ~15 | 文件操作 |
| 其他 | ~50 | Arrow、日志、Catalog 等 |

**总计约 400+ 个 API 函数**

## 小结

本章详细分析了 DuckDB 的 C API 扩展接口：

1. **函数指针表设计**：所有 API 通过 `duckdb_ext_api_v1` 结构体提供，确保 ABI 稳定性
2. **全面的 API 覆盖**：涵盖数据库操作、查询执行、类型系统、函数注册等所有核心功能
3. **版本管理**：支持稳定版和不稳定版两种 API 类型
4. **资源管理**：所有创建的资源都需要显式销毁
5. **错误处理**：统一的错误报告机制

C API 使得 DuckDB 扩展开发可以使用任何支持 C FFI 的语言，大大扩展了扩展开发的可能性。
