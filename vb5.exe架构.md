## 结论

从函数名、类名、COM 接口、地址布局和子系统符号看，这**高度像 Microsoft Visual Basic 6.0 IDE / VBA 宿主与设计环境的主程序**（通常对应 `VB6.EXE` 一类二进制），而不是普通 VB 编译产物。它是：

- **32 位 Windows PE，基址约 `0x00400000`**，代码集中在 `.text`，从 `0x00401000` 到 `0x0057xxxx`；
- **MSVC/C++ 编写、重度 COM/OLE、静态链接 CRT** 的巨型单体桌面应用；
- 同时包含 **IDE 外壳、项目/组件对象模型、窗体设计器、控件运行时、属性浏览器、菜单设计器、组件库、AddIn/VBA 宿主、类型库与 EXE 生成器**；
- 采用大量 **C++ 多重继承 + COM 接口 + thunk adjustor**，例如 `QueryInterface/AddRef/Release`、`[thunk]:...adjustor{...}`；
- 具有明显的 **设计时/运行时双模态**：同一套 `DESK/CTL/CDFrame` 体系既服务窗体设计，也服务窗体运行、预览和 OLE 控件宿主。

---

## 一、总体架构形态

可以把它理解为一个“**Win32 单体外壳 + COM 对象模型 + 设计器/运行时宿主 + 持久化/编译器工具链**”的架构：

```text
WinMain / 消息循环 / Mso Component Manager / Dock 框架
        |
IDE 外壳：主窗口、项目窗口、工具箱、属性浏览器、菜单设计器、预览、组件库、AddIn
        |
VBA/VB 对象模型：CVBEobj, CVBApp, VBProjects, VBComponents, VBProperties, VBWindows
        |
项目/组件模型：Project, Projitem 及其子类（Module/Class/Form/OCX/UserControl/Designer/Ref...）
        |
窗体/控件体系：DESK/DESKREF, CTL 基类, FORM/LABEL/EDIT/LBOX/COMBO/... , CDFrame
        |
COM/OLE 控件宿主：OLECTL, COcx, COleClientObj, CSite, CEventSink, 属性通知、拖放
        |
持久化：FILEREF/IStream/IStorage, CPropertySet/Bag/Dictionary, Cio* 文本/二进制读写
        |
类型库/注册/编译：InMemTypeLib, OnDiskTLib, COAServer, CExeFile, ExeMake, 版本资源
        |
基础设施：CThreadData, DblLstBase/CARR/Ntbl, CEnumGeneric, 错误/异常, 国际化/IME
```

---

## 二、关键子系统与证据

### 1. IDE 外壳与停靠窗口系统

代表符号：

- `WinMain`, `InitFMain`, `InitGlobal`, `MainHwndCreate`, `MainFTranslateMessage`, `ThunderMsgLoop`
- `CDockManager`, `CPane`, `CContainer`, `CSlider`, `CDockWinRect`
- `CProjWin`, `ProjWin*`, `ToolHwndCreate`, `PbrsFCreate`, `PreviewFInitWindow`

说明：这是典型 VB6 IDE 外壳，包含可停靠面板、项目浏览器、工具箱、属性栏、预览窗口和主消息循环。`CMsoComponent/CMsoCMHandler` 表明它接入了 Office/Mso Component Manager 的消息与空闲处理机制。

### 2. VBA/VB 对象模型与自动化接口

代表符号：

- `CVBEobj`, `CVBApp`, `CVBApplication`
- `VBProjects`, `VBProject`, `VBComponents`, `VBComponent`
- `VBProperties`, `VBProperty`, `VBWindows`, `VBWindow`
- `VBAddInManager`, `CVBAddIn`, `VBFormTemplate`, `VBControlTemplate`
- `VBDispatch`, `VBClassFactory`, `VBEventConnection`, `CConnectionPtBase`

这是 VB IDE 暴露给 VBA/AddIn/自动化的对象模型层。大量 `QueryInterface/AddRef/Release/GetIDsOfNames/Invoke` 和 connection point 类说明它重度依赖 COM 自动化与事件连接。

### 3. 项目/组件模型

代表符号：

- `Project`, `Projitem`
- `CProjitemModule`, `CProjitemClass`, `CProjitemVisual`, `CProjitemOCX`, `CProjitemUserControl`, `CProjitemDesigner`, `CProjitemPropertyPage`, `CProjitemDocObject`, `CProjitemRef`, `CProjitemTypelibRef`, `CProjitemRes`
- `ProjectGroup`, `GroupItem`, `CProjListElm`, `CDuplicateListElm`

说明：`Project` 是项目聚合根，`Projitem` 是组件/文件项基类，子类覆盖模块、类模块、窗体、OCX 引用、用户控件、设计器、属性页、文档对象、引用和资源。`ProjectGroup` 支持项目组。重复模块/共享源文件检查由 `CDuplicateListElm` 等处理。

### 4. 窗体、控件与设计器

代表符号：

- `DESK`, `DESKREF`, `DESKUSER`, `DESKOLE`, `DESKDOC`, `DESKPAGE`, `DFRAMEDESK`
- `CTL`, `FORM`, `LABEL`, `PUSH`, `EDIT`, `LBOX`, `COMBO`, `CHECK`, `OPT`, `FRAME`, `SCROLL`, `TIMER`, `IMG`, `PIX`, `LINE`, `SHAPE`, `MDI`, `DIR`, `DRIVE`, `FILE`, `QUERY`, `CLIENT`, `APP`, `SCRN`, `CTLPRINT`
- `CDFrame`, `CDFrame::CtlProc`, `FormCtlProc`, `StdCtlWndProc`, `VBDefControlProc`
- `Flex*`, `_Ed*`, `EdHctl*`, `EdDrawGrid`, `_Grab*`, `_Target*`

说明：`DESK` 可视为窗体/设计面/运行时桌面容器，`DESKREF` 是持久化描述；`CTL` 是控件基类，各类控件通过 `CtlProc` 窗口过程驱动。`Flex*` 和 `_Ed*` 是设计时抓取手柄、移动、缩放、选择、网格、组选、撤销等编辑逻辑。`CDFrame` 是设计器框架控件，连接窗体设计器与 OLE/控件宿主。

### 5. COM/OLE 控件宿主与 OCX 体系

代表符号：

- `OLECTL`, `COcx`, `COleClientObj`, `COcx::CSite`, `COcx::CEventSink`
- `CPropertyBag`, `CPropertySet`, `CPropertyNotifySink`
- `CClipDataObjectObj`, `CDropTarget`, `ODDDataObject`, `ODDDropSource`
- `OleC*`, `OleStd*`, `CreateInitEmbeddedObject`, `OleLoadWithLicKey`
- `InMemTypeLib`, `COnDiskTLib`, `CTypeInfoMerged`, `CTypeInfoOcxEvt`
- 大量 `[thunk]:...adjustor{...}`

说明：这是 VB6 作为 ActiveX/OCX 容器和宿主的核心。`COcx/COleClientObj/CSite` 实现 OLE 客户端站点、原位激活、事件 sink、属性通知、拖放和数据对象。类型库合并/委托类用于把 OCX 类型信息与 VB 扩展属性、事件合并。

### 6. 持久化与文件格式

代表符号：

- `FILEREF`, `CFileRep`, `IStream/IStorage` 相关 `Read/Write/Seek/Stat`
- `CPropertySet`, `CPropertyBag`, `CDictionary`, `CProperty`
- `Cio*`：`CioErrSaveForm`, `CioLoadTextDesk`, `CioReadTextProperties`, `CioSaveTextProperties`, `CioWriteClassModuleHeader`
- `_SaveTextReferences`, `WriteObjectBlock`, `TagFRXBlobs`, `AddToFRXRefList`
- `Project::ProcessProperties`, `Project::ReadTextProjFile`, `Project::ErrWriteProjData`, `ProjectGroup::Load/Save`

说明：持久化层统一使用 `FILEREF/IStream` 抽象，属性用 property set/bag/dictionary 表示。`Cio*` 与 `FRX` blob 相关符号说明它支持 VB 的文本窗体/二进制附属数据格式、项目文件和项目组文件。`ProcessProperties`、`ReadTextProjFile` 等是 `.vbp`/文本项目解析和写出的核心。

### 7. 类型库、注册、许可与 EXE 生成

代表符号：

- `InMemTypeLib`, `COnDiskTLib`, `CTypeInfoMerged`, `DelegatorTypeInfo`
- `COAServer`, `CRegData`, `ErrSrvRegisterExe`, `_DoRegistration`, `_DoTlRegistration`
- `CExeFile`, `IExeFile`, `ExeMake`, `_WriteRubyExeData`, `_WriteVerResource`, `_CreateGroupIconResource`
- `LicKeyFromClsid`, `OleLoadWithLicKey`, `FHasLicensedCtls`

说明：IDE 内含类型库构建/合并、COM 注册、许可 key、PE/EXE 写入、版本资源、图标资源、资源段写入等逻辑。`CExeFile`/`ExeMake` 表明它具备生成可执行文件的能力，符合 VB6 “Make EXE” 架构。

### 8. 属性浏览器、工具箱、菜单设计器、组件库

代表符号：

- `Pbrs*`, `_UpdateEdit`, `_Update`, `_RegeneratePropList`, `_OleAddProps`, `_FlushProperty`
- `Tool*`, `Pal*`, `_ToolKey`, `ToolAtPoint`, `ToolGroup*`
- `MenuDesign`, `MD*`, `MenuDesDlgFn`, `RbySaveMenuRecursive`
- `CompGall*`, `AddInsertableDialogProc`, `AddControlsDialogProc`, `AddDesignersDialogProc`
- `Preview*`, `PVDraw*`, `FMoveSelectedRect`, `FAcceptNewPosition`

说明：这是 IDE 的生产力工具层。属性浏览器支持 OCX 属性、枚举、字体、颜色、对象选择、数据绑定；工具箱支持分组和拖放；菜单设计器维护菜单树；组件库/插入组件对话框管理可插入控件和设计器；预览窗口支持设计态布局预览。

### 9. 数据绑定、DAO/数据库、DDE、剪贴板、拖放

代表符号：

- `DATAACCESS`, `DATAACCESS` 相关 `DB*`, `HrDB*`, `ICursor`, `CQueryAS`, `CIRCNSink`
- `DAO`, `DAORecordset`, `_ErrDbCreateDynaset`, `_HrOpenDynaset`
- `Dde*`, `DdeCallback`, `DdeErr*`, `ErrLink*`
- `Clip*`, `_HrClip*`, `ODD*`, `CDropTarget`, `VBDataObject`

说明：VB6 控件支持数据绑定、DAO/记录集、异步查询、DDE 链接、剪贴板和 OLE 拖放。`DATAACCESS`/`ICursor`/`CQueryAS` 是数据访问抽象；`Dde*` 是 DDE 会话与链接；`ODD*`/`CDropTarget` 是 OLE 拖放体系。

### 10. 国际化、IME、Bidi 与语言资源

代表符号：

- `Intl*`, `LoadLangDLL`, `LoadLangDLLMain`, `LocalizeFilename`, `GetIntlExtension`
- `CImeKey`, `JP*IME*`, `_SetImeMode`, `_GetImeMode`, `SetIMECurPosHelper`
- `BidiSetStyle`, `BidiSetStyle32`, `_CheckBiDi`, `IsEuroLang`

说明：IDE 支持多语言资源 DLL、本地化文件名、IME、日文 IME、BiDi/RightToLeft、Euro 语言等，是国际化版本 VB6 IDE 的典型特征。

### 11. 基础设施

代表符号：

- `CThreadData`：线程本地/每线程状态；
- `DblLstBase`, `DblLstElm`, `CARR`, `Ntbl*`：链表、控件数组、名称表；
- `CEnumGeneric`, `CEnumVariant`, `CEnumConnections`：COM 枚举器；
- `CMemStream`, `CFileRep`, `FILEREF`：内存流与文件抽象；
- `_WinMainCRTStartup`, `_malloc`, `_free`, `_qsort`, `__except_handler3`：静态 CRT 与异常处理；
- `CMsgFilter`：OLE 消息过滤/忙碌对话框。

---

## 三、运行与设计双模态

这个架构最显著的特征是“设计时”和“运行时”共享同一套窗体/控件基础设施：

- 设计时：`Flex*`、`_Ed*`、`EdHctl*`、`_Grab*`、`Tool*`、`Pbrs*`、`MenuDesign`、`Preview*`；
- 运行时：`FormCtlProc`、`StdCtlWndProc`、`CTL::Invoke`、`EvtErrFire*`、`RefCallMethFromVtable`、`DeskUserModeKey`；
- 预览/运行：`RenderForm`、`MetafilePaintFormHDC`、`ShowMethShow/Hide`、`FormShowModal`。

因此它不是简单的“编辑器 + 独立运行时”，而是 **IDE 内部嵌入窗体宿主和控件运行时**，并通过 `DESK/DESKREF/CTL/CDFrame` 统一设计态、运行态和 OLE 控件宿主。

---

## 四、架构风格评价

1. **巨型单体 Win32 桌面应用**：所有核心逻辑集中在一个 EXE 的 `.text` 中，静态 CRT，函数数量巨大。
2. **COM 重度架构**：对象模型、控件宿主、事件、属性通知、拖放、类型库均通过 COM/OLE 接口实现；多重继承 thunk 很普遍。
3. **文档/设计器/控件宿主混合模式**：`DESK` 类似窗体文档，`CDFrame/CTL` 类似控件宿主，`CProjWin/CDockManager` 是 IDE 外壳。
4. **统一属性持久化模型**：`CPropertySet/Bag/Dictionary` 贯穿控件属性、窗体持久化、OCX 属性和项目文件。
5. **可扩展点丰富但耦合高**：AddIn、Designer、OCX、UserControl、PropertyPage、TypeLib 都是扩展点，但对象模型和全局状态高度耦合。
6. **线程模型偏单 UI 线程 + 每线程上下文**：`CThreadData`、`ProjectThread`、`DeskrefThread` 表明有线程上下文管理，但整体仍是经典 Win32 消息驱动、COM STA 风格。

---

## 五、进一步验证建议

由于这里只有 `.text` 函数元数据，没有导入表、资源段、调用图和数据段，建议在 IDA/Ghidra 中继续验证：

1. 查看导入表是否包含 `OLE32/OLEAUT32/DDE/COMCTL32/SHELL32/COMDLG32/DAO` 等；
2. 看 `WinMain` → `InitGlobal` → `MainHwndCreate` 的初始化链；
3. 看 `Project::OpenText`、`ProcessProperties`、`CioSaveTextDesk` 确认 `.vbp/.frm/.frx` 处理；
4. 看 `ExeMake/CExeFile` 调用链确认 Make EXE；
5. 看 `CVBEobj/VBProjects` 与 VBA 宿主接口，确认 VBA 版本与自动化模型；
6. 看 `COcx/COleClientObj` 的接口表，确认 OCX 容器能力。

总体判断：**这是一个典型的 VB6 IDE 巨型单体架构：Win32 外壳 + VBA/VB 对象模型 + 窗体设计器/运行时 + ActiveX/OLE 宿主 + 属性包持久化 + 类型库/EXE 生成工具链。**
