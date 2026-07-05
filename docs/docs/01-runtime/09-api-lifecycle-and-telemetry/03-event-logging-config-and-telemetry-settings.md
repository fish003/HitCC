# 1P event logging 配置与 telemetry 设置

## 本页用途

- 整理 `Qg7()` / `gg7()` 远程配置、1P event logging batch config 与 endpoint 边界。
- 归纳哪些 settings/env 会修改 telemetry runtime、exporter 或事件 payload。

## 相关文件

- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../06-stream-processing-and-remote-transport.md](../06-stream-processing-and-remote-transport.md)
- [../../02-execution/06-context-runtime-and-tool-use-context.md](../../02-execution/06-context-runtime-and-tool-use-context.md)
- [../../05-appendix/02-evidence-map.md](../../05-appendix/02-evidence-map.md)

## 1P event logging 配置来源：`Qg7()` / `gg7()`

1P event logging 不是写死常量初始化，而是显式吃两组动态配置，见 `package/preprocessed/cli.extracted.bundle.pretty.js`。

### `tengu_event_sampling_config`

- `gg7()` 读取 `TZ("tengu_event_sampling_config", {})`
- `vZ1(eventName)` 再从结果里按事件名取 `sample_rate`
- `sample_rate`
  - 不是数字、或小于 `0`、或大于 `1`：视为无效，返回 `null`
  - 大于等于 `1`：等价于不抽样
  - 小于等于 `0`：等价于完全丢弃
  - `0~1`：按概率采样

### `tengu_1p_event_batch_config`

- `Fg7()` 读取 `TZ("tengu_1p_event_batch_config", {})`
- `Qg7()` 会把这组配置映射到 `BatchLogRecordProcessor` 与 `ZZ1` exporter：
  - `scheduledDelayMillis`
    - 优先 `q.scheduledDelayMillis`
    - 否则回退到 `parseInt(process.env.OTEL_LOGS_EXPORT_INTERVAL || "10000")`
  - `maxExportBatchSize`
    - 优先 `q.maxExportBatchSize`
    - 默认 `200`
  - `maxQueueSize`
    - 优先 `q.maxQueueSize`
    - 默认 `8192`
  - `skipAuth`
    - 透传到 `new ZZ1({ skipAuth })`
  - `maxAttempts`
    - 透传到 `new ZZ1({ maxAttempts })`
  - `path`
    - 透传到 `new ZZ1({ path })`
  - `baseUrl`
    - 透传到 `new ZZ1({ baseUrl })`

因此更稳的结论是：

- 1P event logging 的 flush cadence 既受 GrowthBook/remote config 影响，也受 `OTEL_LOGS_EXPORT_INTERVAL` 兜底
- batch exporter 的 endpoint、认证策略与重试上限都不是固定编译时常量

## 哪些设置会修改遥测数据或上报行为

这一层当前需要明确拆开，不要再混成“telemetry 开关”一个词。

本地已能正证的入口主要有三类：

- 通过 settings `env` 注入、最后进入 `process.env` 的环境变量
- remote config / GrowthBook 下发的 1P event logging 配置
- 业务设置本身，它们不是 telemetry 专用开关，但会直接改写 telemetry payload 里的字段值

### A. 产品运行态模式位

这组变量主要决定“哪些 telemetry / 外围流量路径会被旁路”，而不等于“telemetry runtime 一定不初始化”。

| 设置 | 当前作用 |
| --- | --- |
| `DISABLE_TELEMETRY` | 把运行态切到 `no-telemetry`，通过 `Khe()/NB()` 压掉 startup telemetry 和部分外围路径；当前本地 bundle 没看到它直接短路 `initializeTelemetry()` |
| `DO_NOT_TRACK` | 也会把运行态切到 `no-telemetry`；直接读点在 `qHs()`，语义接近 privacy-level 入口，不是 3P OTEL exporter 主开关 |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 把运行态切到 `essential-traffic`，影响面比 `DISABLE_TELEMETRY` 更宽，会压掉更多非必要出网/上报链路 |
| `DISABLE_ERROR_REPORTING` | 只直接作用于错误上报 sink `O6(...)`，不等于 OTEL / 1P event logging / startup telemetry 全停 |

### B. 标准 3P OpenTelemetry 组装与 exporter 配置

这组变量主要决定“是否组装 3P telemetry exporter、发到哪里、多久 flush 一次”。

| 设置 | 当前作用 |
| --- | --- |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | 标准 3P OTEL metrics/logs/traces exporter 总开关 |
| `OTEL_METRICS_EXPORTER` | metrics exporter 类型集合 |
| `OTEL_LOGS_EXPORTER` | logs exporter 类型集合；决定 `GO(...)` 事件是否可能真正发出 |
| `OTEL_TRACES_EXPORTER` | traces exporter 类型集合 |
| `OTEL_METRIC_EXPORT_INTERVAL` | metrics 导出间隔 |
| `OTEL_LOGS_EXPORT_INTERVAL` | logs 导出间隔；同时也是 1P event logging batch delay 的 env fallback |
| `OTEL_TRACES_EXPORT_INTERVAL` | traces 导出间隔 |
| `OTEL_EXPORTER_OTLP_ENDPOINT` 与 `OTEL_EXPORTER_OTLP_*` | OTLP exporter endpoint、headers、protocol、TLS/client cert 等连接参数 |
| `CLAUDE_CODE_OTEL_SHUTDOWN_TIMEOUT_MS` | telemetry shutdown 时的 flush timeout |
| `CLAUDE_CODE_OTEL_FLUSH_TIMEOUT_MS` | 显式 flush 时的 timeout |

这里要强调：

- 这组 `OTEL_*` 大量来自 vendored OpenTelemetry SDK
- 但它们确实会改变客户端遥测的上报后端、批处理节奏和连接参数
- 因此不能因为它们“不是产品自定义键名”就把它们排除在遥测配置面之外

### C. beta tracing 旁路

这组变量不是标准 OTEL 主开关，而是另一条 debug / tracing 旁路。

| 设置 | 当前作用 |
| --- | --- |
| `ENABLE_BETA_TRACING_DETAILED` | 打开 beta tracing 条件之一 |
| `BETA_TRACING_ENDPOINT` | beta tracing 的 log/trace OTLP endpoint |

当前更稳的边界是：

- `ENABLE_BETA_TRACING_DETAILED + BETA_TRACING_ENDPOINT` 可以绕过标准 `CLAUDE_CODE_ENABLE_TELEMETRY`
- 因此它们属于独立的 telemetry 配置面

### D. 会直接改写事件 payload 内容的 env

这组变量不是“是否上报”的开关，而是会改变每条事件带什么字段、字段是否脱敏。

| 设置 | 当前作用 |
| --- | --- |
| `OTEL_LOG_USER_PROMPTS` | 决定 `user_prompt` 事件里 `prompt` 是原文还是固定 `<REDACTED>` |
| `OTEL_METRICS_INCLUDE_SESSION_ID` | 决定通用 OTEL 事件属性里是否带 `session.id` |
| `OTEL_METRICS_INCLUDE_VERSION` | 决定是否带 `app.version` |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | 决定是否带 `user.account_uuid` 与 `user.account_id` |
| `CLAUDE_CODE_ACCOUNT_TAGGED_ID` | 改写 `user.account_id` 的值 |
| `CLAUDE_CODE_WORKSPACE_HOST_PATHS` | 为通用 OTEL 事件属性补 `workspace.host_paths` |

因此当前可直接写死：

- `user_prompt` 默认不是明文上报
- session/account/version 这几类公共维度是否出现，受独立 env 控制

### E. 1P event logging 的 remote config

这组不是本地一级 settings 键，而是通过 `TZ(...)` 读取的远端/实验配置。

| 设置 | 当前作用 |
| --- | --- |
| `tengu_event_sampling_config` | 按事件名配置 `sample_rate`，决定 1P internal event 是否抽样、全发或直接丢弃 |
| `tengu_1p_event_batch_config` | 控制 1P event logging 的 batch delay、队列大小、导出批大小、认证策略、重试次数、path、baseUrl |

因此 1P event logging 的“采样率”和“批处理发送策略”当前不是写死常量。

### 1P event logging 的 endpoint 不完全跟随自定义 base URL

`BYr` 构造 event logging endpoint 时有一个容易误读的边界：

- 显式传入 `baseUrl` 时，用传入值
- 否则只有 `ANTHROPIC_BASE_URL === "https://api-staging.anthropic.com"` 时走 staging
- 其他情况下默认使用 `https://api.anthropic.com`

也就是说，1P event logging 默认仍指向官方 API host，并不会因为任意第三方 `ANTHROPIC_BASE_URL` 自动改发到中转站。这一点和推理 data plane 的 `ANTHROPIC_BASE_URL` 覆盖不是同一层。

### F. 不是 telemetry 专用开关，但会改写 telemetry 字段值的业务设置

这些设置不应被归类成“telemetry settings”，但它们会直接进入 request/startup/internal event payload：

- `model`
- `effortLevel`
- `fastMode`
- provider 切换
- permissions / sandbox 相关模式

例如：

- request telemetry 会记录 `model`、`effortValue`、`fastMode`、`permissionMode`、`provider`
- usage/counter 层会把 `input_tokens / output_tokens / cache_read_input_tokens / cache_creation_input_tokens` 分别计入 input、output、cacheRead、cacheCreation，并在 UI breakdown 里显示 `cache hit: ...%`；作为索引名时可写成 `cache-hit`，但当前 UI 文本是带空格的 `cache hit`
- tool decision / tool result telemetry 会在参数可摘要时带 `tool_parameters`，例如 permission decision、tool success result 和部分 result record 都会把 `SBt(...)` / 参数摘要序列化进事件 payload
- request telemetry 还会通过 `zOo()` 记录 `baseUrl / envModel / envSmallFastModel`
  - `baseUrl` 来自 `ANTHROPIC_BASE_URL`；处理前会去掉 username、password、query、hash
  - 若 host 属于官方 Anthropic 域名，保留规范化后的 URL
  - 若不是官方 Anthropic 域名，记录的是规范化 URL 的短 hash，不是完整明文 URL
  - `envModel / envSmallFastModel` 来自 `ANTHROPIC_MODEL / ANTHROPIC_SMALL_FAST_MODEL`，按环境变量模型名明文进入事件 payload
- internal event core metadata 也会带 `model`、`betas`、`client_type` 等上下文

`cache hit` / `cache-hit` 与 cache token 字段的证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:448553-448609` 与 `package/sdk-tools.d.ts:103-107`；`tool_parameters` 证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:363262`、`532099-532103`、`532462-532652`。这类字段证明客户端会记录和呈现 cache read/write token 与部分 tool 参数摘要；不能单独证明服务端实际 TTL、价格策略或后端如何使用这些事件。

这和 `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` 下的 `/v1/models?limit=1000` 枚举不是同一条路径。后者会向自定义 gateway 请求模型列表，并把原始 `baseUrl` 和过滤后的模型列表写入本地 `gateway-models.json`；前者是 request 事件维度，非官方 URL 只保留 hash，环境变量模型名保留明文。

因此文档层更稳的分类应该是：

- “控制遥测 runtime / sink 的设置”
- “控制事件内容是否脱敏/是否附带维度的设置”
- “会间接改写遥测字段值的业务设置”

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
