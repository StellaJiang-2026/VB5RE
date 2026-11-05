# `EXMGR::GenerateFunction` —— 无 GOTO 重构

## 一、重构思路

原函数通过 12+ 处 `goto LABEL_xx` 在阶段之间跳转，包括正常路径、恢复路径和清理路径。为了完全去掉 GOTO，采用以下结构性改写：

1. **抽出一个上下文结构 `FnGenCtx`**，把原函数的 30+ 个局部变量收敛到一处，避免阶段函数依赖外层作用域；
2. **按阶段拆分为独立子函数**，每个子函数返回 `HRESULT`，调用者根据返回值决定是否进入错误处理；
3. **原 `LABEL_61/119/124/125` 四个清理块合并进两个函数**：
   - `ErrorRecovery_Restore`：处理 `LABEL_119`（恢复保存状态 / 丢弃哈希）；
   - `ErrorRecovery_Cleanup`：处理 `LABEL_124 + LABEL_125`（清字段、回写、转换错误号、`EndFuncTemp`）；
   - 原 `LABEL_61` 的 `CntlCleanStack/RestoreContext/TpoolReset` 因幂等，并入 `ErrorRecovery_Cleanup`；
4. **原 `LABEL_39/66/76/95/96/101` 通过 `if/else + return` 或循环条件化消解**；
5. **Native 失败不再 `__trapThrow`**，而是返回 `hr` 由主函数统一走 `ErrorRecovery_Cleanup`；语义一致（原本 throw 后也是跳 `LABEL_61`），代价可忽略。

---

## 二、上下文结构

```cpp
struct FnGenCtx
{
    // 输入
    EXMGR*            self;
    UINT              hfdefn;
    FunctionRecord*   rec;

    // 阶段 2：从记录读出
    UINT              firstLine;
    UINT              lineCount;
    UINT              endLine;
    BodyKind          kind;          // rec->flags & 3
    DWORD             stmtHash;      // rec+32

    // 阶段 3：准备结果
    BYTE              compileState;  // 5 = 复用已保存状态；7 = 重建块

    // 阶段 4：逐行编译
    BOOL              fEnded;        // 遇到 End Sub/End Function
    INT               lastFuncLine;
    INT               fnCodeStart;   // 当前行尺寸基线
    INT               codeStartOffset; // codeOffsets[0]

    // 阶段 5：RTMI 收尾
    INT               frameSizeVal;  // rec+18（最终回写）
    RTMI*             pPrevPrtmi;    // 种类 2 时旧 RTMI，收尾后替换异常帧记录
};
```

---

## 三、重构后的主函数（无 GOTO）

```cpp
HRESULT EXMGR::GenerateFunction(EXMGR* self, UINT hfdefn, int /*unused*/)
{
    // ---------- 阶段 0：栈检查 ----------
    if (StackRemaining() < 0x430)
        return VBERR_STACK_OVERFLOW;            // 0x800A001C（VB err 28）

    FnGenCtx ctx{};
    ctx.self = self;
    ctx.hfdefn = hfdefn;
    ctx.rec    = self->GetFnRecord(hfdefn);
    ctx.pPrevPrtmi = nullptr;
    ++self->compileNestDepth;                   // this+24 的 +16

    // ---------- trap 建立 ----------
    TRAP_CONTEXT trap;
    HRESULT hr = TrapEnter(&trap);              // __trapCreate + _setjmp3
    if (FAILED(hr))
    {
        // 原始异常入口 LABEL_61：统一走恢复 + 清理
        return ErrorRecovery_Restore(ctx, hr);  // 内含 LABEL_61/119/124/125
    }

    // ---------- 阶段 1-3：准备代码块 ----------
    hr = PrepareCodeBlock(ctx);
    if (FAILED(hr))
        return ErrorRecovery_Restore(ctx, hr);

    // ---------- 阶段 4：逐行编译（kind=3 跳过）----------
    if (ctx.kind != BODY_NO_CODE)
    {
        hr = CompileBodyLines(ctx);
        if (FAILED(hr))
            return ErrorRecovery_Restore(ctx, hr);

        hr = EnsureFunctionEnded(ctx);          // 原 LABEL_66 的未结束检查
        if (FAILED(hr))
            return ErrorRecovery_Restore(ctx, hr);

        // ---------- 阶段 5：RTMI 收尾 ----------
        hr = EmitRTMI(ctx);
        if (FAILED(hr))
            return ErrorRecovery_Restore(ctx, hr);
    }

    // ---------- 阶段 6：Native 代码生成 ----------
    hr = GenerateNativeCode(ctx);               // 内部失败返回 hr（原 __trapThrow）
    if (FAILED(hr))
        return ErrorRecovery_Restore(ctx, hr);

    // ---------- 成功收尾 ----------
    FinishSuccess(ctx);                         // 原 LABEL_101 尾部收尾
    TrapLeave(&trap);
    return S_OK;
}
```

---

## 四、阶段函数

### 4.1 `PrepareCodeBlock`（原 LABEL_23/LABEL_29/LABEL_31/LABEL_38/LABEL_39）

```cpp
HRESULT EXMGR::PrepareCodeBlock(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // 读取记录字段
    ctx.firstLine  = rec->firstLine;            // rec+28
    ctx.lineCount  = rec->lineCount;            // rec+27
    ctx.endLine    = ctx.firstLine + ctx.lineCount;
    ctx.kind       = (BodyKind)(rec->flags & 3);
    ctx.stmtHash   = rec->stmtHash;             // rec+32
    ctx.frameSizeVal = -132;                    // 缺省占位（后续由 RTMI 推算覆盖）
    ctx.fEnded     = FALSE;
    ctx.lastFuncLine = 0;

    // kind == 3：直接跳到 Native 阶段，不做任何块准备
    if (ctx.kind == BODY_NO_CODE)
        return S_OK;

    // 更新 optFlags：bit0 = headByte bit7（无需生成代码），清 bit5（未进入函数体）
    self->optFlags = (self->optFlags & ~1) | ((rec->headByte >> 7) & 1);
    self->optFlags &= ~(1 << 5);
    self->currentLine = ctx.firstLine;

    // ---------- 分流：useSavedState vs reuseCodeBlock ----------
    BOOL useSavedState = FALSE;
    BOOL reuseCodeBlock = FALSE;

    switch (ctx.kind)
    {
    case BODY_ENC_OLD:                          // 种类 1
        self->optFlags |= (1 << 5);             // 置 bit5：已进入函数体内
        useSavedState = TRUE;
        break;

    case BODY_NORMAL:                           // 种类 0
        if ((rec->flags & 0x20) && !(rec->flags & 0x10))
            useSavedState = TRUE;               // 有保存状态且未完成 → 复用
        else
            reuseCodeBlock = TRUE;              // 否则重建块
        break;

    case BODY_MODULE_LEVEL:                     // 种类 2：先记录旧 RTMI
        ctx.pPrevPrtmi = PrtmiOfHfdefn(self->module, ctx.hfdefn);
        reuseCodeBlock = TRUE;
        break;
    }

    if (useSavedState)
        return PrepareBySaveState(ctx);
    else
        return PrepareByReuseBlock(ctx);
}
```

### 4.2 `PrepareBySaveState`（复用已保存状态）

```cpp
HRESULT EXMGR::PrepareBySaveState(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // 模块种类 >= 4 时置 flags125 高字节 bit6
    if (self->module->kind >= 4)
        self->flags125 = (self->flags125 & 0x3F) | 0x40;
    else
        self->flags125 &= 0x3F;

    ctx.compileState = 5;                       // 编译起点状态码：复用

    HRESULT hr = EXMGR::SaveFunctionState(self, ctx.hfdefn);
    if (FAILED(hr))
        return hr;

    BOOL fDebugInfo = (self->module->flags164 & 0xC0) != 0;
    hr = EXTBL::BeginFuncNew(self->excode,
                             (rec->flags & 4) >> 2,
                             fDebugInfo);
    if (FAILED(hr))
        return hr;

    // 继承已有帧大小
    rec = self->GetFnRecord(ctx.hfdefn);
    ctx.frameSizeVal = rec->frameSize;

    EXMGR::HashSet(self, ctx.stmtHash);
    self->currentStmtHash = ctx.stmtHash;       // this+77
    self->frameSize       = ctx.frameSizeVal;   // this+608
    self->currentHfdefn   = ctx.hfdefn;         // this+228

    // 阶段 4 的入口初始化
    InitBodyLoopState(ctx);
    return S_OK;
}
```

### 4.3 `PrepareByReuseBlock`（重建代码块，原 LABEL_23/LABEL_29/LABEL_31）

```cpp
HRESULT EXMGR::PrepareByReuseBlock(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // ---------- 原 LABEL_23：按 hasBlock 决定是否 FreeExcodeBlock ----------
    BOOL needFreeOldBlock = (rec->flags & 0x08) && (ctx.kind != BODY_MODULE_LEVEL);
    if (needFreeOldBlock)
    {
        EXTBL::FreeExcodeBlock(self->excode, ctx.hfdefn);
        if (self->module->kind >= 4)
            self->optFlags |= (1 << 22);        // 高字节 bit6
    }
    else
    {
        if (self->module->kind >= 4)
            self->optFlags |= (1 << 23);        // 高字节 bit7
    }
    self->flags125 &= 0x3F;

    // ---------- 原 LABEL_31：丢弃旧哈希 + 新建哈希 ----------
    ctx.compileState = 7;

    if (ctx.stmtHash != (DWORD)-1)
        HashDestroy(self->fnTable, ctx.stmtHash, ctx.hfdefn);

    HRESULT hr = EXMGR::HashCreate(self, &ctx.stmtHash);
    if (FAILED(hr))
        return hr;

    rec->stmtHash = ctx.stmtHash;               // rec+32

    BOOL fDebugInfo = (self->module->flags164 & 0xC0) != 0;
    hr = EXTBL::BeginFuncNew(self->excode,
                             (rec->flags & 4) >> 2,
                             fDebugInfo);
    if (FAILED(hr))
        return hr;

    // 清 rec->flags 的 0x20（无保存状态）和 0x10（未完成）
    rec->flags &= ~0x20;
    rec->flags &= ~0x10;

    EXMGR::HashSet(self, ctx.stmtHash);
    self->currentStmtHash = ctx.stmtHash;
    self->frameSize       = ctx.frameSizeVal;   // this+608
    self->currentHfdefn   = ctx.hfdefn;

    InitBodyLoopState(ctx);
    return S_OK;
}
```

### 4.4 `InitBodyLoopState`（原 LABEL_39 的头部）

```cpp
void EXMGR::InitBodyLoopState(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;

    self->frameSize       = ctx.frameSizeVal;
    self->currentHfdefn   = ctx.hfdefn;
    self->state228        = ctx.hfdefn;
    self->state76         = -1;
    self->state84         = -1;
    self->compileState2   = 12;

    ctx.codeStartOffset   = self->excode->codeSize;  // pExcode+12
    ctx.fnCodeStart       = ctx.codeStartOffset;

    self->currentLine     = ctx.firstLine;
}
```

### 4.5 `CompileBodyLines`（原主循环 + 函数提前结束）

```cpp
HRESULT EXMGR::CompileBodyLines(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    while (self->currentLine < ctx.endLine)
    {
        // ---------- 进度回调（每 50 行）----------
        if ((g_ideCallbackEnabled & 0x20) != 0
            && (ctx.lineCount % 50) == 0
            && ctx.lineCount > 50)
        {
            if (!(*g_ideCallback)())
                return USER_CANCEL_CODE;        // 0x800A9C64：后续不再转错误号
        }

        OP_LINE line;
        OPCODE_MGR::GetLine(self->opcodeMgr, self->currentLine, &line);

        // ---------- 非代码行 / 本行不产码 ----------
        if (line.isNonCode() || (line.flags2 & 1))
        {
            SetLineCodeSize(self, self->currentLine, 0);
            ++self->currentLine;
            --ctx.lineCount;
            continue;
        }

        // ---------- 编译一行 ----------
        HRESULT hr = EXMGR::CompileLine(self,
                                        ctx.compileState,
                                        &line,
                                        &ctx.fEnded);
        if (FAILED(hr))
            return hr;

        INT lineSize = 0;
        if ((self->flags126 & 0x20) == 0)       // 非 Native 模式才计尺寸
            lineSize = self->excode->codeSize - ctx.fnCodeStart;

        ctx.fnCodeStart += lineSize;
        SetLineCodeSize(self, self->currentLine, lineSize);

        // ---------- 本行结束了函数体：进入"检查后续行"子循环 ----------
        if (ctx.fEnded)
        {
            ctx.lastFuncLine = self->currentLine;
            ++self->currentLine;

            HRESULT hrCheck = CheckTrailingLines(ctx);
            if (FAILED(hrCheck))
                return hrCheck;

            return S_OK;                        // 已到函数结束
        }

        ++self->currentLine;
        --ctx.lineCount;
    }
    return S_OK;                                // 循环走完但 fEnded 可能仍为 FALSE
}
```

### 4.6 `CheckTrailingLines`（原 fEnded 后的内层循环 + LABEL_66）

```cpp
HRESULT EXMGR::CheckTrailingLines(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;

    if (self->currentLine >= ctx.endLine)
        return S_OK;

    while (self->currentLine < ctx.endLine)
    {
        OP_LINE line;
        OPCODE_MGR::GetLine(self->opcodeMgr, self->currentLine, &line);

        if (!line.isNonCode())
        {
            // 后续行仍有可执行代码：调用 CheckBetweenFuncs 校验并报错
            self->lineStateA = 0;
            self->lineAux8   = line.aux8;
            self->lineStateB = -1;
            self->lineAux16  = line.aux16;
            self->lineStateC = 0xFFFF;

            HRESULT hr = EXMGR::CheckBetweenFuncs(self);
            if (FAILED(hr))
                return hr;
        }

        SetLineCodeSize(self, self->currentLine, 0);
        ++self->currentLine;
    }
    return S_OK;
}
```

### 4.7 `EnsureFunctionEnded`（原 LABEL_66 的未结束错误分支）

```cpp
HRESULT EXMGR::EnsureFunctionEnded(FnGenCtx& ctx)
{
    if (ctx.fEnded)
    {
        // 完成：记录最后一行 + 收尾状态
        ctx.self->lineStateB = -1;
        ctx.self->currentStmtLine = ctx.lastFuncLine;
        return S_OK;
    }

    // 始终未遇到 End Sub/End Function：按记录标志选择错误号
    FunctionRecord* rec = ctx.self->GetFnRecord(ctx.hfdefn);
    ctx.self->lineStateB      = -1;
    ctx.self->currentStmtLine = ctx.self->currentLine - 1;

    BOOL bExpectEnd;
    if (rec->encByte1 & 0x10)                    // 记录 +1 的 bit4
    {
        if ((rec->encByte58 & 8) == 0)
            bExpectEnd = (rec->encField == -1);
        else
            bExpectEnd = ((rec->encByte44 & 0x3F) == 0x18
                          && (rec->encByte45 & 7) == 0);
    }
    else
    {
        return EXMGR::CreateError(ctx.self, 0x9CBE);
    }
    return EXMGR::CreateError(ctx.self,
                              bExpectEnd ? 0x9C8F : 0x9C90);
}
```

### 4.8 `EmitRTMI`（原 RTMI 收尾分派）

```cpp
HRESULT EXMGR::EmitRTMI(FnGenCtx& ctx)
{
    FunctionRecord* rec = ctx.rec;

    // 分派：种类非 0 或已有代码块 → 走 RTMI 分支；否则 EndFunc
    if (ctx.kind != BODY_NORMAL || rec->flags & 0x08)
    {
        if (ctx.kind == BODY_ENC_OLD)
            return EmitRTMI_Enc(ctx);           // 种类 1：EnC 修正
        else
            return EmitRTMI_ModuleOrOldBlock(ctx); // 种类 2 / 种类 0 有块
    }
    return EmitRTMI_NormalEndFunc(ctx);         // 种类 0 无块
}
```

### 4.9 `EmitRTMI_Enc`（种类 1：EnC 修正）

```cpp
HRESULT EXMGR::EmitRTMI_Enc(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // 1. 构造 FEDSYM 函数符号
    FEDSYM fedFn;
    FEDSYM::BeFuncdefn(&fedFn, self->module, ctx.hfdefn);

    if (self->exgenStateValid)                  // *(this+126) < 0
    {
        EXGEN_STATE exgen;
        EXMGR::GetExGenState(self, &exgen);
        EXGEN::ExgenFunction(&fedFn, &exgen, self->curFuncState);
    }

    // 2. excode 代码块 4 字节对齐
    UINT codeAligned = (self->excode->field6 + 3) & 0xFFFC;
    self->excode->field6 = codeAligned;

    // 3. 取旧 RTMI 信息
    RTMI* pOld = EXTBL::PrtmiOldExcodeBlock(self->excode, ctx.hfdefn);
    INT   oldCodeSize = pOld->codeSize;
    INT   oldFrame    = pOld->frameSize;
    INT   oldFlags    = pOld->flags;

    // 4. 发出新 RTMI
    HRESULT hr = EXMGR::SetAndEmitRTMI(self, ctx.hfdefn, codeAligned,
                                        ctx.codeStartOffset,
                                        ctx.stmtHash, oldFrame);
    if (FAILED(hr))
        return hr;

    // 5. 计算 EnC 帧尺寸差
    INT encFrameSize = -self->frameSizeVal - 129;
    encFrameSize = (encFrameSize & ~3) | ((127 - self->field304) & 0xFC);
    INT delta = encFrameSize - oldCodeSize;

    // 6. 继承旧标志位
    RTMI* pNew = (RTMI*)((char*)self->excode->base + codeAligned);
    pNew->flags |= oldFlags;

    // 7. 挂到模块函数表
    FEDSYM fed2;
    FEDSYM::BeFuncdefn(&fed2, self->module, ctx.hfdefn);
    buildModule::SetRTMI(self->module->buildMod, &fed2, pNew);

    // 8. 修正 Continue 跳转
    hr = EXMGR::FixupFuncForContinue(self, ctx.hfdefn, delta);
    if (FAILED(hr))
    {
        self->excode->base = NULL;
        return hr;
    }

    // 9. 清空 excode 临时块
    self->excode->state20 = -1;
    self->excode->base    = NULL;
    if (self->excode->largeBos)
        EXTBL::ClearLargeBosTable(self->excode);

    // 10. 丢弃已保存状态
    EXMGR::DiscardSavedFunctionState(self);
    self->flags124 &= ~0x20;

    return FinalizeRTMICommon(ctx);
}
```

### 4.10 `EmitRTMI_ModuleOrOldBlock`（种类 2 / 种类 0 有块）

```cpp
HRESULT EXMGR::EmitRTMI_ModuleOrOldBlock(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // 1. 构造 FEDSYM 函数符号
    FEDSYM fedFn;
    FEDSYM::BeFuncdefn(&fedFn, self->module, ctx.hfdefn);

    if (self->exgenStateValid)
    {
        EXGEN_STATE exgen;
        EXMGR::GetExGenState(self, &exgen);
        EXGEN::ExgenFunction(&fedFn, &exgen, self->curFuncState);
    }

    // 2. 计算帧大小与对齐偏移
    INT frameSize     = (self->state153 - 5) & 0xFC;
    INT alignedOffset = (self->excode->codeSize + 3) & 0xFFFC;
    self->excode->codeSize = alignedOffset;

    // 3. 发 RTMI
    HRESULT hr = EXMGR::SetAndEmitRTMI(self, ctx.hfdefn, alignedOffset,
                                        ctx.codeStartOffset,
                                        ctx.stmtHash, frameSize);
    if (FAILED(hr))
        return hr;

    // 4. 收缩代码块
    EXTBL::ShrinkExcodeBlock(self->excode);

    // 5. 计算新 RTMI 并继承旧标志位
    RTMI* pNew = (RTMI*)((char*)self->excode->base + alignedOffset);
    if (ctx.pPrevPrtmi)
        pNew->flags |= ctx.pPrevPrtmi->flags;

    // 6. 挂到模块函数表
    FEDSYM fed2;
    FEDSYM::BeFuncdefn(&fed2, self->module, ctx.hfdefn);
    buildModule::SetRTMI(self->module->buildMod, &fed2, pNew);

    // 7. 清空 excode 临时块
    self->excode->state20 = -1;
    self->excode->base    = NULL;
    if (self->excode->largeBos)
        EXTBL::ClearLargeBosTable(self->excode);

    // 8. 有保存状态则丢弃
    if (rec->flags & 0x20)
    {
        EXMGR::DiscardSavedFunctionState(self);
        self->flags124 &= ~0x20;
    }

    return FinalizeRTMICommon(ctx);
}
```

### 4.11 `EmitRTMI_NormalEndFunc`（种类 0 无块）

```cpp
HRESULT EXMGR::EmitRTMI_NormalEndFunc(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // 1. 构造 FEDSYM 函数符号
    FEDSYM fedFn;
    FEDSYM::BeFuncdefn(&fedFn, self->module, ctx.hfdefn);

    // 2. 仅当 optFlags 含 0x800000 且不含 0x200000 时才跑 ExgenFunction
    if ((self->optFlags & 0x800000) && !(self->optFlags & 0x200000))
    {
        EXGEN_STATE exgen;
        EXMGR::GetExGenState(self, &exgen);
        EXGEN::ExgenFunction(&fedFn, &exgen, self->curFuncState);
    }

    // 3. Native 模式：直接 EndFuncTemp，机器码稍后由 nativeGenProcedure 生成
    if (self->flags126 & 0x20)
    {
        EXTBL::EndFuncTemp(self->excode);
        return MarkBlockExists(ctx);            // 原 LABEL_76
    }

    // 4. 非 Native：对齐 + 发 RTMI + EndFunc
    INT frameSize     = (self->state153 - 5) & 0xFC;
    INT alignedOffset = (self->excode->codeSize + 3) & 0xFFFC;
    self->excode->codeSize = alignedOffset;

    HRESULT hr = EXMGR::SetAndEmitRTMI(self, ctx.hfdefn, alignedOffset,
                                        ctx.codeStartOffset,
                                        ctx.stmtHash, frameSize);
    if (FAILED(hr))
        return hr;

    hr = EXTBL::EndFunc(self->excode, ctx.hfdefn, alignedOffset, NULL);
    if (FAILED(hr))
        return hr;

    return MarkBlockExists(ctx);                // 原 LABEL_76
}
```

### 4.12 `MarkBlockExists`（原 LABEL_76 / LABEL_95 / LABEL_96 前半）

```cpp
HRESULT EXMGR::MarkBlockExists(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = self->GetFnRecord(ctx.hfdefn);

    // 标记已有代码块
    rec->flags |= 0x08;

    // 有保存状态：丢弃；否则直接进共同收尾
    if (rec->flags & 0x20)
    {
        EXMGR::DiscardSavedFunctionState(self);
        self->flags124 &= ~0x20;
    }
    return FinalizeRTMICommon(ctx);
}
```

### 4.13 `FinalizeRTMICommon`（原 LABEL_96 + 阶段 6 入口）

```cpp
HRESULT EXMGR::FinalizeRTMICommon(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = self->GetFnRecord(ctx.hfdefn);

    // 回写栈帧大小
    rec->frameSize = self->frameSize;           // rec+18

    // 标记已生成代码 + 保留"已保存状态"位
    rec->flags |= 0x03;
    rec->flags |= 0x20;

    // 清除首行到末行的"已改动"标记
    BASIC_TYPESRC::ClearLineChangedRange(self->typeSrc,
                                         rec->firstLine,
                                         ctx.endLine - 1);

    // 复位当前函数上下文
    self->lineStateA       = -1;
    self->lineStateB       = -1;
    self->lineState3       = -1;
    self->currentStmtHash  = -1;
    self->currentStmtLine  = 0xFFFF;
    self->flag194          = 0;
    self->flag96           = 0;

    // 用新 RTMI 替换异常帧里的旧记录
    if (ctx.pPrevPrtmi)
    {
        RTMI* pNew = PrtmiOfHfdefn(self->module, ctx.hfdefn);
        ExecReplacePrtmiInExframes(ctx.pPrevPrtmi, pNew);
    }

    // 种类 2：通知模块代码体编译完成
    if (ctx.kind == BODY_MODULE_LEVEL)
    {
        FEDSYM fedMod;
        FEDSYM::BeModule(&fedMod, self->module);
        HRESULT hr = buildModule::NotifyModuleCompiled(self->module->buildMod,
                                                       &fedMod);
        ExecUpdateImptblInExframes();
        if (FAILED(hr))
            return hr;
    }
    return S_OK;
}
```

### 4.14 `GenerateNativeCode`（原 LABEL_101）

```cpp
HRESULT EXMGR::GenerateNativeCode(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;
    FunctionRecord* rec = ctx.rec;

    // 仅 Native 模式才把 BIL 转 LoBil 并生成机器码
    if ((self->flags126 & 0x20) == 0)
        return S_OK;

    FEDSYM fedFn;
    FEDSYM::BeFuncdefn(&fedFn, self->module, ctx.hfdefn);

    rec = self->GetFnRecord(ctx.hfdefn);
    DWORD stmtHash = rec->stmtHash;

    EXMGR::HashSet(self, stmtHash);
    self->currentStmtHash = stmtHash;

    // BIL → LoBil
    TransformToLoBil(self->module, self->bilAlloc, &fedFn, &self->stmtList);

    // LoBil → 机器码
    HRESULT hr = nativeGenProcedure(&fedFn, self->curFuncState);
    if (FAILED(hr))
        return hr;

    // 复位
    self->lineStateA       = -1;
    self->lineState3       = -1;
    self->currentStmtHash  = -1;
    return S_OK;
}
```

### 4.15 `FinishSuccess`（原成功收尾）

```cpp
void EXMGR::FinishSuccess(FnGenCtx& ctx)
{
    EXMGR* self = ctx.self;

    // 释放表达式树
    BILALLOC::DoneBilTrees(self->bilAlloc);

    // 清字段 + 恢复状态
    self->ResetCleanupFields();

    // 递归计数递减
    --self->compileNestDepth;
}
```

### 4.16 `ErrorRecovery_Restore`（原 LABEL_61 + LABEL_119 + LABEL_124 + LABEL_125）

```cpp
HRESULT EXMGR::ErrorRecovery_Restore(FnGenCtx& ctx, HRESULT hr)
{
    EXMGR* self = ctx.self;

    // ---------- 原 LABEL_61：清栈/恢复上下文/复位池（幂等）----------
    EXMGR::CntlCleanStack(self);
    if (self->flags124 & 0x40)
        EXMGR::RestoreContext(self);
    if (self->field78 != -1)
        EXMGR::TpoolReset(self);

    // ---------- 原 LABEL_119：恢复保存状态 / 丢弃哈希 ----------
    FunctionRecord* rec = self->GetFnRecord(ctx.hfdefn);
    if (rec->flags & 0x20)
    {
        EXMGR::RestoreOldFunctionState(self, ctx.hfdefn);
        self->flags124 &= ~0x20;
    }
    else if (ctx.kind == BODY_MODULE_LEVEL && ctx.stmtHash != (DWORD)-1)
    {
        HashDestroy(self->fnTable, ctx.stmtHash, ctx.hfdefn);
        rec->stmtHash = -1;
        rec->flags   &= ~0x20;
        rec->flags   &= ~0x10;
    }

    // ---------- 原 LABEL_124：清行状态 ----------
    self->lineStateA       = -1;
    self->lineStateB       = -1;
    self->currentStmtHash  = -1;
    self->flag194          = 0;
    self->flag96           = 0;

    // ---------- 原 LABEL_125：最终清理 ----------
    if (ctx.pPrevPrtmi)
    {
        // 回写首行尺寸（用旧 RTMI 的 lineSize）
        INT lineTbl     = self->opcodeMgr->lineTable;
        INT oldLineSize = ctx.pPrevPrtmi->lineSize;
        rec = self->GetFnRecord(ctx.hfdefn);
        SetLineCodeSize(self, rec->firstLine, oldLineSize);
    }

    --self->compileNestDepth;
    BILALLOC::DoneBilTrees(self->bilAlloc);
    self->ResetCleanupFields();

    // 非"用户取消"时把 HRESULT 转成 VB 错误号
    if (hr != USER_CANCEL_CODE)                 // 0x800A9C64
    {
        UINT vbErr = EberrOfHresult(hr);
        hr = EXMGR::CreateError(self, vbErr);
    }

    // 严重错误态：把错误上下文记入全局
    if ((INT)self->flags124 < 0)
        *(DWORD*)(g_errContext + 9) = self->field53;

    // 关闭临时 excode 块
    self->lineStateB      = -1;
    self->currentStmtLine = 0xFFFF;
    EXTBL::EndFuncTemp(self->excode);

    return hr;
}
```

---

## 五、路径映射表（原 GOTO → 现结构化）

| 原 `goto LABEL_xx` | 现结构 |
|---|---|
| `LABEL_23` | `PrepareByReuseBlock` 里的 `if (needFreeOldBlock) {...} else {...}` |
| `LABEL_29` / `LABEL_31` | `PrepareByReuseBlock` 尾部 |
| `LABEL_38` / `LABEL_39` | `InitBodyLoopState` + `CompileBodyLines` 入口 |
| `LABEL_50` | `CompileBodyLines` 里 `if (lineSize)` 三目赋值 |
| `LABEL_61` | `ErrorRecovery_Restore` 头部 |
| `LABEL_66` | `EnsureFunctionEnded` |
| `LABEL_76` | `MarkBlockExists` |
| `LABEL_95` | `MarkBlockExists` 里 `if (rec->flags & 0x20)` |
| `LABEL_96` | `FinalizeRTMICommon` |
| `LABEL_101` | `GenerateNativeCode` |
| `LABEL_118` / `LABEL_119` | `ErrorRecovery_Restore` 中段 |
| `LABEL_124` / `LABEL_125` | `ErrorRecovery_Restore` 尾段 |
| `LABEL_132` | 每个子函数 `return` / 主函数 `return` |

---

## 六、结论

- **完全无 GOTO**：所有跳转通过 `if/else`、`switch`、`while` 条件化、子函数早返回，以及把 4 个清理块合并进 `ErrorRecovery_Restore` 消解；
- **阶段化更清晰**：准备（两种）→ 逐行 → 收尾（三种）→ Native → 成功/失败，每段一个子函数；
- **语义保持**：
  - `kind == 3` 直接跳 Native 收尾；
  - `kind == 0 有块` 走"模块体"分支（与种类 2 一致，原逻辑如此）；
  - EnC / 模块体 / 无块三条 RTMI 路径分别独立；
  - 错误恢复仍做"`CntlCleanStack` → `RestoreOldFunctionState`/`HashDestroy` → 回写首行尺寸 → 转 VB 错误号 → `EndFuncTemp`"；
  - 用户取消码 `0x800A9C64` 不再转错误号；
- **上下文收敛**：`FnGenCtx` 明确携带 `hfdefn/firstLine/endLine/kind/stmtHash/fEnded/pPrevPrtmi` 等跨阶段状态，避免原函数里到处 `*(this+xxx)` 的隐式依赖。

这套结构等同于原函数的分支覆盖：`kind ∈ {0,1,2,3}` × `hasBlock ∈ {0,1}` × `hasSavedState ∈ {0,1}` × `NativeMode ∈ {0,1}`，各路径均已映射。
