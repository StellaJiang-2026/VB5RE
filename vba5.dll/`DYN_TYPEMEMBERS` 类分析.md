# `DYN_TYPEMEMBERS` 类分析

从函数清单看，**`DYN_TYPEMEMBERS` 是 VB 编译器的“动态类型成员管理与内存布局计算器”**。它负责一个类型（类/UDT/枚举/事件源）的成员集合的分配、布局（偏移/大小/对齐）、校验、名称缓存、以及向 `ITypeLib`/`ITypeInfo` 导出和与 EXE 中的类型信息同步。

名字可理解为 **Dynamic Type Members**（动态类型成员）。它服务于 `BASIC_TYPEROOT`/`BASIC_TYPEINFO` 的类型系统，负责“这个类型在内存里怎么摆、每个成员在哪、多大、怎么对齐、怎么导出”。

---

## 一、总体定位

**`DYN_TYPEMEMBERS` = 类型成员集合 + 内存布局引擎 + 类型库导出器 + 类型信息同步器**

它做六件事：

1. **成员句柄分配**：`AllocHmembers` / `AllocHmembersNestedType` / `GetNewMemid`；
2. **内存布局计算**：`Layout` / `LayoutBases` / `LayoutMembers` / `LayoutDataMembers` / `LayoutVarOfHvdefn` / `LayoutEventSources` / `LayoutNestedType` / `LayoutFuncOfHfdefnEdit` / `LayoutStaticVarOfHvdefn` / `ResetOvfts`；
3. **大小/对齐计算**：`GetSizeAlignmentOf*` / `AdjustForAlignment` / `AlignMember`；
4. **类型校验**：`VerifyValidUDT` / `VerifyFunctionUdts` / `VerifyEnumList`；
5. **名称缓存/绑定**：`BuildNameCache` / `BuildBindNameTable` / `UpdateNameCacheOf*` / `UpdateBinderOptimization`；
6. **类型库导出/同步/序列化**：`CopyToITypeLib*` / `CopyDataToITypeInfo*` / `UpdateWithEXE` / `Read` / `Write`。

---

## 二、按功能分类

### 1. 生命周期与 COM 引用

| 函数 | 作用 |
|---|---|
| `DYN_TYPEMEMBERS(void)` | 构造 |
| `Init(SHEAP_MGR*, BASIC_TYPEROOT*)` | 初始化：关联堆管理器和所属模块 |
| `AddRef(void)` / `Release(void)` | COM 引用计数（只有 0x0B 字节，说明是简单内联实现） |
| `Invalidate(void)` | 使布局/缓存失效 |
| `IsLaidOut(void)` | 是否已完成布局 |
| `MakeLaidOut(int*)` | 标记/执行布局完成 |

说明它可能是 **COM 接口 `ITypeInfo` 的一个成员子对象**，由类型信息对象持有并做引用计数。

---

### 2. 成员句柄分配（Hmembers）

| 函数 | 作用 |
|---|---|
| **`AllocHmembers(void)`** | 分配整个类型的成员句柄表（函数体最大 0x9D2，是核心） |
| `AllocHmembersNestedType(uint)` | 为嵌套类型分配成员句柄 |
| `GetNewMemid(int)` | 分配新的成员 ID（MEMID） |

- “Hmember” 是 VB 类型系统中成员句柄（handle to member），每个成员（字段、方法、属性、事件、枚举项）都有一个。
- `AllocHmembers` 遍历类型的所有成员，分配句柄并建立索引，是布局前的准备。

---

### 3. 内存布局计算（`Layout*` 家族，核心）

| 函数 | 作用 |
|---|---|
| **`Layout(int*)`** | **布局总入口**，调用下面各子布局 |
| `LayoutBases(tagTYPEKIND, uint*, uint*)` | 布局基类子对象（COM 类的基类部分） |
| `LayoutMembers(int*)` | 布局所有成员 |
| `LayoutDataMembers(tagTYPEKIND, uint*, uint*, uint*, ulong)` | 布局数据成员（字段） |
| `LayoutVar(TYPEDESCKIND, ushort*, ushort*, uint, int, uint)` | 布局单个变量 |
| `LayoutVarOfHvdefn(tagTYPEKIND, ulong, ushort*, uint*, int, uint, int, ushort*, uint*)` | 按变量定义（Hvdefn）布局单个变量（体量很大 0x732） |
| `LayoutVarOfHvdefnEdit(ulong, uint*)` | 编辑态下布局变量 |
| `LayoutStaticVarOfHvdefn(tagTYPEKIND, ulong, uint*)` | 布局静态变量 |
| `LayoutEventSources(int*)` | 布局事件源（Event Sinks 的偏移/大小） |
| `LayoutNestedType(ulong, uint)` | 布局嵌套类型 |
| `LayoutFuncOfHfdefnEdit(ulong)` | 编辑态下布局函数 |
| `ResetOvfts(void)` | 重置对象 vtable 偏移（Ovft） |

这是 `DYN_TYPEMEMBERS` 的**核心职责**：计算类/UDT 的内存布局，包括：
- 基类子对象偏移；
- 数据成员偏移/大小/对齐；
- 静态变量区；
- 事件源（Event Sink）偏移；
- 嵌套类型；
- 对象 vtable 偏移（Ovft，用于 COM 接口分派）；
- 编辑态布局（`*Edit`）。

---

### 4. 大小/对齐计算

| 函数 | 作用 |
|---|---|
| `AdjustForAlignment(uint, uint, int)` | 按对齐要求调整大小 |
| `AlignMember(ushort*, uint, uint, int)` | 对齐单个成员 |
| `GetSizeAlignmentOfArray(tagTYPEKIND, ulong, uint*, uint*, int)` | 数组的大小/对齐 |
| `GetSizeAlignmentOfHtdefn(...)` | 按类型定义取大小/对齐（总入口） |
| `GetSizeAlignmentOfHtdefnNonUdt(...)` | 非 UDT 类型的大小/对齐 |
| `GetSizeAlignmentOfHtdefnUdt(ulong, uint*, uint*, int)` | UDT 类型的大小/对齐 |

这些是布局时的“度量工具”：给定类型/成员，返回其大小和对齐要求，供 `Layout*` 累加偏移。

---

### 5. 类型校验

| 函数 | 作用 |
|---|---|
| `VerifyValidUDT(ulong, int, tagTYPEKIND*, int*, int*, int*, int*)` | 校验 UDT 合法性（循环引用、大小、对齐等） |
| `VerifyFunctionUdts(ulong)` | 校验函数参数/返回值的 UDT |
| `VerifyEnumList(ITypeInfo*, ulong)` | 校验枚举列表 |
| `IsLaidOut(void)` | 是否已布局（布局前需校验） |

布局前必须确认类型合法（无循环 UDT、无非法递归等），否则布局会出错。

---

### 6. 名称缓存 / 绑定表

| 函数 | 作用 |
|---|---|
| `BuildNameCache(uint, int)` | 构建名称缓存 |
| `BuildBindNameTable(void)` | 构建绑定名称表 |
| `UpdateNameCacheOfBaseClass(ITypeInfo*, uint)` | 更新基类名称缓存 |
| `UpdateNameCacheOfHdefnList(ulong, INFOKIND, int, uint)` | 按句柄列表更新名称缓存 |
| `UpdateBinderOptimization(uint, uint, int)` | 更新绑定器优化信息 |
| `UpdateFdefnOfVdefn(ulong)` | 更新变量定义关联的函数定义 |
| `GetEnumNames(ulong, ushort**, long*, uint, uint*)` | 取枚举名称 |
| `GetDefnTypeBind(DEFN_TYPEBIND**)` / `GetDefnTypeBindSemiDeclared(...)` | 取定义类型绑定 |
| `GetTypeKind(void)` | 取类型种类（Class/UDT/Enum/Event 等） |
| `GetNewMemid(int)` | 新成员 ID |

- 名称缓存和绑定表用于快速查找成员/枚举名，对应 `GEN_PROJECT` 里 `NAMMGR`/`GENPROJ_BINDNAME_TABLE` 的成员级数据。
- `UpdateNameCacheOfBaseClass` 说明类的基类成员也会进名称缓存。

---

### 7. 类型库导出（`CopyToITypeLib*` / `CopyDataToITypeInfo*`）

| 函数 | 作用 |
|---|---|
| **`CopyToITypeLib(ITypeLib*, ITypeInfo*, int)`** | 把该类型导出到 `ITypeLib`（普通类型） |
| `CopyToITypeLibEnum(ITypeLib*, ulong, ITypeInfo*)` | 导出枚举类型 |
| `CopyToITypeLibEvent(ITypeLib*, ITypeInfo*)` | 导出事件源 |
| **`CopyDataToITypeInfo(ITypeInfo*, ICreateTypeInfo*, int)`** | 把成员数据复制到 `ITypeInfo`（核心，体量 0xF98） |
| `CopyDataToITypeInfoEnum(ITypeInfo*, ulong, ICreateTypeInfo*)` | 复制枚举数据 |
| `CopyDataToITypeInfoEvent(ITypeInfo*, ICreateTypeInfo*)` | 复制事件数据 |
| `WriteEnumNames(ICreateTypeInfo*, ushort**, long*, uint)` | 写枚举名称 |

- 这些函数把 `DYN_TYPEMEMBERS` 里计算好的成员布局、名称、ID、类型信息，写入 COM 类型库（`ITypeLib`/`ITypeInfo`/`ICreateTypeInfo`）。
- 对应 `GEN_PROJECT::CopyToITypeLib(IExeFile*)` 在 `MakeExe` 里把工程类型库写入 EXE 的流程：`GEN_PROJECT` 调各类型的 `DYN_TYPEMEMBERS::CopyToITypeLib*` 完成逐类型导出。
- `CopyDataToITypeInfo` 是**类型信息持久化到类型库的核心**。

---

### 8. 与 EXE 类型信息同步

| 函数 | 作用 |
|---|---|
| **`UpdateWithEXE(ITypeInfo*, TINFOKIND)`** | 用 EXE 中的类型信息更新/合并当前类型（体量 0xE10，很大） |
| `UpdateWithExeEnum(void)` | 枚举类型的 EXE 同步 |

- 编译时若引用已有的 EXE/类型库，需要把外部类型信息合并进来，保持成员布局/ID 一致。
- `UpdateWithEXE` 是**类型兼容性/增量编译**的关键：保证新建类型与 EXE 中已有类型布局一致。

---

### 9. 序列化

| 函数 | 作用 |
|---|---|
| `Read(IStream*)` | 从流读取成员布局 |
| `Write(IStream*)` | 向流写入成员布局 |

- 与 `GEN_PROJECT::Read/Write`、`ReadCanonicalProject`/`WriteCanonicalProject` 配合，把类型成员布局持久化到工程文件/EXE。

---

## 三、在编译链路中的位置

```
GEN_PROJECT（工程）
  ├─ 类型库/引用库管理
  ├─ 模块集合（BASIC_TYPEROOT / BASIC_TYPEINFO）
  │     └─ 每个类型（类/UDT/Enum/Event）
  │           └─ DYN_TYPEMEMBERS          ← 本类
  │                 ├─ AllocHmembers      分配成员句柄
  │                 ├─ Verify*            校验 UDT/枚举/函数
  │                 ├─ Layout*            计算内存布局
  │                 │    ├─ LayoutBases
  │                 │    ├─ LayoutDataMembers / LayoutVar*
  │                 │    ├─ LayoutEventSources
  │                 │    ├─ LayoutNestedType
  │                 │    └─ ResetOvfts
  │                 ├─ BuildNameCache / UpdateNameCache*
  │                 ├─ CopyToITypeLib* / CopyDataToITypeInfo*  导出类型库
  │                 ├─ UpdateWithEXE      与 EXE 类型信息同步
  │                 └─ Read / Write       序列化
  └─ MakeExe → CopyToITypeLib → 逐类型导出 → FinishEXE
```

- `DYN_TYPEMEMBERS` 是**类型级**的，比 `GEN_PROJECT`（工程级）和 `BASIC_TYPEROOT`（模块级）更细。
- 它不生成 IL/代码，而是**为类型/成员计算内存布局和元数据**，供 BIL 生成（`EXMGR`）、Native 代码生成（偏移/大小）、类型库导出和链接使用。

---

## 四、关键结论

1. **它是类型成员布局引擎**：`Layout*` + `GetSizeAlignmentOf*` + `AlignMember` 计算类/UDT/事件源的字段偏移、大小、对齐、vtable 偏移。
2. **它是成员句柄管理器**：`AllocHmembers` / `GetNewMemid` 为每个成员分配 Hmember/MEMID。
3. **它是类型校验器**：`VerifyValidUDT` / `VerifyFunctionUdts` / `VerifyEnumList` 保证布局前类型合法。
4. **它是类型库导出器**：`CopyToITypeLib*` / `CopyDataToITypeInfo*` 把成员元数据写入 `ITypeLib`/`ITypeInfo`，是 `GEN_PROJECT::CopyToITypeLib` 的逐类型执行者。
5. **它是 EXE 类型同步器**：`UpdateWithEXE` 把外部 EXE/类型库中的类型信息合并进来，保证布局兼容。
6. **它是名称/绑定缓存构建者**：`BuildNameCache` / `BuildBindNameTable` / `UpdateNameCacheOf*` 为成员名查找和绑定优化提供数据。
7. **它是 COM 对象**：有 `AddRef`/`Release`/构造/`Init`，被类型信息对象持有并引用计数。

---

## 五、一句话总结

> **`DYN_TYPEMEMBERS` 是 VB 编译器的“动态类型成员管理与内存布局引擎”**：它为一个类型（类/UDT/枚举/事件源）分配成员句柄，校验类型合法性，计算基类/字段/静态变量/事件源/嵌套类型/vtable 的偏移、大小和对齐，构建名称与绑定缓存，并把成员元数据导出到 `ITypeLib`/`ITypeInfo`、与 EXE 中的类型信息同步、以及序列化到工程/EXE 文件。它是 `GEN_PROJECT` 类型库导出和 Native 代码生成（字段偏移）之间的关键类型级组件。
