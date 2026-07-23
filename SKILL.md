---
name: e-language
description: Comprehensive Easy Language (EPL/易语言) project development guidance for understanding, generating, editing, reviewing, debugging, compiling, and migrating .e/.ec projects and their unpacked text workspaces. Use when working with 易语言 syntax, e-packager src/*.txt or window XML, AutoLinker MCP tools, AutoLinker headless command-line compilation of arbitrary disk .e files, assemblies/classes/subroutines, modules and support libraries, BlackMoon/黑月 compilation, 黑月界面类/黑月类模块, RC resources, pure-code Win32 UI, Windows components/events, DLL declarations, custom data types, class-method callbacks and dynamic x86 callback trampolines, layered windows, GDI/GDI+ drawing, 置入代码 or embedded x86 machine code/assembly, build errors, or conversions between 易语言 and other languages.
---

# 易语言开发

## 核心原则

把易语言项目当作由 IDE 管理的结构化工程，不要当作可任意改写的普通文本仓库。

1. 先识别工作模式：AutoLinker MCP 真实 IDE、AutoLinker 无头编译磁盘 `.e`、e-packager 解包目录、只读代码片段，或语言迁移任务。
2. 先读取实际源码、依赖公开接口和项目规范，再推导修改；不要凭其他语言经验补全易语言命令。
3. 区分语言结构与支持库命令：`.子程序`、`.如果` 等是语言结构；`取文本长度` 等通常来自核心或第三方支持库，签名必须从当前工程的 `elib/`、`ecom/`、`header/` 或工具搜索结果确认。
4. 优先做最小、局部、可验证的修改。保留页面类型、声明槽位、事件绑定、缩进、注释和未涉及的代码。
5. 写入前建立真实基线，写入后用结构校验和编译结果判断正确性，不以“看起来像易语言”作为完成标准。

## 选择工作流

### AutoLinker MCP 连接到已打开的 IDE

必须完整阅读 [autolinker-mcp.md](references/autolinker-mcp.md)，并按其刷新、探索、真实页读取、CAS 写入和编译流程操作。

### AutoLinker 无头编译磁盘 `.e`

需要在终端、CI 或 Agent 的 `exec_command`、`shell_command` 等命令执行工具中编译指定磁盘 `.e` 文件时，必须完整阅读 [autolinker-headless-compile.md](references/autolinker-headless-compile.md)。直接调用目标 `e.exe` 的 `--autolinker-headless-compile` 参数；仅在需要处理 AutoLinker 加载前的启动弹窗时使用可选的 `AutoLinkerTest.exe` 启动器。不要求先连接当前工程的 MCP 会话。

### 使用 e-packager 第三方工具链

必须完整阅读 [e-packager.md](references/e-packager.md)，按其解包、文本工作区、依赖/资源更新、回包和一致性校验流程操作。e-packager 是独立第三方工具，不是易语言原生工程机制；理解 `.e/.ec`、模块与支持库语义时另读 [project-model.md](references/project-model.md)，编辑文本声明时另读 [declarations-and-text-format.md](references/declarations-and-text-format.md)。

### 使用黑月编译或黑月界面类

涉及黑月/BlackMoon 编译、黑月界面类、黑月类模块、纯代码窗口、同名 `.rc`、资源对话框或黑月 DLL 时，必须完整阅读 [blackmoon.md](references/blackmoon.md)。需要检查或修改 `.e/.ec` 及黑月模块源码时，同时按 [e-packager.md](references/e-packager.md) 解包并以当前依赖的公开接口为准。

### 生成或修改易语言代码

必须阅读：

- [language-basics.md](references/language-basics.md)：类型、字面量、表达式、调用、数组和控制流。
- [declarations-and-text-format.md](references/declarations-and-text-format.md)：程序集、子程序、变量、参数、常量、DLL、自定义数据类型的固定字段。
- [patterns.md](references/patterns.md)：可复用的正确示例与反例。

涉及窗口、组件事件、类、Win32 API、DLL、内存或回调时，再完整阅读 [windows-and-interop.md](references/windows-and-interop.md)。使用 `UpdateLayeredWindow`、32 位 `DIBSection`、GDI/GDI+ 绘图、预乘 alpha 或透明窗口时，必须按该参考中的分层窗口与画板流程检查样式、像素格式、资源所有权和清理顺序。

涉及 `置入代码`、内嵌汇编、机器码字节集、寄存器或栈帧时，必须完整阅读 [embedded-machine-code.md](references/embedded-machine-code.md)，并同时读取 [windows-and-interop.md](references/windows-and-interop.md)。先确认编译器、后端、目标位数和实际 ABI，再生成或修改机器码；不得照抄未反汇编验证的十进制字节。

涉及第三方类回调模块、类方法回调、运行时动态回调跳板或可执行内存时，必须同时完整阅读上述两份参考。把历史模块视为经典 Win32/x86 ABI 的案例，不把其对象布局、方法序号、栈偏移或机器码当作跨版本语言保证。

### 排错、优化、审查或迁移

必须阅读 [engineering-and-debugging.md](references/engineering-and-debugging.md)。迁移时还要读取语言与项目模型参考，以区分语言语义、支持库行为、UI 事件和平台互操作边界。

### 涉及黑月编译或黑月界面类

当工程依赖黑月界面类、黑月类模块、黑月核心库等，或需要用黑月后端编译原生 Win32 EXE/DLL、创建 RC 资源与从资源/代码创建窗口时，必须完整阅读 [blackmoon.md](references/blackmoon.md)。黑月是与普通编译、静态编译不同的 x86 本机编译与链接后端，相关模块存在多个不兼容代际；先按其证据优先级识别 API 家族和版本，再生成代码，不要用普通编译结果代替黑月编译验证。窗口、控件事件和 DLL 互操作细节另读 [windows-and-interop.md](references/windows-and-interop.md)。

## 修改前检查

执行修改前确认以下事实：

- 当前项目类型、目标位数和入口点。
- 目标页是普通程序集、窗口程序集、类、固定表还是只读依赖。
- 子程序或事件所在页面，以及同名项是否已存在。
- 被调用命令来自核心库、支持库、易模块还是当前工程；确认其参数顺序、可空/参考/数组属性和返回类型。
- 输入源码是 IDE 真实页、工作区镜像还是旧快照。
- 是否存在项目级 `AGENTS.md`、同名 `.AGENTS.md` 或用户约束。

信息不足时继续读取或搜索，不要以猜测填补影响正确性的事实。

## 生成代码的强制规则

- 使用易语言结构，不要输出 C/C++/C#/JavaScript/Python 的花括号、`return`、`switch`、`for`、`while`、`new`、`this` 或 `super` 风格代码。
- 保留以 `.` 开头的系统指令；注释使用单引号 `'`。
- 每个 `.子程序` 内先连续写全部 `.参数`，再连续写全部 `.局部变量`，最后才写执行语句。
- 只可省略声明行尾部未使用槽位；中间空槽必须用逗号保留。
- 需要双分支时使用 `.如果` / `.否则` / `.如果结束`；单分支可使用 `.如果真` / `.如果真结束`，不得混用闭合方式。
- 条件、判断和循环块必须严格嵌套并完整闭合。
- 数组通常从 1 开始索引；不要照搬零基数组习惯。
- 变量声明后已有类型默认值；除非需要非默认初值或明确重置，不生成冗余初始化。
- 普通程序集和窗口程序集的子程序名按全工程解析；新增或重命名前检查重名。
- 控件事件必须留在所属窗口程序集，不能为了整理代码而移动到其他页。
- 只修改某个子程序时，不重写整个页面，不重复添加 `.版本 2`。
- 不确定某条命令是否存在时，搜索当前工程和依赖公开接口；不得发明“看起来合理”的中文命令。
- 使用 `置入代码` 时，只提交可追溯到汇编源码或反汇编结果的常量机器码，并按目标编译模式验证最终产物中的指令、栈平衡和控制流。

## 验证完成条件

按风险从低到高执行：

1. 检查声明字段、声明顺序、字符串/数组维数、控制流闭合和名称冲突。
2. 确认修改没有破坏窗口事件、类公开性、DLL 参数、资源常量或依赖边界。
3. 使用可用的差异预览或结构校验工具。
4. 对实现类任务执行与项目类型匹配的编译；读取完整编译输出，修正根因后重试。
5. 涉及运行期行为、线程、窗口消息、文件/网络、内存和 DLL 时，补充真实场景测试。

只有源码写入成功且所需验证通过，才宣告完成；无法使用 IDE、依赖或编译器时，明确报告未验证部分。
