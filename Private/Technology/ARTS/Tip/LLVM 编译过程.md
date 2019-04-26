# LLVM 编译过程

- 替换宏，将代码补全
- 词法、语法分析，生成AST（抽象语法树），方便代码静态检查
- AST生成IR（中间状态代码），和平台无关
- IR生成不同平台机器码，iOS就是Mach-O

![](https://github.com/imwangxuesen/Blog/blob/master/Private/temp/LLVM%E7%BC%96%E8%AF%91%E6%B5%81%E7%A8%8B.png?raw=true)