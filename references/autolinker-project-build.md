# AutoLinker 项目级配置编译

项目级配置编译适合给同一个 `.e/.ec` 工程保存多套输出方案，例如 Debug、Release、控制台测试或模块发布。它由 IDE 菜单触发，和 MCP 的单次 `compile_with_output_path` 是两条不同的路径。

## 配置和执行

1. 在易语言 IDE 中打开目标 `.e` 或 `.ec`。
2. 打开“工具菜单 → AutoLinker 设置 → 编译配置”，新建或编辑配置并保存。
3. 打开“主菜单 → 编译 → 按配置编译”，选择配置名称。

未打开 `.e/.ec` 或配置文件无效时，设置页不可用；没有配置时菜单会显示灰色提示，不会自动创建 Debug/Release 等默认项。配置保存后菜单会重新读取配置列表。

## 配置文件

配置保存在当前源文件旁边：`<文件名>.autolinker.json`。实际持久化字段使用下列 snake_case 名称；WebView 设置页内部使用的 camelCase 消息字段不是文件格式。

```json
{
  "version": 1,
  "active_configuration": "Debug",
  "configurations": [
    {
      "name": "Debug",
      "target": "win_console_exe",
      "static_compile": true,
      "output_path": "{ProjectDir}\\build\\Debug\\{ProjectName}.exe",
      "pre_build": ["Write-Output '开始编译：{ProjectName}'"],
      "post_build": ["Write-Output '编译完成：{OutputPath}'"]
    }
  ]
}
```

删除最后一个配置时，AutoLinker 会删除这个旁路 JSON 文件。手工维护时保留 UTF-8 JSON、配置名称唯一，并先备份文件；优先通过设置页保存，避免写出设置页无法读取的字段。

## 目标、输出和变量

`target` 可取 `auto`、`win_exe`、`win_console_exe`、`win_dll`、`ecom`。`ecom` 不支持静态编译；要运行模块中的测试入口，应选择 `win_console_exe`，而不是发布为 `ecom`。输出路径建议使用绝对路径或以 `{ProjectDir}` 开头，并让扩展名与目标匹配（`.exe`、`.dll`、`.ec`）。相对输出路径按项目目录解析。

输出路径可使用前四个变量；编译前/后动作还可使用 `{OutputPath}`：

| 变量 | 值 |
| --- | --- |
| `{ProgramDir}` | 当前易语言 `e.exe` 所在目录 |
| `{ProjectDir}` | 当前 `.e/.ec` 所在目录 |
| `{ProjectName}` | 源文件名（不含扩展名） |
| `{SourcePath}` | 当前源文件绝对路径 |
| `{OutputPath}` | 已解析的输出路径；供编译前/后动作引用，编译后动作使用最终产物路径 |

编译前/后动作每行执行一条 PowerShell 命令。任一编译前动作失败或包含未知变量都会停止任务；编译后动作只在产物编译成功后执行，失败会在 IDE 输出和 AutoLinker 日志中报告。不要在动作中依赖交互式输入。

## 执行语义和排错

选择菜单后，AutoLinker 会先生成当前工程的快照，再启动一个独立、默认隐藏的 `e.exe` 实例执行无头编译。任务是异步提交的，菜单返回“已提交”不等于产物已经生成；应等待 IDE 输出中的“异步编译完成”，并检查输出文件及日志。快照代表点击菜单时的 IDE 内存状态，之后对原工程的修改不会改变已经排队的任务。

每套配置使用独立输出路径，避免多个任务互相覆盖。若启动弹窗、缺少支持库或输出未更新，按 [autolinker-headless-compile.md](autolinker-headless-compile.md) 检查结果 JSON 和 `AutoLinker/Log`。

“编译”菜单同时包含 IDE 原生命令和 AutoLinker 扩展项。当前版本只消费实际挂在“按配置编译”子菜单中的命令，失效或被摘除的命令会交回 IDE；遇到“新建/保存/编译”等普通菜单无反应时，先更新 AutoLinker，并在禁用其他插件后复测，不要把所有 `WM_COMMAND` 都归因于项目配置功能。
