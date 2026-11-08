# AutoLinker MCP 工作流

## 目录

- 项目与下载
- 能力边界
- 服务与会话
- 强制操作顺序
- 探索源码
- 编辑源码
- 固定表与只读路径
- 图片与声音资源
- 编译与运行验证
- 常见失败处理

## 项目与下载

- 项目地址：[aiqinxuancai/AutoLinker](https://github.com/aiqinxuancai/AutoLinker)
- Release 下载页：[GitHub Releases](https://github.com/aiqinxuancai/AutoLinker/releases)

在 Release 下载页打开最新版本，下载 `AutoLinker-<version>.zip` 并解压。将其中的 `AutoLinker.fne` 放入易语言安装目录的 `lib` 目录，重启易语言 IDE，并在支持库配置中启用 AutoLinker。不要下载 GitHub 自动生成的 `Source code` 压缩包，它不包含可直接安装的 Release 成品。

## 能力边界

当前源码支持普通程序集/类、固定表、原生窗口 XML 及本地图片/声音资源的读写。先查看连接实例的 `tools/list`，不要固定工具数量或用旧 README 的“窗口只读”描述判断能力。

涉及新建窗口、控件增删改、布局或事件绑定时，阅读 [autolinker-native-windows.md](autolinker-native-windows.md)。已有 XML 可用文本写工具，新建窗口和原子更新事件源码用 `write_window`；单独新增事件子程序不等于完成绑定。窗口整体删除、窗口本身改名和更换已有绑定的程序集尚未开放。

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
read_real_file（已有 TXT / XML 的写前基线）
        ↓
diff_file（TXT 预览）或 edit_file / multi_edit_file / write_file / write_window
        ↓
compile_with_output_path（实现任务或用户要求验证时）
```

单实例时可直接调用 `get_current_eide_info`。无论是否切换实例，都先确认当前 `source_file_path`，再刷新镜像。`write_window.code_files` 的基线是专门例外：使用刷新后 `read_file` 的镜像哈希，详见窗口参考。

## 探索源码

- 已知代码项名：优先 `read_code_item`。
- 未知位置：用 `search_code` 批量搜索相关标识符，再用 `read_files` 批量读取必要文件。
- 列目录：用 `list_files`，路径是镜像相对路径。
- 多文件：一次使用 `read_files`，不要串行重复 `read_file`。
- 分页：只有继续上一页时才传其返回的 `mirror_generation`、`next_offset` 或 `next_source_byte_offset`；不得猜测代次值。
- 依赖 API：搜索 `ecom/`、`elib/`、`header/`；这些区域只读。

读取范围以能正确实施任务为准，不设机械调用次数；上下文足够后立即修改。

## 编辑源码

单独编辑已有 TXT 或 XML 前，必须对同一个 `file_path` 调用 `read_real_file`，使用返回的完整真实内容和 `code_hash` 作为基线；分页时取得所有必要内容，不能把局部视图当完整页覆盖。窗口事务内的 `code_files` 按窗口参考取镜像基线。

- `edit_file`：单处精确替换。
- `multi_edit_file`：同一文件多个原子替换。
- `write_file`：大块修改或完整页面覆盖；`full_code` 必须基于完整真实页。
- `diff_file`：预览 TXT 页结构化差异，不写回；不要假定支持 XML。
- `restore_file_snapshot`：恢复 TXT 页快照，先取得当前真实页哈希；不用于 XML。
- `add_new_file`：新建普通程序集或类；新建窗口用 `write_window`。
- `write_window`：创建或替换窗口 XML，可通过 `code_files` 在同一事务更新事件源码。
- `delete_file`：删除普通程序集/类及其全部成员和子程序。先 `read_real_file`，传 `file_path` 与 `expected_base_hash`；不支持窗口绑定程序集、XML、固定表或导入模块，也不自动修复其他页引用或保存工程。成功检查 `deleted=true`、`verified=true` 和 `workspace_mirror_refreshed=true`；若已删除但刷新失败，只刷新镜像，不重复删除。

外部调用 `edit_file`、`multi_edit_file`、`write_file`、`diff_file` 时必须传 `expected_base_hash`；恢复时传 `expected_current_hash`。哈希冲突表示 IDE 内容已变化，应重新读取真实页并重新生成修改，不能绕过 CAS。

写工具返回 `ok=true` 且 `verified=true` 后，不要为了确认而再次读取或重复写入。需要验证时直接编译；只有失败后才重新读取定位。

## 固定表与只读路径

- 可编辑固定表：`src/.数据类型.txt`、`src/.DLL声明.txt`、`src/.全局变量.txt`，以及工具明确支持的常量表。
- 当前工程 `src/*.xml` 经窗口事务写入；详细范围、哈希与事件规则见 [autolinker-native-windows.md](autolinker-native-windows.md)。
- `ecom/`、`elib/`、`header/` 是依赖公开信息，只读。
- 常量表可能含长文本和二进制资源。除非任务明确且工具确认支持，不整体覆盖。
- 模块工具只在用户明确要求时调用：`list_imported_modules`、`add_module_to_project`、`remove_module_from_project`。实际导入需 `allow_blocking_import=true`；移除工程引用不用 `delete_file`。支持库搜索/添加等内置工具不一定对外公开，按当前工具清单判断。

## 图片与声音资源

- `list_resources`：可选 `resource_type=all|picture|sound`，枚举当前工程本地资源；不包含导入模块资源。返回 `resource_id`、`sha256`（原始字节摘要）、`resource_hash`（含元数据的记录摘要）及可用于源码的 `reference=#名称`。
- `add_resource`：传 `resource_type=picture|sound`、唯一 `name`、现有文件的绝对 UTF-8 `file_path` 和可选 `comment`。两个资源表都存储原始字节集，支持任意二进制及空文件，单文件最多 64 MiB；不转换媒体格式，不覆盖同名资源。
- `delete_resource`：先枚举，传整数 `resource_id` 和对应 `resource_hash` 作为 `expected_hash`，不能用 `sha256` 代替。删除不改写源码里的 `#名称` 引用，不删除常量或模块导入资源。

资源增删成功会使镜像失效，继续读源码前调用 `refresh_workspace_mirror`。工具修改 IDE 内存，不自动保存 `.e`。需要更换资源时先检查引用和重名；没有原子替换接口，不把删除后新增当作单次安全覆盖。

## 编译与运行验证

先用 `get_current_eide_info` 确认项目类型、IDE 实例、运行状态和可用编译器。编译前 IDE 应为空闲，运行或断点暂停时先按任务需要结束调试。

- 普通易语言编译：`compiler` 省略或为 `eide`，完全省略 `blackmoon_mode`。`target` 为 `auto|win_exe|win_console_exe|win_dll|ecom`；EXE/DLL 默认静态编译，动态编译显式传 `static_compile=false`。模块工程的 `auto` 生成控制台测试程序；发布 `.ec` 用 `target=ecom`，静态参数省略或为 `false`。
- 黑月编译：必须显式 `compiler=blackmoon`，完全省略 `static_compile`（传 `false` 也冲突），`blackmoon_mode=asm|cxx|vcxx`（默认 `vcxx`）。只接受 EXE/DLL 工程，`target` 为 `auto` 或工程实际类型；需已加载 `BlackMoon.fne` 且相应配置可用。单独传 `blackmoon_mode` 不会切换编译器；冲突时按返回的 `error` / `hint` 修正。成功还要求临时配置恢复成功。

例如普通控制台测试调用：

```json
{"compiler":"eide","target":"win_console_exe","static_compile":true,"output_path":"D:/work/build/test.exe"}
```

### EideCore 自动后端

将匹配版本的独立控制台 `EideCore.exe` 放在 `<易语言主程序目录>/tools/`，普通/静态 EXE/DLL 的 AI/MCP 工具编译自动返回 `backend=eidecore`，调用参数不变；不需要旁置 `EideCore.dll`。未安装核心、黑月、`target=ecom` 和直接 `e.exe` 无头编译仍使用原流程。已进入核心分支后遇到版本或编译错误会报错，不静默回退 IDE。

核心分支编译当前 IDE 内存快照（含未保存修改），要求工程已具有 `.e` 路径；首次未保存的工程须先保存。临时源码放在原源文件目录以保留相对依赖语义，不用磁盘旧源码替代。成功核对响应、PE 目标和 SHA-256 后才发布产物，失败保留旧产物。

检查 `ok`、`output_belongs_to_current_compile`、后端诊断与产物验证，不能只看 `output_file_exists`。`diagnostics[].code/raw_code` 是原生数字错误码；`line_basis=native_page_text_1based` 的行号对应返回的 `source_text`，不直接当解包 TXT 的行号。`request_validation` 错配时先检查请求归属，保留 `backend_result`，不要将别次响应映射成当前源码错误。

### 运行验证

需要验证一段逻辑的实际行为时，在 `_启动子程序` 中调用待验证子程序并用 `标准输出` 打印结果，然后显式指定控制台 EXE 目标（必要时启用静态编译）再运行产物。易模块（`.ec`）也可包含 `_启动子程序`，验证模块逻辑时同样显式编译为控制台 EXE，而不是编译为模块——编译成模块不会执行入口。若入口没有执行，先确认工程中是否存在 `_启动窗口` 抢占了入口，详见 [language-basics.md](language-basics.md) 的「程序入口」。验证用的入口代码属于调试脚手架，完成后还原或明确告知用户。

编译成功必须以工具返回的成功状态、完整输出和产物指纹为准，不以弹窗消失或输出文件曾经存在为准。编译失败时读取第一组根因错误，定位源码和依赖后再修复。

涉及 UI、线程、网络、文件、DLL、回调或运行时状态的修改，编译通过仍不等于行为正确；按用户场景启动 IDE/产物并检查日志。

## 常见失败处理

- `workspace_refresh_required`：调用 `refresh_workspace_mirror` 后重试原读取。
- `expected_base_hash` 缺失：单独 TXT/XML 写入先 `read_real_file`；仅 `write_window.code_files` 使用刷新后镜像 `read_file` 的哈希。
- 哈希冲突：真实页已变化，重新读取并重新应用修改。
- 精确文本未匹配：核对真实页返回内容、换行、引号和 IDE 等价改写；改用更小替换或基于完整页的写入。
- 结构校验失败：修正声明字段、参数/局部变量顺序或控制流闭合，不能关闭校验蒙混写入。
- 页面映射失败：用 `list_files`、`get_current_page_info` 和 `get_current_eide_info` 确认页面类型和路径。
- 编译长时间等待：检查语法错误、IDE 弹窗和工具超时输出，不机械重复调用。
- `ide_execution_in_progress`：当前在运行或断点暂停，结束调试后重试；`ide_execution_state_unavailable` 表示状态不能可靠定位，不强行绕过。
- 窗口事务失败：检查回滚及安全快照字段，按 [窗口参考](autolinker-native-windows.md) 恢复基线；不要用候选哈希继续写入。

## 本地命令与无头编译

AutoLinker 内置 AI 可使用 `exec_command` / `write_stdin` 执行经用户确认的本地非交互式命令，
但这两个工具不是外部 MCP 的源码工具，也不能替代本参考列出的读取、编辑和编译接口。
它们不提供 TTY/ConPTY，只接受管道式输出，非空输入只支持 `Ctrl-C`。外部 Agent 需要编译磁盘
上的其它 `.e` 时，应按 [autolinker-headless-compile.md](autolinker-headless-compile.md)
使用专用参数；不要把外部 MCP 会话的 `compile_with_output_path` 当作磁盘文件编译器。

## 维护依据

2026-11-08 对照 AutoLinker 的 `src/AIChatPublicToolCatalog.inc`、`src/AIChatToolRegistry.cpp`、`docs/program-unit-deletion.md`、`docs/binary-resource-tools.md`、`docs/eidecore-tool-compile.md` 和 `docs/compile-execution-state-verification.md` 更新。工具存在性与参数以连接实例为准；仓库源码支持不代表所有已发布安装都已更新。
