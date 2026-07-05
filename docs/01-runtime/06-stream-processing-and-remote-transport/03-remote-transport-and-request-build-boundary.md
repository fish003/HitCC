# Remote transport 与 request build 最后边界

## 本页用途

- 整理 `sdk-url` / bridge / ingress transport、事件 replay、auth/header 与本地 request build 的最后边界。
- 区分远程传输层、model adapter 和 session/bridge 控制面的职责。

## 相关文件

- [../04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../09-api-lifecycle-and-telemetry.md](../09-api-lifecycle-and-telemetry.md)
- [../../03-ecosystem/02-remote-persistence-and-bridge.md](../../03-ecosystem/02-remote-persistence-and-bridge.md)

## 远程 transport：`sdk-url` / bridge / ingress

这一块现在也能与 provider 层明确分离：

- `sdk-url` 只在 `--print + --input-format=stream-json + --output-format=stream-json` 下启用
- structured input source 会从本地实现切到远端 transport
- transport factory `Ro4(url, headers, sessionId, refreshHeaders)` 至少分三路：
  - `CLAUDE_CODE_USE_CCR_V2=1` -> `Nj6`
  - `ws/wss + CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1` -> `L18`
  - `ws/wss` 默认 -> `y18`

其中：

- `y18` 是纯 WebSocket transport
  - 支持 `X-Last-Request-Id`
  - 支持重放 buffered messages
  - 支持 4003 后 refresh token 再重连
  - 自带 ping/pong、keep_alive、sleep 检测
- `L18` 是 hybrid ingress transport
  - 下行读取 `client_event`
  - 用 `Last-Event-ID` / `from_sequence_num` 续传
  - 上行通过独立 POST 写入
- `Nj6` 是 CCR v2 路径
  - 会把 `ws/wss` URL 改写为 `http/https` 的 `/worker/events/stream`

这说明：

- provider/gateway 与 remote ingress transport 是两套正交层
- `sdk-url` 不等于“直接把模型请求发到另一个 provider”
- 它更像“把 CLI 主循环绑定到远端 session ingress”

### transport factory 的精确分流

`Ro4(...)` 现在其实可以写到 URL 改写级别：

- `CLAUDE_CODE_USE_CCR_V2=1`
  - 不再保留 `ws/wss`
  - 会把：
    - `wss:` -> `https:`
    - `ws:` -> `http:`
  - 再把 pathname 改成：
    - `.../worker/events/stream`
  - 最终产物是 `new Nj6(url, headers, sessionId, refreshHeaders)`
- 否则若协议仍是 `ws/wss`
  - `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1` -> `new L18(...)`
  - 否则 -> `new y18(...)`

因此 CCR v2 不是“在 WebSocket 上切协议”，而是一开始就切成 **HTTP/SSE + worker subpaths**。

### transport 协议粒度还能再收紧

这一层现在已经不只是“像 event pipe”，而是可以直接写到收发对象形态：

- `sU8` 会把 transport 包成一个 `PassThrough` 输入流
  - `transport.setOnData(...)` 收到的每段字符串，会直接写回本地 `inputStream`
  - 因此 headless `--print --sdk-url --input-format=stream-json` 看到的仍是同一套 `stream-json` 事件流
- `y18.write(A)`
  - 直接把单个事件对象做 `JSON.stringify(A) + "\\n"` 后发到 WebSocket
  - 若对象自带 `uuid`，只会顺手进入本地 replay buffer
  - 没看到任何把事件再提升成 `messages/system/userContext/systemContext` 级请求对象的包装
- `L18.write(A)`
  - 只会对 `stream_event` 做短暂缓冲
  - 真正上行时发的是 `POST { events: [...] }`
  - 也就是把同一批本地事件对象整体送进 ingress，而不是重新拼 prompt
- `Nj6`
  - 下行 SSE 只接受 `event: client_event`
  - 解析后只取 `payload`
  - 若 `payload.type` 存在，就原样 `JSON.stringify(payload) + "\\n"` 回灌给本地
  - 上行 POST 也只是把当前事件对象原样发给 worker/session 侧 endpoint
- `R18`
  - CCR v2 上行拆成：
    - `/worker/events`
    - `/worker/internal-events`
    - `/worker/events/delivery`
    - `/worker`
  - 承载的是 client event、internal event、delivery ack、worker state/metadata
  - 不是 prompt assembly 请求体

这说明：

- `sdk-url / remote-control / CCR v2` 这组远端接入，当前本地可见职责都是 **session event plane**
- 这里传的是：
  - `stream-json` 事件
  - control request / response
  - keep_alive
  - worker metadata / delivery 状态
- 这里没看到传：
  - 重新组装后的 `messages/system`
  - 新的 `userContext/systemContext`
  - 额外 compat / verification prompt 片段

### 三个 transport 的本地状态机还能继续写实

#### `y18`：WebSocket close/reconnect 规则

- 默认 autoReconnect 打开
- permanent close code 集合当前已能直接写死：
  - `1002`
  - `4001`
  - `4003`
- 但 `4003` 有一个明确例外：
  - 若提供了 `refreshHeaders()`
  - 且刷新后的 `Authorization` 与旧值不同
  - 就不会当成 permanent close
  - 而是更新 headers 后继续 reconnect
- 建连时若已有 `lastSentId`
  - 会补 `X-Last-Request-Id`
  - open 后按对端确认的 last id 重放本地 buffered messages
- 连接期间还会主动发两类保活：
  - websocket ping/pong
  - 周期性 `keep_alive` data frame

#### `L18`：hybrid ingress 的上行批处理参数

- `stream_event` 不会立刻 POST
  - 先放进 `streamEventBuffer`
  - `100ms` 后批量 flush
- uploader 参数当前可直接写成：
  - `maxBatchSize: 500`
  - `maxQueueSize: 100000`
  - `baseDelayMs: 500`
  - `maxDelayMs: 8000`
  - `jitterMs: 1000`
- 真正 POST body 固定是：

```json
{ "events": [ ... ] }
```

- 单次 POST timeout 是 `15000ms`
- `4xx` 且不是 `429`
  - 视为 permanent client error
  - 丢弃当前 batch
- `429`、`5xx` 与网络错误
  - 视为 retryable
  - 交给 uploader 指数退避
- `close()` 时会：
  - 先尝试 flush
  - 最多等 `3000ms`
  - 再真正关闭 uploader 和基类 transport

#### `L18` 下行续传字段

Hybrid/SSE 下行建连时，当前已能直接看到：

- 若 `lastSequenceNum > 0`
  - query string 补 `from_sequence_num=<n>`
  - header 也补 `Last-Event-ID: <n>`
- 固定补：
  - `Accept: text/event-stream`
  - `anthropic-version: 2023-06-01`
- 若 auth 走 `Cookie`
  - 会显式删除 `Authorization`

这说明它不是只靠本地 buffer，而是显式支持 **服务端 sequence-based replay**。

#### `Nj6` / `R18`：CCR v2 的 worker 面

- `R18` 初始化时会建立四个独立 uploader：
  - `eventUploader` -> `POST /worker/events`
  - `internalEventUploader` -> `POST /worker/internal-events`
  - `deliveryUploader` -> `POST /worker/events/delivery`
  - `workerState` -> `PUT /worker`
- 这些请求都固定带：
  - `worker_epoch`
  - `anthropic-version: 2023-06-01`
- 初始化 `PUT /worker` 的首个状态是：
  - `worker_status: "idle"`
  - `external_metadata.pending_action: null`
- `409` 会被解释成 worker epoch mismatch
- `401/403` 连续达到阈值，会直接走 epoch mismatch / 退出，而不是无限重试

因此若服务端还会额外追加 `verification / context / systemContext`，更像是：

- 模型 message API 收到本地 payload 之后的黑箱后处理
- 或 bundle 外远端 runtime 的附加能力

而不是当前本地 `sdk-url / remote-control` transport 自己又做了一次 prompt 重写。

## 本地 request build 的最后边界

围绕“远端/服务端是否额外注入 context / compat / verification”，当前本地 bundle 还能再收紧一步。

### 本地可见的主模型 payload producer

本页已归档的主模型请求里，直接把 prompt/request body 交给 `beta.messages.create(...)` 的本地入口是：

- `Hpc(...)`
  - streaming 主路径
- `_pc(...)`
  - non-streaming fallback

其中主路径 `Hpc(...)` 的最后落体已经是：

```text
payload = {
  model: Gu(model),
  messages: Glm(Ipc(messagesForAPI, ...), ...),
  system: Wlm(systemSections, ...),
  tools,
  tool_choice,
  betas,
  metadata: XLe(),
  max_tokens,
  thinking,
  output_config,
  context_management,
  speed
}
-> client.beta.messages.create(payload, ...)
```

### builder callback `wt(qe)` 的条件字段表

`Hpc(...)` 里的真正 payload builder 不是平铺常量，而是一组条件装配：

- `model`
  - body 里写 `Gu(model)`，用于把候选模型名映射到 provider 接受的模型名
- `messages`
  - `Glm(Ipc(messagesForAPI, serverFallbackReplayEligible), promptCachingEnabled, cacheTtl, skipCacheWrite, forkPointUuid)`
- `system`
  - `Wlm(systemSections, enablePromptCaching, { skipGlobalCacheForSystemPrompt, cacheTtl })`
- `tools`
  - `Hci(toolSchemas + extraToolSchemas, model)`
  - `advisor` 命中时还会额外 push `advisor_20260301`
- `betas`
  - 来自模型默认 beta、tool-search beta、prompt cache beta、fast beta、auto-mode beta、cache-edit beta 等组合
- `metadata`
  - 固定来自 `XLe()`
  - `XLe()` 把 `CLAUDE_CODE_EXTRA_METADATA`、`device_id`、`account_uuid`、`session_id` 序列化进 `metadata.user_id`
- `max_tokens`
  - 优先级：
    - retry context 的 `maxTokensOverride`
    - request option 的 `maxOutputTokensOverride`
    - `Kut(model)` 按模型能力和 `CLAUDE_CODE_MAX_OUTPUT_TOKENS` 得出的默认值
- `thinking`
  - 仅当 thinking 未禁用且模型支持时才存在
  - adaptive thinking 与 budget-based thinking是两条分支
- `temperature`
  - 只有 **不启用 thinking** 时才会显式写入
- `context_management`
  - 只有模型/能力组合满足，且 beta 集合包含对应开关时才会写出
- `output_config`
  - 由 effort / taskBudget / outputFormat 等组合生成
- `speed`
  - 只有 fast mode 实际生效时才会写成 `"fast"`
- `fallbacks / fallback_credit_token`
  - server-side refusal fallback 参数由 `xRl(...)` 条件写入
  - `fallback_credit_token` 只在已拿到可用 credit code 且后端/model 匹配时随下一次请求体发送；被发送后本地会立即清空待发 token，避免重复使用
- `diagnostics.previous_message_id`
  - 只在 prompt cache diagnosis beta 生效、agentic query、first-party/Anthropic AWS 默认 host 条件满足时写入

因此当前更稳的说法不是“payload 有这些字段”，而是：

- `Hpc(...)` 先统一求出一份 **条件性 builder**
- streaming 与 non-streaming fallback 都复用同一份 builder
- streaming 只在最后追加 `stream: true` 和当次 header
- non-streaming fallback 会复用 builder 结果，再通过 `Vlm(...)` 把 `max_tokens` 压到同步请求上限

`_pc(...)` 自己并不重新拼 prompt。  
它接收的是 `Hpc(...)` 里传入的 builder callback，也就是同一套 `messages/system/tools` 构造结果。同步路径只额外做：

- `Vlm(...)` 限制 `max_tokens`
- `model: Gu(model)` 保险映射
- `Apc(...)` 生成当次 headers

因此 non-streaming fallback 不是第二套 prompt assembly。

### request options、headers、telemetry metadata 的分层

当前主模型请求面可以按四层拆开：

| 层 | 典型字段 | 生成位置 | 消费位置 | 边界 |
| --- | --- | --- | --- | --- |
| prompt payload | `system / messages / tools / tool_choice` | `Flm(...)`、`Wlm(...)`、`Glm(...)`、`Hci(...)` | `beta.messages.create(...)` body | 这是模型可见主体；`sdk-url`/bridge 不在本层二次改写 |
| request options body | `model / betas / max_tokens / thinking / output_config / context_management / speed / fallbacks / fallback_credit_token / diagnostics` | `wt(qe)` | `beta.messages.create(...)` body | 受 provider/model/feature flag/retry/fallback 影响；不是 transcript 内容 |
| API call options headers | `x-client-request-id / traceparent / anthropic-usage-limit / anthropic-dispatch-id` | `Apc(...)` 与 `Hpc(...)` 发请求前分支 | SDK request options 的 `headers`；错误/success telemetry 回读部分字段 | 不进入 `payload.messages/system`；`anthropic-dispatch-id` 失败时可 strip/retry |
| telemetry metadata | `previousRequestId / queryTracking / querySource / requestId / clientRequestId / firstAttemptRequestId / gateway / attemptStartTimes` | `Dlm(...)`、`oRf(...)`、`Apc(...)`、response/error headers | `NDl(...)`、`KOo(...)`、`UDl(...)`、`F7t(...)`、`kmo(...)`、Perfetto | 用来关联日志、trace 和 retry/fallback，不是服务端风控规则的本地证明 |
| request body metadata | `metadata.user_id` 内的 `device_id / account_uuid / session_id / CLAUDE_CODE_EXTRA_METADATA` | `XLe()` | API body 的 `metadata` 字段 | 字段名是 `user_id`，值是 JSON 字符串；remote 只影响 `account_uuid` 来源 |

几条容易混淆的关联规则：

- `previousRequestId` 来自 `Dlm(messages)`，只向后找最近一条带 `requestId` 的 assistant transcript entry；它不等同于当前响应的 `request_id`。
- `requestId` 是服务端响应或 SDK error 上的 id；streaming 成功后来自 `withResponse().request_id`，错误路径可从 `requestID / error.request_id` 回补。
- `clientRequestId` 由 `Apc(...)` 在 first-party 或 Anthropic AWS 默认 host 路径生成，随 `x-client-request-id` header 发出；streaming -> non-streaming fallback 成功时 success telemetry 会避免把旧 streaming 的 client id 当作最终同步请求 id。
- `traceparent` 由 active LLM span 注入；默认只在 first-party / Anthropic AWS 默认 host 条件下传播，`CLAUDE_CODE_PROPAGATE_TRACEPARENT` 可显式放宽。
- `queryTracking { chainId, depth }` 由上层 query loop 生成，进入 `NDl / KOo / UDl`；fallback、refusal 和 prompt-cache diagnosis 只消费这组链路信息，不重新生成 prompt。
- `firstAttemptRequestId` 只在 streaming 起始请求和最终成功请求 id 不一致时上报，主要覆盖 streaming 404 或中途转 non-streaming 的关联。

直接证据：

- `XLe()` metadata：`package/preprocessed/cli.extracted.bundle.pretty.js:713876-713898`
- `Apc(...)` header 生成：`package/preprocessed/cli.extracted.bundle.pretty.js:714082-714098`
- `_pc(...)` 复用 builder：`package/preprocessed/cli.extracted.bundle.pretty.js:714100-714160`
- `Hpc(...)` request builder 与 streaming dispatch：`package/preprocessed/cli.extracted.bundle.pretty.js:714372-715230`
- `NDl / KOo / UDl` telemetry consumer：`package/preprocessed/cli.extracted.bundle.pretty.js:560547-561050`

### `sdk-url / remote-control` 本地可见代码里只换 transport，不改 prompt

bridge/session manager 侧做的事情当前可直接写成：

- 用 `--print --sdk-url --input-format=stream-json --output-format=stream-json` 启动子进程
- 注入 `CLAUDE_CODE_SESSION_ACCESS_TOKEN`
- 注入 `CLAUDE_CODE_POST_FOR_SESSION_INGRESS_V2=1`
- 视情况再加 `CLAUDE_CODE_USE_CCR_V2=1`

然后只做：

- stream-json I/O
- control request / permission request 转发
- activity / transcript 采集

当前没看到它在子进程外再修改：

- `messages`
- `system`
- `userContext`
- `systemContext`
- compat / verification 指令

### `verification_agent` 在本地仍只有识别位，没有发起点

当前本地 bundle 中：

- `verification_agent` 只出现在 `Hpc(...)` 的 `isAgenticQuery` 判定里
- 没有发现任何本地 `querySource: "verification_agent"` 发起点
- 可见 subagent 的正常 querySource 生成规则是：
  - built-in -> `agent:builtin:<agentType>`
  - custom -> `agent:custom`
  - 因此即便本地真的存在一个 `subagent_type: "verification"`，按当前可见通路它也不该自然落成 `verification_agent`
- 另外还有一个重要矛盾：
  - bundle 文本里会提示“spawn verification agent (subagent_type=\"verification\")”
  - 也残留了一整段 verifier system prompt
  - 但当前可见 built-in agent 注册表没有把 `verification` 注册进去

因此更稳的本地结论是：

- 本地 bundle 没看到 `Hpc / _pc` 之后还有第二次 request-level context 注入
- `sdk-url / remote-control` 在本地可见范围内只是 transport/ingress 层
- `verification_agent` 更像 **服务端/动态注入/未完全接线残留**，而不是当前本地 bundle 可直接跑通的一条 querySource
- 若 `context / compat / verification` 还有额外注入，更可能发生在服务端黑箱，不在本地 bundle 内

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
