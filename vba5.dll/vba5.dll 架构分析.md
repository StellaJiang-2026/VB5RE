# vba5.dll 架构分析

从这份符号表看，**vba5.dll 是 Visual Basic for Applications 5.0 的“编译器 + 运行时 + IDE 组件”一体化 DLL**。它同时承担四件事：

1. **VBA 工程/模块的编译**（前端 → BIL → LowBil → Native → EXE 链接）；
2. **VBA 运行时**（`rtc*` 内置函数、`__vba*` 辅助、`Err`/`Collection` 对象、异常/内存管理）；
3. **VBA IDE 的编辑器/调试器/对象浏览器**（`CPANE`、`VIEWMGR`、`BROWSEDATA`、`WATCHMGR`）；
4. **VBA 扩展对象模型**（`CVbaExt*`、`IVbaProject`/`IVbaModule` 等 COM 接口）。

下面按子系统归纳。

---

## 一、总体分层

```
┌─────────────────────────────────────────────────────────┐
│  顶层驱动：GEN_PROJECT / ExecProj / ExecMod              │
│  （工程对象、COM 接口、MakeExe 驱动）                     │
├─────────────────────────────────────────────────────────┤
│  构建/链接：buildProject / buildModule / CExeFile        │
├─────────────────────────────────────────────────────────┤
│  代码生成：EXMGR / EXGEN / BILALLOC / gCIL / nativeGen*  │
│  （BIL → LowBil → Native → C2）                          │
├─────────────────────────────────────────────────────────┤
│  前端：scanner / parser / genOpc / BASIC_TYPESRC        │
├─────────────────────────────────────────────────────────┤
│  类型系统：BASIC_TYPEROOT / DYN_TYPEMEMBERS / IMPMGR /   │
│  NAMMGR / TYPEMGR / ITypeInfo/ITypeLib/ITypeComp 实现    │
├─────────────────────────────────────────────────────────┤
│  运行时：rtc* / __vba* / VBACollection / VBAErr / 异常   │
├─────────────────────────────────────────────────────────┤
│  IDE：CPANE / VIEWMGR / MODVWR / BROWSEDATA / WATCHMGR  │
├─────────────────────────────────────────────────────────┤
│  基础设施：内存分配器 / 流 / 字符串 / COM 枚举器 / CRT   │
└─────────────────────────────────────────────────────────┘
```

---

## 二、顶层驱动：工程与构建

### 2.1 工程对象

| 类 | 角色 |
|---|---|
| **`GEN_PROJECT`** | 编译器工程对象，实现 `ITypeLib`/`ICreateTypeLib`/`IVbaProject`，是 `MakeExe` 的驱动者，管理模块、引用库、类型库、绑定缓存、编译流程 |
| **`ExecProj` / `ExecMod`** | COM 层的工程/模块对象，实现 `IVbaProject`/`IVbaModule`，负责持久化和 IDE 接口 |
| **`epiProject` / `epiModule`** | 链接期工程/模块描述（EPI = Executable Project Information） |
| `PROJLIST` / `ProjectManager` / `CRefList` | 工程列表、引用关系、引用库列表管理 |
| `CVbaExtReferences` / `CVbaExtReference` | VBA 扩展引用对象模型 |

关系：`ExecProj`（COM/持久化）→ `GEN_PROJECT`（编译期）→ `epiProject`（链接期）。

### 2.2 构建/链接

| 类 | 角色 |
|---|---|
| **`buildProject`** | 构建工程，`StartEXE`/`FinishEXE`/`AddModule`/`CompileModule` |
| **`buildModule`** | 构建模块，`Init`/`CompleteEPI`/`SetRTMI`/`NotifyModuleCompiled` |
| `buildLookupTables` | 构建期查找表（名称/GUID/类型索引） |
| **`CExeFile`** | EXE 文件写入，`WriteExe`/`WriteExePart`/`WriteRsrcDir`/`WriteTypelib1To` |
| `EXEWrite*` / `EXEFixup*` | EXE 节/修复/导出/资源写入 |
| `memAlloc` / `memSection` / `memoryBlock` / `FIXUP` | 链接内存分配、节、修复 |
| `ExeNameTable` | EXE 名称表 |
| `Epibind` / `EPIBIND` / `BindptrFromPepibind` | EPI 绑定 |

---

## 三、代码生成：BIL → LowBil → Native

### 3.1 BIL 生成核心

| 类 | 角色 |
|---|---|
| **`EXMGR`** | 表达式/执行管理器，BIL 生成核心（`CompileLine`、`Make*`、`Process*`、`Bind*`、`Coerce*`、`Cntl*`、`Tpool*`） |
| **`BILALLOC`** | BIL 树分配器（`AllocBilTree`、`CopyBilTree`、`MakeAdrNode`、`MakeSeq*Node`） |
| **`GEN_STACK`** | 生成栈（`Push`/`Pop`/`Insert`/`NextElem`） |
| `BBT<BILSYMNODE>` / `BBT<BILTYPNODE>` | 平衡二叉树（符号/类型节点） |
| `stmtExpr` / `stmtList` / `caseList` | BIL 节点/语句列表 |

### 3.2 表达式代码生成

| 类 | 角色 |
|---|---|
| **`EXGEN`** | 表达式代码生成（`ExgenRvalue`/`ExgenLvalue`/`ExgenCall`/`ExgenFunction`/`Emit*`） |
| `EXADDR` / `EXGEN_STATE` | 地址模式/生成状态 |
| `EXTBL` | 外部表/代码块（`BeginFuncNew`/`EndFunc`/`CreateAndEmitStub`/`Emit*_Bil`/`SetRTMI`） |
| `RTMI` / `EXFRAME` | 运行时元信息/执行帧 |
| `TransformToLoBil` / `Transform_*` / `BuildLoTree_*` | BIL → LowBil 转换 |

### 3.3 Native 代码生成

| 函数 | 角色 |
|---|---|
| `gCIL*`（`gCILgen*`、`gCILwrite*`、`gCILprocBody`、`gCILstmtList`） | Native 代码/IL 生成（注意：这里的 “CIL” 是 VB 的中间表示，不是 .NET CIL） |
| `nativeGenInit` / `nativeGenModuleBeg` / `nativeGenModuleEnd` / `nativeGenProcedure` / `nativeGenDone` | Native 代码生成驱动 |
| `LoBil_Initialize` / `LoBil_Cleanup` | LoBil 生命周期 |
| `C2.EXE`（外部） | 真正的后端，把中间文件编译成 `.obj` |

### 3.4 前端

| 类/函数 | 角色 |
|---|---|
| `scanner` | 词法分析（`scan`/`scanIdentifier`/`scanNumericConstant`/`scanStringConstant`/`scanRecordLineCont`） |
| `parser` | 语法分析（`parseStatement`/`parseTerm`/`parseExpr`/`parseVariable`/`parseProcDecl`/`parse*Stmt`） |
| `genOpc` | 操作码生成（`genOpcodesForStmt`/`genOpcodesForVal`/`genOpcodesForProc`/`genOpcodesForDim`） |
| `ptCreate*` | 语法树节点构造（`ptCreateOperNode`/`ptCreateVarNode`/`ptCreateProcNode`） |
| `BASIC_TYPESRC` | 源码管理/编辑/断点/行变更 |
| `OPCODE_MGR` | 操作码行管理（`GetLine`/`InsertLine`/`ReplaceLine`） |
| `OPCODE_TRAVERSE` / `OPCODE_SEARCH` | 操作码遍历/搜索 |

---

## 四、类型系统与绑定

### 4.1 模块/类型基础

| 类 | 角色 |
|---|---|
| **`BASIC_TYPEROOT`** | 模块对象，管理编译状态（`EnsureIn*State`、`Compile`、`Decompile`、`GetExMgr`/`GetDtmbrs`/`GetEntMgr`/`GetImpMgr`/`GetNamMgr`/`GetViewmgr`） |
| **`BASIC_TYPEINFO`** | 类型信息对象 |
| **`BASIC_TYPESRC`** | 源码/编辑/断点/行变更 |
| **`DYN_TYPEMEMBERS`** | 动态类型成员布局引擎（`AllocHmembers`、`Layout*`、`GetSizeAlignmentOf*`、`CopyToITypeLib*`、`UpdateWithEXE`） |
| `BASIC_CLASS` / `BASIC_DISPINTERFACE` | 类/调度接口运行时 |
| `FEDSYM` / `FEDTYP` / `FEDARG` / `FEDSINK` | 符号/类型/参数/事件槽的封装层 |

### 4.2 管理器

| 类 | 角色 |
|---|---|
| **`IMPMGR`** | 导入管理器（`Himptype*`、`BindTypeInfo`、`SetPtinfo`、`CheckLayoutDep`） |
| **`NAMMGR`** | 名称管理器（`HlnamOfStr`、`StrOfHlnam`、`BindWithExbCache`、`hash*`） |
| **`TYPEMGR`** | 类型管理器（`AddLibRef`、`TypeLibLoaded`、`CanProjChange`、`VisitDependentProj`） |
| **`GUIDMGR`** | GUID 管理器（`AddGuid`、`HguidOfGuid`） |
| **`VERMGR`** | 版本管理器（`MarkReferencingModulesForDecompilation`） |
| **`INSTMGR`** | 实例管理器（`CreateResDescTbl`、`StaticResDescTblComplete`） |
| **`ENTRYMGR`** | 入口点管理器（`AllocDllentrydefn`） |

### 4.3 类型绑定（DEFN_TYPEBIND 家族）

`DEFN_TYPEBIND`（基类）→ `DYN_TYPEBIND` / `GENPROJ_TYPEBIND` / `EXTERNAL_TYPEBIND` / `MOCKUP_TYPEBIND` / `UNQUAL_TYPEBIND` / `REC_TYPEBIND`，负责名称绑定、成员查找、类型解析。

### 4.4 ITypeInfo/ITypeLib/ITypeComp 实现

| 类 | 角色 |
|---|---|
| `CEcTypeInfo` / `CEcTypeComp` / `CEcProjTypeComp` | Ec 前缀的 COM 类型信息/组件实现 |
| `WrapTypeInfo` / `WrapTypeLib` / `WrapTypeComp` | 包装器（用于对外暴露工程类型库） |
| `REC_TYPEINFO` / `REC_TYPEBIND` | 记录类型（UDT）的 ITypeInfo 实现 |
| `CDefnTypeComp` | 定义类型组件 |
| `TINFO_LOADER` / `TINFO_CONTENT_LOADER` / `FUNCDESC_LOADER` | 类型信息加载器（从 ITypeInfo 加载到内部符号） |
| `BLENDER` | 类型库混合（`BlendTypeLibsOfControls`、`BlendFuncsAndVars`、`BlendCoClass`） |

### 4.5 名称缓存/绑定表

`DYN_BINDNAME_TABLE` / `GENPROJ_BINDNAME_TABLE`：名称→绑定缓存，`BuildTable`/`FindIndexOfHlnam`/`TraverseDefnList`。

---

## 五、运行时

### 5.1 内置函数（`rtc*`）

覆盖 VBA 语言所有内置函数：`rtcMsgBox`、`rtcInputBox`、`rtcLeft/Right/Mid`、`rtcTrim`、`rtcUCase/LCase`、`rtcInStr`、`rtcStrComp`、`rtcIsArray/IsDate/IsEmpty/IsNull/IsNumeric/IsObject`、`rtcDateAdd/DateDiff/DatePart`、`rtcFormat`、`rtcShell`、`rtcEnviron`、`rtcCreateObject/GetObject`、`rtcFile*`、`rtcDir`、`rtcRandom*`、`rtcSgn/Abs/Int/Fix`、`rtcTypeName`、`rtcArray/Choose/Switch/IIf`、`rtcFV/IPMT/PMT/PPMT/NPV/IRR` 等。

### 5.2 运行时辅助（`__vba*`）

类型转换/强制/运算/数组/记录/字符串/文件/异常辅助：`__vbaStrCat`、`__vbaVarAdd/Sub/Mul/Div`、`__vbaCyAdd`、`__vbaRecAssign`、`__vbaFileOpen`、`__vbaGet/ Put`、`__vbaFreeVar`、`__vbaCastObj`、`__vbaCheckType`、`__vbaForEach*` 等。

### 5.3 运行时对象

| 类 | 角色 |
|---|---|
| `VBACollection` | `Collection` 对象（红黑树实现，`Add`/`Item`/`Remove`/`NewEnum`） |
| `VBAErr` | `Err` 对象（`Number`/`Description`/`Source`/`HelpFile`/`HelpContext`/`Raise`/`Clear`） |
| `VBAExtension` | `VBA` 扩展对象 |
| `VBAEnum` / `CEnumVariant` | `IEnumVARIANT` 实现 |
| `CVBFiles` / `CVBFormat` | `IVBFiles`/`IVBFormat` 接口 |

### 5.4 异常/内存/运行时基础设施

- `__vba*` 异常辅助、`_lblEX_*` 运行时标签；
- `RUN_INSTMGR` 运行时实例管理（`CreateObjInstance`/`ConstructModInstance`/`MakeZombieVtable`）；
- `EXFRAME` 执行帧、`RTMI` 运行时元信息；
- `BASIC_CLASS` / `BASIC_CLASS_ZOMBIE` / `Zombie*` 僵尸实例；
- `EVENT_SINK` / `EVENT_SOURCE_TABLE` / `CConPnt` / `CConPntCont` 事件/连接点。

---

## 六、IDE / 编辑器 / 调试器

### 6.1 编辑器核心

| 类 | 角色 |
|---|---|
| `VIEWMGR` | 视图管理器（行操作、断点、通知） |
| `LINEMGR` / `CACHELN` / `LNBUF` / `LNBUFZ` / `LNBUFCLIP` / `LNSINK` | 行管理/缓存/缓冲/剪贴板 |
| `CPANE` | 代码编辑器面板（光标/选择/输入/绘制/滚动） |
| `MODVWR` | 模块视图（多面板/拆分/滚动条） |
| `PANE` / `NPANE` / `LPANE` / `VLPANE` / `WLPANE` / `LLPANE` / `TOOLBAR_PANE` / `SPLITTER_PANE` / `SPLITTER_EBWND` | 各类面板 |
| `DWIN` / `EBWNDFRAME` | 窗口框架 |
| `XACTMGR` / `RECORD` / `XACTEL` | 编辑事务/Undo/Redo |
| `BMMGR` | 书签管理 |
| `BPMGR` | 断点管理 |
| `GOBACK` | 跳转历史 |
| `ISTRG` | 中间字符串流 |

### 6.2 调试

| 类 | 角色 |
|---|---|
| `WATCHMGR` / `WATCH` / `WATCH_ITEMDATA` | 监视窗口/表达式求值 |
| `LOCAL_ITEMDATA` / `VARIABLE_ITEMDATA` / `OBJECTCHILDREN` / `OBJECT_MEMBER` / `VALUETYPE` | 局部变量/对象成员/值类型显示 |
| `CALLSTACK_ITEM` / `CreateCallstack` | 调用栈 |
| `BRKEDIT_INFO` / `NTEX_HEAD` | 断点编辑/异常头 |
| `IMMEDMOD_INFO` | 立即窗口模块 |
| `ERRINFO` / `BTSRC_ERR` | 错误信息 |

### 6.3 对象浏览器

| 类 | 角色 |
|---|---|
| `BROWSEDATA` / `BRWSYM` | 对象浏览器数据/符号 |
| `OBJBRW` / `OBJBRW_PANE` / `OBJBRW_TOOLBAR_PANE` | 对象浏览器 UI |
| `PARAMWALKER` / `SORTINFO` | 参数遍历/排序 |
| `FILTITER` | 类型过滤迭代器 |

### 6.4 其他 UI

`POPUPMGR` / `POPUPTIP` / `NAMELIST` / `VALUETIP` 自动完成/提示；`BLTTER` 位图绘制；`TCHANGE` 文本变更；`CACHELN` 行缓存。

---

## 七、内存管理（多套分配器并存）

| 分配器 | 角色 |
|---|---|
| `block_allocator` / `small_block_allocator` / `large_block_allocator` | 前端/符号使用的块分配器 |
| `commitAllocator` | 提交式分配器 |
| `norls_allocator` | No-RLS 分配器（`nraAlloc`/`nraFree`/`nraToss`） |
| `memHeapAlloc` / `memSimpleAlloc` | 堆/简单分配器 |
| `SHEAP_MGR` / `BLK_MGR` / `BLKMGR32` / `DYNBLKMGR1632` / `BLK_DESC` / `BLKDESC32` | 堆块管理 |
| `memAlloc` / `memSection` / `memoryBlock` / `FIXUP` | 链接期内存/节/修复 |
| `CZeroInit` | 零初始化分配 |
| `ProfMemAlloc` / `ProfMemFree` / `ProfVirtualAlloc` | 性能分析分配包装 |

多套并存说明 vba5.dll 由不同子系统演进合并而来，各自保留自己的分配器。

---

## 八、COM / 对象模型

### 8.1 VBA 扩展对象模型（VBIDE 扩展）

`CVbaExtModule` / `CVbaExtEditor` / `CVbaExtEditors` / `CVbaExtMembers` / `CVbaExtMember` / `CVbaExtReferences` / `CVbaExtReference`，实现 `IVbaExt*` 接口，让插件访问 VBA 工程结构。

### 8.2 其他 COM 实现

| 类 | 角色 |
|---|---|
| `CDataSource` / `CDataObject` / `CDropSource` / `CDropTarget` / `CEnumFormatEtc` | OLE 拖放/剪贴板 |
| `CFileStream` / `CStreamBuf` / `CCompressedStreamBuf` | `IStream` 实现（文件/内存/压缩） |
| `CConPnt` / `CConPntCont` / `CConnectionPoint` / `CEnumConPnts` / `CEnumCons` | 连接点/枚举器 |
| `CVBAControlMgr` / `ExtendedControl` / `ExtendedControlCF` / `ExtendedControl_*` | 控件管理/扩展控件 |
| `CDispDllMod` / `CDispVbaStdMod` | DLL 模块/标准模块的 IDispatch |
| `CSupportErrorInfo` | `ISupportErrorInfo` |
| `CVbaTypeChangeEvents` | 类型变更事件 |
| `CEnumVariant` / `CEnumConnectionPoints` / `CEnumConnections` | 各类枚举器 |
| `CEcTypeInfo` / `CEcTypeComp` / `CEcProjTypeComp` | Ec 类型信息/组件 |
| `WrapTypeInfo` / `WrapTypeLib` / `WrapTypeComp` | 包装器 |
| `CDefnTypeComp` | 定义类型组件 |
| `CExeFile` | EXE 写入接口 |
| `ExecProj` / `ExecMod` | `IVbaProject`/`IVbaModule` |
| `VBACollection` / `VBAErr` / `VBAExtension` / `VBAEnum` | VBA 运行时对象 |

### 8.3 adjustor thunk

清单中有大量 `[thunk]:XXX::AddRef`adjustor{4}` / `{8}` / `{12}``，说明这些类**多重继承多个 COM 接口**（`ITypeLib` + `ICreateTypeLib` + `IVbaProject` 等），需要 this 指针调整 thunk。

---

## 九、基础设施

### 9.1 字符串/值

`valSequence` / `valElement` 值序列；`STRING_CONVERTER` 字符串转换；`BSTR_ARRAY` / `BSTRDATA_ARRAY` BSTR 数组；`Xsz*` 字符串工具；`UnmangleBstr` / `SzMangledOfSz` 名称修饰。

### 9.2 序列化/流

`WriteByte` / `WriteShort` / `WriteLong` / `ReadByte` / `ReadShort` / `ReadLong` / `WriteBstr` / `ReadBstr` / `WriteVariant` / `ReadVariant` 等底层序列化；`CStreamBuf` / `CCompressedStreamBuf` / `CFileStream` 流实现；`SerRead*` / `SerWrite*` EPI 序列化。

### 9.3 国际化/代码页

`oGetLocaleInfo` / `oCompareString` / `oLCMapString` / `oGetStringTypeEx` / `Intl*` / `plcGetLocaleFromLcid` / `GetSystemCP` / `_setmbcp` / `_mbtowc` / `_wctomb`。

### 9.4 CRT

大量标准 CRT 函数（`_malloc`/`_free`/`_realloc`/`_strlen`/`_memmove`/`_qsort`/`_time`/`_sprintf`/`_snprintf`/`_sscanf`/`_atoi`/`_atol`/`_itoa`/`_ltoa`/`_ultoa`/`_strtod`/`_strtol`/`_strtoul`/`_pow`/`_exp`/`_log`/`_floor`/`_modf`/`_sin`/`_cos`/`_tan`/`_atan`/`_sqrt`/`_rand`/`_srand`/`_setjmp3`/`_longjmp`/`_except_handler3`/`_global_unwind2`/`_local_unwind2`/`_purecall`/`_onexit`/`_doexit`/`_cinit`/`_initterm`/`_heap_alloc`/`_heap_init`/`_lock`/`_unlock`/`_mtinit`/`_mtterm`/`_ioinit`/`_ioterm`/`_setargv`/`_setenvp`/`_parse_cmdline`/`_getenv`/`_putenv`/`_searchenv`/`_spawn`/`_exec`/`_system`/`_beginthread`/`_endthread`/`_assert`/`_abort`/`_exit`/`_cexit`/`_c_exit`/`_atexit`/`_cinit`/`_doexit`/`_FF_MSGBANNER`/`_NMSG_WRITE`/`_amsg_exit`/`_callnewh`/`_CxxThrowException`/`_CxxFrameHandler`/`_chkesp`/`_chkstk`/`_alloca_probe`/`_adjust_fdiv`/`_safe_fdiv`/`_fprem`/`_fpatan`/`_fptan`/`_control87`/`_controlfp`/`_clearfp`/`_statusfp`/`_set_statfp`/`_set_exp`/`_decomp`/`_d_inttype`/`_sptype`/`_fpclass`/`_copysign`/`_except1`/`_raise_exc`/`_handle_exc`/`_handle_qnan1`/`_umatherr`/`_matherr`/`_fptrap`/`_abstract_cw`/`_hw_cw`/`_statfp`/`_clrfp`/`_ctrlfp`/`_aullrem`/`_aulldiv`/`_allmul`/`_allshl`/`_allshr`/`_allrem`/`_alldiv`/`_chkstk`/`_global_unwind2`/`_local_unwind2`/`_unwind_handler`/`_abnormal_termination`/`_seh_longjmp_unwind`/`__NLG_*`/`_purecall`/`__purecall`/`_cintrindisp`/`_ctrandisp`/`_trandisp`/`_fload`/`_frnd`/`_ld12cvt`/`_ld12tod`/`_ld12tof`/`__mtold12`/`___multtenpow12`/`___ld12mul`/`___add_12`/`___shl_12`/`___shr_12`/`_rt_probe_read4`/`_get_int_arg`/`_get_int64_arg`/`_get_short_arg`/`_write_char`/`_write_string`/`_write_multi_char`/`_setvbuf`/`_flsbuf`/`_filbuf`/`_fclose`/`_fflush`/`_flush`/`_flushall`/`_flsall`/`_fcloseall`/`_close`/`_close_lk`/`_commit`/`_dup`/`_dup2`/`_eof`/`_ungetc`/`_getc`/`_putc`/`_stdio`/`_iob`/`_popen`/`_pclose`/`_pipe`/`_read`/`_write`/`_open_osfhandle`/`_get_osfhandle`/`_alloc_osfhnd`/`_free_osfhnd`/`_set_osfhnd`/`_lock_fhandle`/`_unlock_fhandle`/`_stat`/`_fstat`/`_access`/`_chmod`/`_umask`/`_mkdir`/`_rmdir`/`_chdir`/`_getcwd`/`_getdcwd`/`_findfirst`/`_findnext`/`_findclose`/`_fullpath`/`_splitpath`/`_makepath`/`_searchenv`/`_getenv`/`_putenv`/`_spawn`/`_exec`/`_system`/`_beginthread`/`_endthread`/`_mutex`/`_initterm`/`_onexit`/`_setjmp`/`_longjmp`/`_assert`/`_abort`/`_exit`/`_cexit`/`_c_exit`/`_atexit`/`_cinit`/`_doexit`/`_FF_MSGBANNER`/`_NMSG_WRITE`/`_amsg_exit`/`_callnewh`/`_new`/`_delete`/`_malloc`/`_free`/`_realloc`/`_calloc`/`_msize`/`_heapadd`/`_heapset`/`_heapchk`/`_heapwalk`/`_nh_malloc`/`_callnewh`/`_CxxThrowException`/`_CxxFrameHandler`/`_purecall`/`_chkesp`/`_chkstk`/`_alloca_probe`/`_adj_fdiv_*`/`_safe_fdiv*`/`_fdiv_*`/`_fprem*`/`_fpatan`/`_fptan`）——说明它静态链接了 CRT，并包含浮点/SEH/C++ 运行时支持。

### 9.5 输出/诊断

`outFile`（`outFileOpen`/`outFileWriteData`/`outFilePatchByte`/`outFileDone`）、`error`、`parserError`、`EberrOfHresult` / `EberrOfScode` / `TiperrOfOFErr`、`ShowCompileError` / `ShowSimpleError` / `ShowErrDelayed` / `SaveErrText` / `GetErrMsg` 错误报告链。

---

## 十、关键设计模式

1. **分层编译管线**：`scanner` → `parser` → `genOpc` → `BASIC_TYPESRC`/`OPCODE_MGR` → `EXMGR::CompileLine`（BIL）→ `TransformToLoBil`（LowBil）→ `gCIL*`/`nativeGen*`（Native）→ `C2.EXE`（.obj）→ `buildProject::FinishEXE`（EXE）。
2. **状态机式模块编译**：`BASIC_TYPEROOT::EnsureIn*State`（Undeclared/SemiDeclared/Declared/SrpBuilt/Compiled/Runnable）驱动增量编译。
3. **多套类型绑定并存**：`DEFN_TYPEBIND` 家族按场景（动态/工程/外部/模拟/非限定/记录）选择不同绑定实现。
4. **COM 多重继承 + adjustor thunk**：`GEN_PROJECT` 同时实现 `ITypeLib`/`ICreateTypeLib`/`IVbaProject`，`CVBAControlMgr`/`CVbaExtReferences`/`VBAErr` 等多接口类需要 adjustor thunk。
5. **多套内存分配器并存**：块/提交/No-RLS/堆/简单/堆块管理器，反映子系统演进合并。
6. **BIL 树 + 生成栈**：`BILALLOC` 分配树节点，`GEN_STACK` 做表达式/控制流栈式生成，`EXTBL` 管理函数代码块和 RTMI。
7. **trap/异常保护**：`__trapCreate`/`_setjmp3`/`__trapThrow`/`__trapRemove` 包裹编译/生成流程。
8. **IDE 与编译共享对象**：`VIEWMGR`/`CPANE`/`BROWSEDATA`/`WATCHMGR` 直接操作 `BASIC_TYPEROOT`/`FEDSYM`/`EXFRAME`，IDE 与编译器深度耦合。
9. **运行时对象模型**：`VBACollection` 用红黑树，`VBAErr` 用线程局部错误状态，`RUN_INSTMGR` 管理对象实例与僵尸实例。

---

## 十一、一句话总结

> **vba5.dll 是 VB6/VBA5 的“编译器 + 运行时 + IDE + 扩展对象模型”一体化 COM DLL**：它用 `scanner/parser/genOpc` 做前端，用 `EXMGR`/`BILALLOC`/`EXGEN` 生成 BIL/LowBil，用 `gCIL*`/`nativeGen*`/`C2.EXE` 生成 Native 代码，用 `buildProject`/`CExeFile` 链接 EXE；用 `BASIC_TYPEROOT`/`DYN_TYPEMEMBERS`/`IMPMGR`/`NAMMGR`/`TYPEMGR` 管理类型与绑定；用 `rtc*`/`__vba*`/`VBACollection`/`VBAErr` 提供运行时；用 `CPANE`/`VIEWMGR`/`BROWSEDATA`/`WATCHMGR` 提供 IDE；用 `CVbaExt*`/`ExecProj`/`ExecMod` 提供 COM 扩展模型；底层由多套内存分配器、流实现、CRT、SEH、浮点支持支撑。
