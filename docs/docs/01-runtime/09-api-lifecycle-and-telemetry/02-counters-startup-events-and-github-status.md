# Counter、startup telemetry 与 GitHub 状态

## 本页用途

- 整理 `Fd8(...)` counter 注册点与 `tengu_startup_telemetry` 事件字段。
- 说明 startup telemetry 与 `initializeTelemetry()` 的分工。

## 相关文件

- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../06-stream-processing-and-remote-transport.md](../06-stream-processing-and-remote-transport.md)
- [../../02-execution/06-context-runtime-and-tool-use-context.md](../../02-execution/06-context-runtime-and-tool-use-context.md)
- [../../05-appendix/02-evidence-map.md](../../05-appendix/02-evidence-map.md)

## `Fd8(...)`：counter 注册点

`Fd8(meter, createCounter)` 当前已直接确认注册这些 counters：

- `claude_code.session.count`
- `claude_code.lines_of_code.count`
- `claude_code.pull_request.count`
- `claude_code.commit.count`
- `claude_code.cost.usage`
- `claude_code.token.usage`
- `claude_code.code_edit_tool.decision`
- `claude_code.active_time.total`

这说明 session 级统计不是散落在业务代码里随意打点，而是：

- 先在 telemetry init 成功后统一注册
- 再由全局状态对象 `V8` 保存 counter 引用
- 后续业务路径只做 `counter.add(...)`

## startup telemetry 不是 `initializeTelemetry()` 的一部分

`s4m(...)` 会打：

- `q("tengu_startup_telemetry", {...})`

但它有自己独立的 gating：

- 若 `NB()` 为真，直接返回

而 `NB()` 又等于：

- 使用 3P provider
- 或 `Khe() === true`

而 `Khe()` 来源于：

- `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`
- `DISABLE_TELEMETRY`
- `DO_NOT_TRACK`

所以 startup telemetry 的结论应写成：

- 它不是 OTEL exporter 的一部分
- 它属于 1P internal event logging
- 但会被 3P provider、`DISABLE_TELEMETRY`、`DO_NOT_TRACK`、`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 一起压掉

### `tengu_startup_telemetry` 字段表

当前 `2.1.197` 的 readable bundle 中，对应入口是 `s4m(...)`。它会并行读取 `gb()`、`eIe()`、`ucr()`，再拼上 sandbox、UI preference、settings/env 摘要和证书配置侧标志位后调用 `q("tengu_startup_telemetry", ...)`。
旧文档里的 `amz() / RH() / bP6() / rp8()` 只能作为历史别名理解，当前复核应以 `s4m(...)` 为准。

当前能直接写实的字段如下，见 `package/preprocessed/cli.extracted.bundle.pretty.js`。

| 字段 | 来源 | 当前可确认语义 |
| --- | --- | --- |
| `is_git` | `gb()` | 当前工作目录是否处于 Git 仓库语境 |
| `worktree_count` | `eIe()` | 当前仓库可见的 worktree 数 |
| `gh_auth_status` | `ucr()` | 本地 `gh` CLI 鉴权状态；当前可确认枚举为 `not_installed / authenticated / not_authenticated` |
| `sandbox_enabled` | `$o.isSandboxingEnabled()` | sandbox 总开关是否开启 |
| `are_unsandboxed_commands_allowed` | `$o.areUnsandboxedCommandsAllowed()` | 是否允许执行非 sandbox 命令 |
| `is_auto_bash_allowed_if_sandbox_enabled` | `$o.isAutoAllowBashIfSandboxedEnabled()` | sandbox 开启时是否自动放行 bash |
| `auto_updater_disabled` | `nge()` | auto updater 是否被禁用 |
| `prefers_reduced_motion` | `Lr().prefersReducedMotion ?? false` | UI reduced motion 偏好 |
| `theme` | `pc("theme", "dark").value` | 当前主题设置值 |
| `set_env_var_count` / `set_env_vars` | `$Nc()` | settings/env 中已设置 env var 的数量和逗号连接的键名摘要 |
| `nondefault_setting_count` / `nondefault_settings` | `ONc(e)` | 当前启动配置中非默认设置的数量和键名摘要 |
| `set_user_settings_count` / `set_user_settings` | `NNc(Lr())` | 用户级 settings 中已设置键的数量和键名摘要 |
| `has_node_extra_ca_certs` | `process.env.NODE_EXTRA_CA_CERTS` | 是否配置额外 CA 证书环境变量 |
| `has_client_cert` | `process.env.CLAUDE_CODE_CLIENT_CERT` | 是否配置 client cert |
| `has_use_system_ca` | `vQe("--use-system-ca")` | 是否显式传入 `--use-system-ca` |
| `has_use_openssl_ca` | `vQe("--use-openssl-ca")` | 是否显式传入 `--use-openssl-ca` |
| `cert_store` | `process.env.CLAUDE_CODE_CERT_STORE` | 仅在设置时记录 cert store 配置值 |

这里还有一个边界要写清：

- `o4m()` 只上报这些 CA / cert 开关是否存在
- `NODE_EXTRA_CA_CERTS` 与 `CLAUDE_CODE_CLIENT_CERT` 当前只记录 presence，不把证书路径或证书内容直接塞进 startup telemetry
- `CLAUDE_CODE_CERT_STORE` 是例外：当前会记录配置值本身，因此重写或审计时不能把所有证书相关字段都概括成布尔 presence

这些字段不是证书信任行为本身。实际 TLS / CA trust 解析在网络层完成：默认 `CLAUDE_CODE_CERT_STORE` 等价于 `bundled,system`，会合并 runtime bundled root CA 与 system CA；`NODE_EXTRA_CA_CERTS` 作为追加 CA。该运行时行为见 [../11-non-llm-network-paths.md](../11-non-llm-network-paths.md) 的“共享 TLS / CA trust 层”。

### `gh_auth_status` 的枚举值已经可以钉死

`gh_auth_status` 对应 `ucr()`，当前本地实现非常具体，见 `package/preprocessed/cli.extracted.bundle.pretty.js`：

- 先 `Qw("gh")` 检查本机是否存在 `gh` 可执行文件
- 若不存在，返回 `not_installed`
- 若存在，执行 `gh auth token`
  - `timeout=5000`
  - `stdout/stderr` 都忽略
  - `reject=false`
- `exitCode === 0` 时返回 `authenticated`
- 否则返回 `not_authenticated`

因此这不是一个泛化的“GitHub 登录状态”，而是更窄的：

- 本地 `gh` CLI 是否安装
- 以及 `gh auth token` 是否能在 5 秒内成功返回 token

当前没看到它会：

- 区分多 host / 多 profile
- 区分 token scope
- 复用 Claude 自己的 GitHub import-token / remote session 鉴权状态

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
