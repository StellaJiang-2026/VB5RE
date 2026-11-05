# `EXMGR::CompileLine` 分析总结

## 一、一句话定位

`EXMGR::CompileLine` 是 VB 编译器 **“逐行编译”的核心分派器**：它接收一行 `OP_LINE`（伪指令流），从游标逐个取出 10 位操作码，按操作码分派到表达式/语句/控制流/声明/条件编译等处理器，最终把结果落到 **BIL 求值栈** 和 **BIL 语句列表**，并返回 `HRESULT`（成功为 `S_OK` 或 `nullptr`，失败为错误码）。它是 `EXMGR::GenerateFunction` 主循环中每行调用一次的函数。

---

## 二、在编译管线中的位置

```
scanner/parser/genOpc
   └─ 生成 OP_LINE 伪指令流 → BASIC_TYPESRC / OPCODE_MGR
        └─ EXMGR::GenerateFunction（逐函数）
             └─ 主循环每行：
                  OPCODE_MGR::GetLine(...) → OP_LINE
                  EXMGR::CompileLine(...)  ← 本函数
                       ├─ CompileBinary / CompileUnary / CompileIntrin / CompileRef
                       ├─ ProcessFor / ProcessForEach / ProcessSelect / ProcessWith ...
                       ├─ CntlBind_If / Else / EndIf / DoWhileFor / EndLoop / Select / With
                       ├─ ProcessFuncDefn / ProcessEndExit / ProcessVarDefn ...
                       └─ TStPush / TStPop_Bil（BIL 求值栈）
             └─ 函数体结束后：TransformToLoBil → nativeGenProcedure
```

它处在 **前端（操作码行）与 BIL 树构造之间**，是 BIL 生成的主要入口。

---

## 三、输入 / 输出 / 契约

| 参数 | 类型 | 语义 |
|---|---|---|
| `this` | `EXMGR*` | 执行管理器；反编译写成 `BASIC_TYPESRC**` 是残留，函数体全部是 `EXMGR::` 调用 |
| `compileFlags` | `UINT` | 编译模式位标志（见下节） |
| `pLine` | `const OP_LINE*` | 行描述：`+0` 操作码字、`+1` 行属性、`+2` 指令流字节数（字）、`+4` 行上下文指针 |
| `pfFuncEnded` | `int*` | 输出：遇 `End Sub/End Function/Exit` 等置 `1`，通知 `GenerateFunction` 结束逐行循环 |

**返回**：
- 成功：`nullptr`（`CL_Success`）
- 失败：`HRESULT` 错误码（`CL_Error`），可能是 `0x800Axxxx` 或 `CreateError` 产生的 VB 错误号。

**契约**：本函数只负责“一行”，不负责函数级状态切换、excode 块准备/收尾、RTMI 发出——这些由调用者 `GenerateFunction` 负责。

---

## 四、行上下文与 `savedCtx` 机制

函数开头：

```c
qmemcpy(savedCtx, this, 0x18);      // 保存 this 前 24 字节
memset(this, 0, 0x18);              // 清空工作区
*(this + 6) = 453;                  // 默认语句类
if ((pLine[1] & 2) == 0) *(this+6) = 452;
```

- `this+0..23` 是 **行级临时工作区**（语句类型、引用种类、类型参数等），每行重置。
- `savedCtx` 保存上一行的值，退出时 `qmemcpy(this, savedCtx, 0x18)` 恢复——**这 24 字节是跨行状态**（例如 With 栈、语句上下文），不能丢。
- 所以本函数是“**借用**”这 24 字节做行内工作，而非独占。

其他关键字段：

| 偏移 | 含义 |
|---|---|
| `this+146` | 当前行剩余指令流字节数（每取 2 字节减 2，取完则注入 `opCode=260` 作为行尾哨兵） |
| `this+147` | 当前操作数相对行基址的偏移 |
| `this+71/72` | 操作码游标（`this+72` 是 `&opcodePtr`） |
| `this+22` | 当前操作码（低 10 位） |
| `this+124` | 内部状态（bit1 = Select 块标志） |
| `this+126` | 编译模式（bit4 = 条件编译、bit5 = Native） |
| `this+9` | `BASIC_TYPESRC*`（行改动检查） |
| `this+31` | 编译器主标志（bit1 = 预扫描/声明态） |
| `this+43/44` | `norls` 分配游标/页尾，用于 BIL 节点分配 |
| `this+54` | 当前 `Me` 的模块类型信息 |

---

## 五、操作码解码

```c
opcodeRaw = **ppCursor;             // 取 2 字节
opGroup   = opcodeRaw >> 10;        // 高 6 位：操作数组数/分组
opcode    = opcodeRaw & 0x3FF;      // 低 10 位：操作码本体
```

- **高 6 位 `opGroup`** 传给 `hr`（在很多分支里作为“操作数个数/分组”的隐式参数，例如 `case 0xB7/0xCE` 用高字当立即数）。
- **低 10 位 `opcode`** 决定 switch 分派。
- 行尾哨兵：`this+146 == 0` 时注入 `opCode=260`，大多数 case 会 `continue` 跳过它；`case 0x104` 直接 `CL_Success` 结束。

---

## 六、编译标志位语义

`compileFlags`（`flags`）多位含义（从分支使用推断）：

| 位 | 名称 | 含义 |
|---|---|---|
| `0x01` | `CF_EMIT` | 生成代码（需要构造 BIL 节点并 `TStPush`/`AppendStmtList`） |
| `0x02` | `CF_??` | 仅出现在 `flags == 2` 分支（`case 0xE3`），特殊模式 |
| `0x04` | `CF_CHECK` | 声明后检查/控制栈配对（`CntlIsEmpty_Bil`、`LabelFindUndefined`、`CntlBind_With` 等） |
| `0x08` | `CF_PRESCAN` | 声明/预扫描态（不做代码生成，只做声明登记、跳过操作数） |
| `0x10` | `CF_CCONLY` | 条件编译专用（`case 0xFE`） |
| `0x20` | `CF_WITH` | With 上下文（`case 0x71/0xF5`，可能把 `flags &= ~1` 抑制后续生成） |
| `0x09` | `CF_EXPR` | `CF_EMIT \| CF_PRESCAN`：需要表达式结果（很多 `(flags & 9)` 判断） |

典型用法：
- `(flags & 9) == 0` → 既不是 EMIT 也不是 EXPR，纯声明/跳过模式。
- `(flags & 8) != 0` → 声明态，很多 case 直接 `continue` 或只做 `Skip*`。
- `(flags & 1) != 0` → 需要生成语句/节点。
- `(flags & 4) != 0` → 需要做控制栈配对/标签检查。

---

## 七、操作码分派的功能簇

按 case 分组归纳：

### 7.1 表达式编译

| case | 处理器 | 说明 |
|---|---|---|
| `0x00–0x13` | `CompileBinary` | 二元运算 |
| `0x15–0x1C` | `CompileUnary` | 一元运算 |
| `0x14/0x45/0x56/.../0xF1` | `CompileIntrin` | 内建函数（Len/Mid$/Chr$ 等） |
| `0x84–0x89/0x8D/0x8E/0xC3/0xC4` | `CompileStringFuncs` | 字符串函数 |
| `0x44` | `CompileRef` + `ProcessArrayBase` | 数组引用基址 |
| `0xD2–0xDD/0xF6` | `CompilePrint` | Print 语句 |
| `0xBF/0xC1/0xE1/0xE2/0x100` | `CompileRedim` | ReDim |
| `0x94/0xEB/0xEC/0x102` | `NotifyWatchType` + 栈顶类型判断 | 调试监视 |
| `0x96` | `ProcessFuncDefn`/`ProcessFuncDefnConstExprs` | 函数定义 |
| `0x9D` | `ProcessTypeOf` | TypeOf |
| `0xE3` | 特殊：`flags==2` 设 `excode+16 \|= 0x10`；`flags==32` 直接成功 | 函数体结束标记 |

### 7.2 引用/标识符

| case | 说明 |
|---|---|
| `0x20–0x34/0x41/0x42` | 设置 `this+1`（引用种类）与 `this+2`（类型参数）后 `goto CL_CompileRef` → `CompileRef` |
| `0x35–0x40/0x43` | 同上但 `goto CL_TypeRef`（With 类型引用，先 `WithTop`/`ProcessWatchWith` 压栈） |
| `0x27/0x2E` | `CL_WatchType1/2`：直接设 `this+1` |
| `0x55/0xA5/0xDE/0xE8` | 引用种类 7 |
| `0x75/0x76/0x77` | 引用种类 3 + `*this \|= 1` |
| `0x5B/0xBD/0xBE` | `Me` 引用：构造节点，`SetSimpleVtype(22)`；`opCode==190` 额外设 bit3 |
| `0xD0` | `TStPush(nullptr)`（Nothing） |
| `0xD1` | 弹栈顶 + 构造 0x70 节点（参数命名） |
| `0xCF` | 弹栈顶 + 构造 0x71 节点 |

### 7.3 常量/字面量构造

| case | 节点 | 类型 |
|---|---|---|
| `0xAB` | `MakeCnsSpecialNode(0)` | Empty |
| `0xA9` | 节点 + `SetSimpleVtype(13)` | Date |
| `0xAA/0xB4` | `SetSimpleVtype(12/11)` | Integer/Byte |
| `0xAC/0xAE/0xB1` | 0x?? 节点 | 类型码由 `opGroup` 高位决定 |
| `0xAD/0xAF/0xB2` | `SetSimpleVtype(8)` + `GetLongOpnd` | Long |
| `0xB0` | `SetSimpleVtype(22)` + 0 值 | Object/Null |
| `0xB3` | `SetSimpleVtype(10)` + double | Double |
| `0xB5` | `SetSimpleVtype(6)` + `opGroup` 高字 | 指针/地址 |
| `0xB6` | `MakeLitStr` | 字符串字面量 |
| `0xB7` | `hr<2` 用 `SetSimpleItype(9)` + 高字；`hr==2/3` 用 `MakeCnsSpecialNode(1/2)` | Boolean/Nothing/Null |

### 7.4 控制流绑定

| case | 绑定函数 | 说明 |
|---|---|---|
| `0x5F` | `CntlBind_DoWhileFor` | Do 开始 |
| `0x61/0x66/0x9E` | `CntlBind_DoWhileFor` | Do While/Until |
| `0x62/0xF4` | `CntlBind_DoWhileFor` | 条件分支 |
| `0x63/0x64` | `CntlBind_Else` | Else/ElseIf |
| `0x65` | `CntlBind_Else(0x65)` | 条件 Else |
| `0x6A/0x6B` | `CntlBind_EndIf` | End If |
| `0x68` | `RestoreContext` | 上下文恢复 |
| `0x69/0x6D/0x6F` | `ProcessEndExit` + 配对检查 + `LabelFindUndefined` + `CheckBetweenFuncs` | End/Exit 函数体 |
| `0x71` | `WithTop` + `WithPop_Bil` + `CntlBind_EndWith` | With 结束 |
| `0x78` | `CntlBind_ExitDoFor` | Exit Do/For |
| `0x79` | `ProcessExitFor` | Exit For |
| `0x7A/0x7B/0x7C` | `ProcessEndExit` | End/Exit |
| `0x9B/0x9C` | `CntlBind_If` | If 条件 |
| `0xB9/0xF3` | `CntlBind_EndLoop` | EndLoop |
| `0xBA/0xBB` | `CntlBind_EndLoop` + 节点类型 14 | 条件 EndLoop |
| `0xEA` | `ProcessSelect` + `this+124 \|= 2` | Select Case 开始 |
| `0x53/0x54/0x6E` | `ProcessCaseElse/ProcessCaseDone/ProcessEndSelect` | Select 分支/结束 |
| `0x4B–0x52` | `ProcessCases` | Select Case 各分支 |
| `0xC9/0xE5` | `LabelRef` + `AppendStmtList` | GoTo/GoSub/Resume |
| `0x99/0x9A` | `LabelRef` + `GetGosubListSym` | GoSub 带列表 |
| `0xA8` | `LabelDef` | 标签定义 |
| `0xF5` | `ProcessWith` + `CntlBind_With` | With 开始 |
| `0xF2` | `ProcessVarDefn` | Dim/变量声明 |
| `0x9F` | 非法语句（`CF_PRESCAN` 下 `continue`，否则 `CL_ErrIllegal`） | |

### 7.5 声明 / 类型 / 参数

| case | 处理器 | 说明 |
|---|---|---|
| `0xCA/0xCB` | `CL_TypeDecl`：`PopCoerceConsume` + `LabelRef` 循环 + `AllocTmp` + `MakeSymTempVar` + `GetGosubListSym` | Type 声明（kind 23/24） |
| `0x96` | `ProcessFuncDefn` / `ProcessFuncDefnConstExprs` | Sub/Function 定义 |
| `0xF0` | `GetLongOpnd` + 设置 `this+126 \|= 0x10`、`this+26/27` | 特殊声明 |
| `0xF8` | `ProcessCCConst` | 条件编译常量 |
| `0xF9/0xFB` | `ProcessCCIf` | 条件编译 If |
| `0xFE` | 条件编译结束（`flags & 0x10` 时 `CL_Success`） | |
| `0x101` | `flags & 0x20` 时 `flags \|= 1` | |
| `0x103` | `flags & 8` 时清 `this+126` bit4、`this+27 = -1` | |

### 7.6 跳过 / 游标推进

| case | 说明 |
|---|---|
| `0x46` | `SkipFixedBytes(2)` + 可能 `AppendStmtExpr(nullptr)` |
| `0x47` | `CL_EndOfStmt`：复位语句状态，`FreetmpEmit` + `TpoolReset` |
| `0x5A` | `GetLongOpnd` + `SetContext` |
| `0x5C` | 跳 2 字节（`CL_SkipOperand`） |
| `0x70/0xCD` | `flags & 8` 时继续，否则 `CL_ErrIllegal` |
| `0xA3/0xA8` | 读操作数后 `CL_ReadOperand`（`LabelDef`） |
| `0xA6` | `SkipFixedBytes(变长)` |
| `0xE0` | `SkipFixedBytes(2)` + `SkipStr` |
| `0xE4` | `CL_SkipStr` |
| `0xF7` | `OPCODE_TRAVERSE::TraverseBuffer` 遍历到 `0x96(150)/0x104(260)` 为止，重置游标 |
| `0xFF` | `this+92 = this+147`（记录位置） |

### 7.7 标志/状态操作

| case | 说明 |
|---|---|
| `0x5D` | `flags & 8` 且 `hr & 0x1C` 非 0 → 错误 `0x9CD3`；否则设 `fInvalidCtx` 或 `tmpByte[0]` |
| `0x5E` | `flags & 8` 时 `fSeenStmt = 1`，否则错误 `0x9C75` |
| `0x58/0x59` | `ProcessOpCoerce` |
| `0x1D` | 常量传播：栈顶为 `0x160000` 常量 → `GetValueProp`，否则置 bit1 |
| `0xBC/0xE7` | `ProcessLsetRset` |

---

## 八、BIL 节点构造与求值栈

### 8.1 节点分配模式

```c
p = *(this + 43);            // norls 分配游标
*(this + 43) = p + N;
if (*(this + 44) < p + N)   // 页尾检查
    p = norls_allocator::nraAllocNewPage((this + 38), N);
```

- 节点大小常见 `20`（表达式节点）、`64`（语句列表节点）、`8`（标签链表条目）。
- 分配后手工填 `p[2]=0; p[3]=this+147-2;`（节点类型/位置），再 `SetSimpleVtype/SetSimpleItype`。

### 8.2 求值栈

- `TStPop_Bil(this)`：弹出栈顶节点。
- `TStPush(this, node)`：压入节点。
- 典型模式：
  ```c
  pWatchNode = TStPop_Bil(this);
  ... 构造新节点（引用旧节点）...
  TStPush(this, newP);
  ```
- `MakeCns*` 系列在 `case 0xAB/0xA9/0xAA/...` 中直接构造常量节点入栈。
- `CompileBinary/CompileUnary/CompileIntrin/CompileStringFuncs` 内部通常也从栈取操作数、压结果。

### 8.3 节点 tag（`*node & 0xFFFF0000`）

| tag | 含义 |
|---|---|
| `0x0F0000` | 需 `CompileWatchRet` 处理 |
| `0x120000` | 数组引用（需 `BiltypArrayGetElemType`） |
| `0x140000` | With 对象（需解引用 `node+4`） |
| `0x160000` | 常量（可 `GetValueProp` 传播） |
| 其他 | 类型码直接 `*node >> 16` |

---

## 九、控制流绑定与栈

`Cntl*` 家族（`CntlPush/Pop/Top/IsEmpty/SearchStack/CleanStack/GetErrorAndClean` + `CntlBind_*`）是**控制流 BIL 栈**：

- 遇到 `If` → `CntlBind_If` 压入控制项；
- `Else/ElseIf` → `CntlBind_Else`；
- `End If` → `CntlBind_EndIf` 弹出并闭合；
- `Do/Loop/For/Next` → `CntlBind_DoWhileFor/EndLoop/ExitDoFor`；
- `Select Case` → `ProcessSelect` + `ProcessCases/CaseElse/CaseDone/EndSelect`；
- `With` → `WithPush/WithTop/WithPop_Bil` + `CntlBind_With/EndWith`。

`flags & 4` 时在 `End/Exit` 分支做**配对检查**：`CntlIsEmpty_Bil` 非空则 `CntlGetErrorAndClean` 报错；`LabelFindUndefined` 查未定义标签报 `0x9CA4`；`CheckBetweenFuncs` 校验函数间行。

---

## 十、错误处理模型

两个统一出口：

```c
CL_Success:
    if ((flags & 0x30) == 0) {          // 非结束态先清理临时池
        FreetmpEmit(this, *(this+33));
        TpoolReset(this);
    }
    qmemcpy(this, savedCtx, 0x18);      // 恢复 24 字节跨行状态
    return nullptr;

CL_Error:
    TpoolReset(this);                   // 复位临时池
    qmemcpy(this, savedCtx, 0x18);      // 恢复 24 字节跨行状态
    return hr;                          // 返回错误码
```

- **`TpoolReset` 幂等**：成功/失败都会调用（成功分支看 `flags & 0x30`，失败无条件）。
- **`savedCtx` 必须恢复**：这 24 字节是跨行状态（With 栈、语句上下文），清空后必须还原。
- **`CL_ErrInvalidCtx`**：`fInvalidCtx` 为真时遇到执行语句 → `CreateError(0x9CB2)`。
- **`CL_ErrIllegal`**：非法语句 → `CreateError(0x9C74)`。
- 错误码既有 `0x9Cxx`（VB 编译错误），也有 `0x800Axxxx`（HRESULT），统一由调用者 `GenerateFunction` 的 `EberrOfHresult` 转换。

---

## 十一、增量编译相关

```c
if ((compileFlags & 0x1A) == 0
    && BASIC_TYPESRC::HasLineChanged(*(this+9), *(this+50)))
    EXMGR::FunctmpinfoRemove(this, *(this+50));
```

- 若当前行**已改动**（`HasLineChanged`），丢弃该函数已保存的临时编译状态（`FunctmpinfoRemove`）。
- 这支撑 **EnC / 增量编译**：只有改动的行才触发重编译，未改动的行复用已有 BIL/RTMI。
- 结合 `GenerateFunction` 里的 `HasLineChanged`/`ClearLineChangedRange`，构成行级增量编译链。

---

## 十二、关键设计要点

1. **操作码驱动的巨型 switch**：以 10 位操作码分派，覆盖表达式、语句、控制流、声明、条件编译、调试监视、跳转/标签。高 6 位 `opGroup` 在很多 case 里当立即数/分组使用。
2. **BIL 求值栈 + 语句列表双输出**：表达式走 `TStPush/Pop`，语句走 `AppendStmtList/AppendStmtExpr`；两者最终由 `GenerateFunction` 交给 `TransformToLoBil`。
3. **`savedCtx` 24 字节借用机制**：本函数把 `this+0..23` 当行内工作区，但必须保存/恢复跨行状态——这是最容易出错的地方。
4. **编译标志位控制生成路径**：`CF_EMIT(1)`、`CF_CHECK(4)`、`CF_PRESCAN(8)`、`CF_CCONLY(0x10)`、`CF_WITH(0x20)`、`CF_EXPR(9)`；不同分支用 `(flags & x)` 决定是否生成节点、是否跳过操作数、是否做配对检查。
5. **控制流栈 `Cntl*`**：If/Else/EndIf、Do/Loop、For/Next、Select/Case、With 的嵌套与配对，全部由 `CntlBind_*` 维护；`End/Exit` 分支做配对完整性检查。
6. **错误/成功统一出口**：`CL_Error`/`CL_Success` 都先 `TpoolReset` 再恢复 `savedCtx`，返回 `hr`/`nullptr`；中间无提前 return，保证状态一致。
7. **增量编译友好**：行改动时 `FunctmpinfoRemove` 丢弃函数级临时信息；`HasLineChanged` 避免重编译未改动行。
8. **反编译类型残留**：签名写成 `BASIC_TYPESRC** this` 是反编译器误判，函数体全部是 `EXMGR::` 调用，实际应为 `EXMGR*`；`savedCtx` 的 24 字节恰好是 `EXMGR` 前 6 个 dword（`+0..+23`）。

---

## 十三、一句话总结

> **`EXMGR::CompileLine` 是 VB 编译器的“单行 BIL 生成分派器”**：以 10 位操作码为键，把一行伪指令流分派到 `CompileBinary/Unary/Intrin/Ref/Print/Redim`、`ProcessFor/Select/With/FuncDefn/VarDefn/EndExit`、`CntlBind_If/Else/EndIf/DoWhileFor/EndLoop/With`、`LabelDef/LabelRef` 等处理器，结果落到 **BIL 求值栈** 与 **BIL 语句列表**；`compileFlags` 的 `EMIT/CHECK/PRESCAN/CCONLY/WITH` 位控制生成/检查/跳过路径；`savedCtx` 保存并恢复 24 字节跨行状态；成功/失败统一走 `CL_Success`/`CL_Error` 做 `TpoolReset` + 上下文恢复。它是 `GenerateFunction` 逐行循环中每次调用的执行者，也是 BIL 树构造的主要入口。
