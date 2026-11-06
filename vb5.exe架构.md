# VB6 IDE 架构总结（基于 NameAddressPublic.txt 导出符号表分析）

这份文件是 **Visual Basic 6.0 集成开发环境（IDE）** 主程序（VB6.EXE）的导出/符号表，包含约 2 万多个函数、类成员、虚表、全局变量与数据符号。下面从架构维度进行归纳，并给出思维导图。

---

## 一、总体架构判断

| 维度 | 结论 |
|---|---|
| 程序类型 | 32 位 Windows GUI 应用（PE 格式，导入 KERNEL32/USER32/GDI32/ADVAPI32/ole32/OLEAUT32/VBA5.DLL） |
| 编程语言/工具链 | MSVC 6.0（`___xc_a`、`__except_handler3`、`_$E` 异常表、`___initterm` 等 CRT 符号是典型 MSVC 6 特征）+ C++ 类体系 |
| 架构风格 | **分层 + 组件化 + 文档/视图变体 + 脚本宿主**：IDE 外壳 + VBA 编辑器宿主 + 窗体设计器/运行时引擎 + 编译器/链接器 + 工程管理 + 调试器 + 属性浏览器 + 停靠窗口框架 |
| 核心依赖 | VBA5.DLL（VBA 编辑器/类型库/编译器前端）、OLE32/OLEAUT32（COM、TypeLib、自动化）、Office 组件（MSO 工具栏/命令栏/组件管理器） |
| 设计模式特征 | 大量 COM 接口实现（IDispatch/IOleObject/IOleInPlaceSite/IPersist*）、虚表与 `[thunk]` 调整器、引用计数（AddRef/Release）、工厂模式（ClassFactory）、单例（g_ThreadData、Dock_pdockMain 等） |

---

## 二、分层架构

### 1. 启动与线程层
- `WinMain`、`_WinMainCRTStartup`、`InitFMain`、`InitBase`、`InitGlobal`
- `MainHwndCreate` / `MainHwndIDEOwnerCreate` / `MainHwndCreate`
- 线程模型：`CThreadData`、`InitThread`、`g_ThreadData`、`ProjectThread`、`DeskrefThread`、`VBThreadObject`、`InitVBAThread`、`InitUIThread`
- 消息循环：`ThunderMsgLoop`、`CMsoComponent::PushMsgLoop`、`MainFTranslateMessage`、`FTranslateAccelerator`
- 终止：`InitTerminate`、`TerminateCmdbars`、`UninitOffice`、`VBApiCleanup`

### 2. IDE 外壳/UI 层
- 主框架：`FnwpMain`、`FnwpIDEOwner`、`MainUpdateTitle`、`MainActivateNextWindow`
- **停靠窗口框架**：`CDockManager`、`CContainer`、`CPane`、`CSlider`、`DockCreateToolPane/Window`、`_SaveToDockFile`
- **命令栏/工具栏/菜单**：`CMsoUser`、`CMsoToolbarSetUser`、`CMsoToolbarUser`、`CMsoButtonUser`、`CMsoMenuUser`、`CMsoDropdownUser`、`CmdBar*`、`FInitializeCmdbars`、`FLoadAcceleratorTables`
- **属性浏览器**：`Pbrs*`（`_PbrsWP`、`_UpdateEdit`、`_HrAddProps`、`_RegeneratePropList`、`PbrsShow`）
- **对象浏览器**：`HostObjectBrowserDlg`、`BrowGetItem`、`ObwinHicon`
- **工程窗口**：`CProjWin`、`CProjWinItem`、`ProjWin*`
- **工具箱/组件画廊**：`Tool*`、`CompGall*`、`AddControlsDialogProc`
- **调色板**：`CPal*`、`FnwpCpal`、`DrawActiveTc`
- **预览窗口**：`PreviewWin*`、`PVWindowWndProc`
- **状态栏**：`Pbar*`、`CMsoVBProgressLocation`
- **对话框**：`DlgBox*`、`VBDialogBoxParam`、`MessageBoxPVoid`、`FilePrintDialogProc`、`SearchDlgProc`、`WatchDlgProc`、`CallstackDlgProc`、`AttributesDialog`

### 3. 工程/项目管理层
- **Project**：`Project` 类（`newProject`、`fSave`、`OpenText`、`ProcessProperties`、`ReadTextProjFile`、`ErrWriteProjData`、`AddFile`、`AddReference`、`AddOCX`）
- **工程组**：`ProjectGroup`（`Load`、`Save`、`SaveItems`、`IsGroupFile`）
- **工程项**：`Projitem` 及子类 `CProjitemModule`、`CProjitemClass`、`CProjitemForm`、`CProjitemDesigner`、`CProjitemUserControl`、`CProjitemPropertyPage`、`CProjitemDocObject`、`CProjitemOCX`、`CProjitemRef`、`CProjitemTypelibRef`、`CProjitemRes`、`CProjitemRelDoc`
- **工程配置**：`CProjProfiles`、`CProjProfSection`、`CProjProfKeyVal`、`Project::SetProjKeyValue`、`ReadProfileString`
- **文件抽象**：`CFileRep`、`FILEREF`（实现 IStream）、`OpenLongFile`、`HrFrefOpen`
- **MRU/模板**：`ProjFSetMRUMenuItems`、`_PromptNewProject`、`_ProcessTemplate`、`WizLoadAndRunFile`

### 4. 脚本宿主 / VBA 集成层（Host ↔ VBA5.DLL）
- **宿主接口实现**：`CVbaHost`（`IVbaHost`、`IVbaProvideStorage`、`IExpressionService`、`IVbaModule`、`IVbaProject`、`IVbaExtModule`、`IVbaExtReferences`、`IVbaExtEditors`）
- **Eb\* API 集**：`EbInitHost`、`EbInitHostPart2`、`EbMember*`、`EbSearch*`、`EbGetCallstack*`、`EbShowCode`、`EbSetMode`、`EbGetOptions`、`EbGetVBAObject`、`EbTerm` 等约 150 个宿主回调
- **Tip\* API 集**：`TipCreateProject`、`TipCompileProject`、`TipMakeExe2`、`TipGetFunctionId`、`TipGetArgType`、`TipAddRefLib`、`TipSetProjectDest` 等约 150 个编译器/类型库回调
- **Host\* 回调**：`HostGetBaseClassTypeInfo`、`HostCreateInstance`、`HostRegisterClassInformation`、`HostShowModule`、`HostLoadInstance`、`HostGetTextFile`、`HostDisplayHelp`、`HostEnterBreakMode` 等
- **VBA 编辑器宿主**：`VbaWindowWndProc`、`VBACreateWindow`、`VBACreateDebugPanes`、`HostCreateWindowImpl`、`HostCreateDebugWindow`
- **类型库/类型信息**：`InMemTypeLib`（实现 ITypeLib2）、`MdlTypeInfo`、`EventTypeInfo`、`PropEvtTypeInfo`、`CtlArrayTypeInfo`、`ExtenderMdlTypeInfo`、`DelegatorTypeInfo`、`CTypeInfoMerged`、`CTypeInfoOcxEvt`、`StubTypeInfo`、`BaseTypeInfo`
- **VBA 对象模型**：`CVBEobj`、`CVBApp`、`VBProjects`、`VBProject`、`Components`、`Component`、`VBComponents`、`VBComponent`、`VBProperties`、`VBProperty`、`VBWindows`、`VBWindow`、`VBLinkedWindows`、`VBAddInManager`、`VBAddIn`、`CVBEvents`、`CProjectsEvents`、`CComponentsEvents`、`CCControlsEvents`、`CSControlsEvents`、`CFileEvents`、`CReferencesEvents`、`CommandBarEvents`

### 5. 窗体设计器 / 运行时 UI 引擎
- **DESK/CTL 对象模型**：`DESK`（窗体）、`CTL`（控件基类）、`CCtlMap`（控件映射）、`CTLMAPENT`（控件映射项）、`CTLSCOLL`、`CDeskFrame`
- **控件实现**（每种控件一个类 + 虚表 + 属性表 + 事件表 + 方法表）：`FORM`、`MDI`、`APP`、`CLIP`、`SCRN`、`PRINT`/`CTLPRINT`、`LABEL`、`PUSH`、`EDIT`、`LBOX`、`COMBO`、`CHECK`、`OPT`、`FRAME`、`SCROLL`/`HSCROLL`/`VSCROLL`、`TIMER`、`IMG`、`PIX`、`SHAPE`、`LINE`、`MENU`/`CTLMENU`、`DIR`、`DRIVE`、`FILE`、`CLIENT`（OLE 容器）、`COcx`（OCX 宿主）、`QUERY`（Data 控件）、`CDFrame`（设计器框架）、`CtlUser`（用户控件）、`CtlDoc`、`CtlPage`
- **绘制/图形**：`Graf*`（`GrafMethLine`、`GrafMethCircle`、`GrafPaintPicture`、`GrafSetPen`、`GrafScale*`）、`VBDRAWCONTEXT`、`Br*`（画刷/画笔）、`Font*`、`Pict*`
- **属性/事件/方法派发**：`Prop*`（`PropGetCtlProp`、`PropSetCtlProp`、`PropLoadProp`、`PropSaveProp`）、`Evt*`（`EvtErrFire`、`EvtLpevtOfId`）、`Meth*`（`MethCallMethod`、`MethLpmethOfHxmodImeth`）、`Ref*`（`RefGetProp2`、`RefSetPropNormal`、`RefCallMethFromVtable`）
- **控件注册/类工厂**：`MdlRegisterInternalCtls`、`MdlHxmodRegisterInternal`、`VtableCreateCtl`、`CreateNewThunk`、`ThunkMake`、`FindThunk`、`MdlHxmod*`
- **窗口过程**：`FormCtlProc`、`StdCtlWndProc`、`CommonGizWndProc`、`LabelCtlProc`、`EditCtlProc`、`LBoxCtlProc`、`ComboCtlProc`、`CheckCtlProc`、`OptCtlProc`、`PushCtlProc`、`TimerCtlProc`、`ImgCtlProc`、`PixCtlProc`、`ShapeCtlProc`、`LineCtlProc`、`DirCtlProc`、`DriveCtlProc`、`FileCtlProc`、`QueryCtlProc`、`MenuCtlProc`、`ClientCtlProc`、`MDICtlProc`、`FrameCtlProc`、`ScrollCtlProc`、`PrintCtlProc`、`AppCtlProc`、`ScrnCtlProc`

### 6. OLE / ActiveX / COM 层
- **OLE 容器**：`COleClientObj`（`IOleClientSite`、`IOleInPlaceSite`、`IAdviseSink`、`IDispatch`、`IOleControlSite`、`IBoundObjectSite`）、`CPropNotifySink`、`CEvt`、`CAmbient`、`CReverse`
- **OCX 宿主**：`COcx`（`IOleObject`、`IOleInPlaceObject`、`IOleControl`、`IPersistStreamInit`、`IPersistPropertyBag`、`IViewObject2`、`IDataObject`、`IServiceProvider`、`IQuickActivate` 等）
- **OCX 缓存/类型库**：`OleCCreateOCXCache`、`OleCLoadOCXCache`、`_CacheOcxFill`、`COnDiskTLib`、`OcxRef`、`ControlOcxRef`、`ServerOcxRef`
- **OLE 通用工具**：`OleStd*`（`OleStdGetData`、`OleStdCreateStorageOnHGlobal`、`OleStdGetObjectDescriptorData`）、`OleC*`（`OleCGetDisp`、`OleSetLong`、`OleCFEnabled`、`OleCQI`）
- **数据拖放**：`ODDDataObject`、`ODDDropSource`、`CDropTarget`、`VBDataObject`、`VBDataObjectFiles`、`Odd_*`
- **属性包**：`CPropertySet`、`CProperty`、`CDictionary`、`CPropertyBag`、`BagOnAStream`、`VBPropBag`、`ErrLoadFromPropset`
- **设计器/文档对象**：`DESKUSER`、`DESKOLE`、`DESKDOC`、`DESKPAGE`、`DFRAMEDESK`、`CDFrame`、`CDocObjHwnd`
- **类工厂/注册**：`CPrivClassFactory`、`CPubClassFactory`、`COAServer`、`VBClassFactory`、`CVBServer`、`_DoRegistration`、`_DoTlRegistration`、`COAServer::ErrRegister`
- **Hyperlink/Async**：`VBHyperlink`、`VBAsyncProperty`、`VBDataBindings`、`VBDataBinding`

### 7. 编译 / 链接 / 生成 EXE 层
- **编译入口**：`TipCompileProject`、`TipCompileProjectFull`、`TipCompileModule`、`TipMakeExe2`、`TipFinishExe2`、`TipAbortExe`、`TipStartMakeExe`
- **EXE 生成**：`ExeMake`、`ExeNew`、`CExeFile`（`WriteExe`、`WriteExePart`、`NewExePart`、`WriteRsrcDir`、`CalcMaxSize`、`WriteDllFixup`、`ApplyFixup`）、`ExeNameTable`、`ExeWrite`、`ExeFinish`
- **资源写入**：`RsrcWriteToExe`、`_WriteVerResource`、`_CreateGroupIconResource`、`_WriteRubyExeData`
- **注册/版本**：`ErrSrvRegisterExe`、`CRegData::WriteRegFile`、`Project::SetVer`、`_CheckLibRegistration`
- **启动代码/Thunk**：`abStartUpCode`、`abStartUpCodeDll`、`CreateNewThunk`、`ThunkMake`、`ThunkSetExecutableState`

### 8. 调试器层
- **断点/步进**：`DbgBreak`、`DbgContinue`、`DbgSStep`、`DbgStart`、`DbgReset`、`DbgResetAndStart`、`EbSetStepMode`、`EbGetStepMode`、`EbFExitBreakMode`、`EnterBreak`、`ExitBreak`
- **调用栈**：`EbGetCallstackCount`、`EbGetCallstackFunction`、`EbGetDebWndCallstackIndex`、`DbgUpdateCallstack`、`CallstackDlgProc`、`ErrGetCallstackString`
- **监视/即时窗口**：`HostWatchDlg`、`HostInstantWatchDlg`、`EbIsWatchExprOk`、`EbWatchDlgModSelected`
- **异常**：`EbRaiseException`、`EbShowException`、`EbExceptionHandled`、`EbCopyExceptionInfo`、`HresultRichErrorFromException`

### 9. 错误/本地化/工具层
- **错误处理**：`Error`、`ErrorHr`、`ErrorRemember*`、`GetErrMsg`、`HrFromDosError`、`ErrFromRegError`、`_HrFromErr`、`IdErrMsgHr`、`HresultRichError`
- **国际化**：`Intl*`（`IntlInitializeTable`、`IntlLpstrStringOfID`、`LoadIntlString`）、`LoadLangDLL`、`LocalizeFilename`、`GetIntlExtension`、`JP*`（IME：`JPActIME`、`JPSetIMEMode`、`JPGetImmContext`）
- **字符串/内存**：`StrHszMakeFrom*`、`AllocBstr*`、`CHSZ`、`AnsiBuffer`、`WideBuffer`、`CMemStream`、`DblLstBase`/`DblLstElm`（双向链表）、`Carr*`（动态数组 `tagCARR`）
- **文件/路径**：`CFileRep`、`FILEREF`、`GetLongPathName`、`DetermineDosPathNameType`、`_GetCacheFileName`、`RbyCopyFile`
- **注册表/INI**：`RbyGetProfileString`、`RbySetProfileString`、`RegOpenKeyExA`、`HkeyOpenClsid`、`ProjFSetMRUMenuItems`

### 10. CRT / 运行时库层
- MSVC 6 CRT：`_malloc`、`_free`、`_realloc`、`_strlen`、`_strcpy`、`_strcat`、`_strstr`、`_atoi`、`_atol`、`_atof`、`_qsort`、`_bsearch`、`_mktime`、`_localtime`、`_time`、`_srand`、`_rand`、`__except_handler3`、`__global_unwind2`、`__local_unwind2`、`__chkstk`、`__ftol`、`__cfltcvt`、`__heap_init`、`__ioinit`、`__setargv`、`__setenvp`
- 浮点/十进制转换：`___strgtold12`、`___mtold12`、`___ld12mul`、`__ld12tod`、`__ld12tof`、`_atodbl`、`__atoflt`、`__fltout2`、`_$I10_OUTPUT`

---

## 三、思维导图

```
VB6 IDE (VB6.EXE) 架构
│
├── 1. 启动与线程层
│   ├── 入口: WinMain / _WinMainCRTStartup / InitFMain / InitBase
│   ├── 主窗口: MainHwndCreate / MainHwndIDEOwnerCreate
│   ├── 线程数据: CThreadData / g_ThreadData / InitThread
│   ├── 线程对象: ProjectThread / DeskrefThread / VBThreadObject
│   ├── 线程初始化: InitVBAThread / InitUIThread / InitThread
│   ├── 消息循环: ThunderMsgLoop / CMsoComponent::PushMsgLoop
│   ├── 消息过滤: CMsgFilter (IMessageFilter) / MainFTranslateMessage
│   └── 终止: InitTerminate / TerminateCmdbars / UninitOffice / VBApiCleanup
│
├── 2. IDE 外壳 / UI 层
│   ├── 主框架窗口: FnwpMain / FnwpIDEOwner / MainUpdateTitle
│   ├── 停靠窗口框架
│   │   ├── CDockManager / CContainer / CPane / CSlider
│   │   ├── DockCreateToolPane / DockCreateToolWindow
│   │   └── _SaveToDockFile / CDockManager::LoadUI
│   ├── 命令栏 / 工具栏 / 菜单
│   │   ├── CMsoUser / CMsoToolbarSetUser / CMsoToolbarUser
│   │   ├── CMsoButtonUser / CMsoMenuUser / CMsoDropdownUser
│   │   ├── CmdBar* (GetMenuItemID/ModifyMenu/CheckMenuItem...)
│   │   └── FInitializeCmdbars / FLoadAcceleratorTables
│   ├── 属性浏览器 Pbrs*
│   │   ├── _UpdateEdit / _HrAddProps / _RegeneratePropList
│   │   ├── _PbrsWP / _PbrsWPButtonFixer / _PbrsWPListFixer
│   │   └── PbrsShow / PbrsTrackPropChange / PbrsFKey
│   ├── 对象浏览器: HostObjectBrowserDlg / BrowGetItem / ObwinHicon
│   ├── 工程窗口: CProjWin / CProjWinItem / ProjWinSetItemText
│   ├── 工具箱 / 组件画廊: Tool* / CompGall* / AddControlsDialogProc
│   ├── 调色板: CPal* / FnwpCpal / DrawActiveTc / SetCustomColor
│   ├── 预览窗口: PreviewWin* / PVWindowWndProc / FDrawEntirePreviewWindow
│   ├── 状态栏: Pbar* / CMsoVBProgressLocation
│   ├── 进度条: StartProgressBar / UpdateProgressBar / CloseProgressBar
│   └── 对话框
│       ├── DlgBox* / VBDialogBoxParam / MessageBoxPVoid
│       ├── 文件对话框: DlgIdFileOpen / DlgFFileSave / DlgFileNew
│       ├── 搜索: SearchDlgProc / EndSearch / FillEBSrchDlg
│       ├── 监视: WatchDlgProc / InstantWatchDlgProc
│       ├── 调用栈: CallstackDlgProc
│       ├── 过程属性: AttributesDialog / AttributesDlgProc
│       ├── 引用: AddRefDlgProc / AddRefInit / AddNewReference
│       ├── 选项: EditorFormatDlgProc / GeneralOptionsDlgProc / PrefSetUpOptions
│       └── 工程选项: ProjOptsShowOptions / ProjectComponentPropsDlgProc
│
├── 3. 工程 / 项目管理层
│   ├── Project 类
│   │   ├── 生命周期: newProject / OpenText / fSave / ErrWriteProjData
│   │   ├── 读取: ReadTextProjFile / ProcessProperties / ReadProjitemsFromText
│   │   ├── 添加: AddFile / AddReference / AddOCX / NewModule / NewForm
│   │   ├── 配置: SetProjKeyValue / ReadProfileString / ProcessType
│   │   └── 编译信息: GetCompileInfo / SetCompileInfo / GetVer / SetVer
│   ├── ProjectGroup (工程组): Load / Save / SaveItems / IsGroupFile
│   ├── Projitem 及子类
│   │   ├── CProjitemModule / CProjitemClass
│   │   ├── CProjitemForm / CProjitemDesigner
│   │   ├── CProjitemUserControl / CProjitemPropertyPage / CProjitemDocObject
│   │   ├── CProjitemOCX / CProjitemRef / CProjitemTypelibRef
│   │   ├── CProjitemRes / CProjitemRelDoc / CProjitemVisual
│   │   └── 通用: LoadFromFile / SaveText / DisconnectFromFile / CanRemoveFromProject
│   ├── 工程配置 CProjProfiles / CProjProfSection / CProjProfKeyVal
│   ├── 文件抽象: CFileRep / FILEREF (IStream) / OpenLongFile / HrFrefOpen
│   └── 模板 / MRU: ProjFSetMRUMenuItems / _PromptNewProject / WizLoadAndRunFile
│
├── 4. 脚本宿主 / VBA 集成层 (Host ↔ VBA5.DLL)
│   ├── 宿主实现: CVbaHost (IVbaHost / IVbaProvideStorage / IExpressionService)
│   ├── Eb* API (~150 个)
│   │   ├── 初始化: EbInitHost / EbInitHostPart2 / EbInitThread / EbTerm
│   │   ├── 成员查询: EbMemberBeginQuery / EbMemberGet* / EbMemberSet*
│   │   ├── 搜索: EbSearchGetCurState / EbSearchFind / EbSearchReplace
│   │   ├── 代码窗口: EbShowCode / EbHideCode / EbGetInsertionPoint
│   │   ├── 调试: EbGetCallstack* / EbSetStepMode / EbRaiseException
│   │   ├── 选项: EbGetOptions / EbSetOptions / EbCheckOptions
│   │   └── 扩展: EbCreateCVbaExtModule / EbCreateCVbaExtReferences
│   ├── Tip* API (~150 个)
│   │   ├── 工程: TipCreateProject / TipSetProjectDest / TipSetProjName
│   │   ├── 模块: TipCreateStdModule / TipCreateClass / TipDeleteModule
│   │   ├── 编译: TipCompileProject / TipCompileProjectFull / TipCompileModule
│   │   ├── 生成: TipStartMakeExe / TipMakeExe2 / TipFinishExe2 / TipAbortExe
│   │   ├── 函数: TipGetFunctionId / TipGetFuncInfo / TipGetArgCount / TipGetArgType
│   │   ├── 引用: TipAddRefLib / TipRemoveRefLib / TipReplaceRefLib
│   │   └── 选项: TipGetOption / TipSetOption / TipGetOption
│   ├── Host* 回调
│   │   ├── 类型: HostGetBaseClassTypeInfo / HostRegisterClassInformation
│   │   ├── 实例: HostCreateInstance / HostLoadInstance / HostGetVBAObject
│   │   ├── 窗口: HostCreateWindow / HostShowWindow / HostActivateWindow
│   │   ├── 文本: HostGetTextFile / HostNotifyTextLoadDone
│   │   ├── 帮助: HostDisplayHelp / HostQueryEbHelpFile
│   │   └── 调试: HostEnterBreakMode / HostTestUserInterrupt
│   ├── VBA 编辑器窗口: VbaWindowWndProc / VBACreateWindow / VBACreateDebugPanes
│   ├── 类型库 / 类型信息
│   │   ├── InMemTypeLib (ITypeLib2) / CreateInMemoryTypeLib
│   │   ├── MdlTypeInfo / EventTypeInfo / PropEvtTypeInfo / CtlArrayTypeInfo
│   │   ├── ExtenderMdlTypeInfo / DelegatorTypeInfo
│   │   ├── CTypeInfoMerged / CTypeInfoOcxEvt / StubTypeInfo / BaseTypeInfo
│   │   └── OnDiskTypeLib: COnDiskTLib / CreateOnDiskTypeLib
│   ├── VBA 对象模型
│   │   ├── 顶层: CVBEobj / CVBApp / CVBEobj::CreateEditors
│   │   ├── 工程集合: VBProjects / VBProject / VBProjectTemplate
│   │   ├── 组件: Components / Component / VBComponents / VBComponent
│   │   ├── 属性: VBProperties / VBProperty / CVBProperties / CVBProperty
│   │   ├── 窗口: VBWindows / VBWindow / VBLinkedWindows / CWindowElem
│   │   ├── 外接程序: VBAddInManager / VBAddIn / CVBAddIn
│   │   ├── 事件: CVBEvents / CProjectsEvents / CComponentsEvents / CCControlsEvents / CSControlsEvents / CFileEvents / CReferencesEvents / CommandBarEvents
│   │   └── 连接点: VBProjects_ItemCnxtnPt / VBComponents_ItemCnxtnPt / CReferencesEvents_ItemCPt
│   └── 窗体模板: VBFormTemplate / CVBForm / VBControlTemplates / VBControlTemplate / CVBFormControls / CVBSelectedControls / CVBContainedControls
│
├── 5. 窗体设计器 / 运行时 UI 引擎
│   ├── 核心对象模型
│   │   ├── DESK (窗体) / CTL (控件基类) / CCtlMap / CTLMAPENT / CTLSCOLL
│   │   ├── CDeskFrame / CtlArrayTypeInfo / tagCARR (控件数组)
│   │   └── DblLstBase / DblLstElm (双向链表)
│   ├── 控件实现 (类 + 虚表 + 属性/事件/方法表)
│   │   ├── 容器类: FORM / MDI / FRAME / CDFrame / CtlUser / CtlDoc / CtlPage
│   │   ├── 标准控件: LABEL / PUSH / EDIT / LBOX / COMBO / CHECK / OPT
│   │   ├── 图形控件: IMG / PIX / SHAPE / LINE
│   │   ├── 滚动条: SCROLL / HSCROLL / VSCROLL
│   │   ├── 文件系统: DIR / DRIVE / FILE
│   │   ├── 其他: TIMER / MENU / CTLMENU / CLIP / APP / SCRN / PRINT / CTLPRINT
│   │   ├── OLE/OCX: CLIENT / COcx / QUERY (Data 控件)
│   │   └── 属性表: Form_rgbpprop / Edit_rgbpprop / ... / 事件表 Form_rgbpevt / 方法表 Form_rgpmeth
│   ├── 绘制 / 图形
│   │   ├── Graf*: GrafMethLine / GrafMethCircle / GrafPaintPicture / GrafSetPen
│   │   ├── GrafScale* / VBDRAWCONTEXT / Realize*
│   │   ├── 画刷/画笔: BrHbrushCreate / BrHpenCreate / BrPurge
│   │   └── 字体: Font* / FontErrLoad / FontErrSave / CFontNotification
│   ├── 属性 / 事件 / 方法派发
│   │   ├── Prop*: PropGetCtlProp / PropSetCtlProp / PropLoadProp / PropSaveProp
│   │   ├── Evt*: EvtErrFire / EvtLpevtOfId / EvtFWillFire / InvokeEvent
│   │   ├── Meth*: MethCallMethod / MethLpmethOfHxmodImeth
│   │   └── Ref*: RefGetProp2 / RefSetPropNormal / RefCallMethFromVtable
│   ├── 控件注册 / 类工厂 / Thunk
│   │   ├── MdlRegisterInternalCtls / MdlHxmodRegisterInternal / MdlHxmod*
│   │   ├── VtableCreateCtl / VtableCreateUnloaded / VtableInstall
│   │   ├── CreateNewThunk / ThunkMake / FindThunk / ThunkSetExecutableState
│   │   └── MDL 结构 / mdlForm / mdlApp / mdlClip / mdlMDI / ...
│   ├── 窗口过程
│   │   ├── FormCtlProc / StdCtlWndProc / CommonGizWndProc
│   │   ├── LabelCtlProc / EditCtlProc / LBoxCtlProc / ComboCtlProc
│   │   ├── CheckCtlProc / OptCtlProc / PushCtlProc / TimerCtlProc
│   │   ├── ImgCtlProc / PixCtlProc / ShapeCtlProc / LineCtlProc
│   │   ├── DirCtlProc / DriveCtlProc / FileCtlProc / QueryCtlProc
│   │   ├── MenuCtlProc / ClientCtlProc / MDICtlProc / FrameCtlProc
│   │   └── ScrollCtlProc / PrintCtlProc / AppCtlProc / ScrnCtlProc
│   ├── 设计器交互
│   │   ├── Ed*: EdHctlCreate / EdFDeleteCtl / EdFSelectCtl / EdSnapPos
│   │   ├── Flex*: FlexSetupGrid / FlexFInstall / FlexUpdateZ
│   │   ├── Grab: _GrabLDown / _GrabMouseMove / _GrabLUp
│   │   └── 对齐: Align* / FormatAlign* / FormatSpace*
│   └── 数据绑定: VBDataBindings / VBDataBinding / BoundInitiate / DBInitDataAccessStruct
│
├── 6. OLE / ActiveX / COM 层
│   ├── OLE 容器: COleClientObj / CPropNotifySink / CEvt / CAmbient / CReverse
│   │   ├── 接口: IOleClientSite / IOleInPlaceSite / IAdviseSink / IOleControlSite / IBoundObjectSite
│   │   └── 方法: FNewClio / FInitOleCtl / DestroyOleCtl / OleCQI / OleCGetDisp
│   ├── OCX 宿主: COcx
│   │   ├── 接口: IOleObject / IOleInPlaceObject / IOleControl / IPersist* / IViewObject2 / IDataObject / IQuickActivate / IServiceProvider
│   │   ├── 状态机: OC_STATE / TransitionUp / TransitionDown / InPlaceActivate / InPlaceDeactivate
│   │   └── Site: COcx::CSite / CEventSink / GetWindowContext / GetDC / InvalidateRect
│   ├── OCX 缓存 / 类型库
│   │   ├── OleCCreateOCXCache / OleCLoadOCXCache / _CacheOcxFill
│   │   ├── OcxRef / ControlOcxRef / ServerOcxRef
│   │   └── COnDiskTLib / CreateOnDiskTypeLib / AddTypeInfo
│   ├── OLE 工具: OleStd* / OleC* / OleTranslateColor / OleCreateFontIndirect
│   ├── 数据拖放
│   │   ├── ODDDataObject / ODDDropSource / CDropTarget
│   │   ├── VBDataObject / VBDataObjectFiles / CEnumVariant
│   │   └── Odd_*: Odd_OnDragDrop / Odd_OnDragOver / Odd_StartDragAttempt
│   ├── 属性包: CPropertySet / CProperty / CDictionary / CPropertyBag / BagOnAStream / VBPropBag
│   ├── 设计器/文档对象
│   │   ├── DESKUSER / DESKOLE / DESKDOC / DESKPAGE / DFRAMEDESK
│   │   ├── CDFrame / CProjitemDesigner / CDocObjHwnd
│   │   └── 接口: IActiveDesigner / IDeskOle / IDesk / IOleDocument / IPropertyPage2
│   ├── 类工厂 / 注册 / 服务
│   │   ├── CPrivClassFactory / CPubClassFactory / VBClassFactory
│   │   ├── COAServer / CVBServer / _DoRegistration / _DoTlRegistration
│   │   ├── CProfferService / CSimpleSite / CVbeProjItem
│   │   └── CMsgFilter / CMsoComponent / CMsoCMHandler
│   ├── Hyperlink / Async / DataBinding
│   │   ├── VBHyperlink / Dispatch<Hyperlink>
│   │   ├── VBAsyncProperty / Dispatch<AsyncProperty>
│   │   └── VBDataBindings / VBDataBinding / Dispatch<DataBinding*>
│   └── 自动化派发
│       ├── VBDispatch / DispatchBase / Dispatch<T>
│       ├── Dispatch<AmbientProperties> / Dispatch<PropertyBag> / Dispatch<DataObject>
│       └── Dispatch<SelectedControls> / Dispatch<ContainedControls> / Dispatch<ParentControls>
│
├── 7. 编译 / 链接 / 生成 EXE 层
│   ├── 编译入口: TipCompileProject / TipCompileProjectFull / TipCompileModule
│   ├── 生成流程: TipStartMakeExe / TipMakeExe2 / TipFinishExe2 / TipAbortExe
│   ├── ExeMake / ExeNew / ExeWrite / ExeFinish
│   ├── CExeFile
│   │   ├── WriteExe / WriteExePart / NewExePart / NewResourcePart
│   │   ├── WriteRsrcDir / WriteNames / EndSection / CalcMaxSize
│   │   ├── WriteDllFixup / WriteInternalFixup / WriteBSSFixup / ApplyFixup
│   │   └── AllocBSS / SetBaseAddress / CreatePublicName
│   ├── ExeNameTable: HnameOfLpsz / HnameLookupLpsz / WriteNames / CalcOffsets
│   ├── 资源: RsrcWriteToExe / _WriteVerResource / _CreateGroupIconResource
│   ├── 注册/版本: ErrSrvRegisterExe / CRegData::WriteRegFile / Project::SetVer
│   ├── 启动代码/Thunk: abStartUpCode / abStartUpCodeDll / CreateNewThunk / ThunkMake
│   └── 组生成: MakeGroup / _MakeGroup / _BuildProjects / _PromptBuildProjects
│
├── 8. 调试器层
│   ├── 断点/步进: DbgBreak / DbgContinue / DbgSStep / DbgStart / DbgReset
│   ├── 步进模式: EbSetStepMode / EbGetStepMode / EbFExitBreakMode
│   ├── 断点模式: EnterBreak / ExitBreak / Mode_break / STEPMODE
│   ├── 调用栈: EbGetCallstackCount / EbGetCallstackFunction / DbgUpdateCallstack
│   ├── 监视: HostWatchDlg / HostInstantWatchDlg / EbIsWatchExprOk
│   ├── 异常: EbRaiseException / EbShowException / EbExceptionHandled
│   └── 调试窗口: VBACreateDebugPanes / HostCreateDebugWindow / CreateDebugToolbar
│
├── 9. 错误 / 本地化 / 工具层
│   ├── 错误处理
│   │   ├── Error / ErrorHr / ErrorRemember* / ErrorRememberExcepinfo
│   │   ├── GetErrMsg / IdErrMsgHr / HrFromDosError / ErrFromRegError
│   │   ├── HresultRichError / HresultRichErrorHr / ErrSetRichErrorInfoFromDaoError
│   │   └── _HrFromErr / ErrFVbaErr / ErrFThunErr / ErrFThunOrVbaErr
│   ├── 国际化
│   │   ├── Intl*: IntlInitializeTable / IntlLpstrStringOfID / LoadIntlString
│   │   ├── LoadLangDLL / LocalizeFilename / GetIntlExtension
│   │   ├── 字符串表: IntlfAllocStringTable / IntlInitializeStrings
│   │   └── IME: JP* / JPSetIMEMode / JPGetImmContext / CImeKey
│   ├── 字符串 / 内存
│   │   ├── StrHszMakeFrom* / AllocBstr* / AllocBstrLenWfromA / CHSZ
│   │   ├── AnsiBuffer / WideBuffer / CMemStream
│   │   ├── DblLstBase / DblLstElm (双向链表)
│   │   └── tagCARR / CarrHcarrCreate / CarrErrAdd / CarrErrRemove
│   ├── 文件 / 路径
│   │   ├── CFileRep / FILEREF / OpenLongFile / HrFrefOpen / FrefClose
│   │   ├── GetLongPathName / DetermineDosPathNameType / _GetCacheFileName
│   │   └── RbyCopyFile / RbyAppendFromFile / RbyKillFile
│   ├── 注册表 / INI
│   │   ├── RbyGetProfileString / RbySetProfileString / RbyGetProfileInt
│   │   ├── RegOpenKeyExA / RegQueryValueExA / HkeyOpenClsid
│   │   └── ProjFSetMRUMenuItems / ProjFGetMRUItem / ProjFSelectMRUItem
│   └── 杂项工具
│       ├── Misc*: MiscDraw3DFrame / MiscMoveWindowIntoView / MiscStrTok
│       ├── Rby*: RbyExtractFileName / RbyGetNextLong / RbyGetFileLength
│       ├── 排序: _qsort / _shortsort / _swap
│       └── 字符串格式化: _itoa / __ltoa / __ultoa / __itoa
│
└── 10. CRT / 运行时库层 (MSVC 6.0)
    ├── 启动: _WinMainCRTStartup / __heap_init / __ioinit / __setargv / __setenvp
    ├── 内存: _malloc / _free / _realloc / __nh_malloc / __heap_alloc
    ├── 字符串: _strlen / _strcpy / _strcat / _strstr / _strchr / _strncmp / _strnicmp
    ├── 数字转换: _atoi / _atol / _atof / _strtoul / _itoa / __ltoa / __ultoa
    ├── 搜索/排序: _qsort / _bsearch
    ├── 时间: _time / _mktime / _localtime / _gmtime / _ctime / _asctime
    ├── 浮点/十进制: ___strgtold12 / ___mtold12 / ___ld12mul / __ld12tod / __fltout2
    ├── 异常: __except_handler3 / __global_unwind2 / __local_unwind2 / _Catch / Catch
    ├── 文件 I/O: __write / __read / __close / __lseek / __commit / _fclose
    ├── 线程/锁: __lock / __unlock / __lockexit / __unlockexit / _TlsAlloc
    └── 进程/线程: _exit / _doexit / ___onexitinit / _atexit / __onexit
```

---

## 四、关键架构特征总结

1. **双引擎结构**：IDE 外壳（C++ 类体系）与 VBA 脚本引擎（VBA5.DLL）通过 `Eb*`/`Tip*`/`Host*` 三组 C 风格回调 API 紧密耦合，形成“宿主-脚本”架构。

2. **COM 无处不在**：几乎所有 UI 对象（窗体、控件、工程、组件、属性、窗口）都实现 `IDispatch`，并大量实现 OLE 容器/站点/持久化接口，以支持 ActiveX 控件嵌入与属性包持久化。

3. **元数据驱动**：每种控件由 `MDL`（模块描述）+ 属性表（`tagPROP*`）+ 事件表（`tagEVT`）+ 方法表（`tagMETH`）+ 虚表（`_CXXX_vtbl`）驱动，通过 `MdlRegisterInternalCtls` 注册、`Thunk` 机制实现动态派发。

4. **设计器/运行时同构**：`DESK`/`CTL` 对象模型同时服务于设计态（`Ed*` 交互、`Flex*` 拖拽、`Grab` 手柄）和运行态（`FormCtlProc` 消息处理、`Graf*` 绘制），通过 `Mode` 状态切换。

5. **类型库自举**：`InMemTypeLib` 在内存中构造 ITypeLib2，`MdlTypeInfo`/`EventTypeInfo`/`CTypeInfoMerged` 等按需生成类型信息，支持用户控件/设计器的动态类型合并。

6. **自包含链接器**：`CExeFile` 实现了完整的 PE 文件生成（节区、重定位、导出、资源目录、版本资源、DLL Fixup），使 IDE 能独立生成 EXE/DLL/OCX 而不依赖外部链接器。

7. **停靠窗口框架自研**：`CDockManager`/`CContainer`/`CPane`/`CSlider` 构成完整的可停靠/浮动/分割窗口系统，支持布局序列化到注册表/流。

8. **Office 组件复用**：通过动态加载 MSO.DLL，复用 Office 的命令栏/工具栏/组件管理器（`IMsoComponentManager`、`IMsoToolbarSet`），实现与 Office 一致的 UI 体验。
