# e-packager 第三方工具链

## 目录

- 定位与适用场景
- 能力总览
- 解包
- 文本工作区结构
- 编辑边界
- 回包
- 刷新依赖与资源
- 导出支持库接口
- 一致性验证
- 版本与自更新
- Agent 标准工作流
- 限制与风险

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
7. 比较原工程与工作区，执行快速往返或字节一致性验证。
8. 查询版本并从 GitHub Release 自更新。

它不是易语言编译器，也不能替代 IDE 的窗口设计器、调试器和真实运行测试。文本能成功回包不代表业务代码能编译或运行。

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
| `src/` | 源码 `.txt`、窗口定义 `.xml` 和固定表 | 编辑受支持的 `.txt`；窗口 XML 默认只读 |
| `project/` | 回包元数据；`.e` 还可能包含原生快照 | 不手工修改或删除 |
| `header/` | `.ec` 的公开接口头文件 | 派生只读，不参与回包 |
| `ecom/` | `.e` 引用的模块解包工作区 | 派生只读，不参与主工程回包 |
| `elib/` | 支持库公开接口 | 派生只读，不参与回包 |
| `image/` | 图片及任意二进制资源，索引为 `image/list.json` | 通过 `update` 维护 |
| `audio/` | 音频及任意二进制资源，索引为 `audio/list.json` | 通过 `update` 维护 |
| `tool/e-packager.exe` | 随工作区携带的工具 | 可用于当前工作区回包 |
| `info.json` | 来源类型、路径、时间和 MD5 等信息 | 保留 |
| `AGENTS.md` | 面向 Agent 的工作区说明 | 修改前完整阅读 |

完整解包 `.e` 时，模块被导出到 `ecom/<模块名>/`，支持库接口被导出到 `elib/`。`project/.module.json` 中的 `resolvedPath` 和 `localWorkspace` 是本机派生辅助字段，不参与回包。

## 编辑边界

- 业务代码主要位于 `src/*.txt`。
- 窗口 `src/*.xml` 表达设计器信息；没有明确 schema 和往返测试时保持只读，通过 IDE 设计器修改布局/控件。
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

`.ec` 工作区回包的实际输出始终为 `.e` 格式；无参默认路径可能为 `pack/<原文件名>.ec.e`。不要只看扩展名推断内部格式。

加密工程解包后，默认回包结果不保留打开密码；需要继续加密时显式向 `pack` 传 `--password`。不要在日志或提交中泄露密码。

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

按名称查找 `.fne/.fnr/.dll` 以及 `--add-elib` 仅受 Win32 版支持。更新依赖前必须获得用户授权，因为它会改变工程依赖、部署要求和编译环境。

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

执行往返并校验字节一致性：

```powershell
e-packager verify-roundtrip <input.e|input.ec> <work-dir> <output.e|output.ec> [--password <text>]
```

使用次序：

1. 引入 e-packager 或升级版本时先对未修改工程执行 `verify-roundtrip`。
2. 修改文本后执行 `pack`，再用易语言 IDE 打开和编译。
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
4. 搜索后局部编辑 `src/*.txt`；不修改派生接口和元数据。
5. 需要依赖或资源变更时显式运行 `update`，并审查索引变化。
6. 回包到新文件，不直接覆盖唯一原件。
7. 用 IDE/AutoLinker 编译并执行任务相关测试。
8. 对工具本身或格式兼容性存疑时执行 `compare-bundle` / `roundtrip` / `verify-roundtrip`。

## 限制与风险

- e-packager 与底层 OpenEpl 工程解析能力都可能存在格式兼容缺口；使用前备份。
- 成功解包不保证所有编辑信息、窗口行为或第三方扩展都能无损回包。
- 成功回包不保证语法正确、依赖齐全或运行行为正确。
- `ecom/`、`elib/`、`header/` 是快照，依赖更新后可能过期；用 `update` 刷新。
- 密码、本机绝对路径和私有模块位置可能出现在命令或派生元数据中，记录日志和提交前检查敏感信息。
- 不对计算中的动态调用、混淆代码、加密模块或原生 DLL 行为作静态文本保证。
