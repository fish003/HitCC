# Block family、BU4 与 slash command 编译结果

## 本页用途

- 整理本地输入 block family 的当前边界、`BU4(...)` 普通输入消息构造器与 slash command 细分分支。
- 区分本地产出类型、下游兼容 block 与 prompt slash command 的编译结果。

## 相关文件

- [04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../../02-execution/01-tools-hooks-and-permissions.md](../../02-execution/01-tools-hooks-and-permissions.md)
- [../../02-execution/05-attachments-and-context-modifiers.md](../../02-execution/05-attachments-and-context-modifiers.md)

## 输入 block family 的当前边界

### `ihz(...)` 活逻辑只直接特判 `text` / `image`

就输入编译本身，当前本地 bundle 能直接确认的活逻辑只有：

- `image`
  - 会被 `ai(...)` 归一化
- `text`
  - 只有最后一个 `text` block 会被提成主文本 `W`

也就是说，`ihz(...)` 本地真正依赖的规则仍是：

- “最后一个 `text` block = 主 prompt”
- 其它 block 只是前置内容

### 当前本地 producer 也主要只产出 `text` / `image`

目前能直接看到会把 block 数组喂进 `AU8(...)` / `ihz(...)` 的本地活 producer，主要也都落在这两类：

- 用户输入数组本身
  - 当前只看见 `text` / `image`
- pasted contents
  - 只会转成 `image`
- bridge inbound file attachments
  - 会先变成 `@"path"` 文本前缀，而不是 `document` block
- `queued_command`
  - 运行时还原成的也是 `text + image` 组合

### `document` 等 block 当前更像“下游兼容类型”，不是本地输入编译 producer

bundle 下游仍认识更多 content block：

- `document`
- `thinking`
- `tool_use`
- `tool_result`
- `server_tool_use`

但当前本地证据更稳地支持：

- 这些类型主要出现在 transcript / API payload / assistant runtime
- 没有直接看到本地输入侧 producer 把它们作为用户输入 block 喂给 `ihz(...)`

因此此页当前最稳的写法应是：

- **输入编译入口已正证的 block family：`text` / `image`**
- **`document` 等类型属于更大 message/content schema 的兼容面，不应直接写成当前本地输入编译的活 producer**

## `BU4(...)`：普通输入消息构造器

### 固定动作

`BU4(...)` 内部有几件已经能直接确认的动作：

1. 生成新的 promptId
2. 写入全局 promptId cache
3. 取首个 text block 做情绪/continue telemetry
4. 取最后一个 text block 做 `user_prompt` telemetry
5. 调 `Q8(...)` 生成本轮 user message

其中 telemetry 至少包含：

- `tengu_input_prompt`
  - `is_negative`
  - `is_keep_going`
- `user_prompt`
  - `prompt_length`
  - `prompt`
  - `prompt.id`

### 内容块组装规则

#### 有 pasted image 时

若 `q.length > 0`，`BU4(...)` 会把当前轮 user message 组装成：

```text
content = [
  ...当前文本或 block 数组,
  ...pasted image blocks
]
```

并额外设置：

- `imagePasteIds`
- `permissionMode`
- `isMeta`

注意这里的顺序是：

- 文本/已有 block 在前
- pasted image block 在后

#### 没有 pasted image 时

直接把 `input` 原样作为 `content` 传入 `Q8(...)`。

### 返回值结构

`BU4(...)` 返回的不是只有当前轮 user message，而是：

```text
{
  messages: [
    currentTurnUserMessage,
    ...attachmentMessages
  ],
  shouldQuery: true
}
```

这说明：

- 当前轮主 user message 与 attachment transcript item 仍是两类对象
- 它们在更后面的 prompt assembly 阶段才会进一步线性化/合并

## slash command 细分分支

### `processSlashCommand(...)` 的一级分流

#### 解析失败

若 `JC8(...)` 失败：

- 返回“Commands are in the form `/command [args]`”
- `shouldQuery = false`

#### 命令名不存在，但长得像 skill 名

若命令不存在，且 `i74(name)` 认为它是合法 slash token，且它又不是一个真实文件路径：

- 返回 `Unknown skill: ${name}`
- 如果有参数，还会额外给一条 warning system message
- `shouldQuery = false`

#### 命令名不存在，但不像合法 slash token

这种情况不会直接报错，而是退化成普通 user prompt：

- 直接构造一条 user message
- `shouldQuery = true`

因此并不是所有未知 `/xxx` 都当本地命令错误处理。

### `nx_(...)` 的二级分流

命令存在后，`nx_(...)` 会再按 command type 分流：

- `local-jsx`
  - 打开本地 UI 组件
  - 可选择直接 `skip`
- `local`
  - 只做本地命令执行
  - 返回 stdout/stderr
  - `shouldQuery = false`
- `prompt`
  - 若 `context === "fork"`：走 `lx_(...)`
  - 否则走 `r74(...)`

### prompt slash command 的编译结果

`r74(...)` 这条路径已经可以写成：

```text
messages = [
  1. 一个 user message，内容是 rx_(command, args)
  2. 一个 user meta message，内容是 command prompt block 数组
  3. prompt 文本触发出来的 attachment transcript items
  4. 一个 command_permissions attachment
]

shouldQuery = true
allowedTools = parsed command allowedTools
model = command.model
effort = command.effort
```

这里最重要的不是“slash command 也会 query”，而是：

- 它 query 的并不是原始 `/cmd args`
- 而是本地把 command prompt 展开后的 block 数组

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
