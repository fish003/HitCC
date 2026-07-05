# 流量开关、请求生命周期与边界

## 本页用途

- 整理 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 与 `DISABLE_TELEMETRY` 的差异。
- 收束 request lifecycle telemetry、API body logging、hash/base URL 字段与仍需保守表述的边界。

## 相关文件

- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../06-stream-processing-and-remote-transport.md](../06-stream-processing-and-remote-transport.md)
- [../../02-execution/06-context-runtime-and-tool-use-context.md](../../02-execution/06-context-runtime-and-tool-use-context.md)
- [../../05-appendix/02-evidence-map.md](../../05-appendix/02-evidence-map.md)

## `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`DISABLE_TELEMETRY` 与 `DO_NOT_TRACK`：当前可确认的差异

### 模式判定

```text
qHs():
  CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC -> "essential-traffic"
  DISABLE_TELEMETRY -> "no-telemetry"
  DO_NOT_TRACK -> "no-telemetry"
  else -> "default"

Ki()  = mode === "essential-traffic"
Khe() = mode !== "default"
Kfn() = first matching env name
```

因此：

- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
  - `Ki() === true`
  - `Khe() === true`
- `DISABLE_TELEMETRY`
  - `Ki() === false`
  - `Khe() === true`
- `DO_NOT_TRACK`
  - `Ki() === false`
  - `Khe() === true`

这意味着三者不是同义词。`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 进入 essential-traffic mode；`DISABLE_TELEMETRY` 和 `DO_NOT_TRACK` 进入 no-telemetry mode。

还能进一步补一层 helper 关系：

```text
NB() =
  CLAUDE_CODE_USE_BEDROCK
  || CLAUDE_CODE_USE_VERTEX
  || CLAUDE_CODE_USE_FOUNDRY
  || Khe()
```

因此：

- `NB()` 不是“telemetry disabled”判定
- 而是一个更宽的“某些外围能力需要关掉/绕开”的 helper
- provider 切到 3P，或进入 `essential-traffic / no-telemetry` 模式，都会让 `NB() === true`
- 当前文档中的旧名 `BO()` / `cl8()` 分别对应当前 bundle 的 `Ki()` / `Khe()`；继续复核时应优先使用当前名

### 当前已能直接确认的 gating 矩阵

#### `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`

先要区分两个层次：

- `Ki()` helper 直接命中的分支
- 同一个“essential-traffic”模式下、但不是通过 `Ki()` helper 守卫的分支

如果只按 `Ki()` 这个 helper 本身看，当前命中点已经能基本列全。

##### A. `Ki()` helper 直接命中的分支

**直接压掉出网/上报函数：**

- `O6(...)` 错误上报
- `jg7()` model capabilities 拉取
- `l28()` fast mode prefetch
- `$Tq()` rate-limit / quota 相关预拉取
- `Zu()` account settings 读取
- `dA6()` `claude_code_grove` 读取
- `vG_()` org metrics opt-out API
- `U1z()` bug/feedback 上报
- `J1A()` changelog 拉取
- `Zbz()` `/api/claude_cli/bootstrap`

这里还能再写细一点：

- `jg7()`
  - 通过 `client.models.list(...)` 拉模型能力
- `$Tq()`
  - 通过 `y1_()` 发一个 `source: "quota_check"` 的轻量模型请求，读取 quota / rate-limit 头
- `Zu()`
  - `GET /api/oauth/account/settings`
- `dA6()`
  - `GET /api/claude_code_grove`
- `vG_()`
  - `GET /api/claude_code/organizations/metrics_enabled`
- `U1z()`
  - `POST /api/claude_cli_feedback`
- `Zbz()`
  - `GET /api/claude_cli/bootstrap`

**不直接出网，但会提前关掉产品能力入口：**

- `p$("allow_product_feedback")`
  - 当本地没有 policy limits 配置时，`Ki() && qM_.has("allow_product_feedback")` 会直接返回 `false`
- `feedback/bug` 命令的 `isEnabled`
  - 显式把 `Ki()` 作为禁用条件之一

**只阻止后台刷新，不等于绝对阻止底层 endpoint：**

- `vk4()`
  - 只负责触发 `T1A()` 的后台 passes eligibility 刷新
  - 这里的 `Ki()` 命中说明“启动/后台刷新”会被压掉
  - 但底层 `C$z()` referral eligibility endpoint 本身在当前可见代码里没有单独 `Ki()` 守卫

##### B. 同一模式下，但不是 `Ki()` helper 直接命中的分支

下面这些路径同样会被 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 影响，但实现不是 `if (Ki())`：

- `s4m(...)` startup telemetry
  - 通过 `NB() -> Khe()` 被压掉
- `Wg7()` MCP registry 拉取
  - 直接检查 `process.env.CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`

因此如果未来继续补证据，必须避免把“`Ki()` helper 命中列表”和“整个 essential-traffic mode 的全部影响面”混成一张表。

##### C. `DISABLE_TELEMETRY` / `DO_NOT_TRACK` 不会一起压掉的东西

这一点现在也能说得更硬：

- `DISABLE_TELEMETRY`
  - 只会让 `qHs() === "no-telemetry"`
  - 因此 `Khe() === true`
  - 但 `Ki() === false`
- `DO_NOT_TRACK`
  - 同样进入 `no-telemetry`
  - 因此 `Khe() === true`
  - 但 `Ki() === false`
- 直接后果是：
  - 那些用 `Ki()` 守卫的“非必要流量”不会自动一起停
  - 例如 `Wg7()` 这类直接看 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 的路径，不会因为 `DISABLE_TELEMETRY` 被压掉

所以更稳的工程结论是：

- `DISABLE_TELEMETRY`
  - 更像“停 telemetry 相关面 + 触发 `Khe()` 族旁路”
- `DO_NOT_TRACK`
  - 通过同一 mode helper 进入 no-telemetry，影响面应按 `DISABLE_TELEMETRY` 同类处理
- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
  - 才是“真正的非必要出网总开关”

当前没看到它会阻断的主链：

- `Hpc(...)` / `Kur(...)` / `/v1/messages`
- auth token / API key 主鉴权
- remote managed settings 获取

### 一个更实用的本地判断口径

如果未来重写时只想快速判断“这个功能会不会被模式开关压掉”，当前更稳的顺序是：

1. 先看它是不是直接用 `Ki()`
2. 再看它是不是直接读 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
3. 再看它是不是通过 `NB()/Khe()` 被间接旁路
4. 最后再看它是否单独依赖 `CLAUDE_CODE_ENABLE_TELEMETRY` 或 beta tracing env

不要再把这些路径压缩成一个单布尔 `TelemetryEnabled`。

所以它更准确的语义是：

- 关闭“外围增强与遥测相关流量”
- 不是把 inference data plane 关掉

#### `DISABLE_TELEMETRY`

当前在 bundle 里能直接看到的作用更窄：

- 让模式变成 `no-telemetry`
- 因而 `Khe() === true`
- 进而让 `NB()` 为真，压掉 `tengu_startup_telemetry`

但当前本地 bundle **没有直接看到** 它：

- 让 `Ki()` 变真
- 或直接短路 `initializeTelemetry()`
- 或直接传入 `yXa()`

所以当前更稳的判断应写成：

- `DISABLE_TELEMETRY` 在本地 bundle 里主要是一个“外层模式位”
- 它不会像 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 那样批量压掉所有 `Ki()` 守卫路径
- 它也不等同于“彻底不初始化 telemetry runtime”

#### 本地 bundle 中，`DISABLE_TELEMETRY` 只有一个直接读点

这一点现在可以收紧得更硬，不必继续写成“暂时没看到更多”。

当前本地 bundle 对 `DISABLE_TELEMETRY` 的直接读取只看到一处，见 `package/preprocessed/cli.extracted.bundle.pretty.js`：

- `qHs()`
  - `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC -> "essential-traffic"`
  - `DISABLE_TELEMETRY -> "no-telemetry"`
  - `DO_NOT_TRACK -> "no-telemetry"`
  - 否则 `"default"`
- `Khe()` 只是 `qHs() !== "default"`
- `NB()` 再把 `Khe()` 并入 startup telemetry / 若干外围路径的 gating

与之相对，3P OpenTelemetry 初始化主链 `Wb_()` 读的是另一套变量，见 `package/preprocessed/cli.extracted.bundle.pretty.js`：

- `yXa()` 只看 `CLAUDE_CODE_ENABLE_TELEMETRY`
- `Jb_()` / `Mb_()` / `Pb_()` 从 `OTEL_*` 环境变量组装 metrics/logs/traces exporter
- shutdown 提示里明确写的是“Disable OpenTelemetry by unsetting `CLAUDE_CODE_ENABLE_TELEMETRY`”

因此当前能正证的本地结论应改成：

- `DISABLE_TELEMETRY` 在本地 bundle 里不是 3P OTEL exporter 的主开关
- 标准 3P OTEL logs/metrics/traces 是否组装，直接取决于 `CLAUDE_CODE_ENABLE_TELEMETRY`
- `DISABLE_TELEMETRY` 与 `DO_NOT_TRACK` 当前可见作用仍主要是把运行态切到 `no-telemetry`，从而通过 `Khe()/NB()` 压掉 startup telemetry 和部分外围流量

#### `OTEL_*` 命中很多，但大多属于 vendored OpenTelemetry

继续按字面搜索后，这个边界也可以写得更清楚：

- bundle 里确实有大量 `OTEL_*` 环境变量命中
- 但其中相当一部分来自 vendored OpenTelemetry SDK 自身
- 对产品侧行为最关键的本地 wrapper 仍主要集中在：
  - `yXa()`
    - 读 `CLAUDE_CODE_ENABLE_TELEMETRY`
  - `fb_()`
    - 读 `BETA_TRACING_ENDPOINT`
  - `Wb_()`
    - 组装 readers / log exporters / trace exporters
  - `qHs()`
    - 读 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
    - 读 `DISABLE_TELEMETRY`
    - 读 `DO_NOT_TRACK`

因此不要把“bundle 内存在很多 `OTEL_*` 读点”误解成“产品层自己维护了一大套额外 gating”。  
更稳的工程划分是：

- `OTEL_*`
  - 主要属于 exporter/SDK 配置面
- `CLAUDE_CODE_ENABLE_TELEMETRY`
  - 标准 3P OTEL 组装总开关
- `BETA_TRACING_ENDPOINT` / `ENABLE_BETA_TRACING_DETAILED`
  - beta tracing 旁路
- `DISABLE_TELEMETRY` / `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
  - 产品运行态模式位
  - `DO_NOT_TRACK` 走同一个 no-telemetry helper，但不是 essential-traffic mode

#### 当前还能直接确认的间接影响面

除了 `tengu_startup_telemetry`，`DISABLE_TELEMETRY / DO_NOT_TRACK -> Khe() -> NB()` 在本地还会命中这些路径：

- 1P event logging 总开关
  - `b_6() = !NB()`
  - `vU6(...)`、`TZ1(...)`、`Qg7()`、`EA9()` 都先检查 `b_6()`
  - 因此 `DISABLE_TELEMETRY` 与 `DO_NOT_TRACK` 会把 first-party internal event logging 与 GrowthBook experiment event logging 一并压掉
- `tengu_context_size`
  - `tb4(...)` 开头就是 `if (NB()) return`
  - 所以这条上下文规模统计也会被压掉
- feedback survey
  - `W_8() = Khe()`
  - session quality / transcript ask 这组 survey 展示逻辑里会直接 `if (W_8()) return`
  - 因此 `DISABLE_TELEMETRY` 与 `DO_NOT_TRACK` 也会关闭产品反馈 survey 弹出

这里要特别区分：

- `/bug` 命令是否可用，当前仍主要受 `Ki()` 和 `allow_product_feedback` 控制
- feedback survey 则会被 `W_8()` 直接关掉

也就是说，`DISABLE_TELEMETRY` 与 `DO_NOT_TRACK` 会关 survey，但当前本地并不能证明它们会像 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 一样把 `/bug` 命令入口也一并关掉

#### `DISABLE_ERROR_REPORTING`

`O6(error)` 的 return-early 条件当前已经能直接写死，见 `package/preprocessed/cli.extracted.bundle.pretty.js`：

- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`
- `DISABLE_ERROR_REPORTING`
- `Ki() === true`

因此这里要避免继续写成“`O6(...)` 只会被 `Ki()` 压掉”。

更准确的说法是：

- `DISABLE_ERROR_REPORTING` 只直接作用于错误上报 sink
- 它和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、provider 切换开关会在 `O6(...)` 这一层汇合
- 这不等同于 `initializeTelemetry()`、`tengu_startup_telemetry`、1P event logging 都被同一变量关闭

## 一个反直觉但现在已能确认的点

从本地代码直读，下面这件事是成立的：

- 即使 `DISABLE_TELEMETRY=1`
- 只要启动链仍调用 `Rg8() -> H4A() -> initializeTelemetry()`
- 并且别的条件满足
- telemetry runtime 仍可能被初始化

特别是：

- beta tracing 旁路 `rj()` 不读取 `DISABLE_TELEMETRY`
- 它只看 `ENABLE_BETA_TRACING_DETAILED + BETA_TRACING_ENDPOINT`

所以如果未来要重写，这里必须明确拆成：

- 非必要流量模式
- 1P startup telemetry
- 标准 OTEL logs/metrics/traces
- beta tracing debug path

不能继续混成一个 `TelemetryEnabled` 布尔值。

## Request lifecycle telemetry 字段表

请求级 telemetry 现在可以在本页给出独立字段表；更长的调用链仍放在 stream/transport 专题页。

| 事件 | 入口 | sink | 字段来源与边界 |
| --- | --- | --- | --- |
| `tengu_api_query` | `NDl(...)` | `q(...)` | 在 `Kur(...)/beta.messages.create(...)` 之前触发；记录 `model`、`messagesLength`、`temperature`、`betas`、`permissionMode`、`querySource`、`messageClientPlatform`、`queryChainId/queryDepth`、`thinkingType`、`effortValue`、`fastMode`、`previousRequestId`、`provider`、`buildAgeMins`、`baseUrl/envModel/envSmallFastModel`；此时还没有服务端 `requestId` |
| `tengu_api_error` | `KOo(...)` | `q(...)` | 只在整次 request 最终失败时触发；记录归一化 `error/errorType/status`、`messageCount/messageTokens`、最后一次 attempt 的 `durationMs`、含重试总耗时 `durationMsIncludingRetries`、`attempt`、`requestId/clientRequestId`、`didFallBackToNonStreaming`、`gateway`、query tracking、`fastMode`、`previousRequestId`、attribution 与 base URL/model env 摘要 |
| `tengu_api_success` | `UDl(...) -> _Rf(...)` | `q(...)` | 只在最终成功后触发；记录 usage/cost、duration、attempt、`ttftMs`、`requestId`、`firstAttemptRequestId`、`stop_reason`、fallback 状态、TTY/print 状态、query tracking、permission mode、cache strategy、输出文本/thinking/tool-use 长度、输入 image/document/text 统计、`isPostCompaction`、attribution、base URL/model env 摘要和 `timeSinceLastApiCallMs` |
| `api_refusal` | `F7t(...)` | `Jc(...)` | 当最终消息 `stop_reason === "refusal"` 且带 `stop_details` 时触发；记录 `model`、`request_id`、`speed`、`attempt`、`server_fallback_hop`、query source、effort、是否有 category/explanation；只有 enhanced telemetry 条件满足时才写入白名单 category |

同一 request 还会在 `KOo(...)` / `UDl(...)` 侧写 3P event logger：

- error：`Jc("api_error", ...)`，多次重试耗尽时还会写 `Jc("api_retries_exhausted", ...)`
- success：`Jc("api_request", ...)`
- assistant response：`Jc("assistant_response", ...)`，默认 `response` 是 `<REDACTED>`，只有 `OTEL_LOG_ASSISTANT_RESPONSES` 或 `OTEL_LOG_USER_PROMPTS` 打开时才走 `wP(...)` 记录截断后的明文

这几条链路必须分开理解：

- `q(...)` 是 1P internal event logging 的业务事件名。
- `Jc(...)` 是 3P OTEL-style event logger 属性事件，若 logger 未初始化会丢弃并只打印一次 warning。
- `requestId` 来自服务端响应或错误对象；`clientRequestId` 来自 first-party 路径生成并通过 `x-client-request-id` header 发出。
- `traceparent` 由当前 active span 注入，只有 first-party / Anthropic AWS 默认 host 或显式 `CLAUDE_CODE_PROPAGATE_TRACEPARENT` 条件满足时进入 request header。

## API body logging 与 redaction

当前可见 redaction 规则也可以收紧：

- user prompt span 属性默认写 `<REDACTED>`；`OTEL_LOG_USER_PROMPTS` 打开后才记录明文。
- assistant response 默认写 `<REDACTED>`；`OTEL_LOG_ASSISTANT_RESPONSES` 或 `OTEL_LOG_USER_PROMPTS` 打开后才记录明文，并经 `wP(...)` 限制到 `61440` 字符，超出时追加 `[TRUNCATED - Content exceeds 60KB limit]`。
- beta tracing 的 request detail 只有 `ENABLE_BETA_TRACING_DETAILED + BETA_TRACING_ENDPOINT` 且入口条件满足时启用；它不是标准 `CLAUDE_CODE_ENABLE_TELEMETRY` 路径。
- `tengu_api_success` 的 `requestContentTelemetry` 是 image/document/text/token 估算和长度统计，不是把完整 request body 明文写进 1P event。

## 与请求生命周期页的关系

本页负责回答：

- telemetry 何时初始化
- event logger / meter / tracer 何时存在
- 哪些开关会关掉哪些外围流量
- request lifecycle 四类事件分别记录什么字段、走哪个 sink、默认是否 redacted

而更长的 request 调用链、retry/fallback 时机与 stream 状态，见：

- [06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md](../06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md)

## 当前仍保留的边界

- `DISABLE_TELEMETRY` / `DO_NOT_TRACK` 是否还会被远端服务端侧额外解释，本地 bundle 无法证明
- 1P event logging 的完整服务端协议未在 bundle 中展开，只能确认本地初始化与调用点
- 若只统计 `Ki()` helper 直接命中的分支，当前已基本覆盖
- 但若按“essential-traffic mode 的全部影响面”统计，仍需继续补直接读 env 或经 `Khe()/NB()` 间接生效的外围路径

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
