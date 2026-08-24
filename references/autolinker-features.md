# AutoLinker 功能与设置

本文补充 AutoLinker 内置 AI 和 IDE 菜单功能的使用入口。涉及 MCP、无头编译或项目级配置编译时，分别阅读 [autolinker-mcp.md](autolinker-mcp.md)、[autolinker-headless-compile.md](autolinker-headless-compile.md) 和 [autolinker-project-build.md](autolinker-project-build.md)。

## 安装和兼容性

1. 从 Release 获取最新版 `AutoLinker.fne`，放入易语言安装目录的 `lib`，在支持库设置中启用并重启 IDE。
2. 当前主要支持易语言 5.95；Windows 7 不在支持范围内。
3. 已安装版本优先使用“工具菜单 → AutoLinker 设置 → 关于 → 组件更新”更新 AutoLinker 或 e-packager。替换正在加载的 `AutoLinker.fne` 前必须退出 IDE。

AutoLinker 会按需创建运行目录，不需要预先复制配置文件：

| 用途 | 路径 |
| --- | --- |
| 持久化根目录 | `{易语言安装目录}\\AutoLinker\\` |
| AI 配置 | `{易语言安装目录}\\AutoLinker\\AIConfig.json` |
| 日志 | `{易语言安装目录}\\AutoLinker\\Log\\` |
| 全局 SKILL | `{易语言安装目录}\\AutoLinker\\Skills\\` |
| 其他功能配置 | `{易语言安装目录}\\AutoLinker\\Config\\` |
| 临时缓存/镜像 | `%TEMP%\\AutoLinker\\` |

这里的“易语言安装目录”是当前 `e.exe` 所在目录，不是工程 `.e` 所在目录。工程级规范文件为 `.e` 旁的同名 `{文件名}.AGENTS.md`；工程级技能位于 `<工程目录>\\.agents\\skills\\`。

## 内置 AI 对话

左侧“AutoLinker AI 对话”页可直接描述需求，AutoLinker 会按当前 IDE 工程上下文搜索、读取和修改代码。输入 `$skillname` 可让当前轮次强制使用指定技能。

### 图片、计划和目标

- 图片可从输入区选择、粘贴或拖放；单条消息最多 10 张，单个原始文件不超过 20 MB。发送前确认当前模型确实支持图像理解；“自动识别模型能力”是默认选项。
- 计划模式用于调查、方案比较和高影响修改审批。计划阶段只能探索、读取和设计方案，不能写源码、回滚、编译或执行本地命令；提交后必须由用户批准才实施。
- 目标模式用于跨多轮持续任务，支持暂停、恢复、完成和清除。简单的一次性问题不要开启目标模式。计划模式存在时，目标推进和计时会暂停。

### 右键菜单和回退

代码编辑器右键菜单可执行 AI 优化当前函数、添加注释、翻译函数/变量名、翻译选中文本，以及按当前页类型添加代码。鼠标后退键可回到上一个修改位置。

## AI SKILL 和工程规范

通过“工具菜单 → AutoLinker 设置 → AI SKILL”或对话页顶部的 SKILL 入口管理技能。全局技能位于 `{易语言安装目录}\\AutoLinker\\Skills\\`，工程级技能位于 `<工程目录>\\.agents\\skills\\`；同名工程级技能优先。支持本地目录、`SKILL.md`、ZIP、skills.sh 和 GitHub URL，安装技能不会自动执行其中脚本。

通过“工具菜单 → AutoLinker 设置 → 当前项目 AGENTS.md”编辑当前工程的同名规范文件。该文件会作为工程规则注入内置 AI 功能。

## 其他设置页

统一入口是“工具菜单 → AutoLinker 设置”：

| 页面 | 用途 |
| --- | --- |
| AI 接口 | 模型配置组、协议、API Key、源码编辑模式、图片能力和 Tavily |
| MCP 服务 | 查看外部 Agent 使用的本地 MCP 配置 |
| AI 对话配色 | 选择或编辑内置/自定义对话配色 |
| 链接器 | 维护多套 `link.ini`；实际切换在“主菜单 → 编译” |
| EC 模块切换 | 编译时使用静态 ec、调试时使用动态 ec；成对文件必须在同一目录 |
| 核心库函数重写 | 为静态链接维护覆盖核心库函数的 `.lib` 规则 |
| 编译配置 | 维护项目级“按配置编译”，详见 [autolinker-project-build.md](autolinker-project-build.md) |
| 日志优化 | 调整编译日志 Hook 和调试输出路径 |
| 关于 | 查看版本、项目链接和组件更新 |

核心库重写只适用于明确的 32 位静态链接场景；自有 `.lib` 通常需要关闭 `/GL`，并接受链接器 `/FORCE` 带来的告警。不要把它和普通 MCP 编译参数混为一谈。

工具菜单还提供“打开项目目录”“打开 AutoLinker 配置目录”“打开易语言目录”。

## AI 服务配置边界

服务商、协议、模型和 API Key 应在“工具菜单 → AutoLinker 设置 → AI 接口”配置，并先执行连通性测试。不要把 API Key 写入工程规范、源码或公开日志；协议、Base URL 和模型名必须来自同一服务商。模型名称和可用思考等级会变化，不在本技能中固化推荐列表。
