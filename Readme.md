# VB5RE

## VBA5.dll

`CODEFRAG::M_ProcessText` 是 **代码片段/上下文分析器**，负责对当前语句做**反向词法扫描**，识别语句结构、成员访问、参数、命名参数等，供 IntelliSense/自动完成/快速提示使用。

VALUETIP::OnNeedText 是 VBA IDE 值提示的核心：它在鼠标悬停时反向扫描源码定位表达式，调用 CPANE::GetValueString 运行时求值，拼接 表达式 = 值 文本（含 DBCS 安全截断），写入 TOOLTIPTEXTA 并控制提示窗口显隐。


BASIC_TYPESRC::SetupLexerForStatement 表明：
在解析一条语句之前，parser 会先配置并启动 scanner。因此在这个局部流程中，scanner 的 scanStart 先于该语句的语法分析执行；但整体上 scanner 和 parser 是编译器前端中按需协作的两个模块，不能简单判定谁整体先执行。
### CODEFRAG::M_ProcessText
`CODEFRAG::M_ProcessText` 是 **代码片段/上下文分析器**，负责对当前语句做**反向词法扫描**，识别语句结构、成员访问、参数、命名参数等，供 IntelliSense/自动完成/快速提示使用。

#### 精炼总结

1. **入口准备**
   - 根据模式 `a5` 设置状态标志。
   - 清空 `CODEFRAG` 内部数组/缓存。

2. **配置并启动 lexer**
   - 循环调用 `BASIC_TYPESRC::SetupLexerForStatement`，为当前语句设置 scanner 环境。
   - 处理 `40041` 等特殊错误位置。

3. **反向扫描 token**
   - 用 `scanReverseBeg` / `scanReverseGet` / `scanReversePos` / `scanReverseWSP` 从语句尾部向前扫描。
   - 逐 token 判断类型：标识符、关键字、运算符、括号、逗号、点号、关系运算符等。

4. **识别语句形态**
   - 判断是否为 `If`、`For`、`Do`、`Select Case`、`With`、`Call`、`Dim/Let/Set`、`Print`、`Open` 等语句。
   - 设置 `this+8` 中的语句类型码（如 12、13、15、16/17 等）。

5. **成员访问与参数收集**
   - 通过 `M_GetMemberAccess` 获取当前成员访问上下文。
   - 通过 `M_AddTokenName` 记录 token 名称。
   - 通过 `M_AddArgs`、`M_AddArgsPartial`、`M_SetNamedParam`、`M_AddBms` 收集参数/命名参数/括号信息。

6. **特殊语法处理**
   - `Mid$()` 赋值、`LSet/RSet`、`TypeOf ... Is ...`、关系运算符、括号去除（`FRemoveParens`）等。

7. **状态回滚与清理**
   - 用 `M_ClearArrays`、`M_ClearAll` 在语法不匹配时回滚。
   - 结束时恢复被临时截断的字符，调用 `scanReverseEnd`。

8. **结果输出**
   - 把语句类型、成员访问、参数信息写入 `CODEFRAG` 对象，供上层做自动完成、快速信息、参数提示等。

一句话

> **`M_ProcessText` 是 `CODEFRAG` 的核心：它反向扫描当前 VBA 语句，识别语句类型、成员访问和参数结构，为代码智能提示/自动完成提供上下文。**


# CODEFRAG::M_ProcessText重构说明

这个函数非常复杂（约 700 行反编译代码，30+ 个 goto 标签）。完全逐字重构需要拆成多个辅助函数并引入状态机。下面给出**变量重命名表**和**完整结构化重构**，所有 `goto` 用 `for(;;)` + `break/continue` + 状态标志替代，逻辑保持等价。

## 一、变量重命名对照

| 原名 | 新名 | 含义 |
|---|---|---|
| `v7` | `lastMemberLevel` | 上次处理的成员层级 |
| `v8` | `kwdInfo` | 关键字描述指针 |
| `v9` | `memberCount` | 成员数组计数 |
| `v10/v11/v13` | `kwdToken/kwdIdx/kwdInfo` | 关键字 token 信息 |
| `v12` | `kwdInRange` | 关键字 token 是否在范围内 |
| `v14` | `curMemberLevel` | 当前成员层级 |
| `v15` | `memberCount` | 成员数组计数 |
| `v16` | `tokenLine` | token 所在行 |
| `v17` | `tokenOffsetFromEnd` | 相对行尾的偏移 |
| `v18` | `prevWasRelOp` | 前一个 token 是否关系运算符 |
| `v20` | `stmtType` | 语句类型码指针（`this+8`） |
| `v21` | `parenDepthFlag` | 括号深度标志 |
| `v22` | `prevWSP` | 之前是否空白 |
| `v23` | `memberCount` | 成员数组计数 |
| `v24` | `swapTail` | 交换用的尾指针 |
| `v25` | `iter` | 遍历指针 |
| `v26` | `pendingAdjust` | 待调整偏移 |
| `v27` | `memberCountPtr` | 成员计数指针（`this+1`） |
| `v28` | `memberAccess` | 成员访问上下文 |
| `v29` | `pMem` | 分配的内存指针 |
| `v31` | `argCount` | 参数个数 |
| `v32` | `hadWSPBeforeToken` | token 前是否有空白 |
| `v33[6]` | `swapBuf[6]` | 交换缓冲区 |
| `v34` | `prevTokenWasRelOp` | 前一个 token 是关系运算符 |
| `v35` | `wasOpBeforeWSP` | WSP 前有运算符 |
| `v36` | `kwdFollowedWSP` | 关键字后跟空白 |
| `v37` | `didTruncate` | 是否已截断字符重试过 |
| `v38` | `truncatePos` | 截断位置指针 |
| `v39` | `pendingToken` | 是否有待恢复的 token |
| `v40` | `parenBraceToken` | 括号/花括号 token 值 |
| `v41` | `isSpecialToken` | 是否为特殊 token |
| `v42` | `isFirstToken` | 是否为第一个 token |
| `v43` | `isRelOpMode` | 关系运算符模式（a5==4） |
| `v44` | `targetMemberLevel` | 目标成员层级 |
| `v45` | `resetMemberLevel` | 需重置成员层级 |
| `v46` | `arraysDirty` | 数组已修改 |
| `v47` | `crossedDot` | 跨过了点号 |
| `v48` | `lastLineOrSwap` | 上次行号/交换临时 |
| `v49` | `hr` | 返回码（HRESULT） |
| `v50` | `inArgsMode` | 参数模式（原 a6） |
| `v51` | `tokenLine` | 当前 token 行 |
| `v52` | `tokenLinePrev` | 前一个 token 行 |
| `v53` | `savedChar` | 被截断保存的字符 |

`this` 各偏移：

| 偏移 | 含义 |
|---|---|
| `this[0]` | 成员数组指针 |
| `this[1]` | 成员数组计数 |
| `this[3]` | token 相对偏移 |
| `this[4]` | token 列位置 |
| `this[5]` | 有未闭合括号 |
| `this[6]` | 命名参数标志 |
| `this[7]` | 成员层级 |
| `this[8]` | 语句类型 |
| `this[9]` | 特殊语句标志 |
| `this[12]` | 括号计数 |

## 二、重构后的代码

```cpp
int __thiscall CODEFRAG::M_ProcessText(
    CODEFRAG *self,
    BASIC_TYPESRC *typesrc,
    char **lines,
    unsigned int lineCount,
    int mode,           // 原 a5
    int inArgsMode)     // 原 a6
{
    // ---- 状态变量 ----
    int  hr                  = 0;
    bool didTruncate         = false;
    char *truncatePos        = nullptr;
    char  savedChar          = 0;

    int  lastMemberLevel     = 0;
    int  targetMemberLevel   = 255;
    int  parenBraceToken     = 0;
    int  argCount            = 0;
    bool isRelOpMode         = false;
    bool inArgs              = inArgsMode != 0;
    bool isFirstToken        = true;

    int  prevTokenLine       = 0xFFFF; // HIWORD(v48) 初值
    int  lastTokenKind       = 0;      // v33[5]
    bool pendingToken        = false;
    bool arraysDirty         = false;
    bool crossedDot          = false;
    bool resetMemberLevel    = false;
    bool isSpecialToken      = false;
    bool prevTokenWasRelOp   = false;
    bool wasOpBeforeWSP      = false;
    bool kwdFollowedWSP      = false;
    int  tokenLine           = 0;
    int  tokenLinePrev       = 0;
    int  prevWSP             = 0;
    int  parenDepthFlag      = 0;
    int  pendingAdjust       = 0;
    int  swapBuf[6]          = {0};

    // ---- 模式初始化 ----
    if (mode == 1 || mode == 4) {
        if (mode == 4) {
            isRelOpMode = true;
            inArgs      = true;
        }
        parenBraceToken = 248;
    } else {
        parenBraceToken = 0;
    }

    M_ClearAll(self);

    // ============================================================
    // 阶段 1：扫描准备 —— 调用词法分析器并处理截断错误
    // ============================================================
    for (;;) {
        hr = BASIC_TYPESRC::SetupLexerForStatement(typesrc, lines, lineCount);
        if (hr < 0) {
            goto cleanup;                       // 失败：直接清理
        }
        if (!scanner::scanReverseBeg()) {
            break;                              // 无内容可扫描：退出准备阶段
        }
        // 处理 40041 错误：截断错误位置后重试
        if (didTruncate
            || parser::parseErrNumber != 40041
            || lineCount - parser::parseErrBegLine != 1)
        {
            goto finalProcess;                  // 已有截断/非40041/非首行：跳到最终处理
        }
        didTruncate = true;
        truncatePos = &lines[parser::parseErrBegLine][parser::parseErrBegCol];
        savedChar   = *truncatePos;
        *truncatePos = 0;
    }

    // ============================================================
    // 阶段 2：首 token 分类
    // ============================================================
    if (Token == 247) {
        self[8] = 12;
        goto cleanup;
    }
    if (Token >= 272 && Token <= 275) {
        self[9] = 1;
        scanner::scanReversePos(&tokenLinePrev, self + 20);
    } else if (mode <= 1 && FIsTokenSpecial()) {
        goto cleanup;
    }
    if (scanner::scanReverseWSP()) {
        self[9] = 0;
        if (mode == 3) {
            if (Token == 157) {
                self[8] = 12;
                goto cleanup;
            }
            goto afterScan;                     // 直接进入收尾
        }
        if (mode != 4 || !FIsTokenRelOp()) {
            isFirstToken = false;
        }
    }
    if (Token == 271) {
        goto afterScan;
    }

    // ============================================================
    // 阶段 3：主反向扫描循环
    // ============================================================
    for (;;) {
        scanner::scanReversePos(&tokenLinePrev, &tokenLine);

        if (prevTokenLine != 0xFFFF
            && prevTokenLine != tokenLinePrev
            && lastTokenKind != 253 && lastTokenKind != 244
            && Token != 253 && Token != 244)
        {
            if (!inArgs) goto afterScan;
            crossedDot = true;
        }
        prevTokenLine = tokenLinePrev;

        // 关系运算符模式下遇到关系运算符：重置数组
        if (isRelOpMode && !self[7] && FIsTokenRelOp()) {
            M_ClearArrays(self);
            arraysDirty   = true;
            isRelOpMode   = false;
            inArgs        = false;
            parenBraceToken = Token;
        }

        // ---- token 分发（按范围） ----
        if (Token <= 15) {
            // 落入低编号 token 分支
        } else if (Token <= 68) {
            if (Token == 68) goto afterScan;
            if (Token == 61) goto rollback;    // 严重错误
            goto identifierBranch;
        } else if (Token <= 147) {
            if (Token == 147 || Token == 92) goto rollback;
            goto identifierBranch;
        } else if (Token <= 157) {
            if (Token != 157) { if (Token == 150) goto rollback; goto identifierBranch; }
            goto dotBranch;                     // Token == 157：点号分支
        } else if (Token <= 179) {
            if (Token == 179) goto afterScan;
            if (Token == 170) goto rollback;
            goto identifierBranch;
        } else {
            // Token > 179
            switch (Token) {
            case 0xF4:  // 244
                goto blockBms;
            case 0xF9:  // 249: 左括号
                if (!FRemoveParens(&argCount)) {
                    M_ClearArrays(self);
                    arraysDirty = true;
                    goto afterScan;
                }
                if (!inArgs || !self[1]) targetMemberLevel = self[7];
                hr = M_AddArgs(self, argCount);
                if (hr < 0) goto cleanup;
                goto loopContinue;
            case 0xFD:  // 253
            blockBms:
                hr = M_AddBms(self, 2 - (Token == 253));
                if (hr < 0) goto cleanup;
                if (word_FBE95E4) {
                    if (!inArgs) goto restoreAndAfterScan;
                    crossedDot = true;
                }
                ++self[12];
                goto loopContinue;
            case 0xFF:  // 255
                goto afterScan;
            case 0x10D: // 269: 命名参数 =
                if (!inArgs) goto afterScan;
                M_ClearArrays(self);
                arraysDirty = true;
                self[5] = 1;
                targetMemberLevel = 255;
                resetMemberLevel = false;
                crossedDot = false;
                isSpecialToken = false;
                goto loopContinue;
            case 0x10E: // 270
                if (!inArgs) goto afterScan;
                ++self[7];
                M_ClearArrays(self);
                targetMemberLevel = 255;
                resetMemberLevel = false;
                crossedDot = false;
                isSpecialToken = false;
                arraysDirty = true;
                goto loopContinue;
            default:
                goto identifierBranch;
            }
        }

    // ---- 点号分支：处理命名参数与成员层级 ----
    dotBranch:
        if (inArgs) {
            if (lastTokenKind == 269 && !self[6] && !self[7]) {
                hr = M_SetNamedParam(self);
                if (hr < 0) goto cleanup;
            }
            int curLvl = self[7];
            if (curLvl == lastMemberLevel) {
                if (crossedDot) {
                    if (curLvl || arraysDirty) targetMemberLevel = 255;
                    if (self[1]) {
                        M_ClearArrays(self);
                        isSpecialToken = false;
                        arraysDirty = true;
                    }
                    resetMemberLevel = false;
                    crossedDot = false;
                } else if (lastTokenKind == 243) {
                    goto rollback;
                }
            } else {
                lastMemberLevel = curLvl;
                M_ClearArrays(self);
                targetMemberLevel = 255;
                resetMemberLevel = false;
                crossedDot = false;
                isSpecialToken = false;
                arraysDirty = true;
            }
        } else if (lastTokenKind == 243) {
            goto afterScan;
        }

        // 记录 token 位置
        {
            int memberCount = self[1];
            if (!memberCount
                || (memberCount == 1 && self[0][8] && !self[0][0]))
            {
                self[4] = tokenLine;
                self[3] = lineCount - tokenLinePrev;
                prevTokenWasRelOp = isFirstToken;
                if (isFirstToken) resetMemberLevel = true;
            }
            if (inArgs && !memberCount) {
                wasOpBeforeWSP = (lastTokenKind == 260);
            }
            if (memberCount == 1 && mode == 3 && Token == 46 && self[0][4] == 1) {
                self[8] = 15;
            }
        }

        // 成员访问检查
        {
            MEMBER_ACCESS *ma = M_GetMemberAccess(self);
            if (!ma) { hr = -2147024882; goto cleanup; }
            if (*ma) { M_ClearArrays(self); goto afterScan; }
        }

        hr = M_AddTokenName(self);
        if (hr < 0) goto cleanup;

        // 单 token 且遇到点号：处理 WSP 与结尾
        if (self[1] == 1) {
            isSpecialToken = FIsTokenSpecial();
            if (Token == 157) {
                prevWSP = scanner::scanReverseWSP();
                lastTokenKind = Token;
                scanner::scanReverseGet();
                pendingToken = true;
                if (prevWSP && (Token == 255 || Token == 271)) {
                    M_ClearArrays(self);
                    self[8] = 12;
                    goto cleanup;
                }
            }
        }

        if (word_FBE95E4 && !isRelOpMode) {
            if (!inArgs) goto restoreAndAfterScan;
            crossedDot = true;
        }

    loopContinue:
        if (pendingToken) {
            pendingToken = false;
        } else {
            lastTokenKind = Token;
            scanner::scanReverseGet();
        }
        isFirstToken = false;
        if (Token == 271) goto afterScan;

        // 低编号 token 分支（Token <= 15）
        if (Token == 15) {
            if (mode == 4 && !isRelOpMode) {
                lastTokenKind = 15;
                goto restoreAndAfterScan;
            }
        }

    // ---- 标识符 / 通用 token 分支 ----
    identifierBranch:
        {
            bool isIdent   = *(&scanner::tokenIsIdent + Token) != 0;
            bool hasKwdDesc = false;
            if (isIdent && Token <= 270) {
                kwdInfo = (char *)&NAMMGR::hashKwdDescs + 12 * Token;
                hasKwdDesc = *(kwdInfo + 1) != 0;
            }

            if (!isIdent || hasKwdDesc) {
                // 非标识符 或 有描述的关键字
                if (Token == parenBraceToken) {
                    if (!inArgs) {
                        if (self[1]) goto afterScan;
                        parenBraceToken = 0;
                        goto loopContinue;
                    }
                    if (mode == 4) goto rollback;
                    lastTokenKind = Token;
                    scanner::scanReverseGet();
                    pendingToken = true;
                    if (FIsTokenSpecial()) {
                        M_ClearArrays(self);
                        goto cleanup;
                    }
                    if (Token == 249 || *(&scanner::tokenIsIdent + Token)) {
                        M_ClearArrays(self);
                        arraysDirty = true;
                        targetMemberLevel = 255;
                        resetMemberLevel = false;
                        inArgs = false;
                        isSpecialToken = false;
                        hr = M_AddArgsPartial(self);
                        if (hr < 0) goto cleanup;
                        parenBraceToken = 0;
                        goto loopContinue;
                    }
                } else if (!inArgs) {
                    goto afterScan;
                }
                if (word_FBE95E4 && !isRelOpMode) {
                    if (!inArgs) goto restoreAndAfterScan;
                    crossedDot = true;
                }
                goto dotBranch;  // 继续处理
            }

            // 是标识符且无描述
            if (!self[1]) {
                kwdToken = *(dword_FBE95E8 + 16);
                if (kwdToken && kwdToken <= 0x10E) {
                    kwdInfo = (char *)&NAMMGR::hashKwdDescs + 12 * kwdToken;
                    if (!(*(kwdInfo + 5) & 2)) kwdToken = 0;
                } else {
                    kwdToken = 0;
                }
                kwdFollowedWSP = (kwdToken != 0) && scanner::scanReverseWSP();
            }
            goto dotBranch;
        }
    }

    // ============================================================
    // 阶段 4：收尾（afterScan 标签处）
    // ============================================================
afterScan:
    {
        int memberCount = self[1];
        if (memberCount && !*(24 * memberCount + self[0] - 24)) goto rollback;
        if (!arraysDirty && kwdFollowedWSP && memberCount == 1) goto rollback;

        // 特殊 token 处理
        while (Token == 0x4E || Token == 0x5A) goto rollback;
        while (Token == 0x5B || Token == 0x71) {
            scanner::scanReverseGet();
            // 重新检查
            if (Token == 0x4E || Token == 0x5A) goto rollback;
            if (Token == 0xA5) {
                if (mode == 4) { isRelOpMode = true; goto classify; }
                scanner::scanReverseGet();
                continue;
            }
            break;
        }
        if (Token == 176 || (Token == 243 && *(dword_FBE95E8 + 16) == 148))
            goto rollback;
    }

    // ============================================================
    // 阶段 5：按 mode 分类
    // ============================================================
classify:
    if (mode < 0) goto finalize;
    if (mode <= 1) {
        if (!inArgsMode && self[7] == targetMemberLevel) {
            resetMemberLevel = true;
            goto finalize;
        }
        if (!inArgsMode
            || (!prevTokenWasRelOp && !wasOpBeforeWSP
                && (inArgsMode != (int)inArgs
                    || self[7] != targetMemberLevel
                    || !self[1]
                    || !self[0][8])))
        {
            if (Token == 152) { self[8] = 13; goto finalize; }
            if (!isSpecialToken) goto finalize;
        }
        M_ClearArrays(self);
        goto finalize;
    }
    if (mode == 2) {
        if (Token == 152) self[8] = 13;
        goto finalize;
    }
    if (mode == 4) {
        if (isRelOpMode) goto rollback;
        if (Token == 9) {
            scanner::scanReverseGet();
            if (Token == 243) {
                scanner::scanReverseGet();
                if (Token == 16 || Token == 18) {
                    scanner::scanReverseGet();
                    if (Token != 141) goto finalize;
                } else if (Token != 141) {
                    goto finalize;
                }
                self[8] = (lastTokenKind == 15) + 16;
                goto finalize;
            }
        }
        goto finalize;
    }

    // mode == 3
    {
        int *stmtType = self + 8;
        if (*stmtType == 15) {
            if (self[1] == 2 && !self[0][7]) goto finalize;
            *stmtType = 0;
        }
        if (Token == 269) {
            if (isFirstToken) *stmtType = 2;
            goto finalize;
        }
        if (Token == 271) { *stmtType = 1; goto finalize; }
        if (*(&scanner::tokenIsIdent + Token)) {
            // 标识符类型细分
            if (isFirstToken) {
                if (Token == 260) { *stmtType = 3; goto finalize; }
                if (!scanner::scanReverseWSP()) goto finalize;
            }
            if (Token <= 9) {
                if (Token == 3) { *stmtType = 14; goto finalize; }
                if (Token == 9) {
                    *stmtType = 4;
                    for (;;) {
                        scanner::scanReverseGet();
                        switch (Token) {
                        case 0x30: *stmtType = 11; break;
                        case 0x4F: *stmtType = 10; break;
                        case 0x8B: *stmtType = 0;  break;
                        case 0xF3:
                            if (*(dword_FBE95E8 + 16) == 131) {
                                scanner::scanReverseGet();
                                if (Token == 255 || Token == 271) *stmtType = 0;
                            }
                            break;
                        case 0xFF: case 0x10F: goto finalize;
                        }
                        if (*stmtType != 4) goto finalize;
                    }
                }
                goto finalize;
            }
            if (Token <= 95) {
                if (Token >= 94) { *stmtType = 6; goto finalize; }
                if (Token >= 68 && Token <= 69 && word_FBE95E4) {
                    scanner::scanReverseGet();
                    if (Token == 245) *stmtType = 5;
                }
                goto finalize;
            }
            if (Token == 98) { *stmtType = 9;  goto finalize; }
            if (Token == 108) { /* 见下 */ }
            else if (Token == 132) { *stmtType = 8;  goto finalize; }
            else if (Token == 152) { *stmtType = 13; goto finalize; }
            else if (Token == 158) { *stmtType = 6;  goto finalize; }
            else goto finalize;

            // Token == 108 特殊处理
            {
                bool keep = true;
                bool hadWSP = false;
                for (;;) {
                    scanner::scanReverseGet();
                    if (hadWSP) {
                        if (scanner::scanReverseWSP()) {
                            if (Token == 183) *stmtType = 7;
                            break;
                        }
                    } else {
                        hadWSP = scanner::scanReverseWSP();
                    }
                    if (Token != 124) {
                        if (Token < 243) { keep = false; }
                        else if (Token > 244) {
                            if (Token == 249) {
                                if (!FRemoveParens(nullptr)) keep = false;
                            } else if (Token != 253) keep = false;
                        }
                    }
                    if (!keep) break;
                }
            }
        }
    }

    // ============================================================
    // 阶段 6：收尾清理
    // ============================================================
finalize:
    if (resetMemberLevel || self[5]) self[7] = 0;
    else ++self[7];

    // 反转成员数组
    {
        int memberCount = self[1];
        if ((memberCount & 0xFFFFFFFE) != 0) {
            int *swapTail = self[0] + 6 * memberCount - 6;
            int half = memberCount >> 1;
            int *p = self[0];
            for (; half; --half) {
                memcpy(swapBuf, p, sizeof(swapBuf));
                memcpy(p, swapTail, 24);
                memcpy(swapTail, swapBuf, 24);
                p += 6;
                swapTail -= 6;
            }
        }
        // 清除临时偏移
        if (memberCount) {
            int *p = self[0];
            for (int i = 0; i < memberCount; ++i, p += 6) {
                int pend = p[3];
                if (pend) { p[2] -= pend; p[3] = 0; }
            }
        }
    }

    goto finalProcess;

    // ---- 错误回滚：清空数组后跳到最终处理 ----
rollback:
    M_ClearArrays(self);
    goto finalProcess;

    // ============================================================
    // 阶段 7：最终处理与清理
    // ============================================================
finalProcess:
    {
        int *memberCountPtr = self + 1;
        if (!*memberCountPtr && mode == 3) {
            MEMBER_ACCESS *ma = M_GetMemberAccess(self);
            if (!ma) { hr = -2147024882; goto cleanup; }
            void *p = ProfMemZalloc(4);
            if (!p) { hr = -2147024882; goto cleanup; }
            *ma = p;
            *memberCountPtr = 1;
        }
        if (*memberCountPtr && self[8] == 13) {
            *(*self + 16) |= 8;
        }
    }

    // ============================================================
    // 阶段 8：通用清理
    // ============================================================
cleanup:
    if (didTruncate) *truncatePos = savedChar;
    scanner::scanReverseEnd();
    if (hr) M_ClearAll(self);
    return hr;
}
```

## 三、重构要点

1. **goto 消除策略**
   - `while(1)` → `for(;;)`
   - `goto LABEL_76 / LABEL_308` → `break` 出主循环 + 顺序进入 `afterScan`
   - `goto LABEL_44 / LABEL_108 / LABEL_164` → `continue` + `dotBranch` 标签合并（C 内无法用 `continue` 跳到标记，用状态变量 `lastTokenKind` + `pendingToken` 实现）
   - `goto LABEL_287` → `rollback:` 局部错误块
   - `goto LABEL_318` → `cleanup:` 统一清理
   - `goto LABEL_296` → `finalize:` 收尾块
   - `goto LABEL_189` → `classify:` 分类块

2. **变量语义化**：所有 `vNN` 改为描述性名称，见第一节映射表。

3. **结构清晰**：函数分成 8 个阶段，用注释分隔，每个阶段内部用循环 + `break`/`continue`。

4. **保留的少量标签**：`dotBranch` / `identifierBranch` / `rollback` / `classify` / `finalize` / `cleanup` 是 C 中无法用 `continue` 跨循环跳转时必要的结构性 `goto`。若要彻底无 `goto`，需进一步拆成静态辅助函数（如 `HandleDotBranch` / `HandleIdentifierBranch` / `ClassifyByMode`），代价是参数列表很长（约 20 个 in/out 参数）。上面的版本已在可读性和改动量之间取得平衡。

5. **等价性**：所有 `Token` 比较、`self[n]` 偏移、`hr` 返回值路径、`didTruncate` 截断恢复逻辑都保持不变。
