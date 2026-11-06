# VBA 运行时/IDE DLL 架构思维导图

```text
VBA Runtime / IDE DLL (VBA5.dll / VBA6.dll)
│
├─ 1. 宿主与线程层 (Host & Thread)
│   ├─ EBAPP        全局应用对象（单例）
│   │   ├─ EbInitHost / EbInitHostPart2 / EbInitHostRn
│   │   ├─ EbTerm
│   │   ├─ GetVbaTypeLib
│   │   ├─ SetLicenseInfoCache
│   │   ├─ BeginTypemgrXact / EndTypemgrXact
│   │   └─ FNeedToDisplayStateLossWarning
│   ├─ EBTHREAD     每线程运行时上下文
│   │   ├─ EbInitThread / EbTermThread
│   │   ├─ SetFreezeEvents / FreezeAllEvents
│   │   ├─ SetStepMode / GetStepMode
│   │   ├─ ReleaseAllAppObject
│   │   └─ ProjHasCodeOnStack
│   ├─ 线程局部存储
│   │   ├─ TlsAlloc / TlsGetValue / TlsSetValue
│   │   └─ _g_EbThread / _g_Ebapp
│   └─ DllMain / _DllMainCRTStartup / EbthreadDllInit/Term
│
├─ 2. 项目与模块管理层 (Project & Module Model)
│   ├─ ExecProj / ExecMod            运行期项目/模块 COM 对象
│   │   ├─ Load / Save / Reset / Destroy
│   │   ├─ GetMethodPtr / GetFunctionId / GetLpfnOfFunctionId
│   │   └─ VerifyRefLibs / SaveModule
│   ├─ GEN_PROJECT                   编译器侧项目对象
│   │   ├─ 模块表：GetModule / CreateModule / DeleteModule / CopyModule
│   │   ├─ 引用表：GetRefLib / AddRefLib / RemoveRefLib / VerifyReferences
│   │   ├─ 编译：Compile / Decompile / MakeExe
│   │   ├─ 类型条目：AddTypeEntry / UnAddTypeEntry / DestructTypeEntry
│   │   ├─ 限定缓存：InitQualCache / QualCacheOfHimptype / ReleaseQualCache
│   │   └─ COM 类型库：QueryInterface / GetTypeInfo / FindName / CreateTypeInfo
│   ├─ BASIC_TYPEINFO                模块 COM 外壳（ITypeInfo/IVbaModule/ICreateTypeInfo/IProvideClassInfo）
│   │   ├─ GetTypeAttr / GetFuncDesc / GetVarDesc / GetNames / Invoke
│   │   ├─ AddRefTypeInfo / AddImplType / SetFuncAndParamNames
│   │   └─ GetBTSrc / GetEmbeddedTypeInfo / LoadFromText / SaveAsText
│   ├─ BASIC_TYPEROOT                模块类型根（聚合根 + 状态机）
│   │   ├─ 子管理器：GetNamMgr / GetImpMgr / GetEntMgr / GetExTbl / GetExMgr / GetDtmbrs
│   │   ├─ 状态机：EnsureInUndeclared/SemiDeclared/Declared/SrpBuilt/Compiled/Runnable
│   │   ├─ 编译/反编译：Compile / DecompileTo*State
│   │   ├─ 持久化：Read / Write / ReadFixed / WriteFixed / ReadFromStream / WriteToStream
│   │   ├─ REC_TYPEINFO：Create/Alloc/Read/Write/Free/Lookup
│   │   └─ 事件源：CreateEventSourceTable / GetEventSourceTable / GetWtinfo*
│   ├─ BASIC_TYPESRC                 模块源码模型
│   │   ├─ 行管理：InsertLine / ReplaceLine / DeleteLine / DeleteRange / GetLine / FastGetLine
│   │   ├─ 编辑：EditLine / GetEditClass / GetEditKind / UndoGetEditKind
│   │   ├─ 预处理器：BasicPreprocessor / ParseDirective / IncludeStack*
│   │   ├─ 成员列表：CMLInit / CMLFuncDefn / ConstructMemberLists
│   │   ├─ 属性行：SetAttribute / ListAttribute / Set_ExtAttr / Get_ExtAttr
│   │   ├─ 错误：SetCompileError / SetBindError / LookupErrorCol
│   │   ├─ 断点：SetBkpt / ClearBkpt / HasBkpt
│   │   ├─ 调试插入：InsertWatchProc / InsertImmediateProc / DeleteDebugProc
│   │   └─ 中断事务：Initialize/Add/Abort/CompleteBreakModeTransaction
│   └─ CRefList                        类型库引用列表
│
├─ 3. 编译器前端 (Compiler Front-End)
│   ├─ scanner                          词法分析器
│   │   ├─ 生命周期：scanInitialize / scanStart / scanSetLocale
│   │   ├─ 正向：scan / scanLookAhead / scanGetTokenPos / scanGetTokenCol
│   │   ├─ 常量：scanIdentifier / scanNumericConstant / scanStringConstant / scanDateConstant
│   │   ├─ 续行/注释：scanRecordLineCont / scanRecordCommentConts / scanRecordReparseConts
│   │   ├─ 反向：scanReverseBeg / scanReverseGet / scanReverseWSP / scanReversePos / scanReverseEnd
│   │   └─ 全局：Token / tokenIsIdent / scanHashTbl / scannerLCID
│   ├─ parser                           语法分析器 + P-code 生成入口
│   │   ├─ 入口：parserInit / parserDone / parseOneLine / parseStatement
│   │   ├─ 表达式：parseExpr / parseTerm / parseVariable / parseArray / parseArgList / parseTypeofExpr
│   │   ├─ 声明：parseDimRedim / parseConsDecl / parseTypeDecl / parseProcDecl / parseImplements / parseOptionStmt / parseDefStmt / parseAttribute
│   │   ├─ 控制流：parseIfPart / parseThenPart / parseCaseStmt / parseForStmt / parseDoLoopStmt / parseOnStmt / parseResumeStmt
│   │   ├─ I/O：parseOpenStmt / parseSeekStmt / parseGetPutStmt / parseLockStmt / parseInputStmt / parsePrintStmt / parseNameStmt
│   │   ├─ 图形：parsePoint / parseLineMethod / parsePsetMethod / parseScaleMethod / parseCircleMethod
│   │   ├─ P-code：genPcodeBuffFull / genPcodeString / generatePcode
│   │   └─ 错误：parseSetError / parser::parseErrNumber
│   ├─ genOpc                           操作码生成器（AST → P-code）
│   │   ├─ 总入口：genOpcodesForStmt / genOpcodesForVal
│   │   ├─ 参数：genOpcodesForArg / genOpcodesForArgList / genArgExprs
│   │   ├─ 控制流：genOpcodesForIfBlock / If / Case / On / Print / Assert
│   │   ├─ 声明：genOpcodesForDim / CnsDcl / OneCns / TypeDef / Proc / Implem / Events / VarDecl
│   │   ├─ 数组：genOpcodesForOneDim / Redim / Array
│   │   └─ 工具：genOpcodeBOS / genOpcodeLno / genOpcodeLab / genOpcodeComment / genOpcodeErr
│   └─ 全局：g_ptdata / parseHashTbl / labelIdent / lineNoIdent / parseLineFlags / genPcodeBuff*
│
├─ 4. 中间表示与代码生成 (IR & Codegen)
│   ├─ BILALLOC                         BIL 节点分配
│   │   ├─ MakeIndirOf / MakeArgListNode / MakeSeqNode / MakeAdrNode / CopyBilTree
│   │   └─ SetSimpleItype / SetSimpleVtype / SetTypeFromHvdefn / SetVoidPtrType
│   ├─ Transform_*                      BIL 树变换
│   │   ├─ 表达式：Transform_Expression / Coerce / Assign / Call / Concat / RelOp
│   │   ├─ 控制流：Transform_IntegralFor / VariantFor / CyFor / ForEach / Select / OnErr / Resume
│   │   └─ 数组/IO：Transform_FixedArray / RedimArray / InlinedRedimArray / IO / Len / InStr
│   ├─ BuildLoTree_*                    低层 BIL 树构造
│   ├─ EXMGR                            表达式生成管理器（BIL → 机器码/P-code）
│   │   ├─ Init / Write / Read / PrepareForCompile / GenerateModule / GenerateFunction
│   │   ├─ 绑定：ExBind / BrowseExBind / ProcessBindResult / ProcessLateBound
│   │   ├─ 强制转换：Coerce / MakeCVar / MakeHardType / DecideCoercion / HandleVariantCompare
│   │   ├─ 临时池：TpoolInit / TpoolAdd / TpoolAddFrees / TpoolRelease
│   │   ├─ 控制栈：CntlPush / CntlPop / CntlBind_If / CntlBind_Select / CntlBind_DoWhileFor
│   │   ├─ 哈希：HashCreate / HashSet / HashLookup / HashAdd / WalkHashTable
│   │   └─ 错误：CreateError / CreateErrorOffset / CreateErrorOpline / DeferCompileError
│   ├─ EXGEN                            低层代码生成（BIL → 目标指令）
│   │   ├─ 函数：ExgenFunction / ExgenRvalue / ExgenAsg / Exgen_Call / ExgenBinary / ExgenAddr
│   │   ├─ 地址模式：EmitAddrMode / MakeAddrModeSym / MakeAddrModeDataMem / MakeAddrModeIndex
│   │   ├─ 控制流：Exgen_ElseOrCase / Exgen_BlkEnd / Exgen_JumpLabel / Exgen_OnGo
│   │   └─ 辅助：Exgen_LateCall / Exgen_IO / Exgen_GetPut / Exgen_DestructRec / Exgen_EndProcGetRetInfo
│   ├─ EXTBL                            扩展代码块表
│   │   ├─ BeginFuncNew / EndFunc / FreeExcodeBlock / CreateAndEmitStub
│   │   ├─ 发射：EmitLong_Bil / EmitBytes_Bil / EmitBos_Bil / EmitBranch_Bil / EmitEx_Bil / EmitByte/Short
│   │   └─ BOS 表：EmitLargeBosTable / AddToLargeBosTbl / FixupBos_Bil
│   └─ OPCODE_MGR / OPCODE_TRAVERSE / OPCODE_SEARCH   行 P-code 存储与遍历
│
├─ 5. 类型系统 (Type System)
│   ├─ TYPE_DATA                       类型数据（定义存储）
│   │   ├─ 分配：AllocVarDefn / AllocParamDefn / AllocFuncDefn / AllocTypeDefn / AllocRecTypeDefn / AllocArrayDescriptor
│   │   ├─ 释放：FreeVarDefn / FreeFuncDefn / FreeTypeDefn / FreeRecTypeDefn / FreeArrayDescriptor
│   │   ├─ 查询：GetFuncDesc / GetVarDesc / GetNames / GetDocumentation / HdefnOfHmember
│   │   ├─ 修改：AddFuncDesc / AddVarDesc / SetFuncAndParamNames / SetFuncDocString / SetFuncHelpContext
│   │   └─ 等价：EquivTypeDefns / EquivFuncDefns / EquivTypeDefnsIgnoreByRef
│   ├─ TYPE_DEFN / VAR_DEFN / FUNC_DEFN / PARAM_DEFN / MEMBER_DEFN / RECTYPE_DEFN / DLLENTRY_DEFN
│   ├─ TYPE_DEFN 辅助：DefnSize / QtdefnBase / IsAliasToIntrinsic
│   ├─ RESDESCTBL                      资源描述表（对象布局）
│   │   ├─ AddResdesc / AddArrayLock / AddECChunk / CbResourceSize
│   │   ├─ ConstructInst / DestructInst / CopyInst / ConvertInst / AssignInst
│   │   └─ TrapRead / TrapWrite / TrapFixup
│   ├─ REC_TYPEINFO / REC_TYPEBIND      UDT 类型信息/绑定
│   ├─ WrapTypeLib / WrapTypeInfo / WrapTypeComp   类型库包装
│   ├─ CEcTypeInfo / CEcTypeComp / CEcProjTypeComp 编译期类型信息/绑定
│   ├─ IMPMGR                          类型导入管理器
│   │   ├─ 分配：HimptypeAlloc / HimptypeDelete / NewEntry
│   │   ├─ 查找：GetHimptype / GetHimptypeIfExists / GetHimptypeWithHighestDepkind
│   │   ├─ 绑定：BindTypeInfo / LoadTypeInfo / GetTypeInfo / HookUpBaseTypeInfo
│   │   ├─ 基类：SetBaseClass / SetBaseTypeInfos / SetDefaultInterfaceOfBase / SetDefaultSourceOfBase
│   │   ├─ 布局：LayoutDependents / CheckLayoutDep / CheckRemainingDep
│   │   └─ 序列化：Read / Write / WriteLayoutEntries
│   ├─ TYPEMGR                         全局类型库管理器
│   │   ├─ TypeLibLoaded / TypeLibUnloaded / TypeLibMoved
│   │   ├─ GetTypeLib / RegisterDynTypeInfo / UnRegisterDynTypeInfo
│   │   ├─ AddLibRef / RemoveLibRef / FIsLibRef / DeleteLibRef
│   │   └─ 依赖：CanProjChange / VisitDependentProj / HasCodeOnStack
│   └─ NAMMGR                          名字哈希表
│       ├─ Init / InitKeyWord / Write / Read / CleanNames
│       ├─ 哈希：hashName / hashString / rehashName / rehashIdents
│       ├─ 查询：HlnamOfStr / HlnamOfStrW / StrOfHlnam / BstrOfHlnam / IsName
│       ├─ 绑定：AddExbnodeOfHlnam / BindWithExbCache / GetExbindOfHlnam
│       └─ 关键字：hashKwdDescs / QueryKwidOfStr / UHelpContextOfKwid
│
├─ 6. 符号系统 (Symbol System — FED*)
│   ├─ FEDSYM                          符号包装
│   │   ├─ Be*：BeModule / BeProject / BeVardefn / BeFuncdefn / BeRectdefn / BeLabel / BeVbcsym / BeMockupExbind
│   │   ├─ WalkOver*：WalkOverProjCont / WalkOverProject / WalkOverModule / WalkOverProc / WalkOverUdt
│   │   ├─ Get*：GetDllEntryDefn / GetFuncDefn / GetHdefn / GetHimptype / GetDefnTypebind
│   │   └─ 查询：BilsymGetName / BilsymGetType / BilsymGetDefnKind / BilsymIsMember / BilsymGetAccess
│   ├─ FEDTYP                          类型包装
│   │   ├─ Be*：BeVbctyp / BeHimptype / BeTypedefn / BePtrToBiltyp / BeVoidPtrType
│   │   ├─ Get*：GetTypeDefn / GetHimptype / GetTypeInfo / GetTdesckind / GetTypeData
│   │   └─ 查询：BiltypGetName / BiltypGetTypeKind / BiltypGetSize / BiltypArrayGetElemType / BiltypIsObject
│   ├─ FEDARG                          参数包装
│   │   ├─ BilargIsOptional / IsExplicitDefault / IsRetval / IsByVal / IsParamArray / IsAny / IsImplicitType
│   │   ├─ BilargGetName / GetType / GetSym / GetDefaultString / ProduceDefaultValue
│   │   └─ FEDARG::GetTypeData / GetBtroot / GetNammgr
│   ├─ FEDSINK                         事件接收器包装
│   │   └─ BilsinkGetName / GetOInst / GetCtlCookie / GetEPICookie / IsEventProperty / IsImplements / BeSource
│   ├─ FEDDIM / FEDTYP 维度
│   ├─ Bil* 系列辅助：BilInitialize / BilTerminate / BilFileLineOfLogln / BilResolveMemberList / BilClearAllCookies
│   └─ BBT<BILSYMNODE> / BBT<BILTYPNODE>   符号/类型平衡二叉树
│
├─ 7. 运行时执行引擎 (Runtime Execution Engine)
│   ├─ 解释器入口
│   │   ├─ ProcEntry                    函数入口（栈检查、帧构建、朋友检查）
│   │   ├─ ProcCallEngine / MethCallEngine  过程/方法调用引擎
│   │   ├─ DllFunctionCall              DLL 调用
│   │   └─ ConstExprEntry / WatchExprEntry / ImmediateEntry
│   ├─ 表达式引擎 (EX* 指令集)
│   │   ├─ 值：ExLateId / ExLateIdSt / ExLateIdCallSt / ExLateMemNamed / ExVarIndexLdVar / ExVarIndexStVar
│   │   ├─ 对象：ExNew / ErrExNew / HrExCast / __vbaCastObj / GetIEnumVARIANT
│   │   ├─ For Each：__vbaForEachCollVar / __vbaNextEachCollVar / __vbaForEachAry / __vbaNextEachAry / __vbaForEachVar
│   │   ├─ 变体：__vbaFreeVar / __vbaFreeVarg / __vbaVarMove / __vbaVarCopy / __vbaVarDup / ExSetVarVar
│   │   ├─ 循环：ExForLoopInit / ForLoopInit / ExNewForType / NewForType
│   │   ├─ 逻辑：__vbaVarNot / __vbaVarAnd / __vbaVarOr / __vbaVarXor / __vbaVarEqv / __vbaVarImp / GetLogical
│   │   └─ 运算：__vbaVarAdd / Sub / Mul / Div / Idiv / Mod / Pow
│   ├─ 运行时库 (rtc* / rt* / __vba*)
│   │   ├─ 字符串：rtcLeftBstr / rtcRightBstr / rtcMidBstr / rtcTrimBstr / rtcInStr / rtcStrConvVar / __vbaStrCat / __vbaStrComp
│   │   ├─ 日期：rtcDateAdd / rtcDateDiff / rtcDatePart / rtcGetYear / rtcSetDateVar / rtcPackDate
│   │   ├─ 数学/金融：rtcSin / rtcCos / rtcSqr / rtcDDB / rtcIPMT / rtcPMT / rtcIRR / rtcNPV
│   │   ├─ 交互：rtcMsgBox / rtcInputBox / rtcShell / rtcSendKeys / rtcDoEvents / rtcAppActivate
│   │   ├─ 文件：rtcFileOpen / rtcFileClose / rtcFileLen / rtcFileCopy / rtcKillFiles / rtcDir / rtcFileDateTime
│   │   ├─ 类型转换：rtcStrFromVar / rtcVarFromVar / rtcCVErrFromVar / rtI4FromVar / rtCyFromVar / rtDecFromVar
│   │   ├─ 数组：rtConstructArray / rtCopyArray / __vbaLbound / __vbaUbound
│   │   └─ 注册表：rtcGetSetting / rtcSaveSetting / rtcDeleteSetting / rtcGetAllSettings
│   ├─ 异常与错误
│   │   ├─ EbRaiseException / EbRaiseExceptionCode / EbGetExceptionInfo / EbExceptionHandled / EbCopyExceptionInfo
│   │   ├─ BasicExceptionHandler / _ImmedExceptionHandler / _BasicMethExceptionHandler / ConstExceptionHandler
│   │   ├─ _NTStdCatchHandler / _NTBasicProcHandler / NTBasicException / IsVbaException / IsUserReset
│   │   └─ EberrOfScode / EberrOfHresult / SetErrorInfoOfEberr / SetErrorInfoOfHresult
│   ├─ 栈帧与调试
│   │   ├─ EXFRAME：FCalledByNonBasic / FCalledByDebugger / IsImmedWinFrame / SetPtexIp / IsFunctionOnCallStack / ReplacePrtmi / IsStubCall
│   │   ├─ GrowFrame / lblExitProc* / ExitBasicProcUnwind
│   │   └─ UiLayerBreak / UiLayerEndStmt / UiLayerNotifyBegin/End
│   ├─ 实例管理
│   │   ├─ INSTMGR / RUN_INSTMGR：InitForClass / ConstructModInstance / DestructModInstance / CreateObjInstance / CreatePartialBasicInstance / CreateObjInstanceWithParts
│   │   ├─ 僵尸化：BASIC_CLASS_ZOMBIE / MakeZombieInstance / MakeZombieVtable / MakeZombieEventVtable / BuildZombieThunkTable / GetZombieThunk
│   │   └─ 扩展：CreateExtension / AttachModuleOfExtension / SetExtensionMode
│   └─ 线程上下文：g_currentErrorTrap / g_pbtRoot / g_pexmgr / g_pimpmgr / g_pbilalloc / g_symWorld / g_symProjIntrinsics
│
├─ 8. 调试器层 (Debugger)
│   ├─ WATCHMGR                        监视管理器
│   │   ├─ CreateWatch / DeleteWatch / InsertWatch / RemoveWatch / ValidateWatch
│   │   ├─ EvalWatch / EvalWatches / EvalBreakWatches / WatchGetValue
│   │   ├─ ExecuteImmedLogln / InvokeImmedSub / CreateImmedContext / DeleteImmedContext
│   │   ├─ GetContext / UpdateWatchExframe / ExframePushed / ExframePop
│   │   └─ Reset / MakeRunnable / NotifyDecompile / SetEngineFlags / ClearValues
│   ├─ WATCH / CODE_CONTEXT / IMMEDMOD_INFO   监视项/代码上下文/立即窗口模块
│   ├─ BPMGR                           断点管理器
│   │   ├─ ClearAllBp / ClearBpInModule / ClearBp / ToggleBp / ToggleBpOnLine
│   │   └─ KeepModuleLoaded / FindSlot
│   ├─ BMMGR                           书签管理器
│   │   ├─ CreateBmmgr / BmmgrDelete / ToggleBookmark / GetNextBookmark
│   │   └─ HasBookmarkOnLine / LinesInserted / LinesDeleted / ClearAllBookmarks
│   ├─ CALLSTACK_ITEM                  调用栈
│   │   └─ CreateCallstack / UpdateCallstack / ShowCallstackDlg / SelectCallstackItem / EbGetCallstackCount / EbGetCallstackFunction
│   ├─ 中断模式
│   │   ├─ EnterBasicBreakMode / ExitHostBreakMode / EnterBreakMode / EnterHostBreakMode
│   │   ├─ DetermineDebugOption / ShowCantEditMsg / GetFirstDisplayableExframe / IsDisplayable
│   │   ├─ EbReset / EbProjectReset / ResetProject / RecompileFunction
│   │   └─ SetNextStmt / StepToCursor / ClearStepToCursor / SetNextStatementToPtex
│   └─ 对话框：ShowWatchDlg / ShowEditWatchDlg / ShowInstantWatchDlg / HasWithSyntax / BstrGetValueTip
│
├─ 9. IDE 前端 (IDE Front-End)
│   ├─ 应用/窗口框架
│   │   ├─ UIAPP / CreateUiapp / UiappDelete / EnsureUiDllInit / InitUiFull / UiTerm
│   │   ├─ DWIN                          文档窗口容器
│   │   └─ EBWNDFRAME / IEbwnd / SetEbwndfrmOfHwnd / GetEbwndfrmOfHwnd / ResizeAllEbwnds / GetActiveEbwnd
│   ├─ 模块视图 (MODVWR)
│   │   ├─ Create / Init / SetModule / CreatePanes / InitPanes / ChangeTopPane
│   │   ├─ 布局：UpdateSizes / GetPaneRect / RectOfRectid / YOfRectId / XOfRectId / FRectVisible
│   │   ├─ 滚动条：RefreshScrollbars / HideScrollbars / UpdateScrollbarsNow / RefreshScrollbar
│   │   ├─ 绘制：OnPaint / DrawSplitterBar / Paint3DFrame / Invalidate3DBorders
│   │   ├─ 拖拽：DragkindOfPt / StartDrag / EndDrag / DragSplitter / DropSplitter / Split
│   │   └─ 消息：OnWndMsg / SendMessageToPaneAtPt / OnIMEWindowMessage
│   ├─ 面板体系 (PANE)
│   │   ├─ PANE 基类：Paint / NotifyResized / NotifyFontChanged / ProcessWindowMessage / GetSizeInfo / Keydown / DoChar
│   │   ├─ CPANE 代码窗格（核心）
│   │   │   ├─ 创建：Create / Init / CPANE / ~CPANE
│   │   │   ├─ 绘制：Paint / PaintLines / PaintLine / PaintIMEComposition / DrawMarginWidgetBarBackground / FMarginWidgetOnLine
│   │   │   ├─ 光标/选择：RefreshCursorPos / DisplayCursorPos / FGetSelection / SetSelection / MoveIpAndSelect / CancelSelection / GetSelectionOnLine / GetLineOfSelection
│   │   │   ├─ 编辑：InsertChar / InsertNewLine / JoinLine / InsCh / DelCh / DeleteRange / IndentOutdent / CommentBlock / InsertAutoEnd
│   │   │   ├─ 导航：CurUp / CurDown / CurLeft / CurRight / CurHome / CurEnd / CurPageUp / CurPageDown / MoveByWord / CurTab / CurBackspace / CurDel
│   │   │   ├─ 坐标：XFromIch / IchFromX / YFromLn / LnFromY / DichWidth / LnTopLast
│   │   │   ├─ 滚动：Scroll / SetLnTop / SetIchLeft / AdjustIchLeft / AdjustLnTop / Page
│   │   │   ├─ 剪贴板/拖放：Paste / CanPaste / CanCopy / OperateOnSelection / SerializeSelection / Deserialize / GetOleDropEffect / OnOleDrop
│   │   │   ├─ 事务：BeginEditXact / EndEditXact / BeforeEndEditXact / AfterEndEditXact
│   │   │   ├─ 值提示：GetValueString
│   │   │   └─ IME：GetIMECompositionOnLine / DoIMEComposition / DetermineIMEComposition / GetIMEWindow
│   │   ├─ NPANE 过程/对象下拉框
│   │   ├─ LPANE 列表窗格基类（DrawItem / StartLIEnum / NextLIEnum / ToggleExpansion / StartEditInPlace）
│   │   ├─ VLPANE / WLPANE / LLPANE   监视/局部变量窗格
│   │   ├─ SPLITTER_EBWND / SPLITTER_PANE   分割窗口
│   │   └─ PANEMGR                      面板管理器
│   ├─ 工具栏
│   │   ├─ OBTBAR：Init / Paint / PaintBtn / OnWndMsg / BtnOfPoint / SetBtnState / InvalidateBtn / GetBtnRect / OnLButtonDown/Up
│   │   ├─ TOOLBAR_PANE / OBJBRW_TOOLBAR_PANE
│   │   └─ OBTBAR::S_EnsureInitialized / g_rgbtninfo
│   ├─ 弹出提示
│   │   ├─ POPUPMGR：ShowPopup / ShowNameList / ShowQuickTip / ReplaceCurrentWord / FKeydown / BeforeChar / DoChar / SelectCurWordInNameList
│   │   ├─ NAMELIST：EnsureInitialized / SelectName / Show / Move / Paint / M_DrawItem / ProcessListboxWndMsg
│   │   ├─ POPUPTIP：EnsureInitialized / Size / PositionWindow / Show / Paint / OnSysColorChange
│   │   └─ VALUETIP：OnNeedText / OnMouseMove / EnsureInitialized / Activate / SetOwner / ReleaseOwner / GetValueTipExpressionExtent / M_GetTime
│   ├─ 对象浏览器 (OBJBRW)
│   │   ├─ OBJBRW：M_Init / M_EnsureHaveSplitwin / M_LoadSymbols / M_ShowContextMenu / M_DrawItem / M_DrawSymbol / M_GoToSymbol / M_SelectSymbol / M_GoBack / M_GoForward / M_Search
│   │   ├─ OBJBRW_PANE：M_ProcessWmSize / M_GetSearchRect / M_EnsureHeaderFillsWindow / M_ProcessMessage / M_ProcessListboxWndMsg / M_Paint
│   │   ├─ BROWSEDATA：M_Init / M_SetSearchPattern / M_AddSymbol / M_GetInfo / M_SortSymbols / M_RemoveDuplicates / M_Visit / M_GetDescription / M_Append*
│   │   ├─ BRWSYM / BRWKIND / BRSPEC / SORTINFO  浏览器符号/类别
│   │   ├─ OBJECT_MEMBER / OBJECTCHILDREN / VALUETYPE / VARIABLE_ITEMDATA / LOCAL_ITEMDATA / BASICVALUE_ITEMDATA
│   │   └─ OB_LOC / OB_LOC_STACK / OBJBRW_PANE::M_NotifyHandler
│   ├─ 代码片段/智能感知
│   │   ├─ CODEFRAG：M_ProcessText / M_ClearAll / M_ClearArrays / M_SetNamedParam / M_AddTokenName / M_AddBms / M_AddArgs / M_AddArgsPartial / M_GetMemberAccess
│   │   ├─ GetSymbolUnderCursor / GetSymbolAtLocation / GetCodefragAtLocation
│   │   └─ Brwdata / FSourceContainsPattern / FindIdentifierOnLine
│   ├─ 视图/行/文本管理
│   │   ├─ VIEWMGR：Init / RegisterCodePane / RegisterLnotify / GetCpaneOfModvwr / SendNotification / GetCurrentStatement / GetExtOfCurStmt / EditRange / RouteScrln / InsertAndCommit / ShowLines / CommitAll
│   │   ├─ LINEMGR：Init / GetLine / GetPchOfLine / PCachelnOfScrln / Commit / ReplaceLine / InsertLine / DeleteLine / DeleteLines / LoglnOfScrln / ScrlnOfLogln / FilelnOfLogln
│   │   ├─ LCT / CACHELN / LNBUF / LNBUFZ / LNBUFCLIP / ISTRG
│   │   └─ LNSINK / LNOTIFY           行变更通知
│   ├─ 撤销/重做
│   │   ├─ XACTMGR：NewPhrase / OpenXact / CloseXact / AcceptXact / RollbackXact / CreateXactel / PlayXact / DoUndoOrRedo / CanUndoOrRedo / RecordEdit / RecordSelection
│   │   └─ XACTEL / XACTEDIT / XACTMODE / Undo / Redo / CanUndo / CanRedo
│   ├─ 打印
│   │   └─ PRTCODE：BeginPrint / EndPrint / GetNumPages / PrintPage / SetPage / PrintPhysicalLine / ClnpapOfLn / PrintPageLines / FitLineToPage / IchBeforeX
│   ├─ 搜索替换
│   │   ├─ FindNextOccurrence / SetSearchString / CheckPattern / PostReplaceBoundUpdate / ReplaceText
│   │   ├─ EbSearchReplace / EbSearchReplaceAll / EbSearchFind / EbSearchFindNext / EbSearchCanReplace / EbSearchGetCurState
│   │   └─ 正则引擎：RECompile / REMatch / REParse* / RECharType / RECharLen / RESetLcid
│   ├─ 帮助
│   │   └─ DoHelp / ValidateHelpFile / AddTypeLibsHelp / PollTypeLibs / SrchHelpInTypeLib / AddHelpForMember
│   └─ 全局 UI 状态：g_pobjbrw / g_pbdEditor / g_hfontNormal / g_hfontBold / g_hbrWindow / g_hbrHighlight / g_rgpbltter / g_hcrs*
│
├─ 10. 控件宿主 (ActiveX Control Host)
│   ├─ ExtendedControl                 控件包装
│   │   ├─ 子对象：RAW_CTRL_OBJECT（IDispatch 转发）/ RAW_CTRL_SINK（事件接收）/ PRIVATE_UNKNOWN（连接点容器/连接点）
│   │   ├─ ConnectSink / Zombify
│   │   └─ vftable + adjustor thunk（多重继承）
│   ├─ ExtendedControlCF               类工厂
│   │   └─ CreateExtendedControl / VBAGetExtendedControlCF
│   ├─ CVBAControlMgr                  控件管理器
│   │   ├─ Init / DefineControl / UndefineControl / RemoveDeletedControls / HideControl / ForceControlCreation
│   │   ├─ ControlAddToIndexTable / GetIndexOfControl / SwapEntries / ValidateControl
│   │   ├─ CONTROL_INFO：GetTypeInfo / SetName / GetName / GetParent / Hmvdefn
│   │   └─ ProvideControlsForTemplate
│   ├─ WRAPPER_EVENT_SINK              事件接收包装
│   ├─ VBAExtension                    扩展模块
│   │   ├─ CreateStandalone / Create / AttachModule / EnsureInRunMode / EnsureInDesignMode / ReleaseRuntimeInst / SyncCtrlCookies / ZombieMe
│   │   └─ PRIVATE_UNKNOWN：GetBasicTypeInfo
│   └─ 全局：FormControlAttributeLiteral / ParseControlAttributeLiteral / SetSinkCommon
│
├─ 11. COM/OLE 基础设施 (COM/OLE Infrastructure)
│   ├─ 连接点
│   │   ├─ CConPntCont / CConPnt / CEnumConPnts / CEnumCons / ConCache
│   │   └─ RemoveConnectionsOfBasicClass / RaiseEventOnBasicClass / TipAdviseVBAInstPart / CONNECTION_NODE
│   ├─ 拖放
│   │   ├─ CDropSource / CDropTarget / DoDragDrop / RegisterDragDrop / RevokeDragDrop
│   │   └─ FilterDropEffect / SetPaneDragSourceCur / PCpaneDragSourceCur
│   ├─ 剪贴板/数据对象
│   │   ├─ CDataObject / CDataSource / CEnumFormatEtc / DATACACHE_ENTRY
│   │   └─ OleSetClipboard / OleGetClipboard / OleFlushClipboard / ReleaseStgMedium
│   ├─ 流
│   │   ├─ CFileStream / CStreamBuf / CCompressedStreamBuf
│   │   └─ CreateStreamOnHGlobal / StgCreateDocfile / CreateILockBytesOnHGlobal
│   ├─ 错误对象
│   │   ├─ CSupportErrorInfo / IErrorInfo / SetErrorInfo / GetErrorInfo / CreateErrorInfo
│   │   └─ FSupportsErrorInfo / SetRichErrorOfHr
│   ├─ 标准接口桩 vftable：IUnknown / IDispatch / ITypeLib / ITypeInfo / ITypeComp / ICreateTypeLib / ICreateTypeInfo / IProvideClassInfo / ISupportErrorInfo / IExternalConnection / IPersistStorage / IConnectionPoint* / IEnumConnection* / IEnumVARIANT / IDropSource / IDropTarget / IDataObject / IEnumFORMATETC / IStream / IClassFactory / IClassFactory2 / IPropertyNotifySink / IErrorInfo / ITypeChangeEvents / IProxyManager / IORPCDebugNotify
│   ├─ 调试：COrpcDebugHandlers / InitializeORPCDebugging
│   └─ 全局：_IID_* / _CLSID_* / _GUID_* / _g_guidVBA / _IID_StdOle
│
├─ 12. 构建与链接 (Build & Link)
│   ├─ buildProject：Create / Destroy / CompileModule / DecompileModule / AddModule / DeleteModule / SwapModules / SaveRDT / StartEXE / FinishEXE / PstlOfPtlib / BuildEtypeOfBiltyp
│   ├─ buildModule：Alloc / Realloc / Free / AddMember / GetRTMI / SetRTMI / BuildVariable / BuildConstant / BuildUDT / BuildMethod / BuildDeclare / BuildEventDecl / BuildRDT / GetBilimportOf*
│   ├─ CExeFile                           PE 写出
│   │   ├─ Init / NewExePart / NewResourcePart / SetExePartFlags / WriteExePart / WriteExePartAsStream / WriteExePartAsFile
│   │   ├─ WriteDllFixup / WriteInternalFixup / WriteBSSFixup / WriteNameFixup / WriteExport
│   │   ├─ WriteRsrcDir / ApplyFixup / CalcMaxSize / WriteExe
│   │   └─ SetInitialIP / AllocBSS / SetExeInfo / SetBaseAddress / AddDirective / Write
│   ├─ EXE 辅助：EXEOffsetOfPv / EXEFixup* / EXEBlockOfSection / EXEWritePeproj / EXEWritePemod / EXEFixupsOfMemid
│   ├─ 资源：CompleteResource / CompleteModule / CompleteProject / CompleteDllFunction / ReleaseLibrary / ResolveAllImports / MakeCallbackThunk
│   └─ 全局：g_pExeFile / g_exeroot / g_hexepartStart / _exeheader / _rgsectheader / _symboltable / _rgbImportPogoTemplate
│
├─ 13. 持久化与内存 (Persistence & Memory)
│   ├─ 内存分配器
│   │   ├─ norls_allocator：nraInit / nraAlloc / nraDone / nraFree / nraToss
│   │   ├─ commitAllocator：cmaInitT / cmaMore / cmaFree
│   │   ├─ small_block_allocator：sbaInit / sbaAllocBlock
│   │   ├─ large_block_allocator：lbaInit / lbaAllocMore
│   │   ├─ block_allocator：baGetM / baGet0
│   │   └─ SHEAP_MGR / BLK_DESC / BLK_MGR / BLKMGR32 / DYNBLKMGR1632 / memSimpleAlloc / memHeapAlloc / memAlloc / memSection
│   ├─ 序列化
│   │   ├─ ReadBstr / ReadBstrA / ReadByte / ReadCy / ReadDec / ReadGuid / ReadN / ReadSz / ReadUShort / ReadULong / ReadVariant
│   │   ├─ WriteBstr / WriteByte / WriteCy / WriteDec / WriteGuid / WriteN / WriteSz / WriteUShort / WriteULong / WriteVariant / WriteWString
│   │   └─ TrapWrite* / GetPos / Rewind / SetSize / SetPos
│   ├─ 压缩
│   │   └─ CompressWorkSpaceSizeLZNT1 / CompressBufferLZNT1 / DecompressBufferLZNT1 / LZNT1CompressChunk / LZNT1DecompressChunk / LZNT1FindMatchStandard
│   └─ 平衡二叉树
│       └─ BBTREE：LeftRotate / RightRotate / UnrollAndBalance / AddHelper / BBAdd / BBAddUnique / FindHelper / BBFind / RecursiveDestroy / Clear
│
├─ 14. 国际化与 DBCS (Internationalization)
│   ├─ 标识符字符集：DBisIdentBegJapan/Korea/Taiwan/PRC / DBisIdentMid* / charToType / typeChar2char / narrowString / mapDBCStoSBCS
│   ├─ 排序/大小写：CompareStringA/W / LCMapStringA/W / GetStringTypeA/W / CharUpperBuffA/W / CharLowerBuffA/W / IntlProperWidth
│   ├─ 字符集表：_DBisIdentBegTab / _DBisIdentMidTab / _charFlags / _charTypes / _typeCharChars / g_leadByteTable
│   ├─ 区域：g_UserLCID / g_SystemLangId / g_UserLangId / g_iLang / g_rgLanguages / _cpidListWin/Mac/Dos* / InitLocaleStruct / InitMonths / InitDays
│   ├─ IME：_pImmGetContext / _pImmSetCompositionWindow / _pImmGetCompositionString / _pImmSetCandidateWindow / _pEndIMEShare / _pFInitIMEShare
│   └─ 正则 DBCS：REMatch / RECharType / RECharLen / RESetLcid / _mbsnccnt_jpacc / MovePBackwards / fIsLigature
│
├─ 15. CRT 静态库 (C Runtime)
│   ├─ 启动/退出：_CRT_INIT / _DllMainCRTStartup / __amsg_exit / _doexit / __onexit / _atexit / ___onexitinit
│   ├─ 堆：_malloc / _free / _realloc / _calloc / __nh_malloc / __heap_alloc / __msize
│   ├─ 字符串：_strlen / _strcpy / _strcat / _strstr / _strchr / _strrchr / _strpbrk / _strspn / _strcspn / _strncat / _strtok / _strncpy / _strncmp / __strcmpi / __strnicmp / __strdup / __strupr / _wcscmp / _wcslen / _wcscpy / _wcscat / _wcschr / _wcsrchr / _wcsncpy / _wcsncat / _wcsncmp / __wcsicmp / __wcsnicmp
│   ├─ 转换：_atoi / _atol / __itoa / __ltoa / __ultoa / __itow / __ltow / __ultow / _strtol / _strtoul / _wcstoul / _wcstoxl / _sprintf / __snprintf / _sscanf / _swprintf / __vsnprintf
│   ├─ 时间：_time / ___tzset / __tzset_lk / __isindst / _cvtdate / ___loctotime_t / ___timet_from_ft
│   ├─ 文件：__open / __sopen / __read / __write / __lseek / __close / __commit / __stat / __getcwd / __getdcwd / __chdir / __mkdir / __rmdir / _remove / _rename / __access / __findfirst / __findnext / __findclose / __getdrive / __chdrive
│   ├─ 浮点：_pow / _log / _log10 / _exp / _floor / _modf / __CIpow / __CIlog / __CIexp / __cfltcvt / __cftoe / __cftof / __cftog / __fassign / __forcdecpt / __cropzeros / __fptostr / __fltout2 / ___dtold / __control87 / __controlfp / __abstract_cw / __hw_cw / __ld12cvt / __atodbl / __atoflt
│   ├─ 定点：__vbaDecMul / __vbaDecDiv / __vbaDecAdd / __vbaDecSub / __vbaDecCmp / __vbaDecFix / __vbaDecInt / __vbaDecNeg / __vbaDecAbs
│   ├─ 长整数：__allmul / __aulldiv / __aullrem / ___addl / ___add_12 / ___shl_12 / ___shr_12 / ___mtold12 / ___strgtold12
│   ├─ 异常：__setjmp3 / _longjmp / __global_unwind2 / __local_unwind2 / __except_handler3 / _seh_longjmp_unwind / __abnormal_termination / __NLG_Notify / __NLG_Dispatch / __NLG_Return2
│   ├─ 多字节：_mbtowc / __mbtowc_lk / _wctomb / __wctomb_lk / _wcstombs / __wcstombs_lk / __mbschr / __mbsrchr / __mbspbrk / __mbsnccnt / __mbsnbcnt / __mbsinc / __ismbblead / __mbslen / __mbsncat / __mbsnbcpy / __mbstok / __mbsicmp / __mbsnicmp / __mbscmp / __mbsstr / __mbsbtype / __mbbtype / __mbsspnp / __mbscpy / __mbscat
│   ├─ 区域 CRT：_setmbcp / _getSystemCP / _CPtoLCID / _setSBCS / ___initmbctable / ___crtGetEnvironmentStringsA / ___crtCompareStringA / ___crtLCMapStringA/W / ___crtGetStringTypeA/W / ___crtGetLocaleInfoA/W / ___getlocaleinfo
│   └─ 其他：_qsort / _shortsort / _swap / _rand / _srand / __mtinit / __mtterm / __getptd / __freeptd / __ioinit / __ioterm / __setenvp / __setargv / _parse_cmdline
│
└─ 16. 跨模块全局状态 (Global State)
    ├─ 编译器：g_ptdata / g_ptdata / parseHashTbl / labelIdent / lineNoIdent / parseLineFlags / genPcodeBuffAddr/Next/Last / Token / Value / tokenIsIdent / scanHashTbl / scannerLCID
    ├─ 符号/类型：g_symWorld / g_symProjIntrinsics / g_symProc / g_pbtRoot / g_pexmgr / g_pimpmgr / g_pbilalloc / GlobalAllocator / g_rgIntrinsicTypes / var_type2_VARTYPE / g_opcode_info / g_rgexinfo / g_rgoperinfo
    ├─ 运行时：g_currentErrorTrap / g_Ebapp / g_EbThread / g_cThreads / g_cExprSrv / g_fVbcInitialized / g_fLoadingProject / g_projmgr
    ├─ UI：g_pobjbrw / g_pbdEditor / g_pbdMemberQuery / g_hfontNormal/Bold/StrikeOut / g_hbrWindow/Highlight / g_rgpbltter / g_hcrs* / g_cySplitBar / g_cxy3dFrame / c_szObtbarWndClass / c_szObPaneClass
    ├─ 国际化：g_UserLCID / g_SystemLangId / g_UserLangId / g_iLang / g_leadByteTable / _DBisIdentBegTab / _DBisIdentMidTab / _charFlags / _charTypes
    ├─ 构建：g_pExeFile / g_exeroot / g_hexepartStart / _exeheader / _rgsectheader / _symboltable
    └─ 选项：g_szIni* 系列 / g_hkey / g_fChicago / g_fUseChicagoUI
```

## 一句话总览

> **这个 DLL 是一个完整的 VBA 运行时 + IDE：底部是自建内存分配器与 CRT 静态库，向上依次是持久化/压缩、类型系统（TYPE_DATA/IMPMGR/TYPEMGR/NAMMGR）、符号系统（FED*）、编译器前端（scanner→parser→genOpc→EXMGR/EXGEN→EXTBL）、运行时解释器（ProcEntry/EX* 指令/rtc* 库/RUN_INSTMGR 僵尸化）、调试器（WATCHMGR/BPMGR/中断模式）、IDE 前端（MODVWR/CPANE/PANE/LPANE/OBJBRW/POPUPMGR/VALUETIP/XACTMGR/搜索替换/打印/帮助）、控件宿主（ExtendedControl/CVBAControlMgr/VBAExtension）、COM/OLE 基础设施（连接点/拖放/剪贴板/流/错误对象）、构建链接（buildProject/buildModule/CExeFile）、国际化和 DBCS 支持，全部通过 EBAPP/EBTHREAD 宿主层暴露给 Office/VB 宿主，导出 `Eb*`/`Tip*`/`rtc*` 供外部调用。**
