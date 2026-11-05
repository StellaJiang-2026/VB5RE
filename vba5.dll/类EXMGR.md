从这份函数清单可以非常清楚地看出 **`EXMGR` 是 VB 编译器的“表达式/语句中间代码（BIL）生成与语义处理核心”**。它处于前端（语法树/操作码行）和后端（LowBil、Native 代码）之间，负责把源码语义转换成 BIL，再推进到 LowBil/Native。

`EXMGR` 最可能是 **Expression Manager / Execution Manager**（表达式/执行管理器）。它的函数名覆盖了从“逐行编译”到“生成函数/模块”的完整链路。

---

## 一、总体定位

**`EXMGR` = BIL 生成器 + 表达式/语句语义分析器 + 模块/函数代码生成驱动**

它做四件事：

1. **把 OP_LINE（操作码行/AST）编译成 BIL 树**（`CompileLine` 及 `Compile*`）；
2. **构造 BIL 树节点**（大量 `Make*`：常量、符号、调用、赋值、类型转换、控制流）；
3. **做绑定/解析/类型检查/隐式转换**（`Bind*`、`Browse*`、`Process*`、`Check*`、`Coerce*`）；
4. **驱动模块/函数级代码生成**（`GenerateModule`、`GenerateFunction`、`TransformToLoBil`、`nativeGenProcedure`）。

---

## 二、按功能分类

### 1. 核心编译入口（BIL 生成）

| 函数 | 作用 |
|---|---|
| **`CompileLine(uint, OP_LINE const*, int* const)`** | **逐行编译的核心入口**，把一行操作码/AST 编译成 BIL |
| `CompileBinary` / `CompileUnary` | 二元/一元表达式编译 |
| `CompileIntrin` | 内部函数/内建调用编译 |
| `CompilePrint` / `CompileRedim` / `CompileRef` / `CompileStringFuncs` / `CompileWatchRet` | 特定语句/表达式的 BIL 生成 |
| `PrepareForCompile` | 编译前准备（初始化状态、栈、哈希等） |
| `SetAndEmitRTMI` | 发射运行时元信息（RTMI），记录代码块偏移/大小 |
| `FixupFuncForContinue` | 修复 Continue 相关跳转 |

`CompileLine` 是 `EXMGR::GenerateFunction` 里逐行循环调用的那个函数，**BIL 树就是在这里一行行构造出来的**。

---

### 2. 模块/函数级代码生成

| 函数 | 作用 |
|---|---|
| **`GenerateModule(int)`** | 模块级生成驱动：遍历函数表，逐函数调用 `GenerateFunction` |
| **`GenerateFunction(ulong, int)`** | 函数级生成：`CompileLine` 构造 BIL → `TransformToLoBil` → `nativeGenProcedure` |
| `GetBilimportForRdt` | 取 RDT 的 BilImport |
| `NotifyWatchType` | Watch/调试类型通知 |

---

### 3. BIL 节点构造（`Make*` 家族）

这是 EXMGR 数量最多的一类，负责构造各种 `stmtExpr` 树节点：

**常量/字面量**
- `MakeCnsI2Node`、`MakeCnsI4Node`、`MakeCnsSpecialNode`
- `MakeLitHard`、`MakeLitStr`、`MakeLitStrOfHlnam`、`MakeLitVar`、`MakeLitDefault`

**符号/变量/临时**
- `MakeSymNode`、`MakeSymNodeHvdefnPtdata`、`MakeSymTempVar`
- `MakeVarIndexNode`、`MakeStoreAdrTemp`、`MakeFStLongTemp`

**调用/操作**
- `MakeOneArgCall`、`MakeTwoArgCall`、`MakeThreeArgCall`、`MakeHelperCall2`
- `MakeUnaryOpNode`、`MakeSeq1Node`、`MakeHardType`

**赋值/记录/数组**
- `MakeArrayAssign`、`MakeRecAssignTree`、`MakeRecordAssign`、`MakeCopyRecordLetRHS`
- `MakeRedimTree`、`MakeRedimParamArray`、`MakeAryLock`、`MakeDestructRecord`
- `MakeGetPutAryDescr`、`MakeGetPutDescr`、`MakeFxRecAryDesc`

**参数/转换/返回**
- `MakeByrefArrayParam`、`MakeByrefCopyInOut`、`MakeCDargRefAry`
- `MakeCastIn`、`MakeCastOut`、`MakeCVar`、`MakeCVarNonOwner`
- `MakeCRec2UniFuncRet`、`MakeVectorRet`、`MakeFloatCoerce`
- `MakeHiddenParam`、`MakeFreeListArg`、`MakeFreetmpList`、`MakeFreetmps`

**其他**
- `MakeMeQualifier`、`MakeNewIfNullNode`、`MakePrdtTree`、`MakeClearOutNode`
- `MakeImportOfBytes`、`MakeImportOfParent`、`MakeStFixedStr`
- `MakeOpstrInChunkFromBstr`、`MakeIntrinsicModeArg`

这些 `Make*` 就是 **BIL 树的构造器**：每个函数返回一个 `stmtExpr*` 节点，由 `CompileLine`/`Compile*` 组合成整棵 BIL 树。

---

### 4. 绑定 / 解析 / 类型检查

| 类别 | 函数 |
|---|---|
| 绑定 | `ExBind`、`ExBindExtName`、`BindDefaultProperty`、`BindMangledName`、`BindObjQual`、`BindObjQualLessArgsLd`、`BrowseExBind`、`BrowseReference`、`BrowseResolveArgs`、`CheckIdBind` |
| 参数处理 | `ProcessArg`、`ProcessArgArray`、`ProcessArgByptrObj`、`ProcessArgByrefRec`、`ProcessArgByvalIntr`、`ProcessArguments`、`ProcessNamedArgs`、`ProcessLateNamedArgs`、`ProcessParamArray`、`ProcessLateArguments`、`ProcessLateBound`、`ProcessLateBoundResult`、`ProcessLateIdParamArray` |
| 类型转换 | `Coerce`、`CoerceArrayToAnsi`、`CoerceArrayToVariant`、`CoerceByValVar`、`CoerceDateFinish`、`CoerceLiteral`、`CoerceRecToAnsi`、`CoerceToAnsi`、`CoerceToVector`、`CoerceValueConsume`、`DecideCoercion` |
| 语义检查 | `CheckAssignCoerce`、`CheckAssignCoerceDarg`、`CheckAssignCoerceSlim`、`CheckAssignCoerceSlimer`、`CheckBetweenFuncs`、`CheckConstantness`、`CheckFormLeftHack`、`CheckImplicitUsage`、`CheckPropNoParams`、`CheckAndSetParamArray` |
| 类型查询 | `GetLenUDT`、`GetLenUDTHelper`、`GetSizeAndPad`、`GetHimptypeOfHvdefn`、`GetItOfQtdefn`、`MapHimptype`、`ObjstOfHimptype`、`TypeCharMatch`、`FarecLookupAndEnter`、`FarecSetFuncLife` |
| 名称/作用域 | `IsGlobalNameUsed`、`IsLocalNameUsed`、`GetNamedLocals`、`SetTreeToLocalBind` |

---

### 5. 语句/控制流处理（`Process*` 家族）

`Process*` 是 **把各类 VB 语句/操作码转换为 BIL** 的处理器，覆盖 VB 语言主要语句：

- 声明/定义：`ProcessConstDecl`、`ProcessVarDefn`、`ProcessFuncDefn`、`ProcessFuncDefnConstExprs`
- 赋值/引用：`ProcessLetSetRHS`、`ProcessRHSSet`、`ProcessRHSStore`、`ProcessQualRef`、`ProcessDictRef`、`ProcessVariantIndex`
- 控制流：`ProcessFor`、`ProcessForEach`、`ProcessNext`、`ProcessExitFor`、`ProcessSelect`、`ProcessEndSelect`、`ProcessCases`、`ProcessCaseDone`、`ProcessCaseElse`、`ProcessWith`、`ProcessWatchWith`、`ProcessEndExit`
- 表达式/运算：`ProcessOperator`、`ProcessOpCoerce`、`ProcessIs`、`ProcessTypeOf`、`ProcessSubscripts`、`ProcessArrayBase`、`ProcessArrayRef`、`ProcessBoundCheck`、`ProcessByptr`、`ProcessByrefIntrinsic`、`ProcessByrefIntrinsicLate`
- 函数/调用：`ProcessFuncRef`、`ProcessFuncRet`、`ProcessFuncThis`、`ProcessIntrinsicArg`、`ProcessSpecialSyntax`
- I/O：`ProcessOpen`、`ProcessClose`、`ProcessInput`、`ProcessInputItem`、`ProcessLineInput`、`ProcessLock`
- 其他：`ProcessNew`、`ProcessErase`、`ProcessRedim`、`ProcessLsetRset`、`ProcessMidStmt`、`ProcessGetPutRec`、`ProcessSystemError`、`ProcessCCConst`、`ProcessCCIf`、`ProcessCCNameLoad`

---

### 6. BIL/控制流栈与标签

| 函数 | 作用 |
|---|---|
| `CntlPush`/`CntlPop`/`CntlTop`/`CntlIsEmpty_Bil`/`CntlSearchStack`/`CntlCleanStack`/`CntlInit`/`CntlGetErrorAndClean` | **控制流 BIL 栈**（If/For/Do/Select/With 等嵌套控制） |
| `CntlBind_If`、`CntlBind_Else`、`CntlBind_EndIf`、`CntlBind_DoWhileFor`、`CntlBind_EndFor`、`CntlBind_EndLoop`、`CntlBind_ExitDoFor`、`CntlBind_Select`、`CntlBind_Case`、`CntlBind_CaseElseDone`、`CntlBind_EndSelect`、`CntlBind_With`、`CntlBind_EndWith` | 控制流绑定/收尾 |
| `TStPush`/`TStPop_Bil`/`TStInsert`/`TStInsertAtBottom`/`TStDepth_Bil` | **临时语句栈**（语句级 BIL 暂存） |
| `WithPush`/`WithTop`/`WithPop_Bil`/`BuildWithState`/`BuildWithStateNewDefn`/`ClearWithState` | With 语句状态 |
| `LabelDef`/`LabelRef`/`LabelFindUndefined`/`LabelGetLineNum`/`MapLabelToLineNum` | 标签定义/引用/行号映射 |
| `AllocBilLabel`、`AllocBilCaseList` | BIL 标签/Select Case 列表分配 |

---

### 7. LowBil 与临时变量池

| 函数 | 作用 |
|---|---|
| `AllocLowbilTmp`、`ResetLowbilTmps` | 分配/重置 LowBil 临时变量 |
| `AllocTmp`、`AllocTmpArray`、`AllocTmpRecFuncRet`、`AllocTmpWithHtdefn`、`AllocVarDefn`、`AllocVarDefnWithHtdefn`、`AllocSimpleTempVarDefn` | 临时变量/变量定义分配 |
| `FreetmpAlloc`、`FreetmpAdd`、`FreetmpEmit`、`FreetmpEmitCase`、`FreetmpEmitHelper` | 临时变量释放（Freetmp）分配/发射 |
| `TpoolInit`、`TpoolReset`、`TpoolAdd`、`TpoolAddFreeProcTmps`、`TpoolAddFreeStmtTmps`、`TpoolAddFrees`、`TpoolAddFreetmpFrees`、`TpoolAddResDesc`、`TpoolAddResDescAryLock`、`TpoolAddResDescProcTmps`、`TpoolAddResDescStmtTmps`、`TpoolInitCheckpoint`、`TpoolRestoreFromCheckpoint`、`TpoolDiscardCheckpoint` | **临时池（Tpool）**：管理临时变量、释放列表、资源描述 |
| `FunctmpinfoAdd`/`FunctmpinfoLookup`/`FunctmpinfoLookupWithContext`/`FunctmpinfoRemove`/`FunctmpinfoAdjustLines` | 函数临时信息表 |
| `HashCreate`/`HashAdd`/`HashLookup`/`HashSet`/`HashAddFrees`/`HashAddResDesc`/`HashCreateCheckpoint`/`HashRestoreFromCheckpoint`/`HashDiscardCheckpoint`/`HashRead`/`HashWrite`/`WalkHashTable` | 哈希表（作用域/临时变量/资源描述） |

---

### 8. 错误/调试/序列化/状态

| 函数 | 作用 |
|---|---|
| `CreateError`、`CreateErrorOffset`、`CreateErrorOpline`、`CreateErrorOffset_Bil`、`CreateErrorOpline_Bil`、`CreateError_Bil`、`SetNonContinuableError` | 错误创建/定位 |
| `AddErlTable`、`AdjustLineNumbers`、`FunctmpinfoAdjustLines`、`LabelGetLineNum`、`MapLabelToLineNum`、`ClearLineChangedList`（由外部调用） | 行号/Erl 调试信息 |
| `Read`、`Write` | EXMGR 状态序列化 |
| `SaveFunctionState`、`RestoreOldFunctionState`、`DiscardSavedFunctionState`、`RestoreContext`、`SetContext` | 函数/上下文状态保存恢复 |
| `CntlGetErrorAndClean`、`ReleaseResource`、`ReleaseResourceFuncDefn` | 资源/错误清理 |
| `GetExGenState`、`EXGEN::ExgenFunction`（外部） | 表达式生成状态 |
| `Init`、`Read`、`Write`、`RegisterTypeinfoWithImpmgr` | 初始化/类型库注册 |
| `WalkHashTable`、`FindArrayRefExcode`、`GetGosubListSym` | 遍历/查找 |
| `SkipFixedBytes`、`SkipStr`、`SkipVarBytes`、`GetLongOpnd`、`ExtractLiteral`、`GetConstI4`、`GetConstValue` | 字节/常量/字面量读取 |
| `FMustDeferLitRef`、`ForceVariantValue`、`HandleVariantCompare`、`CvtVarBstr` | Variant/字面量处理 |
| `DisallowBaseUsage`、`FuncLiftCheck`、`FuncLiftCheckStart`、`FuncAsSub`、`FuInvokekindOfRefkind` | 函数/调用检查 |
| `GetRHSParamdefn`、`GetQualifier`、`InsertHiddenParam`、`InsertNamedArg`、`SetArgsRequirements`、`SetArraydim`、`SetHvdefnCurrentFuncRetval`、`SetIntrinsicResult`、`SetOFrameTempVar`、`SetSimpleItype`、`SetSimpleVtype`、`SetTreeLiteral`、`SetTypebinds` | 参数/类型/结果设置 |
| `RequestAddr`、`RequestLvalue`、`RequestMember`、`RequestRvalue`、`RequestRvalueArg` | 左值/右值/地址/成员请求 |
| `OwnerDescRecTypeInfo`、`OwnerDescTypeDefn`、`OwnerDescUdt`、`OwnerDescVarDefn` | 所有者描述（类型/变量） |
| `PopCheckIntrinsicArg`、`PopCoerceConsume`、`PopProcessIntrinsicArg` | 参数弹出/消费 |
| `SetNonPropCall`、`FoundNonPropCall` | 非属性调用标记 |

---

## 三、EXMGR 在编译链路中的位置

结合你之前给的函数：

```
GEN_PROJECT::MakeExe
  └─ EnsureInCompiledState
       └─ BASIC_TYPEROOT::Compile
            └─ EXMGR::GenerateModule        ← EXMGR 模块级驱动
                 └─ EXMGR::GenerateFunction ← EXMGR 函数级驱动
                      ├─ EXMGR::CompileLine（循环）    ← BIL 生成
                      │    ├─ Process* / Compile*
                      │    └─ Make*（构造 BIL 节点）
                      ├─ TransformToLoBil              ← BIL → LowBil
                      └─ nativeGenProcedure            ← LowBil → Native
```

所以 EXMGR 是**从 OP_LINE/AST 到 BIL，再到 LowBil，并驱动 Native 代码生成的核心类**。

---

## 四、一句话总结

> **`EXMGR` 是 VB 编译器的表达式/语句中间代码（BIL）生成与语义处理核心**：它通过 `CompileLine` 和大量 `Make*` 把操作码行/AST 编译成 BIL 树，通过 `Bind*`/`Process*`/`Check*`/`Coerce*` 完成绑定、参数处理、类型检查和隐式转换，通过 `Cntl*`/`TSt*`/`With*`/`Label*` 管理控制流和标签，通过 `Tpool*`/`Freetmp*`/`Hash*` 管理临时变量和资源，最终由 `GenerateModule`/`GenerateFunction` 驱动 BIL → LowBil → Native 代码生成。

如果你需要，我可以进一步：
1. 把 `EXMGR` 的 `Make*` 家族按 BIL 节点类型整理成结构表；
2. 分析 `CompileLine` 内部如何调度 `Process*` 和 `Make*`；
3. 分析 `Tpool*` / `Freetmp*` 的临时变量生命周期；
4. 分析 `Cntl*` 控制流 BIL 栈如何实现 If/For/Select/With 的嵌套。

你想深入哪一块？
