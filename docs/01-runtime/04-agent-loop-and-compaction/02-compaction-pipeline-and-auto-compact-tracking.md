# 压缩链与 `compactTracking`

## 本页用途

- 单独承接主循环里的压缩链，把 `microcompact`、autocompact、各类 compact producer 和 transcript rebuild 合同拆开写清楚。
- 固定当前 `compactTracking` 的控制面与 telemetry 面；旧文档里的 `autoCompactTracking` 是同一语义域的历史名称。

## 相关文件

- [../04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../03-input-compilation.md](../03-input-compilation.md)
- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../06-stream-processing-and-remote-transport.md](../06-stream-processing-and-remote-transport.md)
- [../../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md](../../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md)
- [../../03-ecosystem/07-tui-system/02-transcript-and-rewind.md](../../03-ecosystem/07-tui-system/02-transcript-and-rewind.md)

## 压缩链

### 预压缩：`microcompact`

当前本地明确活着的 microcompact 不是“复杂 cache edit producer”，而是：

- 入口：`pF(messages, toolUseContext, querySource)`
- 主体：`V4_(...)`
- 触发条件：
  - `tengu_slate_heron.enabled === true`
  - `querySource` 为空或以 `repl_main_thread` 开头
  - 最后一条 assistant 距今超过 `gapThresholdMinutes`
- 实际效果：
  - 从历史 `tool_use` 中只保留最近 `keepRecent`
  - 更老的对应 `tool_result.content` 改写成 `[Old tool result content cleared]`

因此当前 bundle 下，microcompact 的“活效果”就是旧 `tool_result` 文本清空。

### autocompact：`deps.autocompact(...)`

调用参数里最关键的是 `cacheSafeParams`：

```ts
{
  systemPrompt: K,
  userContext: _,
  systemContext: z,
  toolUseContext: v,
  forkContextMessages: F,
  stickyBetas
}
```

当前主循环主体 `oRf(...)` 不是直接硬编码某个旧压缩函数名，而是通过 `deps.autocompact(...)` 调用自动压缩依赖。旧文档里的 `DEq(...)` 只能作为历史检索线索。  

这只能说明自动压缩会把一份 `cacheSafeParams` 继续往下传，不能再概括成“compact 总是直接拿当前 request snapshot”。  
当前更稳的写法是：

- shared-prefix 分支会复用调用方给出的 `cacheSafeParams`
- 但 `fD4(...)` 这类 helper 也可以 fresh-build 新的 `cacheSafeParams`
- 而 summarize 核心调用本身仍有专用 prompt 路径，不等同于主线程 `Lx8(...) / dj4(...)` 分层

### compact 成功后，本轮会立即发生什么

1. 产出 `compactionResult`。
2. 记录 telemetry。
3. 若有 `taskBudget`，扣减 `P`。
4. `Q` 被重置为：
   - `compacted: true`
   - `turnId: uuid`
   - `turnCounter: 0`
   - `consecutiveFailures: 0`
5. `FOo(result)` 产出的 compact 边界/摘要/attachment/hook 被逐条 `yield`。
6. `SHe(result)` 产出的完整 transcript-like messages 写回下一轮 `messages`。

### `compactionResult -> FOo(...) / SHe(...)` 的固定合同

这一层是当前 `2.1.197` 相对旧文档最需要修正的地方。

当前有两个不同 adapter：

```ts
function FOo(result) {
  return [
    result.boundaryMarker,
    ...result.summaryMessages,
    ...result.attachments,
    ...result.hookResults,
  ]
}

function SHe(result) {
  return [
    result.boundaryMarker,
    ...result.summaryMessages,
    ...result.messagesToKeep,
    ...result.attachments,
    ...result.hookResults,
  ]
}
```

这说明当前公共合同不是单一 `hn(...)`。主循环自动压缩成功后：

- 对用户/UI/SDK `yield` 的是 `FOo(result)`，不包含 `messagesToKeep`。
- 写回下一轮模型输入和 transcript-like message 链的是 `SHe(result)`，包含 `messagesToKeep`。

因此 `compactionResult` 对外真正稳定的字段是：

- `boundaryMarker`
- `summaryMessages`
- `attachments`
- `hookResults`
- `messagesToKeep`
- 以及若干统计字段：
  - `preCompactTokenCount`
  - `postCompactTokenCount`
  - `truePostCompactTokenCount`
  - `compactionUsage`
  - 某些路径还会带 `userDisplayMessage`

也就是说，`oRf(...)` 根本不理解 compact 内部细节。  
它只要求 producer 产出一份同时兼容 `FOo(...)` 和 `SHe(...)` 的 `compactionResult`，再按“可见输出”和“下一轮 message 链”两个面分别展开。

### `compact_boundary` 的基础结构

边界 marker 的基型来自 `bn6(...)`：

```ts
{
  type: "system",
  subtype: "compact_boundary",
  content: "Conversation compacted",
  compactMetadata: {
    trigger,
    preTokens,
    userContext?,
    messagesSummarized?,
    preCompactDiscoveredTools?,
    preservedSegment?,
  },
  logicalParentUuid?,
}
```

这里有两个容易混淆的点：

- `logicalParentUuid` 在 message 顶层，不在 `compactMetadata` 里。
- `preservedSegment` 不是基型自带，而是后续由 `Z$o(...)` 条件性补上；旧文档里的 `Xu1(...)` 只作为历史检索线索。

### `preservedSegment` 的精确语义

`Z$o(...)` 的写法已经能把这个字段压实：

```ts
preservedSegment: {
  headUuid: firstKeptMessage.uuid,
  anchorUuid: producerChosenAnchorUuid,
  tailUuid: lastKeptMessage.uuid,
}
```

也就是：

- `headUuid`
  - 被保留段的第一条消息
- `anchorUuid`
  - producer 选择的重连锚点
- `tailUuid`
  - 被保留段的最后一条消息

其中当前已经能明确看到两种写法：

- `session-memory compact`
  - `anchorUuid = summaryMessage.uuid`
- `partial compact`
  - `anchorUuid = boundaryMarker.uuid`

所以它不是“永远指向 summary”的统一字段，而是：

- **给 rewind / transcript relink 用的显式锚点信息**

当前 bundle 里的还原侧也确实按这个结构消费：

- `kWz(...)` 会读取 `compact_boundary.compactMetadata.preservedSegment`
- 按 `tailUuid -> ... -> headUuid` 的 parent 链回走
- 成功后把 `headUuid.parentUuid` 改到 `anchorUuid`
- 再把原本挂在 `anchorUuid` 下的其他节点重挂到 `tailUuid`

因此 `preservedSegment` 不是 UI 注释，而是 transcript 重连协议的一部分。

### 当前几类 compact producer 的结构差异

#### 1. full compact

返回结构是：

- `boundaryMarker = bn6(...)`
- `summaryMessages = [compact summary]`
- `attachments = restored file reads / task status / plan / invoked skills / tool snapshots / hook results`
- `hookResults = SessionStart hook results`
- **没有** `messagesToKeep`

这意味着 full compact 的结果是：

- `boundary + summary + attachments + hooks`
- 没有保留一段“尾部原消息”继续挂在 summary 后面

对应地，当前 full compact 没有走 `Z$o(...)` 这一类 preserved-segment adapter，所以：

- **full compact 默认不带 `preservedSegment`**

#### 2. partial compact

返回结构是：

- `boundaryMarker = Z$o(boundary, anchorUuid, messagesToKeep, preCompactMessages)`
- `summaryMessages = [compact summary]`
- `messagesToKeep = w`
- `attachments = restored file reads / task status / plan / invoked skills / tool snapshots`
- `hookResults = SessionStart hook results`

这里 `w` 是未被 summarize 的保留前缀。  
因此 partial compact 明确是：

- **summary + kept segment 并存**
- **boundary 一定带 `preservedSegment`**

并且这里的 `preservedSegment.anchorUuid` 不是 summary UUID，而是：

- `boundaryMarker.uuid`

并且边界 marker 里会把这些手动 partial compact 语义写入 `compactMetadata`：

- `userContext`
- `messagesSummarized`

#### 3. session-memory / preserved-tail compact

返回结构是：

- `boundaryMarker = Z$o(boundary, summaryMessageUuid, keptTailMessages, preCompactMessages)`
- `summaryMessages = [session-memory summary]`
- `messagesToKeep = keptTailMessages`
- `attachments = [plan reference?]`
- `hookResults = Stop/SessionStart 类 compact hook 结果`

这条路径和 partial compact 一样：

- 会保留尾段消息
- 会补 `preservedSegment`

但这里的 `preservedSegment.anchorUuid` 指向的是：

- `summaryMessages[0].uuid`

但它的边界基型来自 `bn6("auto", preTokens, lastOriginalUuid)`，没有单独写入：

- `userContext`
- `messagesSummarized`

所以 session-memory compact 与 partial compact 不是完全同构，只是都落在同一个 `FOo(...) / SHe(...)` 公共合同里。

### compact 失败后，本轮不会立刻退出

若 `deps.autocompact(...)` 只返回失败状态和 `consecutiveFailures`：

- `Q` 会被更新成带 `consecutiveFailures` 的 tracking 对象。
- 当前 turn 继续往下走，不会因为 autocompact 失败直接终止。

### `compactTracking` / `autoCompactTracking` 的完整语义

这一块现在也能从“状态对象里有个 tracking”收紧到“谁在读、读了做什么”。

#### 真正读取 `Q` 的是 autocompact 依赖

`oRf(...)` 自己除了：

- compact 成功后重置 `Q`
- normal tool round 结束后 `Q.turnCounter++`
- 某些重试分支把 `Q` 原样带入下一轮

并不会解释 `Q`。真正消费它的是 `deps.autocompact(...)` 背后的自动压缩入口。

#### `consecutiveFailures` 是本地 circuit breaker，不是 telemetry 装饰

自动压缩入口一开始就检查：

- 若 `DISABLE_COMPACT` 打开，直接不 compact
- 若 `tracking?.consecutiveFailures >= 3`
  - 直接跳过后续 autocompact 尝试

这说明 `consecutiveFailures` 的语义不是“失败计数留档”，而是：

- **同一条 `oRf(...)` 状态链里的 autocompact 熔断器**

#### `compacted / turnId / turnCounter` 组成的是“compact chain metadata”

当自动压缩真准备 compact 时，会从 `Q` 组装：

```ts
{
  isRecompactionInChain: z?.compacted === true,
  turnsSincePreviousCompact: z?.turnCounter ?? -1,
  previousCompactTurnId: z?.turnId,
  autoCompactThreshold: Bn6(model),
  querySource
}
```

这里三个位的实际含义可以直接写成：

- `compacted`
  - 之前是否已经成功做过一次 autocompact
- `turnId`
  - 上一次 autocompact 链的 UUID
- `turnCounter`
  - 上一次 autocompact 之后，已经成功走过多少个“正常工具轮”

也就是说它不是“总 turn 计数”，而是：

- **post-autocompact turns since last successful autocompact**

#### `turnCounter` 只在一个位置递增

`Q.turnCounter++` 只发生在：

- 本轮存在 `tool_use`
- 工具执行结束
- 没被 `hook_stopped_continuation` 截断
- 即将进入下一正常 turn

因此它**不会**在这些分支增长：

- `max_output_tokens` 续写还原
- `stop_hook_blocking`
- reactive compact retry
- 无工具直接完成

这点很重要，因为它决定 `turnsSincePreviousCompact` 统计的不是“所有循环次数”，而是：

- **compact 后真正完成过的 tool round 数**

#### session-memory compact 与 full auto compact 对这份 metadata 的消费深度并不一样

当前 autocompact 仍能看到两类结构差异：

```text
deps.autocompact(...)
  -> 先试 session-memory / preserved-tail compact
  -> 不成再走 full auto compact
```

其中：

- session-memory / preserved-tail compact
  - 实际只消费 `autoCompactThreshold`
  - 用来检查 session-memory compact 后的 token 数是否仍然超阈值
  - 不读取 `turnId / turnCounter / compacted`
- full auto compact
  - 会把整份 metadata 打进 `tengu_compact`
  - 包括：
    - `isRecompactionInChain`
    - `turnsSincePreviousCompact`
    - `previousCompactTurnId`
    - `autoCompactThreshold`
    - `willRetriggerNextTurn`
  - 但当前没看到它再把这些字段反喂回 compact prompt 或算法分支

所以更稳的结论是：

- `compactTracking` 同时承担：
  - **熔断控制**
  - **compact chain telemetry / observability metadata**
- 但当前没有证据表明：
  - `turnId / turnCounter` 会改变 summarize prompt 内容
  - 或驱动额外还原/UI 分支

#### 哪些分支会保留或清空 `Q`

- autocompact 成功
  - 重置成新的 `{ compacted: true, turnId, turnCounter: 0, consecutiveFailures: 0 }`
- autocompact 失败
  - 保留原链并更新 `consecutiveFailures`
- `max_output_tokens_recovery`
  - 原样带入 `Q`
- `stop_hook_blocking`
  - 原样带入 `Q`
- `next_turn`
  - 原样带入 `Q`
- reactive compact retry
  - 重置为新的 compact tracking，并把 `hasAttemptedReactiveCompact` 标为已尝试
- precomputed compact swap
  - 同样重置为新的 compact tracking，但 `transition.reason` 写成 `precomputed_compact_swap`

因此当前不能再写成“reactive compact 一定清空 autocompact tracking”。在 `2.1.197` 里，成功进入 compact swap 后会重新建立一条 compact tracking 链。

### shared-prefix / fallback 的两段式总结链

当前本地 bundle 下可直接写成：

```text
full / partial compact
     -> try shared-prefix summarize
     -> fallback summarize
session-memory compact
  -> preserved-tail/session-memory summarize path
```

更关键的结论是：

- full / partial compact 才有 shared-prefix 尝试入口。
- session-memory compact 当前不走这条 shared-prefix 路径。
- shared-prefix 只试一次。
- retry 只包在 fallback summarize 外层。

## 当前仍需保守表述的点

- `compactTracking` 的“控制面 + telemetry 面”已经能确认，但 session-memory compact 之外是否还有 bundle 外消费者写入/读取同类 tracking，当前不做过推。

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
