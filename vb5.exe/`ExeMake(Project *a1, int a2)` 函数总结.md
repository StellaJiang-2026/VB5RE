# `ExeMake(Project *a1, int a2)` 函数总结

## 函数职责

这是 **VB6 IDE “生成 EXE / 工程” 功能的主控函数**，对应菜单 `File → Make xxx.exe`。它把当前 `Project` 对象编译成一个可执行文件（EXE / ActiveX DLL / ActiveX EXE / OCX，取决于工程类型），并完成 COM 注册、类型库写出、版本资源写入、AddIn 通知等完整流程。

参数与返回值：

- `a1`：要编译的 `Project` 对象；
- `a2`：交互模式标志，`1` 表示从 IDE 菜单交互调用（会弹文件对话框、通知 AddIn），`0` 表示自动化/批处理；
- 返回值 `v83`：`1` 表示生成成功，`0` 表示失败/中止。

---

## 整体流程

### 1. 初始化与前置校验

- 栈上构造 5 个 `CFileRep`（`v66/v91/v68/v63/v64`）和 1 个 `CMemStream`（`v69`）；
- 读取工程状态字节 `*((_BYTE*)a1+700)`，若为 `0xFF` 则 `v92=0`，否则查表 `dword_587DCC` 得到 `v92`（是否使用自定义基地址，默认基址 `0x400000`）；
- 检查 `Project::GetProjKeyValue(a1,33)` 与 `CheckIfValidUnattended`，无效则返回 `0x8004xxxx` 类错误；
- 调用 `VerifyServerRestrictions` 校验服务器限制；
- 调用 `BeginMakeReferences` 开始引用解析；
- 成功则 `SetWaitCursor()` 并 `goto LABEL_8` 进入主编译循环。

### 2. 主编译循环（`LABEL_8`）

每次尝试都会：

1. 重置 `v89/v83/PfrefOfIcon/v80/v88`、`CMemStream::Reset`；
2. 用 `Project::GetEXEName` 两次取输出名：`v66`（原路径）和 `v91`（最终路径）；
3. 若全局 `dword_58AC7C` 有效，则加目录、`SetFullPathAbsolute`、`Project::SetEXEName`；
4. 交互模式（`a2`）下：
   - 已加载 FC AddIn 时走 `IdExtIDEGetNewFileName`，否则返回 `6` 触发 `DlgFMakeExe` 文件对话框；
   - 返回 `2` 表示取消；
   - 文件名变化时 `fExtIDERequestChangeFileName` 并置 `v73=1`；
5. 非交互且工程未设 EXE 名时，从路径反推 `Project::SetEXEName`；
6. `TipStartMakeExe` 启动进度条；
7. `_CheckExeInUse` 检查目标 EXE 是否被占用；
8. `SetPictureIcon` 取工程图标，非默认图标则 `PictGetPfrefOfIcon` + `HrFrefReOpen` 打开图标文件；默认图标则从资源 `0x100` 通过 `HrFrefOpenResource` 打开；
9. 置全局 `Cio_fMakeExe = 1`；
10. `Project::GetVer` 取版本号、`FixLicensingInfo`；
11. 若工程类型为 `3`（ActiveX 类工程），补全工程 Doc 字符串；
12. `CheckBackgroundCodeLoad`、`StartupSet` 取启动对象配置（返回 `1` 即错误 `50050`，启动对象无效）；
13. `CVBServer::ErrStartMakeExe` 启动编译服务器；
14. `ErrExtIDERequestWriteFile` 请求写文件权限；
15. 设置模块目录、加入 `VBAEXE5.LIB`；
16. 构造编译器参数结构 `v70`：
    - 结构大小 `48`，运行时 `MSVBVM50.DLL`；
    - 读取 `VBCompiler\C2Switches` / `VBCompiler\LinkSwitches`；
    - 输出路径、模块目录、主次版本号、基地址（`v92 ? 自定义 : 0x400000`）；
    - 若工程未禁用 `Key 23` 且 `Rby_Edition` 存在，则按 `Key 25~32` 设置 DLL/EXE、线程模型、优化、启动模式、编译选项、无 UI、未处理异常、快速调用、混合线程等标志；
    - `_FMtProj` 决定是否置多线程位；
17. 若 `v72`（工程有公共类）且 `FAnyPublicClasses`，生成 `.TLB` 或 `.VBR` 类型库路径，扩展名冲突则报 `50017`；
18. **`TipMakeExe2` 调用编译器**（实际编译动作）；
19. 成功后把版本信息写回 `CVBServer`，遍历所有工程项同步版本字段；
20. `RsrcWriteToExe` 写资源、`_GetSubMain` 取 Sub Main 入口；
21. `CRegData::CreateData` 构造 COM 注册数据；
22. 启动进度条，`CVBServer::ErrBeforeWritingExe` 写前检查；
23. `_WriteRubyExeData` 写 EXE 映像主体；
24. `_WriteVerResource` 写 VERSIONINFO 资源；
25. 存在类型库时通过 `HrFrefOpen` / `CRegData::WriteRegFile` 写 `.TLB` 文件；
26. 调用 `IExeFile` vtable+28（设置入口点/初始 IP）；
27. 如启用自动递增，`v81++` 修订号。

### 3. 收尾与错误处理（`LABEL_143` / `while` 顶部 / `LABEL_97`）

- `LABEL_143`：`_ErrFromHr` 把 HRESULT 映射成工具错误码；
- 循环顶部：若 `v89` 存在，按 `v95` 决定 `TipAbortExe` 或 `TipFinishExe2`，再 `ErrEndMakeExe`；
- 成功则 `v83 = 1`；
- `ExtIDEAfterWriteFile` / `ExtIDEAfterChangeFileName` 通知 AddIn；
- `CVBServer::ErrSucceedMakeExe` 对 ActiveX 服务器做 COM 注册；
- 恢复工程 Doc 字符串；
- `Cio_fMakeExe = 0`、释放进度条、`IPicture`、堆内存、`FrefDelete` 图标引用、`Project::SetVer` 恢复版本；
- 严重错误（`v95 && v95 != 59999`）调用 `Error()` 后 `break`；
- 非重试状态（`v80 != 2`）`goto LABEL_97`；
- `EndMakeReferences`、成功时 `ReloadDesksUsingUserControls` 刷新使用用户控件的窗体；
- 销毁所有 `CFileRep` / `CMemStream`，返回 `v83`。

---

## 关键设计点

| 点 | 说明 |
|---|---|
| **重试循环** | `v80 == 2` 时跳回 `LABEL_8` 重试；常见于用户从错误对话框选“重试” |
| **伪错误 59999** | 表示“用户取消/静默中止”，不弹错误框 |
| **错误 50050** | 启动对象无效 |
| **错误 50017** | 类型库输出扩展名与输入冲突（`.TLB/.VBR`） |
| **`v92`** | 是否使用自定义基地址；`0` 时固定 `0x400000` |
| **`a2` 交互模式** | 控制文件对话框、AddIn 回调、线程窗口枚举、工程 Doc 设置 |
| **`Cio_fMakeExe`** | 全局“正在 Make EXE”标志，供其他模块判断状态 |
| **`VBAEXE5.LIB`** | 链接时使用的 VB6 运行时导入库 |
| **`MSVBVM50.DLL`** | 生成 EXE 的运行时依赖 |
| **类型库按需生成** | 只有存在公共类（`FAnyPublicClasses`）才写 `.TLB`/`.VBR` |
| **ActiveX 注册** | `ErrSucceedMakeExe` 在成功后做 COM 注册，非 ActiveX 工程跳过 |
| **工程项同步** | 编译后遍历 `ProjLppiFirstProjitem` 同步版本字段到所有工程项 |

---

## 一句话概括

`ExeMake` 是 VB6 IDE 的 **“Make EXE”总控函数**：它先做工程/服务器/引用校验，然后在重试循环里反复执行“确定输出名 → 取图标 → 组装编译器参数 → 调用编译器 → 写资源/版本/类型库/注册数据 → 写 EXE 映像 → 通知 AddIn → 收尾恢复”的完整流水线，最终返回是否成功。
