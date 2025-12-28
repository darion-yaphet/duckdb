# 第十章：扩展开发实战

## 10.1 概述

本章将从实际开发角度出发，介绍如何创建、构建、测试和发布一个 DuckDB 扩展。我们将分别演示 C++ 扩展和 C API 扩展的开发流程，并总结开发过程中的最佳实践。

```
扩展开发流程
┌─────────────────────────────────────────────────────────────────┐
│  1. 项目初始化  →  2. 功能实现  →  3. 构建配置  →  4. 测试验证  │
│        ↓                               ↓              ↓        │
│    CMakeLists.txt              静态/动态构建      sqllogictest  │
│    源文件结构                   依赖管理          单元测试       │
└─────────────────────────────────────────────────────────────────┘
                                 ↓
┌─────────────────────────────────────────────────────────────────┐
│  5. 调试优化  →  6. 打包签名  →  7. 发布分发                    │
│        ↓               ↓              ↓                         │
│    GDB/LLDB      元数据生成      仓库提交                       │
│    日志输出      RSA 签名        版本管理                       │
└─────────────────────────────────────────────────────────────────┘
```

## 10.2 项目结构

### 10.2.1 C++ 扩展项目结构

标准的 C++ 扩展项目布局：

```
my_extension/
├── CMakeLists.txt                 # 主构建配置
├── include/
│   └── my_extension.hpp           # 扩展类声明
├── src/
│   ├── my_extension.cpp           # 扩展入口
│   ├── functions/
│   │   ├── scalar/                # 标量函数
│   │   │   └── my_scalar.cpp
│   │   ├── aggregate/             # 聚合函数
│   │   │   └── my_aggregate.cpp
│   │   └── table/                 # 表函数
│   │       └── my_table_func.cpp
│   └── types/                     # 自定义类型
│       └── my_type.cpp
└── test/
    └── sql/
        └── my_extension.test      # sqllogictest 测试
```

### 10.2.2 C API 扩展项目结构

C API 扩展更加简洁：

```
my_capi_extension/
├── CMakeLists.txt                 # 构建配置
├── include/
│   └── my_functions.h             # 函数声明
├── src/
│   ├── my_extension.c             # 扩展入口
│   └── my_functions.c             # 函数实现
└── test/
    └── sql/
        └── my_capi_extension.test
```

## 10.3 C++ 扩展开发

### 10.3.1 扩展类实现

首先定义扩展类：

```cpp
// include/my_extension.hpp
#pragma once

#include "duckdb.hpp"

namespace duckdb {

class MyExtension : public Extension {
public:
    void Load(ExtensionLoader &loader) override;
    std::string Name() override;
    std::string Version() const override;
};

} // namespace duckdb
```

实现扩展入口：

```cpp
// src/my_extension.cpp
#include "my_extension.hpp"
#include "duckdb/main/extension/extension_loader.hpp"

namespace duckdb {

// 内部加载函数
static void LoadInternal(ExtensionLoader &loader) {
    // 1. 注册标量函数
    ScalarFunction hello_fun(
        "hello_world",
        {},                              // 无参数
        LogicalType::VARCHAR,            // 返回 VARCHAR
        HelloWorldFunction               // 函数指针
    );
    loader.RegisterFunction(hello_fun);

    // 2. 注册表函数
    TableFunction my_table_fun(
        "my_table_function",
        {LogicalType::VARCHAR},          // 参数类型
        MyTableFunction,                 // 主函数
        MyTableBind,                     // 绑定函数
        MyTableInit                      // 初始化函数
    );
    loader.RegisterFunction(my_table_fun);

    // 3. 注册自定义类型（可选）
    auto my_type = LogicalType::USER("MY_TYPE");
    loader.RegisterType("MY_TYPE", std::move(my_type));

    // 4. 注册配置选项（可选）
    auto &config = DBConfig::GetConfig(loader.GetDatabaseInstance());
    config.AddExtensionOption(
        "my_extension.debug",
        "Enable debug mode",
        LogicalType::BOOLEAN,
        Value(false)
    );
}

void MyExtension::Load(ExtensionLoader &loader) {
    LoadInternal(loader);
}

std::string MyExtension::Name() {
    return "my_extension";
}

std::string MyExtension::Version() const {
#ifdef EXT_VERSION_MY_EXTENSION
    return EXT_VERSION_MY_EXTENSION;
#else
    return "";
#endif
}

} // namespace duckdb

// C++ 入口点宏
extern "C" {
DUCKDB_CPP_EXTENSION_ENTRY(my_extension, loader) {
    duckdb::LoadInternal(loader);
}
}
```

### 10.3.2 标量函数实现

实现一个简单的标量函数：

```cpp
// src/functions/scalar/my_scalar.cpp
#include "duckdb.hpp"
#include "duckdb/function/scalar_function.hpp"

namespace duckdb {

// 函数实现：返回 "Hello, World!"
static void HelloWorldFunction(
    DataChunk &args,
    ExpressionState &state,
    Vector &result
) {
    // 设置常量结果
    result.SetValue(0, Value("Hello, World!"));
    result.SetVectorType(VectorType::CONSTANT_VECTOR);
}

// 带参数的函数示例
static void GreetFunction(
    DataChunk &args,
    ExpressionState &state,
    Vector &result
) {
    auto &name_vector = args.data[0];
    auto count = args.size();

    UnaryExecutor::Execute<string_t, string_t>(
        name_vector, result, count,
        [&](string_t name) {
            return StringVector::AddString(
                result,
                "Hello, " + name.GetString() + "!"
            );
        }
    );
}

// 注册函数集合
ScalarFunctionSet GetGreetFunctions() {
    ScalarFunctionSet set("greet");

    // 无参数版本
    set.AddFunction(ScalarFunction(
        {},
        LogicalType::VARCHAR,
        HelloWorldFunction
    ));

    // 带 VARCHAR 参数版本
    set.AddFunction(ScalarFunction(
        {LogicalType::VARCHAR},
        LogicalType::VARCHAR,
        GreetFunction
    ));

    return set;
}

} // namespace duckdb
```

### 10.3.3 表函数实现

实现数据生成表函数：

```cpp
// src/functions/table/my_table_func.cpp
#include "duckdb.hpp"
#include "duckdb/function/table_function.hpp"

namespace duckdb {

// 绑定数据：存储函数参数
struct MyTableBindData : public TableFunctionData {
    idx_t count;
    string prefix;

    MyTableBindData(idx_t count, string prefix)
        : count(count), prefix(std::move(prefix)) {}
};

// 全局状态：跨线程共享
struct MyTableGlobalState : public GlobalTableFunctionState {
    idx_t current_row = 0;
    mutex lock;
};

// 绑定函数：解析参数
static unique_ptr<FunctionData> MyTableBind(
    ClientContext &context,
    TableFunctionBindInput &input,
    vector<LogicalType> &return_types,
    vector<string> &names
) {
    // 获取参数
    idx_t count = input.inputs[0].GetValue<int64_t>();
    string prefix = "row_";
    if (input.inputs.size() > 1) {
        prefix = input.inputs[1].GetValue<string>();
    }

    // 定义输出列
    return_types.push_back(LogicalType::BIGINT);
    names.push_back("id");

    return_types.push_back(LogicalType::VARCHAR);
    names.push_back("name");

    return make_uniq<MyTableBindData>(count, prefix);
}

// 初始化函数
static unique_ptr<GlobalTableFunctionState> MyTableInit(
    ClientContext &context,
    TableFunctionInitInput &input
) {
    return make_uniq<MyTableGlobalState>();
}

// 主函数：生成数据
static void MyTableFunction(
    ClientContext &context,
    TableFunctionInput &data,
    DataChunk &output
) {
    auto &bind_data = data.bind_data->Cast<MyTableBindData>();
    auto &state = data.global_state->Cast<MyTableGlobalState>();

    lock_guard<mutex> guard(state.lock);

    idx_t remaining = bind_data.count - state.current_row;
    idx_t to_output = MinValue<idx_t>(remaining, STANDARD_VECTOR_SIZE);

    if (to_output == 0) {
        return;  // 数据生成完毕
    }

    // 填充数据
    auto id_data = FlatVector::GetData<int64_t>(output.data[0]);
    for (idx_t i = 0; i < to_output; i++) {
        id_data[i] = state.current_row + i;
    }

    for (idx_t i = 0; i < to_output; i++) {
        string name = bind_data.prefix + std::to_string(state.current_row + i);
        output.data[1].SetValue(i, Value(name));
    }

    state.current_row += to_output;
    output.SetCardinality(to_output);
}

// 获取表函数
TableFunction GetMyTableFunction() {
    TableFunction func(
        "generate_rows",
        {LogicalType::BIGINT},           // 必需参数
        MyTableFunction,
        MyTableBind,
        MyTableInit
    );

    // 可选参数
    func.named_parameters["prefix"] = LogicalType::VARCHAR;

    return func;
}

} // namespace duckdb
```

### 10.3.4 聚合函数实现

实现自定义聚合函数：

```cpp
// src/functions/aggregate/my_aggregate.cpp
#include "duckdb.hpp"
#include "duckdb/function/aggregate_function.hpp"

namespace duckdb {

// 聚合状态
struct MyAvgState {
    double sum = 0;
    int64_t count = 0;
};

// 状态初始化
static void MyAvgInit(AggregateInputData &input, data_ptr_t state) {
    new (state) MyAvgState();
}

// 状态更新（添加值）
static void MyAvgUpdate(
    Vector inputs[], AggregateInputData &input,
    idx_t input_count, Vector &state_vector, idx_t count
) {
    D_ASSERT(input_count == 1);
    auto &input_vector = inputs[0];

    UnifiedVectorFormat input_data;
    input_vector.ToUnifiedFormat(count, input_data);

    auto states = FlatVector::GetData<MyAvgState *>(state_vector);
    auto values = UnifiedVectorFormat::GetData<double>(input_data);

    for (idx_t i = 0; i < count; i++) {
        auto state = states[i];
        if (input_data.validity.RowIsValid(input_data.sel->get_index(i))) {
            state->sum += values[input_data.sel->get_index(i)];
            state->count++;
        }
    }
}

// 状态合并（并行聚合）
static void MyAvgCombine(
    Vector &state_vector, Vector &combined,
    AggregateInputData &input, idx_t count
) {
    auto states = FlatVector::GetData<MyAvgState *>(state_vector);
    auto combined_states = FlatVector::GetData<MyAvgState *>(combined);

    for (idx_t i = 0; i < count; i++) {
        auto &source = *states[i];
        auto &target = *combined_states[i];
        target.sum += source.sum;
        target.count += source.count;
    }
}

// 最终结果
static void MyAvgFinalize(
    Vector &state_vector, AggregateInputData &input,
    Vector &result, idx_t count, idx_t offset
) {
    auto states = FlatVector::GetData<MyAvgState *>(state_vector);
    auto result_data = FlatVector::GetData<double>(result);
    auto &result_validity = FlatVector::Validity(result);

    for (idx_t i = 0; i < count; i++) {
        auto &state = *states[i];
        if (state.count == 0) {
            result_validity.SetInvalid(offset + i);
        } else {
            result_data[offset + i] = state.sum / state.count;
        }
    }
}

// 状态销毁
static void MyAvgDestroy(Vector &state_vector, AggregateInputData &input,
                          idx_t count) {
    auto states = FlatVector::GetData<MyAvgState *>(state_vector);
    for (idx_t i = 0; i < count; i++) {
        states[i]->~MyAvgState();
    }
}

// 获取聚合函数
AggregateFunction GetMyAvgFunction() {
    return AggregateFunction(
        "my_avg",
        {LogicalType::DOUBLE},           // 输入类型
        LogicalType::DOUBLE,             // 返回类型
        AggregateFunction::StateSize<MyAvgState>,
        MyAvgInit,
        MyAvgUpdate,
        MyAvgCombine,
        MyAvgFinalize,
        nullptr,                         // simple_update
        nullptr,                         // bind
        MyAvgDestroy
    );
}

} // namespace duckdb
```

## 10.4 C API 扩展开发

### 10.4.1 扩展入口

C API 扩展使用不同的入口宏：

```c
// src/my_extension.c
#include "my_functions.h"
#include "duckdb_extension.h"

DUCKDB_EXTENSION_ENTRYPOINT(
    duckdb_connection connection,
    duckdb_extension_info info,
    duckdb_extension_access *access
) {
    // 注册函数
    RegisterAddNumbersFunction(connection);
    RegisterMyTableFunction(connection);

    // 可选：使用不稳定 API
#ifdef DUCKDB_EXTENSION_API_VERSION_UNSTABLE
    // 测试查询
    duckdb_result result;
    if (duckdb_query(connection, "SELECT 1", &result) != DuckDBSuccess) {
        access->set_error(info, "Query failed");
        return false;
    }
    duckdb_destroy_result(&result);
#endif

    return true;  // 初始化成功
}
```

### 10.4.2 标量函数实现（C API）

使用 C API 实现标量函数：

```c
// src/my_functions.c
#include "my_functions.h"
#include "duckdb_extension.h"

DUCKDB_EXTENSION_EXTERN

// 两数相加函数实现
static void AddNumbersTogether(
    duckdb_function_info info,
    duckdb_data_chunk input,
    duckdb_vector output
) {
    // 获取行数
    idx_t input_size = duckdb_data_chunk_get_size(input);

    // 获取输入向量
    duckdb_vector a = duckdb_data_chunk_get_vector(input, 0);
    duckdb_vector b = duckdb_data_chunk_get_vector(input, 1);

    // 获取数据指针
    int64_t *a_data = (int64_t *)duckdb_vector_get_data(a);
    int64_t *b_data = (int64_t *)duckdb_vector_get_data(b);
    int64_t *result_data = (int64_t *)duckdb_vector_get_data(output);

    // 获取有效性掩码
    uint64_t *a_validity = duckdb_vector_get_validity(a);
    uint64_t *b_validity = duckdb_vector_get_validity(b);

    if (a_validity || b_validity) {
        // 存在 NULL 值
        duckdb_vector_ensure_validity_writable(output);
        uint64_t *result_validity = duckdb_vector_get_validity(output);

        for (idx_t row = 0; row < input_size; row++) {
            if (duckdb_validity_row_is_valid(a_validity, row) &&
                duckdb_validity_row_is_valid(b_validity, row)) {
                result_data[row] = a_data[row] + b_data[row];
            } else {
                duckdb_validity_set_row_invalid(result_validity, row);
            }
        }
    } else {
        // 无 NULL 值，直接计算
        for (idx_t row = 0; row < input_size; row++) {
            result_data[row] = a_data[row] + b_data[row];
        }
    }
}

// 注册函数
void RegisterAddNumbersFunction(duckdb_connection connection) {
    // 创建标量函数
    duckdb_scalar_function function = duckdb_create_scalar_function();

    // 设置名称
    duckdb_scalar_function_set_name(function, "add_numbers");

    // 添加参数
    duckdb_logical_type bigint_type =
        duckdb_create_logical_type(DUCKDB_TYPE_BIGINT);
    duckdb_scalar_function_add_parameter(function, bigint_type);
    duckdb_scalar_function_add_parameter(function, bigint_type);

    // 设置返回类型
    duckdb_scalar_function_set_return_type(function, bigint_type);

    // 清理类型
    duckdb_destroy_logical_type(&bigint_type);

    // 设置函数指针
    duckdb_scalar_function_set_function(function, AddNumbersTogether);

    // 注册并清理
    duckdb_register_scalar_function(connection, function);
    duckdb_destroy_scalar_function(&function);
}
```

## 10.5 构建配置

### 10.5.1 C++ 扩展 CMakeLists.txt

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.5...3.29)

project(MyExtension)

# 包含头文件目录
include_directories(include)

# 定义源文件
set(EXTENSION_NAME my_extension)

set(EXTENSION_FILES
    src/my_extension.cpp
    src/functions/scalar/my_scalar.cpp
    src/functions/aggregate/my_aggregate.cpp
    src/functions/table/my_table_func.cpp)

# 构建静态扩展（链接到 DuckDB）
build_static_extension(${EXTENSION_NAME} ${EXTENSION_FILES})

# 构建动态扩展（可加载）
set(PARAMETERS "-warnings")
build_loadable_extension(${EXTENSION_NAME} ${PARAMETERS} ${EXTENSION_FILES})

# 链接依赖库（如需要）
# target_link_libraries(${EXTENSION_NAME}_loadable_extension my_dependency)

# 安装配置
install(
    TARGETS ${EXTENSION_NAME}_extension
    EXPORT "${DUCKDB_EXPORT_SET}"
    LIBRARY DESTINATION "${INSTALL_LIB_DIR}"
    ARCHIVE DESTINATION "${INSTALL_LIB_DIR}")
```

### 10.5.2 C API 扩展 CMakeLists.txt

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 2.8.12...3.29)

project(MyCAPIExtension)

include_directories(include)

set(EXTENSION_NAME my_capi_extension)
set(EXTENSION_FILES
    src/my_extension.c
    src/my_functions.c)

# 选择 API 版本
option(USE_UNSTABLE_C_API
    "Use the unstable C Extension API (tied to exact DuckDB version)"
    FALSE)

if(USE_UNSTABLE_C_API)
    # 不稳定 API：可使用最新功能
    build_loadable_extension_capi_unstable(
        ${EXTENSION_NAME}
        ${EXTENSION_FILES})
else()
    # 稳定 API：指定最低版本
    set(CAPI_MAJOR 1)
    set(CAPI_MINOR 2)
    set(CAPI_PATCH 0)

    build_loadable_extension_capi(
        ${EXTENSION_NAME}
        ${CAPI_MAJOR} ${CAPI_MINOR} ${CAPI_PATCH}
        ${EXTENSION_FILES})
endif()
```

### 10.5.3 与第三方库集成

如果扩展依赖第三方库：

```cmake
# 示例：集成第三方库
set(EXTENSION_FILES
    ${EXTENSION_FILES}
    # 第三方库源码
    ../../third_party/mylib/mylib.cpp)

# 或者链接预编译库
target_link_libraries(
    ${EXTENSION_NAME}_loadable_extension
    mylib_static)

# 包含第三方头文件
include_directories(
    include
    ../../third_party/mylib/include)
```

## 10.6 测试策略

### 10.6.1 sqllogictest 测试

DuckDB 使用 sqllogictest 作为主要测试框架：

```sql
# test/sql/my_extension.test

# 扩展名称
# group [my_extension]

# 加载扩展（如果未静态链接）
require my_extension

# 测试标量函数
query I
SELECT add_numbers(1, 2)
----
3

# 测试 NULL 处理
query I
SELECT add_numbers(1, NULL)
----
NULL

# 测试表函数
query IT
SELECT * FROM generate_rows(3)
----
0	row_0
1	row_1
2	row_2

# 测试带参数的表函数
query IT
SELECT * FROM generate_rows(2, prefix := 'item_')
----
0	item_0
1	item_1

# 测试聚合函数
statement ok
CREATE TABLE test_data AS
SELECT * FROM (VALUES (1.0), (2.0), (3.0)) AS t(value)

query R
SELECT my_avg(value) FROM test_data
----
2.0

# 错误测试
statement error
SELECT add_numbers('not', 'numbers')
```

### 10.6.2 C++ 单元测试

对于复杂逻辑可添加 C++ 测试：

```cpp
// test/unit/my_extension_test.cpp
#include "catch.hpp"
#include "duckdb.hpp"
#include "my_extension.hpp"

TEST_CASE("My extension basic functionality", "[my_extension]") {
    DuckDB db(nullptr);
    Connection con(db);

    // 加载扩展
    con.Query("LOAD my_extension");

    SECTION("add_numbers function") {
        auto result = con.Query("SELECT add_numbers(10, 20)");
        REQUIRE(result->GetValue(0, 0).GetValue<int64_t>() == 30);
    }

    SECTION("null handling") {
        auto result = con.Query("SELECT add_numbers(10, NULL)");
        REQUIRE(result->GetValue(0, 0).IsNull());
    }
}
```

### 10.6.3 运行测试

```bash
# 构建测试
make debug

# 运行所有测试
./build/debug/test/unittest

# 运行特定测试
./build/debug/test/unittest "test/sql/my_extension.test"

# 运行测试组
./build/debug/test/unittest "[my_extension]"
```

## 10.7 调试技巧

### 10.7.1 GDB/LLDB 调试

```bash
# 构建 debug 版本
make debug

# 使用 GDB 调试
gdb --args ./build/debug/duckdb

# 在 GDB 中
(gdb) break duckdb::MyExtension::Load
(gdb) run
(gdb) step
(gdb) print loader
```

### 10.7.2 日志输出

在扩展中添加日志：

```cpp
#include "duckdb/logging/log_manager.hpp"

static void MyFunction(DataChunk &args, ExpressionState &state,
                       Vector &result) {
    // 调试日志
    DUCKDB_LOG(context, LogType::INFO, "MyFunction called with %d rows",
               args.size());

    // ... 函数实现
}
```

### 10.7.3 断言检查

使用 D_ASSERT 进行调试检查：

```cpp
static void MyFunction(DataChunk &args, ExpressionState &state,
                       Vector &result) {
    // 仅在 debug 构建中生效
    D_ASSERT(args.ColumnCount() == 2);
    D_ASSERT(args.data[0].GetType() == LogicalType::BIGINT);

    // ... 函数实现
}
```

## 10.8 发布与分发

### 10.8.1 版本管理

在扩展头文件中定义版本：

```cpp
// 通过构建系统定义
#define EXT_VERSION_MY_EXTENSION "1.0.0"

// 在 CMake 中
add_definitions(-DEXT_VERSION_MY_EXTENSION="${MY_EXTENSION_VERSION}")
```

### 10.8.2 元数据生成

构建时自动生成扩展元数据：

```bash
# 扩展文件结构
my_extension.duckdb_extension
├── 扩展代码（编译后）
└── 256 字节尾部元数据
    ├── 扩展名称
    ├── ABI 类型
    ├── 平台标识
    ├── DuckDB 版本
    └── 签名（如有）
```

### 10.8.3 签名流程

对于发布到仓库的扩展：

```bash
# 1. 生成密钥对（官方维护）
openssl genrsa -out private_key.pem 2048
openssl rsa -in private_key.pem -pubout -out public_key.pem

# 2. 计算扩展哈希
sha256sum my_extension.duckdb_extension

# 3. 签名（使用私钥）
openssl dgst -sha256 -sign private_key.pem \
    -out signature.bin my_extension.duckdb_extension

# 4. 附加签名到扩展文件
# （由构建系统自动完成）
```

### 10.8.4 仓库提交

提交到社区仓库的步骤：

1. **Fork 仓库**：Fork `duckdb/community-extensions`

2. **添加扩展描述**：
```yaml
# extensions/my_extension/description.yml
name: my_extension
version: 1.0.0
repository: https://github.com/myuser/my_extension
description: My awesome DuckDB extension
build_type: cmake
```

3. **配置构建**：
```yaml
# extensions/my_extension/build.yml
cmake_options:
  - -DENABLE_FEATURE=ON
dependencies:
  - libmylib-dev
```

4. **提交 PR**：创建 Pull Request 到主仓库

## 10.9 常见问题排查

### 10.9.1 加载失败

```
Error: Extension "my_extension" could not be loaded:
Invalid magic bytes
```

**原因**：扩展使用了不同版本的 DuckDB 编译
**解决**：使用相同版本重新编译扩展

### 10.9.2 符号未定义

```
Error: undefined symbol: _ZN6duckdb...
```

**原因**：C++ ABI 不匹配或依赖库未链接
**解决**：
- 确保使用相同编译器版本
- 检查 CMakeLists.txt 中的链接配置

### 10.9.3 签名验证失败

```
Error: Extension signature verification failed
```

**原因**：扩展未签名或签名无效
**解决**：
- 开发时使用 `SET allow_unsigned_extensions=true`
- 发布时确保正确签名

### 10.9.4 版本不兼容

```
Error: Extension requires DuckDB version 1.0.0, current version is 0.9.0
```

**原因**：扩展针对较新版本编译
**解决**：
- 升级 DuckDB
- 或使用 C API 稳定版本降低依赖

## 10.10 本章小结

本章介绍了 DuckDB 扩展开发的完整流程：

| 开发阶段 | 关键步骤 | 工具/技术 |
|---------|---------|----------|
| 项目初始化 | 创建目录结构 | CMake |
| 功能实现 | 标量/表/聚合函数 | C++ 或 C API |
| 构建配置 | 静态/动态构建 | build_static/loadable_extension |
| 测试验证 | 功能测试 | sqllogictest |
| 调试优化 | 问题排查 | GDB/LLDB、日志 |
| 发布分发 | 签名发布 | 社区仓库 |

关键要点：
1. **选择合适的 API**：C++ API 性能最佳，C API 兼容性最好
2. **遵循代码规范**：使用 DuckDB 的命名约定和代码风格
3. **完善测试覆盖**：通过 sqllogictest 测试所有功能路径
4. **注意版本兼容**：使用 C API 稳定版本提高兼容性
5. **安全发布**：通过官方渠道签名发布扩展

通过本系列十章的学习，读者应该对 DuckDB 扩展系统有了全面的理解，能够开发、测试和发布自己的扩展，扩展 DuckDB 的功能边界。
