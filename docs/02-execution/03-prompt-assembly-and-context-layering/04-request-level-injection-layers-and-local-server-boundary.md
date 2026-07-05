# Request-level 注入分层与本地/服务端边界

## 本页用途

- 单独承接 request-level 注入需要如何分层理解，以及本地 request build 与服务端黑箱之间的边界。
- 把 `prompt-text`、`schema/request options`、`transport` 三层拆开，并收口 `Hpc(...)` 落到 provider 调用前本地还能证明什么。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../04-non-main-thread-prompt-paths.md](../04-non-main-thread-prompt-paths.md)
- [../../01-runtime/06-stream-processing-and-remote-transport.md](../../01-runtime/06-stream-processing-and-remote-transport.md)
- [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)

## request-level 注入与黑箱边界

### request-level 注入要分成 `prompt`、`schema/request options`、`transport` 三层

“request-level 注入”如果不分层，很容易把显式 request 字段和黑箱 context 混在一起。  
当前 `Hpc(...)` 的 request builder 已足够把这三层拆开：

```text
payload = {
  model: Gu(model),
  messages: Glm(Ipc(messagesForAPI, ...), ...),
  system: Wlm(systemSections, ...),
  tools: Hci(toolSchemas, model),
  tool_choice,
  betas,
  metadata: XLe(),
  max_tokens,
  thinking,
  temperature?,
  context_management?,
  output_config?,
  speed?
}
-> client.beta.messages.create(payload)
```

因此更稳的分层应是：

#### 1. prompt-text 注入

这才是本页真正讨论的 prompt layering 本体：

- `messages = Glm(Ak(...), ...)`
- `system = Wlm(Ec(...), ...)`

其中 `Hpc(...)` 还能直接确认几类“进入 system 文本块本体”的 request-level 补段：

- `tTn(...)` 生成的 attribution / billing header section
- `$Rn(...)` 生成的 Claude Code / Agent SDK identity section
- `systemPrompt selection` 产物本身
- 若 advisor model 命中，`Nol` 这类在 `Ec(...)` 前显式拼入 `system sections` 的本地段

也就是说：

- 本地确实存在 request build 阶段的附加 system section
- 但它们仍是 **`Hpc(...)` 内显式可见的本地拼装**
- 不是“发送后又被某个本地黑箱二次追加”

#### 2. schema / request-options 注入

这些字段会改变一次请求的能力与行为，但**不属于 prompt 文本**：

- `tools`
- `tool_choice`
- `betas`
- `metadata`
- `max_tokens`
- `thinking`
- `temperature`
- `context_management`
- `output_config`
- `speed`
- `fallbacks / fallback_credit_token`
- `diagnostics.previous_message_id`

这里最容易误写的是两项：

- `advisor_20260301` 是通过 `tools` 注入的额外 server tool schema，不是 system 文本段
- `context_management` / `output_config` 是 request options，不是额外 `context` 文本
- `metadata: XLe()` 的字段名是 `user_id`，但值是包含 `device_id / account_uuid / session_id / CLAUDE_CODE_EXTRA_METADATA` 的 JSON 字符串；它是 request metadata，不是 prompt 文本

#### 3. transport / headers

再往外一层是 provider client 与 transport：

- `f8(...)` / provider wrapper 负责 provider client、默认 headers、auth 与 request wrapper
- `sdk-url / remote-control / bridge` 负责 transport/ingress
- `Apc(...)` 负责按条件生成 `x-client-request-id` 和 `traceparent`
- `Hpc(...)` 会在特定 first-party 条件下补 `anthropic-usage-limit` 或 `anthropic-dispatch-id`

它们会影响**请求如何被发送**，但从当前本地可见代码看：

- 不会在 `Hpc(...)` 之后再改写 `messages/system`
- 也没有看到第二个本地 prompt builder 在 transport 层重新拼 context
- header / trace / request metadata 的完整关联矩阵放在 [06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md](../../01-runtime/06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md)

### 本地 request build 与服务端黑箱的边界

围绕“是否还会额外追加 `context / compat / verification`”，这一页现在可以把本地边界写得更硬。

当前本地可见的主 request-level payload producer 是 `Hpc(...)`。

其中主链 `Hpc(...)` 的最后落体已经是：

```text
payload = {
  messages: Glm(Ipc(messagesForAPI, ...), ...),
  system: Wlm(systemSections, ...),
  tools,
  ...
}
-> client.beta.messages.create(payload)
```

少量专用 side-query/helper 路径如果不走主 loop，也仍然是在本地显式组 payload 后直接调用 provider；当前没有看到一层隐藏的本地 prompt proxy 在 provider 调用前二次拼接 `system/messages`。

因此本地 bundle 当前能直接确认：

- `Hpc(...)` 完成 payload 后，没有看到第二次本地 request-level `system/messages` 注入
- 专用 helper 也是本地显式拼 payload 后直接调用 provider，不是隐藏 prompt proxy
- `sdk-url / remote-control` 在本地可见代码里只换 transport / ingress，不改 `userContext / systemContext / system`
- 如果还存在额外的 `context / compat / verification` 拼装，更可能位于：
  - `client.beta.messages.create(...)` 之后的 provider / first-party 服务端黑箱
  - 而不是当前 CLI 本地链路

但这一页仍应保留一个克制边界：

- 本地代码只能证明 **发送前最后一个可见 payload producer 到这里为止**
- 不能反证服务端绝对不会再做：
  - server-side system augmentation
  - server tool wiring
  - verification / compat / policy prompt wrapping
- 因此“服务端是否还会追加黑箱 context”目前仍然属于 **本地 bundle 无法正证或反证的剩余未知点**

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
