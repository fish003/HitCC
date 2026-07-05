# AU8 / ihz 预处理与主分流

## 本页用途

- 整理 `AU8(...)` 输入编译入口、`ihz(...)` 预处理器、bash/slash/prompt 三条主分流。
- 说明 remote-control slash gate、pasted image 与 attachment loading 在输入编译前段的位置。

## 相关文件

- [04-agent-loop-and-compaction.md](../04-agent-loop-and-compaction.md)
- [../../02-execution/01-tools-hooks-and-permissions.md](../../02-execution/01-tools-hooks-and-permissions.md)
- [../../02-execution/05-attachments-and-context-modifiers.md](../../02-execution/05-attachments-and-context-modifiers.md)

## 总览

输入不是“原样塞进模型”，而是先经过一层本地编译：

```text
AU8(...)
  -> ihz(...)
     -> 多模态 block 归一化
     -> pasted image 归一化
     -> remote slash gate
     -> attachment producer 生成 attachment transcript entries
     -> bash / slash command / normal prompt 分流
     -> BU4(...)
     -> m5A(...) 追加图片尺寸类 meta message
  -> qe1(...) UserPromptSubmit
  -> 可能再注入 hook_additional_context / hook_success / blocking stop
  -> 返回给 CC(...)
```

这条链的直接结论：

- slash command 是本地解析和本地分流，不是交给模型猜。
- attachment 是本地 producer 生成的 transcript item，不是模型自己“发现”。
- 多模态输入的最终 `user.message.content` 不是纯字符串，而是 block 数组。
- `UserPromptSubmit` 不在输入最前面，而是在 `ihz(...)` 完成之后。

## `AU8(...)`：输入编译入口

### 入参形状

`AU8(...)` 直接接收当前轮最上层提交参数：

- `input`
- `preExpansionInput`
- `mode`
- `setToolJSX`
- `context`
- `pastedContents`
- `ideSelection`
- `messages`
- `setUserInputOnProcessing`
- `uuid`
- `isAlreadyProcessing`
- `querySource`
- `canUseTool`
- `skipSlashCommands`
- `bridgeOrigin`
- `isMeta`
- `skipAttachments`

其中最关键的几项是：

- `input`
  - 可能是字符串
  - 也可能是 block 数组
- `mode`
  - 决定是 `prompt`、`bash` 还是别的路径
- `pastedContents`
  - pasted image 单独走一条归一化链
- `skipSlashCommands`
  - 主要用于 remote-control 下的 slash command 限制
- `messages`
  - 传给 attachment producer，用于做 delta/reminder 类 attachment 判定

### 返回形状

当前更稳的写法不应只保留最小字段，而应写成“编译输出超集”：

```ts
interface CompiledInput {
  messages: TranscriptLikeMessage[]
  shouldQuery: boolean
  resultText?: string
  allowedTools?: string[]
  model?: string
  effort?: string
  nextInput?: string | ContentBlock[]
  submitNextInput?: boolean
}
```

原因是：

- 普通输入通常只返回 `messages + shouldQuery`
- slash command 路径还可能回填：
  - `allowedTools`
  - `model`
  - `effort`
  - `nextInput`
  - `submitNextInput`

### 顶层顺序

`AU8(...)` 的本地顺序已经可以直接写死：

```text
1. 若 mode=prompt 且 input 是普通字符串且不是 meta，先 setUserInputOnProcessing
2. 调 ihz(...)
3. 若 ihz 返回 shouldQuery=false，立即结束
4. 否则取文本 prompt（pU(A) || ""）
5. 执行 qe1(...): UserPromptSubmit hooks
6. 消费 hook 输出：
   - blockingError -> 直接停止
   - preventContinuation -> 写入停止消息并终止 query
   - additionalContexts -> 生成 hook_additional_context attachment
   - message -> 直接并入 messages
7. 返回给主循环
```

## `ihz(...)`：输入预处理器

`ihz(...)` 负责的不是单一“附件处理”，而是整段输入编译的主分流器。

### 第 1 段：把 `input` 归一成“前置 block + 主文本”

`ihz(...)` 先把输入拆成：

- `W`
  - 最终主文本字符串
- `G`
  - 主文本之前的原始 block
- `v`
  - 归一化后的完整输入
- `Z`
  - 额外的图片尺寸/来源说明文本

规则已经能直接确认：

1. 若 `input` 是字符串：
   - `W = input`
   - `G = []`
2. 若 `input` 是 block 数组：
   - 逐个遍历
   - `image` block 先走 `ai(...)` 做本地图像归一化
   - 成功拿到尺寸后，用 `Qv6(...)` 生成一条文本说明，塞进 `Z`
   - 图像 block 自身被替换成归一化后的 `u.block`
3. 最后一个 block 若是 `text`
   - 它会被视为主文本 `W`
   - 前面的 block 进入 `G`
4. 最后一个 block 若不是 `text`
   - `W = null`
   - 整个数组都留在 `G`

因此当前轮输入的真正语义是：

- “最后一个 text block” 更像主 prompt
- 它前面的 block 更像当前轮附带的多模态前置内容

### 第 2 段：pasted image 走独立归一化链

`pastedContents` 不和 `input` 里的 `image` block 复用同一套容器，而是单独处理：

1. 先从 `pastedContents` 里筛出图片项
2. 收集它们的 `id`，形成 `imagePasteIds`
3. 用 `mSq(...)` 取 source-path map
4. 每张 pasted image 都转成：
   - 一个真正的 `image` content block
   - 一条可选的尺寸/来源说明文本，继续塞进 `Z`
5. 最终这些 pasted image block 汇总到 `C`

这意味着 bundle 里已经明确区分了两类图像来源：

- 输入数组内直接携带的 `image` block
- 编辑器/剪贴板粘贴进来的 image

两者最后都会进入同一轮 `user` content，但 bookkeeping 不同：

- pasted image 会额外记录 `imagePasteIds`

### 第 3 段：remote-control 下的 slash command gate

`skipSlashCommands` 为真，且主文本 `W` 以 `/` 开头时，`ihz(...)` 会先做一次本地 gate：

1. 用 `JC8(W)` 解析 slash command
2. 用 `UU(commandName, commands)` 找命令定义
3. 若命令存在：
   - `sp8(command)` 为真：把 `x = false`
     - 这不是直接报错
     - 而是表示该 slash command 不能按普通本地 slash path 执行
   - `sp8(command)` 为假：
     - 直接短路返回
     - 结果是：
       - 一条当前输入的 user message
       - 一条 `<local-command-stdout>/${command} isn't available over Remote Control.</local-command-stdout>`
       - `shouldQuery = false`

因此这里不是“所有 remote slash command 都禁用”，而是：

- 一部分命令被本地彻底拦截
- 另一部分命令只是改变后续是否走 slash path

### `sp8(...)` 的当前实名命令族

`sp8(...)` 现在已经可以直接写成：

```ts
function sp8(command) {
  if (command.type === "local-jsx") return false
  if (command.type === "prompt") return true
  return AWz.has(command)
}
```

当前本地 bundle 里，`AWz` 命中的实名本地命令是：

- `compact`
- `clear`
- `cost`
- `release-notes`
- `files`
- `stub`
  - 隐藏且未启用

因此 remote-control 下的 slash gate 现在应收成：

- 所有 `prompt` command
  - 允许继续进入本地 slash 编译链
- 少量指定 `local` command
  - 也允许继续进入本地 slash 编译链
- 其余 `local` / `local-jsx` command
  - 直接被拦成 `isn't available over Remote Control`

这里还有一个产品层旁证：

- bridge system/init 会把 `commands:M.current.filter(sp8)` 发给远端

这说明 `sp8(...)` 不只是一个本地 if/else gate，它还是 remote-control 可见 slash universe 的筛选器。

### 第 4 段：attachment loading gate

attachment 不是永远加载，实际 gate 是：

```text
I = !isMeta
    && W !== null
    && (
         mode !== "prompt"
         || x
         || !W.startsWith("/")
       )
```

然后才会执行：

```text
PC8(KE6(W, toolUseContext, ideSelection, [], messages, querySource))
```

这几条非常关键：

- meta 输入不做 attachment loading
- 没有主文本时不做 attachment loading
- `prompt` 模式下，如果它是本地 slash command 且 `x === false`，也不会走普通 attachment loading

换句话说，普通用户 prompt 与 prompt slash command 的 attachment 生成时机并不相同。

## `ihz(...)` 的三条主分流

### 1. `mode === "bash"`

当 `W !== null && mode === "bash"` 时：

- 调 `processBashCommand(W, G, p, toolUseContext, setToolJSX)`
- 直接返回
- 不进入普通 `BU4(...)`

这里 `G` 会作为 `precedingInputBlocks` 带入 bash 输入包装，所以 bash 模式也支持前置多模态 block。

### 2. 本地 slash command

当 `W !== null && !x && W.startsWith("/")` 时：

- 调 `processSlashCommand(W, G, C, p, toolUseContext, setToolJSX, uuid, isAlreadyProcessing, canUseTool)`
- 返回 slash command 自己编译出的结果

这里四组输入各自职责很清楚：

- `G`
  - 原始输入数组里主文本前的 block
- `C`
  - pasted image block
- `p`
  - attachment transcript entries
- `W`
  - slash command 原始文本

### 3. 普通 prompt

其它情况最终都进入：

```text
m5A(BU4(v, C, imagePasteIds, p, uuid, permissionMode, isMeta), Z)
```

也就是：

1. 先由 `BU4(...)` 生成当前轮 user message
2. 再由 `m5A(...)` 把 `Z` 里的图片尺寸/来源说明包成额外一条 meta user message 追加进去

因此图片尺寸类说明不是塞进同一条 user content block，而是追加成后置 meta message。

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
