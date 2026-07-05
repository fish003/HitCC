# `hookSpecificOutput` 与特殊事件消费语义

## 本页用途

- 单独承接 `hookSpecificOutput`、`SubagentStart` 和多类特殊事件的 JSON output 消费语义。
- 把“schema 支持什么”和“调用方真正消费什么”拆开写实。

## 相关文件

- [../02-hook-system.md](../02-hook-system.md)
- [../01-tool-execution-core.md](../01-tool-execution-core.md)
- [../03-permission-mode-and-classifier.md](../03-permission-mode-and-classifier.md)
- [../04-policy-sandbox-and-approval-backends.md](../04-policy-sandbox-and-approval-backends.md)
- [../../04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md](../../04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md)

### `hookSpecificOutput`：当前已能直接确认的分支

当前 JSON output parser 现在已经能直接确认这些 `hookSpecificOutput.hookEventName` 分支：

- `PreToolUse`
  - `permissionDecision`
  - `permissionDecisionReason`
  - `updatedInput`
  - `additionalContext`
- `UserPromptSubmit`
  - `additionalContext`
  - `sessionTitle`
  - `suppressOriginalPrompt`
- `UserPromptExpansion`
  - `additionalContext`
- `SessionStart`
  - `additionalContext`
  - `initialUserMessage`
  - `sessionTitle`
  - `watchPaths`
  - `reloadSkills`
- `Setup`
  - `additionalContext`
- `SubagentStart`
  - `additionalContext`
- `PostToolUse`
  - `additionalContext`
  - `updatedToolOutput`
  - `updatedMCPToolOutput`
- `PostToolBatch`
  - `additionalContext`
- `PostToolUseFailure`
  - `additionalContext`
- `Stop`
  - `additionalContext`
- `SubagentStop`
  - `additionalContext`
- `Notification`
  - `additionalContext`
- `PermissionRequest`
  - `decision`
- `PermissionDenied`
  - `retry`
- `Elicitation`
  - `action`
  - `content`
- `ElicitationResult`
  - `action`
  - `content`
- `CwdChanged`
  - `watchPaths`
- `FileChanged`
  - `watchPaths`
- `MessageDisplay`
  - `displayContent`
- `WorktreeCreate`
  - `worktreePath`

### 顶层 JSON output 字段：`terminalSequence / continueOnBlock / defer`

除了 `hookSpecificOutput`，当前 JSON output parser 还会消费一组顶层字段。这里需要把 schema 支持和活语义分清：

| 字段 | 当前语义 | 边界 |
| --- | --- | --- |
| `terminalSequence` | hook 可返回一段 terminal sequence，解析后进入 `terminalSequence` 输出 | 有 allowlist：只允许 OSC `0/1/2/9/99/777` 与 BEL；`OSC 9` body 还有额外限制，不是任意 escape passthrough |
| `continue: false` | 转成 `preventContinuation: true` | 若带 `stopReason` 会一起上浮；真正是否改变业务流程取决于调用方是否消费 `preventContinuation` |
| `continueOnBlock` | 控制 block 结果是否阻断后续 continuation | 当前证据显示它会反向影响 `preventContinuation`，更像“block 但允许继续”的细粒度开关 |
| `decision: "block"` | 转成 `blockingError` | 这是通用阻断面，多个事件都可能消费 |
| `systemMessage` | 转成 `hook_system_message` attachment | 不等于直接改 system prompt；它作为 hook 结果消息进入上下文 |
| `suppressOutput` | 抑制通用成功提示 | 不改变控制流 |

`PreToolUse` 的 `permissionDecision` 还有一个当前容易漏掉的值：

- `defer`
  - 只在 print-mode 侧有活语义。
  - interactive mode 下会被忽略并记录提示。
  - 只支持单个 tool call；同批次有多个 tool call 时会被忽略，避免 resume 后 orphan sibling。

因此 `defer` 不能写成新的通用 permission mode，也不能写成所有 hook 都能暂停执行。它是 `PreToolUse` 针对 print/resume 场景的窄语义。

### 低频输出的 transcript / UI 边界

`hookSpecificOutput` 不能统一理解成“写回 transcript”或“注入 prompt”。当前更稳的边界是按输出通道分开：

| 输出通道 | 活消费者 | transcript / UI 边界 |
| --- | --- | --- |
| `additionalContext` | `IC(...)` 产出 `additionalContexts`；各调用方决定是否包装成 `hook_additional_context` | `Stop / SubagentStop / SubagentStart / UserPromptSubmit / SessionStart` 等调用方会把它作为 meta attachment 或上下文片段接回；未消费它的事件不会自动改 prompt |
| `systemMessage` | `IC(...)` 统一转成 `hook_system_message` attachment | 是 hook 结果消息，不是直接改写 `payload.system` |
| `terminalSequence` | `Lur(...)` / `kur(...)` allowlist 后调用 `A4o(...)` | 只走 terminal 输出侧；不会进入模型上下文或 permission mode |
| `displayContent` | `MessageDisplay` 的 streaming / completed-message display adapter | display-only；替换屏幕上的 delta / 展示文本，不改变已存 assistant 内容，也不改变模型后续可见 transcript |
| `watchPaths` | `SessionStart / CwdChanged / FileChanged` 的 watch path consumer | 只影响 FileChanged watcher 列表，不是 prompt 或 permission 输出 |
| `permissionRequestResult` | ask 后的 `PermissionRequest` hook consumer | 只在 permission ask 分支有意义，不能替代 `PreToolUse permissionDecision` |
| `retry` | `PermissionDenied` hook consumer | 只告诉模型可重试被拒工具，不是通用 retry policy |
| `elicitationResponse / elicitationResultResponse` | MCP elicitation 前后 hook consumer | 只覆盖 MCP elicitation action/content，不是普通 tool result 改写 |
| `worktreePath` | `WorktreeCreate` consumer | 只用于 worktree 创建路径；command hook 仍以 stdout 最后一段非空文本为准 |

因此，低频输出的保守实现应是 typed channel：

- prompt/context 类：`additionalContext`
- transcript message 类：`hook_success / hook_non_blocking_error / hook_system_message / hook_stopped_continuation`
- permission 类：`permissionBehavior / permissionRequestResult / retry`
- display-only 类：`displayContent / terminalSequence`
- lifecycle side-effect 类：`watchPaths / reloadSkills / sessionTitle / worktreePath`

不要把这些通道压成一个 `string output`，也不要把 display-only 输出写成模型可见上下文。

### `SubagentStart`：本地活消费链与顺序

`SubagentStart` 现在不该只记成“支持 `additionalContext`”，而应写成一条更具体的链。

#### 输入形状

`Mzt(agentId, agentType, signal)` 传给通用 hook runner 的 hook input 当前可直接写成：

```text
{
  ...xd(undefined),
  hook_event_name: "SubagentStart",
  agent_id,
  agent_type
}
```

也就是说，这一事件当前稳定带：

- `session_id`
- `transcript_path`
- `cwd`
- `hook_event_name: "SubagentStart"`
- `agent_id`
- `agent_type`

#### 匹配与执行顺序

`et1(...)` 的匹配阶段本身是有顺序的：

- 先收集 `V8z(...)` 返回的 matcher 列表
- 再按 hook 类型重排成：
  - `command`
  - `prompt`
  - `agent`
  - `http`
  - `callback`
  - `function`

但真正执行时，`FC(...)` 会：

- 把每个匹配 hook 变成一个 async generator
- 用 `MC8(...)` 并发跑
- 通过 `Promise.race(...)` 按**完成先后**吐回结果

所以需要明确区分两层：

- matcher/hook 列表顺序：稳定
- hook 返回结果顺序：**完成顺序，不保证等于 matcher 顺序**

#### 当前调用点只消费 `additionalContext`

`SubagentStart` 最大的实现边界在调用方，不在 schema。

`_3(...)` 对 `Mzt(...)` 的消费现在只有：

```text
for await (let r of Mzt(...)) {
  if (r.additionalContexts?.length > 0) z6.push(...r.additionalContexts)
}
if (z6.length > 0) {
  messages.push(ci({
    type: "hook_additional_context",
    content: z6,
    hookName: "SubagentStart",
    ...
  }))
}
```

因此当前本地活语义应写成：

- `additionalContext`
  - 会被收集
  - 会被打包成一个 `hook_additional_context` attachment
  - 会并入 subagent sidechain messages 尾部

而这些东西当前**没有看到活消费者**：

- `blockingError`
- `preventContinuation`
- `systemMessage`
- 通用 `hook_success` / `hook_non_blocking_error` message

换句话说：

- `FC(...)` 通用能力比 `SubagentStart` 当前调用点更强
- 但 `SubagentStart` 的本地活消费面只有 **附加上下文注入**

### Hook 事件全集：本地 bundle 内已基本闭环

运行时事件枚举 `Mp` 与内嵌帮助元数据合起来，当前本地 bundle 可见的事件至少包括：

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

### `O4t("compact")`：本地活入口已经收紧

这一点现在可以写得更硬：

- `Tpp()` 只在 memory scan 末尾消费一次当前 `Yao`
- 消费后立刻把 `Yao` 重置回 `session_start`
- 当前本地 bundle 内，明确命中的 setter 是 compact 成功后调用 `O4t("compact")`
- 该路径同时清空 `hS` cache、memory scan cache 与相关注入状态

因此就当前本地 bundle 可见代码而言，`O4t("compact")` 的活语义已经可以收紧为：

- compact 成功后，为下一次 `Qv()` / `hS()` 相关 memory scan 预置 `load_reason = "compact"`

目前没再看到第二个本地活入口。  
剩余不确定性只在 bundle 外或服务端黑箱。

### `Stop / SessionEnd / SubagentStop / TaskCreated / TaskCompleted / WorktreeRemove` 的 JSON output 消费语义

这 6 个点现在可以分两组写死。

#### 第一组：走通用 hook runner，控制流仍靠通用字段

- `Stop`
- `SubagentStop`
- `TaskCreated`
- `TaskCompleted`

这些事件都会进通用 hook runner，但并不是每个字段都会被调用方消费。当前 JSON parser 已显式消费：

- `PreToolUse`
- `UserPromptSubmit`
- `UserPromptExpansion`
- `SessionStart`
- `Setup`
- `SubagentStart`
- `PostToolUse`
- `PostToolBatch`
- `PostToolUseFailure`
- `Stop`
- `SubagentStop`
- `PermissionDenied`
- `PermissionRequest`
- `Notification`
- `CwdChanged`
- `FileChanged`
- `MessageDisplay`
- `Elicitation`
- `ElicitationResult`

这里需要分清两层：

- `Stop / SubagentStop` 有 event-specific `additionalContext`
- `TaskCreated / TaskCompleted` 当前没有 event-specific branch
- 四者真正改变控制流时，仍主要靠顶层通用字段

这些通用字段包括：

- `continue: false`
  - 变成 `preventContinuation: true`
  - 若带 `stopReason`，会一并上浮
- `decision: "block"`
  - 变成 `blockingError`
- `systemMessage`
  - 变成 `hook_system_message` attachment
- `suppressOutput`
  - 只影响成功时是否额外生成通用 `"hook completed"` 提示，不改变控制流

更具体地说：

- `Stop`
  - 调用方会消费 `blockingError` 与 `preventContinuation`
  - `blockingError` 会被包装成 stop feedback 再塞回模型继续一轮
  - `preventContinuation` 会产出 `hook_stopped_continuation`，并让当前 stop 路径停止继续
- `SubagentStop`
  - 与 `Stop` 同骨架，只是作用对象换成 subagent
- `TaskCreated`
  - 当前调用方只检查 `blockingError`
  - 因此真正能阻止创建任务的，是 blocking path
  - `preventContinuation` / `systemMessage` 对 task create 本身没有控制效果
- `TaskCompleted`
  - 显式 task status 变更路径里，当前调用方只检查 `blockingError`
  - turn 结束时的 teammate/task 收尾路径里，会同时消费 `blockingError` 与 `preventContinuation`

因此这组事件可以直接下结论：

- 没有 event-specific JSON branch
- 真正有控制效果的是通用 `blocking / preventContinuation`
- 其余最多只影响 transcript/UI 提示

#### 第二组：走 `UC(...)`，JSON 基本不进入业务语义

- `SessionEnd`
- `WorktreeRemove`

这两个事件都不走 `FC(...)`，而是直接走 `UC(...)`。

`UC(...)` 对 command/http hook 的 JSON 只会做两类底层处理：

- `decision === "block"` -> `blocked: true`
- 把 `systemMessage / watchPaths` 之类挂回结果对象

但它们的调用方当前都没有继续消费这些字段：

- `SessionEnd`
  - 只对失败 hook 的 stderr 做 `process.stderr.write(...)`
  - 然后清理 session hook 状态
  - 成功 hook 的 JSON 不会改 session end 流程
- `WorktreeRemove`
  - 只看“是否配置了 hook、是否有结果、哪些 hook 失败”
  - 失败会打日志
  - 不消费 `blocked / systemMessage / watchPaths`

所以这组事件现在可以直接写成：

- JSON 可以被底层解析
- 但当前业务调用方不消费其语义字段
- 实际效果基本只剩“命令是否成功/失败”与 stderr 日志

### `WorktreeCreate` 要单独记

`WorktreeCreate` 是 worktree 族里唯一必须保留特例的事件：

- command hook：当前仍以 stdout 路径为准
- callback/http hook：可通过 `hookSpecificOutput.worktreePath` 产出 worktree 路径

这也是为什么 `WorktreeCreate` 在 schema 与 `UC(...)` 中有专门分支，而 `WorktreeRemove` 没有。

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
