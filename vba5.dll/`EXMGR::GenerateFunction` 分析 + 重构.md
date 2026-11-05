# `EXMGR::GenerateFunction` 分析 + 重构

## 一、函数定位

这是 VB 编译器 **逐函数代码生成核心**：给定一个函数定义句柄 `hfdefn`（相对函数记录表的字节偏移），它完成：

1. 逐行把源码编译成 **BIL 语句树**（`CompileLine`）；
2. 按函数体种类（普通/EnC 旧块/模块级代码体/无代码）走不同的 **excode 代码块准备**路径；
3. 收尾时发出 **RTMI**（运行时方法信息），必要时做 EnC 修正或通知模块编译完成；
4. 若启用 Native 模式，再 **BIL → LoBil → 机器码**（`TransformToLoBil` + `nativeGenProcedure`）。

它是 `EXMGR::GenerateModule` 遍历函数表时调用的执行者，内部完全靠 **`__trapCreate`/`_setjmp3`/`__trapThrow`/`__trapRemove`** 做异常保护，并在失败时统一走恢复链。

---

## 二、关键数据结构

### 2.1 函数记录 `FunctionRecord`（`hfdefn` 偏移）

| 偏移 | 类型 | 名称 | 含义 |
|---|---|---|---|
| +0 | BYTE | `headByte` | bit7 = `fNoCode`（Declare/空过程等） |
| +18 | WORD | `frameSize` | 栈帧大小（结束时回写） |
| +27 | BYTE | `lineCount` | 函数占用源码行数 |
| +28 | WORD | `firstLine` | 起始源码行号 |
| +32 | DWORD | `stmtHash` | 语句哈希句柄（`-1` 无效） |
| +38 | BYTE | `flags` | 状态标志 |
| +44 | BYTE | `encByte44` | EnC 相关（缺 End 判断用） |
| +58 | BYTE | `encByte58` | EnC 相关 |

`flags`（`rec->flags`）位定义：

| 位 | 名称 | 含义 |
|---|---|---|
| bit0-1 | `bodyKind` | 函数体种类 |
| bit3 (0x08) | `hasExcodeBlock` | 已有 excode 代码块 |
| bit4 (0x10) | `completed` | 已完成 |
| bit5 (0x20) | `hasSavedState` | 存在已保存的函数编译状态 |

### 2.2 函数体种类 `BodyKind`

| 值 | 名称 | 路径 |
|---|---|---|
| 0 | `BODY_NORMAL` | 普通过程：按 `hasSavedState && !completed` 决定"恢复"还是"重建块" |
| 1 | `BODY_ENC_OLD` | 已有旧 excode 块，需 EnC 修正（`SaveFunctionState` + EnC fixup） |
| 2 | `BODY_MODULE_LEVEL` | 模块级代码体（`Sub New`/模块初始化等），先取旧 RTMI 以便替换 |
| 3 | `BODY_NO_CODE` | 不产生代码，直接跳到 Native 收尾 |

### 2.3 `EXMGR` 内部字段

| 字段 | 含义 |
|---|---|
| `this+6` | `OPCODE_MGR`（行管理器） |
| `this+8` | `BASIC_TYPEROOT*`（当前模块） |
| `this+9` | `BASIC_TYPESRC*`（源码/行变更） |
| `this+10` | `EXTBL*`（excode 代码块管理器） |
| `this+11` | 函数记录表（表首 dword = 记录区基址） |
| `this+31` | `optFlags`（bit0 = fNoCode, bit5 = 已进入函数体内） |
| `this+50` | 当前编译行号 |
| `this+57/76/228` | 当前 `hfdefn` |
| `this+77` | 当前语句哈希 |
| `this+124/125/126` | EXMGR 内部状态标志 |
| `this+126 & 0x20` | **本地代码（Native）模式** |
| `this+128` | 语句列表输出（给 `TransformToLoBil`） |
| `this+152` | `BILALLOC*` |
| `this+153` | 编译子状态（12 = 逐行编译中） |
| `this+608` | 当前帧大小 |
| `this+612` | 当前函数状态码 |

---

## 三、重构后的易读伪代码

```cpp
// EXMGR::GenerateFunction
//   hfdefn = 函数记录相对函数表基址的字节偏移
HRESULT EXMGR::GenerateFunction(EXMGR* self, UINT hfdefn, int /*unused*/)
{
    // ============================================================
    // 阶段 1：前置检查 + 陷阱建立
    // ============================================================
    if (StackRemaining() < 0x430)          // 剩余栈 < 1072 字节
        return VBERR_STACK_OVERFLOW;       // 0x800A001C（VB err 28）

    FunctionRecord* rec = self->GetFnRecord(hfdefn);   // *(this+44) + hfdefn
    ++self->compileNestDepth;              // this+24 的 +16

    TRAP_CONTEXT trap;
    if (__trapCreate(0x20, &trap) != 0 && _setjmp3(...) != 0) {
        hr = trapErrCode;
        goto ErrorRecovery;                // LABEL_61
    }

    // ============================================================
    // 阶段 2：读取函数记录 + 读取种类
    // ============================================================
    UINT  firstLine = rec->firstLine;
    UINT  lineCount = rec->lineCount;
    UINT  endLine   = firstLine + lineCount;
    BodyKind kind   = (BodyKind)(rec->flags & 3);
    DWORD stmtHash  = rec->stmtHash;

    BOOL  fEnded    = FALSE;              // 是否遇到 End Sub/End Function
    INT   lastFuncLine = 0;
    INT   frameSizeVal = -132;            // 缺省帧大小占位（后续由 RTMI 推算覆盖）
    INT   codeStartOffset;                // codeOffsets[0]：函数起始代码偏移
    RTMI* pPrevPrtmi = NULL;              // 种类 2 时保存旧 RTMI

    // 种类 3：直接去 Native 收尾
    if (kind == BODY_NO_CODE)
        goto NativeCodeGen;                // LABEL_101

    // ============================================================
    // 阶段 3：设置编译上下文
    // ============================================================
    // 3.1 初始化 optFlags
    {
        DWORD optFlags = self->optFlags;                        // this+31
        optFlags = (optFlags & ~1) | ((rec->headByte >> 7) & 1); // bit0 = fNoCode
        optFlags &= ~(1 << 5);                                   // 清 bit5：未进入函数体
        self->optFlags      = optFlags;
        self->currentLine   = firstLine;
    }

    // 3.2 分流：SaveFunctionState 还是 重建代码块
    BOOL useSavedState  = FALSE;
    BOOL reuseCodeBlock = FALSE;

    switch (kind) {
    case BODY_ENC_OLD:                     // 种类 1
        self->optFlags |= (1 << 5);        // 置 bit5：已进入函数体内
        useSavedState = TRUE;
        break;

    case BODY_NORMAL:                      // 种类 0
        if ((rec->flags & 0x20) && !(rec->flags & 0x10))
            useSavedState = TRUE;          // 有保存状态且未完成 → 复用
        else
            reuseCodeBlock = TRUE;         // 否则重建块
        break;

    case BODY_MODULE_LEVEL:                // 种类 2
        pPrevPrtmi = PrtmiOfHfdefn(self->module, hfdefn);  // 先记旧 RTMI
        reuseCodeBlock = TRUE;
        break;
    }

    if (useSavedState) {
        // ---------- 路径 A：SaveFunctionState ----------
        if (self->module->kind >= 4)
            self->flags125 = (self->flags125 & 0x3F) | 0x40;
        else
            self->flags125 &= 0x3F;

        compileState = 5;                   // 编译起点状态码

        if (FAILED(hr = EXMGR::SaveFunctionState(self, hfdefn)))
            goto Cleanup_WithError;         // LABEL_125

        BOOL fDebugInfo = (self->module->flags164 & 0xC0) != 0;
        if (FAILED(hr = EXTBL::BeginFuncNew(self->excode,
                                           (rec->flags & 4) >> 2,
                                           fDebugInfo)))
            goto Cleanup_WithError;

        frameSizeVal      = rec->frameSize;  // 继承已有帧大小
        EXMGR::HashSet(self, stmtHash);
        self->currentStmtHash = stmtHash;    // this+77
    }
    else { // reuseCodeBlock
        // ---------- 路径 B：重建代码块 ----------
        if (!(rec->hasExcodeBlock) || kind == BODY_MODULE_LEVEL) {
            // 无旧块 / 模块体：不释放旧块（模块体可能有历史块）
            if (self->module->kind >= 4)
                self->optFlags |= (1 << 23); // 高字节 bit7
        } else {
            // 有旧块：先释放，再按需置高字节 bit6
            EXTBL::FreeExcodeBlock(self->excode, hfdefn);
            if (self->module->kind >= 4)
                self->optFlags |= (1 << 22); // 高字节 bit6
        }
        self->optFlags = self->optFlags;
        self->flags125 &= 0x3F;

        // 丢弃旧哈希 + 新建哈希
        compileState = 7;
        if (stmtHash != (DWORD)-1)
            HashDestroy(self->fnTable, stmtHash, hfdefn);
        if (FAILED(hr = EXMGR::HashCreate(self, &stmtHash)))
            goto Cleanup_WithError;

        rec->stmtHash = stmtHash;

        BOOL fDebugInfo = (self->module->flags164 & 0xC0) != 0;
        if (FAILED(hr = EXTBL::BeginFuncNew(self->excode,
                                           (rec->flags & 4) >> 2,
                                           fDebugInfo)))
            goto Cleanup_WithError;

        EXMGR::HashSet(self, stmtHash);
        self->currentStmtHash = stmtHash;
    }

    // ============================================================
    // 阶段 4：逐行编译
    // ============================================================
    self->frameSize       = frameSizeVal;          // this+608
    self->currentHfdefn   = hfdefn;                // this+228
    self->stateA          = hfdefn;                // this+76
    self->stateB          = -1;
    self->stateC          = -1;
    self->compileState2   = 12;                    // this+612

    codeStartOffset = self->excode->codeSize;      // pExcode+12
    INT fnCodeStart  = codeStartOffset;

    while (self->currentLine < endLine) {
        // 进度回调（每 50 行一次）
        if ((g_ideCallbackEnabled & 0x20) && lineCount % 50 == 0 && lineCount > 50) {
            if (!(*g_ideCallback)()) {
                hr = 0x800A9C64;                   // 用户取消
                goto ErrorRecovery;
            }
        }

        OP_LINE line;
        OPCODE_MGR::GetLine(self->opcodeMgr, self->currentLine, &line);

        if (line.isNonCode() || (line.flags2 & 1)) {
            // 非代码行 / 本行不产码 → 尺寸 0
            SetLineCodeSize(self, self->currentLine, 0);
        } else {
            // 编译一行
            if (FAILED(hr = EXMGR::CompileLine(self, compileState, &line, &fEnded)))
                goto ErrorRecovery;

            INT lineSize = 0;
            if ((self->flags126 & 0x20) == 0)    // 非 Native 模式才计尺寸
                lineSize = self->excode->codeSize - fnCodeStart;

            fnCodeStart += lineSize;
            SetLineCodeSize(self, self->currentLine, lineSize);

            if (fEnded) {
                // 函数体结束：检查剩余行是否只含空行/注释
                lastFuncLine      = self->currentLine;
                self->currentLine++;
                if (self->currentLine >= endLine) break;

                while (1) {
                    OPCODE_MGR::GetLine(self->opcodeMgr, self->currentLine, &line);
                    if (!line.isNonCode()) {
                        // 后面还有可执行代码 → 走 CheckBetweenFuncs 报错
                        self->lineStateA  = 0;
                        self->lineAux8    = line.aux8;
                        self->lineStateB  = -1;
                        self->lineAux16   = line.aux16;
                        self->lineStateC  = 0xFFFF;
                        if (FAILED(hr = EXMGR::CheckBetweenFuncs(self)))
                            goto ErrorRecovery_119;
                    }
                    SetLineCodeSize(self, self->currentLine, 0);
                    if (++self->currentLine >= endLine)
                        goto CheckFuncEnded;    // LABEL_66
                }
            }
        }

        self->currentLine++;
        --lineCount;
    }

CheckFuncEnded:   // LABEL_66
    if (!fEnded) {
        // 始终未遇到 End Sub/End Function
        rec = self->GetFnRecord(hfdefn);
        if (rec->encByte1 & 0x10) {
            BOOL bExpectEnd = (rec->encByte58 & 8) == 0
                              ? (rec->something == -1)
                              : ((rec->encByte44 & 0x3F) == 0x18 &&
                                 (rec->encByte45 & 7) == 0);
            hr = EXMGR::CreateError(self, bExpectEnd ? 0x9C8F : 0x9C90);
        } else {
            hr = EXMGR::CreateError(self, 0x9CBE);
        }
        goto Cleanup_WithTrap;
    }

    // ============================================================
    // 阶段 5：RTMI 收尾
    // ============================================================
    rec = self->GetFnRecord(hfdefn);
    self->lineStateA       = 0;
    self->currentStmtLine  = lastFuncLine;

    if (kind != BODY_NORMAL || rec->hasExcodeBlock) {
        // 走 RTMI 分支：kind 可能为 1、2，或 0 但携带旧块
        if (kind == BODY_ENC_OLD) {
            // ---------- 种类 1：EnC 修正 ----------
            FEDSYM fedFn;
            FEDSYM::BeFuncdefn(&fedFn, self->module, hfdefn);
            if (self->exgenStateValid) {
                EXMGR::GetExGenState(self, &exgen);
                EXGEN::ExgenFunction(&fedFn, &exgen, self->curFuncState);
            }

            // 4 字节对齐 excode +6 字段
            codeStartOffset = (self->excode->field6 + 3) & 0xFFFC;
            self->excode->field6 = codeStartOffset;

            // 取旧 RTMI
            RTMI* pOld = EXTBL::PrtmiOldExcodeBlock(self->excode, hfdefn);
            INT oldCodeSize = pOld->codeSize;
            INT oldFrame    = pOld->frameSize;
            INT oldFlags    = pOld->flags;

            if (FAILED(hr = EXMGR::SetAndEmitRTMI(self, hfdefn, codeStartOffset,
                                                  codeStartOffset_orig, stmtHash, oldFrame)))
                goto Cleanup;

            // EnC 帧修正
            INT encFrameSize = -self->frameSizeVal - 129;
            encFrameSize = (encFrameSize & ~3) | ((127 - self->field304) & 0xFC);
            INT delta = encFrameSize - oldCodeSize;

            RTMI* pNew = (RTMI*)((char*)self->excode->base + codeStartOffset);
            pNew->flags |= oldFlags;

            FEDSYM fed2;
            FEDSYM::BeFuncdefn(&fed2, self->module, hfdefn);
            buildModule::SetRTMI(self->module->buildMod, &fed2, pNew);

            if (FAILED(hr = EXMGR::FixupFuncForContinue(self, hfdefn, delta)))
                goto Cleanup;

            self->excode->state20 = -1;
            self->excode->base    = NULL;
            if (self->excode->largeBos) EXTBL::ClearLargeBosTable(self->excode);

            EXMGR::DiscardSavedFunctionState(self);
            self->flags124 &= ~0x20;
            goto Finalize_RTMI;            // LABEL_96
        }
        else {
            // ---------- 种类 2：模块级代码体 ----------
            FEDSYM fedFn;
            FEDSYM::BeFuncdefn(&fedFn, self->module, hfdefn);
            if (self->exgenStateValid) {
                EXMGR::GetExGenState(self, &exgen);
                EXGEN::ExgenFunction(&fedFn, &exgen, self->curFuncState);
            }

            INT frameSize     = (self->state153 - 5) & 0xFC;
            INT alignedOffset = (self->excode->codeSize + 3) & 0xFFFC;
            self->excode->codeSize = alignedOffset;

            if (FAILED(hr = EXMGR::SetAndEmitRTMI(self, hfdefn, alignedOffset,
                                                  codeStartOffset, stmtHash, frameSize)))
                goto Cleanup;

            EXTBL::ShrinkExcodeBlock(self->excode);
            RTMI* pNew = (RTMI*)((char*)self->excode->base + alignedOffset);
            if (pPrevPrtmi) pNew->flags |= pPrevPrtmi->flags;

            FEDSYM fed2;
            FEDSYM::BeFuncdefn(&fed2, self->module, hfdefn);
            buildModule::SetRTMI(self->module->buildMod, &fed2, pNew);

            self->excode->state20 = -1;
            self->excode->base    = NULL;
            if (self->excode->largeBos) EXTBL::ClearLargeBosTable(self->excode);

            if (rec->hasSavedState) {
                EXMGR::DiscardSavedFunctionState(self);
                self->flags124 &= ~0x20;
            }
            goto Finalize_RTMI;            // LABEL_96
        }
    }
    else {
        // ---------- 种类 0 且无代码块：EndFunc 收尾 ----------
        FEDSYM fedFn;
        FEDSYM::BeFuncdefn(&fedFn, self->module, hfdefn);

        // 仅当 optFlags 含 0x800000 且不含 0x200000 时才跑 ExgenFunction
        if ((self->optFlags & 0x800000) && !(self->optFlags & 0x200000)) {
            EXMGR::GetExGenState(self, &exgen);
            EXGEN::ExgenFunction(&fedFn, &exgen, self->curFuncState);
        }

        if (self->flags126 & 0x20) {
            // Native 模式：直接 EndFuncTemp；机器码稍后由 nativeGenProcedure 生成
            EXTBL::EndFuncTemp(self->excode);
            goto MarkBlockExists;          // LABEL_76
        }

        INT frameSize     = (self->state153 - 5) & 0xFC;
        INT alignedOffset = (self->excode->codeSize + 3) & 0xFFFC;
        self->excode->codeSize = alignedOffset;

        if (FAILED(hr = EXMGR::SetAndEmitRTMI(self, hfdefn, alignedOffset,
                                              codeStartOffset, stmtHash, frameSize)))
            goto Cleanup;

        if (FAILED(hr = EXTBL::EndFunc(self->excode, hfdefn, alignedOffset, NULL)))
            goto Cleanup;

MarkBlockExists:   // LABEL_76
        rec->hasExcodeBlock = TRUE;        // rec[38] |= 8
        if (!rec->hasSavedState)
            goto Finalize_RTMI;
        else
            goto DiscardSaved;
    }

DiscardSaved:   // LABEL_95
    EXMGR::DiscardSavedFunctionState(self);
    self->flags124 &= ~0x20;

Finalize_RTMI:  // LABEL_96
    rec = self->GetFnRecord(hfdefn);
    rec->frameSize = self->frameSize;       // rec+18
    rec->flags |= 3;                        // 标记已生成代码
    rec->flags |= 0x20;                     // 保留"已保存状态"位

    BASIC_TYPESRC::ClearLineChangedRange(self->typeSrc,
                                         rec->firstLine,
                                         endLine - 1);

    self->lineStateA       = -1;
    self->lineStateB       = -1;
    self->lineState3       = -1;
    self->currentStmtHash  = -1;
    self->currentStmtLine  = 0xFFFF;
    self->flag194          = 0;
    self->flag96           = 0;

    if (pPrevPrtmi) {
        RTMI* pNew = PrtmiOfHfdefn(self->module, hfdefn);
        ExecReplacePrtmiInExframes(pPrevPrtmi, pNew);
    }

    if (kind == BODY_MODULE_LEVEL) {
        FEDSYM fedMod;
        FEDSYM::BeModule(&fedMod, self->module);
        hr = buildModule::NotifyModuleCompiled(self->module->buildMod, &fedMod);
        ExecUpdateImptblInExframes();
        if (FAILED(hr)) {
            BILALLOC::DoneBilTrees(self->bilAlloc);
            self->ResetCleanupFields();
            --self->compileNestDepth;
            TRAP_REMOVE();
            return hr;
        }
    }
    goto NativeCodeGen;

NativeCodeGen:  // LABEL_101
    {
        FEDSYM fedFn;
        FEDSYM::BeFuncdefn(&fedFn, self->module, hfdefn);

        if (self->flags126 & 0x20) {
            // 本地代码模式：BIL → LoBil → 机器码
            rec = self->GetFnRecord(hfdefn);
            stmtHash = rec->stmtHash;
            EXMGR::HashSet(self, stmtHash);
            self->currentStmtHash = stmtHash;

            TransformToLoBil(self->module, self->bilAlloc, &fedFn, &self->stmtList);
            if (FAILED(hr = nativeGenProcedure(&fedFn, self->curFuncState)))
                __trapThrow(hr);

            self->lineStateA      = -1;
            self->lineState3      = -1;
            self->currentStmtHash = -1;
        }

        // 成功收尾
        BILALLOC::DoneBilTrees(self->bilAlloc);
        self->ResetCleanupFields();
        --self->compileNestDepth;
        TRAP_REMOVE();
        return 0;
    }

// ================================================================
// 错误恢复 / 清理路径
// ================================================================

ErrorRecovery:   // LABEL_61
    EXMGR::CntlCleanStack(self);
    if (self->flags124 & 0x40) EXMGR::RestoreContext(self);
    if (self->field78 != -1)   EXMGR::TpoolReset(self);

ErrorRecovery_119:   // LABEL_119
    rec = self->GetFnRecord(hfdefn);
    if (rec->hasSavedState) {
        EXMGR::RestoreOldFunctionState(self, hfdefn);
        self->flags124 &= ~0x20;
    } else if (kind == BODY_MODULE_LEVEL && stmtHash != (DWORD)-1) {
        HashDestroy(self->fnTable, stmtHash, hfdefn);
        rec->stmtHash = -1;
        rec->flags   &= ~0x20;
        rec->flags   &= ~0x10;
    }
    goto Cleanup;

Cleanup:   // LABEL_124
    self->lineStateA      = -1;
    self->lineStateB      = -1;
    self->currentStmtHash = -1;
    self->flag194         = 0;
    self->flag96          = 0;
    // 落到 Cleanup_WithError

Cleanup_WithError:   // LABEL_125
    if (pPrevPrtmi) {
        // 回写首行尺寸（用旧 RTMI 的 linesize）
        INT lineTbl = self->opcodeMgr->lineTable;
        INT oldLineSize = pPrevPrtmi->lineSize;
        rec = self->GetFnRecord(hfdefn);
        SetLineCodeSize(self, rec->firstLine, oldLineSize);
    }

    --self->compileNestDepth;
    BILALLOC::DoneBilTrees(self->bilAlloc);
    self->ResetCleanupFields();

    if (hr != 0x800A9C64) {              // 用户取消码不再转换
        UINT vbErr = EberrOfHresult(hr);
        hr = EXMGR::CreateError(self, vbErr);
    }
    if ((INT)self->flags124 < 0)
        *(DWORD*)(g_errContext + 9) = self->field53;

    self->lineStateB      = -1;
    self->currentStmtLine = 0xFFFF;
    EXTBL::EndFuncTemp(self->excode);

ReturnHr:   // LABEL_132
    TRAP_REMOVE();
    return hr;
}
```

---

## 四、主要分支一览

| 阶段 | 分流条件 | 路径 |
|---|---|---|
| 栈检查 | 剩余 < 0x430 | 返回 `0x800A001C`（栈溢出） |
| 种类判断 | `kind == 3` | 直接 Native 收尾（LABEL_101） |
| 块准备 | `kind == 1` | `SaveFunctionState` + EnC 修正路径 |
| 块准备 | `kind == 0` 且 `hasSavedState && !completed` | `SaveFunctionState` 路径 |
| 块准备 | `kind == 2` | 记录旧 RTMI → 重建块 |
| 块准备 | `kind == 0` 其他 | 重建块 |
| 逐行 | 非代码行 / `flags2 & 1` | 尺寸 = 0 |
| 逐行 | 代码行 | `CompileLine` → 记尺寸 |
| 函数结束 | `fEnded == TRUE` | 检查后续行 → 收尾 |
| 收尾 | `kind == 1` | EnC：`PrtmiOldExcodeBlock` + `SetAndEmitRTMI` + `FixupFuncForContinue` |
| 收尾 | `kind == 2` | 模块体：`SetAndEmitRTMI` + `ShrinkExcodeBlock` + `NotifyModuleCompiled` |
| 收尾 | `kind == 0` 无块 | `SetAndEmitRTMI` + `EndFunc` |
| 收尾 | Native 模式 | `EndFuncTemp` 直接收尾，机器码稍后生成 |
| Native | `flags126 & 0x20` | `TransformToLoBil` + `nativeGenProcedure` |
| 失败 | `__trapThrow`/`CreateError` | `LABEL_61`→`LABEL_119`→`LABEL_124/125` |

---

## 五、关键要点

1. **异常保护**：整个函数被 `__trapCreate`/`_setjmp3` 包裹；正常路径 `__trapRemove` 后返回 `0`，失败路径先清理（`CntlCleanStack`/`RestoreContext`/`TpoolReset`），再统一转 VB 错误号返回。
2. **三处状态机**：
   - `FunctionRecord.flags`（bit0-1 kind / bit3 hasBlock / bit4 completed / bit5 hasSavedState）
   - `EXMGR.flags124/125/126`（EXMGR 内部编译状态）
   - `compileState`（5 = 复用已保存状态，7 = 重建）
3. **逐行编译的尺寸记账**：非代码行尺寸 0；代码行尺寸 = `excode->codeSize - fnCodeStart`，写入 `lineTable[line] +6`，供调试器行-地址映射。
4. **`kind != 0` 或已有块的函数必须发 RTMI**：`SetAndEmitRTMI` 后按种类做 EnC 修正或 shrink；`EndFunc`/`EndFuncTemp` 关闭代码块。
5. **Native 模式是最后一步**：只有 `flags126 & 0x20` 时才 `TransformToLoBil` + `nativeGenProcedure`，否则 BIL 树停留在 `this+152`，由 `BILALLOC::DoneBilTrees` 释放。
6. **进度回调**：每 50 行回调一次 IDE，允许用户取消（`hr = 0x800A9C64`，后续不再转错误号）。
7. **EnC 支持**：种类 1 分支通过 `PrtmiOldExcodeBlock`/`FixupFuncForContinue` 修正已编译过程；失败恢复路径里 `RestoreOldFunctionState`/`DiscardSavedFunctionState` 处理保存状态。

**一句话总结**：`EXMGR::GenerateFunction` 是 VB 编译器的“单函数代码生成执行器”，按函数种类分流准备 `excode` 代码块，逐行 `CompileLine` 生成 BIL 并记账行尺寸，收尾时发出/修正 RTMI 并通知模块，最后在 Native 模式下把 BIL 推进到 LoBil/机器码；异常路径统一清理并转 VB 错误号。
