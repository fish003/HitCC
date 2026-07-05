# 控制面 API 与外围服务：Remote Environments 与 Sessions

## Remote / environment / code session control plane

### 环境枚举与创建

#### `GET /v1/environment_providers`

- 用途：列出远程环境
- 调用入口：`Hoe(...)`
- 认证：Bearer + `x-organization-uuid`
- 返回：`environments`

当前本地能正证到的消费字段至少包括：

- `environment_id`
- `name`
- `kind`

这些字段在本地直接用于：

- 默认环境选择
- UI 环境列表展示
- teleport / remote task / schedule 的环境绑定

当前没看到主流程依赖更多 provider-specific 深字段。

#### `POST /v1/environment_providers/cloud/create`

- 用途：创建 `anthropic_cloud` 默认环境
- 调用入口：`Smt(name, signal, accessToken)`，以及 remote setup 的默认环境兜底逻辑
- 认证：Bearer + `x-organization-uuid` + `ccr-byoc-2025-07-29`
- 载荷包含：
  - `environment_type`
  - `cwd`
  - `languages`
  - `network_config`

因此 environment provider 不是 UI 配置概念，而是服务端可创建、可枚举的执行环境实体。

### 本地对 environment 列表的选择策略

环境列表的本地消费逻辑现在也已足够写实：

- 若 settings 里有 `remote.defaultEnvironmentId`
  - 优先按 `environment_id` 精确命中
- 否则优先选：
  - `kind === "anthropic_cloud"`
- 再否则退化成：
  - 第一个 `kind !== "bridge"` 的环境
  - 再不行才是列表第一个

因此本地 bundle 当前明确区分了：

- 可执行默认环境
- bridge 自己注册出来的环境

不是把所有 `environments[]` 当成同一类对象。

### environment 列表的实际消费面

继续追本地调用点后，这组字段的消费面可以再收紧一层：

- remote environment picker
  - 列表项只展示：
    - `name`
    - `environment_id`
  - 默认值选择逻辑仍只依赖：
    - `environment_id`
    - `kind`
- `/schedule`
  - 会把环境列表渲染成：
    - `name`
    - `environment_id`
    - `kind`
  - 若列表为空，会尝试 `POST /v1/environment_providers/cloud/create`
  - 新建成功后，把返回对象直接作为唯一可选环境继续后续 prompt 组装
- teleport / remote task
  - 最终真正写回 create session body 的也只是：
    - `environment_id`

当前仍没看到主流程去消费 provider 返回对象里的更深层 `config`、network 配置或 provider-specific metadata。  
因此至少对 CLI 本地主链来说，remote environment 更像：

- 可枚举的远端执行槽位
- 其核心身份字段是 `environment_id`
- `name / kind` 主要用于选择和展示

### code session 主接口

#### `GET /v1/code/sessions` 与 SDK `GET /v1/sessions`

- 用途：列出远程 code sessions
- 调用入口：当前 UI/session 列表路径会走 `BASE_API_URL /v1/code/sessions` 侧的列表与 rehydrate 链；managed-agents SDK 也暴露 `client.beta.sessions.list(...)` 到 `/v1/sessions?beta=true`
- 认证：Bearer + `x-organization-uuid` + `ccr-byoc-2025-07-29`
- 本地会从 `session_context.sources` 中回推 git repo 信息

#### `POST /v1/sessions`

这是远程控制面的核心创建接口，至少有两条本地入口：

1. `_q(...)`
   - 用于 teleport / remote task 创建
2. `u0f(...)`
   - 用于 `/remote-control` 启动后的远程会话创建

共同 payload 语义可收敛成：

- `title`
- `events`
- `session_context`
  - `sources`
  - `outcomes`
  - `model`
  - `seed_bundle_file_id`
  - `environment_variables`
- `environment_id`
- 在 bridge 路径上还能看到：
  - `source: "remote-control"`
  - `permission_mode`

teleport / remote task 路径里，`session_context` 还能再收紧为：

- `sources`
  - git repo source
  - 或空数组
- `seed_bundle_file_id`
  - bundle 上传成功时出现
- `outcomes`
  - GitHub repo 语义 outcome 等
- `environment_variables`
  - 显式透传
- `model`
  - 当前远端会话绑定的模型

这说明 remote session create 不是简单“发一句 prompt”，而是同时绑定：

- repo/bundle 来源
- 运行环境
- 初始事件流
- permission mode

#### `GET /v1/sessions/{id}`

- 用途：拉单个 session 详情
- 调用入口：bridge helper `x7t(...)`；managed-agents SDK 也有 `client.beta.sessions.retrieve(...)`
- 认证：Bearer + `x-organization-uuid` + `ccr-byoc-2025-07-29`

#### `POST /v1/sessions/{id}/events`

- 用途：向远端 session 注入用户消息或控制响应
- 调用入口：
  - managed-agents SDK 的 `client.beta.sessions.events.send(...)`
  - bridge `sendPermissionResponseEvent(...)`
- 认证：Bearer + `x-organization-uuid`
- 与 session 的关系：
  - body 里显式带 `events[]`
  - 每条 event 含 `session_id`

#### `PATCH /v1/sessions/{id}`

- 用途：改标题
- 调用入口：bridge `TLl(...)`，其标题包装为 `EOo(...)`
- 认证：Bearer + `x-organization-uuid`

#### `POST /v1/sessions/{id}/archive`

- 用途：归档远端 session
- 调用入口：
  - remote task 的 `bq(id)`
  - bridge `d0f(...)`
- 认证：Bearer + `x-organization-uuid`
- 特征：`409` 会被视为“已归档”而不是硬错误

#### `GET /v1/code/sessions/{id}/events/stream`

- 用途：远端 code session 的实时事件订阅通道
- 调用入口：`kWo.connect()`
- 认证：`authHeaders()` 产生的 Bearer / org 相关 header
- 特征：SSE，带 `Accept: text/event-stream`；断线恢复会使用 `from_sequence_num` 和 `Last-Event-ID`

所以 remote session 不是纯 REST；当前 `2.1.197` 本地可见的实时通道是：

- REST control path
- SSE event stream path

### 当前证据边界

- CLI remote task / remote-control 使用的是本地可见的 `environment_providers`、`/v1/sessions` 与 `/v1/code/sessions/{id}/events/stream` 链。
- 抽取 bundle 内还包含 managed-agents SDK 的 `/v1/sessions?beta=true`、`/v1/sessions/{id}/events/stream?beta=true`、resources、threads 等 endpoint wrapper；这些属于 SDK surface，不应直接等同于 Claude Code CLI 的 remote-control 主链。
- 当前没有在本地 bundle 中正证到旧文档写法里的 `sessions/ws/{id}/subscribe` WebSocket 主链；若后续重新发现，应作为独立兼容路径补回。

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
