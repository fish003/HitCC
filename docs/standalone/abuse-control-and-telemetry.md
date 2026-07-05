# 封控与遥测专题

## 定位

本文是独立专题文档，不挂入 `recovery-docs/docs` 的现有导航体系。它把 `@anthropic-ai/claude-code@2.1.197` 本地 bundle 中可见的封控、风控、遥测、请求指纹、组织 policy、执行拦截、remote-control 设备信任、Gateway 防护与 refusal/fallback 协议面放在同一张图里。

证据主入口是当前发行版抽取出的：

- `package/preprocessed/cli.extracted.bundle.pretty.js`
- `package/preprocessed/cli.extracted.bundle.js`
- `package/sdk-tools.d.ts`

旧恢复文档只作为线索索引。凡是涉及服务端处罚、封号、蒸馏识别、人工审核、模型内部 safety 判定的内容，本文只写本地客户端能证明的字段生成、发送、消费、记录和执行边界。

## 总结

`2.1.197` 的封控/遥测不是单一机制。当前本地可见的是多层系统：

- prompt 内隐写只确认 `currentDate` 这一条自然语言标记链；它把 `ANTHROPIC_BASE_URL` host 命中状态与中国时区状态编码进日期句子。
- 模型请求会带客户端、session、remote、agent、trace、dispatch、usage 等 headers。
- request body 的 `metadata.user_id` 是 JSON 字符串，里面可包含 extra metadata、device id、account uuid、session id。
- request lifecycle telemetry 会记录 query、success、error、refusal、base URL 指纹、provider、permission mode、token/cost/duration 等字段。
- 1P event logging 是独立上传通道，默认走 `/api/event_logging/v2/batch`，有采样、batch、auth fallback、失败落盘。
- 3P OTEL、1P internal event logging、startup telemetry、beta tracing 是四条不同链路，不能压成一个 `TelemetryEnabled`。
- enterprise/team 控制面通过 remote managed settings 与 `policy_limits` 下发能力限制。
- Auto mode classifier、permission mode、sandbox、审批 backend 约束工具执行。
- workspace trust 会限制 project/local helper 执行和 project-scoped permission 扩展。
- subprocess env scrub、script call cap、deep-link 参数校验、UNC/WebDAV/git-dir 检测，属于入口和子进程防护。
- remote-control/bridge 有 OAuth access token、environment secret、session ingress token、worker jwt、Trusted Device token 等多级凭据。
- bridge event 可按 `device_attestation_status` 做 feature-gated drop/filter。
- Claude Code Gateway 带 SSRF、DNS、CIDR、TLS pin、CSRF、rate limit、payload size、spend limit 等防护。
- refusal/fallback 协议包含 `cyber / bio / frontier_llm / reasoning_extraction` category、server-side fallback、fallback credit、transcript retraction、model consent fallback。

这些事实只能证明客户端会产生、传递、记录或执行这些控制信号。服务端如何组合信号、如何判定封禁、如何识别蒸馏、是否人工审核，仍是 bundle 外黑箱。版本边界也要分开写：`2.1.197` 的本地 bundle 已确认存在这条 marker；`2.1.201` 的 npm native 包抽样核对已看不到 `Asia/Shanghai / Asia/Urumqi` 字面量，`currentDate` 附近恢复成普通 `Today's date is`；`2.1.91` 是否为引入起点尚未逐版本验证。

## 分层模型

| 层 | 本地可见对象 | 本地动作 | 主要边界 |
| --- | --- | --- | --- |
| Prompt 标记 | `currentDate` apostrophe/date 变体 | 生成、发送 | 只确认这一处自然语言隐写；服务端解释不可证明 |
| 请求 headers | `x-app`、`User-Agent`、session、remote、agent、dispatch、usage、additional protection | 生成、发送、条件重试 | 标记客户端和请求上下文；不是单项处罚证据 |
| Request metadata | `metadata: XLe()`、`metadata.user_id` JSON | 生成、发送 | 字段名是 `user_id`，内容是 JSON 字符串 |
| Trace/correlation | `x-client-request-id`、`traceparent`、`previousRequestId`、query chain | 生成、发送、记录 | 用于跨请求关联；trace 注入受条件限制 |
| Request telemetry | `tengu_api_query/error/success`、`api_refusal` | 记录 | 记录生命周期和 refusal；不直接等于服务端判定规则 |
| 1P event logging | `ClaudeCodeInternalEvent`、`GrowthbookExperimentEvent` | 结构化、上传、失败落盘 | 独立 `/api/event_logging/v2/batch` 通道 |
| 3P OTEL | metrics/logs/traces exporter | 初始化、导出 | 由 `CLAUDE_CODE_ENABLE_TELEMETRY` 与 `OTEL_*` 控制 exporter |
| 流量模式 | `DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | gating | 三者影响面不同 |
| 组织 policy | remote managed settings、`policy_limits` | 拉取、缓存、执行 | policy 生成规则在服务端 |
| 执行封控 | Auto mode classifier、permission、sandbox、approval | 执行、记录 | 约束工具和命令，不是模型 refusal category |
| Workspace trust | helper gate、additional directories clipping | 执行、记录 | 未受信项目不能先跑 credential/header/proxy helper |
| 子进程/入口 | env scrub、script cap、deep-link/UNC/git guard | 执行、记录 | 防止配置、协议处理器、恶意 repo 扩权或泄露 |
| Remote control | bridge credentials、Trusted Device、attestation filter | 发送、消费、执行 | 限制 remote worker 和 event ingress |
| Gateway | SSRF/DNS/CIDR/TLS/rate/CSRF/spend | 执行 | 企业 gateway/client 侧可见防护 |
| Refusal/fallback | refusal category、fallback model、fallback credit、retraction | 消费、执行、记录 | category 判定和 credit mint 在服务端 |

## 端到端数据流

把上面的层压成一条请求路径，可以看到封控/遥测信号并不是在同一个位置产生：

```text
startup
  -> settings / managed settings / policy_limits / GrowthBook
  -> prompt assembly
  -> request body + request headers + trace context
  -> stream response / refusal / fallback
  -> transcript canonicalization
  -> request lifecycle telemetry
  -> 1P event logging / 3P OTEL / local failed-event queue
```

关键切分：

- startup 阶段负责读取 settings、policy、feature flag、设备/账号/环境状态，也会发 startup telemetry。
- prompt assembly 阶段当前只确认 `currentDate` marker 进入自然语言 user context。
- request builder 阶段生成 headers、`metadata.user_id`、trace/correlation、quota/fallback 相关参数。
- stream consumer 阶段消费 refusal、fallback block、fallback credit、response headers 和 request id。
- transcript 层要处理 tombstone、supersedes、retraction，保持后续模型看到的 canonical history。
- telemetry/logging 层记录统计、状态、hash、长度、事件和归因，不应倒推成服务端处罚规则。

这种切分对恢复很重要：prompt marker、request identity、policy gate、sandbox gate、remote-control attestation、gateway 防护、refusal fallback 各自有不同的触发点和失败模式。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:53856-53877`
- `package/preprocessed/cli.extracted.bundle.pretty.js:176240-176623`
- `package/preprocessed/cli.extracted.bundle.pretty.js:279103`
- `package/preprocessed/cli.extracted.bundle.pretty.js:713876-714098`
- `package/preprocessed/cli.extracted.bundle.pretty.js:558740-559248`
- `package/preprocessed/cli.extracted.bundle.pretty.js:868141-868177`

## Prompt 内标记

当前唯一能闭环到自然语言 prompt 的标记链是 `currentDate`：

```text
Wla(base64)
  -> XOR 91
  -> split(",")
  -> edp()/tdp() 得到 domain list / keyword list
  -> rdp() 检查 ANTHROPIC_BASE_URL host 与 timezone
  -> odp(known, labKw) 选择 apostrophe 变体
  -> qla(date) 生成 Today?s date is ...
  -> userContext.currentDate
```

关键行为：

- `Wla(...)` 用 base64 加 XOR 91 解出两组名单。
- `rdp()` 读取 `ANTHROPIC_BASE_URL` host。
- `rdp()` 还检查 timezone 是否为 `Asia/Shanghai` 或 `Asia/Urumqi`。
- `odp(...)` 根据 `known / labKw` 选择普通 apostrophe、right single quote、modifier letter apostrophe、modifier letter prime：`' / ’ / ʼ / ʹ`。
- `qla(...)` 在中国时区下把日期里的 `-` 改成 `/`。
- 输出进入 `userContext.currentDate`，随后由 `otr(...)` 包成前置 `<system-reminder>` user meta message。

编码形态：

- `known` 是 1 bit，表示 host 命中 domain list。
- `labKw` 是 1 bit，表示 host 命中 keyword list。
- `cnTZ` 是 1 bit，表示 timezone 是 `Asia/Shanghai` 或 `Asia/Urumqi`。
- apostrophe 承载 `known / labKw` 两个 bit，日期分隔符承载 `cnTZ`，因此最多形成 3 bit marker。

边界：

- 这不是 URL blocklist。
- 这不是 gateway/proxy classifier。
- 这不是通用危险域名拦截表。
- 这不是 API request 的 top-level `system` 字段，而是 `messages[]` 前置 user meta context。
- 请求 headers、telemetry、policy limits、Trusted Device、model fallback 都和反滥用有关，但不属于“藏进自然语言 prompt 的隐写”。
- 服务器是否解码该 marker、如何用于账号风控、封禁或反蒸馏策略，本地 bundle 无法证明。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:274124-274175`
- `package/preprocessed/cli.extracted.bundle.pretty.js:279103`
- `package/preprocessed/cli.extracted.bundle.pretty.js:713236-713254`

## 请求身份与 headers

模型 API client 会组装请求身份面。当前能直接看到的字段包括：

- `x-app: cli / cli-bg`
- `User-Agent`
- `X-Claude-Code-Session-Id`
- `x-claude-remote-container-id`
- `x-claude-remote-session-id`
- `x-client-app`
- `x-claude-code-agent-id`
- `x-claude-code-parent-agent-id`
- `x-anthropic-additional-protection`
- `anthropic-usage-limit`
- `anthropic-dispatch-id`

其中需要分开看：

- `x-anthropic-additional-protection: true` 只在 `CLAUDE_CODE_ADDITIONAL_PROTECTION` 打开时加入。
- `anthropic-usage-limit: extended` 受 first-party、官方 base URL、query depth、feature flag、auxiliary/compact 等条件影响。
- `anthropic-dispatch-id: v2s` 受 first-party、官方 base URL、非 auxiliary、feature flag 影响。若 stream 首事件前因该 header 失败，会 retry without it，并记录 fallback telemetry。

这层可以证明客户端会把请求来源、会话、remote 环境、SDK client app、agent lineage 与部分灰度/额度信号发给上游。它不能证明任一 header 是独立封禁依据。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:161719-161738`
- `package/preprocessed/cli.extracted.bundle.pretty.js:715190-715207`
- `package/preprocessed/cli.extracted.bundle.pretty.js:716335-716338`
- `package/preprocessed/cli.extracted.bundle.pretty.js:717157-717158`

## Request metadata 与 trace

API request body 还有一个 metadata 通道：

```text
metadata: XLe()
```

`XLe()` 返回：

```text
{
  user_id: JSON.stringify({
    ...CLAUDE_CODE_EXTRA_METADATA,
    device_id,
    account_uuid,
    session_id
  })
}
```

边界：

- `CLAUDE_CODE_EXTRA_METADATA` 必须是 JSON object。
- `device_id` 来自本地设备标识。
- `account_uuid` 优先取 remote/account 环境或当前账号状态。
- `session_id` 来自当前 Claude Code session。
- `metadata.user_id` 不是裸 user id，而是 JSON 字符串容器。

请求 correlation 还有：

- `x-client-request-id`
- `traceparent`
- `previousRequestId`
- `queryTracking.chainId / depth`

`x-client-request-id` 只在 first-party 或 Anthropic AWS 默认 host 等条件下生成。`traceparent` 依赖 active span，并受 first-party、Anthropic AWS 默认 host 或 `CLAUDE_CODE_PROPAGATE_TRACEPARENT` 控制。

### 身份与关联字段字典

| 字段/来源 | 载体 | 当前含义 | 边界 |
| --- | --- | --- | --- |
| `device_id` | `metadata.user_id` JSON、1P event、OTEL resource | 本地设备标识 | 可关联设备；不是账号 UUID |
| `account_uuid` | `metadata.user_id` JSON、1P event auth | 当前账号或 remote/account 环境 | 可能为空字符串 |
| `session_id` | `metadata.user_id` JSON、1P event、OTEL resource | 当前 Claude Code session | 不等于 remote session id |
| `X-Claude-Code-Session-Id` | request header | CLI session header | 只证明客户端发送 session 维度 |
| `x-claude-remote-session-id` | request header / OTEL resource | Remote session 维度 | 仅 remote 场景有值 |
| `x-claude-code-agent-id` | request header | 当前 agent identity | 用于 agent lineage |
| `x-claude-code-parent-agent-id` | request header | parent agent identity | 用于子代理关联 |
| `x-client-request-id` | request header / telemetry | 客户端生成的 request correlation id | 生成有 provider/base URL 条件 |
| `traceparent` | request header | active span propagation | 默认不对所有 provider 注入 |
| `anthropic-dispatch-id` | request header | first-party 灰度/路由信号 | 首事件前失败会去掉后重试 |
| `anthropic-usage-limit` | request header | extended usage lane 信号 | 受 query depth、source、feature flag 条件限制 |
| `_PROTO_skill_name` / `_PROTO_plugin_name` | telemetry/event metadata | skill/plugin/marketplace 归因 | 属于事件归因面，不是 API body metadata |

这张表的恢复含义是：同一个“用户/会话”概念在不同层有不同字段。重写时应保留载体边界，不要把 `metadata.user_id`、headers、OTEL resource、1P event auth 合并成一个全局 `userId`。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:713876-713898`
- `package/preprocessed/cli.extracted.bundle.pretty.js:714082-714098`
- `package/preprocessed/cli.extracted.bundle.pretty.js:176520-176596`
- `package/preprocessed/cli.extracted.bundle.pretty.js:232202-232244`

## Quota、rate-limit 与轻量探测

客户端会发一个最小模型请求做 quota/rate-limit probe：

```text
source: "quota_check"
messages: [{ role: "user", content: "quota" }]
max_tokens: 1
metadata: XLe()
```

这不是用户 prompt 的隐藏内容，主要用于读取 quota / rate-limit headers。当前解析的 header 包括：

- `anthropic-ratelimit-unified-representative-claim`
- `anthropic-ratelimit-unified-overage-status`
- `anthropic-ratelimit-unified-overage-disabled-reason`
- `anthropic-ratelimit-unified-5h-utilization / reset`
- `anthropic-ratelimit-unified-7d-utilization / reset`
- `anthropic-ratelimit-unified-7d_oi-utilization / reset`
- `anthropic-ratelimit-unified-overage-utilization / reset`

这些字段会影响 rate-limit event、fast mode cooldown、extra usage/Fable overage consent 和 UI 选择面。它们属于额度状态控制面，不应写成 prompt 隐写或服务端处罚规则。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:281414-281496`
- `package/preprocessed/cli.extracted.bundle.pretty.js:837840-838004`

## Request lifecycle telemetry

请求级 telemetry 至少有四类事件：

| 事件 | 入口 | Sink | 字段类型 |
| --- | --- | --- | --- |
| `tengu_api_query` | `NDl(...)` | `q(...)` | model、message count、temperature、betas、permissionMode、querySource、queryTracking、thinking/effort、fastMode、previousRequestId、provider、buildAge、baseUrl/envModel/envSmallFastModel |
| `tengu_api_error` | `KOo(...)` | `q(...)` | error type/status、requestId/clientRequestId、retry/fallback、gateway、querySource、permissionMode、duration、attempt、base URL/model env 摘要 |
| `tengu_api_success` | `UDl(...) -> _Rf(...)` | `q(...)` | usage/cost、duration、attempt、ttft、requestId、stop_reason、fallback、query tracking、permission mode、cache strategy、文本/thinking/tool/result 尺寸 |
| `api_refusal` | `F7t(...)` | `Jc(...)` | model、request id、speed、attempt、server_fallback_hop、query source、effort、category/explanation presence |

base URL 指纹的边界：

- 官方 Anthropic / Claude host 会保留规范化 URL。
- 非官方 host 会 hash 成短指纹。
- gateway/proxy classifier 可以从 headers 中识别 `litellm / helicone / portkey / cloudflare-ai-gateway / kong / braintrust` 等。

明文边界：

- request telemetry 记录的是字段、统计量、hash、长度、token、成本、状态。
- 它不是完整 request body 明文转储。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:560484-560514`
- `package/preprocessed/cli.extracted.bundle.pretty.js:560547-560577`
- `package/preprocessed/cli.extracted.bundle.pretty.js:560649-560678`
- `package/preprocessed/cli.extracted.bundle.pretty.js:560721-560749`
- `package/preprocessed/cli.extracted.bundle.pretty.js:560872-560928`
- `package/preprocessed/cli.extracted.bundle.pretty.js:561146-561161`

## Telemetry 四条链

Claude Code CLI 的 telemetry 至少要拆成四层：

1. Request lifecycle telemetry
   - `NDl / KOo / UDl / F7t`
   - 记录模型请求生命周期、成功、失败、refusal。
2. Startup telemetry
   - `tengu_startup_telemetry`
   - 记录 git/worktree、gh auth、sandbox、theme、settings/env 摘要、证书开关 presence 等。
3. 3P OTEL metrics/logs/traces
   - `initializeTelemetry()`
   - 由 `CLAUDE_CODE_ENABLE_TELEMETRY` 与 `OTEL_*` 组装 exporter。
4. 1P event logging / GrowthBook event logger
   - `/api/event_logging/v2/batch`
   - 有 event sampling、batch config、auth fallback、失败落盘。

这四条链的初始化、开关、sink、字段形态不同。重写时不能用一个布尔变量代表全部。

## Telemetry 开关矩阵

| 变量 | 当前可见作用 | 注意点 |
| --- | --- | --- |
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` | 进入 `essential-traffic` mode，`Ki() === true`，压掉多条非必要出网/上报路径 | 真正更接近“非必要流量总开关” |
| `DISABLE_TELEMETRY` | 进入 `no-telemetry` mode，`Khe() === true`，压掉 startup telemetry、1P event logging、survey 等部分路径 | 当前没看到它直接短路 `initializeTelemetry()` |
| `DO_NOT_TRACK` | 与 `DISABLE_TELEMETRY` 同样进入 `no-telemetry` mode | 不是 essential-traffic mode |
| `DISABLE_ERROR_REPORTING` | 直接作用于错误上报 sink `O6(...)` | 不等于 OTEL / 1P event logging / startup telemetry 全停 |
| `CLAUDE_CODE_ENABLE_TELEMETRY` | 标准 3P OTEL exporter 总开关 | 控制 exporter 组装，不是所有 telemetry runtime 的总开关 |
| `OTEL_*` | 标准 OTEL exporter endpoint、headers、protocol、flush interval、TLS/client cert 等 | 大量来自 vendored OpenTelemetry SDK |
| `ENABLE_BETA_TRACING_DETAILED` + `BETA_TRACING_ENDPOINT` | beta tracing debug path | 可独立于标准 `CLAUDE_CODE_ENABLE_TELEMETRY` 拉起 tracer/logger |

`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 当前会压掉的已知路径包括 error reporting、model capabilities fetch、fast mode prefetch、quota/rate-limit prefetch、account settings、Grove、org metrics opt-out、feedback、changelog、bootstrap、MCP registry 等。

当前没看到它阻断：

- `/v1/messages` 主推理 data plane
- auth token / API key 主鉴权
- remote managed settings 获取

### 非必要流量下的功能退化

`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 不只是“少打 telemetry”。它会让若干产品功能直接不可用或跳过预取：

| Surface | `Ki()` 下当前行为 | 恢复边界 |
| --- | --- | --- |
| model capabilities fetch | 直接跳过 | 不能依赖远端 capability 作为启动必需项 |
| quota/rate-limit prefetch | 直接跳过 | 不影响主请求实时收到 rate-limit error |
| account settings / Grove / org metrics opt-out | 直接跳过 | 属于非必要控制面或体验面 |
| feedback / survey / changelog / bootstrap | 直接跳过或禁用 | 不是推理必要路径 |
| MCP registry | 受限或跳过 | 不等于本地 MCP 配置失效 |
| Design Sync / Projects / live preview / send user file | 直接不可用或 `isEnabled() === false` | 产品功能退化，不应报成 API 鉴权失败 |
| Remote Control eligibility | feature flag 评估不可用时给出明确原因 | remote-control 依赖 GrowthBook/feature flag |
| Push/mobile notification | Remote Control 相关 transport 不可用时降级 | 本地 terminal notification 可独立存在 |

`DISABLE_TELEMETRY` / `DO_NOT_TRACK` 走 `Khe()` 的 no-telemetry privacy mode。部分 UI 会据此隐藏 thumbs feedback 等 rating 入口，因为 rating event 会成为 no-op；这和 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 的“限制非必要出网”不是同一个语义。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:53856-53877`
- `package/preprocessed/cli.extracted.bundle.pretty.js:273409-273412`
- `package/preprocessed/cli.extracted.bundle.pretty.js:281499`
- `package/preprocessed/cli.extracted.bundle.pretty.js:524654-524668`
- `package/preprocessed/cli.extracted.bundle.pretty.js:525754`
- `package/preprocessed/cli.extracted.bundle.pretty.js:579841-580103`
- `package/preprocessed/cli.extracted.bundle.pretty.js:725870-725947`
- `package/preprocessed/cli.extracted.bundle.pretty.js:838151`

## 1P event logging

1P event logging 的 endpoint 是：

```text
/api/event_logging/v2/batch
```

默认情况下 endpoint host 不完全跟随任意 `ANTHROPIC_BASE_URL`。显式 `baseUrl` 会优先使用；否则 staging 只在 `ANTHROPIC_BASE_URL === "https://api-staging.anthropic.com"` 时使用；其他情况默认 `https://api.anthropic.com`。

上传行为：

- 认证优先走 first-party bearer / API key selector。
- trust 未建立或 OAuth 过期时可降级无 auth。
- 带 auth 命中 `401` 时会再试一次无 auth。
- 失败批次会落盘到 `telemetry/1p_failed_events.<session>.<uuid>.json`；重启后会扫描同 session 前缀的旧失败文件并后台重试。

配置来源：

- `tengu_event_sampling_config`
  - 按事件名配置 `sample_rate`。
  - 无效值返回 `null`。
  - `>= 1` 等价全发。
  - `<= 0` 等价全丢。
  - `0~1` 按概率采样。
- `tengu_1p_event_batch_config`
  - 控制 scheduled delay、batch size、queue size、skipAuth、maxAttempts、path、baseUrl。
  - `scheduledDelayMillis` 缺省时回退 `OTEL_LOGS_EXPORT_INTERVAL || "10000"`。

事件形态：

- `GrowthbookExperimentEvent`
  - `event_id`、`timestamp`、`experiment_id`、`variation_id`、`environment`、`device_id`、`session_id`、`auth`、`user_attributes`、`experiment_metadata`。
- `ClaudeCodeInternalEvent`
  - `core`、`env`、`process`、`auth`、`additional_metadata`。
  - `device_id` 与 `email` 会提升到顶层。
  - `_PROTO_skill_name`、`_PROTO_plugin_name`、`_PROTO_marketplace_name`、`_PROTO_code`、`_PROTO_head_sha` 会从 additional metadata 中拆出并提升。

这说明 1P event logging 有独立归因面，可以记录 skill、plugin、marketplace、code/head sha、session、model、client type、remote env、auth 等上下文。它不等于模型 API payload 的 `metadata.user_id`。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:176520-176596`
- `package/preprocessed/cli.extracted.bundle.pretty.js:176240-176319`
- `package/preprocessed/cli.extracted.bundle.pretty.js:176476-176623`

## OTEL 与 API body logging 的 redaction

默认 redaction 边界：

- `user_prompt` 默认写 `<REDACTED>`，只有 `OTEL_LOG_USER_PROMPTS` 打开才记录明文。
- assistant response 默认写 `<REDACTED>`，只有 `OTEL_LOG_ASSISTANT_RESPONSES` 或 `OTEL_LOG_USER_PROMPTS` 打开才记录明文。
- 明文 assistant response 经过 `wP(...)`，限制到 60KB。
- API body logging 会把 assistant history 里的 `thinking.thinking` 和 `redacted_thinking.data` 替换成 `<REDACTED>`。
- `tengu_api_success` 的 `requestContentTelemetry` 是 image/document/text/token 估算和长度统计。

这说明本地日志默认不把用户 prompt、assistant response、thinking 明文直接打进 OTEL/body log。打开相关 env 后，明文边界会变化。

OTEL resource 与 body log 还有两层需要区分：

- `OTEL_RESOURCE_ATTRIBUTES` 会被解析为 resource attributes，但 key/value 长度和字符集受限；当已有 auth identity metadata 时，`user.*` / `identity.*` 前缀会被过滤，避免外部 env 覆盖身份类字段。
- `OTEL_METRICS_INCLUDE_SESSION_ID` 默认 true，会加入 `session.id`；remote session 存在时也会加入 `ccr.session.id`。
- `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` 默认 true，账号存在时可加入 `user.account_uuid` 与 tagged account id。
- `OTEL_METRICS_INCLUDE_VERSION` 默认 false，打开后加入 `app.version`。
- `OTEL_METRICS_INCLUDE_ENTRYPOINT` 默认 false，打开后加入 `app.entrypoint`。
- API raw body logging 支持 inline 与 file mode；file mode 只把 `body_ref`、`body_length` 等写入事件，body 本体落到本地文件路径。
- inline body 超过 60KB 会截断并标记 `body_truncated`。

恢复时要把“默认 redaction”与“用户显式打开明文/文件 body log”分开。后者改变本地日志的敏感度，但仍不能推出服务端如何处理这些日志。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:175522-175525`
- `package/preprocessed/cli.extracted.bundle.pretty.js:232202-232320`
- `package/preprocessed/cli.extracted.bundle.pretty.js:232303-232310`
- `package/preprocessed/cli.extracted.bundle.pretty.js:560431-560432`
- `package/preprocessed/cli.extracted.bundle.pretty.js:561068-561075`

## 组织 policy 与 remote managed settings

enterprise/team 控制面至少有两条远端链：

```text
GET /api/claude_code/settings
  -> remote managed settings
  -> policySettings
  -> effective settings merge

GET /api/claude_code/policy_limits
  -> restrictions / compliance_taints / monitoring_notice / defaults
  -> capability gate
```

`policy_limits` 的本地语义：

- `restrictions` 是逐 capability 的 allow/deny。
- `compliance_taints` 可映射到 capability deny。
- `monitoring_notice.text` 会移除控制字符。
- `monitoring_notice.url` 只接受 `https://`。
- `defaults` 可作为 policy default 来源。
- fetch 结果会记录 `tengu_policy_limits_fetch`。

Eligibility：

- first-party provider。
- 官方 base URL。
- first-party API key、WIF、或带 inference scope 的 OAuth。
- prosumer OAuth、无 auth、无 inference scope、自定义 base URL 默认不合格。

Cache：

- canonical JSON 算 `sha256:<hex>`，作为 `If-None-Match`。
- `304` 使用缓存。
- `404` 视为无限制。
- schema 校验失败则拒绝新结果。
- fetch 失败时可使用 stale cache。

Remote managed settings 里的高价值 policy keys：

- `availableModels / enforceAvailableModels / modelOverrides`
- `forceLoginOrgUUID / forceLoginMethod / forceRemoteSettingsRefresh`
- `disableRemoteControl / disableAgentView / disableWorkflows / disableArtifact`
- `allowManagedHooksOnly / allowManagedPermissionRulesOnly / allowManagedMcpServersOnly`
- `allowedHttpHookUrls / httpHookAllowedEnvVars`
- `strictPluginOnlyCustomization`
- `strictKnownMarketplaces / blockedMarketplaces / disableSideloadFlags`
- `extraKnownMarketplaces / pluginSuggestionMarketplaces`
- `otelHeadersHelper`

消费边界：

- `allowManagedHooksOnly` 打开后，只运行 managed settings 里的 hooks。
- `allowedHttpHookUrls` 限制 HTTP hook URL；空数组等价不允许任何 HTTP hook URL。
- `httpHookAllowedEnvVars` 与 hook 自身 `allowedEnvVars` 取交集。
- `allowManagedPermissionRulesOnly` 打开后，user/project/local/CLI/session 的 allow/deny/ask 会整体失效。
- `allowManagedMcpServersOnly` 打开后，allowed MCP server 只读 managed allowlist；deny 仍合并各层，deny 优先。
- `strictPluginOnlyCustomization` 会把 skills/hooks/agents/MCP 等 surface 锁到 plugin-provided 或 managed source。
- `strictKnownMarketplaces / blockedMarketplaces` 在下载前检查 source。
- `disableSideloadFlags` 拒绝 `--plugin-dir`、`--plugin-url`、`--agents`、非 SDK `--mcp-config` 等入口。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:66073-66366`
- `package/preprocessed/cli.extracted.bundle.pretty.js:124140-124260`
- `package/preprocessed/cli.extracted.bundle.pretty.js:164680-165140`
- `package/preprocessed/cli.extracted.bundle.pretty.js:177809-177925`
- `package/preprocessed/cli.extracted.bundle.pretty.js:725398-725640`

## Auto mode classifier、permission 与 sandbox

Auto mode classifier 的 threat model 明确针对拥有 shell、filesystem、API credentials、可长时间运行的 autonomous coding agent。风险类别包括：

- prompt injection
- scope creep
- accidental damage

分类规则含 `HARD BLOCK` 和 `SOFT BLOCK`。重点边界：

- 用户意图是最终信号，但高风险动作需要高证据门槛。
- 工具结果不能被当成用户意图。
- 跨 session message 不能授权危险动作。
- 子代理 delegation、encoded command、wrapper code execution 都要穿透审查。
- classifier bypass attempt 应阻断。
- 两阶段解析失败、请求失败、transcript too long 时偏 fail-closed。

Permission/sandbox 另一层控制：

- `permissions.defaultMode = "auto"` 只有 `policySettings / userSettings / flagSettings` 能授予；project/local 不能授予默认 auto mode。
- Auto mode 维护连续/总 denial 计数，超过阈值时 CLI 退回人工确认，headless/无交互面直接 abort。
- `allowUnsandboxedCommands: false` 会忽略 `dangerouslyDisableSandbox`。
- credentials rules 能 deny credential file/dir read，或 unset 指定 env var。
- sandbox network allow side 可合并 `sandbox.network.allowedDomains` 与 `WebFetch(domain:...)` allow rules。
- `allowManagedDomainsOnly` 开启时，network allow side 只保留 managed 来源；deny side 仍合并所有来源。
- filesystem allow/deny 会合并 `Edit(...)`、`Read(...)` permission rules 与 sandbox settings。
- `allowManagedReadPathsOnly` 开启时，allowRead 只保留 managed `sandbox.filesystem.allowRead`；denyRead 不同步收缩。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:63000-63155`
- `package/preprocessed/cli.extracted.bundle.pretty.js:457284-457366`
- `package/preprocessed/cli.extracted.bundle.pretty.js:457906-458068`
- `package/preprocessed/cli.extracted.bundle.pretty.js:458121-458440`
- `package/preprocessed/cli.extracted.bundle.pretty.js:179337-179358`
- `package/preprocessed/cli.extracted.bundle.pretty.js:718501-718543`
- `package/preprocessed/cli.extracted.bundle.pretty.js:719140-719176`

## Workspace trust 与 helper gate

project/local settings 配置的本地 helper 在 workspace trust 未建立前会被跳过或拒绝：

- `apiKeyHelper`
- `awsAuthRefresh`
- `awsCredentialExport`
- `gcpAuthRefresh`
- MCP `headersHelper`
- `proxyAuthHelper`
- `otelHeadersHelper`

这保护的是“本地命令型配置”。未受信项目不能让 CLI 在 trust dialog 前先执行 credential/header/proxy/OTEL helper。

Project-scoped permission 扩展也会被裁剪：

- `.claude/settings.json` 的 `permissions.additionalDirectories` 会被忽略。
- `.claude/settings.local.json` 的 `permissions.additionalDirectories` 会被忽略。
- 日志记录被丢弃 entry 数量和来源。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:163820-164040`
- `package/preprocessed/cli.extracted.bundle.pretty.js:340222-340235`
- `package/preprocessed/cli.extracted.bundle.pretty.js:96402-96464`
- `package/preprocessed/cli.extracted.bundle.pretty.js:164706-164792`
- `package/preprocessed/cli.extracted.bundle.pretty.js:287242-287276`

## 子进程、脚本与入口防护

外部 channel/plugin 输入会被标记成非用户消息，只能作为 situational awareness。

子进程侧：

- `CLAUDE_CODE_SCRIPT_CAPS` 可限制特定脚本文本的累计调用次数。
- 超限错误明确指向 untrusted-input workflows 中 repeated write exfiltration 的防护。
- `subprocessEnv()` 会删除 OAuth token、subscription/rate-limit tier、background auth/socket token、`OTEL_*` 与若干 provider secret。
- sandbox env scrub 会 deny read/write `.env*`、`.npmrc`、`.yarnrc*`、`.netrc`、`.ssh`、`.config/gh`、git config/hooks、workspace package metadata、CI runner file-command 路径等。

入口侧：

- `--handle-uri <uri>` 后存在额外参数时，判定为 deep-link argument injection 并拒绝。
- Windows UNC path 会触发 WebDAV 风险提示并进入 permission check。
- `cd` 与 `git` 复合命令、bare-repo 指标、`.git` 文件/符号链接指向不可验证位置、命令创建 git 内部文件后再运行 git，都会阻止普通自动放行。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:185-193`
- `package/preprocessed/cli.extracted.bundle.pretty.js:178016`
- `package/preprocessed/cli.extracted.bundle.pretty.js:179905-179934`
- `package/preprocessed/cli.extracted.bundle.pretty.js:179964-180115`
- `package/preprocessed/cli.extracted.bundle.pretty.js:361680-361715`

## Remote Control、bridge 与 Trusted Device

Remote control 背后是一套 environment worker control plane，不只是远程 session 列表。

可见 API：

- `POST /v1/environments/bridge`
- `POST /v1/environments/{environmentId}/bridge/reconnect`
- `GET /v1/environments/{environmentId}/work/poll`
- `POST /v1/environments/{environmentId}/work/{workId}/ack`
- `POST /v1/environments/{environmentId}/work/{workId}/heartbeat`
- `POST /v1/environments/{environmentId}/work/{workId}/stop`
- `DELETE /v1/environments/bridge/{environmentId}`

凭据分层：

- OAuth access token
  - environment register / reconnect / archive / deregister / stop
- `environment_secret`
  - work poll
- `session_ingress_token`
  - work ack / heartbeat / session ingress / permission response
- `worker_jwt`
  - env-less code session bridge transport
- `worker_epoch`
  - v2 transport epoch/version

Trusted Device：

- policy key 是 `require_trusted_devices`。
- feature gate 是 `tengu_sessions_elevated_auth_enforcement`。
- enrollment 调 `POST ${BASE_API_URL}/api/auth/trusted_devices`。
- 成功后保存 `device_token` 到本地 credential store。
- bridge/control-plane headers 可带 `X-Trusted-Device-Token`。
- 服务端返回 untrusted-device 类 403 时，本地会清 cache，并在节流窗口内尝试 lazy enrollment 后重试。

Bridge event attestation：

- event 中的 `device_attestation_status` 会被规范化。
- 默认不强制。
- `tengu_bridge_attestation_enforce` 打开且 Trusted Device policy 生效时，读取 `tengu_bridge_attestation_enforce_config`。
- config 有 `accept_level` 与 `accept_statuses`。
- 达不到要求的 `user` / `control_response` event 可被 drop，并记录 `bridge_event_attestation`。
- 非 user/control_response 或未强制时，不能外推成拒绝。

Remote Control eligibility 不是单个开关。当前本地检查链包括：

- 必须使用官方 Anthropic API 路径，非 `api.anthropic.com` 会拒绝。
- cloud session 内默认不可用。
- managed setting `disableRemoteControl` 可直接禁用。
- 需要 claude.ai subscription auth。
- 长期 token、`claude setup-token` 或 `CLAUDE_CODE_OAUTH_TOKEN` 这种 inference-only token 不满足 full-scope/profile 要求。
- 需要能确定 organization UUID。
- 需要 `policy_limits` / compliance policy 允许。
- 需要 GrowthBook/feature flag 可用；`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`DISABLE_TELEMETRY`、`DO_NOT_TRACK` 或 `DISABLE_GROWTHBOOK` 可能让 eligibility 无法评估。
- `tengu_ccr_bridge` gate 没开时会提示未对账户启用或 feature-flag service 不可达。

这说明 remote-control 封控同时跨 provider/base URL、auth scope、subscription、organization policy、feature flag 和 Trusted Device。重写时不要只检查 `disableRemoteControl`。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:365620-365685`
- `package/preprocessed/cli.extracted.bundle.pretty.js:365768-365934`
- `package/preprocessed/cli.extracted.bundle.pretty.js:682424-682680`
- `package/preprocessed/cli.extracted.bundle.pretty.js:725840-725947`

## Claude Code Gateway 防护

Gateway 不是普通 provider proxy。当前可见防护包括：

- gateway login 要求 gateway host 解析到 private/internal address。
- 经 HTTP proxy 时，proxy host 也要在私网。
- HTTPS gateway 会读取 TLS certificate fingerprint，并写入 trust pin。
- OIDC discovery 的 `jwks_uri / token_endpoint / userinfo_endpoint` 经过 SSRF 防护。
- `fetch` wrapper 阻断 cloud metadata、link-local、loopback、unspecified 等地址。
- DNS 解析后检查每个返回地址。
- gateway server 有 allow/deny CIDR、trusted proxies、max URL/header/body size。
- device authorization 与 verify 有 rate limit。
- 有 CSRF check。
- admin / spend / identity 数据有保留期 cleanup。
- spend limit blocked 分支会在 gateway 层阻断使用。

这层承担企业访问控制、上游模型路由、OIDC device flow、审计/用量控制和 SSRF 防护。它不能证明 Anthropic 主服务端封禁逻辑。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:163140-163224`
- `package/preprocessed/cli.extracted.bundle.pretty.js:163225-163338`
- `package/preprocessed/cli.extracted.bundle.pretty.js:370340-370430`
- `package/preprocessed/cli.extracted.bundle.pretty.js:861527-862580`
- `package/preprocessed/cli.extracted.bundle.pretty.js:863428-864327`
- `package/preprocessed/cli.extracted.bundle.pretty.js:864500-865345`

## Refusal、fallback 与反蒸馏相关 category

服务端 refusal 在本地有三类消费路径：

1. UI/错误消息
   - `stop_reason == "refusal"` 时构造拒绝提示。
   - `cyber` category 满足条件时使用 cybersecurity topic 文案。
   - message 保留 `stop_details`。
2. telemetry
   - `F7t(...)` 打 `api_refusal`。
   - category 白名单包括 `cyber / bio / frontier_llm / reasoning_extraction`。
   - enhanced telemetry 条件满足时才写 category。
3. fallback
   - request body 可加入 server-side fallback 参数。
   - stream 中可能出现 `fallback` content block。
   - fallback 后产生 `model_refusal_fallback` system event。
   - 400 上游不接受 fallback beta/header 时，会 strip 后 retry。

与反蒸馏最相关的是：

- `reasoning_extraction` 与 `frontier_llm` 出现在 refusal category 白名单。
- 客户端会消费这些 category，并可把它们带入 telemetry、system event、UI 文案或 fallback 决策。

边界：

- 本地 bundle 不能证明服务端如何判定 `reasoning_extraction`。
- 本地 bundle 不能证明某个 prompt 会被识别为蒸馏。
- 本地 bundle 不能证明 category 与封禁、计费、人工审核之间的服务端关系。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:282960-283016`
- `package/preprocessed/cli.extracted.bundle.pretty.js:553314-553318`
- `package/preprocessed/cli.extracted.bundle.pretty.js:553496-553515`
- `package/preprocessed/cli.extracted.bundle.pretty.js:561146-561161`
- `package/preprocessed/cli.extracted.bundle.pretty.js:714674-714770`
- `package/preprocessed/cli.extracted.bundle.pretty.js:714843-714904`
- `package/preprocessed/cli.extracted.bundle.pretty.js:838441-838493`

## Server-side fallback 与 fallback credit

Fallback 协议包括：

- `server-side-fallback-2026-06-01`
- `fallback-credit-2026-06-01`
- request body `fallbacks: [{ model }]`
- request body `fallback_credit_token`
- response body `fallback_credit_token` extract 后删除
- Bedrock 形态下 body `anthropic_beta`

Fallback credit token 错误分类：

- malformed
- wrong org
- expired
- invalid model
- extra forbidden
- other

Telemetry：

- `tengu_fallback_credit_minted`
- `tengu_rotunda_pennant_credit_echoed`
- `tengu_fallback_credit_outcome`
- `tengu_rotunda_pennant_strip`

这条线更像拒绝后重试、计费归属、协议兼容与 fallback lane 的控制面。服务端如何 mint token、校验 token、判定 category，仍在 bundle 外。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:122663-122664`
- `package/preprocessed/cli.extracted.bundle.pretty.js:282309-282357`
- `package/preprocessed/cli.extracted.bundle.pretty.js:553314-553341`
- `package/preprocessed/cli.extracted.bundle.pretty.js:714480-715232`
- `package/preprocessed/cli.extracted.bundle.pretty.js:715204-715220`

## Transcript retraction

Server fallback 不只是换模型。客户端要维护 canonical transcript：

- fallback block 可 materialize 成 `fallback` content block，或转成 `server_fallback` internal event。
- fallback 目标模型不在 org `availableModels` allowlist 时，客户端会 decline swap，丢弃 fallback 响应，tombstone 相关消息。
- mid-stream fallback 会 tombstone 已流出的 refused leg。
- retained text 可用 `refusal_continuation` 继续拼接。
- 新 assistant message 可写 `supersedesUuids`。
- `model_refusal_fallback` system event 带 `retractedMessageUuids` 与 `refusedUserMessageUuid`。
- SDK schema 对外映射为 `retracted_message_uuids / refused_user_message_uuid`。
- remote/SDK consumer 验证 retraction signal 来源，只接受 worker source；source missing/mismatch 会打 `tengu_refusal_retraction_unauthenticated_signal`。

重写时这比 UI 文案更关键。拒绝前已经送达的 partial、tool_use、tool_result 不能继续被后续模型当成正常历史。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:558740-558850`
- `package/preprocessed/cli.extracted.bundle.pretty.js:559001-559248`
- `package/preprocessed/cli.extracted.bundle.pretty.js:765506-765615`
- `package/preprocessed/cli.extracted.bundle.pretty.js:837913-838523`

## Model fallback、off-switch 与 consent fallback

普通模型 fallback 的 trigger 包括：

- `model_not_found`
- `permission_denied`
- `overloaded`
- `server_error`
- `last_resort`
- `model_blocked`

`model_blocked` 来自 per-model kill switch / model error override。存在 fallback model 时触发换模型；没有 fallback 时生成 rate-limit 形态错误。

`switchModelsOnFlag` 控制 refusal fallback lane：

- 默认 true。
- 打开时 safety measures flag message 后自动切换模型继续。
- 关闭后 session 暂停。
- 设置变更记录 `tengu_refusal_fallback_setting_changed`。

Fable/extra usage consent 可能产生 `model_consent_fallback`，表示用户拒绝、dismiss，或 consent 后 entitlement 未 provision，session 切回 fallback/default model。

证据锚点：

- `package/preprocessed/cli.extracted.bundle.pretty.js:66714-66720`
- `package/preprocessed/cli.extracted.bundle.pretty.js:280790-280934`
- `package/preprocessed/cli.extracted.bundle.pretty.js:585268-585276`
- `package/preprocessed/cli.extracted.bundle.pretty.js:714400-714415`
- `package/preprocessed/cli.extracted.bundle.pretty.js:716300-716660`
- `package/preprocessed/cli.extracted.bundle.pretty.js:724823-724940`
- `package/preprocessed/cli.extracted.bundle.pretty.js:838506-838533`

## 重写接口建议

重写时建议把封控/遥测拆成这些接口族，不要做成一组全局 if：

```text
PromptMarkerProvider
RequestIdentityHeadersBuilder
RequestMetadataBuilder
TraceContextInjector
RequestLifecycleTelemetry
InternalEventLogger
OtelRuntime
TrafficMode
PolicyLimitsClient
ManagedSettingsPolicy
PermissionDecisionEngine
SandboxPolicyCompiler
WorkspaceTrustGate
SubprocessEnvironmentScrubber
RemoteBridgeCredentialManager
TrustedDeviceClient
BridgeAttestationFilter
GatewaySecurityPolicy
RefusalFallbackCoordinator
TranscriptRetractionManager
```

兼容要点：

- 字段名、header 名、event 名、system event subtype、SDK schema 字段要保留。
- `DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 的影响面要分开。
- `CLAUDE_CODE_ENABLE_TELEMETRY` 只作为标准 3P OTEL exporter 组装开关处理。
- 1P event logging 要独立于 3P OTEL runtime。
- prompt marker 只能实现已确认的 `currentDate` 链。
- `policy_limits` 和 remote managed settings 要 fail-closed 的地方不能放宽。
- workspace trust 需要先于 helper 执行和 project permission 扩展。
- fallback/retraction 要维护 canonical transcript，而不仅是 UI 告警。

## 信号分类与恢复优先级

恢复时可以按“是否会改变行为”给信号分优先级：

| 优先级 | 信号类型 | 代表对象 | 恢复要求 |
| --- | --- | --- | --- |
| P0 | 行为阻断 | policy deny、Auto mode hard block、sandbox deny、workspace trust helper gate、Gateway SSRF/drop、Trusted Device event drop | 必须保留阻断点和错误语义 |
| P0 | transcript correctness | fallback tombstone、`supersedesUuids`、`retracted_message_uuids` | 必须保持 canonical history，不然后续模型会吃到错误上下文 |
| P1 | 请求协议 | headers、`metadata.user_id`、fallback betas、fallback credit echo、traceparent | 必须保留字段名和条件，否则服务端兼容性会漂移 |
| P1 | policy/control-plane cache | remote managed settings、`policy_limits` ETag/stale cache/schema reject | 必须保留 fail-closed 与 stale fallback |
| P2 | 运营/审计 telemetry | `tengu_api_*`、`api_refusal`、startup telemetry、1P events、OTEL | 字段可模块化，但 event name、归因字段和 redaction 边界要稳定 |
| P2 | 产品功能退化 | nonessential traffic 下禁用 feedback、registry、remote-control eligibility、design sync 等 | 用户可见原因要清晰，不能误报成主 API 不可用 |
| P3 | 诊断/调试 | debug auth state、body log file ref、request id lookup text | 可独立实现，但不能泄露 secret |

对“封控”这个词要保持精确：有些信号只是 telemetry 或 correlation，有些是本地执行拦截，有些是服务端 refusal 的消费协议。只有 P0 行为阻断和 transcript correctness 会直接改变客户端执行结果；P1 会影响协议兼容；P2/P3 主要影响可观测性和诊断。

## 不能外推的结论

当前本地证据不能推出：

- Anthropic 服务端一定用 `currentDate`、telemetry 或某个 header 做封号。
- `currentDate` marker 就是完整封控、反蒸馏或账号处罚机制。
- `2.1.91` 是该 marker 的已确认引入版本。
- `reasoning_extraction` 的服务端判定规则已经恢复。
- 某个 prompt 一定会被判为蒸馏。
- telemetry、Gateway、request headers 任一单项能独立识别所有滥用。
- `metadata.user_id`、`x-client-request-id`、`traceparent`、`_PROTO_*` 任一单项就是封禁依据。
- `fallback_credit_token`、`anthropic-usage-limit`、`anthropic-dispatch-id`、`x-anthropic-additional-protection` 任一单项就是处罚依据。
- server-side fallback / fallback-credit 的服务端计费、token mint、token 校验与 category 判定规则已经恢复。
- 本地 bundle 覆盖了所有服务端风控策略。

当前能写实的是：客户端存在多层封控、审计、遥测、policy、设备信任、fallback 和执行约束机制。服务端策略按黑箱接口处理。
