# DuckDB Catalog 系统深度解析 - 第四章：依赖管理系统

本章深入分析 DuckDB 的依赖管理系统，包括 DependencyManager 的实现、Subject/Dependent 关系模型、级联删除机制以及依赖验证。

## 4.1 依赖管理概述

### 4.1.1 什么是依赖

在数据库中，对象之间存在各种依赖关系：

- **视图依赖于表**：视图的 SELECT 查询引用了表
- **索引依赖于表**：索引建立在表的列上
- **外键依赖于主键表**：外键约束引用另一个表的主键
- **函数依赖于类型**：函数参数或返回值使用了用户定义类型
- **对象依赖于 Schema**：所有对象都属于某个 Schema

### 4.1.2 DependencyManager 核心职责

```cpp
// src/include/duckdb/catalog/dependency_manager.hpp

class DependencyManager {
public:
    DependencyManager(DuckCatalog &catalog);

    //===--------------------------------------------------------------------===//
    // 依赖注册
    //===--------------------------------------------------------------------===//
    void AddObject(CatalogTransaction transaction, CatalogEntry &object,
                   const LogicalDependencyList &dependencies);

    void AddOwnership(CatalogTransaction transaction,
                      CatalogEntry &owner, CatalogEntry &entry);

    //===--------------------------------------------------------------------===//
    // 删除检查
    //===--------------------------------------------------------------------===//
    catalog_entry_set_t CheckDropDependencies(
        CatalogTransaction transaction, CatalogEntry &object, bool cascade);

    void DropObject(CatalogTransaction transaction,
                    CatalogEntry &object, bool cascade);

    //===--------------------------------------------------------------------===//
    // 修改处理
    //===--------------------------------------------------------------------===//
    void AlterObject(CatalogTransaction transaction,
                     CatalogEntry &old_obj, CatalogEntry &new_obj,
                     AlterInfo &alter_info);

    //===--------------------------------------------------------------------===//
    // 提交验证
    //===--------------------------------------------------------------------===//
    void VerifyExistence(CatalogTransaction transaction, DependencyEntry &object);
    void VerifyCommitDrop(CatalogTransaction transaction,
                          transaction_t start_time, CatalogEntry &object);

    //===--------------------------------------------------------------------===//
    // 导出排序
    //===--------------------------------------------------------------------===//
    void ReorderEntries(catalog_entry_vector_t &entries);

private:
    DuckCatalog &catalog;
    CatalogSet subjects;      // 被依赖对象集合
    CatalogSet dependents;    // 依赖方集合
};
```

---

## 4.2 Subject/Dependent 关系模型

### 4.2.1 概念定义

```
┌─────────────────────────────────────────────────────────────────┐
│                    Subject/Dependent 模型                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Dependent                        Subject                        │
│  (依赖方)                         (被依赖方)                      │
│     │                                │                           │
│     │         依赖于                  │                           │
│     └────────────────────────────────┘                           │
│                                                                  │
│  示例:                                                           │
│  • View V 依赖于 Table T                                         │
│    - V 是 Dependent（依赖方）                                    │
│    - T 是 Subject（被依赖方）                                    │
│                                                                  │
│  • Index I 依赖于 Table T                                        │
│    - I 是 Dependent                                              │
│    - T 是 Subject                                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2.2 双向存储结构

DependencyManager 使用两个 CatalogSet 分别从两个方向存储依赖关系：

```
┌─────────────────────────────────────────────────────────────────┐
│                    DependencyManager                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  subjects CatalogSet:                                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 按 Dependent 分组，存储其依赖的 Subject 列表                  ││
│  │                                                              ││
│  │ Key: MangleName(View_V)                                      ││
│  │ Entries:                                                     ││
│  │   - DependencySubjectEntry(V → T)                            ││
│  │   - DependencySubjectEntry(V → Schema)                       ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
│  dependents CatalogSet:                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ 按 Subject 分组，存储依赖它的 Dependent 列表                  ││
│  │                                                              ││
│  │ Key: MangleName(Table_T)                                     ││
│  │ Entries:                                                     ││
│  │   - DependencyDependentEntry(T ← V)                          ││
│  │   - DependencyDependentEntry(T ← Index_I)                    ││
│  └─────────────────────────────────────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2.3 名称编码

使用 MangledEntryName 唯一标识条目：

```cpp
// 格式: Type + '\0' + Schema + '\0' + Name
MangledEntryName::MangledEntryName(const CatalogEntryInfo &info) {
    auto &type = info.type;
    auto &schema = info.schema;
    auto &name = info.name;

    // 使用 null 字符分隔，确保唯一性
    this->name = CatalogTypeToString(type) + '\0' + schema + '\0' + name;
}

// 示例:
// Table main.customers → "TABLE_ENTRY\0main\0customers"
// View main.active_customers → "VIEW_ENTRY\0main\0active_customers"
```

---

## 4.3 依赖类型与标志

### 4.3.1 DependencyDependentFlags

```cpp
// 依赖方的标志
class DependencyDependentFlags {
public:
    bool IsBlocking() const;   // 是否阻止删除（需要 CASCADE）
    bool IsOwnedBy() const;    // 是否被 Subject 拥有

    DependencyDependentFlags &SetBlocking();
    DependencyDependentFlags &SetOwnedBy();
};
```

### 4.3.2 DependencySubjectFlags

```cpp
// 被依赖方的标志
class DependencySubjectFlags {
public:
    bool IsOwnership() const;  // 是否拥有 Dependent

    DependencySubjectFlags &SetOwnership();
};
```

### 4.3.3 依赖类型总结

| Dependent 标志 | Subject 标志 | 含义 | 示例 |
|----------------|--------------|------|------|
| Blocking | - | 普通依赖，删除需要 CASCADE | 视图依赖于表 |
| - | - | 非阻塞依赖，随 Subject 自动删除 | 索引依赖于表 |
| OwnedBy | Ownership | 所有权关系，Subject 删除时 Dependent 也删除 | 序列被表列拥有 |

---

## 4.4 依赖注册

### 4.4.1 AddObject

当创建新对象时注册依赖：

```cpp
void DependencyManager::AddObject(
    CatalogTransaction transaction,
    CatalogEntry &object,
    const LogicalDependencyList &dependencies) {

    if (IsSystemEntry(object)) {
        return;  // 系统条目不需要依赖管理
    }
    CreateDependencies(transaction, object, dependencies);
}
```

### 4.4.2 CreateDependencies

```cpp
void DependencyManager::CreateDependencies(
    CatalogTransaction transaction,
    const CatalogEntry &object,
    const LogicalDependencyList &dependencies) {

    // 设置依赖标志
    DependencyDependentFlags dependency_flags;
    if (object.type != CatalogType::INDEX_ENTRY) {
        // 索引不需要 CASCADE，删除表时自动删除
        // 其他对象默认为阻塞依赖
        dependency_flags.SetBlocking();
    }

    const auto object_info = GetLookupProperties(object);

    // 验证所有依赖在同一个 Catalog 中
    for (auto &dependency : dependencies.Set()) {
        if (dependency.catalog != object.ParentCatalog().GetName()) {
            throw DependencyException(
                "Cross catalog dependencies are not supported.");
        }
    }

    // 为每个依赖创建双向链接
    for (auto &dependency : dependencies.Set()) {
        DependencyInfo info {
            /*dependent = */ DependencyDependent{
                GetLookupProperties(object), dependency_flags},
            /*subject = */ DependencySubject{
                dependency.entry, DependencySubjectFlags()}
        };
        CreateDependency(transaction, info);
    }
}
```

### 4.4.3 CreateDependency

创建双向依赖链接：

```cpp
void DependencyManager::CreateDependency(
    CatalogTransaction transaction, DependencyInfo &info) {

    DependencyCatalogSet subjects(Subjects(), info.dependent.entry);
    DependencyCatalogSet dependents(Dependents(), info.subject.entry);

    auto subject_mangled = MangleName(info.subject.entry);
    auto dependent_mangled = MangleName(info.dependent.entry);

    // 检查并继承现有标志
    auto existing_subject = subjects.GetEntry(transaction, subject_mangled);
    auto existing_dependent = dependents.GetEntry(transaction, dependent_mangled);

    if (existing_subject) {
        auto &existing = existing_subject->Cast<DependencyEntry>();
        info.subject.flags.Apply(existing.Subject().flags);
        subjects.DropEntry(transaction, subject_mangled, false, false);
    }
    if (existing_dependent) {
        auto &existing = existing_dependent->Cast<DependencyEntry>();
        info.dependent.flags.Apply(existing.Dependent().flags);
        dependents.DropEntry(transaction, dependent_mangled, false, false);
    }

    // 在 dependents 集合中创建条目（记录谁依赖于 Subject）
    CreateDependent(transaction, info);
    // 在 subjects 集合中创建条目（记录 Dependent 依赖于谁）
    CreateSubject(transaction, info);
}
```

### 4.4.4 依赖注册示意图

```
创建视图 V 依赖于表 T:

1. CreateDependency(V → T)
         │
         ▼
2. CreateSubject:
   subjects[V].add(DependencySubjectEntry{
       subject: T,
       flags: {}
   })
         │
         ▼
3. CreateDependent:
   dependents[T].add(DependencyDependentEntry{
       dependent: V,
       flags: {Blocking}
   })

结果:
┌─────────────────┐         ┌─────────────────┐
│    subjects     │         │   dependents    │
├─────────────────┤         ├─────────────────┤
│ V:              │         │ T:              │
│   └── T         │         │   └── V         │
│                 │         │   (Blocking)    │
└─────────────────┘         └─────────────────┘
```

---

## 4.5 所有权依赖

### 4.5.1 AddOwnership

用于建立所有权关系（如序列被表列拥有）：

```cpp
void DependencyManager::AddOwnership(
    CatalogTransaction transaction,
    CatalogEntry &owner,
    CatalogEntry &entry) {

    if (IsSystemEntry(entry) || IsSystemEntry(owner)) {
        return;
    }

    const auto owner_info = GetLookupProperties(owner);
    const auto entry_info = GetLookupProperties(entry);

    // 检查 owner 是否已被其他对象拥有
    ScanDependents(transaction, owner_info, [&](DependencyEntry &dep) {
        if (dep.Dependent().flags.IsOwnedBy()) {
            throw DependencyException(
                "%s can not become the owner, it is already owned by %s",
                owner.name, dep.EntryInfo().name);
        }
    });

    // 检查 entry 是否已经拥有其他对象
    ScanSubjects(transaction, entry_info, [&](DependencyEntry &other) {
        if (other.Dependent().flags.IsOwnedBy()) {
            throw DependencyException(
                "%s already owns %s. Cannot have circular dependencies",
                entry.name, other.EntryInfo().name);
        }
    });

    // 检查 entry 是否已被其他对象拥有
    ScanDependents(transaction, entry_info, [&](DependencyEntry &other) {
        if (other.Subject().flags.IsOwnership()) {
            if (LookupEntry(transaction, other) != &owner) {
                throw DependencyException(
                    "%s is already owned by %s",
                    entry.name, other.EntryInfo().name);
            }
        }
    });

    // 创建所有权依赖
    DependencyInfo info {
        /*dependent = */ DependencyDependent{
            GetLookupProperties(owner),
            DependencyDependentFlags().SetOwnedBy()},
        /*subject = */ DependencySubject{
            GetLookupProperties(entry),
            DependencySubjectFlags().SetOwnership()}
    };
    CreateDependency(transaction, info);
}
```

### 4.5.2 所有权关系示例

```sql
-- 创建带 IDENTITY 列的表（自动创建序列）
CREATE TABLE orders (
    id INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    customer_id INTEGER
);

-- 序列被 orders.id 列拥有
-- 删除表时，序列也会被自动删除
```

```
所有权依赖结构:

Table orders                     Sequence orders_id_seq
     │                                │
     │ owns                           │
     └───────────────────────────────┘

删除 orders 时:
1. 检查 orders 的 subjects（它拥有什么）
2. 发现 orders_id_seq 有 Ownership 标志
3. 自动删除 orders_id_seq
```

---

## 4.6 删除时的依赖检查

### 4.6.1 CheckDropDependencies

```cpp
catalog_entry_set_t DependencyManager::CheckDropDependencies(
    CatalogTransaction transaction,
    CatalogEntry &object,
    bool cascade) {

    if (IsSystemEntry(object)) {
        return catalog_entry_set_t();
    }

    catalog_entry_set_t to_drop;          // 需要删除的对象
    catalog_entry_set_t blocking_dependents;  // 阻塞删除的对象

    auto info = GetLookupProperties(object);

    // 1. 扫描依赖于 object 的所有对象
    ScanDependents(transaction, info, [&](DependencyEntry &dep) {
        auto entry = LookupEntry(transaction, dep);
        if (!entry) {
            return;
        }

        if (!CascadeDrop(cascade, dep.Dependent().flags)) {
            // 非级联且有阻塞依赖
            blocking_dependents.insert(*entry);
        } else {
            // 可以级联删除
            to_drop.insert(*entry);
        }
    });

    // 2. 如果有阻塞依赖且非级联，抛出错误
    if (!blocking_dependents.empty()) {
        string error_string = StringUtil::Format(
            "Cannot drop entry \"%s\" because there are entries that depend on it.\n",
            object.name);
        error_string += CollectDependents(transaction, blocking_dependents, info);
        error_string += "Use DROP...CASCADE to drop all dependents.";
        throw DependencyException(error_string);
    }

    // 3. 检查 object 拥有的对象
    ScanSubjects(transaction, info, [&](DependencyEntry &dep) {
        auto flags = dep.Subject().flags;
        if (flags.IsOwnership()) {
            // object 拥有这个对象，也需要删除
            auto entry = LookupEntry(transaction, dep);
            to_drop.insert(*entry);
        }
    });

    return to_drop;
}
```

### 4.6.2 CascadeDrop 判断

```cpp
static bool CascadeDrop(bool cascade, const DependencyDependentFlags &flags) {
    if (cascade) {
        return true;  // 显式指定级联
    }
    if (flags.IsOwnedBy()) {
        // 被拥有的对象不能在没有 CASCADE 时删除拥有者
        return false;
    }
    return !flags.IsBlocking();  // 非阻塞依赖可以自动删除
}
```

### 4.6.3 删除流程图

```
DROP TABLE T:

         ┌──────────────────────────────────────────┐
         │ 1. CheckDropDependencies(T, cascade)     │
         └─────────────────┬────────────────────────┘
                           │
                           ▼
         ┌──────────────────────────────────────────┐
         │ 2. ScanDependents: 查找依赖于 T 的对象    │
         │    - View V (Blocking)                   │
         │    - Index I (非 Blocking)               │
         └─────────────────┬────────────────────────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         cascade=false              cascade=true
              │                         │
              ▼                         ▼
    ┌─────────────────────┐   ┌─────────────────────┐
    │ V 是 Blocking       │   │ 将 V, I 加入删除列表│
    │ 抛出 DependencyEx   │   │                     │
    └─────────────────────┘   └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │ 3. ScanSubjects:    │
                              │    查找 T 拥有的对象│
                              │    - Sequence S     │
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │ 4. 将 S 加入删除列表│
                              └──────────┬──────────┘
                                         │
                                         ▼
                              ┌─────────────────────┐
                              │ 5. 返回 {V, I, S}   │
                              └─────────────────────┘
```

### 4.6.4 DropObject

执行实际删除：

```cpp
void DependencyManager::DropObject(
    CatalogTransaction transaction,
    CatalogEntry &object,
    bool cascade) {

    if (IsSystemEntry(object)) {
        return;
    }

    // 1. 检查并收集需要删除的对象
    auto to_drop = CheckDropDependencies(transaction, object, cascade);

    // 2. 清理 object 的所有依赖记录
    CleanupDependencies(transaction, object);

    // 3. 递归删除收集到的对象
    for (auto &entry : to_drop) {
        auto set = entry.get().set;
        D_ASSERT(set);
        set->DropEntry(transaction, entry.get().name, cascade);
    }
}
```

---

## 4.7 修改时的依赖处理

### 4.7.1 AlterObject

处理对象修改时的依赖更新：

```cpp
void DependencyManager::AlterObject(
    CatalogTransaction transaction,
    CatalogEntry &old_obj,
    CatalogEntry &new_obj,
    AlterInfo &alter_info) {

    if (IsSystemEntry(new_obj)) {
        return;
    }

    const auto old_info = GetLookupProperties(old_obj);
    const auto new_info = GetLookupProperties(new_obj);

    vector<DependencyInfo> dependencies;

    // 1. 检查依赖于此对象的其他对象
    ScanDependents(transaction, old_info, [&](DependencyEntry &dep) {
        bool disallow_alter = true;

        // 某些修改是允许的
        switch (alter_info.type) {
        case AlterType::ALTER_TABLE: {
            auto &alter_table = alter_info.Cast<AlterTableInfo>();
            switch (alter_table.alter_table_type) {
            case AlterTableType::FOREIGN_KEY_CONSTRAINT:
            case AlterTableType::ADD_COLUMN:
                disallow_alter = false;
                break;
            default:
                break;
            }
            break;
        }
        case AlterType::SET_COLUMN_COMMENT:
        case AlterType::SET_COMMENT:
            disallow_alter = false;
            break;
        default:
            break;
        }

        if (disallow_alter) {
            throw DependencyException(
                "Cannot alter entry \"%s\" because there are entries that depend on it.",
                old_obj.name);
        }

        // 收集需要更新的依赖
        auto dep_info = DependencyInfo::FromDependent(dep);
        dep_info.subject.entry = new_info;
        dependencies.emplace_back(dep_info);
    });

    // 2. 收集此对象依赖的其他对象
    ScanSubjects(transaction, old_info, [&](DependencyEntry &dep) {
        auto entry = LookupEntry(transaction, dep);
        if (!entry) {
            return;
        }
        auto dep_info = DependencyInfo::FromSubject(dep);
        dep_info.dependent.entry = new_info;
        dependencies.emplace_back(dep_info);
    });

    // 3. 如果名称改变，清理旧的依赖链接
    if (!StringUtil::CIEquals(old_obj.name, new_obj.name)) {
        CleanupDependencies(transaction, old_obj);
    }

    // 4. 重新创建依赖
    for (auto &dep : dependencies) {
        CreateDependency(transaction, dep);
    }
}
```

---

## 4.8 提交时验证

### 4.8.1 VerifyExistence

在提交创建依赖时，验证被依赖对象仍然存在：

```cpp
void DependencyManager::VerifyExistence(
    CatalogTransaction transaction,
    DependencyEntry &object) {

    auto &subject = object.Subject();
    CatalogEntryInfo info;

    if (subject.flags.IsOwnership()) {
        info = object.SourceInfo();
    } else {
        info = object.EntryInfo();
    }

    auto &type = info.type;
    auto &schema = info.schema;
    auto &name = info.name;

    // 查找 Schema
    auto &duck_catalog = catalog.Cast<DuckCatalog>();
    auto &schema_catalog_set = duck_catalog.GetSchemaCatalogSet();
    auto lookup_result = schema_catalog_set.GetEntryDetailed(transaction, schema);

    // 查找条目
    if (type != CatalogType::SCHEMA_ENTRY && lookup_result.result) {
        auto &schema_entry = lookup_result.result->Cast<SchemaCatalogEntry>();
        EntryLookupInfo lookup_info(type, name);
        lookup_result = schema_entry.LookupEntryDetailed(transaction, lookup_info);
    }

    // 如果被依赖对象已删除，拒绝提交
    if (lookup_result.reason == CatalogSet::EntryLookup::FailureReason::DELETED) {
        throw DependencyException(
            "Could not commit creation of dependency, subject \"%s\" has been deleted",
            object.SourceInfo().name);
    }
}
```

### 4.8.2 VerifyCommitDrop

在提交删除时，验证没有新的依赖产生：

```cpp
void DependencyManager::VerifyCommitDrop(
    CatalogTransaction transaction,
    transaction_t start_time,
    CatalogEntry &object) {

    if (IsSystemEntry(object)) {
        return;
    }

    auto info = GetLookupProperties(object);

    // 检查是否有新的依赖在事务开始后创建
    ScanDependents(transaction, info, [&](DependencyEntry &dep) {
        auto dep_committed_at = dep.timestamp.load();
        if (dep_committed_at > start_time) {
            // 事务开始后有新的依赖产生
            throw DependencyException(
                "Could not commit DROP of \"%s\" because a dependency "
                "was created after the transaction started",
                object.name);
        }
    });

    // 检查拥有的对象
    ScanSubjects(transaction, info, [&](DependencyEntry &dep) {
        if (!dep.Dependent().flags.IsOwnedBy()) {
            return;
        }
        auto dep_committed_at = dep.timestamp.load();
        if (dep_committed_at > start_time) {
            throw DependencyException(
                "Could not commit DROP of \"%s\" because a dependency "
                "was created after the transaction started",
                object.name);
        }
    });
}
```

---

## 4.9 导出排序

### 4.9.1 ReorderEntries

确保导出时被依赖的对象先于依赖它的对象：

```cpp
void DependencyManager::ReorderEntries(catalog_entry_vector_t &entries) {
    // 使用最高的 commit ID，可以读取所有已提交数据
    CatalogTransaction transaction(
        catalog.GetDatabase(),
        TRANSACTION_ID_START - 1,
        TRANSACTION_ID_START - 1);
    ReorderEntries(entries, transaction);
}

void DependencyManager::ReorderEntries(
    catalog_entry_vector_t &entries,
    CatalogTransaction transaction) {

    catalog_entry_vector_t reordered;
    catalog_entry_set_t visited;

    for (auto &entry : entries) {
        ReorderEntry(transaction, entry, visited, reordered);
    }

    D_ASSERT(entries.size() == reordered.size());
    entries = reordered;
}

void DependencyManager::ReorderEntry(
    CatalogTransaction transaction,
    CatalogEntry &entry,
    catalog_entry_set_t &visited,
    catalog_entry_vector_t &order) {

    auto &catalog_entry = *LookupEntry(transaction, entry);

    if (visited.count(catalog_entry)) {
        return;  // 已处理
    }

    // 先处理此条目依赖的对象
    catalog_entry_vector_t dependents;
    auto info = GetLookupProperties(entry);
    ScanSubjects(transaction, info, [&](DependencyEntry &dep) {
        dependents.push_back(dep);
    });

    for (auto &dep : dependents) {
        ReorderEntry(transaction, dep, visited, order);
    }

    // 然后添加此条目
    visited.insert(catalog_entry);
    order.push_back(catalog_entry);
}
```

### 4.9.2 排序示例

```
原始顺序:
[View_V, Table_T, Index_I, Schema_S]

依赖关系:
- V 依赖于 T, S
- T 依赖于 S
- I 依赖于 T, S

排序后:
[Schema_S, Table_T, Index_I, View_V]

导出顺序确保:
1. Schema_S 先创建
2. Table_T 在 Schema 之后创建
3. Index_I 和 View_V 在 Table 之后创建
```

---

## 4.10 本章小结

本章详细分析了 DuckDB 的依赖管理系统：

1. **Subject/Dependent 模型**：
   - Subject：被依赖的对象
   - Dependent：依赖方对象
   - 双向存储：subjects 和 dependents 两个 CatalogSet

2. **依赖类型**：
   - Blocking：阻塞依赖，删除需要 CASCADE
   - 非 Blocking：随 Subject 自动删除（如索引）
   - Ownership：所有权关系，Subject 删除时 Dependent 也删除

3. **依赖注册**：
   - `AddObject()`：创建对象时注册依赖
   - `AddOwnership()`：建立所有权关系
   - 使用 MangledEntryName 唯一标识条目

4. **删除检查**：
   - `CheckDropDependencies()`：检查阻塞依赖
   - `CascadeDrop()`：判断是否可以级联删除
   - 级联删除自动删除依赖对象和拥有的对象

5. **修改处理**：
   - `AlterObject()`：更新依赖链接
   - 某些修改被禁止（如有依赖时）

6. **提交验证**：
   - `VerifyExistence()`：验证被依赖对象存在
   - `VerifyCommitDrop()`：验证没有新依赖产生

7. **导出排序**：
   - `ReorderEntries()`：确保正确的创建顺序

---

## 4.11 核心源文件索引

| 文件 | 说明 |
|------|------|
| `src/include/duckdb/catalog/dependency_manager.hpp` | DependencyManager 定义 |
| `src/catalog/dependency_manager.cpp` | DependencyManager 实现 |
| `src/catalog/catalog_entry/dependency/dependency_entry.cpp` | DependencyEntry 基类 |
| `src/catalog/catalog_entry/dependency/dependency_subject_entry.cpp` | Subject 条目 |
| `src/catalog/catalog_entry/dependency/dependency_dependent_entry.cpp` | Dependent 条目 |
| `src/catalog/dependency_list.cpp` | LogicalDependencyList 实现 |
| `src/catalog/dependency_catalog_set.cpp` | DependencyCatalogSet 封装 |
