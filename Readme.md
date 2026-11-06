# VB5RE

## VBA5.dll

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
