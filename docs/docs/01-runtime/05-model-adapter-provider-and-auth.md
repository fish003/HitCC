# 模型适配层、Provider 选择与鉴权

## 本页用途

- 用来说明 `HSt / m1o / Hpc / Kur` 这一层如何向主循环暴露统一的模型调用门面。
- 用来集中梳理 provider 工厂、first-party / 3P 分支以及 auth token / apiKey 的来源优先级。

## 相关文件

- [04-agent-loop-and-compaction.md](./04-agent-loop-and-compaction.md)
- [07-web-search-tool.md](./07-web-search-tool.md)
- [06-stream-processing-and-remote-transport.md](./06-stream-processing-and-remote-transport.md)
- [10-control-plane-api-and-auxiliary-services.md](./10-control-plane-api-and-auxiliary-services.md)
- [../04-rewrite/02-open-questions-and-judgment.md](../04-rewrite/02-open-questions-and-judgment.md)
- [../05-appendix/01-glossary.md](../05-appendix/01-glossary.md)

## 模型调用适配层：HSt / H.callModel / Hpc / Kur

### 依赖注入点

当前主循环依赖对象提供：

- `callModel: HSt`
- `microcompact`
- `autocompact: OOo`
- `uuid`

说明 `zN(...) / oRf(...)` 本身不关心底层 API，只关心 `callModel` 这个高层流接口。

### `HSt(...)` 与 `Hpc(...)` 的职责

高可信职责：

- 将 `messages + systemPrompt + tools + options` 组装成一次高层模型请求
- 统一暴露 `stream_event / assistant / system(api_retry) / attachment` 事件流
- 负责把底层 SDK streaming 归一成主循环所需要的事件模型

从可读 bundle 看，`HSt(...)` 本身很薄：

- `HSt(...)` 只是通过 `m1o(...)` 包一层 fixture/VCR 逻辑，再 `yield* Hpc(...)`
- `Xo(...)` 则是“跑完整条流，只取最后一个 assistant”的便捷包装

因此真正的 provider 调用、streaming 状态累积、fallback 与错误收敛，核心都在 `Hpc(...)`，不是 `HSt(...)` 这个入口名本身。

### `m1o(...)`：测试夹具/VCR 包装层，不是运行时 replay

`m1o(messages, run)` 的行为现在也已可见：

- 若 `vtr()` 未开启：直接 `yield* run()`
- 若开启：通过 `wtr(...)` 尝试 fixture replay
- 有 replay 则直接 `yield *replay`
- 否则收集真实结果数组，结束后再原样 `yield *K`

继续往下追后，这条链已经可以从“像 replay”改写成更精确的结论：

- fixture 开关在普通发行运行路径下不会打开
- 这不是 session/transcript replay，而是测试期的 API fixture 读写器
- fixture 根目录为 `process.env.CLAUDE_CODE_TEST_FIXTURES_ROOT ?? cwd`
- 文件名来自“过滤掉 meta user message 后的输入消息内容”哈希：
  - 先做 message content 归一化
  - 再对每段内容取 sha1 前 6 位
  - 最终落成 `fixtures/<hash>-<hash>-....json`
- 若 fixture 存在：读取 `output`，做占位符反向展开后直接返回
- 若 fixture 不存在：
  - CI 且未开启 `VCR_RECORD` -> 直接报错，要求录制 fixture
  - 否则执行真实请求，并把 `input/output` 写回 fixture

因此 `m1o / wtr / vtr` 并不是生产运行时的 replay/还原机制，而是测试夹具层；对真实 provider/gateway/transport 主路径可以视为硬旁路。

### `Kur`：重试/降级包装层

已确认处理：

- 401/token 失效 -> 刷新/重建 client
- 429/overloaded -> retry-after / 退避 / 关闭 fastMode 再试
- 连续 529 + fallbackModel -> 抛 `Lw6(original, fallback)`
- context limit overflow -> 下调 `maxTokensOverride` 后重试

现在还能更精确地描述：

- `Kur(A, q, K)` 是 async generator
- `A` 通常就是 lazy client factory：`() => f8(...)`
- `q` 是“拿到 client 后真正发请求”的回调
- `K` 是 retry context，至少含 `model / thinkingConfig / signal / fallbackModel`

其内部逻辑已可归纳为：

- 仅在以下情况重建 client：
  - 首次请求
  - 401
  - OAuth token revoked
  - Bedrock auth 错误
  - Vertex credential refresh 错误
- fastMode 下遇到 429/529，不会立刻失败，而是优先关闭 fastMode 再试
- 命中 `input length and max_tokens exceed context limit` 时，会解析报错字符串并写回 `z.maxTokensOverride`
- 普通 retry 前会 `yield rx1(...)`

还能再收紧成一张更接近实现的矩阵：

- 最大重试次数来源：
  - `K.maxRetries`
  - 否则 `CLAUDE_CODE_MAX_RETRIES`
  - 再否则默认 `10`
- fast mode 特殊分支：
  - 只有 `u4()` 路径才会真的读写 `z.fastMode`
  - 若 `429/529` 且带 `anthropic-ratelimit-unified-overage-disabled-reason`
    - 直接记录原因
    - 关闭 `fastMode`
    - 继续下一次 attempt
  - 若 `retry-after` 很短
    - 先按 header 等待
    - 不一定立刻关 `fastMode`
  - 若等待窗口较长
    - 记录 unified reset / overage telemetry
    - 关闭 `fastMode`
    - 再继续尝试
  - 若命中 `"Fast mode is not enabled"`
    - 也会直接关 `fastMode`
- `529 / overloaded` 不是无限重试：
  - 会累计连续 `529`
  - `initialConsecutive529Errors` 还能从 streaming fallback 继承
  - 达到阈值 `3` 后：
    - 有 `fallbackModel` -> 抛 `Lw6(original, fallback)`
    - 无 fallback -> 进入统一 overloaded 失败路径
- retryable 判定当前至少包括：
  - 网络层 `m0`
  - `408`
  - `409`
  - `429`
  - `401`
  - `5xx`
  - `x-should-retry: true`
  - remote 模式下的 `401/403`
  - overloaded / context overflow 解析成功
- context overflow 处理不是简单减一点：
  - 从错误字符串解析 `inputTokens / maxTokens / contextLimit`
  - 目标值近似 `contextLimit - inputTokens - 1000`
  - 再与 `thinking budget + 1`、`FLOOR_OUTPUT_TOKENS` 取上界
  - 回写为 `z.maxTokensOverride`
- 持续性重试与普通重试不是同一条时钟：
  - 普通重试按 attempt 编号指数退避
  - `NN8() && MNq(error)` 命中的持久性重试单独累计次数
  - 等待期间会反复 `yield rx1(...)`

`rx1(...)` 现在也已经能确认形状：

```ts
{
  type: "system",
  subtype: "api_error",
  error,
  retryInMs,
  retryAttempt,
  maxRetries,
}
```

因此 `Kur` 不只是“内部重试器”，也承担了向上游报告 API retry 的职责。

### 底层来源

这层现在已经可以从“Anthropic 风格封装”收敛到更具体的判断：

- 发行版内嵌的是改名后的 `Anthropic` SDK，而不是直接依赖外部 Anthropic SDK。
- first-party 默认 `baseURL` 是 `https://api.anthropic.com`。
- 仍沿用 Anthropic 风格环境变量：
  - `ANTHROPIC_BASE_URL`
  - `ANTHROPIC_API_KEY`
  - `ANTHROPIC_AUTH_TOKEN`
- SDK 认证头支持两路：
  - `X-Api-Key`
  - `Authorization: Bearer ...`
- 通用版本头为 `anthropic-version: 2023-06-01`。
- beta 特性通过 `anthropic-beta` 叠加，而不是 Anthropic 原始的 beta 头名字。

### provider 工厂：`f8(...)`

bundle 中已经能确认存在统一的 client factory：

```ts
async function f8({
  apiKey,
  maxRetries,
  model,
  fetchOverride,
  source,
  agentContext,
})
```

其职责是：

- 组装 first-party / 3P provider 通用 `defaultHeaders`
- 注入 request wrapper、timeout、retry、fetchOptions
- 按环境变量选择最终 provider client

通用 headers 至少包括：

- `x-app: cli`
- `User-Agent`
- `X-Claude-Code-Session-Id`
- `x-claude-remote-container-id`
- `x-claude-remote-session-id`
- `x-client-app`
- `x-claude-code-agent-id`
- `x-claude-code-parent-agent-id`
- `x-anthropic-additional-protection`（条件开启）

### provider 选择分支

已确认至少分为 5 条明确分支：

- `gateway` -> 带 cloud gateway JWT 与 gateway `baseURL` 的 first-party SDK client
- `firstParty` -> first-party Anthropic SDK client
- `bedrock / mantle` -> `AnthropicBedrock`
- `foundry` -> `AnthropicFoundry`
- `vertex` -> `AnthropicVertex`

切换位不是运行时自动探测，而是环境变量控制：

- `CLAUDE_CODE_USE_BEDROCK`
- `CLAUDE_CODE_USE_VERTEX`
- `CLAUDE_CODE_USE_FOUNDRY`

因此重写时应把“provider 选择”做成显式策略层，而不是把 first-party 与 3P provider 混在一个 client 内部用 if/else 打补丁。

### first-party / subscription / token 语义

从 auth 判定逻辑可确认：

- first-party 才会真正涉及 Claude account subscription / OAuth token 语义。
- 如果启用了 Bedrock / Vertex / Foundry，则 subscription token 逻辑基本被短路。
- first-party 模式下同时存在多种 auth 来源：
  - `CLAUDE_CODE_OAUTH_TOKEN`
  - `ANTHROPIC_AUTH_TOKEN`
  - `ANTHROPIC_API_KEY`
- `apiKeyHelper`
- bundle 中还专门有 auth conflict 警告，说明原版明确区分“Claude 账号订阅 token”与“外部 API key / auth token”。

现在还能进一步把判定链写清：subscription / OAuth 模式并不是“只要有 OAuth token 就算”，而是一个更严格的“没有落到外部 key/token 模式、也没有切到 3P provider”才成立的分支。

### first-party 身份态

因此 first-party 至少有两种非 3P 身份：

- subscriber：走 OAuth account / subscription 语义
- API customer：走 first-party API key / auth token 语义

### `f8(...)` 最终如何选 `apiKey` 与 `authToken`

在 first-party 分支里，`f8(...)` 最终传给 SDK client 的关键关系可以写成：

```ts
{
  apiKey: subscriberMode ? null : resolvedApiKey,
  authToken: subscriberMode ? oauthAccessToken : undefined,
}
```

这说明：

- subscriber 模式不会再给 SDK 传 `apiKey`
- subscriber 模式直接走 `authToken = OAuth access token`
- API customer 模式则走 `apiKey`

另外，`f8(...)` 在构造 client 前还有一条额外路径：

```ts
if (!subscriberMode) await maybeAttachBearerAuth(defaultHeaders, nonInteractive)
```

这里会把：

- `ANTHROPIC_AUTH_TOKEN`
- 或 `apiKeyHelper` 返回值

写入 `Authorization: Bearer ...`。

也就是说，API customer 模式下可能同时存在：

- SDK 级 `apiKey`
- header 级 `Authorization`

这也是为什么原版会专门区分“subscription token”和“外部 auth token / API key”。

### auth token 来源优先级

当前可见逻辑可以还原出一条比较明确的优先级链：

1. `--bare / CLAUDE_CODE_SIMPLE` 下，只承认 `apiKeyHelper`
2. `ANTHROPIC_AUTH_TOKEN`（仅非 remote/desktop）
3. `CLAUDE_CODE_OAUTH_TOKEN`
4. `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` / CCR OAuth token file
5. `apiKeyHelper`（仅非 remote/desktop）
6. 本地缓存的 claude.ai OAuth token，且 scope 含 `user:inference`
7. 否则 `none`

### API key 来源优先级

API key 侧也已经能还原出主优先级：

1. `ANTHROPIC_API_KEY`
2. API key file descriptor
3. `apiKeyHelper`
4. `/login managed key`
  - macOS keychain 或本地 config 中保存的 primaryApiKey
5. 否则 `none`

一个容易误读的点是：传给 `apiKeyHelper` 相关路径的 non-interactive 判断不是 token getter。它只影响 trust / 执行策略，不能把它理解成 auth source。

### `tp / UP8 / HZ1 / MZ1 / jD`：account/profile 元数据链已经闭环

这一组函数现在也不再需要保留为未知点，但它们属于 first-party auth / control-plane 元数据链，不属于模型 inference data-plane 本身：

- `tp(accessToken)`
  - 原始接口：`GET ${BASE_API_URL}/api/oauth/profile`
  - 认证：`Authorization: Bearer <oauth access token>`
  - 返回 account / organization 原始 profile
- `UP8(accessToken)`
  - 是 `tp(...)` 的语义包装
  - 把 `organization_type` 映射成 `subscriptionType`
    - `claude_max -> max`
    - `claude_pro -> pro`
    - `claude_enterprise -> enterprise`
    - `claude_team -> team`
  - 额外抽取：
    - `rateLimitTier`
    - `hasExtraUsageEnabled`
    - `billingType`
    - `displayName`
    - `accountCreatedAt`
    - `subscriptionCreatedAt`
  - 并保留 `rawProfile`
- `HZ1(accessToken)`
  - 调 `GET ${ROLES_URL}`
  - 把结果写入本地 `oauthAccount`
    - `organizationRole`
    - `workspaceRole`
    - `organizationName`
- `MZ1()`
  - 启动期补资料入口
  - 先读取环境变量 `CLAUDE_CODE_ACCOUNT_UUID / CLAUDE_CODE_USER_EMAIL / CLAUDE_CODE_ORGANIZATION_UUID` 作为最小 account seed
  - 然后 `await rz()` 刷新 token（若需要）
  - 仅当满足以下条件才继续拉 profile：
    - 当前身份是 subscriber 路径
    - token 具有 `user:profile` scope
    - 本地 `oauthAccount` 尚未具备 `billingType / accountCreatedAt / subscriptionCreatedAt`
  - 满足时才调用 `tp(accessToken)` 并覆盖/补齐本地 `oauthAccount`
- `jD()`
  - 统一“拿 organization UUID”的 helper
  - 先读 `oauthAccount.organizationUuid`
  - 若没有且 token 具备 `user:profile`，则退回 `tp(accessToken)` 取 `organization.uuid`

### 精确时机：启动期 / 登录期 / 刷新期

这条元数据链的触发时机也已经明确：

- 启动期
  - 初始化主流程里直接调用 `MZ1()`
  - 调用点位于全局 init 早期，紧跟 1P event logging 初始化之后
  - 这里是“异步触发但不阻塞后续 init”的 best-effort profile backfill
- 登录期
  - `fO6(...)` 是安装 OAuth token 的主入口
  - 它会先写 `oauthAccount` 的基础资料，然后 `y06(...)` 持久化 token
  - 随后总是尝试 `HZ1(accessToken)` 拉取 role/name
  - 若是 subscriber（`xR(scopes)`），再调用 `n14()` 拉 `first_token_date`
  - 若不是 subscriber，则改走 `JZ1(accessToken)` 为 first-party API customer 创建 API key
- 刷新期
  - `WU6(refreshToken)` 在 refresh 成功后，只在“本地还缺 profile 衍生字段”时才补调用 `UP8(newAccessToken)`
  - 它不会重新调用 `HZ1(...)`
  - 因此 role/name 主要在登录安装 token 时更新，而不是每次 refresh 时刷新

### 对 gateway / transport 的实际意义

这条链对主模块的意义现在也已明确：

- inference data-plane
  - `f8(...)` / `Kur(...)` / `Hpc(...)` 主要只依赖已选定 provider 与凭据解析结果
- control-plane / web sessions / remote
  - `UM()` 要求必须有 claude.ai OAuth access token，而不是 API key
  - `UM()` 还会调用 `jD()` 取 organization UUID
  - `fetchSession / list sessions / environment providers / remote create` 都走这条 `accessToken + orgUUID` 路径

所以 `tp / UP8 / HZ1 / MZ1 / jD` 的职责不是“决定如何发模型请求”，而是“补齐 first-party account metadata，并支撑需要 org UUID 的 control-plane API”。

### first-party API 面

当前可确认的 first-party 数据面接口至少有：

- `/v1/messages?beta=true`
- `/v1/messages/count_tokens?beta=true`
- `/v1/models?beta=true`
- `/v1/files`

另外还有一条容易和普通 `/v1/models` 混淆的 gateway discovery 路径。它不是默认开启，触发条件同时要求：

- `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY` 为真
- 当前 provider 仍判定为 `firstParty`
- 当前 host 不是官方 first-party host
- 存在 `ANTHROPIC_BASE_URL`
- 存在 `ANTHROPIC_AUTH_TOKEN` 或 API key
- 没有进入 essential-traffic 模式

命中后，客户端会请求：

```text
${ANTHROPIC_BASE_URL.rstrip("/")}/v1/models?limit=1000
```

请求会带：

- `Authorization: Bearer <ANTHROPIC_AUTH_TOKEN>` 或 `x-api-key`
- `anthropic-version: 2023-06-01`
- `User-Agent`
- `ANTHROPIC_CUSTOM_HEADERS` 解析出的自定义 header

响应只接受 Anthropic models-list 形状：

```json
{"data":[{"id":"...","display_name":"..."}]}
```

随后本地会过滤出 `id` 以 `claude` 或 `anthropic` 开头的模型，并写入：

```text
<appRoot>/cache/gateway-models.json
```

缓存内容包含原始 `baseUrl`、`fetchedAt` 与模型列表。之后模型选择项会用这些缓存项生成 `description: "From gateway"` 的选项。

所以这里确实存在一段“枚举自定义 gateway / 第三方兼容端点模型列表”的逻辑。它的实际边界是：

- 目标是用户配置的 `ANTHROPIC_BASE_URL`，不是默认的 Anthropic event logging endpoint
- 请求会把 `ANTHROPIC_CUSTOM_HEADERS` 原样并入 discovery 请求头
- 返回的第三方模型名只在本地筛选、缓存，并进入模型选择项
- 本地缓存会保留原始 `ANTHROPIC_BASE_URL`，不同于 request telemetry 里的 URL 处理

这段更像显式 opt-in 的兼容能力：打开开关后，客户端会向自定义 gateway 探测它广告出来的 Anthropic-compatible 模型。当前代码证据不能推出“默认把第三方完整 API URL 和完整模型列表上报给 Anthropic”。能确认的是：request telemetry 另有 `baseUrl / envModel / envSmallFastModel` 字段，其中非官方 `baseUrl` 会规范化后 hash，`envModel / envSmallFastModel` 来自环境变量模型名，按明文字段进入事件 payload。

同时还存在控制面/外围接口：

- OAuth token / roles / create_api_key
- `remote-control` environment 注册
- telemetry event logging
- MCP proxy

因此“gateway”不应理解成只有一个推理 endpoint，而是：

- inference/data plane：`api.anthropic.com` 上的 `/v1/...`
- control plane：OAuth / remote / file / telemetry / MCP proxy 等外围服务

控制面接口的完整归类、host 分层与远程环境/session 关系，已经单独拆到 [10-control-plane-api-and-auxiliary-services.md](./10-control-plane-api-and-auxiliary-services.md)，本页不再继续堆叠 endpoint 总表。

### beta/header 族

目前已补出的 beta/header 证据包括：

- `structured-outputs-2025-12-15`
- `token-counting-2024-11-01`
- `message-batches-2024-09-24`
- `files-api-2025-04-14`
- `skills-2025-10-02`

这说明：

- 模型调用门面不只是“发聊天消息”
- 还会驱动 structured output、token counting、files、skills 等能力面
- `sdkBetas` 是 session/runtime 级别状态，而不是单次请求局部常量

### 3P provider 细节

已补出的 3P provider 行为：

- Bedrock
  - 使用 `AnthropicBedrock`
  - 读取 AWS region / AWS creds
  - 支持 `AWS_BEARER_TOKEN_BEDROCK`
  - 可通过 `CLAUDE_CODE_SKIP_BEDROCK_AUTH` 跳过鉴权
- Foundry
  - 使用 `AnthropicFoundry`
  - 读取 `ANTHROPIC_FOUNDRY_BASE_URL` 或 `ANTHROPIC_FOUNDRY_RESOURCE`
  - `resource` 会展开成 `https://<resource>.services.ai.azure.com/anthropic/`
  - 认证是 API key 与 Azure AD token provider 二选一
- Vertex
  - 使用 `AnthropicVertex`
  - 借助 `GoogleAuth`
  - `/v1/messages` 会被改写到 `projects/.../publishers/anthropic/models/...:rawPredict`

### 模型名映射层

provider 之间并不共享完全相同的模型 ID。

当前 2.1.197 的模型注册表已经显式包含 Sonnet 5、Fable 5、Opus 4.7 与 Opus 4.8。可见事实如下，证据来自 `package/preprocessed/cli.extracted.bundle.pretty.js:114189-114425` 与 `package/sdk-tools.d.ts:429`：

| family / alias | 当前默认或最新值 | provider 映射边界 | context / capability 边界 |
| --- | --- | --- | --- |
| `sonnet` | `claude-sonnet-5` | first-party / Bedrock / Vertex / Foundry / Anthropic AWS / Mantle / Gateway 都有独立 provider id；`sonnet` alias 在部分 3P provider 上仍可落到 Sonnet 4.x | `claude-sonnet-5` 标记 `window: 1000000`、`native_1m`、`supports_1m_beta`、`effort / max_effort / xhigh_effort / mid_conv_system / context_management` |
| `opus` | 默认 `claude-opus-4-8`；`latest_per_family.opus = claude-opus-4-8` | `opus` alias 的 per-provider default 不完全一致：Bedrock / Vertex / Foundry 可落到 `claude-opus-4-6`，Mantle / Anthropic AWS / Gateway 可落到 `claude-opus-4-7` | Opus 4.7 和 4.8 都标记 1M context；4.8 额外有 `mid_conv_system`、`fast_mode`、`lean_prompt` |
| `fable` | `claude-fable-5` | `fable` alias 默认指向 `claude-fable-5`，provider id 覆盖 first-party / 3P / gateway | 标记 1M context、`xhigh_effort`、`rejects_disabled_thinking`、`mid_conv_system`、`lean_prompt` 与 `fable_5_mitigations` |

因此文档里可以把这些模型族作为当前本地注册表事实写入；不能从本地 bundle 推出外部价格、配额、账户默认投放或服务端灰度状态。

bundle 中还存在 first-party / bedrock / vertex / foundry 四路模型名映射表，例如：

- `claude-sonnet-4-6`
- `us.anthropic.claude-sonnet-4-6`
- `claude-sonnet-4-6`
- `claude-sonnet-4-6`

这会直接影响：

- `--model`
- `--fallback-model`
- `small-fast-model`
- capability 检查

因此重写时需要单独保留 model-name normalization / provider remap 层。

这里的模型名表与 `instruction-discovery` 里的 `ANTHROPIC_BASE_URL` domain/keyword 标记表不是同一机制。模型名表服务于 provider selection、allowlist、fallback 与 gateway 转发；base URL 标记表只在本地生成 `userContext.currentDate` 时影响 apostrophe / 日期格式，当前没有证据显示它参与模型选择或普通 URL 阻断。

### 内嵌 Claude Code Gateway 的上游 URL 与模型映射

当前 bundle 还内嵌了一套 `Claude Code Gateway` server / proxy 模块。它的配置 schema 会显式描述第三方或云厂商上游：

- `upstreams[]`
  - `provider: anthropic / bedrock / vertex / foundry`
  - `name`
  - `base_url`：`anthropic` 必填或默认 `https://api.anthropic.com`；`bedrock / vertex / foundry` 可选
  - 各 provider 自己的 auth 字段
- `models[]`
  - `id`
  - `label`
  - `description`
  - `upstream_model`：按 upstream name 映射到实际上游模型 ID
- `auto_include_builtin_models`

这个模块的 `/v1/models` 会返回 gateway 对客户端广告的模型列表。若开启 `auto_include_builtin_models`，它会根据已配置 upstream provider 自动补一批内置 Claude 风格模型；也可以通过 `models[]` 广告自定义模型。转发 `/v1/messages` 时，如果请求模型命中 `upstream_model` 映射，会把客户端传入的 Anthropic-style model ID 改写成对应上游模型 ID，并在响应 header 里补：

- `x-gateway-upstream`
- `x-gateway-model`
- `x-gateway-upstream-model`

这段确实会处理“第三方 API URL + 第三方模型名称/映射”。但它属于本地/企业自建 gateway 功能：配置里的上游 URL 和模型映射由 gateway 管理方提供，客户端通过 `/v1/models` 看到的是 gateway 广告出来的模型 ID，不等于普通 Claude Code 客户端默认把所有第三方 URL 和模型明文上报给 Anthropic。

底层流事件类型：

- `message_start`
- `message_delta`
- `message_stop`
- `content_block_start`
- `content_block_delta`
- `content_block_stop`

并在本地重建：

- `text_delta`
- `input_json_delta`
- `thinking_delta`
- `signature_delta`
- `citations_delta`
- `compaction_delta`

### 一个关键发现：server tool use

`WebSearchTool.call(...)` 不是自己发 HTTP 搜索，而是再次调用 `HSt(...)` / `Hpc(...)` 这条模型调用链去驱动 `server_tool_use`。

这说明模型调用门面并不只服务于普通聊天或本地 tool use，也会被拿来驱动 server-side tool。

`web_search` 这一条链现在已经单独拆到 [07-web-search-tool.md](./07-web-search-tool.md)：

- 本页只保留一个结论：模型调用门面不只是普通聊天请求门面，也能驱动 server-side tool
- `web_search` 的 schema、版本、流事件回收、`WebSearchOutput` 与 transcript 模板统一在专题页维护
- 这样可以避免 provider 总览页与专题页双份漂移

### 结论

当前模型调用门面不是普通聊天请求器，而是：

- 普通模型对话请求器
- structured output 生成器
- server-side tool use 驱动器

三合一门面。

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
