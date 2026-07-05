# 工具执行内核与 `tool_result` 回写

## 本页用途

- 单独整理工具从 `tool_use` 进入执行器、经过 permission/hook、再回写 transcript 的主链。
- 把 `ToolSearch`、并发执行器、`tool_result` 修复与压缩从权限专题里拆开，避免一页继续混写。

## 相关文件

- [../01-tools-hooks-and-permissions.md](../01-tools-hooks-and-permissions.md)
- [../../01-runtime/04-agent-loop-and-compaction.md](../../01-runtime/04-agent-loop-and-compaction.md)
- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../05-attachments-and-context-modifiers.md](../05-attachments-and-context-modifiers.md)
- [../06-context-runtime-and-tool-use-context.md](../06-context-runtime-and-tool-use-context.md)

## 工具执行内核：`$Yt` / `nTf` / `oTf` / `KHe`

工具执行器是另一条已经非常清楚的主线。

### 调用链

```text
tool_use block
  -> KHe.addTool(...)
    -> KHe.executeTool(...)
      -> $Yt(...)
        -> nTf(...)
          -> oTf(...)
        -> PreToolUse hooks
        -> permission merge
        -> tool.call(...)
        -> PostToolUse hooks
        -> tool_result / attachments / contextLayers
```

旧文档里的 `he6 / Ho_ / Mo_ / Re6 / Zx8` 是旧版压缩名。当前 `2.1.197` 的可见主链应以 `$Yt(...) -> nTf(...) -> oTf(...)` 和 `KHe` runner 为准。

### `$Yt(...)`

职责：

- 根据 tool name / alias 找到工具定义
- 获取 MCP 元信息
- 若工具不存在，立即返回 `tool_result` 错误
- 处理 abort、tool isolation latch、MCP server still connecting 这类进入工具前的错误面

### `nTf(...)` 与 `oTf(...)` 输入校验

顺序大致为：

1. `coerceInput` 与 JSON parse repair 判定
2. `inputSchema.safeParse`
3. `validateInput?.(...)`
4. 特殊工具输入清洗，例：Bash 相关内部字段
5. `backfillObservableInput`

若 schema 失败且使用 deferred tool search，还会提示“先 select 对应 tool 再重试”，说明 tool schema 可能按需发送给模型，而不是永远全量注入。

### deferred tools / `ToolSearch`

这一层现在也可以从“可能存在”收紧到“运行时机制明确存在”。

#### 核心结构

- 工具是否进入 deferred 集合由 `RD(...)` 决定
- `isMcp===true` 的工具会被当成 deferred 候选
- 一部分内建工具也可能因 flag 或 `shouldDefer===true` 进入 deferred 集合
- deferred tool 初始只暴露名字，不暴露完整参数 schema

#### `ToolSearch` 的职责

`ToolSearch` 不是普通 grep/search 工具，而是“把 deferred tool 的完整 schema 拉进当前上下文”的桥接器。

其 prompt 已直接说明：

```text
Until fetched, only the name is known — there is no parameter schema, so the tool cannot be invoked.
```

因此更准确的调用链应写成：

```text
模型看到 deferred tool 名字
-> 先调用 ToolSearch(query)
-> tool_result 中写入 tool_reference(tool_name)
-> 下一次 request build 从历史消息里提取 discovered tool names
-> 只把这些命中的 deferred tool 放回 tools 数组
-> 通过 Du8(...) 序列化为完整 description + input_schema
-> 下一轮才能真正调用该 tool
```

#### 查询形态

当前已确认支持：

- `select:Read,Edit,Grep`
- 普通关键词搜索
- `+slack send` 这类“必须命中某个词”的查询

#### 结果形态

当前运行时 `mapToolResultToToolResultBlockParam(...)` 不是纯文本列表，而是：

- 无匹配时：普通文本 `tool_result`
- 有匹配时：`tool_reference[]`

这说明 `ToolSearch` 返回值不是“让模型自己读一段搜索结果文本”，而更像是工具注册表层面的引用注入。

#### 真实注入链不是 `<functions>` 文本拼接

这里要特别纠正一个容易误判的点：

- `ToolSearch` prompt 文案里提到 `<functions>` / `<function>...`
- 但当前运行时代码里，真正可见的 transcript 结果是 `tool_reference[]`

本地 bundle 里现在能直接钉死的真实链路是：

1. `ToolSearch` 返回 `tool_reference(tool_name)`
2. `BF(...)` 从历史 `tool_result.content[]` 中提取这些 `tool_name`
3. query builder 在开启 tool search 时，只保留：
   - 非 deferred tools
   - `ToolSearch` 本身
   - 已被 `BF(...)` 发现过的 deferred tools
4. 这些被选中的 deferred tools 再经过 `Du8(...)` 序列化成真正的：
   - `name`
   - `description`
   - `input_schema`
5. 最终它们是通过 request body 里的 `tools` 数组重新进入模型调用，而不是靠普通文本消息“临时变可调用”

因此更稳妥的结论不是“`tool_reference` 直接等于 schema”，而是：

- `tool_reference` 是一个发现信号
- 下一轮 request builder 才会把对应 tool 的完整 schema 真正注入

#### 与 prompt layering 的关系

compact / prompt 装配链里已经能看到：

- `deferred_tools_delta`

说明 deferred tool 列表本身是一个独立 attachment/source，而不是临时拼在普通 system prompt 文本里的注释。

### `PreToolUse` hooks

在 permission 之前执行。可做：

- 追加 message/additional context
- 修改输入
- allow/ask/deny/block
- preventContinuation
- stopReason
- 直接 stop

### permission 合并

实际顺序不是单一 `canUseTool()`，而是：

```text
PreToolUse hook
-> 静态 allow/deny 规则
-> canUseTool(UI/SDK/transport)
```

并且 denied/ask 的内容会进入 transcript/tool_result，而不只是 UI 弹窗。

### tool.call 结果

工具返回不是简单字符串。至少可能携带：

- `message` / `messages`
- `StructuredOutput`
- `contextLayers`
- `mcpMeta`
- `stopContinuation`

更具体地说，`oTf(...)` 在 `tool.call(...)` 成功后至少还会做四件事：

1. 调 `mapToolResultToToolResultBlockParam(...)` 生成 transcript 用的 `tool_result`
2. 经过统一的结果尺寸裁剪/落盘包装
3. 执行 `PostToolUse` hooks，允许追加 context 或替换 MCP 输出
4. 把 `messages / attachments / contextLayers / mcpMeta` 一并回交给上层执行器

因此“tool 的结构化输出”和“最终进入 transcript 的 `tool_result`”是两层概念，不应混为一谈。

### `tool_result` 的两层形态

这一点现在可以明确：

#### 1. tool 自己的结构化输出

例如：

- `BashOutput`
- `WebSearchOutput`
- `WebFetchOutput`
- `AskUserQuestionOutput`

这是工具内部/SDK 层的稳定契约。

#### 2. transcript 里的 `tool_result`

真正回灌给模型的是 `mapToolResultToToolResultBlockParam(...)` 产物。

因此不同工具会出现明显差异：

- `WebSearchTool`：结构化结果被降成带 `Links:` 和引用提醒的文本
- `WebFetch`：只回填 `result` 字符串
- `Bash`：会把 `stdout/stderr/backgroundTaskId/persistedOutputPath` 等折叠成文本或 structured content
- `AskUserQuestion`：会把答案拼成 `User has answered your questions: ...`

### `tool_result` 完整性与还原

这部分现在也不只是协议常识，而是本地实现里有专门修复逻辑。

#### 严格配对要求

模型/SDK 层会校验：

- assistant 里出现的 `tool_use.id`
- 下一条 user 里的 `tool_result.tool_use_id`

必须一一对应。

否则会报：

- `tool_use ids were found without tool_result blocks immediately after`
- `unexpected tool_use_id found in tool_result`

#### 本地修复器

`ensureToolResultPairing` 相关逻辑会在 transcript/resume 阶段尝试修复：

- orphaned `tool_result`
- 重复 `tool_result`
- assistant 中缺失配对 result 的 `tool_use`
- `server_tool_use / mcp_tool_use` 与 result 对不上

缺失时会补一个 synthetic error：

```text
[Tool result missing due to internal error]
```

因此 transcript 持久化层并不是“原样存”，而是会做结构修复。

### 超长 `tool_result` 的统一落盘

`mapToolResultToToolResultBlockParam(...)` 之后还会走统一包装：

```text
tool result block
-> H2q(...)
-> 若超阈值则写入 tool-results 目录
-> transcript 内只保留 preview + persisted path
```

当前已确认：

- 空结果会被改写成 `(<tool> completed with no output)`
- 图片类结果不会走文本落盘
- 超长文本结果会写入持久化文件，再把 preview 回填进 transcript
- message budget 阶段还会二次把旧 `tool_result` 替换成 persisted preview

这说明 CLI 对 tool 输出至少有两层压缩：

1. 单次工具结果大小限制
2. 整条消息预算限制

### `PostToolUse` / `PostToolUseFailure` / `PostToolBatch`

调用后还能：

- 添加 attachment
- 添加 message
- 停止 continuation
- 更新 tool output 呈现
- 在同一批工具全部结束后执行 `PostToolBatch`

### `KHe`：当前流式工具 runner

`KHe` 是当前主循环内的长生命周期工具 runner。它不是只负责“批量执行”，而是同时处理排队、并发安全、progress 暂存、streaming fallback 丢弃、interrupt 语义和上下文更新。

### 每个 tool 的状态

- `queued`
- `executing`
- `completed`
- `yielded`

### 关键机制

- 只要当前 executing 的工具都 `isConcurrencySafe=true`，safe 工具可以并行继续进
- unsafe 工具会形成串行边界
- progress 先缓存到 `pendingProgress`
- 若某个 sibling 报错，其他并发 tool 可收到 synthetic cancel error
- streaming fallback 时会丢弃当前 streaming 执行器结果

#### synthetic cancel 的具体来源

当前已确认至少有三类取消原因：

- `user_interrupted`
- `streaming_fallback`
- `sibling_error`

对应会生成不同的 synthetic `tool_result` 错误文本，而不是简单把 sibling 静默丢掉。

#### `pendingProgress`

`KHe` 并不是直接把 progress 写进 transcript。

它会先把 progress message 放进：

- `pendingProgress`

然后再由外层通过 `getCompletedResults()` / `getRemainingResults()` 按时机 `yield` 给 UI / transcript adapter。

这说明流式工具执行器内部本身还承担了一层“进度缓冲器”角色。

### 批次排队与普通工具轮

职责：

- 将 tool_use 依据并发安全性分块
- safe 工具可在当前 executing 工具都安全时继续并发
- unsafe 工具会等待前面的 executing 工具结束，并形成串行边界
- 控制 `contextLayers` 的提交时机

### contextLayers 提交规则

- concurrency-safe 工具产出的 context layers 先按工具 ID 暂存，等安全并发块结束后再按原始 tool block 顺序提交
- 非 concurrency-safe 工具完成且带有 context layers 时，`KHe` 会立刻把 `toolUseContext = NYt(toolUseContext, layers)` 推进

### 设计意义

避免并发执行阶段在共享上下文上产生难以预测的中间状态。

### `AskUserQuestion` 不是普通文本追问

这部分对 tool 调用细节很重要，因为它直接暴露了“需要用户交互的工具”在 permission 层的特例。

#### 工具契约

当前可直接确认：

- 工具名：`AskUserQuestion`
- 输入：`questions[1..4] + answers? + annotations? + metadata?`
- 每个 question 至少有：
  - `question`
  - `header`
  - `options[2..4]`
  - `multiSelect?`
- option 还支持：
  - `label`
  - `description`
  - `Preview?`

输出为：

- `questions`
- `answers`
- `annotations?`

#### 运行时语义

这个工具：

- `isReadOnly() -> true`
- `isConcurrencySafe() -> true`
- `requiresUserInteraction() -> true`
- `checkPermissions(...) -> ask`

也就是说它虽然不修改文件，但它不是“静默可过”的 read-only tool，而是强制进入用户确认/回答路径。

#### transcript 回填

`mapToolResultToToolResultBlockParam(...)` 不会回填原始 JSON，而是拼成：

```text
User has answered your questions: "question"="answer"
```

若有 preview/notes，也会附加进文本。

因此从模型视角看，它消费到的是“用户已回答”的自然语言事实，而不是 UI 表单原始结构。

## platform execution：shell、Perforce 与内嵌搜索 helper

工具执行层还带一组平台兼容合同，不能只按 POSIX Bash 还原。

### shell selection 与 Windows 分支

当前 bundle 明确区分 Bash 与 PowerShell：

- `CLAUDE_CODE_USE_POWERSHELL_TOOL` 是可见 env gate
- Windows 如果找不到 Git Bash 且 PowerShell tool 被禁用，会直接报错，要求启用 shell tool 或移除禁用设置
- plugin command schema 的 shell 描述写明：
  - `bash` 使用 `$SHELL` / bash / zsh / sh
  - `powershell` 使用 `pwsh`
  - 默认是 bash；Windows 且没有 Git Bash 时默认 PowerShell
- TUI permission dispatch 里 Bash / PowerShell 是两条独立审批链，PowerShell 不是 Bash 文案替换

因此重写 shell 工具时至少要保留三层：

```text
platform detection
  -> Git Bash / PowerShell availability
  -> BashTool / PowerShellTool permission and render branch
```

### Perforce mode

`CLAUDE_CODE_PERFORCE_MODE` 是当前可见 env 开关。开启后，写文件路径会把只读文件解释为 Perforce checkout 边界：

- 只读文件会提示先 `p4 edit <file>`
- 提示明确不要用 `chmod` 绕过 Perforce tracking
- system prompt 也会补充 Perforce workspace 说明：未 open for edit 的文件只读，已 writable 的文件可直接改

这不是普通文件系统错误文案，而是 source-control-aware write contract。恢复写入工具时要把它作为平台模式处理。

### embedded `bfs` / `ugrep`

当前 Bash 初始化脚本里会 shadow `find` / `grep`：

- `find` 映射到 embedded `bfs`，带 `-S dfs` 和 `-regextype findutils-default`
- `grep` 映射到 embedded `ugrep`，带 `--ignore-files`、`--hidden`、`-I` 和默认 exclude-dir
- 这条 shadow 只在 ant-native / embedded search tools 可用时接入

它的作用不是改变模型工具名，而是改善 shell 中传统 `find` / `grep` 的行为一致性。文档中仍应把 `Grep/Search/Glob/ToolSearch` 与 shell 内 shadow helper 区分开。

### background shell 与 orphan handling

前台长任务可以被 background：

- `Ctrl+B` / control-request 可把 in-flight Bash/subagent 背景化
- tool result 会先返回“running in the background”，turn 继续推进
- task 后续通过 `task_notification` 回到消息流
- 如果上一轮 Claude Code 进程退出时 background agent 未完成，恢复时会提示 in-process state lost，要求检查 worktree/output，不能假设任务已经落地

这说明 background shell 不是“stdout 暂存”的小功能，而是工具生命周期的一部分：前台阻塞、背景化、完成通知、异常退出恢复都需要建模。

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
