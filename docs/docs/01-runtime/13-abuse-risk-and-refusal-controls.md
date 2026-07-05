# 反滥用、风控与拒绝/fallback 控制面

## 本页用途

- 把 `2.1.197` 本地 bundle 里可见的反滥用、风控、反蒸馏相关机制放到同一页。
- 区分 prompt 内隐写、请求头/telemetry、组织 policy、执行风控、remote control 设备信任、Gateway 防护、模型 refusal/fallback。
- 明确哪些结论能从本地代码证明，哪些仍属于 Anthropic 服务端黑箱。

## 相关文件

- [05-model-adapter-provider-and-auth.md](./05-model-adapter-provider-and-auth.md)
- [06-stream-processing-and-remote-transport.md](./06-stream-processing-and-remote-transport.md)
- [09-api-lifecycle-and-telemetry.md](./09-api-lifecycle-and-telemetry.md)
- [10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md](./10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md)
- [12-settings-and-configuration-system/02-loading-policy-and-merge.md](./12-settings-and-configuration-system/02-loading-policy-and-merge.md)
- [../02-execution/02-instruction-discovery-and-rules.md](../02-execution/02-instruction-discovery-and-rules.md)
- [../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md](../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md)
- [../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md](../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md)
- [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md)

## 一句话结论

`2.1.197` 的风控不是单一“隐藏水印”。本地 bundle 能看到的是一组叠加机制：

- prompt 内只有 `currentDate` 这一条已确认的自然语言隐写链；它能编码 `ANTHROPIC_BASE_URL` host 命中状态与中国时区状态
- 每次模型请求会带固定客户端/session/remote/agent headers，并产生 request lifecycle telemetry
- enterprise/team 控制面通过 remote managed settings 与 `policy_limits` 下发限制
- Auto mode classifier、permission mode、sandbox 和审批后端约束工具执行
- workspace trust gate 会阻止未受信项目/本地配置直接执行 credential/header/proxy helper，并丢弃未受信的 project-scoped permission 扩展
- subprocess env scrub、script call cap、deep-link 参数校验、git/UNC 风险检测，是执行入口和子进程侧的补充防线
- remote-control/bridge 通过 Trusted Device 和 worker secret 收紧远程会话入口
- remote-control event 还可按 `device_attestation_status` 做 feature-flagged drop/filter
- Claude Code Gateway 自带 SSRF、CIDR、rate limit、CSRF、TLS pin、spend limit 等防护
- refusal/fallback 链能识别 `cyber / bio / frontier_llm / reasoning_extraction` 等 category，并把 refusal 转成 UI、telemetry 和 fallback 行为
- quota/overage、server-side fallback、fallback credit、transcript retraction 是一组更隐蔽的协议面，主要用于额度状态、模型切换、拒绝后的重试与历史修正
- OTEL/API body logging 默认会 redact prompt、assistant response 与 thinking 内容，只有显式开关才记录明文

这些机制证明客户端会采集、执行或转发多类风控信号；它们不能直接证明服务端如何判定封禁、如何识别蒸馏、是否有人审或是否把某个信号单独作为处罚依据。以 `2.1.197` 为准可以确认 `currentDate` marker 存在；`2.1.201` 的 npm native 包抽样核对已看不到 `Asia/Shanghai / Asia/Urumqi` 字面量，`currentDate` 附近也恢复成普通 `Today's date is`。`2.1.91` 起点尚未逐版本验证，不能写成已证实的版本边界。

## 机制分层

| 层 | 本地可见机制 | 主要用途 | 证据入口 |
| --- | --- | --- | --- |
| Prompt 内隐写 | `currentDate` 中的 apostrophe 变体、日期格式变体 | 把 base URL host、关键词、时区条件编码进自然语言上下文 | `Wla / rdp / odp / qla` |
| 请求身份面 | `x-app`、`User-Agent`、session id、remote/session/client/agent headers、additional protection header | 标记客户端形态、会话、remote 容器、agent lineage | API client header 组装 |
| Request metadata 面 | `metadata.user_id` JSON、extra metadata、device/account/session id | 把设备、账号、会话和调用方附加 metadata 放进 API payload | `XLe()` |
| Telemetry 面 | `tengu_api_query/error/success`、`api_refusal`、gateway classifier、base URL hash | 记录请求生命周期、错误、成功、refusal、gateway/proxy 指纹 | `NDl / KOo / F7t / UDl` |
| Trace/correlation 面 | `x-client-request-id`、`traceparent`、`previousRequestId`、`queryTracking` | 对齐客户端日志、服务端日志、trace 与相邻请求链 | `Apc / XLa / Dlm` |
| 隐式 header/beta 面 | `anthropic-usage-limit`、`anthropic-dispatch-id`、`cache-diagnosis`、`server-side-fallback`、`fallback-credit` | 灰度 dispatch、额度扩展、缓存诊断、fallback 协议协商 | request build / 400 strip-retry |
| Quota/overage 面 | `quota_check` probe、unified rate-limit headers、overage consent/cache | 探测账号额度、超额使用状态、fast mode / Fable usage credit gate | `Nfp / Oda / Bda / Pai` |
| 组织 policy 面 | `policy_limits`、remote managed settings、`compliance_taints`、monitoring notice | org capability gate、合规 taint、远端托管配置 | `policy_limits` fetch/cache/schema |
| 执行风控面 | Auto mode classifier、permission mode、sandbox、approval backend | 阻断 prompt injection、scope creep、危险工具调用、未授权执行 | classifier prompt 与两阶段 XML classifier |
| Workspace trust/helper 面 | `apiKeyHelper`、cloud auth refresh/export、MCP/proxy/OTEL helper、project-scoped permission 扩展 | 防止未受信项目配置先于 trust dialog 执行本地命令或扩大权限 | helper trust checks / settings merge |
| 子进程与入口面 | deep-link 参数注入检测、subprocess env scrub、script call cap、UNC/WebDAV/git-dir 风险 gate | 收紧 OS protocol、子进程环境、脚本重复写入和 git 命令自动放行 | CLI argv guard / subprocess env / command classifier |
| Remote control 面 | bridge credentials、work secret、Trusted Device token | 限制谁能暴露本地 worker、谁能接管 remote-control session | bridge lifecycle 与 enrollment |
| Bridge attestation 面 | `device_attestation_status`、`tengu_bridge_attestation_enforce(_config)` | 对 remote-control user/control_response event 做设备证明过滤 | bridge event filter |
| Gateway 面 | private network gate、TLS fingerprint pin、SSRF/DNS 防护、rate limit、CSRF、CIDR、payload size、spend limit | 保护企业 gateway、自建 provider 入口和 OIDC/device flow | gateway/server 模块 |
| Refusal/fallback 面 | refusal stop details、fallback model、fallback credit lane、strip/retry、retraction/supersedes | 响应模型/服务端 safety refusal，必要时切模型、回收 partial、修正 transcript | streaming request build 与 fallback handlers |

## 本地动作类型表

这些字段族的恢复口径按本地动作分层，不按“是否导致封禁”分层：

| 字段族 | 本地动作类型 | 边界 |
| --- | --- | --- |
| `currentDate` apostrophe/date 变体 | 本地生成、本地发送 | 进入 `userContext.currentDate`；服务端解释不可证明 |
| API client/session/remote/agent headers | 本地生成、本地发送 | 标记客户端、会话、remote 与 agent lineage；不等于单项处罚依据 |
| `metadata: XLe()` / `metadata.user_id` JSON | 本地生成、本地发送 | 合并 extra metadata、device/account/session id；字段名不是裸 user id |
| `x-client-request-id` / `traceparent` / `previousRequestId` | 本地生成、本地发送、本地记录 | 用于 correlation；trace 注入受 first-party/显式开关限制 |
| `anthropic-usage-limit` / `anthropic-dispatch-id` / `x-anthropic-additional-protection` | 本地生成、本地发送、条件 strip/retry | 受 env、feature flag、first-party/base URL 条件影响；服务端语义不可证明 |
| `cache-diagnosis` / `server-side-fallback` / `fallback-credit` beta | 本地发送、本地消费、条件 strip/retry | 作为协议能力协商；400 兼容分支不能外推为安全判定 |
| quota / unified rate-limit / overage headers | 本地发送、本地消费、本地记录 | 主要影响额度状态、fast mode cooldown、extra usage/consent UI |
| `policy_limits` | 本地消费、本地执行、本地记录 | fetch/cache/schema 后影响 capability gate；server policy 生成规则不可证明 |
| remote managed settings policy keys | 本地消费、本地执行 | 影响 models/hooks/MCP/plugin/permission/marketplace/sideload 等能力面 |
| Auto mode classifier / permission / sandbox | 本地执行、本地记录 | 约束工具执行；不是服务端 refusal category |
| workspace trust helper checks | 本地执行、本地记录 | gate credential/header/proxy/OTEL helper 与 project-scoped permission 扩展 |
| subprocess env scrub / script cap / deep-link/UNC/git guards | 本地执行、本地记录 | 约束子进程、OS 入口和命令分类；不进入模型 prompt 主链 |
| Trusted Device / bridge `device_attestation_status` | 本地发送、本地消费、本地执行 | 限制 remote-control 事件或 enrollment；服务端设备策略仍按黑箱处理 |
| Gateway SSRF/CIDR/TLS/rate-limit/CSRF/spend limit | 本地执行 | bundled gateway/client 侧可见防护；上游运营策略不可证明 |
| refusal/fallback/retraction 字段 | 本地消费、本地执行、本地记录 | 处理 UI、telemetry、model switch、tombstone/retraction；category 判定规则不可证明 |

## Prompt 内隐写：目前只确认 `currentDate`

本地可见的隐写链很窄：

```text
Wla(base64)
  -> XOR 91
  -> split(",")
  -> edp()/tdp() 得到 domain list / keyword list
  -> rdp() 检查 ANTHROPIC_BASE_URL host 与 timezone
  -> odp(known, labKw) 选择 apostrophe 变体
  -> qla(date) 生成 Today?s date is ...
  -> currentDate: qla(GSe())
```

关键点：

- `Wla(...)` 用 base64 + XOR 91 解出两组名单。
- `rdp()` 读取 `ANTHROPIC_BASE_URL` host，并额外检查 timezone 是否是 `Asia/Shanghai` 或 `Asia/Urumqi`。
- `odp(...)` 根据 `known / labKw` 两个布尔值，在普通 apostrophe、right single quote、modifier letter apostrophe、modifier letter prime 之间切换：`' / ’ / ʼ / ʹ`。
- `qla(...)` 会在中国时区下把日期里的 `-` 改成 `/`。
- 这相当于最多 3 bit 的本地 marker：`known` 1 bit、`labKw` 1 bit、`cnTZ` 1 bit。apostrophe 承载前两个 bit，日期分隔符承载时区 bit。
- 最终文本进入 `userContext.currentDate`，再由 `otr(...)` 包成前置 `<system-reminder>` user meta message；它不是直接拼到 API request 的 top-level `system` 字段。

当前已做过的同类排查结论仍成立：请求头、telemetry、policy limits、Trusted Device、model off-switch 都和反滥用有关，但它们不是“藏进自然语言 prompt 的隐写”。同类 prompt 内隐写目前只确认 `currentDate`。服务器是否解码该 marker、是否把它用于封控或反蒸馏策略，仍属于本地 bundle 外的服务端黑箱。

直接证据：

- `package/preprocessed/cli.extracted.bundle.pretty.js:274124-274175`
- `package/preprocessed/cli.extracted.bundle.pretty.js:279103`
- `package/preprocessed/cli.extracted.bundle.pretty.js:713236-713254`
- [../02-execution/02-instruction-discovery-and-rules.md](../02-execution/02-instruction-discovery-and-rules.md)

## 请求头与 telemetry 指纹

模型 API client 会统一组装一组请求头：

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

这说明请求身份面不只区分“是不是 Claude Code”，还会携带 background、remote session、SDK client app、agent/subagent lineage 等上下文。

其中后三个需要分开看：

- `x-anthropic-additional-protection: true` 只在 `CLAUDE_CODE_ADDITIONAL_PROTECTION` 打开时加入。当前 bundle 只能证明客户端会加 header，不能证明服务端具体怎么解释它。
- `anthropic-usage-limit: extended` 只在 query depth 大于 0、first-party、官方 base URL、非普通 auxiliary 或 compact、且 `tengu_lantern_spool` feature flag 打开时加入。它更像额度/链路提示，不是 prompt 隐写。
- `anthropic-dispatch-id: v2s` 只在 first-party、官方 base URL、非 auxiliary、且 `tengu_cedar_lattice` feature flag 打开时加入。stream 在首个事件前因该 header 失败时会 retry without it，并打 `tengu_dispatch_header_fallback`。

request telemetry 又分成三段：

- `NDl(...)` 在请求发出前打 `tengu_api_query`，记录 model、message count、betas、permissionMode、querySource、queryTracking、thinking/effort、fastMode、previousRequestId、provider、buildAge、环境模型配置等。
- `KOo(...)` 在最终失败后打 `tengu_api_error`，记录 error type/status、requestId、clientRequestId、retry/fallback、gateway 分类、querySource、permissionMode 相关上下文。
- `UDl(...) -> _Rf(...)` 在最终成功后打 `tengu_api_success`，记录 token、cost、duration、attempt、requestId、stop_reason、permissionMode、cache strategy、文本/thinking/tool/result 尺寸、gateway、previousRequestId 等。

base URL 也会被处理成 telemetry 指纹：

- 官方 Anthropic / Claude host 保留规范化 URL。
- 非官方 host 走 hash，只保留稳定短指纹。
- gateway/proxy 识别还能通过响应/request headers 的 prefix 区分 `litellm / helicone / portkey / cloudflare-ai-gateway / kong / braintrust` 等。

直接证据：

- request headers：`package/preprocessed/cli.extracted.bundle.pretty.js:161719-161738`
- `anthropic-usage-limit / anthropic-dispatch-id`：`package/preprocessed/cli.extracted.bundle.pretty.js:715190-715207`、`716335-716338`、`717157-717158`
- base URL / gateway classifier：`package/preprocessed/cli.extracted.bundle.pretty.js:560484-560514`、`560547-560577`、`560649-560678`、`560872-560928`、`561146-561161`
- refusal telemetry：`package/preprocessed/cli.extracted.bundle.pretty.js:560721-560749`
- [09-api-lifecycle-and-telemetry.md](./09-api-lifecycle-and-telemetry.md)

## Request metadata、trace correlation 与轻量探测

除了 headers，API payload 自身还有一个容易漏掉的 metadata 通道。`Hpc(...)` 构造 request body 时会设置：

```text
metadata: XLe()
```

`XLe()` 会返回：

```text
{ user_id: JSON.stringify({
    ...CLAUDE_CODE_EXTRA_METADATA,
    device_id,
    account_uuid,
    session_id
  }) }
```

这里的几个边界很重要：

- `CLAUDE_CODE_EXTRA_METADATA` 必须是 JSON object，否则只记错误日志。
- `device_id` 来自本地设备标识。
- `account_uuid` 优先取 remote/account 环境或当前账号状态。
- `session_id` 来自当前 Claude Code session id。
- 字段名仍叫 `user_id`，但内容是一个 JSON 字符串，不是单个裸 user id。

请求外层还有 trace/correlation 通道：

- first-party 或指定路径会生成 `x-client-request-id`。
- 如果有 llm span，且路径允许，会带 `traceparent`。
- `previousRequestId` 会从 transcript 里最近的 assistant request id 回补，用来串起相邻请求。
- `queryTracking` 记录 chain id / depth，用来还原请求链层级。

另外还有一类“轻量探测”不属于主对话 prompt：

- quota/rate-limit probe 会发一个 `source: "quota_check"` 的最小模型请求。
- probe body 是 `messages: [{ role: "user", content: "quota" }]`、`max_tokens: 1`，并同样带 `metadata: XLe()`。
- 这类请求主要读取 quota / rate-limit headers。
- 它属于账号/额度状态探测，不是用户 prompt 的隐藏内容。

rate-limit / overage 解析的 header 包括：

- `anthropic-ratelimit-unified-representative-claim`
- `anthropic-ratelimit-unified-overage-status`
- `anthropic-ratelimit-unified-overage-disabled-reason`
- `anthropic-ratelimit-unified-5h-utilization / reset`
- `anthropic-ratelimit-unified-7d-utilization / reset`
- `anthropic-ratelimit-unified-7d_oi-utilization / reset`
- `anthropic-ratelimit-unified-overage-utilization / reset`

这些字段会进入 `rate_limit_event.rate_limit_info`、fast mode cooldown、extra usage/Fable overage consent 和 UI 选择面。`overageDisabledReason` 的枚举包括 `overage_not_provisioned / org_level_disabled / org_level_disabled_until / out_of_credits / seat_tier_level_disabled / member_level_disabled / seat_tier_zero_credit_limit / group_zero_credit_limit / member_zero_credit_limit / org_service_level_disabled / no_limits_configured / fetch_error / unknown`。

直接证据：

- `XLe()`：`package/preprocessed/cli.extracted.bundle.pretty.js:713876-713898`
- quota request 与 unified headers：`package/preprocessed/cli.extracted.bundle.pretty.js:281414-281496`
- rate-limit event schema：`package/preprocessed/cli.extracted.bundle.pretty.js:837840-838004`
- client request id / traceparent：`package/preprocessed/cli.extracted.bundle.pretty.js:714082-714098`
- [../02-execution/03-prompt-assembly-and-context-layering/04-request-level-injection-layers-and-local-server-boundary.md](../02-execution/03-prompt-assembly-and-context-layering/04-request-level-injection-layers-and-local-server-boundary.md)
- [06-stream-processing-and-remote-transport.md](./06-stream-processing-and-remote-transport.md)

## `policy_limits`、remote managed settings 与合规 taint

enterprise/team 风控至少有两条远端链：

- remote managed settings
  - endpoint: `/api/claude_code/settings`
  - 产物进入 `policySettings`
  - 再参与 effective settings merge
- `policy_limits`
  - endpoint: `/api/claude_code/policy_limits`
  - 产物不走普通 settings merge
  - 直接提供 `restrictions / compliance_taints / monitoring_notice / defaults`

`policy_limits` 的本地 schema 能说明它不是普通 feature flag：

- `restrictions` 是逐 capability 的 allow/deny。
- `compliance_taints` 会额外映射到 capability deny，即使没有显式 restriction 也可能拒绝。
- `monitoring_notice.text` 会移除控制字符，`url` 只接受 `https://`。
- `defaults` 可作为 policy default 值来源。

它的 eligibility gate 也很明确：

- 只在 `firstParty` provider 下启用。
- 自定义 base URL 默认不合格。
- 需要 first-party API key、WIF、或带 inference scope 的 OAuth。
- prosumer OAuth、无 auth、无 inference scope 会被判为不合格。

fetch/cache 行为包括：

- 本地用 canonical JSON 计算 `sha256:<hex>`，放进 `If-None-Match`。
- `304` 使用缓存，`404` 视为无限制。
- response 要过 schema 校验。
- 失败时有旧缓存会使用 stale cache。
- fetch 结果会打 `tengu_policy_limits_fetch`。

remote managed settings 还提供一组更像企业风控/合规管理的键：

- `availableModels / enforceAvailableModels / modelOverrides`
- `forceLoginOrgUUID / forceLoginMethod / forceRemoteSettingsRefresh`
- `disableRemoteControl / disableAgentView / disableWorkflows / disableArtifact`
- `allowManagedHooksOnly / allowManagedPermissionRulesOnly / allowManagedMcpServersOnly`
- `allowedHttpHookUrls / httpHookAllowedEnvVars`
- `strictPluginOnlyCustomization`
- `strictKnownMarketplaces / blockedMarketplaces / disableSideloadFlags`
- `extraKnownMarketplaces / pluginSuggestionMarketplaces`
- `otelHeadersHelper`

`enforceAvailableModels` 的 fail-close 边界也值得保留：policy source 存在但加载失败时，代码会拒绝 cascade-trust 的 model enforcement，不让 user/project settings 冒充可信 policy；`forceLoginOrgUUID` 在读不到可能相关的 managed policy 时也会 fail-close，要求用户联系管理员。

这些键不是只进 schema，存在明确消费点：

- `allowManagedHooksOnly` 打开后，只运行 managed settings 里的 hooks；user/project/local hooks 会被忽略。
- `allowedHttpHookUrls` 限制 HTTP hooks 的目标 URL；空数组等价于不允许任何 HTTP hook URL。
- `httpHookAllowedEnvVars` 会和单个 hook 的 `allowedEnvVars` 取交集，未列入的环境变量不会被插入 header。
- `allowManagedPermissionRulesOnly` 打开后，只尊重 managed permission rules；user/project/local/CLI 参数里的 allow/deny/ask 不再参与。
- `allowManagedMcpServersOnly` 打开后，`allowedMcpServers` 只读 managed allowlist，但 `deniedMcpServers` 仍会从各层合并，deny 优先。
- `strictPluginOnlyCustomization` 可把 skills/hooks/agents/MCP 等 surface 锁到 plugin-provided 或 managed source，阻断 `~/.claude/{surface}/`、`.claude/{surface}/`、settings hooks、`.mcp.json` 这类非 plugin customization。
- `strictKnownMarketplaces` / `blockedMarketplaces` 在下载前检查 source；blocked source 不会落盘。
- `disableSideloadFlags` 会拒绝 `--plugin-dir`、`--plugin-url`、`--agents` 和非 SDK `--mcp-config` 这类 CLI sideload 入口；只在 managed settings 下生效。
- `pluginSuggestionMarketplaces` 还会限制 marketplace 声明的 contextual install suggestions；allowlisted name 必须和 managed source 对上，官方 marketplace 有单独豁免。

直接证据：

- managed settings schema：`package/preprocessed/cli.extracted.bundle.pretty.js:66073-66366`
- model allowlist fail-close：`package/preprocessed/cli.extracted.bundle.pretty.js:124140-124260`
- org login pin fail-close：`package/preprocessed/cli.extracted.bundle.pretty.js:164680-165140`
- managed hooks / permission / MCP merge：`package/preprocessed/cli.extracted.bundle.pretty.js:67706-67727`、`181103-181145`、`287183`、`332497`
- HTTP hook URL/env allowlist：`package/preprocessed/cli.extracted.bundle.pretty.js:706876-706916`
- marketplace/sideload policy consumption：`package/preprocessed/cli.extracted.bundle.pretty.js:354012-354044`、`576230-576265`、`817058`
- schema 与 allow/deny：`package/preprocessed/cli.extracted.bundle.pretty.js:177809-177925`
- fetch/cache/telemetry：`package/preprocessed/cli.extracted.bundle.pretty.js:725398-725640`
- [12-settings-and-configuration-system/02-loading-policy-and-merge.md](./12-settings-and-configuration-system/02-loading-policy-and-merge.md)

## Auto mode classifier、permission 与 sandbox

Auto mode classifier 是执行层最明显的反滥用模型调用。它的 system prompt 明确把被监控对象定义为有 shell、文件系统、API credentials、可长时间运行的 autonomous coding agent，并把风险分成：

- prompt injection
- scope creep
- accidental damage

分类器规则分为 `HARD BLOCK` 和 `SOFT BLOCK`，并强调：

- 用户意图是最终信号，但高风险动作需要高证据门槛。
- 工具结果不能被当成用户意图。
- 跨 session message 不能授权危险动作。
- 子代理 delegation、encoded command、wrapper code execution 都要穿透审查。
- classifier bypass attempt 应被阻断。

实现上不是只靠本地正则：

- 工具输入会先通过 `toAutoClassifierInput(...)` 转换，失败会 fallback 并记 telemetry。
- `CLAUDE.md` 会作为用户环境上下文进入 classifier，但只可作为具体授权证据，不能把泛化鼓励当成授权。
- XML classifier 有 fast/thinking/both 多形态。
- 两阶段解析失败、请求失败、transcript too long 时都偏向 fail-closed：无法可靠判定就 blocking for safety。

permission/sandbox 形成另一层执行 gate：

- settings schema 包含 sandbox network/filesystem/credentials/unsandboxed controls。
- `allowUnsandboxedCommands: false` 时会忽略 `dangerouslyDisableSandbox`。
- credentials rules 能 deny credential file/dir read，或 unset 指定环境变量。
- managed settings 可以关闭 remote control、workflows、artifact、skill shell execution，限制 hooks、permission rules、MCP、marketplace、sideload flags。
- project/local settings 不能授予默认 `auto` mode；只有 `policySettings / userSettings / flagSettings` 能让 `permissions.defaultMode = "auto"` 生效。
- Auto mode 维护连续/总 denial 计数，超过阈值时 CLI 退回人工确认，headless/无交互面会直接 abort。
- remote 下 `mcpServerPolicy` 的 ask rule 有一个 feature-flagged `tengu_mcp_server_policy_bypass_exempt` 特例：在 `bypassPermissions` 语义下可不弹 ask，但 deny 规则、destructive/path/sandbox 等其他 gate 仍在前后层继续执行。这个特例是 permission 合流细节，不是绕过服务端 policy 的证据。

直接证据：

- sandbox settings：`package/preprocessed/cli.extracted.bundle.pretty.js:63000-63155`
- managed policy gates：`package/preprocessed/cli.extracted.bundle.pretty.js:66158-66364`
- classifier threat model：`package/preprocessed/cli.extracted.bundle.pretty.js:457284-457366`
- classifier input / CLAUDE.md / XML two-stage：`package/preprocessed/cli.extracted.bundle.pretty.js:457906-458068`、`458121-458440`
- default auto mode trust gate：`package/preprocessed/cli.extracted.bundle.pretty.js:179337-179358`
- denial limit 与 headless abort：`package/preprocessed/cli.extracted.bundle.pretty.js:718501-718543`、`719140-719176`
- MCP server policy ask 特例：`package/preprocessed/cli.extracted.bundle.pretty.js:718642-718883`
- [../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md](../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md)
- [../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md](../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md)

## Workspace trust、helper 执行与子进程防护

还有一组不直接出现在模型 request 里的边缘防线，容易漏掉。它们的共同点是：配置文件、外部输入、OS deep-link 或子进程环境只要可能把“非用户明确授权的内容”变成命令执行、凭据读取、网络转发或权限扩展，就会先经过 trust/scrub/cap。

### 未受信 workspace 下的 helper gate

project/local settings 配置的 helper 在 workspace trust 未建立前会被跳过或拒绝：

- `apiKeyHelper` 在 project/local 来源下，如果 `workspace trust` 未确认，返回 `null`，并打 `tengu_apiKeyHelper_missing_trust11`。
- `awsAuthRefresh`、`awsCredentialExport`、`gcpAuthRefresh` 在同类条件下不会执行，并分别打 `tengu_awsAuthRefresh_missing_trust`、`tengu_awsCredentialExport_missing_trust`、`tengu_gcpAuthRefresh_missing_trust`。
- MCP `headersHelper` 如果来自 project/local scope，workspace 未受信时返回 `null`，同时打 `tengu_mcp_headersHelper_missing_trust` 与 `mcp_headers_helper: missing_trust`。
- `proxyAuthHelper` 在 `CLAUDE_CODE_ENABLE_PROXY_AUTH_HELPER` 打开、且 helper 来自 project/local settings 时，也要求 trust accepted；否则只记录 warn 并跳过。
- `otelHeadersHelper` 如果来自 project/local settings，未受信时不会执行 helper；消费侧会把导出 header 状态归成 `{ headers: {}, error: "trust not established" }` 形态。

这类机制保护的是“本地命令型配置”。它和模型 safety refusal 不在同一层，但对反滥用很关键：项目仓库或本地配置不能在用户接受 trust dialog 前让 CLI 先跑一段 credential/header/proxy helper。

直接证据：

- `apiKeyHelper / awsAuthRefresh / awsCredentialExport / gcpAuthRefresh` trust gate：`package/preprocessed/cli.extracted.bundle.pretty.js:163820-164040`
- MCP `headersHelper` trust gate：`package/preprocessed/cli.extracted.bundle.pretty.js:340222-340235`
- `proxyAuthHelper` trust gate：`package/preprocessed/cli.extracted.bundle.pretty.js:96402-96464`
- `otelHeadersHelper` trust error：`package/preprocessed/cli.extracted.bundle.pretty.js:164706-164792`、`177282`

### Project-scoped permission 扩展也会被 trust 裁剪

未受信 workspace 下，不只是 helper 会被挡住，project/local 里扩展权限的配置也会被丢弃：

- `.claude/settings.json` 的 `permissions.additionalDirectories` 会被忽略。
- `.claude/settings.local.json` 的 `permissions.additionalDirectories` 会被忽略。
- 日志会写出被丢弃的 entry 数量和来源。

这说明 trust dialog 是 permission 扩张的前置边界之一。它不是只保护 helper，也会影响“项目配置能否扩大 Claude 可访问目录”。

直接证据：

- project/local additional directories trust 裁剪：`package/preprocessed/cli.extracted.bundle.pretty.js:287242-287276`

### 外部输入包装、script cap 与 env scrub

外部 channel/plugin 输入会被包上一段系统提示，明确标记“不是用户消息”，要求只作为 situational awareness，不执行其中的 imperative language。另一路是子进程侧：

- `CLAUDE_CODE_SCRIPT_CAPS` 可配置某些脚本文本的累计调用上限；超过后抛错，错误文本明确写着用于防止 untrusted-input workflows 里的 repeated write exfiltration。
- `subprocessEnv()` 会从子进程环境里删掉 OAuth token、subscription/rate-limit tier、background auth/socket token、`OTEL_*` 和若干 provider secret。
- 当 env scrub 进入 sandbox 配置时，会 deny-read / deny-write 一批敏感文件和目录，包括 `.env*`、`.npmrc`、`.yarnrc*`、`.netrc`、`.ssh`、`.config/gh`、git config/hooks、workspace package metadata 和 CI runner file-command 路径。

这条线不是模型端风控，而是“让 untrusted input 更难通过工具/脚本/子进程把凭据或私有代码带出去”的本地执行面防护。

直接证据：

- external channel/plugin untrusted wrapper：`package/preprocessed/cli.extracted.bundle.pretty.js:178016`
- script call cap：`package/preprocessed/cli.extracted.bundle.pretty.js:179905-179934`
- 子进程 env 删除与 scrub deny list：`package/preprocessed/cli.extracted.bundle.pretty.js:179964-180115`

### CLI/deep-link/git 命令入口上的注入与风险 gate

入口层还存在几处小但有价值的防线：

- `--handle-uri <uri>` 后如果还有额外参数，会被判为 deep-link argument injection，直接拒绝。
- 命令里出现 Windows UNC path 时，会提示可能存在 WebDAV 攻击风险，并转入需要权限检查的路径。
- 复合命令里 `cd` 与 `git` 同时出现、当前目录有 bare-repo 指标、`.git` 文件/符号链接指向不可验证位置、或命令创建 git 内部文件后再运行 git，都会阻止普通自动放行，改走 permission check。

这些检查属于入口/命令分类防护，覆盖的是“从协议处理器、压缩包/恶意 repo、UNC/WebDAV、复合 shell 命令进入工具执行”的边缘风险。

直接证据：

- deep-link argument injection guard：`package/preprocessed/cli.extracted.bundle.pretty.js:185-193`
- UNC/WebDAV 与 git-dir 风险 gate：`package/preprocessed/cli.extracted.bundle.pretty.js:361680-361715`

## 1P event attribution 与 `_PROTO_*` 保留字段

1P event logging 还有一层归因字段，不直接进入模型请求，但属于反滥用/审计时很重要的上下文。

log record 会被转换成 `ClaudeCodeInternalEvent`，其中：

- `device_id`、`email`、auth、env、process、core metadata 会被提升或结构化。
- `_PROTO_skill_name`
- `_PROTO_plugin_name`
- `_PROTO_marketplace_name`
- `_PROTO_code`
- `_PROTO_head_sha`

这些 `_PROTO_*` 是保留字段。转换时会把 skill/plugin/marketplace/code/head sha 从普通 `additional_metadata` 里拆出来，提升成顶层字段；剩余 additional metadata 才会 base64 JSON 编码。

这说明 telemetry/event logging 对“是谁触发的、哪个 skill/plugin/marketplace 触发的、是否带 REPL code/head sha”有专门归因面。它不是 prompt 隐写，也不是模型 API payload 的 `metadata.user_id`，而是 1P event logging 的独立审计/分析通道。

直接证据：

- event transform 与 `_PROTO_*` 提升：`package/preprocessed/cli.extracted.bundle.pretty.js:176520-176596`
- [10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md](./10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md)

## OTEL/API body logging 的 redaction 边界

当前 bundle 里还能看到一条隐私保护线，容易和风控 telemetry 混在一起：

- OTEL prompt 默认写 `<REDACTED>`，只有 `OTEL_LOG_USER_PROMPTS` 打开时才记录明文。
- assistant response 默认写 `<REDACTED>`，只有 `OTEL_LOG_ASSISTANT_RESPONSES` 或 `OTEL_LOG_USER_PROMPTS` 打开时才记录明文。
- API body logging 会把 assistant history 里的 `thinking.thinking` 和 `redacted_thinking.data` 替换成 `<REDACTED>`。
- API body logging 有 60KB 截断边界。

这说明本地日志默认不会把用户 prompt、assistant response、thinking 明文直接打进 OTEL/body log。它是隐私/审计边界，不是模型 refusal category，也不能证明服务端如何做反蒸馏。

直接证据：

- OTEL prompt/assistant redaction 开关：`package/preprocessed/cli.extracted.bundle.pretty.js:232303-232310`、`561068-561075`
- thinking/redacted_thinking body redaction：`package/preprocessed/cli.extracted.bundle.pretty.js:560431-560432`

## Remote Control、bridge 与 Trusted Device

remote-control 不是一个简单远程开关。当前可见链路至少包括：

- environment registration / deregistration
- work poll / ack / heartbeat / stop
- `environment_secret -> work.secret -> session_ingress_token`
- session ingress
- permission response events
- bridge reconnect

Trusted Device 是 remote-control 的额外设备信任层：

- policy key 是 `require_trusted_devices`。
- feature gate 是 `tengu_sessions_elevated_auth_enforcement`。
- enrollment 调 `POST ${BASE_API_URL}/api/auth/trusted_devices`。
- 成功后保存 `device_token` 到本地 credential store。
- bridge/control-plane headers 会在可用时带 `X-Trusted-Device-Token`。
- 服务端返回 untrusted-device 类 403 时，本地会清 cache，并在节流窗口内尝试 lazy enrollment 后重试。

还有一个更隐蔽的 remote event filter：

- bridge event 带 `device_attestation_status` 时会规范化成 attestation status。
- 默认 policy 是不强制；`tengu_bridge_attestation_enforce` 打开且 Trusted Device policy 生效时，才读取 `tengu_bridge_attestation_enforce_config`。
- config 里有 `accept_level` 与 `accept_statuses`。达不到 `accept_level` 的 `user` / `control_response` event 会被记录为接受、config exception 或 drop。
- drop 时会打 `bridge_event_attestation` 失败指标，并调用 drop notifier；非 user/control_response 或未强制时不会简单外推成拒绝。

这类机制约束的是 remote worker 暴露、远程接管入口和 remote event 的设备证明状态，属于显式设备信任/组织 policy，不是 prompt 隐写。

直接证据：

- bridge event attestation filter：`package/preprocessed/cli.extracted.bundle.pretty.js:365620-365685`
- Trusted Device gate/enrollment：`package/preprocessed/cli.extracted.bundle.pretty.js:365768-365934`
- bridge register/poll/ack/stop/heartbeat：`package/preprocessed/cli.extracted.bundle.pretty.js:682424-682680`
- [10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md](./10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md)

## Claude Code Gateway 防护

Gateway 相关代码里能看到一整套企业入口防护：

- gateway login 要求 gateway host 解析到 private/internal address；如果经 HTTP proxy，proxy host 也必须在私网。
- HTTPS gateway 会读取 TLS certificate fingerprint，并写入本地 trust pin；后续用 pin 校验证书。
- OIDC discovery 的 `jwks_uri / token_endpoint / userinfo_endpoint` 会过 SSRF 防护。
- `fetch` wrapper 会阻断 cloud metadata、link-local、loopback、unspecified 等地址，DNS 解析后也会检查每个返回地址。
- gateway server 有 allow/deny CIDR、trusted proxies、max URL/header/body size、device authorization rate limit、device verify rate limit、CSRF check。
- admin / spend / identity 相关数据有保留期 cleanup，spend limit blocked 分支会把使用者挡在 gateway 层。

这说明 Gateway 不是普通 provider proxy。它同时承担 OIDC device flow、企业访问控制、上游模型路由、审计/用量控制和 SSRF 防护。

直接证据：

- private network gate 与 TLS pin：`package/preprocessed/cli.extracted.bundle.pretty.js:163140-163224`、`163225-163338`
- remote settings pin/cache/error：`package/preprocessed/cli.extracted.bundle.pretty.js:370340-370430`
- SSRF/DNS/OIDC endpoint 防护：`package/preprocessed/cli.extracted.bundle.pretty.js:861527-862580`
- spend limit blocked：`package/preprocessed/cli.extracted.bundle.pretty.js:863428-864327`
- protocol/rate limit/CSRF/CIDR/size limits：`package/preprocessed/cli.extracted.bundle.pretty.js:864500-865345`

## Refusal、fallback 与反蒸馏相关 category

服务端 refusal 在本地会进入三类路径：

1. UI/错误消息
   - `stop_reason == "refusal"` 时，客户端会构造面向用户的拒绝提示。
   - category 为 `cyber` 且满足特定条件时，会给出 cybersecurity topic 专门文案。
   - message 上保留 `stop_details`，并把 `message.stop_reason` 设置为 `refusal`。
2. telemetry
   - `F7t(...)` 打 `api_refusal`。
   - category 白名单包括 `cyber / bio / frontier_llm / reasoning_extraction`。
   - 只有 enhanced telemetry 条件满足时才写 category。
3. fallback
   - request body 可合入 server-side fallback 参数。
   - stream 中可能出现 `fallback` content block。
   - fallback 后会产生 `model_refusal_fallback` system event。
   - 400 上游不接受 fallback beta/header 时，会 strip 后 retry。

这里和“反蒸馏”最相关的是两点：

- `reasoning_extraction` 和 `frontier_llm` 已出现在 refusal category 白名单里。
- fallback/refusal 链能把这类 category 带入 telemetry、system event、UI 文案或后续 retry 决策。

但本地 bundle 不能证明服务端如何判定 `reasoning_extraction`，也不能证明“某个 prompt 会被当成蒸馏请求”的具体规则。可写的结论只能是：客户端已经为这些 refusal category 预留并消费了协议字段。

### `server-side-fallback` 与 `fallback-credit` 协议面

新增的隐蔽点主要在 fallback 协议：

- beta 常量包括 `server-side-fallback-2026-06-01` 与 `fallback-credit-2026-06-01`。
- `xRl(...)` 会在符合条件时给 request body 加 `fallbacks: [{ model }]`，并把 `server-side-fallback` beta 加进 sticky/request beta 集合。
- `RRl(...)` 会为 fallback credit lane 补 `fallback-credit` beta；Bedrock 形态下还会把 beta 写进 body 的 `anthropic_beta`。
- non-streaming fallback response 里如果出现 `fallback_credit_token`，客户端会读取 token 后从 response body 删除，避免 token 留在普通 message body 中。
- streaming retry 时，客户端会把 `fallback_credit_token` 作为 top-level request body 参数发出。
- fallback credit token 有专门错误分类：malformed、wrong org、expired、invalid model、extra forbidden、other；400 会触发 strip/retry 或 credit forfeited telemetry。

fallback credit 的 telemetry 也很细：

- mint：`tengu_fallback_credit_minted`，记录 request/model/fallback target/token length/token usage/cache/service tier/speed/query chain。
- echo/redeem：`tengu_rotunda_pennant_credit_echoed`、`tengu_fallback_credit_outcome`，记录 `mint_request_id / mint_model / request_id / client_request_id / model / query_source` 和 usage/cost 相关字段。
- strip/retry：`tengu_rotunda_pennant_strip`，返回 `retry:fallback-credit-strip / retry:server-fallback-strip / retry:fallback-credit-unattributed / retry:fallback-credit-header-strip` 等内部标记。

这是一条“拒绝后重试/计费归属/协议兼容”的控制面。它和反蒸馏相关的交点是：当 primary model 因 `reasoning_extraction / frontier_llm` 等 category refusal 时，fallback lane 可以保留 category、撤回 partial，并决定是否换 fallback model；但服务端如何 mint token、如何判定 category、如何校验 token，仍不在本地 bundle 内。

直接证据：

- beta 常量：`package/preprocessed/cli.extracted.bundle.pretty.js:122663-122664`
- fallback request build 与 token extract：`package/preprocessed/cli.extracted.bundle.pretty.js:553314-553341`
- token 错误分类：`package/preprocessed/cli.extracted.bundle.pretty.js:282309-282357`
- fallback credit mint/outcome/echo/strip：`package/preprocessed/cli.extracted.bundle.pretty.js:714480-715232`
- streaming request body token：`package/preprocessed/cli.extracted.bundle.pretty.js:715204-715220`

### Server fallback 与 transcript retraction

server-side fallback 不是简单“换个模型再问一次”。本地 loop 会处理已经流出的 partial message、tool_use、tool_result 与 UI/transcript 一致性：

- server fallback block 会被 materialize 成 `fallback` content block，或转成 `server_fallback` internal event。
- 如果 fallback 目标模型不在 org `availableModels` allowlist，客户端会 decline 这次 swap，丢弃 fallback 响应，tombstone 相关消息，并生成 refusal/error。
- mid-stream fallback 会 tombstone 已流出的 refused leg；如果有 retained text，会用 `refusal_continuation` 开始拼接，并在新的 assistant message 上写 `supersedesUuids`。
- `model_refusal_fallback` system event 带 `retractedMessageUuids` 与 `refusedUserMessageUuid`，SDK schema 对外映射成 `retracted_message_uuids / refused_user_message_uuid`。
- assistant message schema 也有 `supersedes`，用于告诉消费者“这个 assistant message 替代之前已经送达的 wire uuids”。
- remote/SDK consumer 还会验证 retraction signal 来源，只接受 worker source；source missing/mismatch 会打 `tengu_refusal_retraction_unauthenticated_signal`。

这些字段的价值在于恢复“canonical transcript”：拒绝前已经送达的 partial、工具调用和 tool_result 不能继续被后续模型当成正常历史。对重写来说，这比 UI 横幅更重要。

直接证据：

- server fallback allowlist decline/tombstone：`package/preprocessed/cli.extracted.bundle.pretty.js:558740-558850`
- client retry system event 与 supersedes：`package/preprocessed/cli.extracted.bundle.pretty.js:559001-559248`
- SDK/transcript schema：`package/preprocessed/cli.extracted.bundle.pretty.js:837913-838523`
- retraction consumer 与 unauthenticated signal：`package/preprocessed/cli.extracted.bundle.pretty.js:765506-765615`

直接证据：

- refusal category set：`package/preprocessed/cli.extracted.bundle.pretty.js:561146-561161`
- refusal message：`package/preprocessed/cli.extracted.bundle.pretty.js:282960-283016`
- fallback 参数与 system event：`package/preprocessed/cli.extracted.bundle.pretty.js:553314-553318`、`553496-553515`
- request body fallback 与 400 strip/retry：`package/preprocessed/cli.extracted.bundle.pretty.js:714674-714770`、`714843-714904`
- SDK schema：`package/preprocessed/cli.extracted.bundle.pretty.js:838441-838493`
- [06-stream-processing-and-remote-transport.md](./06-stream-processing-and-remote-transport.md)

## Model fallback、off-switch 与 consent fallback

除 refusal fallback 外，还有更普通但同样重要的模型切换面：

- `model_fallback` 的 trigger 包括 `model_not_found / permission_denied / overloaded / server_error / last_resort / model_blocked`。
- `model_blocked` 来自 per-model kill switch / model error override：如果有 fallback model，会抛出 `oB(..., "model_blocked")` 触发换模型；没有 fallback 时生成 rate_limit 形态的错误消息。
- 529、404 streaming endpoint、stream idle、stream connection before first event 等会触发 streaming retry 或 non-streaming fallback。
- `switchModelsOnFlag` 是 refusal fallback lane 的用户设置，默认 true；文案语义是 safety measures flag message 后自动切换到另一个模型继续聊天，关闭后 session 会暂停而不是自动切换。设置变更会打 `tengu_refusal_fallback_setting_changed`。
- Fable/extra usage consent gate 可能生成 `model_consent_fallback`，表示用户拒绝、dismiss，或 consent 后 entitlement 没有真正 provisioned，session 会切回 fallback/default model。

这类机制说明客户端有“模型可用性/授权/额度/灰度”的恢复路径。它们不是 prompt 隐写，也不能单独证明滥用处罚。

直接证据：

- `switchModelsOnFlag` schema/setting：`package/preprocessed/cli.extracted.bundle.pretty.js:66714-66720`、`280790-280934`、`585268-585276`
- per-model off-switch：`package/preprocessed/cli.extracted.bundle.pretty.js:714400-714415`
- API fallback triggers：`package/preprocessed/cli.extracted.bundle.pretty.js:724823-724940`
- streaming fallback/retry：`package/preprocessed/cli.extracted.bundle.pretty.js:716300-716660`
- `model_fallback / model_consent_fallback` schema：`package/preprocessed/cli.extracted.bundle.pretty.js:838506-838533`

## Cache diagnosis 与请求兼容 strip

还有一条不是反滥用、但和隐藏 beta/header 很像的诊断通道：

- `cache-diagnosis-2026-04-07` beta 会在 request body 里发送 `diagnostics: { previous_message_id: ... }`。
- server 如果拒绝该 beta，会 drop header latch 并以 `retry:cache-diagnosis-beta` 重试。
- cache diagnosis 的 invalid error 会被归类成 `previous_message_id_invalid`。

这说明 2.1.197 对 beta/header 协议不兼容有统一的 strip/retry 习惯。写重写逻辑时，不能把某个 beta 400 当成普通 fatal error；但也不能把 strip/retry 外推成安全策略。

直接证据：

- beta 常量：`package/preprocessed/cli.extracted.bundle.pretty.js:122653`
- diagnostics body 与 retry：`package/preprocessed/cli.extracted.bundle.pretty.js:714756-714775`、`715347-715350`
- invalid classification：`package/preprocessed/cli.extracted.bundle.pretty.js:282840-282850`

## 本地证据不能推出的服务端黑箱

当前文档可以写实这些客户端事实：

- 哪些字段会被本地生成、缓存、发送或记录。
- 哪些 policy 会在本地执行。
- 哪些 refusal/fallback 协议字段会被本地消费。
- 哪些 Gateway / bridge 防护在客户端或 bundled gateway server 中实现。
- 哪些 request metadata、trace id、event attribution 字段会被客户端生成或提升。

但不能把这些事实继续外推成：

- Anthropic 服务端一定用 `currentDate` 或 telemetry 做封号。
- `currentDate` marker 就是完整封控、反蒸馏或账号处罚机制。
- `2.1.91` 是该 marker 的已确认引入版本。
- `reasoning_extraction` 的具体判定规则已被恢复。
- Gateway/telemetry/request header 能单独识别所有蒸馏或滥用行为。
- policy limits、Trusted Device、Auto mode classifier 属于 prompt 隐写。
- `metadata.user_id`、`x-client-request-id`、`traceparent` 或 `_PROTO_*` 任一单项就是封禁依据。
- `fallback_credit_token`、`anthropic-usage-limit`、`anthropic-dispatch-id` 或 `x-anthropic-additional-protection` 任一单项就是封禁、蒸馏判定或处罚依据。
- `server-side-fallback` / `fallback-credit` 的服务端计费、token mint、token 校验与 category 判定规则已被恢复。
- changelog 中的性能百分比、服务端默认模型投放、组织运营侧可用性或灰度开关已被本地 2.1.197 bundle 证明。
- 本地 bundle 已覆盖所有服务端风控策略。

更稳的重写判断是：这些机制要分层实现，字段和事件名要保留兼容；服务端策略只能按黑箱接口处理，不要在客户端重写里伪造不存在的判定能力。
