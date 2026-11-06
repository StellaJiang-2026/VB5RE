# 这份数据是什么

这是一份 **PE 映像的「名字 → 地址 → 是否公开」清单**（`NameAddressPublic.txt`），本质上是 IDA 分析 VBA 运行时 DLL（`VBA5.dll` / `VBA6.dll`，导出表含 `Eb*`/`Tip*`/`rtc*`）后导出的符号表。它覆盖了：

1. **导入函数**（`P` = Public，但其实是外部 DLL 的导入项）
2. **导出函数**（`Eb*`、`Tip*`、`rtc*`、`IID_*`，即 VBA 运行时/IDE 组件导出）
3. **内部代码符号**（`.text` 段里的类方法、全局函数）
4. **数据符号**（vftable、全局变量、字符串、常量表）
5. **CRT 静态库符号**（`_malloc`、`_qsort`、`__setjmp3` 等）

# 一、导入表部分：确认它不是 MFC 程序

前 ~0x165 项是导入项，按 DLL 分组：

| DLL | 数量级 | 典型导入 |
|---|---|---|
| `ADVAPI32` | ~25 | `RegOpenKeyA/W`、`RegQueryValueExA/W`、`RegCreateKeyExA`、`AdjustTokenPrivileges` |
| `GDI32` | ~40 | `CreateCompatibleDC`、`BitBlt`、`StretchBlt`、`TextOutA`、`ExtTextOutA`、`CreateFontIndirectA` |
| `KERNEL32` | ~140 | `HeapAlloc`、`CreateFileMappingA`、`CreateProcessA`、`CreatePipe`、`TlsAlloc`、`MultiByteToWideChar`、`LCMapStringA` |
| `OLEAUT32` | ~60 | `SysAllocString`、`VariantCopy`、`SafeArrayCreate`、`VarBstrFromI4`、`LoadTypeLib`、`RegisterTypeLib` |
| `USER32` | ~200 | `CreateWindowExA`、`RegisterClassA`、`DefWindowProcA`、`PeekMessageA`、`SetWindowsHookExA`、`keybd_event`、`OpenClipboard` |
| `ole32` | ~25 | `CoCreateInstance`、`RegisterDragDrop`、`DoDragDrop`、`OleSetClipboard`、`StgCreateDocfile`、`CreateStreamOnHGlobal` |

**关键判断**：

- **没有 `MFC42.dll` / `mfc*.dll`** → 不是动态 MFC。
- **没有 `Afx*` 符号** → 不是静态 MFC。
- 大量 `A` 版 API（`CreateWindowExA`、`TextOutA`、`RegOpenKeyA`）+ `MultiByteToWideChar`/`LCMapStringA` → 典型 ANSI/DBCS 老式 Win32 程序。
- `OLEAUT32` 全部按**序号**导入（`0FA61350 26`、`201`、`9`…），这是 VB6/VBA 运行时按序号绑定 OLEAUT32 的经典特征。

结论：**原生 Win32 + COM/OLE + 自建 C++ 框架，不是 MFC。**

# 二、导出表部分：VBA 运行时/IDE 的 DLL

导出项集中在 `0FBCAE72` 之后的 `.edata` 区和前面 `Eb*`/`Tip*`/`rtc*` 代码符号：

**宿主/IDE 接口（`Eb*`）**：
`EbInitHost`、`EbTerm`、`EbCloseProject`、`EbExecuteLine`、`EbSetStepMode`、`EbGetCallstackCount`、`EbMemberBeginQuery`、`EbCreateCVbaExtModule`、`EbRegisterVbaHost`、`EbGetVBAObject` …

**项目/模块/类型管理（`Tip*`）**：
`TipCreateProject`、`TipCreateStdModule`、`TipCompileModule`、`TipMakeExe2`、`TipGetFuncInfo`、`TipGetArgType`、`TipAddRefLib`、`TipInvokeMethod` …

**VBA 运行时（`rtc*` / `__vba*` / `rt`）**：
`rtcMsgBox`、`rtcInputBox`、`rtcLeftBstr`、`rtcMidVar`、`rtcDateAdd`、`rtcDDB`、`rtcCreateObject`、`__vbaStrCat`、`__vbaVarAdd`、`rtcFileOpen` …

**COM 对象模型**：
`IID_IVbaProject`、`IID_IVbaModule`、`IID_IVbaHost`、`IID_IVbaProvideStorage`、`CLSID_VbaProject`、`VBAErr`、`VBACollection`、`CVbaExtModule`、`CVbaExtEditor`、`CVbaExtReferences`、`CVbaExtMember` …

**引擎入口**：
`ProcCallEngine`、`MethCallEngine`、`ProcEntry`、`DllFunctionCall` …

这与我们前面分析的 `parser`/`scanner`/`BASIC_TYPEROOT`/`CODEFRAG`/`VALUETIP`/`ExtendedControl` 完全对得上：**这就是 VBA 运行时 DLL（VBA5.dll / VBA6.dll）的符号表。**

# 三、代码段：自建 C++ 框架，不是 MFC

按类分组看 `.text` 符号：

| 模块 | 类/函数 | 作用 |
|---|---|---|
| 编译器前端 | `scanner::*`、`parser::*` | 词法/语法分析 + P-code 生成 |
| 代码生成 | `genOpc::*`、`genOpcodeBOS`、`generatePcode` | AST → P-code |
| 模块模型 | `BASIC_TYPEROOT`、`BASIC_TYPESRC`、`BASIC_TYPEINFO` | 源码/类型/模块聚合根 |
| 类型系统 | `IMPMGR`、`TYPEMGR`、`TYPE_DATA`、`REC_TYPEINFO` | 类型导入、类型数据、UDT |
| 符号系统 | `NAMMGR`、`FEDSYM`、`FEDTYP`、`FEDARG` | 名字哈希、符号包装、类型包装、参数包装 |
| 执行引擎 | `EXMGR`、`EXGEN`、`RUN_INSTMGR`、`INSTMGR` | 表达式生成、运行实例管理、僵尸化 |
| 运行时库 | `rtc*`、`__vba*`、`rt*` | VBA 内建函数 |
| IDE 前端 | `CODEFRAG`、`VALUETIP`、`POPUPMGR`、`NAMELIST`、`MODVWR`、`CPANE`、`LPANE`、`WLPANE`、`OBJBRW` | 代码片段、值提示、弹窗、面板、对象浏览器 |
| 控件宿主 | `ExtendedControl`、`ExtendedControlCF`、`CVBAControlMgr`、`WRAPPER_EVENT_SINK` | ActiveX 控件包装、连接点、事件接收 |
| COM 包装 | `WrapTypeLib`、`WrapTypeInfo`、`WrapTypeComp`、`CEcTypeInfo`、`CEcTypeComp` | 类型库/类型信息/绑定包装 |
| 持久化 | `CFileStream`、`CStreamBuf`、`CCompressedStreamBuf`、`memAlloc`、`memSection`、`SHEAP_MGR`、`BLKMGR32` | 流、压缩流、内存分配、区块管理 |
| 构建/链接 | `buildProject`、`buildModule`、`CExeFile`、`ExecProj`、`ExecMod` | 编译、生成 EXE、资源/重定位 |
| 调试 | `EXFRAME`、`WATCHMGR`、`WATCH`、`BPMGR`、`BMMGR` | 栈帧、监视窗口、断点、书签 |
| 国际化 | `IntlProperWidth`、`DBisIdentBegJapan/Korea/Taiwan/PRC`、`LCMapString`、`GetLocaleInfo` | DBCS 标识符、区域设置 |
| 安全/异常 | `_setjmp3`、`errTrapDesc`、`g_currentErrorTrap`、`StdThrow`、`Catch` | 错误陷阱、异常传播 |
| COM/OLE 基础设施 | `CConPntCont`、`CConPnt`、`CEnumConPnts`、`CEnumCons`、`CDataSource`、`CDropSource`、`CDropTarget`、`CEnumFormatEtc` | 连接点、拖放、剪贴板 |
| 正则/搜索 | `RECompile`、`REMatch`、`REParseRE`、`FindNextOccurrence` | 搜索/替换正则引擎 |

**没有 `Afx*`、`CWinApp`、`CWnd`、`CDocument`、`CView`、`CObject`、`CString`。**  
UI 是自己写的 `CPANE`/`MODVWR`/`PANEMGR`/`OBTBAR` 框架，直接 `RegisterClassA` + `CreateWindowExA` + `WndProc`。

# 四、数据段：大量 vftable 与全局状态

## 1. vftable 清单（COM 接口与自建类）

- 自建类：`CConPntCont`、`CEnumConPnts`、`CConPnt`、`CEnumCons`、`CSupportErrorInfo`、`BASIC_CLASS::PRIVATE_UNKNOWN`、`BASIC_CLASS_ZOMBIE`、`CVBAControlMgr::CONTROL_INFO`、`WRAPPER_EVENT_SINK`、`ExtendedControlCF`、`ExtendedControl::RAW_CTRL_OBJECT`、`ExtendedControl::RAW_CTRL_SINK`、`ExtendedControl::PRIVATE_UNKNOWN`、`WrapTypeLib`、`WrapTypeInfo`、`WrapTypeComp`、`GENPROJ_TYPEBIND`、`DEFN_TYPEBIND`、`DYN_TYPEMEMBERS`、`DYN_TYPEBIND`、`MOCKUP_TYPEBIND`、`EXTERNAL_TYPEBIND`、`UNQUAL_TYPEBIND`、`DYN_BINDNAME_TABLE`、`CExeFile`、`CVbaTypeChangeEvents`、`CDefnTypeComp`、`VBAExtension::PRIVATE_UNKNOWN`、`CEnumVariant`、`CEnumConnectionPoints`、`CConnectionPoint`、`CEnumConnections`、`CFileStream`、`CStreamBuf`、`CCompressedStreamBuf`、`COrpcDebugHandlers`、`VBAErr`、`VBACollection`、`VBAEnum`、`CVbaExtModule`、`CVbaExtEditors`、`CVbaExtEditor`、`CVbaExtReferences`、`CVbaExtReference`、`CVbaExtMembers`、`CVbaExtMember`、`ExprSrv`、`CVBFormat`、`CVBFiles`、`CDispVbaStdMod`、`CDispDllMod`
- 标准 COM 接口的「桩 vftable」：`IUnknown`、`IDispatch`、`IConnectionPointContainer`、`IConnectionPoint`、`ITypeLib`、`ICreateTypeLib`、`ITypeInfo`、`ICreateTypeInfo`、`IProvideClassInfo`、`ISupportErrorInfo`、`IExternalConnection`、`IPersistStorage`、`IDropTarget` 等
- 多继承 adjustor thunk 大量出现（`{for 'IConnectionPoint'}`、`{for 'IConnectionPointContainer'}`、`{for 'IVbaProject'}`、`{for 'ITypeLib'}`、`{for 'IVbaModule'}` 等）

## 2. 关键全局状态

**解析器/扫描器状态**（`0FBE95E0` 附近）：
- `Token`、`Value`、`tokenIsIdent`、`scanHashTbl`、`scannerLCID`、`scanSrcLineTab`、`scanLookAheadToken`、`scannerLANG`
- `parser::parseErrNumber`、`parseErrBegLine`、`parseErrBegCol`、`parseLineFlags`、`parseLineIndent`、`genPcodeBuffNext`、`genPcodeBuffLast`、`genPcodeBuffAddr`、`labelIdent`、`lineNoIdent`、`parseTypeRoot`、`parseHashTbl`
- `g_currentErrorTrap`、`errTrapDesc`

**代码生成表**：
- `stmtExpr2opcode`、`simpleVarRefs`、`simpleMemRefs`、`calledVarRefs`、`calledMemRefs`、`simpleDctRefs`、`calledDctRefs`、`withedMemRefs`、`withedDctRefs`、`withArgMemRefs`、`withArgDictRefs`、`indexRefs`、`caseRelopOpcodes`、`endOpcodes`、`exitOpcodes`、`optionFlags`
- `g_opcode_info`、`g_rgexinfo`、`g_rgbSizeofExok`、`g_rgexcodeLiteral`、`g_rgexcodeCoerce`、`g_rgcoercetbl`、`g_rgoperinfo`、`g_coercetblStr`、`g_rgccinfo`、`g_rgexmcc`

**类型/符号全局**：
- `var_type2_VARTYPE`、`var_type2_AlignSize`、`var_type2_IT`、`var_type2typeDesc`、`g_rgIntrinsicTypes`、`g_tmTypeSizes`、`g_rgcbSizeType`、`g_rgcbAlignment`、`g_rgfBasicIntrinsic`
- `g_rgvdesckindMap`、`g_rgVtypeOfTdesck`、`g_itTdesck`、`g_tdesckIt`、`g_typeCharTdesck`、`g_kwidTdesck`、`g_deftypeKwidTdesck`
- `g_symWorld`、`g_symProjIntrinsics`、`g_symProc`、`g_ptdata`、`g_pbtRoot`、`g_pexmgr`、`g_pimpmgr`、`g_pbilalloc`、`GlobalAllocator`

**UI 全局**：
- `g_pobjbrw`、`g_pbdEditor`、`g_pbdMemberQuery`、`g_hfontNormal`、`g_hfontBold`、`g_hfontStrikeOut`、`g_hbrWindow`、`g_hbrHighlight`、`g_rgpbltter`、`g_pbltterDefault`
- `g_hcrsHSplit`、`g_hcrsVSplit`、`g_hcrsIBeam`、`g_hcrsArrow`、`g_hcrsHand`、`g_cySplitBar`、`g_cxy3dFrame`
- `c_szObtbarWndClass`、`c_szObPaneClass`、`c_szColumnSelectionFormat`、`c_szCurStmtFormat`
- `g_rgbtninfo`、`g_rgBritemsort`、`g_rgBrspecSort`、`g_rgSortinfo`、`g_fSortByType`、`g_fRefinedSort`

**国际化/区域**：
- `g_UserLCID`、`g_SystemLangId`、`g_UserLangId`、`g_iLang`、`g_xchDecimalSep`、`g_xchListSep`、`g_xszListSepSp`
- `g_rgLanguages`、`g_rgcpsforlang`、`_cpidListWin*`、`_cpidListMac*`、`_cpidListDos*`、`g_rgIfcsIn`、`g_rgIfcsLocal`、`ifcsInUS`
- `_DBisIdentBegTab`、`_DBisIdentMidTab`、`_charFlags`、`_charTypes`、`_typeCharChars`、`g_leadByteTable`、`_g_rgchLeadByteTable`

**VBA 错误文本**：
- `g_rgerrTextSubst`、`_rterrs`、`_g_rgusErrorMap`、`_g_rgstrTypeNames`、`_rgszFormatPictures`、`_rglcidGenetive`
- `aVba5Dll`、`aEbcreatecvbaex` 等字符串（导出名前缀 `a` 是 IDA 的 ASCII 字符串命名惯例）

**EXE 构建**：
- `g_pExeFile`、`g_exeroot`、`g_hexepartStart`、`_exeheader`、`_rgsectheader`、`_symboltable`、`_rgbImportPogoTemplate`
- `g_rgchFileInTerm`、`_rgchFileInTerm`、`g_szIni*` 系列（选项 INI 键名）

**COM/OLE 基础设施**：
- `_IID_IUnknown`、`_IID_IDispatch`、`_IID_ITypeInfo`、`_IID_ITypeLib`、`_IID_ITypeComp`、`_IID_ICreateTypeLib`、`_IID_ISupportErrorInfo`、`_IID_IProvideClassInfo`、`_IID_IConnectionPoint`、`_IID_IConnectionPointContainer`、`_IID_IEnumConnectionPoints`、`_IID_IEnumConnections`、`_IID_IStream`、`_IID_IDropSource`、`_IID_IDropTarget`、`_IID_IDataObject`、`_IID_IEnumVARIANT`、`_IID_IPersistFile`、`_IID_IPersistStorage`、`_IID_IExternalConnection`、`_IID_IClassFactory`、`_IID_IClassFactory2`、`_IID_IErrorInfo`、`_IID_IPropertyNotifySink`、`_IID_IProxyManager`、`_GUID_NULL`、`_IID_StdOle`
- `_g_guidVBA`、`_g_szVbaTypeLibFullPath`、`_g_fVbaTypeLibRegisteredProperly`、`_g_hinstSCPDLL`

**CRT 静态库全局**：
- `__environ`、`__wenviron`、`___argv`、`___argc`、`__pgmptr`、`__osver`、`__winver`、`__winmajor`、`__winminor`
- `_iob`、`Stream`、`__cflush`、`__newmode`、`_pctype`、`_pwctype`、`_ctype`、`___mb_cur_max`、`_mbctype`、`_mbcodepage`、`_mblcid`
- `__adjust_fdiv`、`__FPinit`、`__FPmtinit`、`__FPmtterm`、`__ldused`、`__fltused`
- `__aexit_rtn`、`__exitflag`、`_C_Termination_Done`、`_C_Exit_Done`、`__onexitbegin`、`__onexitend`
- `__XcptActTab`、`__First_FPE_Indx`、`__Num_FPE`、`__XcptActTabSize`、`__XcptActTabCount`
- `__NLG_Destination`、`__NLG_Return2`、`__global_unwind2`、`__local_unwind2`、`__except_handler3`
- `_tzstd`、`_tzdst`、`__tzname`、`__timezone`、`__daylight`、`__dstbias`

# 五、从数据能推断的架构结论

## 1. 这是 VBA 运行时 DLL（VBA5/VBA6）
导出名 `EbInitHost`、`EbExecuteLine`、`EbCreateCVbaExtModule`、`TipCreateProject`、`rtcMsgBox`、`ProcCallEngine`、`IID_IVbaProject`、`CLSID_VbaProject` 是 VBA 运行时/IDE 的独有符号。`aVba5Dll` 字符串直接点明。

## 2. 不是 MFC，是自建 C++ + 原生 Win32 + COM
- 无 MFC DLL 导入、无 `Afx*` 符号
- UI 用 `RegisterClassA`/`CreateWindowExA`/`DefWindowProcA`/`WndProc`
- 自建类层次：`CPANE`/`MODVWR`/`PANEMGR`/`LPANE`/`OBTBAR`/`POPUPMGR`/`NAMELIST`/`VALUETIP`/`OBJBRW`
- COM 手写实现：`IUnknown`/`IDispatch`/`IConnectionPoint*`/`ITypeInfo*` 全部自建 vftable + adjustor thunk

## 3. 编译器是传统单遍编译器
- `scanner` → `parser` → `genOpc` → `EXTBL`/`EXMGR` → `buildProject`/`CExeFile`
- 无现代 IR/优化器，P-code 直接生成
- `norls_allocator`/`small_block_allocator`/`large_block_allocator`/`commitAllocator` 是自建内存分配器，非 CRT `malloc`

## 4. 强 DBCS/国际化支持
- `DBisIdentBegJapan/Korea/Taiwan/PRC`、`DBisIdentMid*`
- `LCMapStringA/W`、`CompareStringA/W`、`GetStringTypeA/W`、`IsDBCSLeadByte`
- `g_rgLanguages`、`_cpidListWin/Mac/Dos*` 大量代码页表
- IME 支持：`_pImmGetContext`/`_pImmSetCompositionWindow`/`_pImmGetCompositionString` 等动态加载

## 5. 完整的 IDE 前端
- 代码编辑器：`CPANE`（自绘、行缓存、光标、选择、书签、断点、列选择、自动完成 `POPUPMGR`、值提示 `VALUETIP`、名称列表 `NAMELIST`）
- 对象浏览器：`OBJBRW` + `OBJBRW_PANE` + `OBJBRW_TOOLBAR_PANE` + `BROWSEDATA`
- 监视/调用栈：`WATCHMGR`、`WATCH`、`WLPANE`、`CALLSTACK_ITEM`
- 搜索替换：`FindNextOccurrence` + 自建正则引擎 `RECompile`/`REMatch`
- 撤销/重做：`XACTMGR` + `XACTEL` 事务栈

## 6. 构建/链接全在内部
- `buildProject`/`buildModule` 编译模块
- `CExeFile` 写 PE 文件（`WriteExe`、`NewExePart`、`WriteDllFixup`、`WriteRsrcDir`）
- `g_pExeFile`、`g_exeroot`、`_exeheader`、`_rgsectheader` 是链接器状态

## 7. 运行时执行引擎
- `ProcEntry`/`MethCallEngine`/`ProcCallEngine` 是 P-code 解释器入口
- `EXGEN` 是低层 BIL（Basic Intermediate Language）→ 机器码/P-code 的生成器
- `BILALLOC`/`Transform_*`/`BuildLoTree_*` 是 BIL 树构造与变换
- `RUN_INSTMGR` 管理类实例，`BASIC_CLASS_ZOMBIE`/`Zombie_*` 是中断模式僵尸化机制

## 8. 大量 COM 接口桩
- `_IID_*` 数据符号几乎覆盖了 OLE Automation 全部标准接口
- `CEcTypeInfo`/`CEcTypeComp`/`CEcProjTypeComp`/`WrapTypeInfo`/`WrapTypeLib`/`WrapTypeComp` 是对 `ITypeInfo`/`ITypeLib`/`ITypeComp` 的包装，用于让 VBA 类型系统与 COM 类型库互操作
- `CVbaExtModule`/`CVbaExtEditor`/`CVbaExtReferences`/`CVbaExtMember` 是 VBE 扩展对象模型（`VBA.Extensible`）

# 六、一句话总结

> **这份数据是 VBA 运行时/IDE DLL（VBA5/VBA6）的完整符号表：导入表确认它是原生 Win32 + COM/OLE 程序（无 MFC）；代码段展示了一个自建 C++ 编译器前端（scanner/parser/genOpc）+ 运行时引擎（EXMGR/EXGEN/ProcEntry）+ 自绘 IDE（CPANE/MODVWR/OBJBRW/POPUPMGR/VALUETIP）+ 控件宿主（ExtendedControl/CVBAControlMgr）+ 自建内存分配器（norls/small_block/large_block）+ PE 链接器（CExeFile）+ DBCS 国际化；数据段则包含大量 COM 接口 vftable、opcode/coercion 表、类型/符号全局状态、UI 资源全局、CRT 静态库状态，以及全套 `_IID_*`/`_CLSID_*` GUID。它不是 MFC 程序，也不是普通应用，而是一个完整的、传统单遍编译 + P-code 解释执行的 VBA 运行时与 IDE 实现。**
