# UserPromptSubmit hook 与最终边界

## 本页用途

- 整理 `UserPromptSubmit` hook 的输入、执行位置、输出合并、截断、去重与阻断规则。
- 保留输入编译当前最稳结论、未完全钉死点与证据落点。

## 相关文件

- [04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../../02-execution/01-tools-hooks-and-permissions.md](../../02-execution/01-tools-hooks-and-permissions.md)
- [../../02-execution/05-attachments-and-context-modifiers.md](../../02-execution/05-attachments-and-context-modifiers.md)

## `UserPromptSubmit` hook 的精确位置

### hook 输入

`qe1(...)` 的 hook input 形状已经能写成：

```text
{
  ...hY(permissionMode),
  hook_event_name: "UserPromptSubmit",
  prompt: <plain text prompt>
}
```

其中 `hY(...)` 还会补：

- `session_id`
- `transcript_path`
- `cwd`
- `permission_mode`
- `agent_id?`
- `agent_type`

### hook 只在 `ihz(...)` 之后执行

`AU8(...)` 的顺序已经直接证明：

```text
ihz(...)
  -> shouldQuery?
  -> qe1(...)
```

所以 `UserPromptSubmit` 看见的已经是：

- 经过 slash command / bash / attachment 预处理之后的输入阶段
- 但还没真正进入模型调用

### hook 输出如何并入当前轮

`qe1(...)` 的产物在 `AU8(...)` 里按下面方式消费：

- `blockingError`
  - 直接终止
  - 生成 warning system message
  - 文本里会把 `Original prompt` 一并写出
- `preventContinuation`
  - 直接把一条普通 user message
    - `Operation stopped by hook`
    - 或 `Operation stopped by hook: ${stopReason}`
    塞进当前结果
  - 然后 `shouldQuery = false`
- `additionalContexts`
  - 生成一个 `hook_additional_context` attachment
  - `hookEvent = "UserPromptSubmit"`
  - `toolUseID = hook-${uuid}`
- `message`
  - 直接 push 到 `messages`
  - 若类型是 `hook_success`
    - 其 `content` 会先走 `cU4(...)` 截断

### 截断规则

`cU4(...)` 会把单条 hook 附加文本截到 `10000` 字符。

因此这里当前能写死的结论是：

- `UserPromptSubmit` 可以继续往当前轮注入上下文
- 但这个注入不是无限长原文直通

### matcher 在 `UserPromptSubmit` 上当前不起筛选作用

`et1(...)` 对不同 hook event 会先提一个 `matchQuery`。

但当前 switch 里：

- `PreToolUse / PostToolUse / PostToolUseFailure / PermissionRequest`
  - 用 `tool_name`
- `SessionStart`
  - 用 `source`
- `Setup / PreCompact / PostCompact`
  - 用 `trigger`
- `Notification`
  - 用 `notification_type`
- `InstructionsLoaded`
  - 用 `load_reason`
- `FileChanged`
  - 用 `basename(file_path)`

而 `UserPromptSubmit` 没有专门 case。  
结果是 `matchQuery` 留空，`et1(...)` 会直接采用该事件下的全部 hook matcher，而不是再做 `N8z(...)` 过滤。

因此当前本地实现里：

- `UserPromptSubmit` hook 的 `matcher`
  - **不会参与运行时筛选**
- 只要该事件下注册了 hook
  - 就都会进入候选集

### 多 hook 合并顺序是“完成顺序”，不是“配置顺序”

`FC(...)` 的关键结构是：

```text
matchedHooks = et1(...)
generators = matchedHooks.map(async function* (...) { ... })
for await (result of MC8(generators)) { ... }
```

而 `MC8(...)` 本身是：

- 同时拉起多路 async generator
- `Promise.race(...)`
- 谁先产出就先 `yield`

因此多条 `UserPromptSubmit` hook 并存时：

- 结果流入 `AU8(...)` 的顺序
  - 取决于各 hook 完成先后
- 不是 settings / plugin / session 里的静态声明顺序
- 也不是 `matcher` 列表顺序

### 去重规则只覆盖一部分 hook 类型

`et1(...)` 会对候选集做类型分桶后 dedupe：

- `command`
  - 按 `shell + command` 去重
- `prompt`
  - 按 `prompt` 去重
- `agent`
  - 按 `prompt` 去重
- `http`
  - 按 `url` 去重
- `callback`
  - 不 dedupe
- `function`
  - 不 dedupe

并且 dedupe key 还会拼上：

- `pluginRoot`
- 或 `skillRoot`

所以更精确的结论是：

- 同一 plugin / skill 内的重复 hook 更容易被折叠
- `callback` / `function` hook 则会按原候选数全部保留

### `AU8(...)` 只消费“先到达”的阻断结果

`AU8(...)` 对 `qe1(...)` 的消费是流式的：

- 一旦先看到 `blockingError`
  - 立即返回
- 一旦先看到 `preventContinuation`
  - 立即终止 query 并返回

因此在多 hook 并发时，当前轮真正生效的阻断结果应理解成：

- **先完成并先被 `AU8(...)` 读到的那一个**

这和“最后配置的 hook 覆盖前面”完全不是一回事。

## 当前最稳的还原结论

1. 输入编译不是一个小工具函数，而是一个本地状态机入口。
2. `AU8(...)` 的职责是“编译 + hook 合并”，不是“只做 message 包装”。
3. `ihz(...)` 才是真正的主分流器：
   - 多模态归一化
   - pasted image 归一化
   - attachment 生成
   - bash / slash / prompt 分流
4. 普通 prompt、bash、local slash command、prompt slash command，四条路径的输出形状并不相同。
5. `UserPromptSubmit` 的精确位置已经钉死在 `ihz(...)` 之后、query 之前。
6. `UserPromptSubmit` 多 hook 的当前轮合并顺序是完成顺序，不是配置顺序。
7. `UserPromptSubmit` 的 `matcher` 在当前本地实现里不参与筛选。

## 当前仍未完全钉死

- `Q8(...)` 的完整 user message helper 还有更多边缘字段，但对输入编译主逻辑已经不是阻塞项。
- `sp8(...)` 之外是否还存在 bundle 外或灰度下发的 remote 命令过滤层，当前本地不可见。
- 输入数组里虽然还不能 100% 排除 bundle 外 producer 喂入别的 block，但当前本地活 producer 已明显收敛在 `text` / `image`。

## 证据落点

- `package/preprocessed/cli.extracted.bundle.pretty.js`
  - `BU4(...)`
  - `AU8(...)`
  - `ihz(...)` 与 `m5A(...)`
  - bridge system/init 的 `commands.filter(sp8)`
  - `processSlashCommand(...) / nx_(...) / r74(...)`
  - `MC8(...)`
  - `N8z(...) / V8z(...) / J68(...) / et1(...)`
  - `qe1(...)`
  - `KE6(...)` / `Nq(...)`
  - `sp8(...)` 命中的实名命令对象
- [../02-execution/05-attachments-and-context-modifiers.md](../../02-execution/05-attachments-and-context-modifiers.md)
  - attachment producer/consumer 细节

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
