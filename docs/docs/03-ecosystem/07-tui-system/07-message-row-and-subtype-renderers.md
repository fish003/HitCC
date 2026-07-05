# Message Row 与 Subtype Renderer

## 本页用途

- 把 `Aj6 / k8f(...)` 下游的单条消息渲染器拆成可还原的 subtype 表。
- 固定 top-level message type、content block type 和若干折叠态之间的边界。

## 相关文件

- [../07-tui-system.md](../07-tui-system.md)
- [01-repl-root-and-render-pipeline.md](./01-repl-root-and-render-pipeline.md)
- [02-transcript-and-rewind.md](./02-transcript-and-rewind.md)
- [08-tool-result-renderers.md](./08-tool-result-renderers.md)

## 渲染链入口：`o6l -> gVf(...) -> UQ/Xdf(...)`

当前 2.1.197 readable bundle 里，单条 renderable message 的活链已经能落到：

- `o6l`
  - `memo(gVf, bVf)`，负责单条 row 的 memo / static 判定。
- `gVf(...)`
  - 负责 grouped / collapsed 行的外层归一、动画态、static 态、timestamp/model 行头。
- `UQ`
  - `memo(Xdf, Zdf)`，负责真正的 top-level `message.type` 分派。
- `Xdf(...)`
  - 继续分到 user / assistant / system block renderer 与 grouped / collapsed renderer。

这两层的职责已经可以分开写：

### `gVf(...)`

负责：

- 判断当前消息是不是：
  - `grouped_tool_use`
  - `collapsed_read_search`
- 计算这一行是否处于 in-progress / active collapsed group
- 计算静态态与动画态
- transcript 下为 assistant 行补：
  - 时间戳
  - model 标签

### `Xdf(...)`

负责：

- 按 `message.type` 选最终 renderer
- assistant 行继续拆 content block subtype
- user 行继续拆 content block subtype

## 顶层 `message.type` 分派表

`Xdf(...)` 当前已确认的 top-level 分支如下。

### 1. `attachment`

走：

- `Y74(...)`

说明 attachment 在消息区有独立 renderer，不混入 user/assistant block。

### 2. `assistant`

每个 `message.content[]` block 继续交给：

- `Qdf(...)`

因此 assistant 不是“整条消息一个 renderer”，而是 block list。

### 3. `user`

分两种：

- `isCompactSummary`
  - 走 compact summary renderer
- 普通 user content
  - 每个 block 走 `Jdf(...)`

### 4. `system`

分几种已知稳定分支：

- `compact_boundary`
  - compact boundary renderer
- `microcompact_boundary`
  - 直接不显示
- `read_divider`
  - 直接渲染 inactive divider
- `local_command`
  - 先包装成 text block，再走 text renderer
- 其余 system
  - 走 `Hpl(...)`

这里要特别注意：  
`Hpl(...)` 不是“system subtype 总表”，而是**已经进入消息区、且不属于顶层特判分支**的 system row renderer。

### 5. `grouped_tool_use`

走：

- `mpl(...)`

它会调用具体 tool 的 `renderGroupedToolUse(...)`。

### 6. `collapsed_read_search`

走：

- `cpl(...)`

这是 read/search/repl/memory 操作的折叠摘要行。

## user block subtype：`Jdf(...)`

当前已确认只有三种活 block：

### `text`

走：

- text renderer

它会拿到：

- `planContent`
- `timestamp`
- `isTranscriptMode`

说明普通文本与 plan 内容高亮在这里汇合。

### `image`

走：

- image renderer

使用 `imageIndex` 生成稳定图片编号。

### `tool_result`

走：

- `zpl(...)`

它会接到：

- `lookups`
- `progressMessagesForMessage`
- `tools`
- `style`
- `isTranscriptMode`

也就是说 tool result 不是简单把结果对象打印出来，而是可结合 tool metadata 二次渲染。

这里还需要特别补一句：

- `zpl(...) -> Fpl(...)` 的正常结果分支，实际吃的是 `message.toolUseResult`
- 不是 transcript block 里的 `tool_result.content`

`zpl(...)` 自身的分支表、特殊拒绝/错误字符串、以及 `Read/Bash/Edit/Notebook/WebFetch/WebSearch` 等高价值 renderer 家族，已经继续拆到：

- [08-tool-result-renderers.md](./08-tool-result-renderers.md)

## assistant block subtype：`Qdf(...)`

assistant content block 的已知分支更完整。

### `tool_use`

走：

- tool-use renderer

这是本地工具调用、进度、动画、dot、resolved/error lookup 的主入口。

### `text`

走：

- text renderer

这是普通 assistant 文本 renderer。

### `redacted_thinking`

走：

- redacted-thinking renderer

且在非 transcript、非 verbose 时直接隐藏。

### `thinking`

走：

- thinking renderer

并根据 `lastThinkingBlockId` 决定 transcript 中是否隐藏旧 thinking。

### `server_tool_use`
### `advisor_tool_result`

这两类共走：

- server/advisor tool block renderer

前提是 `sA4(K)` 判定通过。  
说明 server-side tool / advisor result 在 UI 层共享一套 block renderer。

## `grouped_tool_use`：不是 message list 的普通行，而是 tool 自定义聚合 renderer

`mpl(...)` 当前行为很关键：

- 先按 `tool_use_id` 关联每条 tool_use 与对应 user-side tool_result
- 对每个子项计算：
  - `isResolved`
  - `isError`
  - `isInProgress`
  - `progressMessages`
  - `result`
- 然后调用 tool 自己的：
  - `renderGroupedToolUse(...)`

也就是说 grouped tool use 的最终视觉，不是 TUI 固定写死，而是工具可自定义。

### 当前 bundle 里已直接确认的实现者

继续往下追 `renderGroupedToolUse` 的实现后，当前本地 bundle 里能直接确认的 concrete 实现者只有一类：

- `fq`
  - agent / subagent / teammate 启动工具
  - renderer 实现是 `g74(...)`

而且这一点现在还能再收紧：

- 全 bundle 继续搜索后，`renderGroupedToolUse(...)` 的消息区调用站点只有 `mpl(...)`
- `renderGroupedToolUse : ...` 的实现站点也只看到这一处：
  - `fq.renderGroupedToolUse = g74`

因此对“当前本地 bundle”来说，更稳的结论已经不是“目前主要看到 agent 工具族”，而是：

- **当前只看到 agent/subagent 工具族这一种实现者**

`g74(...)` 会把同一条消息里的多次 agent 启动聚合成一棵小树，按子项展示：

- agentType
- description
- taskDescription
- toolUseCount
- tokens
- lastToolInfo
- isAsync / isResolved / isError

顶部 summary 还会区分：

- `Running N agents…`
- `N agents launched`
- `N agents finished`

这说明当前 `grouped_tool_use` 至少不是“很多工具都在用”的普遍机制；  
本地可见活实现里，最明确的是 agent/tool-family。

## `collapsed_read_search`：是操作摘要，不是 transcript 原文

`cpl(...)` 汇总的内容至少包括：

- search count
- read count
- list count
- repl count
- bash count / git-op bash count
- edit file count 与 added/removed line summary
- agent count / other tool count / frame count
- MCP 调用计数
- memory read/search/write
- team memory read/search/write
- hook 执行统计
- latest display hint
- latest thinking summary / thought duration

verbose 下它还能展开成更细的逐条工具视图。  
因此 `collapsed_read_search` 是一种“操作摘要消息”，不是普通 assistant/user transcript。

## `Hpl(...)`：system row 的真实分派

`Hpl(...)` 现在已经可以直接拆成更完整的分派表。

### 专用 renderer 分支

- `turn_duration`
  - `Odf(...)`
- `memory_saved`
  - `Bdf(...)`
- `away_summary`
  - 专用 away summary renderer
- `agents_killed`
  - 内联专用行
- `thinking`
  - 直接隐藏
- `model_refusal_no_fallback`
  - 直接隐藏
- `model_refusal_fallback`
  - 在 feature gate 命中时显示 warning 与 `/config` 提示
- `model_fallback`
  - warning 行
- `bridge_status`
  - `jdf(...)`
- `scheduled_task_fire`
  - inline status 行
- `permission_retry`
  - inline `Allowed <commands>` 行
- `api_error`
  - 当前 `Hpl(...)` 内直接返回 `null`
- `stop_hook_summary`
  - `Rdf(...)`

### 通用字符串分支

若不命中上面的专用分支，且 `message.content` 是字符串，则落回 `Tx_(...)`。

这一层的显示规则是：

- `level === "warning"`
  - warning 色
- `level !== "info"`
  - 显示 dot
- `level === "info"`
  - dimColor

另外还有一个重要过滤：

- `stop_hook_summary` 例外
- 其余 `level === "info"` 的 system 行，在非 verbose 下会被 `Hpl(...)` 直接隐藏

这说明很多 system subtype 即便进入 `Hpl(...)`，也不代表默认就会在普通聊天视图里出现。

### 这条隐藏规则是主聊天默认行为，不是 transcript 总规则

这一点也已经能和 REPL root 对上：

- 普通主聊天页传给 `Aj6(...)` 的 `verbose` 取决于当前设置/模式
- `screen === "transcript"` 的专用分支会把 `verbose` 直接置为 `true`

因此更稳的可见性判断是：

- 普通聊天视图
  - `Hpl(...)` 会把大多数 `level:"info"` system 行直接压掉
- transcript 视图
  - 由于 `verbose: true`，这批 system 行更容易被看见

所以“代码里产出了 system/informational”与“用户在主聊天页默认能看到它”不是一回事。

## `xo(...)`：只有少数 system 行会绑定到某个 `tool_use`

消息区的关联 lookup 并不是对所有 system 行一视同仁。

`xo(...)` 当前对 `system` 的处理只有一条活分支：

- `subtype === "informational"`
  - 且带 `toolUseID`
  - 才会回报对应 `tool_use_id`

这意味着：

- 像 `Tool <name> running for ...` 这类 informational 行，可以挂回某个具体工具
- 但大多数 `system` 行并不会参与 tool-use 级聚合

因此 `system` row 在消息区里更像两类对象：

- 与某次工具调用局部相关的 informational sideband
- 与整轮运行态相关的全局状态/提示

## 当前已能直接枚举的 system subtype producer

把 system message producer、`Xdf(...)` 顶层分支、以及 `Hpl(...)` 内部分支放在一起看，当前至少能直接枚举这些 subtype：

- `compact_boundary`
- `microcompact_boundary`
- `read_divider`
- `local_command`
- `informational`
- `bridge_status`
- `stop_hook_summary`
- `turn_duration`
- `memory_saved`
- `away_summary`
- `agents_killed`
- `model_refusal_no_fallback`
- `model_refusal_fallback`
- `model_fallback`
- `model_consent_fallback`
- `scheduled_task_fire`
- `permission_retry`
- `api_error`
- `api_retry`
- `hook_started`
- `hook_progress`
- `hook_response`
- `task_notification`
- `task_started`
- `task_progress`
- `status`
- `notification`
- `thinking_tokens`
- `init`
- `session_state_changed`
- `elicitation_complete`
- `bridge_state`
- `permission_denied`
- `local_command_output`

其中更稳的可见性边界是：

- `compact_boundary / microcompact_boundary / local_command`
  - 在 `Xdf(...)` 顶层就有特判
- `read_divider`
  - 在 `Xdf(...)` 顶层渲染成 divider
- `turn_duration / memory_saved / away_summary / agents_killed / bridge_status / model_* / scheduled_task_fire / permission_retry / stop_hook_summary`
  - 在 `Hpl(...)` 里有专用 renderer 或专门隐藏规则
- `informational / api_retry / hook_started / hook_progress / hook_response`
  - 更像 `Hpl(...)` 的通用字符串分支候选
- `model_consent_fallback / notification / thinking_tokens / permission_denied / local_command_output / commands_changed / files_persisted / worker_shutting_down / memory_recall`
  - 主要是 SDK / headless / host event surface；进入 TUI 主消息区前通常会被过滤、转换或只作为 sideband 状态消费
- `init / status / task_* / session_state_changed / elicitation_complete / bridge_state`
  - 明显带 protocol / transport / runtime 状态语义
  - 当前 bundle 里能看到 producer，但不等于都会稳定进入主 transcript 行渲染

如果只看当前本地主线程直接产出的 system helper，已经能稳定点名这组：

- `yO(...)`
  - `informational`
- `lb4(...)`
  - `bridge_status`
- `Ij4(...)`
  - `stop_hook_summary`
- `dqA(...)`
  - `turn_duration`
- `kx8(...)`
  - `memory_saved`
- `ib4(...)`
  - `agents_killed`
- `gU(...)`
  - `local_command`
- `bn6(...)`
  - `compact_boundary`
- `rx1(...)`
  - `api_error`

这说明主线程直连模式下的 system 行并不是黑箱事件流，而是本地明确建模的一组内部消息对象。

## protocol/system 与 transcript/system 不是一回事

当前还有两条很关键的边界：

### 1. 持久化/导出不会保留全部 system subtype

`dB8(...)` 与 `AAA(...)` 这两条链当前只明确保留：

- `compact_boundary`
- 特殊 `local_command` 内容

这意味着大量 system subtype 更像运行态 UI 事件，而不是 transcript 长期语义对象。

### 2. remote session adapter 只放行极少数 system subtype

`useRemoteSession.onMessage(...)` 这一层现在已经能写到更细：

- `task_started`
  - 只更新本地 active task 集合
  - 然后直接 `return`
- `task_notification`
  - 只更新/移除本地 active task
  - 然后直接 `return`
- `task_progress`
  - 直接 `return`
- `init`
  - 会先刷新 slash commands
  - 但不会在这里早退
- `status`
  - 会先更新 compacting 标记
  - 但不会在这里统一早退

接下来消息还会进入 `cj6(...)`。  
而 `cj6(...)` 对 `system` 的处理只有三条活分支：

- `init`
  - 映射成 `informational`
  - 文案是 `Remote session initialized (model: ...)`
- `status`
  - 交给 `Ohz(...)`
  - 只有 `status !== null` 才会生成 `informational`
- `compact_boundary`
  - 映射成 `compact_boundary`

其余所有 `system` subtype 都会被打日志后直接 `ignored`。

这意味着对 remote adapter 来说：

- `task_started / task_notification / task_progress`
  - 在 adapter 入口就被消费
- `session_state_changed / bridge_state / elicitation_complete`
  - 即便抵达 adapter，也不会进 transcript message
- `status`
  - 只有携带真实字符串状态时才会落成一条信息行
  - 像 `permissionMode` 变化时那种 `status: null` 事件会被忽略

因此 “bundle 里存在该 subtype” 和 “用户一定会看到一行对应消息” 不能直接画等号；  
至少在 remote session/TUI 这条链上，真正稳定进消息区的 protocol-like `system` 只剩极少数。

### 2.1 SDK / headless schema 还会暴露一组不等价于 TUI row 的 system subtype

当前 schema / print / SDK adapter 里还能看到一些名字，它们容易和 TUI message row 混在一起：

- `model_consent_fallback`
  - schema 与 headless 输出面可见，表示 consent gate 后切到 fallback/default model。
  - 若进入普通 message row，`Hpl(...)` 没有专门分支，只能落到通用字符串 warning 行；它的主证据仍应放在 refusal/fallback 专题。
- `notification`
  - loop-side text notification；headless 汇总会过滤，不是稳定 transcript 正文。
- `thinking_tokens`
  - thinking_delta 估算出来的 live token 进度；用于进度/host surface，不是普通聊天行。
- `permission_denied`
  - SDK adapter 可把它转成 `informational` warning，headless 汇总默认过滤；TUI 主消息区不应把它和 `permission_retry` 混成同一类。
- `local_command_output`
  - SDK adapter 会把它转成 assistant-style text message；它不同于本地 `local_command` system row。
- `commands_changed / files_persisted / worker_shutting_down / memory_recall / task_updated / task_summary / mirror_error`
  - 更接近 SDK / host / control-plane event，当前不应写成主 TUI transcript 的稳定可见 subtype。

所以 subtype 表要拆成三层：

```text
TUI row stable subtype
-> adapter-converted subtype
-> protocol/headless/host event subtype
```

重写时，message renderer 可以为 unknown subtype 保留 registry/fallback，但默认策略应保守：

- unknown top-level `message.type`：记录并忽略，不制造 UI。
- unknown assistant/user block：记录并忽略，不把任意对象当 text。
- unknown system subtype：只有存在 string `content/text` 与明确 level 时才走通用字符串行；否则视为 sideband event。

### 2.5 `bridge_status / bridge_state / session_state_changed` 三者的边界已经能写死

这三个名字很像，但在本地 UI 里的地位明显不同：

- `bridge_status`
  - 是正式 transcript/system 行
  - 文案固定围绕：
    - `/remote-control is active. Code in CLI or at <url>`
  - 走 `Hpl(...)` 专用 renderer
  - 属于**用户可见的产品状态消息**
- `bridge_state`
  - 由 bridge SDK 控制面状态变化主动 enqueue
  - 值至少有：
    - `ready / connected / reconnecting / failed`
  - 但当前 remote adapter / transcript 主链没有把它稳定渲染成普通消息
  - 更像**sideband runtime state**
- `session_state_changed`
  - schema 只有：
    - `idle / running / requires_action`
  - 由 `notifySessionStateChanged` 发射
  - headless 汇总时被显式过滤
  - 当前也没看到本地 transcript/UI 把它稳定转成用户消息
  - 更像**协议级 session 状态信号**

因此当前更稳的可见性判断应是：

- 用户稳定能看到的 remote 状态行
  - 首先是 `bridge_status`
- 运行时内部会流过、但不会稳定落成 transcript 行的
  - `bridge_state`
  - `session_state_changed`

### 3. headless `--print` 结果收集也会主动滤掉一批 protocol event

headless 主循环在汇总 `--print` 输出时，还会显式跳过：

- `control_response / control_request / control_cancel_request`
- `stream_event`
- `keep_alive`
- `streamlined_text`
- `streamlined_tool_use_summary`
- `prompt_suggestion`
- `system.session_state_changed`
- `system.task_notification`
- `system.task_started`
- `system.task_progress`

这说明至少 `session_state_changed / task_*` 这组事件，本地实现已经把它们当作 **sideband protocol/runtime event**，而不是普通 transcript 主体。

## assistant transcript 行为什么会额外显示时间与 model

`gVf(...)` 会在 transcript 模式下，对包含 text block 的 assistant 行追加：

- `gk4(...)`
  - 时间
- `Uk4(...)`
  - model

这说明 transcript 行头信息不是全局 header，而是逐行条件挂载。

## 搜索文本提取也依赖 subtype

`POz(...)` / `XOz(...)` / `DOz(...)` 已经说明：

- assistant `text` 会直接入搜索文本
- assistant `tool_use` 会提取：
  - `command`
  - `pattern`
  - `file_path`
  - `path`
  - `prompt`
  - `description`
  - `query`
  - `url`
  - `skill`
- user `tool_result` 会优先抽：
  - `stdout/stderr`
  - file content
  - content/output/result/text/message

这进一步证明：renderer subtype 不只影响显示，也影响 transcript 搜索语义。

## 当前可稳定列出的 subtype 表

### 顶层 message.type

- `attachment`
- `assistant`
- `user`
- `system`
- `grouped_tool_use`
- `collapsed_read_search`

### user block.type

- `text`
- `image`
- `tool_result`

### assistant block.type

- `tool_use`
- `text`
- `thinking`
- `redacted_thinking`
- `server_tool_use`
- `advisor_tool_result`

### system 已明确的 subtype

- `compact_boundary`
- `microcompact_boundary`
- `local_command`
- `informational`
- `bridge_status`
- `stop_hook_summary`
- `turn_duration`
- `memory_saved`
- `agents_killed`
- `api_error`
- `api_retry`
- `hook_started`
- `hook_progress`
- `hook_response`
- `task_notification`
- `task_started`
- `task_progress`
- `status`
- `init`
- `session_state_changed`
- `elicitation_complete`
- `bridge_state`

## 当前已钉死的结论

- `Aj6` 的“单条消息 renderer”其实是两级分派：先按 message.type，再按 content block.type。
- assistant 与 user 的 block subtype 集合不同，不能复用一套通用 bubble。
- `grouped_tool_use` 与 `collapsed_read_search` 都是高度语义化的聚合消息，不等价于 transcript 原消息。
- transcript 搜索与显示都依赖 subtype-specific 提取逻辑。
- `Hpl(...)` 现在已经能拆到“专用 subtype renderer + 通用字符串 fallback”这一层。
- 普通聊天页与 transcript 视图对 system/info 的可见性不同；`screen === "transcript"` 分支把 `verbose` 直接抬到 `true`。
- 只有带 `toolUseID` 的 `informational` system 行会挂回某个具体 `tool_use`；其余 system 行基本是全局 sideband。
- system subtype 的 producer/schema 集合明显大于最终稳定可见的 transcript 行集合；`notification / thinking_tokens / permission_denied / model_consent_fallback / local_command_output` 等更适合先按 SDK/headless/adapter surface 归档。
- `bridge_status` 是产品态 transcript 行；`bridge_state / session_state_changed` 更像 protocol/runtime sideband event。
- 当前本地 bundle 里，`renderGroupedToolUse(...)` 的实现者当前只看到 agent/subagent 工具族。

## 仍未完全钉死

- 若还有第二个/第三个工具实现 `renderGroupedToolUse(...)`，更可能位于 bundle 外或当前发行版已裁剪掉的分支；本地可见 bundle 内暂未看到第二实现者。
- `tool_result` 大类已经能拆到“正常结果 sidecar / 拒绝态 / 错误态 / tool-specific renderer family”这一层；剩余未补齐的主要是少量低频工具的具体视觉细节。
- protocol-like system subtype 的主边界已基本清楚；若还要继续细分，重点更像是少量低频 producer 的调用表，而不是可见性主干。

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
