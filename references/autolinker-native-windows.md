# AutoLinker 原生窗口与事件编辑

本参考用于通过 MCP 修改当前 IDE 内存中的 `.e` 工程。先按 [autolinker-mcp.md](autolinker-mcp.md) 选择实例并刷新镜像；不要直接编辑镜像目录或回包覆盖正在打开的磁盘工程。

调用前需工程具有 `.e` 路径、IDE 处于空闲状态，并已安装可用的 e-packager；`.ec` 工程不支持此窗口入口。`window_tools_require_idle_ide_main_thread` 或 `unsupported_ide_restore_abi` 应按状态/兼容性处理，不能绕过检查。

## 能力与版本

支持新建窗口，增删改窗口内的控件和菜单，调整属性、顺序、层级及事件绑定。已有 `src/*.xml` 可经 `write_file`、`edit_file`、`multi_edit_file` 修改；新建窗口或同时修改事件源码时使用 `write_window`。`add_new_file` 仍只创建普通程序集或类。

窗口整体删除、窗口本身改名、切换已有绑定的所属程序集尚未开放。依赖窗口不可写。XML 不支持 `restore_file_snapshot`，撤回成功提交的修改需用保留的旧 XML 和相关源码重新提交。

以连接实例的实际工具和 ABI 检查结果为准。2026-11-08 的源码和兼容报告覆盖 5.6、5.71、5.95 及已测试的 5.95 补丁映像；5.0 不支持。这不是所有 5.6 以上版本或第三方控件的兼容保证。工具描述中的“Currently requires e5.95”仍是旧文字，不应据此覆盖当前实现和验证结果；旧安装缺少工具或 ABI 不受支持时，改用 IDE 设计器。

## 基线与提交

1. 读取真实 XML：`read_real_file(file_path="src/窗口名.xml")`，保留完整内容和 `code_hash`。控件标签、属性及事件签名从真实 XML、`窗口.事件定义` 和当前支持库公开接口确认，不凭示例猜测。
2. 只改 XML 时使用现有文本写工具；完整 XML 作为 `write_file.full_code`，并传真实 XML 的 `expected_base_hash`。新增窗口、事件处理器或需要同步引用时用 `write_window`。
3. `write_window.code_files` 有独立基线规则：已有程序集先刷新镜像，用 `read_file` 取得完整页和 `code_hash`；新程序集传空哈希。不要把单独 TXT 写工具的真实页哈希与这里的镜像哈希混用。普通 TXT 写入仍按 `read_real_file` 流程。
4. 已有窗口的 `expected_base_hash` 来自真实 XML；新窗口传空字符串，文件名必须匹配 XML 的窗口名称。`assembly_name` 可省略以创建默认程序集，也可为尚未绑定的窗口指定程序集，不能更换已有绑定。
5. `code_files` 最多 16 个，只允许当前工程程序集 TXT；每项必须有 `file_path`、完整 `full_code`、`expected_base_hash`。窗口 XML 与这些源码在同一事务内提交。

以下是 `write_window` 新建窗口并绑定按钮事件的参数示例；用于已有工程前仍须确认名称无冲突和当前库的真实事件签名：

```json
{
  "file_path": "src/示例窗口.xml",
  "full_xml": "<窗口 名称=\"示例窗口\" 标题=\"示例\" 宽度=\"400\" 高度=\"300\"><按钮 名称=\"确定按钮\" 标题=\"确定\" 左边=\"20\" 顶边=\"20\" 宽度=\"80\" 高度=\"30\"><按钮.事件 名称=\"被单击\" 处理器=\"窗口程序集_示例窗口::确定处理\"/></按钮></窗口>",
  "expected_base_hash": "",
  "assembly_name": "窗口程序集_示例窗口",
  "code_files": [
    {
      "file_path": "src/窗口程序集_示例窗口.txt",
      "full_code": ".版本 2\n.程序集 窗口程序集_示例窗口\n.子程序 确定处理\n示例窗口.标题 ＝ “已单击”\n",
      "expected_base_hash": ""
    }
  ]
}
```

## 控件与事件规则

- 处理器必须属于窗口自己的程序集，返回类型、参数顺序和参考标志须匹配真实事件定义。`窗口.事件定义` 是查询目录，改写它不会改变 FNE 的事件签名。菜单事件使用 `<菜单.事件 名称="单击" 处理器="程序集::子程序"/>`。
- 每条绑定使用独立处理器；多个控件不能共用同一事件处理器，否则返回 `window_event_handler_cannot_be_shared`。把共用业务逻辑提取到普通子程序，再由各个处理器调用。
- 只新增事件子程序不等于绑定事件，必须同步 XML。IDE 会把处理器规范化为 `_控件名_事件名`，以最终返回和刷新后的源码为准。
- 显式解绑时工具会把保留的旧处理器改为 `已解绑_…`，并调整可确认的同程序集直接调用和取地址，结果见 `detached_event_handlers`。同名变量、成员访问或跨程序集引用不能可靠推断时，用 `code_files` 显式同步改名和引用；不要靠删除 XML 后保留自动绑定名称来解绑。
- 列表项目重排/删除后，将 `索引` 更新为从 0 开始的连续值，或全部省略以采用 XML 顺序。这是 XML 集合位置，不是易语言数组下标。
- 保留不涉及的原始扩展属性与资源；第三方 FNE 的交互式私有属性未必能通过此接口持久化，失败时按诊断处理或使用设计器。

## 事务结果与失败处理

窗口写入会捕获包含未保存修改的完整内存工程，用已安装的 e-packager 解包/回包，再通过原生恢复 ABI 载入并核对整个工程。安装了 EideCore 时额外进行加载验证；`source_validation.performed=false` 表示跳过该项，加载验证不等于源码编译。

成功应有 `ok=true`、`verified=true`、最终 `code_hash`、`source_path_preserved=true` 和 `project_saved=false`。这表示内存工程已修改，没有保存原 `.e`。事务可能重置撤销、光标和断点；源码空行/缩进等规范化列在 `source_format_normalized` 中。不要在成功后重复写入；继续编译并验证真实布局和事件行为。

后台处理期间有其他工程修改会拒绝提交，应重新刷新并取基线。加载后验证失败通常会回滚，但必须检查实际 `rolled_back`、`rollback_skipped` 和 `safety_snapshot`；`window_project_changed_after_reload` 表示其他修改使旧快照不能安全覆盖当前工程。不要宣称所有失败都已回滚，不要盲目重试。失败中的 `candidate_code_hash` 不是当前真实页哈希。

## 维护依据

本参考依据 AutoLinker 仓库的 `src/AIChatPublicToolCatalog.inc`、`src/EideWindowTools.cpp`、`docs/native-window-editing.md` 与 `docs/test-results/restore-abi-compatibility-20261108/README.md`。更新时同时核对工具 schema、运行实现和回归范围，不只照抄 README。
