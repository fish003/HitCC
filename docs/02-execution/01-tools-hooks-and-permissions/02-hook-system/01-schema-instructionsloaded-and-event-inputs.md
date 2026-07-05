# Hook Schema、`InstructionsLoaded` 与事件输入

## 本页用途

- 单独承接 hook 配置层 schema、`InstructionsLoaded` 触发链和 HookEvent 输入字段。
- 把“有哪些 hook”“输入长什么样”“哪些 attachment 会进 transcript”固定到同一页。

## 相关文件

- [../02-hook-system.md](../02-hook-system.md)
- [../../02-instruction-discovery-and-rules.md](../../02-instruction-discovery-and-rules.md)
- [../../06-context-runtime-and-tool-use-context.md](../../06-context-runtime-and-tool-use-context.md)
- [../../../05-appendix/02-evidence-map.md](../../../05-appendix/02-evidence-map.md)

## Hook 系统

### Hook 配置层 schema

可确认的 hook event：

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`
- `PostToolBatch`
- `Notification`
- `UserPromptSubmit`
- `UserPromptExpansion`
- `SessionStart`
- `SessionEnd`
- `Stop`
- `StopFailure`
- `SubagentStart`
- `SubagentStop`
- `PreCompact`
- `PostCompact`
- `PermissionRequest`
- `PermissionDenied`
- `Setup`
- `TeammateIdle`
- `TaskCreated`
- `TaskCompleted`
- `Elicitation`
- `ElicitationResult`
- `ConfigChange`
- `WorktreeCreate`
- `WorktreeRemove`
- `InstructionsLoaded`
- `CwdChanged`
- `FileChanged`
- `MessageDisplay`

可确认的 hook def 类型：

- `command`
- `prompt`
- `http`
- `agent`

并支持：

- `timeout`
- `statusMessage`
- `once`
- `async`
- `asyncRewake`
- `allowedEnvVars`
- `model`
- `shell`

### `PreToolUse` 运行时返回

已确认可能产生：

- `message`
- `hookPermissionResult`
- `preventContinuation`
- `stopReason`
- `updatedInput`
- `additionalContexts`
- `stop`

### Hook 结果的 transcript 显示

attachment/type 至少包含：

- `async_hook_response`
- `hook_blocking_error`
- `hook_non_blocking_error`
- `hook_error_during_execution`
- `hook_success`
- `hook_stopped_continuation`
- `hook_system_message`
- `hook_permission_decision`

### 结论

Hook 不是后台机制，而是会进入 transcript 和 UI 呈现。

### `InstructionsLoaded` payload 与触发点

这条现在已经不再是“只有名字，没有形状”。

### 已确认 payload

`InstructionsLoaded` 的 schema 至少包含：

- `hook_event_name: "InstructionsLoaded"`
- `file_path`
- `memory_type: "User" | "Project" | "Local" | "Managed"`
- `load_reason: "session_start" | "nested_traversal" | "path_glob_match" | "include" | "compact"`
- `globs?`
- `trigger_file_path?`
- `parent_file_path?`

### 已确认运行时入口

- `F4t()`：检查是否存在 `InstructionsLoaded` hook
- `X5e(...)`：执行 `InstructionsLoaded` hook
- `Tpp()`：一次性读取当前 load reason，并在消费后重置为 `session_start`
- `O4t(reason)`：设置下一次主扫描应使用的 load reason，并清空 `Qv()` cache

### 已确认触发面

1. `Qv()` / `hS()` 相关主扫描结束后
2. nested memory / opened-file 相关追加装载时

更具体地说：

- 主扫描路径里，会在收集完成后对每个 `User / Project / Local / Managed` memory file 调 `X5e(...)`
- 若该文件是通过 `@include` 引入，则 `load_reason = "include"`
- 否则主扫描路径使用 `Tpp()` 的结果；这一点现在已经可以更具体地确认：
  - `Tpp()` 读取当前 `Yao`
  - 然后把 `Yao` 重置回 `session_start`
  - `O4t("compact")` 是当前本地活代码里最明确的 compact reason 设置点
  - 触发位置在 compact 后清空 `hS` cache 与 memory scan cache，为下一次 context 扫描预置 `load_reason = "compact"`
  - 也就是说它更像主线程 / SDK query 在 compact 后，为下一次 `Qv()` 主扫描预置 `load_reason = "compact"`
  - 它不是 compact 完成时立刻发 hook，而是下一次 memory 扫描结束后才真正 `X5e(...)`
- nested memory 路径里，`load_reason` 会落在：
  - `nested_traversal`
  - `path_glob_match`
  - `include`

还能继续收紧 nested 路径的时序：

- 追加装载逻辑会先判断 `readFileState` 里是否还没有该 memory 文件
- 然后先把它写进 `nested_memory` attachment 与 `readFileState`
- 最后才在 `F4t()` 命中且类型允许时调用：

```text
let reason =
  file.globs ? "path_glob_match"
  : file.parent ? "include"
  : "nested_traversal"

X5e(file.path, file.type, reason, {
  globs: file.globs,
  triggerFilePath,
  parentFilePath: file.parent
})
```

因此这里的三个 reason 不是主扫描 `Tpp()` 分出来的，而是 nested loader 当场按文件来源判定的。

### HookEvent 输入 schema：当前已可直接写实

除了 `PreToolUse` 与 `InstructionsLoaded`，下面这些事件的输入字段也已能从运行时代码直接读出：

- `PostToolUseFailure`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `permission_mode`
  - `agent_id?`
  - `agent_type`
  - `hook_event_name: "PostToolUseFailure"`
  - `tool_name`
  - `tool_input`
  - `tool_use_id`
  - `error`
  - `is_interrupt`
  - `duration_ms?`
- `PostToolBatch`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `permission_mode`
  - `agent_id?`
  - `agent_type?`
  - `hook_event_name: "PostToolBatch"`
  - `tool_calls[]`
    - `tool_name`
    - `tool_input`
    - `tool_use_id`
    - `tool_response?`
- `PermissionDenied`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `permission_mode`
  - `agent_id?`
  - `agent_type?`
  - `hook_event_name: "PermissionDenied"`
  - `tool_name`
  - `tool_input`
  - `tool_use_id`
  - `reason`
- `Notification`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `hook_event_name: "Notification"`
  - `message`
  - `title`
  - `notification_type`
- `SessionStart`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `hook_event_name: "SessionStart"`
  - `source`
  - `agent_type`
  - `model`
  - `session_title?`
- `UserPromptExpansion`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `hook_event_name: "UserPromptExpansion"`
  - `expansion_type: "slash_command" | "mcp_prompt"`
  - `command_name`
  - `command_args`
  - `command_source?`
  - `prompt`
- `StopFailure`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `PERMISSION_MODE?`
  - `agent_id?`
  - `agent_type`
  - `hook_event_name: "StopFailure"`
  - `error`
  - `error_details`
  - `last_assistant_message?`
- `ConfigChange`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `hook_event_name: "ConfigChange"`
  - `source`
  - `file_path`
- `FileChanged`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `hook_event_name: "FileChanged"`
  - `file_path`
  - `event`
- `MessageDisplay`
  - `session_id`
  - `transcript_path`
  - `cwd`
  - `hook_event_name: "MessageDisplay"`
  - `turn_id`
  - `message_id`
  - `index`
  - `final`
  - `delta`

Creative Commons Attribution 4.0 International

Copyright (c) 2026 Hitmux contributors

This work is licensed under the Creative Commons Attribution 4.0 International License.

You are free to:
- Share — copy and redistribute the material in any medium or format
- Adapt — remix, transform, and build upon the material for any purpose, even commercially

Under the following terms:
- Attribution — You must give appropriate credit, provide a link to the license, and indicate if changes were made.

License details:
- Human-readable summary: https://creativecommons.org/licenses/by/4.0/
- Full legal code: https://creativecommons.org/licenses/by/4.0/legalcode
