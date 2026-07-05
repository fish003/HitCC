# 主循环状态、局部缓存与 `yield` 面

## 本页用途

- 单独承接当前 `zN(...) / oRf(...)` 的主状态机骨架，不把压缩链、stop/recovery 分支和工具执行细节继续混在一起。
- 固定跨 turn 状态 `m`、turn 内局部缓存和 `yield` 面，方便后续从状态与事件两个视角复原主循环。

## 相关文件

- [../04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../03-input-compilation.md](../03-input-compilation.md)
- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../06-stream-processing-and-remote-transport.md](../06-stream-processing-and-remote-transport.md)
- [../../03-ecosystem/07-tui-system/01-repl-root-and-render-pipeline.md](../../03-ecosystem/07-tui-system/01-repl-root-and-render-pipeline.md)
- [../../03-ecosystem/07-tui-system/02-transcript-and-rewind.md](../../03-ecosystem/07-tui-system/02-transcript-and-rewind.md)

## 主循环内核：`zN(...) / oRf(...)`

旧文档里的 `CC / po_` 只能作为历史检索线索。当前 `2.1.197` pretty bundle 里对应的主循环外壳和主体已经改成：

- `zN(e)`：外层 async generator 壳。
- `oRf(e, queuedCommandUuids)`：真正的多轮状态机。

### `zN(e)` 只是壳，不是状态机主体

`zN(e)` 只做几件事：

1. 建 `q = []`，用来暂存 queued command 的 UUID。
2. `yield* oRf(e, q)`。
3. 结束后把 `q` 里记录过的 queued command 全部标成 `completed`。
4. 按 `oRf(...)` 返回的 `reason` 给 `turn` telemetry 标成功或失败。

真正的多轮状态机、分支切换、还原逻辑都在 `oRf(e, q)`。

### `oRf(...)` 的最小伪代码

```text
m = initialTurnState(input)
h = undefined                     // taskBudget remaining shadow

while (true):
  destructure m
  yield stream_request_start
  clone toolUseContext.queryTracking

  le = normalize(messages)
  le = applyContentReplacement(le)

  result = deps.autocompact(le, cacheSafeSnapshot, compactTracking)
  if compacted:
    yield FOo(compactionResult)        // boundary / summary / attachments / hooks
    le = SHe(compactionResult)         // rebuilt transcript, includes messagesToKeep
    compactTracking = reset tracking
  else:
    compactTracking = update consecutiveFailures

  setup streaming tool runner / permission mode / model selection
  for await event/message from callModel(...):
    yield raw stream_event + assistant fragments + partial tool results
    accumulate M6 / $6 / T6 / z6
    if streaming fallback happened:
      yield tombstone for orphaned M6
      reset M6 / $6 / T6 / z6 / tool runner

  if aborted during streaming:
    flush remaining streaming tool results
    return aborted_streaming

  if pendingToolUseSummary from previous turn exists:
    await and yield it

  if no tool_use in this turn:
    handle precomputed/reactive compact / max_output_tokens recovery / malformed tool retry / stop hooks / completed
  else:
    execute tools
    emit attachments / queued commands / skill artifacts
    maybe build pendingToolUseSummary promise
    rewrite J for next turn and continue
```

## 长生命周期状态：`m`

`m` 不是“本轮上下文”，而是 `oRf(...)` 在 turn 之间搬运的唯一主状态。

```ts
interface TurnState {
  messages: TranscriptLikeMessage[]
  toolUseContext: ToolUseContext
  maxOutputTokensOverride?: number
  compactTracking?: {
    compacted: boolean
    turnId: string
    turnCounter: number
    consecutiveFailures?: number
  }
  stopHookActive?: boolean
  stopHookBlockingCount: number
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  thinkingOnlyNudged: boolean
  turnCount: number
  pendingToolUseSummary?: Promise<TranscriptLikeMessage | null> | null
  transition?: { reason: string; [k: string]: unknown }
}
```

### 初始化值

`oRf(...)` 入口直接把 `m` 初始化成：

- `messages = e.messages`
- `toolUseContext = e.toolUseContext`
- `maxOutputTokensOverride = e.maxOutputTokensOverride`
- `compactTracking = undefined`
- `stopHookActive = e.stopHookActive ?? false`
- `stopHookBlockingCount = 0`
- `maxOutputTokensRecoveryCount = 0`
- `hasAttemptedReactiveCompact = false`
- `thinkingOnlyNudged = false`
- `turnCount = 1`
- `pendingToolUseSummary = undefined`
- `transition = undefined`

### `m` 只在少数分支里被整包重写

`oRf(...)` 并不是对 `m` 做零散 mutation，而是在关键分支里直接重建整个对象。当前主路径至少能看到这些重写原因：

#### 1. precomputed/reactive compact 成功后重试

```ts
m = {
  messages: compactedMessages,
  toolUseContext: M,
  compactTracking: Alo(uuid, consecutiveRapidRefills),
  maxOutputTokensRecoveryCount: z,
  hasAttemptedReactiveCompact: precomputedResult === undefined,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: J,
  stopHookBlockingCount: 0,
  turnCount: se,
  transition: {
    reason: precomputedResult ? "precomputed_compact_swap" : "reactive_compact_retry"
  }
}
```

#### 2. `max_output_tokens` 直接续写还原

```ts
m = {
  messages: [...le, ...oe, recoveryMetaUserMessage],
  toolUseContext: M,
  compactTracking: ce,
  maxOutputTokensRecoveryCount: z + 1,
  hasAttemptedReactiveCompact: K,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: J,
  stopHookBlockingCount: 0,
  turnCount: se,
  transition: { reason: "max_output_tokens_recovery", attempt: z + 1 }
}
```

#### 3. malformed tool_use 重试

如果模型返回 `stop_reason === "tool_use"` 但没有解析出有效 tool_use block，当前会补一条 meta user message 再重试一次。`transition.reason === "malformed_tool_use_retry"` 会被下一轮读取，用来决定是否允许再次 retry，并记录 retry outcome telemetry。

```ts
m = {
  messages: cleanRetry ? [...le, retryMetaUserMessage] : [...le, ...oe, retryMetaUserMessage],
  toolUseContext: M,
  compactTracking: ce,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: J,
  thinkingOnlyNudged: V,
  stopHookBlockingCount: 0,
  turnCount: se,
  transition: { reason: "malformed_tool_use_retry", cleanRetry }
}
```

#### 4. thinking-only response 重试

当模型以 `end_turn / stop_sequence` 结束但没有可见文本、不是 compact/fork 特殊路径、也没有 tool_use 时，当前会补一条 meta user nudge 并只重试一次。

```ts
m = {
  messages: [...le, thinkingOnlyMetaUserMessage],
  toolUseContext: M,
  compactTracking: ce,
  maxOutputTokensRecoveryCount: z,
  hasAttemptedReactiveCompact: K,
  thinkingOnlyNudged: true,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: J,
  stopHookBlockingCount: 0,
  turnCount: se,
  transition: { reason: "thinking_only_retry" }
}
```

#### 5. stop hook 产生 blocking error，强制补一轮

```ts
m = {
  messages: [...le, ...oe, ...blockingErrors],
  toolUseContext: M,
  compactTracking: ce,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: K,
  maxOutputTokensOverride: undefined,
  pendingToolUseSummary: undefined,
  stopHookActive: true,
  thinkingOnlyNudged: V,
  stopHookBlockingCount: nextBlockingCount,
  turnCount: nextTurnCount,
  transition: { reason: "stop_hook_blocking" }
}
```

#### 6. 正常工具轮结束，进入下一 turn

```ts
m = {
  messages: [...le, ...oe, ...ae],
  toolUseContext: updatedToolUseContext,
  compactTracking: ce,
  turnCount: se + 1,
  maxOutputTokensRecoveryCount: 0,
  hasAttemptedReactiveCompact: false,
  thinkingOnlyNudged: false,
  pendingToolUseSummary: R6,
  maxOutputTokensOverride: undefined,
  stopHookActive: J,
  stopHookBlockingCount: 0,
  transition: { reason: "next_turn" }
}
```

### `transition` 在当前 bundle 里有少量活消费方

这一点已经不能再写成 write-only。

当前能确认的读点包括：

- `G$o(...)` 会读取 `lastTransitionReason`，其中 `precomputed_compact_swap` 会阻止 precompute compact 再次触发。
- API streaming 结束后，`m.transition?.reason === "malformed_tool_use_retry"` 会记录 retry outcome。
- malformed tool-use 分支会用 `m.transition?.reason !== "malformed_tool_use_retry"` 决定是否还能重试。

所以当前更稳的结论是：

- `transition` 仍主要是 branch marker。
- 但它已经会影响 compact precompute gate、malformed tool retry 上限和 telemetry。
- 不能把它写成纯调试字段。

## turn 内局部缓存

### 跨 turn 外但在一次 `oRf(...)` 调用内长期存在的局部变量

- `h`
  - `taskBudget.remaining` 的本地 shadow。
  - compact / reactive compact 成功时会扣减。
- `y`
  - gate snapshot。
  - 当前直接影响：
    - `emitToolUseSummaries`
    - `fastModeEnabled`
- `b`
  - 长生命周期的预取/上下文 hint handle。
  - 不是消息本体的一部分，但可能在工具阶段后 materialize 成 attachment。

### 每 turn 重建的一组缓存

- `le`
  - 当前 turn 真正送去 `callModel` 的 transcript-like messages。
  - 来源顺序是：clone messages -> content replacement -> autocompact。
- `ce`
  - 当前轮后的 `compactTracking` 快照。
- `Ie`
  - `KHe` streaming tool runner，负责 streaming 阶段已完成结果和最终 drain。

### callModel 期间的四个核心缓存

- `oe`
  - 当前 turn 产生的 assistant 片段列表。
  - 注意不是“最后一条 assistant”，而是整轮里所有 assistant 增量片段。
- `ae`
  - 当前 turn 产生的下游 user-side message。
  - 里面既有 `tool_result`，也会混入 attachment、queued command、skill 等后处理产物。
- `Ae`
  - 当前 turn 所有 `tool_use` block。
- `Ae.length > 0`
  - 当前 turn 是否出现过任何 `tool_use`。

### 其他 turn 级临时槽位

- `In`
  - tool-use summary promise。
  - 本轮只创建，不立即 `yield`；下一轮开头才消费 `pendingToolUseSummary`。
- `wt`
  - 工具执行后可能被 `contextLayers` 推进过的新 `toolUseContext`。

## `yield` 面：`oRf(...)` 对外会发什么

`oRf(...)` 不是只吐 assistant 文本。它对外暴露的是一条混合事件流。

### 直接 `yield` 的类别

- `stream_request_start`
  - 每个 turn 一开始先发。
- `stream_event`
  - 原始模型流事件，来自 `HSt/Hpc`。
- `assistant`
  - 按 content block 收口后的 assistant 片段。
- `attachment`
  - compact、hook、attachment producer、skill discovery、file restore、task status 等都走这里。
- `progress`
  - hook/tool/MCP 进度。
- `system`
  - warning / notification / API error message 等。
- `tombstone`
  - streaming 失败后 fallback 到 non-streaming 时，用于作废已经发出去的 orphaned assistant 片段。

### 在本页最重要的几个特殊 `yield`

- `tombstone`
  - 仅在 `callModel` 已经流出 assistant 片段、随后触发 non-streaming fallback 时出现。
  - 会对 `M6` 里的每条 assistant 逐条发 tombstone，然后清空 `M6 / $6 / T6 / z6`。
- `hook_stopped_continuation`
  - stop hook / task-completed hook / teammate-idle hook 要求终止继续时发。
- `max_turns_reached`
  - 两个位置会发：
    - tools 中断后，但下一个 turn 编号已经超限
    - 正常 tool round 结束后，准备进下一轮时超限

### `tombstone` 的下游消费面

这一块现在也能从“会发 tombstone”压到“谁真正处理它”。

#### 1. 主 TUI / direct-connect UI：把已显示的 orphaned assistant 删掉

流式事件适配器 `cy6(...)` 对非 stream event 有一个专门分支：

```ts
if (A.type === "tombstone") {
  removeMessage?.(A.message)
  return
}
```

这说明 tombstone 不是普通 transcript message，而是：

- **对上层 UI 发出的删除信号**

当前已能确认至少两条活调用链会真的传入 `removeMessage`：

- 主 REPL 本地流式适配
- `useRemoteSession.onMessage(...)` 的 direct-connect / remote UI 适配

这两条链里，`removeMessage` 的实质都是：

- 从本地 `messages` 数组里把那条先前插入的 assistant 对象删掉
- 同时清掉与该 UUID 相关的局部流式状态

所以 tombstone 对 TUI/remote UI 来说不是“提示文案”，而是：

- **撤销此前已经显示出来、但随后被 non-streaming fallback 判定为 orphaned 的 assistant 片段**

#### 2. SDK / `--print --output-format=stream-json`：显式吞掉

SDK/headless 查询器 `xo4.submitMessage()` 在主 switch 里有明确分支：

```ts
case "tombstone":
  break
```

效果是：

- 不写入 `this.mutableMessages`
- 不经 `xx8(...)` 对外产出
- 不转成任何 `stream-json` 协议消息

所以对 SDK 消费者来说：

- **不会收到 tombstone 事件本身**
- 他们只能看到最终保留下来的 assistant / user / system / result 流

#### 3. 内部 hook-agent 流式适配：存在分支，但当前通常没有 remover

内部 hook-agent 也复用 `cy6(...)`，但该调用点没有传 `removeMessage` 回调。  
这意味着在那条内部 stop-hook agent 路径里：

- tombstone 分支会命中
- 但因为 `removeMessage` 缺席，实际效果接近 no-op

这条链本身不是用户主界面的显示面，所以它更像：

- 复用通用 adapter 时留下的兼容分支
- 不是 tombstone 的主要业务消费者

#### 4. 更稳的定位

综合这些读点，tombstone 最准确的定位应是：

- **live-output consistency signal**
- 不是长期 transcript 语义对象
- 也不是还原状态机的控制字段

它主要解决的是：

- streaming 已经吐出一段 assistant
- 但底层随后回退到 non-streaming
- 上层需要把这段“已经展示过、但不该保留”的 orphaned assistant 撤销掉

## 当前仍需保守表述的点

- `tombstone` 在主 TUI、remote UI、SDK/headless 的处理已经钉住；但更外围的桥接端、Web/App 前端是否还会有协议层再解释，本地 bundle 里不再强推。

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
