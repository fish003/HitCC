# 控制面 API 与外围服务：OAuth 与 Account Control Plane

## OAuth 与 account control plane

### OAuth authorize / token 交换

当前配置对象 `Ns()` 已直接给出：

- `CONSOLE_AUTHORIZE_URL`
- `CLAUDE_AI_AUTHORIZE_URL`
- `TOKEN_URL`
- `MANUAL_REDIRECT_URL`

主要接口：

- `POST ${TOKEN_URL}`
  - 用途：authorization code exchange、refresh token
  - 调用入口：`c2r(...)` 负责 authorization code exchange，`Bte(...)` 负责 refresh token，`A1(...)` 负责 revoke
  - 认证：无 bearer，直接发 OAuth grant payload

### `/api/oauth/profile`

- 用途：读取 account / organization profile
- 调用入口：`wxe(accessToken)` 直接请求 profile，`mAn(accessToken)` 做 subscription / rate-limit / billing 语义映射
- 认证：Bearer access token
- 与 org 的关系：
  - 返回体里直接含 organization 信息
  - `kj()` 会优先读 `CLAUDE_CODE_ORGANIZATION_UUID` 与本地 `oauthAccount.organizationUuid`，缺失时再从这里兜底拿 `organization.uuid`

### `/api/oauth/claude_cli/roles`

- 用途：拉取 `organizationRole / workspaceRole / organizationName`
- 调用入口：`u2r(accessToken)`
- 认证：Bearer access token
- 与 org 的关系：返回的是当前 OAuth account 在 org/workspace 里的角色语义

### `/api/oauth/claude_cli/create_api_key`

- 用途：把 OAuth 登录安装成可持久化的 first-party API key
- 调用入口：`d2r(accessToken)`
- 认证：Bearer access token
- 与 org 的关系：
  - scope 里显式出现 `org:create_api_key`
  - 这不是模型调用，而是 account -> API credential 的控制面转换

### scope 族

当前已确认的 OAuth scope 至少包括：

- `user:inference`
- `user:profile`
- `org:create_api_key`
- `user:sessions:claude_code`
- `user:mcp_servers`
- `user:file_upload`

这说明 OAuth token 不只是登录态，而是控制面 capability 的总开关。

### 当前证据边界

- host、client id、scope、`TOKEN_URL`、`API_KEY_URL` 与 `ROLES_URL` 来自 `Ns()`。
- authorize URL 构造由 `fAn(...)` 负责，会根据 `loginWithClaudeAi` 在 `CLAUDE_AI_AUTHORIZE_URL` 与 `CONSOLE_AUTHORIZE_URL` 间选择。
- 当前 bundle 能证明本地 token/profile/role/API-key 转换链；服务端对每个 scope 的实际授权策略仍属于服务端黑箱。

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
