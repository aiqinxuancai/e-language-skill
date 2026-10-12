# e-packager 第三方工具链

## 目录

- 项目与下载
- 定位与适用场景
- 能力总览
- 解包
- 文本工作区结构
- 编辑边界
- 回包
- 分级验证与 IDE 编译检查
- 试验性直接编译
- 窗口 XML 编辑
- 刷新依赖与资源
- 导出支持库接口
- 一致性验证
- 版本与自更新
- Agent 标准工作流
- 限制与风险

## 项目与下载

- 项目地址：[aiqinxuancai/e-packager](https://github.com/aiqinxuancai/e-packager)
- Release 下载页：[GitHub Releases](https://github.com/aiqinxuancai/e-packager/releases)

先运行 `e-packager version` 和 `e-packager --help`，已有可用工具时直接使用。`dev` 版本字符串不能证明命令能力；本文按上游 2026-10-12 的源码与文档更新，旧程序可能尚不支持 `validate --level` 等选项。缺少所需能力时明确报告版本差异，不静默跳过验证。

需要下载时，在 Release 下载页打开目标版本，下载 `e-packager-<version>-windows-win32.zip` 并解压。e-packager 是便携式命令行工具，无需安装；可直接运行解压得到的 `e-packager.exe`，或将其所在目录加入 `PATH` 后使用本文命令。不要下载 GitHub 自动生成的 `Source code` 压缩包，它不包含可直接运行的程序。

## 定位与适用场景

e-packager 是独立的第三方易语言工程转换工具，不是易语言 IDE、语言语法、易模块或支持库的一部分。它把二进制 `.e/.ec` 工程转换为可读目录，并把受支持的目录修改回包为易语言工程，使 Git Diff、代码审查、AI Agent 编辑和自动化验证成为可能。

在以下任务中使用它：

- 外部 Agent 需要读取或修改磁盘上的 `.e/.ec`，但没有运行中的易语言 IDE。
- 需要把工程代码转换为可搜索、可比较的文本工作区。
- 需要导出已引用模块和支持库的公开接口供 Agent 查询。
- 需要通过 Git 管理易语言文本源码。
- 需要新增模块、支持库或二进制资源并更新工作区派生内容。
- 需要验证解包/回包是否保持工程一致。

已经通过 AutoLinker MCP 操作 IDE 内存工程时，优先遵循 AutoLinker 工作流；其镜像虽然由 e-packager 生成，但真实页写入、哈希 CAS 和 IDE 状态由 AutoLinker 管理，不要绕过 MCP 直接回包覆盖当前工程。

## 能力总览

e-packager 可以：

1. 解包未加密或带打开密码的 `.e/.ec`。
2. 将代码页、窗口定义、固定表、资源和工程元数据导出为目录。
3. 同步导出已引用 `.ec` 模块和支持库公开接口。
4. 把受支持的文本工作区重新打包为 `.e`。
5. 刷新模块/支持库派生目录，并向资源表加入图片、音频或任意二进制资源。
6. 在 Win32 版本中直接导出 x86 `.fne` 支持库的公开接口。
7. 比较原工程与工作区，执行往返目录一致性验证。
8. 执行可编辑回包、静态语义和 IDE 实际编译三级验证。
9. 在用户明确选择时执行试验性的源码直接编译。
10. 查询版本并从 GitHub Release 自更新。

常规解包/回包不等于编译。新版另有试验性 `compile`，与依赖真实 IDE / AutoLinker 的 `compile-check` 是不同链路；都不能替代任务相关的真实运行测试。

## 解包

```powershell
e-packager unpack <input.e|input.ec> <output-dir>
```

示例：

```powershell
e-packager unpack MyApp.e MyApp\
e-packager unpack MyModule.ec MyModule\
```

只解包主工程、不刷新易模块和支持库派生内容时使用：

```powershell
e-packager unpack MyApp.e MyApp\ --main-only
```

`--main-only` 不生成、更新或删除 `ecom/`、`elib/`，也不写入模块依赖的派生辅助路径。需要完整依赖上下文的 Agent 不应使用此模式。

工程带打开密码时：

```powershell
e-packager unpack Protected.e Protected\ --password 111222333
```

也可把 `.e/.ec` 拖放到 `e-packager.exe`，在源文件旁生成同名目录。自动化任务优先使用显式命令和路径。

## 文本工作区结构

| 路径 | 内容 | Agent 策略 |
| --- | --- | --- |
| `src/` | 源码 `.txt`、窗口定义 `.xml` 和固定表 | 编辑 `.txt`；XML 按下文窗口编辑规则处理 |
| `project/` | 回包元数据；`.e` 还可能包含原生快照 | 保留快照；仅按明确字段语义修改配置 |
| `header/` | `.ec` 的公开接口头文件 | 派生只读，不参与回包 |
| `ecom/` | `.e` 引用的模块解包工作区 | 派生只读，不参与主工程回包 |
| `elib/` | 支持库公开接口 | 派生只读，不参与回包 |
| `image/` | 图片及任意二进制资源，索引为 `image/list.json` | 通过 `update` 维护 |
| `audio/` | 音频及任意二进制资源，索引为 `audio/list.json` | 通过 `update` 维护 |
| `tool/e-packager.exe` | 随工作区携带的工具 | 可用于当前工作区回包 |
| `info.json` | 来源类型、路径、时间和 MD5 等信息 | 保留 |
| `README.md` | 自动生成的工作区及工具来源说明 | 按需阅读 |
| `AGENTS.md` | 面向 Agent 的工作区说明 | 修改前完整阅读 |

完整解包 `.e` 时，模块被导出到 `ecom/<模块名>/`，支持库接口被导出到 `elib/`。`project/.module.json` 中的 `resolvedPath` 和 `localWorkspace` 是本机派生辅助字段，不参与回包。

`project/_meta.json` 的 `systemInfo` 保存工程类型、编译器版本与系统保留数据。`systemInfo.compileType` 已知值为 `0`（窗口 EXE）、`1`（控制台 EXE）、`2`（DLL）、`1000`（易模块），未知整数也应保留。`projectSubsystem` 的 `windows` / `console` 仅用于 EXE；DLL、易模块及未知类型应为 `unknown`。普通源码编辑和 `update` 不改变这些属性；主动转换工程类型时才同步调整两个字段、入口代码和公开接口。旧目录可从 `.native_source.bin` 恢复信息，不要删除原生快照。

## 编辑边界

- 业务代码主要位于 `src/*.txt`。
- 窗口 `src/*.xml` 支持有明确元数据的事件绑定和属性编辑，见下文；未知 schema 或私有属性格式不猜测。
- `project/` 是回包所需元数据，不删除、不重排、不凭猜测手改。
- `ecom/`、`elib/`、`header/` 是派生参考。修改它们不会改变主工程引用的模块或支持库。
- 新增模块、支持库和二进制资源使用 `update`，不要只复制文件或编辑派生接口。
- 编辑声明和控制流时仍需遵循易语言文本格式规则；e-packager 不会把错误业务代码自动修正为合法易语言。

## 回包

```powershell
e-packager pack <input-dir> <output.e|output.ec> [--password <text>]
```

示例：

```powershell
e-packager pack MyApp\ MyApp-new.e
e-packager pack MyApp\ MyApp-protected.e --password 111222333
```

在工作区根目录或 `tool/` 子目录直接运行 `e-packager`，会自动输出到 `pack/`。

未修改的 `.ec` 工作区可以回包为原始 `.ec`，保持字节一致；显式指定 `.e` 则转换为易语言源工程。修改源码或资源后应输出 `.e`，再由易语言编译为模块；工具会拒绝把修改后的工程伪装成 `.ec`，不会复用旧模块快照丢弃修改。

```powershell
e-packager pack MyModule\ MyModule-copy.ec   # 仅用于未修改工作区
e-packager pack MyModule\ MyModule-edited.e  # 修改后交 IDE 编译为模块
```

`pack` 和无参回包默认执行可编辑级别预检。可保留的表达式错误、未知调用等会输出 `source_preserved_unexamined` 警告，并作为原生未检查语句保存，不自动注释、修正或删除源码，也不回退到旧执行快照。无法安全构造的流程块、错误声明、缺失资源或无法解析的类型身份与窗口结构仍会阻止回包。

加密工程解包后，默认回包结果不保留打开密码；需要继续加密时显式向 `pack` 传 `--password`。不要在日志或提交中泄露密码。

## 分级验证与 IDE 编译检查

| 命令或级别 | 检查内容 | 结果边界 |
| --- | --- | --- |
| `validate --level editable` | 检查目录、声明、流程结构并实际构造回包字节 | 表达式问题可为警告；不证明 IDE 已打开或编译成功 |
| `validate --level semantic`（默认） | 静态语法、符号、成员、调用签名及可确定类型检查 | 不调用 IDE，不代表真实编译结果 |
| `validate --level compile` | 可编辑回包后调用 IDE / AutoLinker 编译 | 不要求先通过 semantic；不运行产物 |
| `compile-check <文件>` | 对已有 `.e/.ec` 调用 IDE / AutoLinker 编译 | 不运行产物 |

```powershell
e-packager validate MyApp\ --level editable --diagnostics json
e-packager validate MyApp\ --level semantic --diagnostics json
e-packager validate MyApp\ --level compile --diagnostics json --eide "D:\易语言\e.exe"
e-packager compile-check MyApp-new.e --eide "D:\易语言\e.exe"
e-packager pack MyApp\ MyApp-new.e --compile-check --eide "D:\易语言\e.exe"
```

实际编译要求易语言 IDE 的 `lib` 中已启用 `AutoLinker.fne`。编译选项为 `--eide`、`--compile-target auto|win_exe|win_console_exe|win_dll|ecom`、`--compile-static` 和 `--compile-timeout <秒>`；用于 `validate` 时只能配合 `--level compile`。需要控制台输出验证时显式选择 `win_console_exe`，需要模块目标时选择 `ecom`。

`pack --compile-check` 先生成临时 `.e` 并编译，成功后才提交到目标路径；编译失败不覆盖已有文件。该组合要求 `.e` 输出，不能与非空 `--password` 一起使用，因为 AutoLinker 无法接收回包密码。可先验证未加密工作区，再单独回包加密版本。

`validate --diagnostics json` 的退出码与状态如下（不要套用到其它命令）：

| 退出码 | `status` | 含义 |
| --- | --- | --- |
| `0` | `passed` | 所选级别通过 |
| `1` | `failed` | 验证失败，包括 IDE 明确报告编译失败 |
| `2` | `unavailable` | IDE 不可用、超时或缺少可靠结果，未完成编译验证 |

JSON 含 `level`、`status`、`ok`、`diagnostics`、`runtimeWarnings`，实际编译结果在 `ideResult`。`rebuildVerified=true` 仅说明回包构造成功，`ideOpenVerified=false` 表示未独立验证 IDE 打开。未核实的文件或行号可为 `null`，不要自行推断。静态检查失败时分析诊断；IDE 不可用时报告验证限制，不自动改用试验性 `compile`。

## 试验性直接编译

只有用户明确要求使用无 IDE 编译或 `e-packager compile` 时才调用；不自动用于日常编辑、回包、验证或其它检查失败后的替代。此功能不保证稳定性或与 IDE 兼容。

```powershell
e-packager compile MyApp.e MyApp.exe --arch x86
e-packager compile MyApp\ MyApp-x64.exe --arch x64 --blackmoon-x64-dir "D:\deps\BMC\adapter"
e-packager compile MyLib.e MyLib.dll --dll
```

- 默认 `--compile-mode semantic`，支持 x86/x64，输入为 `.e` 或拆包目录。需要完整 MSVC 工具链、Windows SDK 与对应架构的 BlackMoonModernCore adapter。x64 不加载第三方支持库，不把 x86 工程可编译当作 x64 兼容保证。
- `--subsystem auto|console|windows` 控制 EXE 子系统；`--define <宏>` / `-D <宏>` 指定条件编译。`--codegen-opt speed|size` 默认 `speed`，`--semantic-opt baseline|reachable|typed` 默认 `typed`；`--optimization-report <json>` 可输出优化与编译报告。
- 可用 `--vc-tools-dir`、`--windows-sdk-dir`、`--compiler`、`--linker` 指定完整工具链，`--e-dir` 指定易语言目录。易语言自带的 `linker` 目录不是完整 MSVC/SDK。
- 缺少工具链时可能提示安装 VS2026 Build Tools，缺少现代黑月依赖时也可能询问下载；不要在无人值守脚本中自动回答确认。CI 应预装工具链、通过 `--blackmoon-x86-dir` / `--blackmoon-x64-dir` 指定 adapter 并关闭标准输入。目录输入需提前准备 adapter。
- `--compile-mode legacy-blackmoon` 仅支持 x86，仍依赖 IDE、AutoLinker、传统黑月工具链与核心归档；它不属于无 IDE 编译。仅按明确需求选用，需指定 `--legacy-blackmoon-mode asm|cpp|mfc` 等对应配置。

直接编译的图标和版本属性可放在 UTF-8 的 `project/executable.json`：

```json
{
  "icon": "../assets/app.ico",
  "fileDescription": "程序说明",
  "productName": "产品名称",
  "companyName": "开发者",
  "fileVersion": "1.0.0.0",
  "productVersion": "1.0.0.0"
}
```

字段均可省略，还支持 `author`、`legalCopyright`。配置内图标相对配置文件解析；版本接受一至四段 `0–65535` 数字。`--exe-config release.json` 替换整个默认配置，`--icon assets/app.ico` 优先于配置，命令行路径相对当前目录。该配置仅用于 semantic 直接编译，不写回 `.e`，`update` 会保留；legacy 模式不读取它并拒绝这些新参数。它不提供数字签名。

## 窗口 XML 编辑

使用 Win32 版处理经典 x86 FNE 的设计期接口，x64 版不能直接加载这些接口。先读实际 XML、绑定的窗口程序集和支持库元数据，再局部编辑。

- `窗口.事件定义` 是事件签名目录，仅供查询；修改它不会改变支持库定义。控件实例下的事件节点才是绑定。
- 在目标控件下添加事件节点，并在所属窗口程序集 TXT 中添加匹配的处理器；仅新增子程序不会自动绑定事件。

```xml
<按钮.事件 名称="被单击" 处理器="窗口程序集_启动窗口::点击处理" />
```

以上节点放在目标按钮内；对应 `.子程序 点击处理` 位于该窗口程序集。`索引` 可省略，由真实元数据解析；同时写名称与索引时必须一致。带参数或返回值的事件严格遵循目录中的顺序、类型和参考传递标志。菜单项下使用 `菜单.事件`，事件名为 `单击`，处理器无参数、无返回值。未知事件、重复绑定、签名冲突或无法解析的处理器会被拒绝。

属性值按真实支持库类型填写；图片、图标、声音等二进制属性使用 Base64，日期使用 OLE DATE 数值。字体支持结构化节点，例如：

```xml
<编辑框.字体 高度="-24" 粗细="700" 字符集="134" 字体名="宋体" />
```

已识别的列表项目也可编辑结构化子节点。结构化节点优先于同名 Base64 属性；直接改 Base64 前需删除对应结构化节点。自定义属性 `UD_CUSTOMIZE` 的 Base64 仅表示原始字节，不公开私有布局，不能猜造。

工具通过支持库 setter、保存、重建、getter 回读验证属性；未知、只读、非法或不能持久化的赋值会报错。仅允许交互对话框编辑的属性可能无法无头写入，此时用 IDE 设计器。修改后回包并重新拆包核对绑定/属性，再由 IDE 编译和测试实际事件行为。AutoLinker MCP 的镜像 XML 仍遵循其只读边界，不用本节绕过 MCP 修改内存工程。

## 刷新依赖与资源

刷新已解包工作区中的模块/支持库派生内容：

```powershell
e-packager update MyApp\
```

加入模块：

```powershell
e-packager update MyApp\ --add-ecom D:\modules\Net.ec
e-packager update MyApp\ --add-ecom D:\modules\Net.ec --add-ecom D:\modules\UI.ec
```

加入支持库：

```powershell
e-packager update MyApp\ --add-elib 互联网支持库
e-packager update MyApp\ --add-elib D:\易语言\lib\互联网支持库.fne
```

按名称查找 `.fne/.fnr/.dll` 以及 `--add-elib` 仅受 Win32 版支持。按任务已授权的依赖范围操作，并审查工程依赖、部署要求和编译环境变化。

加入资源：

```powershell
e-packager update MyApp\ --add-image D:\res\logo.png
e-packager update MyApp\ --add-audio D:\res\notify.wav
e-packager update MyApp\ --add-image 启动画面=D:\res\splash.bin
```

`--add-image` 和 `--add-audio` 都向易语言常量资源表写入数据，代码通过 `#资源名` 引用。目录用于分类，不限制文件必须是真实图片或音频；任意二进制数据均可写入。显式资源名比依赖文件 stem 更稳定。

## 导出支持库接口

Win32 版可加载 x86 `.fne` 并导出与 `elib/*.txt` 相同类型的公开接口：

```powershell
e-packager decrypt-fne MyLib.fne MyLib.txt
```

省略输出路径时通常在同目录生成同名 `.txt`。也可把 `.fne` 拖放到 Win32 版程序。x64 版不能加载 x86 支持库，因此不提供此能力。

新版导出还包含属性/事件的零起始索引、枚举、状态与平台、事件返回值及参数/参考传递标志；通用负索引事件仍查窗口 XML 的 `窗口.事件定义`。TXT 是定义表，不是窗口实例当前属性值，也不公开私有自定义数据布局。

导出文本用于查询命令、类型和组件签名，不是支持库实现源码，也不能通过修改文本改变 `.fne`。

## 一致性验证

比较原文件与现有工作区：

```powershell
e-packager compare-bundle <input.e|input.ec> <input-dir> [--password <text>]
```

执行一次解包后立即回包：

```powershell
e-packager roundtrip <input.e|input.ec> <work-dir> <output.e|output.ec> [--password <text>]
```

执行往返并比较两次拆包目录：

```powershell
e-packager verify-roundtrip <input.e|input.ec> <work-dir> <output.e|output.ec> [--password <text>]
```

`verify-roundtrip` 比较 `original_unpacked` 与 `roundtrip_unpacked` 的目录内容（有派生文件忽略规则），不是原始二进制逐字节比较。要求零修改字节一致时，另用 `Get-FileHash` 比较输入与输出哈希；`.ec` 必须输出 `.ec`，显式转换为 `.e` 不属于字节一致测试。

**`verify-roundtrip` 会清空指定的 `<work-dir>` 再创建测试目录**，必须使用新的专用临时路径，不可传已有编辑工作区或原工程目录。

使用次序：

1. 引入 e-packager 或升级版本时先对未修改工程执行 `verify-roundtrip`。
2. 修改文本后按需运行 `validate` 分级验证，再回包并用 IDE / AutoLinker 编译；必要时在 IDE 中检查真实页面。
3. 只有需要验证“无修改往返”时才要求字节一致；业务修改后的输出本来就会不同。
4. 校验失败时保留原文件、工作目录和输出文件，先定位格式支持问题，不继续覆盖原工程。

## 版本与自更新

```powershell
e-packager version
e-packager /update
e-packager /update --force
```

`/update` 从 GitHub Release 下载匹配的 Win32 版本并在当前进程退出后替换程序。当前没有对应发布包的平台可能跳过自更新。自动化环境中不要未经用户允许升级工具；版本变化可能改变导出格式或回包行为。

## Agent 标准工作流

1. 备份原始 `.e/.ec`，记录 e-packager 版本和输入哈希。
2. 完整解包；只有明确不需要依赖信息时才使用 `--main-only`。
3. 阅读工作区 `AGENTS.md`、`info.json` 和与任务相关的 `src/`、`ecom/`、`elib/`、`header/`。
4. 搜索后局部编辑 `src/*.txt`，窗口 XML 按支持范围处理；保留派生接口和原生快照，仅按明确需求修改工程配置。
5. 需要依赖或资源变更时显式运行 `update`，并审查索引变化。
6. 用 `validate --level editable` 检查回包构造，按需用 `semantic` 定位静态问题；回包到新文件，修改过的 `.ec` 工作区输出 `.e`。
7. 用 `validate --level compile`、`compile-check` 或 `pack --compile-check` 调用 IDE/AutoLinker 编译，再执行任务相关测试；分别报告回包、静态检查、实际编译与运行结果。
8. 对工具本身或格式兼容性存疑时执行 `compare-bundle` / `roundtrip` / `verify-roundtrip`。

## 限制与风险

- e-packager 与底层 OpenEpl 工程解析能力都可能存在格式兼容缺口；使用前备份。
- 成功解包不保证所有编辑信息、窗口行为或第三方扩展都能无损回包。
- 成功回包不保证语法正确、依赖齐全或运行行为正确。
- `ecom/`、`elib/`、`header/` 是快照，依赖更新后可能过期；用 `update` 刷新。
- 密码、本机绝对路径和私有模块位置可能出现在命令或派生元数据中，记录日志和提交前检查敏感信息。
- 不对计算中的动态调用、混淆代码、加密模块或原生 DLL 行为作静态文本保证。
