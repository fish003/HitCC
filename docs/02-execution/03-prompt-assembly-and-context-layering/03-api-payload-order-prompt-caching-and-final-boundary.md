# API payload 顺序、prompt caching 与最终边界

## 本页用途

- 单独承接最终发给 `beta.messages.create(...)` 的 `messages/system` 对象级顺序。
- 把 `Hpc / Flm / Glm / Wlm`、prompt caching、system prompt 分块缓存，以及最终 `payload.system / payload.messages` 的边界固定到同一页。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../05-attachments-and-context-modifiers.md](../05-attachments-and-context-modifiers.md)
- [../06-context-runtime-and-tool-use-context.md](../06-context-runtime-and-tool-use-context.md)
- [../../01-runtime/06-stream-processing-and-remote-transport.md](../../01-runtime/06-stream-processing-and-remote-transport.md)

## API payload 顺序与 prompt caching

### 最终 API request 的 `messages[]` 对象级顺序

这一段现在已经可以从“内容顺序”推进到“最终发给 `beta.messages.create(...)` 的 message object 顺序”。

`Hpc(...)` 的 request build 主段现在可以压成：

```text
{ messagesPreNormalize, messagesForAPI, midConvFallback } = Flm(...)
systemBlocks = Wlm(systemSections, enablePromptCaching, ...)
payload = {
  messages: Glm(Ipc(messagesForAPI, ...), ...),
  system: systemBlocks,
  tools: Hci(toolSchemas, model),
  ...
}
-> client.beta.messages.create(payload)
```

这里有四个已经可以直接写死的结论。

#### 1. `Glm(...)` 不再改 message 相对顺序

`Glm(...)` 只做两类事：

- 把归一化后的 `user/assistant` transcript message 映射成 API `role/content`
- 注入 prompt cache 相关 `cache_control`

它不会再重排 `messages[]` 的相对先后。  
因此真正决定对象顺序的是 `Flm(...)` 内部的 `Ak(...)` 与后续 normalize，不是 `Glm(...)`。

这里还能继续收紧到 API payload 级细节。

#### `Glm(...)` 的 cache breakpoint 选择

当前本地直接看到：

```text
lastNonApiSystemIndex =
  从末尾跳过 api_system message 后的最后一个普通 message

if skipCacheWrite:
  breakpoint = last non-api-system message before it
else:
  breakpoint = lastNonApiSystemIndex
```

再逐条把 transcript message 映射成 API message，并把“是否是 breakpoint message”作为布尔位传进去。

因此更稳的语义不是“所有消息都可能带 cache_control”，而是：

- **默认只会把最后一个非 `api_system` message 当作写 cache 的候选**
- `skipCacheWrite: true` 时，会把 breakpoint 左移到上一个非 `api_system` message
- 这正好对应 compact / side-question 这类“想读旧 cache，但不想污染新 cache”的 helper 路径

#### fork point 可以额外钉住一个 breakpoint

当前 `Glm(...)` 还会看 `TEt()` 与 `forkPointUuid`：

- 如果传入 `forkPointUuid`，且对应 message 位于当前 breakpoint 之前或等于它，可以额外标成 cache breakpoint
- 如果没有传 `forkPointUuid` 且本轮不是 `skipCacheWrite`，也可能把倒数第二个非 `api_system` message 标成额外 breakpoint
- 若 fork point 与 `skipCacheWrite` 正好撞到当前 breakpoint，且 `tLl()` 条件成立，会再左移一格

因此当前可见语义不是“只有一处 cache marker”，而是：

- 主 breakpoint 仍由末尾非 `api_system` message 决定
- fork point 机制可把历史消息再钉成一个 cache reuse 边界
- telemetry 会记录 `forkPointPinned` 与 `markerCount`

#### 当前 `Glm(...)` 不再可见 `cache_reference / cache_edits` 注入协议

旧文档里的 `cache_reference / cache_edits / AEq / qEq / Gu1` 这条判断不再适用于当前 `2.1.197` 可见实现。

当前 `Glm(...)` 的形状已经收窄为：

- 跳过尾部 `api_system`
- 选择一个或多个 cache breakpoint
- 分发到 user / assistant / api_system message mapper
- 不在本层插入 `cache_reference`
- 不在本层消费 `cache_edits`

因此这页不再把 `cache_edits` 写作当前本地活协议。若其他 bundle 外路径或服务端协议仍有类似能力，当前本地 bundle 不能证明。

#### `Wlm(...)`：system prompt 也不是一整块 cache，而是按 scope 拆块

`Wlm(...)` 不是直接把 `Ec(systemSections)` 包成一个 text block。  
它会先经 `W9o(...)` 把 system prompt sections 拆成：

- `cacheScope: null`
- `cacheScope: "org"`
- `cacheScope: "global"`

再逐块映射成 API `system` text blocks，并只给 `cacheScope !== null` 的块加：

- `cache_control: UF({ scope, querySource })`

当前本地还直接看到两条关键分支：

1. 找到动态边界 `Jw6`
   - 边界前静态段可打 `global`
   - 边界后动态段不打全局 cache
2. `skipGlobalCacheForSystemPrompt`
   - 直接退回不用 `global` scope 的拆块策略

其中 `skipGlobalCacheForSystemPrompt` 现在还能再写得更硬。

当前本地不是任意调用方手工决定它，而是 `Hpc(...)` 在 request build 阶段按条件计算：

- 先要求全局 system prompt cache 模式已开启
- 再要求本轮 `tools` 数组里存在**活的 MCP tool schema**
- 且这些 MCP tools 不是仅 deferred / pending 的占位项

命中后会把 `skipGlobalCacheForSystemPrompt = true` 传给 `Wlm(...)`。  
此时 `W9o(...)` 会改走：

- billing header -> `cacheScope: null`
- org header -> `cacheScope: "org"`
- 其余 system text -> `cacheScope: "org"`

而不是：

- 边界前静态段 -> `global`
- 边界后动态段 -> `null`

因此这里更稳的判断应是：

- `skipGlobalCacheForSystemPrompt` 是**request-level 的全局降级开关**
- 它不是 compact 专属逻辑
- 主线程、subagent、compact fallback、其他 non-main-thread query 只要带上活的 MCP tool schema，都可能触发这条降级
- 一旦触发，即使 prompt 内存在 `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__`，也不会再产出 `global` scope 那段静态前缀

因此 system prompt 的本地缓存语义现在可以更精确地写成：

- **不是整份 system prompt 一个 cache breakpoint**
- **而是 billing / org / global / dynamic 段拆块后分别决定是否可缓存**
- `__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__` 的真实作用，就是把“可长期复用的静态前缀”和“每轮易变后缀”切开

#### TTL 与 cache break diagnosis

当前本地还可见两层 TTL / diagnosis 语义：

- bundle 内嵌提示明确把默认 `cache_control` 写作 5-minute TTL，并有 `extended_cache_ttl` / `extended-cache-ttl-2025-04-11` beta 常量；这些只能证明客户端知道 5 分钟与扩展 TTL 两档语义，不能推导账户是否启用或价格如何计算。
- cache break 诊断会把 `possible 5min TTL expiry`、`possible 1h TTL expiry`、`likely server-side (prompt unchanged, <5min gap)` 区分开来。
- 诊断缓存会记录 `model / fastMode / globalCacheStrategy / betas / autoModeActive / isUsingOverage / is1hCacheTTL / queryDepth / cacheDiagnosis / effortValue / extraBodyHash` 等字段。
- cache 下降原因会细分为 `system prompt changed`、`tools changed`、`fast mode toggled`、`cache_control changed (scope or TTL)`、`betas changed`、`auto mode toggled`、`effort changed`、`message history mutated` 等。

证据点是 `package/preprocessed/cli.extracted.bundle.pretty.js:122646`、`272384`、`285090-285340`、`802463`。因此 prompt cache 文档可以写 5 分钟 / 1 小时 TTL diagnosis 的本地判断口径；服务端实际 cache-hit 策略仍是黑箱，不能反推出所有 cache miss 的远端原因。

#### 2. 当前轮进入 `Ak(...)` 前的尾部形状

普通当前轮先由 `current-turn input compiler` 生成：

```text
[current-turn user message, ...attachments]
```

如果有 `UserPromptSubmit` hook 产出的 `hook_additional_context / hook_success / hook_non_blocking_error` attachment，它们会在 `AU8(...)` 里继续被追加到这个尾部数组后面。

因此 `Ak(...)` 收到的“当前轮尾部”更准确地是：

```text
current-turn user
-> JPl(...) attachments
-> UserPromptSubmit hook attachments
```

#### 3. `Ak(...) attachment placement` 先把尾随 attachment 左移到上一个边界

`Ak(...) attachment placement` 会从后往前扫：

- 遇到 `attachment` 先暂存
- 遇到 `assistant`
- 或遇到 `user` 且其首个 block 是 `tool_result`

就把暂存 attachment 整批插到这个边界之前。

因此 attachment 的最终锚点不是“当前轮 user 后面”，而是：

- 优先贴到最近一个 `assistant` 之后
- 若没有，就贴到最近一个 `tool_result` user 之后
- 再没有，才留在最前部剩余位置

#### 4. `Ak(...)` 会把相邻 user message 折叠成更少的对象

`Ak(...)` 之后的对象级行为现在也已明确：

- `system` transcript entry 会先转成 user meta message
- `attachment` 会经 `dt1(...)` 展开成一个或多个 user meta message
- 若前一个已输出对象是 `user`，无论是普通 user 还是 meta user，都会立刻用 `Qcm(...)/rdr(...)` 合并
- 末尾还会再经过一次 `d0z(...)`，把所有剩余相邻 user message 继续折叠
- 合并后的单个 user message 内部，`cb4(...)` 会把 `tool_result` block 放到前面，再接 text/meta block
- 若两端边界都是 text block，`c0z(...)` 会在中间补一个换行

所以对“普通当前轮 + attachment”来说，更稳的最终近似已经不是“若干独立 message”，而是：

```text
messages[]
  = [
      otr(...) 生成的前置 user meta message,
      ...历史 assistant/user message,
      previous assistant 或 previous tool_result-user,
      merged user message {
        tool_result blocks first (if any),
        then attachment-derived meta blocks,
        current-turn user text last
      }
    ]
```

也就是说，当前轮 attachment 在最终 API payload 里通常不会保留成独立 `message object`，而会并进它左侧边界之后的那个 user message。

### `system` 与 `messages` 的最终边界

因此主线程最终 request 现在可以更精确地写成：

```text
payload.system
  = Wlm(
      Ec(
        tTn(...) billing/header section
        -> $Rn(...) session metadata section
        -> systemPrompt selection / default sections
        -> CDl(..., systemContext)
      )
    )

payload.messages
  = Glm(
      [
        `otr(userContext)` 生成的前置 user meta,
        ...Ak(...) 归一化后的 transcript user/assistant objects
      ]
    )
```

在当前本地 bundle 可见代码里：

- `systemContext` 默认仍只看到 `{ gitStatus?, perforceMode? }` 和显式 override 透传
- `userContext` 默认仍只看到 `{ claudeMd?, userEmail?, attachedProject?, currentDate }` 和显式 override 透传
- 当前轮 attachment / hook additional context / plan/skill meta 仍全部走 `messages[]`
- 没看到第二套“把这些内容直接改写进 `system`”的本地路径

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
