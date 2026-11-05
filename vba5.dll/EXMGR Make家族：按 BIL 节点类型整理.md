# EXMGR `Make*` 家族：按 BIL 节点类型整理

`Make*` 家族是 EXMGR 的 **BIL 节点构造器**。它们都返回/输出 `stmtExpr*`（BIL 树节点），由 `CompileLine`、`Compile*`、`Process*` 组合成完整 BIL 树。下面按 BIL 节点语义分类整理。

---

## 一、常量 / 字面量节点（Const & Literal）

构造 BIL 常量/字面量叶子节点，供表达式/赋值/调用使用。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeCnsI2Node` | `ushort` | I2 常量节点 | 16 位整型常量 |
| `MakeCnsI4Node` | `ulong` | I4 常量节点 | 32 位整型常量 |
| `MakeCnsSpecialNode` | `CNSVAR_SPECIAL` | 特殊常量节点 | Empty/Null/Nothing/Missing 等特殊值 |
| `MakeLitHard` | `uint, void*, int` | 硬字面量节点 | 原始字节/立即数硬编码字面量 |
| `MakeLitStr` | `char*, ulong` | 字符串字面量节点 | ANSI/普通字符串常量 |
| `MakeLitStrOfHlnam` | `uint` | 字符串字面量节点 | 由名称表索引（Hlnam）构造字符串 |
| `MakeLitVar` | `tagVARIANT*, int` | Variant 字面量节点 | VARIANT 常量 |
| `MakeLitDefault` | `TYPE_DATA* const, ulong, uint* const, stmtExpr**` | 默认值字面量节点 | 按类型生成默认值（0/Empty/"" 等） |
| `MakeOpstrInChunkFromBstr` | `ushort*, ulong* const, int, int` | 字面量/opstr 节点 | 从 BSTR 块构造 opstr 字面量 |
| `ExtractLiteral`（非 Make 但相关） | `stmtExpr*, tagVARIANT*` | 提取值 | 从字面量节点提取 VARIANT |

---

## 二、符号 / 变量 / 引用节点（Symbol & Ref）

构造对符号、变量、局部/成员、临时变量的 BIL 引用节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeSymNode` | `FEDSYM*` | 符号节点 | 由 FEDSYM 构造符号引用 |
| `MakeSymNodeHvdefnPtdata` | `ulong` | 符号节点 | 由变量定义索引构造符号引用 |
| `MakeSymTempVar` | `ulong` | 临时变量符号节点 | 符号化临时变量引用 |
| `MakeVarIndexNode` | `stmtExpr*, uint` | 变量索引节点 | 带索引的变量/数组元素引用 |
| `MakeMeQualifier` | — | 限定符节点 | 构造 `Me` 限定符 |
| `MakePrdtTree` | `FEDTYP*` | 属性/类型描述节点 | 构造属性/类型描述树（Prdt） |
| `MakeImportOfBytes` | `void*, ulong, ulong` | 导入字节节点 | 构造原始字节导入节点 |
| `MakeImportOfParent` | `stmtExpr*, int` | 父导入节点 | 构造父对象/导入引用 |

---

## 三、临时变量 / 地址存储节点（Temp & Address）

构造临时变量存储、地址、长整型临时等 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeStoreAdrTemp` | `stmtExpr*` | 地址临时存储节点 | 把地址存入临时变量 |
| `MakeFStLongTemp` | `stmtExpr**, uint` | 长整型临时存储节点 | 长整型（FSt）临时变量存储 |
| `MakeHardType` | `stmtExpr*` | 硬类型节点 | 给表达式强加硬类型 |
| `MakeClearOutNode` | `stmtExpr*` | ClearOut 节点 | 构造清理/输出参数清理节点 |
| `MakeNewIfNullNode` | `BASIC_TYPEROOT*, stmtExpr*` | NewIfNull 节点 | 为空时执行 New 的条件节点 |

---

## 四、运算符 / 表达式节点（Operator & Expr）

构造一元/二元/复合表达式 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeUnaryOpNode` | `stmtExprOps, stmtExpr*, var_types` | 一元运算节点 | 取负/Not 等一元运算 |
| `MakeSeq1Node` | `stmtExpr*, stmtExpr*, ulong` | 顺序/序列节点 | 顺序执行两个表达式（逗号/序列） |
| `FoldUnaryExpr`（非 Make 但相关） | `stmtExprOps, uint, stmtExpr*` | 折叠节点 | 常量折叠一元表达式 |
| `FoldBinaryExpr`（非 Make 但相关） | `stmtExprOps, uint, stmtExpr*, stmtExpr*` | 折叠节点 | 常量折叠二元表达式 |

---

## 五、函数调用 / 内建调用节点（Call & Intrinsic）

构造函数/过程/内建调用 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeOneArgCall` | `stmtExprOps, stmtExpr*` | 单参调用节点 | 一参数调用 |
| `MakeTwoArgCall` | `stmtExprOps, stmtExpr*, stmtExpr*, uint` | 双参调用节点 | 两参数调用 |
| `MakeThreeArgCall` | `stmtExprOps, stmtExpr*, stmtExpr*, stmtExpr*, uint` | 三参调用节点 | 三参数调用 |
| `MakeHelperCall2` | `stmtExprOps, ulong, stmtExpr*, int` | Helper 调用节点 | 调用运行时辅助函数 |
| `MakeIntrinsicModeArg` | — | 内建模式参数节点 | 构造内建函数模式参数 |
| `MakeVectorRet` | `FEDTYP*, CCINFO const*, stmtExpr*` | 向量返回节点 | 向量/数组返回值构造 |
| `MakeCRec2UniFuncRet` | `stmtExpr*` | 记录到 Unicode 返回节点 | 记录类型返回值转换为 Unicode 返回 |
| `MakeFreeListArg` | `stmtExpr*, FEDSYM*, var_types` | 释放列表参数节点 | 构造释放列表参数 |

---

## 六、参数传递 / 调用约定节点（Arg & Param）

构造参数传递、ByRef/ByVal、隐藏参数、命名参数等 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeByrefArrayParam` | `stmtExpr*` | ByRef 数组参数节点 | ByRef 数组参数传递 |
| `MakeByrefCopyInOut` | `stmtExpr**, uint, var_types, var_types` | ByRef CopyInOut 节点 | ByRef 参数的 CopyIn/CopyOut 包装 |
| `MakeCDargRefAry` | `stmtExpr**` | CDarg 引用数组节点 | 构造 CDarg 引用数组参数 |
| `MakeHiddenParam` | `CCINFO const*, stmtExpr**, int, int` | 隐藏参数节点 | 构造隐藏参数（This/RetVal 等） |
| `InsertHiddenParam`（非 Make 但相关） | `stmtExpr*, stmtExpr*, CCINFO const*` | 插入隐藏参数 | 在参数列表中插入隐藏参数 |
| `InsertNamedArg`（非 Make 但相关） | `stmtExpr*, stmtExpr**, uint` | 插入命名参数 | 插入命名实参 |

---

## 七、类型转换 / 强制转换节点（Coerce & Cast）

构造类型转换、强制转换、Variant 包装等 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeFloatCoerce` | `uint, stmtExpr**` | 浮点转换节点 | 浮点强制转换 |
| `MakeCastIn` | `OBJECT_SUBTYPE, FEDTYP*, FEDSYM**, stmtExpr**` | CastIn 节点 | 对象/接口进入转换 |
| `MakeCastOut` | `OBJECT_SUBTYPE, FEDTYP*, FEDSYM*, FEDTYP*, stmtExpr**` | CastOut 节点 | 对象/接口退出转换 |
| `MakeCVar` | `stmtExpr**` | CVar 节点 | 构造 Variant（拥有者） |
| `MakeCVarNonOwner` | `stmtExpr**` | CVar 非拥有节点 | 构造非拥有 Variant |
| `Coerce*`（非 Make 但相关） | 多种 | Coerce 节点 | 通用/数组/记录/Variant 转换 |

---

## 八、赋值 / 记录 / 数组节点（Assign & Record & Array）

构造赋值、记录复制、数组锁、Redim、Get/Put 等 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeArrayAssign` | `stmtExpr*, stmtExpr*` | 数组赋值节点 | 数组元素赋值 |
| `MakeRecordAssign` | `stmtExpr*, stmtExpr*, int` | 记录赋值节点 | 记录整体赋值 |
| `MakeRecAssignTree` | `stmtExpr*, stmtExpr*, int` | 记录赋值树 | 记录赋值树（含字段级） |
| `MakeCopyRecordLetRHS` | `TYPE_DATA*, ulong, stmtExpr**` | 记录 Let RHS 复制节点 | 记录 Let 右侧复制 |
| `MakeDestructRecord`（重载 1） | `stmtExpr*, int` | 记录析构节点 | 记录析构 |
| `MakeDestructRecord`（重载 2） | `ulong, uint*, stmtExpr**` | 记录析构节点 | 按类型析构记录 |
| `MakeAryLock` | `stmtExpr*, int` | 数组锁节点 | 数组锁定/解锁 |
| `MakeRedimTree` | `uint, uint, uint, stmtExpr*, stmtExpr*, stmtExpr*` | Redim 节点 | 动态数组 Redim 树 |
| `MakeRedimParamArray` | `uint, FEDSYM**` | 参数数组 Redim 节点 | 参数数组 Redim |
| `MakeGetPutAryDescr` | `stmtExpr**, stmtExpr**, stmtExprOps` | 数组 Get/Put 描述节点 | 数组 Get/Put 描述符 |
| `MakeGetPutDescr` | `OWNERDESC*, TEMP_BUFFER*, ITypeInfo*, uint, uint` | Get/Put 描述节点 | 通用 Get/Put 描述符 |
| `MakeFxRecAryDesc` | `stmtExpr*, stmtExpr*` | 记录数组描述节点 | 记录数组描述 |
| `MakeFxstrInput` | `stmtExpr**, stmtExpr**` | 字符串输入节点 | 固定字符串输入 |
| `MakeStFixedStr` | `stmtExpr*, stmtExpr*, int` | 固定字符串存储节点 | 固定长度字符串赋值 |

---

## 九、控制流 / 条件节点（Control Flow）

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeNewIfNullNode` | `BASIC_TYPEROOT*, stmtExpr*` | NewIfNull 条件节点 | 空值判断+New |
| `DecideCondBranch`（非 Make 但相关） | `int, stmtExpr**` | 条件分支节点 | 生成条件分支 |
| `CntlBind_*`（非 Make 但相关） | 多种 | 控制流 BIL | If/For/Do/Select/With 绑定 |

---

## 十、临时释放 / 资源描述节点（Freetmp & ResDesc）

构造临时变量释放、资源描述、临时池相关 BIL 节点。

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeFreetmps` | — | Freetmp 节点 | 生成当前所有临时释放节点 |
| `MakeFreetmpList` | 多个回调 + 上下文 + `var_types` | Freetmp 列表节点 | 按类型生成临时释放列表 |
| `MakeFreeListArg` | `stmtExpr*, FEDSYM*, var_types` | 释放列表参数节点 | 释放列表作为参数 |
| `FreetmpAlloc`/`FreetmpAdd`/`FreetmpEmit`/`FreetmpEmitCase`/`FreetmpEmitHelper`（非 Make 但相关） | 多种 | Freetmp BIL | 临时释放分配/发射 |
| `TpoolAddResDesc`/`TpoolAddResDescAryLock`/`TpoolAddResDescProcTmps`/`TpoolAddResDescStmtTmps`（非 Make 但相关） | `TMPPOOL, ...` | 资源描述 | 临时池资源描述登记 |

---

## 十一、字面量/符号之外的辅助节点

| 函数 | 签名关键参数 | 返回/输出节点 | 用途 |
|---|---|---|---|
| `MakeHardType` | `stmtExpr*` | 硬类型节点 | 给节点标注硬类型 |
| `MakeMeQualifier` | — | Me 限定符节点 | `Me` 引用 |
| `MakePrdtTree` | `FEDTYP*` | Prdt 节点 | 属性/类型描述树 |
| `MakeClearOutNode` | `stmtExpr*` | ClearOut 节点 | 输出参数清理 |
| `MakeIntrinsicModeArg` | — | 内建模式参数 | 内建函数模式参数 |

---

## 十二、汇总结构表（按节点类别）

| 节点类别 | 代表函数 | 返回节点语义 |
|---|---|---|
| 常量/字面量 | `MakeCnsI2Node`、`MakeCnsI4Node`、`MakeCnsSpecialNode`、`MakeLitHard`、`MakeLitStr`、`MakeLitStrOfHlnam`、`MakeLitVar`、`MakeLitDefault`、`MakeOpstrInChunkFromBstr` | 常量/字面量叶子 |
| 符号/引用 | `MakeSymNode`、`MakeSymNodeHvdefnPtdata`、`MakeSymTempVar`、`MakeVarIndexNode`、`MakeMeQualifier`、`MakePrdtTree`、`MakeImportOfBytes`、`MakeImportOfParent` | 符号/变量/成员/导入引用 |
| 临时/地址 | `MakeStoreAdrTemp`、`MakeFStLongTemp`、`MakeHardType`、`MakeClearOutNode`、`MakeNewIfNullNode` | 临时存储/地址/清理/条件 |
| 运算符/表达式 | `MakeUnaryOpNode`、`MakeSeq1Node`（配合 `FoldUnaryExpr`/`FoldBinaryExpr`） | 一元/顺序/折叠表达式 |
| 调用/内建 | `MakeOneArgCall`、`MakeTwoArgCall`、`MakeThreeArgCall`、`MakeHelperCall2`、`MakeIntrinsicModeArg`、`MakeVectorRet`、`MakeCRec2UniFuncRet`、`MakeFreeListArg` | 调用/内建/返回 |
| 参数/约定 | `MakeByrefArrayParam`、`MakeByrefCopyInOut`、`MakeCDargRefAry`、`MakeHiddenParam`（配合 `InsertHiddenParam`/`InsertNamedArg`） | ByRef/隐藏/命名参数 |
| 类型转换 | `MakeFloatCoerce`、`MakeCastIn`、`MakeCastOut`、`MakeCVar`、`MakeCVarNonOwner`（配合 `Coerce*`） | 转换/强制/Variant |
| 赋值/记录/数组 | `MakeArrayAssign`、`MakeRecordAssign`、`MakeRecAssignTree`、`MakeCopyRecordLetRHS`、`MakeDestructRecord`、`MakeAryLock`、`MakeRedimTree`、`MakeRedimParamArray`、`MakeGetPutAryDescr`、`MakeGetPutDescr`、`MakeFxRecAryDesc`、`MakeFxstrInput`、`MakeStFixedStr` | 赋值/记录/数组/Redim/GetPut |
| 控制流 | `MakeNewIfNullNode`（配合 `DecideCondBranch`、`CntlBind_*`） | 条件/分支 |
| 临时释放/资源 | `MakeFreetmps`、`MakeFreetmpList`、`MakeFreeListArg`（配合 `Freetmp*`/`Tpool*`） | 临时释放/资源描述 |

---

## 十三、结论

- **`Make*` 家族 = BIL 节点工厂**：每个函数构造一类 `stmtExpr` 节点，覆盖常量、符号、临时、运算、调用、参数、转换、赋值、记录、数组、控制流、资源释放等。
- **分层关系**：
  - `Make*`：构造**叶子/小节点**；
  - `Compile*`/`Process*`：调用 `Make*` 组合成**语句/表达式子树**；
  - `CompileLine`：逐行把这些子树拼成**函数 BIL 树**；
  - `GenerateFunction`：`TransformToLoBil` + `nativeGenProcedure` 把 BIL 树推进到 LowBil/Native。
- **命名规律**：
  - `MakeCns*` → 常量节点；
  - `MakeLit*` → 字面量节点；
  - `MakeSym*` → 符号节点；
  - `Make*Call`/`MakeHelperCall*` → 调用节点；
  - `Make*Coerce`/`MakeCast*`/`MakeCVar` → 类型转换节点；
  - `Make*Assign*`/`Make*Record*`/`Make*Array*`/`Make*Redim*` → 赋值/记录/数组节点；
  - `Make*Param*`/`MakeHiddenParam`/`MakeByref*` → 参数节点；
  - `MakeFreetmp*`/`MakeFreeListArg` → 临时释放节点。

如果你需要，下一步我可以：
1. 按 **BIL 操作码/节点枚举**（而不是语义分类）再整理一版；
2. 结合 `CompileLine` 分析 `Process*` 如何调度这些 `Make*`；
3. 分析 `stmtExpr` 节点的字段结构（哪些字段对应操作码/左/右/类型/临时等）。
