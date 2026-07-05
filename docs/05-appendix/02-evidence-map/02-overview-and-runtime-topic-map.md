# 证据索引：overview 与 runtime 主题

## 主题到证据的对应关系

### [../../00-overview/00-index.md](../../00-overview/00-index.md)

- 主要证据：当前 `docs/` 目录结构、`Plan.md` 主题矩阵、evidence map 与各主题页的导航关系
- 可信度：高
- 用途：作为知识库总入口与阅读顺序索引，不承载 bundle 直接证据；新增运行时结论应进入对应主题页

### [../../00-overview/01-scope-and-evidence.md](../../00-overview/01-scope-and-evidence.md)

- 主要证据：当前发行版入口证据、抽取 bundle 静态分析范围、运行时探测记录、各主题页的汇总结论
- 可信度：高
- 用途：确定当前 `2.1.197` wrapper + native binary 形态、抽取入口 bundle、哪些结论已经足以指导重写，以及哪些还只是待验证

### [../../00-overview/02-document-style-and-structure-conventions.md](../../00-overview/02-document-style-and-structure-conventions.md)

- 主要证据：知识库当前采用的父页/子页职责边界、拆分页维护规则、迁移与压缩约定
- 可信度：高
- 用途：校对目录结构是否仍与正文职责边界一致，避免把导航页重新写回直接证据页

### [../../01-runtime/01-product-cli-and-modes.md](../../01-runtime/01-product-cli-and-modes.md)

- 主要证据：`l3m()` 顶层入口分支、`l4m()` / `u4m()` 主入口与 Commander 命令树、`o3m()` daemon dispatcher、`Glc` background handlers、update/download/native failure 分支、`--from-pr`、`/review`、`/code-review ultra` / `ultrareview`、Advisor/Monitor 入口、低影响命令筛选、`--help` 输出与非交互判定分支
- 可信度：高
- 后续补证据重点：Chrome/native-host 的更细启动参数；daemon/fleet 的服务端或 feature flag 上游语义仍按黑箱处理

### [../../01-runtime/02-session-and-persistence.md](../../01-runtime/02-session-and-persistence.md)

- 主要证据：session 状态对象、`Zn() / t1() / Sj() / SF() / Zf() / Tk(...)` 路径计算、`Ycc` transcript writer、resume/read-session 解析路径
- 可信度：高
- 后续补证据重点：file-history backup 的更细时机、remote sync 下的持久化冲突边界

### [../../01-runtime/03-input-compilation.md](../../01-runtime/03-input-compilation.md)

- 主要证据：导航页；负责 `AU8 / ihz` 预处理、block/slash command 编译、`UserPromptSubmit` hook 三个拆分页入口
- 可信度：高
- 用途：把输入编译主题拆成稳定阅读顺序，不单独承载直接证据

### [../../01-runtime/03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md](../../01-runtime/03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md)

- 主要证据：`AU8 -> ihz` 调用链、输入归一化、pasted image 处理、remote-control slash gate、attachment loading gate、bash / local slash command / ordinary prompt 三条主分流
- 可信度：高
- 后续补证据重点：remote-control 之外是否还存在第二层命令过滤

### [../../01-runtime/03-input-compilation/02-block-family-bu4-and-slash-command-results.md](../../01-runtime/03-input-compilation/02-block-family-bu4-and-slash-command-results.md)

- 主要证据：输入 block family 边界、`BU4(...)` 普通输入消息构造、`processSlashCommand(...) / nx_(...) / r74(...)`、prompt slash command 编译结果
- 可信度：高
- 后续补证据重点：bundle 外/灰度 producer 是否会把 `document` 等更低频 block 直接喂进 `ihz(...)`

### [../../01-runtime/03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md](../../01-runtime/03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md)

- 主要证据：`UserPromptSubmit` hook 输入、执行位置、输出合并、截断、matcher、完成顺序、去重规则与 `AU8(...)` 阻断结果消费
- 可信度：高
- 后续补证据重点：多 hook 并发完成顺序在实际慢 hook 场景下的稳定性

### [../../01-runtime/04-agent-loop-and-compaction.md](../../01-runtime/04-agent-loop-and-compaction.md)

- 主要证据：导航页；负责主循环/compact 四个拆分页的入口、阅读顺序与专题边界，不单独承载直接证据
- 可信度：高
- 用途：把主循环状态、compact 管线、无工具分支、工具轮与终止返回组织到同一主题组

### [../../01-runtime/04-agent-loop-and-compaction/01-main-loop-state-caches-and-yield-surface.md](../../01-runtime/04-agent-loop-and-compaction/01-main-loop-state-caches-and-yield-surface.md)

- 主要证据：`zN(...) / oRf(...)` 最小状态机骨架、长生命周期状态 `m`、`transition` 活消费点、turn 内缓存、`stream_request_start / assistant / attachment / tombstone` 等 `yield` 面
- 可信度：高
- 后续补证据重点：少量低频 `yield` subtype 的 UI/sidecar 可见性边界

### [../../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md](../../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md)

- 主要证据：`rfa(...)` time-based microcompact、`deps.autocompact(...)`、`compactionResult -> FOo(...) / SHe(...)` 双 adapter 合同、`compact_boundary / preservedSegment / preservedMessages`、`compactTracking`
- 可信度：高
- 后续补证据重点：partial/full/session-memory compact 在更边缘配置下的差异

### [../../01-runtime/04-agent-loop-and-compaction/03-no-tool-branch-recovery-stop-and-reactive-compact.md](../../01-runtime/04-agent-loop-and-compaction/03-no-tool-branch-recovery-stop-and-reactive-compact.md)

- 主要证据：`Ae.length === 0` 的 turn 尾部分支、`J26 / DD4` reactive compact 未接线槽位、`FOo(...) / SHe(...)` compact 接口合同、`max_output_tokens` 还原、Stop/SubagentStop hook 子状态机
- 可信度：高
- 后续补证据重点：reactive compact 未接线结论在其他 build 里的稳定性

### [../../01-runtime/04-agent-loop-and-compaction/04-tool-round-next-turn-and-terminal-reasons.md](../../01-runtime/04-agent-loop-and-compaction/04-tool-round-next-turn-and-terminal-reasons.md)

- 主要证据：`KHe` streaming tool runner、`getCompletedResults() / getRemainingResults()`、`hook_stopped_continuation`、延后一轮消费的 tool summary、`aborted_streaming / aborted_tools / model_error` 等终止返回
- 可信度：高
- 后续补证据重点：少量错误码与终止原因在 remote/backend 变体下的映射差异

### [../../01-runtime/05-model-adapter-provider-and-auth.md](../../01-runtime/05-model-adapter-provider-and-auth.md)

- 主要证据：`HSt / m1o / Hpc / Kur` 包装层、provider factory、first-party/3P 分支、auth token / apiKey 来源优先级、`CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` 下的 `/v1/models?limit=1000` 枚举与 `gateway-models.json` 缓存、request telemetry 的 URL/model 字段、Claude Code Gateway 的 `upstreams[] / models[] / upstream_model` schema、模型名表与 `ANTHROPIC_BASE_URL` 标记表的分层边界
- 可信度：高
- 后续补证据重点：少量 header/cache 边缘行为；控制面接口总表已拆到 `01-runtime/10-control-plane-api-and-auxiliary-services.md`；`WebSearchTool` 细节已拆到 `01-runtime/07-web-search-tool.md`

### [../../01-runtime/06-stream-processing-and-remote-transport.md](../../01-runtime/06-stream-processing-and-remote-transport.md)

- 主要证据：导航页；负责 stream event/fallback、request telemetry/refusal、remote transport 三个拆分页入口
- 可信度：高
- 用途：把 streaming 与 remote transport 主题拆成稳定阅读顺序，不单独承载直接证据

### [../../01-runtime/06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md](../../01-runtime/06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md)

- 主要证据：`Hpc(...)` streaming 累积逻辑、SDK stream event 到内部事件映射、assistant block 增量产出、non-streaming fallback 触发条件与返回形态
- 可信度：高
- 后续补证据重点：低频 server-side result block 的 UI 状态映射差异

### [../../01-runtime/06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md](../../01-runtime/06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md)

- 主要证据：`NDl / KOo / UDl` 的 request/error/success telemetry、`F7t(...)` refusal telemetry、`Dlm(...)` previous request linkage、`Apc(...)` 生成的 `x-client-request-id / traceparent`、`queryTracking`、attempt、`firstAttemptRequestId`、`serverFallbackHop`、`anthropic-usage-limit / anthropic-dispatch-id` strip/retry、`$Dl / fRf / mRf` gateway classifier、`Hpc / Bua` model off-switch 与 per-model block、`stream_request_start` 来源、`CLAUDE_STREAM_IDLE_TIMEOUT_MS` 与 `CLAUDE_CODE_FORCE_SYNC_OUTPUT` 的输出/stream 状态边界
- 可信度：高
- 后续补证据重点：server-side refusal category 与本地 model block 的更多映射样本；服务端如何解释 request/header/trace 字段仍按黑箱处理

### [../../01-runtime/06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md](../../01-runtime/06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md)

- 主要证据：`sdk-url` / bridge / ingress transport、remote stream replay、remote auth/header 边界、本地 request build 最后边界、`Hpc(...)` 条件性 `wt(qe)` builder、`_pc(...)` 复用 builder 的 non-streaming fallback、`XLe()` request metadata、prompt payload / request options / API headers / telemetry metadata / transport metadata 分层矩阵
- 可信度：高
- 后续补证据重点：远端 ingress 的更细 header/sequence 行为；bundle 外 remote runtime 是否还追加服务端 context 仍按 opaque extension 处理

### [../../01-runtime/07-web-search-tool.md](../../01-runtime/07-web-search-tool.md)

- 主要证据：`WebSearchTool.call(...)`、`Kl_(...)`、`_l_(...)`、`Ow4/jw4/Hw4`、`HSt(...) / Hpc(...)`、`client.beta.messages.create(...)`、`sdk-tools.d.ts` 里的 `WebSearchInput/WebSearchOutput`
- 可信度：高
- 后续补证据重点：服务端真实搜索供应商、`web_search_20260209` / `UserLocation` 是否在其他路径已启用

### [../../01-runtime/08-web-fetch-tool.md](../../01-runtime/08-web-fetch-tool.md)

- 主要证据：`WebFetch` 本地工具实现、`Qo1 / CY4 / bY4 / IY4 / do1 / co1 / lo1 / UZ`、`sdk-tools.d.ts` 里的 `WebFetchInput/WebFetchOutput`、generic streaming 对 `web_fetch_tool_result` 的支持、bundle 内嵌 `WebFetchTool20260209` 文档
- 可信度：中高
- 后续补证据重点：`mb8` 预批准列表全集、二进制内容落地保存的 content-type 条件、是否存在尚未命中的 server-side `web_fetch` 运行时包装器

### [../../01-runtime/09-api-lifecycle-and-telemetry.md](../../01-runtime/09-api-lifecycle-and-telemetry.md)

- 主要证据：导航页；负责 telemetry 初始化、startup event、1P event logging 与流量开关四个拆分页入口
- 可信度：高
- 用途：把 API lifecycle / telemetry 主题拆成稳定阅读顺序，不单独承载直接证据

### [../../01-runtime/09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md](../../01-runtime/09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md)

- 主要证据：`Rg8 / H4A / qvz / initializeTelemetry`、`Tu / CF1 / EBq / tL8 / IF1 / yBq`、remote managed settings gating 与启动期 eager-init 分支
- 可信度：高
- 后续补证据重点：remote managed settings 上游返回体、超时策略在更多账号类型下的实际差异

### [../../01-runtime/09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md](../../01-runtime/09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md)

- 主要证据：`Fd8(...)` counter 注册点、`s4m(...)` / `tengu_startup_telemetry` 字段表、`gh_auth_status` 枚举、settings/env 摘要字段、证书 presence / `cert_store` 边界，以及 startup telemetry 和 `initializeTelemetry()` 的分工
- 可信度：高
- 后续补证据重点：低频 startup 字段在 remote / headless 入口下的覆盖差异

### [../../01-runtime/09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md](../../01-runtime/09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md)

- 主要证据：`Qg7()` / `gg7()`、`tengu_event_sampling_config`、`tengu_1p_event_batch_config`、`BYr` event logging endpoint、settings/env 对 telemetry runtime 与 payload 的改写矩阵
- 可信度：高
- 后续补证据重点：OTEL exporter 与 1P event logging 的剩余配置边角

### [../../01-runtime/09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md](../../01-runtime/09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md)

- 主要证据：`qHs / Ki / Khe / Kfn` 对 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`DISABLE_TELEMETRY`、`DO_NOT_TRACK` 的 gating 矩阵，`yXa()` 对 `CLAUDE_CODE_ENABLE_TELEMETRY` 的 3P OTEL exporter 开关，`NDl / KOo / UDl / F7t` request lifecycle 字段表，`zOo / gRf / Ehe / Tfn / Pd` 的 base URL 规范化与 hash，`Jc(...)` event logger、`wP(...)` 60KB 截断和 API body logging/redaction 边界
- 可信度：高
- 后续补证据重点：`DISABLE_TELEMETRY` / `DO_NOT_TRACK` 的服务端侧含义

### [../../01-runtime/10-control-plane-api-and-auxiliary-services.md](../../01-runtime/10-control-plane-api-and-auxiliary-services.md)

- 主要证据：导航页；`Ns()` host/OAuth 配置、`f8(...)` first-party provider client、`uH(...)` org-scoped remote header、`WBp() / N2a(...)` remote managed settings、`Hoe(...) / Smt(...) / _q(...)` remote environment/session 主链
- 可信度：高
- 用途：把 first-party 网络面与控制面主题拆成稳定阅读顺序，避免把总览页误当成接口细节主证据

### [../../01-runtime/10-control-plane-api-and-auxiliary-services/01-hosts-auth-and-data-plane.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/01-hosts-auth-and-data-plane.md)

- 主要证据：first-party 四层网络面划分、`Ns()` 中的 `api.anthropic.com / platform.claude.com / claude.ai / claude.com / mcp-proxy.anthropic.com`、`f8(...)` 数据面鉴权、`uH(...)` org-scoped remote header、`WBp() / N2a(...)` remote settings auth、`/v1/messages / /v1/models / /v1/files`
- 可信度：高
- 后续补证据重点：少量 supporting endpoints 的字段级 schema

### [../../01-runtime/10-control-plane-api-and-auxiliary-services/02-oauth-and-account-control-plane.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/02-oauth-and-account-control-plane.md)

- 主要证据：`Ns()` / `fAn(...)` authorize URL、`c2r(...)` token exchange、`Bte(...)` refresh、`wxe(...) / mAn(...)` profile、`u2r(...)` roles、`d2r(...)` create API key、`kj()` org UUID fallback、OAuth scope 族
- 可信度：高
- 后续补证据重点：role/profile 返回体的更多低频字段

### [../../01-runtime/10-control-plane-api-and-auxiliary-services/03-remote-environments-and-sessions.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/03-remote-environments-and-sessions.md)

- 主要证据：`Hoe(...)` environment 枚举、`Smt(...)` 默认 cloud environment 创建、`_q(...)` teleport/session create、`u0f(...)` bridge session create、`N3n(...)` event route、`kWo.connect()` 的 `/v1/code/sessions/{id}/events/stream` SSE 主链、managed-agents SDK session wrappers
- 可信度：高
- 后续补证据重点：environment/session 返回 schema 的边角字段、错误还原分支，以及旧 `sessions/ws/{id}/subscribe` 是否只属于旧版残留

### [../../01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md)

- 主要证据：`/v1/environments/bridge*`、`environment_secret -> work secret -> session_ingress_token`、`/v1/code/sessions/{id}/bridge` 响应字段、`worker_jwt` 刷新链、`worker_epoch`、`X-Trusted-Device-Token`、`/api/auth/trusted_devices` enrollment、`require_trusted_devices` policy gate
- 可信度：高
- 后续补证据重点：`environment_secret` 轮换边界与服务端正式 TTL 语义

### [../../01-runtime/10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md)

- 主要证据：`YVe(...)` GitHub App check、`SNp() / hac()` token-sync、`mac(...)` import-token、`/api/event_logging/batch`、`ClaudeCodeInternalEvent / GrowthbookExperimentEvent`、`claudeai-proxy` MCP transport
- 可信度：中高
- 后续补证据重点：GitHub beta 常量命名来源与少量外围控制面边界

### [../../01-runtime/11-non-llm-network-paths.md](../../01-runtime/11-non-llm-network-paths.md)

- 主要证据：`connectVoiceStream / ip8 / useVoice`、`EL6 / I3z / b3z`、`initializeGrowthBook / tl / D_8 / EventSource / TZ1 / ZZ1.sendBatchWithRetry`、transcript share endpoint、remote transcript persistence、`1p_failed_events.*.json` 本地补偿缓存、`statsStore / getFpsMetrics`
- 可信度：高
- 后续补证据重点：voice 服务端 STT provider 细节、plugin install counts 上游 stats 数据生成链、`statsStore` 与更大 perf/telemetry exporter 的完整对应关系

### [../../01-runtime/12-settings-and-configuration-system.md](../../01-runtime/12-settings-and-configuration-system.md)

- 主要证据：导航页；负责 source/path、加载与 merge、缓存与写回、CLI 注入与 schema、消费索引与重写边界五个拆分页的入口与边界
- 可信度：高
- 用途：把配置系统从单页大纲改造成按职责拆分的稳定入口，避免把总览页误写成实现细节全集

### [../../01-runtime/12-settings-and-configuration-system/01-source-model-and-paths.md](../../01-runtime/12-settings-and-configuration-system/01-source-model-and-paths.md)

- 主要证据：5 个正式 settings source、`--setting-sources` 真实边界、用户根目录与 enterprise policy 根目录、`userSettings / projectSettings / localSettings / flagSettings / policySettings` 路径族
- 可信度：高
- 后续补证据重点：不同平台上的目录 fallback 差异

### [../../01-runtime/12-settings-and-configuration-system/02-loading-policy-and-merge.md](../../01-runtime/12-settings-and-configuration-system/02-loading-policy-and-merge.md)

- 主要证据：`ye(path)` 与 `_D()` 校验链、`C28(...)` permission rules 预清洗、`PZ3()` effective settings 装配、`policySettings` fallback、remote managed settings 与 policy limits merge 规则、policy limits 的 `restrictions / compliance_taints / monitoring_notice / defaults` schema 与 taint-to-capability deny 映射
- 可信度：高
- 后续补证据重点：服务端二次裁剪与 org policy 变体边界

### [../../01-runtime/12-settings-and-configuration-system/03-cache-refresh-and-writeback.md](../../01-runtime/12-settings-and-configuration-system/03-cache-refresh-and-writeback.md)

- 主要证据：两级 cache 与 plugin overlay cache、统一 cache invalidate、settings 文件 watcher、`ConfigChange` hook、MDM/registry poll、`no(source, patch)` 写回语义、config 面板与 app state 分流
- 可信度：高
- 后续补证据重点：更多热刷新源在非桌面环境中的触发差异

### [../../01-runtime/12-settings-and-configuration-system/04-cli-injection-schema-and-migration.md](../../01-runtime/12-settings-and-configuration-system/04-cli-injection-schema-and-migration.md)

- 主要证据：`--settings / --setting-sources`、`apply_flag_settings`、settings change 的运行时反应链、schema 主分组、启动迁移矩阵与兼容入口
- 可信度：高
- 后续补证据重点：少量历史迁移项的输入兼容边界

### [../../01-runtime/12-settings-and-configuration-system/05-key-consumers-and-rewrite-boundaries.md](../../01-runtime/12-settings-and-configuration-system/05-key-consumers-and-rewrite-boundaries.md)

- 主要证据：高价值键族到消费点索引、`parentSettingsBehavior`、`footerLinksRegexes`、`autoScrollEnabled`、`wheelScrollAccelerationEnabled`、`attribution.sessionUrl`、`CLAUDE_CLIENT_PRESENCE_FILE`、`ANTHROPIC_WORKSPACE_ID`、`CLAUDE_PROJECT_DIR`、`CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN`、`CLAUDE_CODE_FORCE_SYNC_OUTPUT`、`strictPluginOnlyCustomization` surface 边界、`sshConfigs` 负证与剩余不确定性、可直接继承的重写骨架
- 可信度：高
- 后续补证据重点：`sshConfigs` 是否仍有 bundle 外或灰度入口

### [../../01-runtime/13-abuse-risk-and-refusal-controls.md](../../01-runtime/13-abuse-risk-and-refusal-controls.md)

- 主要证据：`Wla / rdp / odp / qla` 的 `currentDate` 标记链、API client request headers、`XLe()` request metadata、`x-client-request-id / traceparent` correlation、`NDl / KOo / F7t / UDl` request/refusal telemetry、`anthropic-usage-limit / anthropic-dispatch-id / x-anthropic-additional-protection` 隐式 header、`quota_check` probe 与 unified rate-limit/overage headers、OTEL/API body logging redaction、1P event logging 的 `_PROTO_*` attribution、`policy_limits` schema/fetch/cache、remote managed settings 的 `availableModels / enforceAvailableModels / forceLoginOrgUUID / forceRemoteSettingsRefresh / otelHeadersHelper / allowManagedHooksOnly / allowedHttpHookUrls / httpHookAllowedEnvVars / allowManagedPermissionRulesOnly / allowManagedMcpServersOnly / strictPluginOnlyCustomization / strictKnownMarketplaces / blockedMarketplaces / disableSideloadFlags / pluginSuggestionMarketplaces` 等 policy 键、Auto mode classifier prompt 与 XML 两阶段 fail-closed、project/local settings 不能授予默认 `auto` mode、denial limit/headless abort、sandbox/managed settings schema、workspace trust 对 `apiKeyHelper / awsAuthRefresh / awsCredentialExport / gcpAuthRefresh / MCP headersHelper / proxyAuthHelper / otelHeadersHelper` 的执行 gate、project/local `permissions.additionalDirectories` trust 裁剪、external channel/plugin untrusted wrapper、`CLAUDE_CODE_SCRIPT_CAPS`、subprocess env scrub、deep-link argument injection guard、UNC/WebDAV/git-dir 风险 gate、Trusted Device enrollment、bridge event `device_attestation_status` filter、Gateway private-network/TLS pin/SSRF/rate-limit/CSRF/CIDR/spend-limit 防护、`switchModelsOnFlag` refusal fallback 开关、`server-side-fallback-2026-06-01 / fallback-credit-2026-06-01`、`fallback_credit_token` mint/echo/outcome/strip、`model_refusal_fallback / model_refusal_no_fallback / model_fallback / model_consent_fallback` schema、`supersedes / retracted_message_uuids / refused_user_message_uuid` transcript retraction、cache diagnosis beta 与 400 strip/retry，以及“本地生成 / 本地发送 / 本地消费 / 本地执行 / 本地记录”动作类型表
- 可信度：高
- 后续补证据重点：服务端如何使用 `currentDate`、telemetry、`frontier_llm / reasoning_extraction` refusal category、`fallback_credit_token`、policy limits、hidden headers 与 account 风控仍是 bundle 外黑箱；当前文档只能证明客户端字段生成、发送、缓存、执行、strip/retry、tombstone/retraction 和消费边界

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
