# VB5RE

## VBA5.dll

BASIC_TYPESRC::SetupLexerForStatement 表明：
在解析一条语句之前，parser 会先配置并启动 scanner。因此在这个局部流程中，scanner 的 scanStart 先于该语句的语法分析执行；但整体上 scanner 和 parser 是编译器前端中按需协作的两个模块，不能简单判定谁整体先执行。
