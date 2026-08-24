# AutoLinker 无头命令行编译

## 适用范围

直接给已安装 AutoLinker 的易语言 `e.exe` 传入无头编译参数，从 PowerShell、CI 或 Agent 的 `exec_command`、`shell_command` 等命令执行工具编译指定磁盘 `.e` 文件。不需要已有 MCP 会话；`AutoLinkerTest.exe` 是处理早期启动弹窗和同一工程并发协调的可选启动器。

该流程会为目标 `.e` 启动独立 IDE 实例，默认隐藏窗口，调用 `compile_with_output_path`，写出结构化 JSON，然后退出该实例；不改变其他已打开 IDE 实例的当前工程。它仍依赖真实易语言 IDE 和已加载的 `AutoLinker.fne`，不是脱离 IDE 的独立编译器。

## 前提与发现

1. 把包含无头编译功能的 `AutoLinker.fne` 安装到目标易语言目录的 `lib`，重启 IDE 并启用该支持库。
2. 确认目标 `e.exe` 和输入 `.e` 存在，提前创建输出目录和结果目录。
3. 优先使用绝对路径。可用 `cmd /c ftype E.Document` 查询当前 `.e` 文件关联的 IDE 路径，但仍要确认该 IDE 安装了正确版本的 AutoLinker。
4. 每个任务使用唯一的输出路径和 `--autolinker-result` 路径。显式命令行请求不依赖共享的
   `headless_compile_request.json`；所有任务仍会原子更新同一易语言目录下的
   `AutoLinker/Log/headless_compile_last.json`，因此不能用它区分并行结果。

## 直接调用 `e.exe`

核心格式：

```text
e.exe <input.e> --autolinker-headless-compile \
  --autolinker-output <output> \
  --autolinker-target <auto|win_exe|win_console_exe|win_dll|ecom> \
  --autolinker-result <result.json> \
  --autolinker-startup-timeout <seconds> \
  [--autolinker-static]
```

请求标记也接受 `--autolinker-compile`。常用别名包括：输出路径
`--autolinker-output-path`、项目上下文 `--autolinker-project-source-path`、静态编译
`--autolinker-static-compile` / `--autolinker-no-static`、结果路径
`--autolinker-result-path`，以及启动超时 `--autolinker-startup-timeout-seconds`。
窗口和生命周期可用 `--autolinker-show-window`、`--autolinker-hide-window`、
`--autolinker-no-exit` / `--autolinker-keep-open`、`--autolinker-exit` 控制。
自动化通常保持默认的隐藏窗口和编译后退出。

在 PowerShell 或 Agent 命令工具中调用：

```powershell
$eExe = 'C:\path\to\e5.95.exe'
$inputE = (Resolve-Path 'D:\project\demo.e').Path
$output = 'D:\project\build\demo.exe'
$resultPath = 'D:\project\build\demo.compile.json'

New-Item -ItemType Directory -Force -Path (Split-Path $output) | Out-Null
& $eExe $inputE `
  --autolinker-headless-compile `
  --autolinker-output $output `
  --autolinker-target auto `
  --autolinker-result $resultPath `
  --autolinker-startup-timeout 120
$compileExit = $LASTEXITCODE

if (-not (Test-Path -LiteralPath $resultPath)) {
  throw "AutoLinker 未生成编译结果：$resultPath"
}
$compileResult = Get-Content -Raw -LiteralPath $resultPath | ConvertFrom-Json
if ($compileExit -ne 0 -or -not $compileResult.ok) {
  throw "易语言编译失败：$($compileResult.error)"
}
```

可把整段 PowerShell 直接交给 `exec_command`、`shell_command` 或等价终端工具。外层命令超时必须覆盖 IDE 启动等待和实际编译时间；`--autolinker-startup-timeout 120` 时建议至少给外层 180 秒。若外层先超时，先检查指定结果 JSON 和目标 IDE 进程，不要立即重复编译。

推荐显式传 `--autolinker-result`。直接调用未传时会按输出路径和调用 ID 生成唯一结果文件：
`<output>.headless.<invocation-id>.json`；没有输出路径时则写入
`AutoLinker/Log/headless_compile_<invocation-id>.json`。无论是否显式指定，最近结果还会写入
`AutoLinker/Log/headless_compile_last.json`，后者不适合作为多个自动化任务的稳定结果地址。

## 目标与静态编译

- 默认使用 `--autolinker-target auto`，让 AutoLinker 根据工程类型解析为 `win_exe`、`win_console_exe`、`win_dll` 或 `ecom`。
- 只有用户明确指定目标或 `auto` 无法正确解析时才手工传目标；输出扩展名应与目标匹配：EXE 用 `.exe`，DLL 用 `.dll`，模块用 `.ec`。
- `--autolinker-static` 只用于 EXE/DLL，并且当前工程与 IDE 必须支持静态编译。模块目标不要传该参数。

## 用控制台 EXE 验证逻辑

需要验证一段逻辑的实际行为而不是只验证能否编译时，在 `_启动子程序` 中调用待验证子程序，并用 `标准输出` 打印结果，然后显式编译为控制台 EXE 并运行产物：

```text
--autolinker-target win_console_exe --autolinker-static
```

要点：

- 显式传 `win_console_exe`，不要依赖 `auto`。窗口 EXE 下 `标准输出` 没有可见控制台。
- 易模块（`.ec`）同样可以包含 `_启动子程序`。验证模块逻辑时对模块工程显式指定 `win_console_exe`（需要单文件产物时加 `--autolinker-static`），而不是编译为 `ecom`——编译成模块不会执行入口，也就无法观察输出。
- 静态编译产出单文件，便于直接运行产物而不必准备运行时支持库；工程或 IDE 不支持静态编译时去掉该参数并确保依赖库可加载。
- 编译成功不等于逻辑正确。仍需运行产物并检查 stdout；入口未被执行时先确认工程内是否存在 `_启动窗口` 抢占了入口（详见 [language-basics.md](language-basics.md) 的「程序入口」）。
- 验证用的入口代码属于调试脚手架，完成后还原或明确告知用户。

## 成功判定与排错

同时满足以下条件才算成功：

1. `e.exe` 进程退出码为 `0`。
2. 结果 JSON 的 `ok` 为 `true`。
3. `compile_result.artifact_verified` 为 `true`，或等价地确认 `output_file_exists` 与 `output_file_modified_after_compile` 均为 `true`。
4. 输出文件存在，并且路径与本次请求一致。

不要只搜索控制台中的“编译成功”文本，也不要因旧输出文件仍存在而判断成功。读取 `compile_result.output_window_text`、`error`、`compile_dialogs`、`ide_message_boxes` 和错误位置字段，先修复第一组根因。

常见退出码：

- `0`：编译及产物验证成功。
- `1`：编译失败，或编译后产物未创建/更新。
- `2`：缺少输出路径、目标不合法等请求错误。
- `3`：IDE 运行时或工程信息等待超时。
- `4`：内部编译工具调用或结果解析失败。
- `6`：启动期存在阻塞 IDE 弹窗，运行时未能就绪。

发生 `ide_runtime_not_ready`、`wait_eide_info_timeout` 或启动弹窗错误时，确认目标 IDE 的 `lib/AutoLinker.fne` 已安装并启用、工程依赖可加载，再查看指定结果 JSON 和 `AutoLinker/Log` 日志。

## 可选启动器

`AutoLinkerTest.exe headless-compile` 是直接调用之外的包装层，会通过命令行启动 `e.exe`，同时尝试捕获 AutoLinker 加载前的早期弹窗。它还按规范化后的项目路径建立命名互斥体：同一 `.e` 工程的启动器任务串行，不同工程可以并行。没有显式 `--result` 时，启动器生成唯一的
`<output>.headless.<invocation-id>.json`，并额外更新兼容文件 `<output>.headless.json`；有显式结果路径时只写指定结果和 AutoLinker 最近结果。

```text
AutoLinkerTest.exe headless-compile <e.exe> <input.e> <output> [--target auto|win_exe|win_console_exe|win_dll|ecom] [--static] [--result path] [--timeout seconds]
```

官方 Release ZIP 当前只打包 `AutoLinker.fne`，不保证包含 `AutoLinkerTest.exe`；启动器通常来自匹配版本源码的 `bin/fne_release` 或自行构建。启动器会把捕获到的启动/IDE 弹窗附加到结果 JSON；直接调用不提供同样的早期弹窗捕获。
