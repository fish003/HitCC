# Hpc stream events 与 non-streaming fallback

## 本页用途

- 整理 `Hpc(...)` 主实现、SDK stream event 到内部事件的映射，以及 non-streaming fallback 条件。
- 说明 assistant block 增量、`stream_event`、fallback 返回形态之间的关系。

## 相关文件

- [../04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../09-api-lifecycle-and-telemetry.md](../09-api-lifecycle-and-telemetry.md)
- [../../03-ecosystem/02-remote-persistence-and-bridge.md](../../03-ecosystem/02-remote-persistence-and-bridge.md)

## 流事件处理与 Remote Transport

### `Hpc(...)`：当前已知的真实主实现

当前已能把主链写成：

```text
g7e(...) / HSt(...)
  -> m1o(...)
    -> Hpc(...)
      -> Kur(clientFactory, streamingRequestFn, retryContext)
        -> client.beta.messages.create({ stream: true }).withResponse()
```

其中 `Hpc(...)` 已确认负责：

- 组装最终 request body
  - normalized messages
  - system prompt
  - tools / extraToolSchemas
  - toolChoice
  - betas
  - metadata
  - max_tokens
  - thinking
  - output_config / context_management / speed
- 管理 streaming 本地状态
  - `t`: 当前 partial message
  - `_6`: 当前 content block 数组
  - `l`: 已产出的 assistant 片段
  - `Z6`: usage 聚合
  - `P6`: stop_reason
- 在每个 SDK event 到来后：
  - 先更新本地状态
  - 再 `yield { type: "stream_event", event: J8, ... }`
- 在 `content_block_stop` 时：
  - 把单个 block 包成一条标准 `assistant` 消息
  - 立刻向上游产出
- 在 `message_delta` 时：
  - 更新最近一条 assistant 的 `usage / stop_reason`
  - 必要时补 refusal / max_output_tokens 相关系统错误

这说明上层看到的 `assistant` 不是只在 `message_stop` 后一次性出现，而是“按 block 收口的 assistant 增量片段”和原始 `stream_event` 并行存在。

### stream event 到内部事件的映射

这一层现在已经不再是纯推断：

- SDK 先把 SSE `message_start / message_delta / message_stop / content_block_start / content_block_delta / content_block_stop` 解析成事件对象
- SDK 内部再把 block 增量累积进当前 message：
  - `text_delta` -> 追加到 `text`
  - `citations_delta` -> 追加到 `citations`
  - `input_json_delta` -> 追加 partial JSON，并尝试填充 `tool_use / server_tool_use`
  - `thinking_delta` -> 追加到 `thinking`
  - `signature_delta` -> 写入 thinking signature
  - `compaction_delta` -> 追加 compaction 内容
- SDK 还会额外触发语义化事件：
  - `text`
  - `citation`
  - `inputJson`
  - `thinking`
  - `signature`
  - `compaction`

CLI 这一层 `Hpc(...)` 自己也做了一次 UI/上层语义映射：

- `message_start` -> 记录 `ttftMs`
- `content_block_start(text)` -> UI 进入 `responding`
- `content_block_start(thinking)` -> UI 进入 `thinking`
- `content_block_start(tool_use / server_tool_use / compaction / mcp_tool_use / result 类 block)` -> UI 进入 `tool-input`
- `content_block_delta(text_delta)` -> 推进响应文本
- `content_block_delta(input_json_delta)` -> 推进 tool/server_tool_use 输入 JSON
- `message_delta(stop_reason)` -> 更新 usage 与 stop reason

因此“stream event -> internal event”的主映射链已经基本还原完成。

这里的“result 类 block”现在也能举到更具体：

- `web_search_tool_result`
- `web_fetch_tool_result`
- `tool_search_tool_result`
- `bash_code_execution_tool_result`
- `text_editor_code_execution_tool_result`
- `mcp_tool_result`

这说明 UI/stream adapter 对 tool 协议的识别不是“只懂本地 `tool_use`”，而是统一覆盖：

- 本地 tool
- server-side tool
- MCP tool
- 内建 schema discovery / tool search

### non-streaming fallback：触发条件与返回形态

fallback 现在已经可以写到相当细：

- stream 正常结束，但：
  - 没收到 `message_start`
  - 或收到了 `message_start`，但没有任何完成的 content block
- stream watchdog 超时
- stream 循环抛错，且未禁用 fallback
- streaming endpoint 创建阶段直接 404

进入 fallback 后走的是：

```text
_pc(...)
  -> Kur(clientFactory, nonStreamingRequestFn, retryContext)
  -> client.beta.messages.create({ stream: false })
```

`_pc(...)` 的行为也已明确：

- 使用单独 timeout
- 仍复用 `Kur`，所以 fallback 期间仍可能产出 `system/api_error`
- 自己不会产出 `stream_event`
- 只在完成后返回最终 assistant message payload
- `Hpc(...)` 再把这个 payload 包成标准 `assistant` 事件向外 `yield`

另外还有两个很具体的 fallback 分支：

- 若主 streaming 路径失败，fallback 会继承 `initialConsecutive529Errors: qk6(error) ? 1 : 0`
- 若 streaming 创建阶段直接 404，也会进入同一条 non-streaming fallback，而不是立刻 fatal

### stream idle watchdog 与 partial retention 边界

当前 byte-stream watchdog 已经可以写成独立机制，证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:162020-162125`：

- idle timeout 先看 `CLAUDE_BYTE_STREAM_IDLE_TIMEOUT_MS`，否则按 provider/default 与 `tengu_byte_stream_idle_timeout_ms` remote config 取值，再夹在最小/最大边界内。
- watchdog 会定期记录 `[Stall] stream_idle_partial ... bytesTotal ... idleDeadlineMs`，真正触发时记录 `[byte-watchdog] firing` 与 `cli_byte_watchdog_fired`。
- 如果检测到系统 sleep/suspend 导致定时器严重 late，会走 sleep/suspend error，而不是把它误报成普通 server stall。
- watchdog 触发会让 stream 进入错误路径，再由上面的 non-streaming fallback / retry 容器决定是否继续。

mid-stream partial retention 只能写本地可见边界：bundle 中有 partial retention / retained text 提示位，但具体哪些服务端错误一定保留哪段 assistant 文本，仍要按 `Hpc(...)` 的事件重建与 tombstone 分支判断，不能泛化成“所有 mid-stream 失败都会保留完整 partial”。

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
