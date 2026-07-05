# `contextLayers` / `contextModifier` 与执行器 Consumer

## 本页用途

- 单独梳理执行后运行态改写的接口、应用时机、并发提交规则，以及当前本地 bundle 的 concrete producer 边界。
- `contextModifier` 是本组文档早期使用的概念名；在当前 `2.1.197` bundle 可见实现里，工具返回侧实际字段是 `contextLayers`，执行上下文里承载这些 overlay 的字段是 `permissionLayers`。

## 运行时接口

当前本地实现不是返回任意 `modifyContext(ctx)` 函数，而是返回一组可枚举的 context layer：

```ts
type ToolContextLayer =
  | { kind: "allowed_tools"; allowedTools: string[] }
  | { kind: "disallowed_tools"; disallowedTools: string[] }
  | { kind: "model"; mainLoopModel: string }
  | { kind: "effort"; effort: string }
  | { kind: "max_thinking_tokens"; maxThinkingTokens: number }
  | { kind: "permission_mode"; mode: PermissionMode }
  | { kind: "working_directory"; directory: string }
  | { kind: "flag_settings"; settings: unknown }

interface ToolContextLayerEnvelope {
  toolUseID: string
  layers: ToolContextLayer[]
}
```

工具返回后，`oTf(...)` 会把 `result.contextLayers` 包成 `{ toolUseID, layers }` 随同 tool result 往上送；`NYt(ctx, layers)` 负责把 layer 追加到 `permissionLayers`，并对 `model / max_thinking_tokens` 同步更新 `ctx.options`。

## consumer：串行路径

串行执行时，context layer 是即时应用的：

```text
iMo(...) / aTf(...)
  -> for each tool result:
       if contextLayers:
         ctx = NYt(ctx, layers)
```

也就是：

- 当前工具结束后
- 下一工具开始前
- 立刻更新 `ToolUseContext`

## consumer：并发路径

并发安全块里不会“谁先完成谁先提交”。

实际语义是：

1. 先按 `toolUseID` 收集 `contextLayers`
2. 等并发块全部完成
3. 再按原始 tool block 顺序依次应用到上下文

因此并发安全块的语义是“块结束后按声明顺序提交”，不是“按完成时间提交”。

## consumer：执行器实例内的非并发工具

流式工具 runner 内部还维护每个 tool 的 `contextLayers` 数组：

- 工具完成后先把 layer 暂存进该 tool 的状态
- 若该工具不是 concurrency-safe，则立即用 `NYt(...)` 更新 runner 内的 `toolUseContext`
- `getCompletedResults()` / `getRemainingResults()` 再按 runner 的可见上下文向外 yield

这和外层批执行路径的规则一致，只是粒度更靠近 executor 内部。

## 当前本地 bundle 内已确认的 concrete producer

### `SkillTool`

`SkillTool` inline 路径返回结果时，会按 skill / slash command 展开结果构造 `contextLayers`。

当前已正证它至少能影响三类状态：

1. `allowed_tools`：后续 permission 计算时表现为 command allow overlay
2. `model`：通过 `NYt(...)` 同步改写 `options.mainLoopModel`
3. `effort`：通过 `permissionLayers` 影响后续 `bg(ctx)` 取值

因此它不是“只附加一段说明文字”的 skill helper，而是真的会改后续运行态。

### `EnterWorktree`

`EnterWorktree` 在 mid-session / pinned working directory 相关路径里可以返回：

```ts
contextLayers: [
  { kind: "working_directory", directory: worktreePath }
]
```

这说明当前本地 concrete producer 不止 `SkillTool`；`SkillTool` 是权限 / model / effort 类 producer，`EnterWorktree` 是 working directory overlay producer。

### fork skill / slash command fork

fork skill 与 fork slash command 会先经 `Dzt(...)` 生成 `allowed_tools / disallowed_tools` layer，再把这些 layer 合入传给 `_3(...)` 的子 agent `permissionLayers`。这条路径不是“父线程 tool result 返回后再提交”，而是 fork 子执行前的上下文预置。

## 本地 bundle 边界

重新扫完后，当前能收紧成：

- 当前 bundle 中未出现字面量 `contextModifier`；本地活实现应以 `contextLayers / permissionLayers / NYt(...)` 为准。
- tool-returned concrete producer 当前正证到：
  - `SkillTool`：`allowed_tools / model / effort`
  - `EnterWorktree`：`working_directory`
- 其它命中点主要是 generic consumer、subagent/fork 预置 layer，或 SDK / bridge 入口上的上下文 overlay。
- 没有看到 tool-returned context layer 直接去修改：
  - `readFileState`
  - `contentReplacementState`
  - `additionalDirectoriesForClaudeMd`
  - `active agent definitions`

因此当前更稳的结论不是“所有上下文修改都是工具返回协议”，而是：

- **工具返回协议负责少量可枚举 layer**
- **更广的上下文状态仍由专门路径直接改写或通过 clone / overlay 注入**

## remote / bridge 的反证

remote / bridge 确实会影响本地上下文状态，但当前可见路径采用的是直接写缓存或入口 overlay，而不是复用同一套 tool-returned `contextLayers` 协议。

已看到的代表性反证是：

- `seed_read_state`
  - 直接把 read state 写入缓存
  - 不是先返回一个 context layer 再由执行器应用

因此本地当前更稳的判断是：

- “远端能改本地上下文”成立
- “远端通过同一套 tool-returned `contextLayers` 协议改上下文”当前没有直接证据

## 与 `ToolUseContext` 的关系

`contextLayers` 的真正价值，不是 UI 展示，而是把工具结果转成下一步运行态。

更准确地说：

```text
tool.call(...)
  -> may return contextLayers
  -> executor applies NYt(ctx, layers)
  -> next tool / next side-path sees updated ToolUseContext
```

因此它和 [06-context-runtime-and-tool-use-context.md](../06-context-runtime-and-tool-use-context.md) 是一组上下游关系：

- `ToolUseContext` 是被改写的对象
- `contextLayers` 是当前 bundle 可见的工具返回改写协议
- `permissionLayers` 是执行上下文里承载 overlay 的运行态字段

## 当前仍未完全钉死

- 远端/服务端侧是否还有 bundle 外 concrete producer，当前不能 100% 排除。
- 少量灰度/已裁剪工具若历史上支持更泛化的 `contextModifier` 函数式协议，当前本地 bundle 无法正证。

## 证据落点

- `package/preprocessed/cli.extracted.bundle.pretty.js`
  - `oTf(...)`：把 tool result 中的 `contextLayers` 包成 `{ toolUseID, layers }`
  - `iMo(...) / aTf(...) / lTf(...)`：串行与并发块提交规则
  - streaming tool runner：非 concurrency-safe 工具完成后即时推进 `toolUseContext`
  - `NYt(...) / Nr(...) / bg(...) / vq(...)`：layer 应用与读取
  - `SkillTool` concrete producer
  - `EnterWorktree` concrete producer
  - `_3(...) / R7n(...)`：fork/subagent 的 `permissionLayers` 预置与克隆
  - `seed_read_state` 反证
- [06-context-runtime-and-tool-use-context.md](../06-context-runtime-and-tool-use-context.md)

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
