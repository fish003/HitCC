# 流事件处理与 Remote Transport

## 本页用途

- 作为 `Hpc(...)` stream event、non-streaming fallback、request lifecycle telemetry 与 remote transport 的导航页。
- 原正文已经拆成三个子页；本页只维护阅读顺序和跨专题边界。

## 相关文件

- [04-agent-loop-and-compaction.md](./04-agent-loop-and-compaction.md)
- [05-model-adapter-provider-and-auth.md](./05-model-adapter-provider-and-auth.md)
- [09-api-lifecycle-and-telemetry.md](./09-api-lifecycle-and-telemetry.md)
- [07-web-search-tool.md](./07-web-search-tool.md)
- [08-web-fetch-tool.md](./08-web-fetch-tool.md)
- [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)
- [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md)

## 拆分后的主题边界

### Hpc stream events 与 non-streaming fallback

见：

- [06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md](06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md)

集中放 `Hpc(...)` 主实现、stream event 映射、assistant 增量产出与 fallback 返回形态。

### 请求 telemetry、refusal 与 stream status

见：

- [06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md](06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md)

集中放 `NDl / KOo / UDl`、`api_refusal`、model block、`stream_request_start` 与本模块剩余状态。

### Remote transport 与 request build 最后边界

见：

- [06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md](06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md)

集中放 `sdk-url` / bridge / ingress transport、远程事件 replay 与本地 request build 边界。

## 建议阅读顺序

1. 先看 [01-hpc-stream-events-and-non-streaming-fallback.md](./06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md)，建立 stream processing 主链。
2. 再看 [02-request-telemetry-refusal-and-stream-status.md](./06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md)，确认 request lifecycle telemetry 与 refusal/model block。
3. 最后看 [03-remote-transport-and-request-build-boundary.md](./06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md)，补齐 remote transport 和 request build 边界。

## 与其它专题的边界

- provider/auth、model selection 与 request body 上游装配，见 [05-model-adapter-provider-and-auth.md](./05-model-adapter-provider-and-auth.md)。
- telemetry 初始化和开关矩阵，见 [09-api-lifecycle-and-telemetry.md](./09-api-lifecycle-and-telemetry.md)。
- remote session、bridge credentials 与 worker lifecycle，见 [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)。

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
