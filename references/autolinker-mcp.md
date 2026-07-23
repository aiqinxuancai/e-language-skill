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

AutoLinker 无法创建或修改原生窗体（易语言窗口界面）。`src/*.xml` 窗口界面定义只读，不能通过任何工具新增、删除或改动窗口、控件及其属性、布局、事件绑定；`add_new_file` 只能新建普通程序集或类，不能新增窗口。需要新增或调整窗口界面时，由用户在易语言 IDE 中手工完成后，再用 `refresh_workspace_mirror` 刷新镜像。

工具能编辑的是窗口对应的程序集 `.txt`（事件与子程序代码）、可编辑固定表和普通程序集/类源码；界面结构本身仍由 IDE 掌管。

## 服务与会话

AutoLinker 在易语言 IDE 中启动本地 Streamable HTTP MCP 服务，通常为 `http://127.0.0.1:19207/mcp`，端口占用时可能顺延。它操作当前 IDE 内存工程，而不是直接解析磁盘 `.e` 文件。

这里的限制只针对 MCP 源码读写工具。需要从终端或 Agent 命令执行工具编译磁盘上的其他 `.e` 文件时，使用独立的 [AutoLinker 无头命令行编译](autolinker-headless-compile.md) 工作流；它会为目标文件启动新的 IDE 实例，不需要把该文件先切换为当前 MCP 工程。

每个外部 MCP 会话在首次源码读写前必须成功调用 `refresh_workspace_mirror`。镜像由 e-packager 从 IDE 当前状态构建，包含未保存修改。不要把旧会话的路径、哈希或分页代次用于新会话。

## 强制操作顺序

```text
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
- `src/*.xml` 是窗口界面定义，只读且无法创建或修改；只能编辑对应窗口程序集 `.txt`。新增或改动窗口、控件需用户在 IDE 中手工完成。
- `ecom/`、`elib/`、`header/` 是依赖公开信息，只读。
- 常量表可能含长文本和二进制资源。除非任务明确且工具确认支持，不整体覆盖。
- 新增/移除模块或支持库只在用户明确要求时使用对应依赖工具。

## 编译与运行验证

先用 `get_current_eide_info` 确认项目类型、IDE 实例和支持的编译目标。`compile_with_output_path` 的 `target` 应匹配工程：`auto`、窗口 EXE、控制台 EXE、DLL 或模块等。

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
