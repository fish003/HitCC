# API 请求生命周期、Telemetry 初始化与开关矩阵

## 本页用途

- 作为 telemetry 初始化、startup event、1P event logging 与流量开关主题的导航页。
- 原正文已经拆成四个子页；本页只维护阅读顺序和跨专题边界。

## 相关文件

- [05-model-adapter-provider-and-auth.md](./05-model-adapter-provider-and-auth.md)
- [06-stream-processing-and-remote-transport.md](./06-stream-processing-and-remote-transport.md)
- [../02-execution/06-context-runtime-and-tool-use-context.md](../02-execution/06-context-runtime-and-tool-use-context.md)
- [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)
- [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md)

## 拆分后的主题边界

### Telemetry 初始化与 remote settings gating

见：

- [09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md](09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md)

集中放 `Rg8 / H4A / qvz / initializeTelemetry`、`Tu / CF1 / EBq / tL8 / yBq` 与启动期等待链。

### Counter、startup telemetry 与 GitHub 状态

见：

- [09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md](09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md)

集中放 `Fd8(...)` counter 注册、`tengu_startup_telemetry` 字段和 `gh_auth_status` 枚举。

### 1P event logging 配置与 telemetry 设置

见：

- [09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md](09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md)

集中放 `Qg7()` / `gg7()`、remote event logging config、event logger endpoint 与设置矩阵。

### 流量开关、请求生命周期与边界

见：

- [09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md](09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md)

集中放 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`、`DISABLE_TELEMETRY`、request lifecycle telemetry 与 API body logging 边界。

## 建议阅读顺序

1. 先看 [01-initialization-and-remote-settings-gating.md](./09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md)，建立启动期初始化模型。
2. 再看 [02-counters-startup-events-and-github-status.md](./09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md)，区分 counters 与 startup event。
3. 然后看 [03-event-logging-config-and-telemetry-settings.md](./09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md)，理解 1P event logging 与配置来源。
4. 最后看 [04-traffic-switches-request-lifecycle-and-boundaries.md](./09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md)，复核开关矩阵和请求级 telemetry。

## 与其它专题的边界

- 模型请求包装、provider/auth、gateway model discovery 见 [05-model-adapter-provider-and-auth.md](./05-model-adapter-provider-and-auth.md)。
- streaming、fallback、remote transport 与请求错误映射见 [06-stream-processing-and-remote-transport.md](./06-stream-processing-and-remote-transport.md)。
- 非模型出网路径、GrowthBook SSE 和 plugin install count 见 [11-non-llm-network-paths.md](./11-non-llm-network-paths.md)。

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
