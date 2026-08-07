# e-language skill

> Agent skill for developing, editing, debugging and compiling 易语言 (Easy Programming Language / EPL) projects — covering AutoLinker MCP, e-packager text workspaces, BlackMoon native compilation, and Win32/GDI+ interoperability.

面向 **易语言（e-language / EPL）** 工程的 AI Agent 技能包：帮助 Agent 正确地理解、生成、修改、审查、排错、编译易语言代码，并处理窗口组件、Win32/DLL、分层透明窗口、GDI/GDI+、原生回调和内嵌机器码等互操作任务，避免把易语言当成普通文本仓库随意改写。

技能正文与全部参考文档均为中文，因为易语言本身以中文标识符和中文命令为主。

## 覆盖的工作模式

- **AutoLinker MCP**：连接到已打开的易语言 IDE，通过 MCP 工具读写「当前 IDE 内存工程」。
- **e-packager**：用第三方工具把二进制 `.e/.ec` 解包为可读文本工作区，编辑后回包。
- **只读代码片段 / 语言迁移**：无 IDE、无工具时的纯代码理解与跨语言迁移。
- **BlackMoon（黑月）**：黑月编译后端与黑月界面类的原生 Win32 编译、RC 资源与窗口创建。

## 安装

```bash
npx skills add aiqinxuancai/e-language-skill
```

> 若你的仓库名不同，把 `aiqinxuancai/e-language-skill` 换成实际的 `owner/repo`。

## 使用

在支持 skill 的 Agent 环境中触发：

- Claude Code：`/e-language`
- OpenAI 系约定：`$e-language`

或让 Agent 在识别到「正在处理易语言工程」时自动加载（依据 `SKILL.md` frontmatter 的 `description`）。

技能采用渐进式披露：默认只加载 `SKILL.md`；`SKILL.md` 会根据当前任务指引 Agent 按需读取 `references/` 下的具体文档，不一次性灌入全部内容。

## 文档地图

| 文件 | 内容 |
| --- | --- |
| `SKILL.md` | 技能入口：核心原则、工作流选择、程序入口、修改前检查、强制规则、验证条件 |
| `references/language-basics.md` | 类型、字面量、表达式、调用、数组、控制流、子程序语义、程序入口（`_启动子程序` / `_启动窗口`） |
| `references/declarations-and-text-format.md` | 程序集/子程序/变量/参数/常量/DLL/自定义类型的固定字段格式 |
| `references/patterns.md` | 可复用的正确示例与常见反例 |
| `references/project-model.md` | 工程、易模块（`.ec`）、支持库（`.fne`）、入口与输出类型、页面与名称解析 |
| `references/windows-and-interop.md` | 窗口/组件事件、Win32/DLL、分层透明窗口、GDI/GDI+ 画板、类方法回调与资源生命周期 |
| `references/embedded-machine-code.md` | `置入代码`、x86 机器码、栈帧/ABI、动态回调跳板、可执行内存与反汇编验证 |
| `references/engineering-and-debugging.md` | 需求分析、编译错误分类、运行期排错、审查、测试、语言迁移 |
| `references/autolinker-mcp.md` | AutoLinker MCP 工作流：刷新镜像、真实页读取、CAS 写入、编译 |
| `references/autolinker-headless-compile.md` | AutoLinker 无头命令行编译磁盘 `.e`：参数、成功判定、控制台 EXE 验证 |
| `references/e-packager.md` | e-packager 解包/回包/依赖资源更新/一致性校验 |
| `references/blackmoon.md` | 黑月编译与黑月界面类：程序入口（`_启动子程序`）、RC 资源、原生窗口、DLL、排错 |

## 要求与边界

- 易语言以 Windows / x86 为主要目标；本技能不假设跨位数行为。
- AutoLinker MCP 需要在易语言 IDE 中加载 `AutoLinker.fne` 后提供本地端口服务。
- e-packager 是独立第三方工具，成功回包不等于代码能编译或运行。
- 透明窗口、GDI/GDI+、动态回调和机器码均依赖实际目标位数、编译后端与 Win32 ABI，必须以最终产物和运行测试验证。
- 从历史工程提炼的内容只保留匿名、可复用的技术机制；技能不收录第三方模块源码、机器码字节、模块名称、作者或其他识别信息。
- 技能只提供方法论与格式约束，不内置编译器；最终正确性以真实 IDE 编译与运行为准。

## 相关项目

- [AutoLinker](https://github.com/aiqinxuancai/AutoLinker)：本技能中 AutoLinker MCP 工作流对应的易语言 IDE 插件。
- [e-packager](https://github.com/aiqinxuancai/e-packager)：把二进制 `.e/.ec` 工程与可读文本工作区互相转换的第三方工具。

## 许可证

[MIT](LICENSE)
