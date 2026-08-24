# AutoLinker MCP 工作流

## 目录

- 项目与下载
- 能力边界
- 服务与会话
- 强制操作顺序
- 探索源码
- 编辑源码
- 固定表与只读路径
- 编译与运行验证
- 常见失败处理

## 项目与下载

- 项目地址：[aiqinxuancai/AutoLinker](https://github.com/aiqinxuancai/AutoLinker)
- Release 下载页：[GitHub Releases](https://github.com/aiqinxuancai/AutoLinker/releases)

在 Release 下载页打开最新版本，下载 `AutoLinker-<version>.zip` 并解压。将其中的 `AutoLinker.fne` 放入易语言安装目录的 `lib` 目录，重启易语言 IDE，并在支持库配置中启用 AutoLinker。不要下载 GitHub 自动生成的 `Source code` 压缩包，它不包含可直接安装的 Release 成品。

## 能力边界

AutoLinker 无法创建或修改原生窗体（易语言窗口界面）。`src/*.xml` 窗口界面定义只读，当前工具不支持修改窗口或控件的位置、大小、层级和属性，不支持添加、删除控件，也不支持新增、删除或修改控件事件绑定；`add_new_file` 只能新建普通程序集或类，不能新增窗口。不要尝试写入窗口 XML，也不要声称已经完成这些界面修改。需要新增或调整窗口界面时，由用户在易语言 IDE 中手工完成后，再用 `refresh_workspace_mirror` 刷新镜像。

工具可以编辑窗口对应的程序集 `.txt`，包括普通子程序和已有控件事件子程序的代码实现；但新增一个事件子程序不代表已经建立控件事件绑定。可编辑固定表和普通程序集/类源码；界面结构及事件绑定仍由 IDE 掌管。

只有用户明确要求解析、查看、参考或复刻磁盘上的其它 `.e/.ec` 文件时，才调用 `e_packager`。
它会把文件解包到当前镜像的 `unimported_code/<文件名>/`，该目录只读；用
`list_files` / `search_code` 定位，再用 `read_file` / `read_files` 读取。相对 `file_path`
按当前工程目录解析，同名参考目录只在解包成功后替换，并会在全量刷新时保留。

## 服务与会话

AutoLinker 在易语言 IDE 中启动本地 Streamable HTTP MCP 服务，外部客户端应连接固定网关
`http://127.0.0.1:19207/mcp`；`19208` 及以上是实例后端端口，不要直接连接。它操作当前 IDE
内存工程，而不是直接解析磁盘 `.e` 文件。

这里的限制只针对 MCP 源码读写工具。需要从终端或 Agent 命令执行工具编译磁盘上的其他 `.e` 文件时，使用独立的 [AutoLinker 无头命令行编译](autolinker-headless-compile.md) 工作流；它会为目标文件启动新的 IDE 实例，不需要把该文件先切换为当前 MCP 工程。

每个外部 MCP 会话在首次源码读写前必须成功调用 `refresh_workspace_mirror`。镜像由 e-packager 从 IDE 当前状态构建，包含未保存修改。`initialize` 会返回 `autolinkerState.source_state`、`source_open` 和 `source_file_path`；状态为 `no_source_open` 时，先让用户打开 `.e/.ec`，不要调用工程读写工具。

同时运行多个 IDE 时，先调用无参数 `list_instances`，再把返回的 `instance_id` 传给
`select_instance`。选择仅对当前 MCP 会话生效，实例退出后不会自动切换到其它工程；切换实例后
必须重新调用 `refresh_workspace_mirror`，不能复用旧镜像、分页代次或哈希。

## 强制操作顺序

```text
initialize
        ↓
list_instances / select_instance（多实例时）
        ↓
get_current_eide_info
        ↓
refresh_workspace_mirror
        ↓
list_files / search_code / read_files / read_code_item
        ↓
read_real_file（每个待修改文件，紧邻写入前）
        ↓
diff_file 或 edit_file / multi_edit_file / write_file
        ↓
compile_with_output_path（实现任务或用户要求验证时）
```

多开 IDE 时，在上述流程前插入 `list_instances` → `select_instance`；单实例时可直接调用
`get_current_eide_info`。无论是否切换实例，都先确认当前 `source_file_path`，再刷新镜像。

## 探索源码

- 已知代码项名：优先 `read_code_item`。
- 未知位置：用 `search_code` 批量搜索相关标识符，再用 `read_files` 批量读取必要文件。
- 列目录：用 `list_files`，路径是镜像相对路径。
- 多文件：一次使用 `read_files`，不要串行重复 `read_file`。
- 分页：只有继续上一页时才传其返回的 `mirror_generation`、`next_offset` 或 `next_source_byte_offset`；不得猜测代次值。
- 依赖 API：搜索 `ecom/`、`elib/`、`header/`；这些区域只读。

读取范围以能正确实施任务为准，不设机械调用次数；上下文足够后立即修改。

## 编辑源码

编辑任何已有页面前，必须对同一个 `file_path` 调用 `read_real_file`，使用返回的 `real_source` 和 `code_hash` 作为基线。

- `edit_file`：单处精确替换。
- `multi_edit_file`：同一文件多个原子替换。
- `write_file`：大块修改或完整页面覆盖；`full_code` 必须基于完整真实页。
- `diff_file`：只预览结构化差异，不写回。
- `restore_file_snapshot`：使用快照回滚，先取得当前真实页哈希。
- `add_new_file`：新建普通程序集或类，不能用于新增窗口（无法创建原生窗体）。

外部调用 `edit_file`、`multi_edit_file`、`write_file`、`diff_file` 时必须传 `expected_base_hash`；恢复时传 `expected_current_hash`。哈希冲突表示 IDE 内容已变化，应重新读取真实页并重新生成修改，不能绕过 CAS。

写工具返回 `ok=true` 且 `verified=true` 后，不要为了确认而再次读取或重复写入。需要验证时直接编译；只有失败后才重新读取定位。

## 固定表与只读路径

- 可编辑固定表：`src/.数据类型.txt`、`src/.DLL声明.txt`、`src/.全局变量.txt`，以及工具明确支持的常量表。
- `src/*.xml` 是窗口界面定义，只读且无法创建或修改；只能编辑对应窗口程序集 `.txt`，包括已有控件事件子程序的代码实现。新增事件子程序不代表已经建立控件事件绑定；新增或改动窗口、控件及其事件绑定需用户在 IDE 中手工完成。
- `ecom/`、`elib/`、`header/` 是依赖公开信息，只读。
- 常量表可能含长文本和二进制资源。除非任务明确且工具确认支持，不整体覆盖。
- 新增/移除模块或支持库只在用户明确要求时使用对应依赖工具。

## 编译与运行验证

先用 `get_current_eide_info` 确认项目类型、IDE 实例和支持的编译目标。`compile_with_output_path` 的 `target` 应匹配工程：`auto`、窗口 EXE、控制台 EXE、DLL 或模块等；EXE/DLL 默认静态编译，只有用户明确要求动态编译时传 `static_compile=false`。模块工程使用 `target=auto` 会生成控制台测试程序；只有发布 `.ec` 时才使用 `target=ecom` 且关闭静态编译。

需要验证一段逻辑的实际行为时，在 `_启动子程序` 中调用待验证子程序并用 `标准输出` 打印结果，然后显式指定控制台 EXE 目标（必要时启用静态编译）再运行产物。易模块（`.ec`）也可包含 `_启动子程序`，验证模块逻辑时同样显式编译为控制台 EXE，而不是编译为模块——编译成模块不会执行入口。若入口没有执行，先确认工程中是否存在 `_启动窗口` 抢占了入口，详见 [language-basics.md](language-basics.md) 的「程序入口」。验证用的入口代码属于调试脚手架，完成后还原或明确告知用户。

编译成功必须以工具返回的成功状态、完整输出和产物指纹为准，不以弹窗消失或输出文件曾经存在为准。编译失败时读取第一组根因错误，定位源码和依赖后再修复。

涉及 UI、线程、网络、文件、DLL、回调或运行时状态的修改，编译通过仍不等于行为正确；按用户场景启动 IDE/产物并检查日志。

## 常见失败处理

- `workspace_refresh_required`：调用 `refresh_workspace_mirror` 后重试原读取。
- `expected_base_hash` 缺失：调用 `read_real_file`，不要用镜像哈希代替。
- 哈希冲突：真实页已变化，重新读取并重新应用修改。
- 精确文本未匹配：核对真实页返回内容、换行、引号和 IDE 等价改写；改用更小替换或基于完整页的写入。
- 结构校验失败：修正声明字段、参数/局部变量顺序或控制流闭合，不能关闭校验蒙混写入。
- 页面映射失败：用 `list_files`、`get_current_page_info` 和 `get_current_eide_info` 确认页面类型和路径。
- 编译长时间等待：检查语法错误、IDE 弹窗和工具超时输出，不机械重复调用。

## 本地命令与无头编译

AutoLinker 内置 AI 可使用 `exec_command` / `write_stdin` 执行经用户确认的本地非交互式命令，
但这两个工具不是外部 MCP 的源码工具，也不能替代本参考列出的读取、编辑和编译接口。
它们不提供 TTY/ConPTY，只接受管道式输出，非空输入只支持 `Ctrl-C`。外部 Agent 需要编译磁盘
上的其它 `.e` 时，应按 [autolinker-headless-compile.md](autolinker-headless-compile.md)
使用专用参数；不要把外部 MCP 会话的 `compile_with_output_path` 当作磁盘文件编译器。
