# 请求 telemetry、refusal 与 stream status

## 本页用途

- 整理 `NDl / KOo / UDl` request lifecycle telemetry、refusal/model block 入口与 `stream_request_start` 来源。
- 保留本模块剩余状态和与 telemetry 专题的边界。

## 相关文件

- [../04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../09-api-lifecycle-and-telemetry.md](../09-api-lifecycle-and-telemetry.md)
- [../../03-ecosystem/02-remote-persistence-and-bridge.md](../../03-ecosystem/02-remote-persistence-and-bridge.md)

## 请求生命周期与 telemetry 三件套：`NDl / KOo / UDl`

这一组现在已经可以从“知道名字”推进到“闭环结构基本钉死”。

当前 `2.1.197` 的混淆名已经和旧文档里的语义名不完全一致。下面仍沿用“request start / error / success”三段语义，但对应当前 bundle 的关键函数应按这个映射复核：

- request start：`NDl(...)`，发 `tengu_api_query`
- final error：`KOo(...)`，发 `tengu_api_error`
- final success：`UDl(...) -> _Rf(...)`，发 `tengu_api_success`
- refusal：`F7t(...)`，发 `api_refusal`

更稳的主链应写成：

```text
oRf(...)
  -> 生成 queryTracking { chainId, depth }
  -> Hpc(...)
    -> previousRequestId = Dlm(messages)
    -> llmSpan = YLa(model, agentContext, requestDetail, normalizedMessages, fastMode)
    -> NDl(...)                         // request start
    -> Kur(clientFactory, requestFn, retryContext)
         -> attemptStartTimes.push(Date.now())
         -> beta.messages.create(...)
         -> requestId = response.request_id
    -> stream success / non-streaming fallback / final error
    -> KOo(...)                         // request error
    -> UDl(...)                         // request success
```

这里最关键的结构点有三个：

- `NDl(...)` 只负责“请求发出前”的 telemetry，不带 `requestId`
- `KOo(...)` 只在最终失败路径调用，不会在每次可还原重试时都打一次
- `UDl(...)` 只在最终成功后调用一次，并把 usage/cost/duration 闭合掉

### `NDl(...)`：真正的 request start telemetry

`NDl(...)` 在 `Hpc(...)` 里、真正进入 `Kur(...)/beta.messages.create(...)` 之前触发。  
它发的是 `tengu_api_query`，当前已能确认的字段包括：

- `model`
- `messagesLength`
- `temperature`
- `betas`
- `permissionMode`
- `querySource`
- `queryTracking`
- `thinkingType`
- `effortValue`
- `fastMode`
- `previousRequestId`
- `provider`
- `buildAgeMins`
- `ANTHROPIC_BASE_URL / ANTHROPIC_MODEL / ANTHROPIC_SMALL_FAST_MODEL` 这类环境侧上下文

其中 `ANTHROPIC_BASE_URL` 不是直接裸写任意自定义 host：

- `Ehe(...)` 会先去掉 username / password / search / hash 等 URL 成分
- `Tfn(...)` 判定为官方 Anthropic / Claude host 时保留规范化后的 URL
- 非官方 host 会走 `Pd(...)`，当前是 `sha256` 后取前 12 位

所以 request start telemetry 能记录“是否配置了自定义 base URL”及其稳定指纹，但当前本地证据下不会把任意第三方完整 URL 明文塞进 `tengu_api_query`。

这说明：

- start telemetry 记录的是“这次准备怎么发”，不是“服务端实际返回了什么”
  - 因为此时还没有 response，所以 `requestId` 不可能出现在 `NDl(...)`
- `permissionMode` 来自 `getToolPermissionContext()` 的解析结果，不是单纯抄 CLI 原始参数

### `Kur(...)`：重试容器，也是 duration 语义的关键前提

`Kur(...)` 不是 telemetry helper，但它决定了后面 `KOo / UDl` 的时间语义。

当前更稳的理解是：

- `o = Date.now()`：整次 request 生命周期起点，包含重试与 fallback
- `w6 = Date.now()`：每次 attempt 真正发请求前都会重置一次
- `K6.push(w6)`：记录每次 attempt 的开始时间，供 tracing / perfetto 使用
- `be`：当前 attempt 编号，由 `Kur(...)` 回调写回
- `Ke`：当前 attempt 的 `fastMode` 实际状态，也由 `Kur(...)` 回写

因此：

- `durationMs` 指向“最后一次 attempt 自己花了多久”
- `durationMsIncludingRetries` 指向“从第一次进入 request 到最终结束总共多久”
- `fastMode` 在 start 与 end/error 两侧不一定完全相同
  - `NDl(...)` 记录的是初始请求意图
  - `KOo(...)` / `UDl(...)` 记录的是 `Kur(...)` 调整后的实际状态

这点很重要，因为 `Kur(...)` 会在 fast mode 不可用、429/529、统一 overage 等条件下把 `fastMode` 关掉再继续尝试。

### `KOo(...)`：最终失败路径的 telemetry

`KOo(...)` 的调用点已经能收束成两类：

- streaming 路径失败，且 fallback 也失败
- 非 fallback 的最终请求失败

也就是说：

- 普通可还原重试不会直接走 `KOo(...)`
- 只有“整次 request 最终失败”才会落这里

它至少会做四件事：

1. 归一化错误信息  
   - `Fo_(error)`：抽错误字符串
   - `fTq(error)`：归类 error type
   - `QT6(error)`：补连接类错误细节

2. 补 request 关联字段  
   - `requestId`
   - `clientRequestId`
   - `previousRequestId`
   - `queryTracking`
   - `querySource`

3. 识别 gateway  
   - `$Dl(...)` 会根据响应 header 前缀识别 `litellm / helicone / portkey / cloudflare-ai-gateway / kong / braintrust`
   - 同一 helper 也会用 `ANTHROPIC_BASE_URL` 的 host suffix 识别 `databricks`

4. 同时写三套 telemetry/tracing  
   - `d("tengu_api_error", ...)`
   - `GO("api_error", ...)`
   - `JS1(llmSpan, { success: false, ... })`

这里几个字段现在可以写得更精确：

- `durationMs`
  - 来自 `Date.now() - w6`
  - 表示最后一次 attempt 的耗时
- `durationMsIncludingRetries`
  - 来自 `Date.now() - o`
  - 表示整次 request 从第一次发起到最终失败的总耗时
- `requestId`
  - 优先取当前已保存的 `r`
  - 否则从异常对象的 `requestID / error.request_id` 回补
- `clientRequestId`
  - 来自 `x-client-request-id`
  - 主要用于把客户端日志和服务端日志对齐
- `didFallBackToNonStreaming`
  - 只表示这次 request 是否中途切到 non-streaming，不等于 fallback 一定成功

另外还能确认一个行为细节：

- 即使是用户 abort，最终也仍会先走 `KOo(...)`，然后再结束 request

### `UDl(...)`：最终成功路径的 telemetry

`UDl(...)` 在 `Hpc(...)` 的成功尾部触发。  
这里已经能确认它不是单纯“打个 success 日志”，而是 request completion 的总收口点。

它会做的事包括：

1. 计算成功态 duration  
   - `S = Date.now() - start`
   - `g = Date.now() - startIncludingRetries`

2. 回写全局 API 计时  
   - `Gd8(g, S)`
   - 第一个参数进入 `totalAPIDuration`
   - 第二个参数进入 `totalAPIDurationWithoutRetries`

3. 发 success telemetry  
   - `Qo_(...)` -> `tengu_api_success`
   - `GO("api_request", ...)`
   - `JS1(llmSpan, { success: true, ... })`

4. 汇总输出侧结构信息  
   - `textContentLength`
   - `thinkingContentLength`
   - `toolUseContentLengths`

这意味着 `UDl(...)` 实际上同时闭合了：

- request 成功日志
- usage/cost 统计
- perfetto / llm span 完成事件
- 全局 session 级 API duration 聚合

### 关键字段语义：当前可直接写死的版本

- `requestId`
  - 成功 streaming 路径取 `withResponse().request_id`
  - 错误路径可从异常对象回补
  - 若 streaming 建连阶段先 404，再走 non-streaming fallback 成功，最终 success telemetry 里的 `requestId` 可能仍为 `null`

- `clientRequestId`
  - 只在 first-party 路径显式生成
  - 通过 `x-client-request-id` 发出

- `gateway`
  - 错误路径由 `$Dl(...)` 从响应 header 或 base URL host 推出
  - 目前是 telemetry 分类字段，不是请求 payload 里的隐藏 prompt 标记
  - 当前主要在错误路径上报和日志提示里使用

- `durationMs`
  - 只看最终 attempt
  - 不包含之前失败重试的等待与重建成本

- `durationMsIncludingRetries`
  - 覆盖整次 request 生命周期
  - 包含重试等待、client 重建、stream -> non-streaming fallback 等前置损耗

- `ttftMs`
  - streaming 路径在第一次 `message_start` 时写入
  - non-streaming fallback 成功时通常没有这个值

- `costUSD`
  - 由 `Le(model, usage)` 定价，再经 `rv6(...)` 计入全局 usage
  - 会递归纳入 advisor 子 usage

- `queryTracking`
  - `oRf(...)` 为每次 query 生成 `{ chainId, depth }`
  - 同一条链上的后续 query 复用 `chainId`
  - 深度随嵌套/继续调用递增

- `previousRequestId`
  - 由 `Dlm(messages)` 从当前 transcript 里向后找最近一条带 `requestId` 的 assistant message
  - 用来把相邻 request 串成链，而不是只靠 query depth

- `permissionMode`
  - 来自当次 `getToolPermissionContext()` 的结果
  - 因此它记录的是 request 发出时的实际权限模式

- `fastMode`
  - start telemetry 记录初始意图
  - success/error telemetry 记录重试器修正后的实际状态

- `requestSetupMs`
  - 来自 `w6 - o`
  - 更准确地说是“从整次 request 进入 `Hpc(...)` 到最终成功 attempt 发出前”的累计 setup 时间
  - 不是纯粹的序列化耗时

- `attemptStartTimes`
  - 每次 attempt 发出前 push 一次时间戳
  - 当前主要用于 `JS1(...)` 对 perfetto/tracing 的 retry 片段还原

### request / header / trace 字段闭环

这一组字段现在可以按 producer 和 consumer 分清：

| 字段 | producer | 发送面 | consumer / 记录面 | 语义边界 |
| --- | --- | --- | --- | --- |
| `previousRequestId` | `Dlm(messages)` | 不进 request header；只进 telemetry 参数 | `NDl / KOo / UDl` | transcript 相邻请求链；不是当前服务端响应 id |
| `requestId` | `withResponse().request_id` 或 SDK error `requestID / error.request_id` | 服务端响应/错误对象 | `KOo / UDl / F7t`、assistant transcript `requestId`、fallback UI events | 当前 API call 的服务端 id；可能在 streaming 404 后和最终 non-streaming success id 不同 |
| `clientRequestId` | `Apc(llmSpan, attempt)` | `x-client-request-id` header | `KOo / UDl / kmo`、错误日志 | 客户端到服务端日志对齐 id；默认只在 first-party 或 Anthropic AWS 默认 host 路径生成 |
| `traceparent` | `Apc(...) -> Lmo(llmSpan)` | `traceparent` header | OTEL span link / downstream trace 采集 | 默认跟随 first-party / Anthropic AWS 默认 host；`CLAUDE_CODE_PROPAGATE_TRACEPARENT` 可显式放宽 |
| `queryTracking.chainId/depth` | `oRf(...)` 上层 query loop | 不直接进 API header/body | `NDl / KOo / UDl`、fallback credit telemetry、cache diagnosis | 描述同一 query 链的层级；和 `previousRequestId` 互补 |
| `attempt` | `Kur(...)` 回调 | 不进 API body | `KOo / UDl / F7t / api_retries_exhausted`、Perfetto attempt events | 当前最终 attempt 编号；普通可恢复 retry 不单独打 final error |
| `firstAttemptRequestId` | streaming request id 在切到 non-streaming 前保存 | 不进 API body | `UDl -> _Rf(...)` | 只在最终 success id 和首个 request id 不一致时上报 |
| `gateway` | `$Dl(headers/baseUrl)` | 不进 request；从响应 headers 或 base URL 推导 | `KOo / UDl` | telemetry 分类字段，当前识别 `litellm / helicone / portkey / cloudflare-ai-gateway / kong / braintrust / databricks` |
| `serverFallbackHop` | fallback block / refusal handling | 不进普通 request header | `F7t(...)` | 只说明 refusal 是否发生在 server-side fallback hop 上 |

还有两条 header 只在严格条件下出现：

- `anthropic-usage-limit: extended`：query depth 大于 0、first-party、官方 base URL、非普通 auxiliary 或 compact、且 `tengu_lantern_spool` feature flag 打开时加入。
- `anthropic-dispatch-id: v2s`：first-party、官方 base URL、非 auxiliary、且 `tengu_cedar_lattice` feature flag 打开时加入；若首个事件前 5xx/连接错误或 body phase 出错，会 strip 后 retry，并记录 `tengu_dispatch_header_fallback`。

这些字段共同用于日志、trace、retry/fallback 和 server-log lookup。当前本地证据只能证明字段生成、发送、记录和 strip/retry 行为，不能推出服务端如何解释这些字段。

这里还要补一个分类边界：

- `model`、`effortLevel`、`fastMode`、provider 切换、permission/sandbox 相关模式，都会直接改写这些 telemetry 字段的值
- 但它们不应被误写成“telemetry 专用设置”
- 更准确地说，它们属于业务运行态设置，而 telemetry 只是把这些运行态选择记录下来

### stream watchdog / output flush 相关 env

当前外围合同层里有两个 env 需要落到 request / stream 文档，而不是 settings schema 页：

| env | 当前语义 | 边界 |
| --- | --- | --- |
| `CLAUDE_STREAM_IDLE_TIMEOUT_MS` | 作为 stream idle timeout 数值进入请求/stream watchdog 侧 | `Number(process.env.CLAUDE_STREAM_IDLE_TIMEOUT_MS) || 0` 说明非数值或空值不会形成有效 timeout；它不是 settings key |
| `CLAUDE_CODE_FORCE_SYNC_OUTPUT` | 参与输出 flush / sync-output 判定 | changelog 里的 `FORCE_SYNC_OUTPUT` 在当前 bundle 里应写成完整 env 名；它影响输出策略，不改变模型请求协议 |

这两项都属于运行进程级开关。恢复实现时应把它们放在 request/transport 与 TUI 输出边界之间理解：它们能改变客户端等待、flush 或 fallback 表现，但不能推出服务端 stream 协议本身发生变化。

## refusal / model block 相关的反滥用入口

请求链里还有两类和反滥用更相关、但不属于 prompt 隐写的机制。

第一类是服务端 refusal 的本地记录。若返回 `stop_details` 且 `stop_reason == "refusal"`，本地会打 `api_refusal` 事件：

- 固定记录 `model / request_id / speed / attempt / server_fallback_hop`
- 记录是否带 `category` 和 `explanation`
- 只有在 enhanced telemetry 条件满足时，才把已知 category 写入事件
- 当前 category 白名单是 `cyber / bio / frontier_llm / reasoning_extraction`

第二类是请求前的 model off-switch / per-model block：

- `Hpc(...)` 会先查 GrowthBook `tengu-off-switch`
- 再查 `tengu-model-error-overrides`
- 命中 per-model block 时，有 fallback model 则抛出 fallback 信号；否则返回 rate-limit 风格错误内容
- 对应 telemetry 是 `tengu_off_switch_query`

这两类机制说明客户端会响应服务端或远端配置下发的安全/可用性状态，但它们是显式控制/记录路径，不是 `currentDate` 那种自然语言载体。

### 当前还保留的边界

这一段虽然已经能闭环，但仍有两个边界要保留：

- `promptCategory` 当前已能确认只出现在错误 telemetry 形参与 spread 中，本地 bundle 未发现任何生产调用方实际传值，因此更像未接线保留槽位
- `gateway` 识别当前只看到 `$Dl(...)` 这条本地分类链；它同时覆盖 header prefix 与少量 base URL host suffix，但不能据此证明服务端侧不存在更多分类

## `stream_request_start` 的真实来源

这个点也已经钉死：

- `stream_request_start` 不是 `Hpc(...)` 发的
- 它来自主循环 `oRf(...)`
- 每轮 `callModel` 前，`oRf(...)` 都会先 `yield { type: "stream_request_start" }`

所以它属于“主循环 UI 协议事件”，不属于 provider/SDK event。

## 本模块剩余状态

到这一步，`provider / gateway / transport` 主模块已经没有阻碍重写的未知点。

还没逐项归档完的，只剩一些外围细枝末节：

- telemetry 初始化链、startup telemetry 与非必要流量 gating 仍未在本页展开
- 这些内容现已转移到 [09-api-lifecycle-and-telemetry.md](../09-api-lifecycle-and-telemetry.md)
- 少量 cache/header 的边缘行为仍未完全枚举
- 少量显然未启用的 dead-code/feature-flag stub（例如恒为 `false/null` 的 override helper）没有继续浪费时间做无效深挖

这些都不再构成 `provider / gateway / transport` 的结构性未知点。

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
