# GEN_PROJECT 类分析

从这份函数清单看，**`GEN_PROJECT` 是 VB 编译器的“工程/项目级核心对象”**。它代表一个工程（Project），负责管理模块集合、类型库/引用库、绑定与名称查找、编译流程驱动、序列化与 IDE 交互，并作为 `MakeExe` 等编译入口的顶层对象。

它处在比 `BASIC_TYPEROOT`（模块）、`EXMGR`（BIL 生成）、`buildProject`（构建工程）更高的层级：**一个 `GEN_PROJECT` 包含多个模块，驱动它们编译，管理类型/引用/绑定，最终交给构建系统生成 EXE。**

---

## 一、总体定位

**`GEN_PROJECT` = 工程/项目对象 + 类型库与引用库管理器 + 绑定/名称解析器 + 编译流程驱动 + 工程序列化**

它做六件事：

1. **工程生命周期与元数据**：创建/初始化/销毁、名称、GUID、版本、LCID、模式、选项、可见性、脏标志；
2. **模块管理**：创建/复制/删除/替换/按名/按索引/按 hlnam 获取模块；
3. **类型库与引用库管理**：添加/删除/查找引用库、LibId 映射、twiddled lib、type info 缓存；
4. **绑定与名称解析**：`Bind`/`FindName`/`FindMembers`/`IsName`、名称缓存、限定类型绑定缓存；
5. **编译驱动**：`Compile`、`MakeExe`、`Decompile`、`NotifyBeginCompile`/`NotifyEndCompile`/`NotifyModuleCompiled`、`LoadAllTypes`、`RemoveEmptyEventSubs`；
6. **序列化与 IDE 交互**：`Read`/`Write`、canonical project、旧二进制、GTLOLE、国际化信息、字符串读写。

---

## 二、按功能分类

### 1. 工程生命周期与元数据

| 函数 | 作用 |
|---|---|
| `GEN_PROJECT::GEN_PROJECT(void)` / `~GEN_PROJECT(void)` | 构造/析构 |
| `Create(ExecProj*, int, GEN_PROJECT**)` | 创建工程对象 |
| `Init(void)` / `InitNew(int)` / `InitGTLOLE(void)` | 初始化 |
| `Destroy(void)` / `Release(void)` / `AddRef(void)` / `QueryInterface` | COM 生命周期 |
| `GetName`/`SetName`/`SetNameGTLOLE` | 工程名 |
| `SetGuid`/`GetCompatibleExe`/`SetCompatibleExe` | GUID/兼容 EXE |
| `GetLcid`/`GetLcidOriginal`/`SetLcid` | 语言/区域 |
| `GetOptions`/`SetOptions` | 编译选项 |
| `GetMode`/`SetMode`/`GetModeEx`/`SetModeEx` | 运行/设计模式 |
| `IsDirty`/`SetDirty`/`IsDirtyByUser`/`SetDirtyByUser`/`SetModified`/`SetModDirtyByUser` | 脏标志 |
| `IsVisible`/`SetVisible`/`HideAllModules` | 可见性 |
| `GetDirectory`/`GetDocumentation`/`SetDocString`/`SetHelpContext`/`SetHelpFileName` | 工程文档/帮助 |
| `SetVersion`/`UpdateVersion`（由 buildProject 调用） | 版本 |
| `IsWizard`/`SetEditInRunTime`/`TranslateSCP` | 向导/运行时编辑/SCP |
| `Reset(int)` | 重置 |

---

### 2. 模块管理

| 函数 | 作用 |
|---|---|
| `GetModuleCount(uint*)` | 模块数 |
| `GetModule(uint, IVbaModule**)` | 按索引取模块 |
| `GetModuleOfName`/`GetModuleOfHlnam` | 按名/按 hlnam 取模块 |
| `CreateModule`（多个重载） | 创建模块（普通/可选初始化/按类型） |
| `CreateClass2`/`CreateClass3`/`CreateClassOptionalInit` | 创建类模块 |
| `CopyModule`（重载） | 复制模块 |
| `CopyBinModule`/`CopyTxtModule` | 复制二进制/文本模块 |
| `ReplaceModule` | 替换模块 |
| `DeleteModule`/`Deleting` | 删除模块 |
| `DemandLoadModule`/`AbortDemandLoadModule`/`IdleLoadText`/`IsTextLoaded` | 按需加载模块文本 |
| `Add(BASIC_TYPEINFO*, ushort*)`/`AddTypeEntry`/`AddTypeEntryGTLOLE`/`UnAddTypeEntry` | 添加类型/模块条目 |
| `DestructTypeEntry`/`RemTypeInfo` | 销毁/移除类型条目 |
| `MakeClassRunnable`/`ReleaseClassInstance` | 类可运行化/释放实例 |
| `RegAppObject` | 注册 App 对象 |

---

### 3. 类型库 / 引用库管理

| 函数 | 作用 |
|---|---|
| `AddDefaultReferences` | 添加默认引用 |
| `AddRefLib`（多个重载）/`AddLibAsRefLib`/`AddPrivateRefLib`/`AddPrivateRefLibOfTwoLibids`/`AddRefLibOfLibId`/`AddRefLibOfTwoLibids` | 添加引用库 |
| `DeleteRefLib`/`RemoveRefLib` | 删除引用库 |
| `FindRefLib`/`GetRefLib`/`GetRefLibCount`/`GetRefLibId`/`GetHRefLib`/`GetIndexOfhRefLib` | 查找/枚举引用库 |
| `GetExtRefLib`/`GetTwiddledRefLib`/`GetPathRefLib`/`GetLibIdOfRefLib` | 取外部/twiddled/路径/LibId |
| `DetectDuplicateRefLib`/`DetectCycle`/`IsTypeLibRemovable` | 重复/循环/可移除检测 |
| `VerifyReferences`/`VerifyNoProjLevelConflict`/`VerifyWithExe` | 引用验证 |
| `UpdateLibId`（重载）/`UpdateLibIdIfNecessary`/`MakeRelativeLibId` | LibId 更新/相对化 |
| `NotifyBinderOfRefLib`/`RebuildRefList(EBREFDLG*)` | 引用库变更通知/重建引用列表 |

---

### 4. 类型库 / TypeInfo 接口（ITypeLib/ITypeInfo 实现）

| 函数 | 作用 |
|---|---|
| `GetTypeInfoCount`/`GetTypeInfo`/`GetTypeInfoType`/`GetTypeInfoLocal`/`GetTypeInfoOfGuid`/`GetTypeInfoMockup` | 类型信息查询 |
| `GetFilteredTypeCount`/`GetFilteredTypeInfo` | 过滤类型 |
| `GetTypeComp`/`GetTypeCompGTLOLE` | 类型组件 |
| `GetLibAttr`/`ReleaseTLibAttr`/`GetLibId`/`SetLibId`/`SetLibFlags` | 类型库属性 |
| `CreateTypeInfo`（重载）/`GetGdtiOfItyp`/`GetCompressedTypeId`/`TypeInfoFromCompressedTypeId`/`WriteTypeId`/`MakeLocalTypeId`/`UpdateTypeId` | 类型创建/压缩 ID/本地 ID |
| `GetQszTypeInfoTypeId`/`GetCanonicalOffset`/`SetCanonicalOffset`/`SetTypeDocString`/`SetTypeHelpContext`/`SetTypeName` | 类型 ID/偏移/文档/名称 |
| `GetPublicClassEnumCount`/`AssignClassIID`/`HteOfHguid` | 公共类枚举/IID 分配 |
| `GetKeyword`/`GetKeywordNoAlloc`/`GetConstants`/`SetConstants`/`SetCCConstantValues`/`GetCCConstValue`/`GetCCConstantString` | 关键字/常量 |

---

### 5. 绑定 / 名称解析 / 缓存

| 函数 | 作用 |
|---|---|
| `Bind`/`BindProc`/`GetBinddescOfSzName` | 绑定名称到类型/成员 |
| `FindName`/`FindMembers`/`IsName`/`IsNameDefInReferencedTlb` | 查找名称/成员/是否名称 |
| `GetIndexOfName`/`GetIndexOfLocalRegId`/`GetRegId`/`GetLocalRegId` | 名称/注册 ID 索引 |
| `AddNameToCache`/`InvalidateNameCache`/`ReadNameCacheArray` | 名称缓存 |
| `InitQualCache`/`ReleaseQualCache`/`AddQualCache`/`GetQualCache`/`QualCacheOfHimptype`/`QualCacheOfOrdinal`/`QualCacheOfTypeInfo`/`QualCacheOfTypeLib`/`SetQualCacheOfOrdinal` | 限定类型绑定缓存（QualCache） |
| `GetDefnTypeBind`/`GetDefnTypeBindSemiDeclared`/`GetUnqualTypeBind`/`GetUnqualTypeBindSemiDeclared` | 定义/非限定类型绑定 |
| `BuildBindingTables`/`FixProjBindTable`/`InvalidateBindingTables` | 绑定表构建/修复/失效 |
| `ProcessIntlInfo`/`ReadIntlInfo`/`WriteIntlInfo` | 国际化信息 |

---

### 6. 编译驱动（最关键）

| 函数 | 作用 |
|---|---|
| **`MakeExe(...)`** | **顶层 Make EXE 主方法**（你之前分析过）：LoadAllTypes → RemoveEmptyEventSubs → Decompile → StartEXE → 逐模块 Native 生成 → C2 → FinishEXE |
| **`Compile(int)` / `Compile(void)`** | 工程级编译驱动 |
| `Decompile(void)` | 反编译工程到中间态，便于重新驱动编译 |
| `LoadAllTypes(void)`/`LoadTypeInfo(uint)`/`SetTargetStateToDeclared`/`SetTargetStateToSemiDeclared` | 加载/推进类型状态 |
| `RemoveEmptyEventSubs(void)` | 删除空事件订阅 |
| `NotifyBeginCompile`/`NotifyEndCompile`/`NotifyModuleCompiled` | 编译通知（IDE） |
| `CopyToITypeLib(IExeFile*)` | 把工程类型库写入 EXE 文件接口 |
| `ReleaseResources`/`CleanEvSrcTbls`/`ResetPrdtInMockupRecTypeInfos` | 资源/事件源/Prdt 清理 |
| `HasState(int*)`/`CheckProjectId`/`UpdateProjectId` | 状态/工程 ID |
| `CacheWithExe(void)` | 与 EXE 缓存 |
| `CanProjChange(...)`/`IsInTextLoad`/`GetStreamName`/`SetStreamName` | 可变更性/文本加载/流名 |

---

### 7. 序列化 / 持久化

| 函数 | 作用 |
|---|---|
| `Read(IStorage*)`/`Write(IStorage*)` | 工程读写 |
| `ReadCanonicalProject`/`WriteCanonicalProject` | 规范工程格式 |
| `ReadOldBinary`/`WriteOldBinary` | 旧二进制格式 |
| `ReadGTLOLE`/`WriteGTLOLE`/`InitGTLOLE` | GTLOLE 格式 |
| `ReadSTLIB`/`WriteSTLIB` | STLIB 格式 |
| `ReadBSTR`/`ReadUnicodeBSTR`/`ReadString`/`ReadStringAsUnicode`/`WriteAsBstr`/`WriteAsBstrA`/`WriteString`/`WriteStringAsUnicode` | 字符串读写 |
| `CreateStream`/`OpenStream`/`SaveAs93`/`SaveAllChanges` | 流/保存 |
| `WriteXlatRecord`/`WriteXlatRecordLong`/`WriteXlatRecordShort`/`WriteXlatRecordString`/`WriteXlatRecordStringA`/`WriteXlatRecordUnicodeString`/`WriteXlatRecordRefLib` | 转换记录（Xlat） |
| `CreateHsz`/`ResetHsz`/`QszOfHsz`/`QtrOfHchunk32`/`QHteRef`/`IhteHash` | HSZ/块/哈希辅助 |
| `operator new(uint)` | 自定义分配 |

---

## 三、在编译链路中的位置

```
GEN_PROJECT::MakeExe                         ← 工程级顶层驱动
  ├─ GEN_PROJECT::LoadAllTypes
  ├─ GEN_PROJECT::RemoveEmptyEventSubs
  ├─ GEN_PROJECT::Decompile
  ├─ GEN_PROJECT::NotifyBeginCompile
  ├─ buildProject::StartEXE
  ├─ GEN_PROJECT::CopyToITypeLib
  ├─ nativeGenInit / LoBil_Initialize
  └─ 逐模块循环：
       ├─ nativeGenModuleBeg
       ├─ BASIC_TYPEROOT::EnsureInCompiledState
       │     └─ BASIC_TYPEROOT::Compile
       │           └─ EXMGR::GenerateModule        ← BIL 生成
       │                 └─ EXMGR::GenerateFunction
       │                       ├─ CompileLine（BIL）
       │                       ├─ TransformToLoBil
       │                       └─ nativeGenProcedure（Native）
       ├─ nativeGenModuleEnd
       └─ RunConsoleApp(C2.EXE)                    ← Native 后端
  └─ GEN_PROJECT::Compile
       └─ buildProject::FinishEXE                  ← 链接 EXE
```

**`GEN_PROJECT` 是最上层：管理工程/模块/类型/引用/绑定，驱动编译；`BASIC_TYPEROOT` 管模块，`EXMGR` 管 BIL/Native 生成，`buildProject` 管构建/链接。**

---

## 四、与其他类的关系

| 类 | 层级 | 关系 |
|---|---|---|
| **`GEN_PROJECT`** | 工程 | 包含多个模块、引用库、类型库；驱动编译 |
| `BASIC_TYPEROOT` / `BASIC_TYPEINFO` | 模块 | `GEN_PROJECT` 的模块集合成员，负责模块级编译状态 |
| `EXMGR` | 表达式/BIL | 由 `BASIC_TYPEROOT::Compile` 调用，负责 BIL/LowBil/Native 生成 |
| `buildProject` | 构建工程 | `GEN_PROJECT::MakeExe` 通过 `StartEXE`/`FinishEXE` 驱动链接 |
| `FEDSYM` | 符号 | 模块/函数符号包装 |
| `IExeFile` / `ITypeLib` / `ITypeInfo` | COM 接口 | `GEN_PROJECT` 实现/使用类型库接口，写入 EXE |

---

## 五、一句话总结

> **`GEN_PROJECT` 是 VB 编译器的工程级核心对象**：它代表一个工程，管理模块集合、类型库与引用库、绑定/名称缓存、工程元数据与序列化，并通过 `MakeExe`/`Compile` 驱动整个编译链路（`LoadAllTypes` → `Decompile` → 逐模块 `EnsureInCompiledState` → `EXMGR` 生成 BIL/Native → C2 后端 → `FinishEXE` 链接），同时实现 `ITypeLib`/`ITypeInfo` 等接口供绑定与类型查询使用。

如果你要继续，下一步可以：
1. 分析 `GEN_PROJECT::Compile(int)` 与 `MakeExe` 的分工；
2. 分析 `GEN_PROJECT` 的成员布局（`this+180` 模块数、`this+51` 模块表、`this+815` 构建对象等）；
3. 分析 `QualCache`/`Bind` 的绑定解析流程；
4. 分析 `ReadCanonicalProject`/`WriteCanonicalProject` 的工程序列化格式。
