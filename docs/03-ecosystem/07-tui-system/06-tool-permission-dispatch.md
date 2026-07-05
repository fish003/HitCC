# Tool Permission 分派树

## 本页用途

- 把 tool permission overlay 从“总分发器”继续拆成可还原的工具类型到审批 UI 映射。
- 固定哪些审批 UI 是专用组件，哪些走通用 wrapper。

## 相关文件

- [../07-tui-system.md](../07-tui-system.md)
- [04-dialogs-and-approvals.md](./04-dialogs-and-approvals.md)
- [../../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md](../../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md)

## 总分发器

`tool-permission` overlay 的核心仍是“通用 permission request shell + tool-specific dialog/descriptor registry”。

它的固定步骤已经很清楚：

1. 生成顶部审批文案。
2. 注册 `app:interrupt`，中断时会 `onReject + onDone + queue reject`。
3. 根据 `toolUseConfirm.tool` 选择具体审批组件或 generic fallback。
4. 透传：
   - `toolUseConfirm`
   - `toolUseContext`
   - `onDone`
   - `onReject`
   - `workerBadge`
   - `setStickyFooter`

因此总分发器不拥有审批细节，真正的语义在 tool-specific descriptor / dialog 分支。

## tool-specific 分派表

### 1. `eD -> pu4(...)`

这是单文件字符串替换编辑审批。

已确认：

- 解析 `old_string / new_string / replace_all`
- 标题是 `Edit file`
- 用 `eg8(...)` 预览 edit patch
- 走 `IQ` 这一类可 diff 的文件编辑审批壳

也就是典型：

```text
str_replace / edit
```

### 2. `Af -> Tm4(...)`

这是写文件或覆盖文件审批。

已确认：

- 解析 `file_path / content`
- 文件存在时标题是 `Overwrite file`
- 不存在时标题是 `Create file`
- 用 `Gm4(...)` 展示完整文件内容或 patch 预览
- 同样走 `IQ` 壳

### 3. `yq -> fm4(...)`

这是 Bash 命令审批。

已确认：

- 普通路径走 `xNz(...)`
- 若 `YE6(command)` 命中 sed/编辑型模式，则转 `du4(...)` 的专用分支
- 支持 destructive warning
- 支持把命令前缀保存成 allow rule
- 标题会区分：
  - `Bash command`
  - `Bash command (unsandboxed)`

这说明 Bash 审批不是单一确认框，而是：

- 普通命令 UI
- 编辑型 shell 命令 UI
- rule suggestion / feedback mode

三层叠加。

### 4. `eU -> KB4(...)`

这是 PowerShell 命令审批。

它与 Bash 很像，但有独立组件：

- 标题固定 `PowerShell command`
- 用 `tm4(...)` 生成选项
- 用 `am4(...)` 推断可保存的 PowerShell 命令前缀
- 有自己的 destructive command warning 规则集

因此 PowerShell 不是 Bash 文案替换，而是独立分支。

### 5. `_f -> Em4(...)`

这是 fetch 类 URL 访问审批。

当前已确认：

- 从输入里取 `url`
- 提取 hostname
- 标题是 `Fetch`
- 选项包括：
  - `Yes`
  - `Yes, and don't ask again for <domain>`
  - `No`

其“don't ask again”写入的是：

- `toolName + domain:<hostname>`

这说明它是面向 URL/domain 级规则的专用审批 UI。

### 6. `Ko -> Rm4(...)`

这是 notebook 编辑审批。

已确认：

- 解析 `notebook_path / cell_id / new_source / cell_type / edit_mode`
- 标题固定 `Edit notebook`
- 预览组件是 `Lm4(...)`
- `edit_mode` 至少包括：
  - `insert`
  - `delete`
  - `replace`

它仍走 `IQ` 壳，但内容预览已 notebook-aware。

### 7. `zf -> Cm4(...)`

这是 `ExitPlanMode` 审批，也就是“计划完成，是否开始执行”。

它不是普通工具审批，而是计划态的专用大面板。

已确认：

- 会展示 plan 内容
- 会展示 requested permissions
- 会根据上下文生成多种“同意后进入哪种 mode”的选项
- `No` 不是简单拒绝，而是“继续 planning 并带反馈”

它本质上是：

```text
plan approval / ready-to-code gate
```

### 8. `je6 -> bm4(...)`

这是进入 plan mode 的审批。

标题固定：

- `Enter plan mode?`

文案会明确说明：

- 先探索代码
- 识别现有模式
- 设计实现方案
- 在获得批准前不改代码

因此它是 plan mode 入口确认框。

### 9. `m76 -> xm4(...)`

这是 Skill 使用审批。

已确认：

- 从输入里解析 `skill`
- 标题格式是 `Use skill "<name>"?`
- 允许：
  - 精确允许某个 skill
  - 允许 `<prefix>:*`
- 使用通用 `HF8(...)` 选项组件，但语义是 skill allowlist

### 10. `Oy6 -> nm4(...)`

这是 `AskUserQuestion` / 多问题问卷审批与作答 UI。

它不是简单 prompt，而是完整问卷系统：

- 多问题分页
- 单选 / 多选 / `__other__`
- 文本输入
- option preview
- 图片粘贴
- review answers
- submit / cancel

在 plan mode 下还额外支持：

- `Respond to Claude`
- `Finish plan interview`

这是当前最重的一类 tool-specific 审批 UI。

### 11. `rU / ru / __ -> Nm4(...)`

这三类工具共用一个通用文件路径审批器。

已确认：

- `rU`
  - `FindFiles`
- `ru`
  - `Grep / Search`
- `__`
  - `Read`
- 若工具提供 `getPath()`，会抽出 path
- 用 `tool.userFacingName()` 生成可读名称
- 根据 `tool.isReadOnly()` 决定标题：
  - `Read file`
  - `Edit file`
- 继续走 `IQ` 壳

这说明 bundle 里至少有三种 path-aware 工具共享这套 UI。  
它们现在已经可以直接点名为：

```text
rU = FindFiles
ru = Grep/Search
__ = Read
```

### 12. `Xa1` 说明 `Nm4(...)` 不是“只服务这三种工具”

另外还能直接看到：

- `Xa1`
  - 即 LSP/代码智能工具
  - `checkPermissions()` 同样走 `n76(...)`
  - 也提供 `getPath({ filePath })`

但当前 tool-specific 分发表里没有看到 `Xa1` 的稳定专用 UI 闭环。  
这说明更稳的判断是：

- `Nm4(...)` 确实是通用 path-aware 审批壳
- `rU / ru / __` 是当前静态表里已点名的三种
- `Xa1` 至少证明：`n76(...)` 这条 path-aware 权限核心并不只服务 `Read / Grep / FindFiles`

但这一段现在还能再收紧一层：

- 对当前 2.1.197 `package/preprocessed/cli.extracted.bundle.pretty.js` 与压缩 bundle 做全文搜索，已经没有 `hVz / SVz / bVz / xVz / RVz / CVz / IVz / uVz` 活字面命中
- 这组名字只能作为旧恢复口径或旧混淆名边界处理，不能继续写成当前 bundle 的本地变量
- 因而就当前 bundle 而言，**不存在把 `Xa1` 直接接到某个旧名保留槽位上的硬证**

因此当前不能再把“`Xa1` 很可能对应这四个槽位之一”写得太实。  
更稳的说法应是：

- `Xa1` 复用了 path-aware permission core
- 但它最终走哪套审批 UI，在本地可见 bundle 里没有闭环
- 若未来存在对应关系，更可能发生在：
  - bundle 外注入
  - build-time 裁剪后缺失的分支
  - 或保留别名/占位槽位

### 13. 旧名保留槽位边界

早期恢复口径里曾把 `hVz / SVz / bVz / xVz` 与 `RVz / CVz / IVz / uVz` 记为 tool-side / component-side 成对保留槽。当前 2.1.197 readable bundle 需要把这件事重新定界：

- 当前 bundle 内这些旧名没有活字面命中。
- 因此不能把它们描述成当前本地运行时的 dormant variable、case label 或 component slot。
- 如果旧名在历史版本里确实对应某组保留槽，它们在当前版本中已经被重新混淆、裁剪，或迁移到 bundle 外 wiring；仅靠当前本地 bundle 不能恢复旧名到当前 tool family 的对应关系。

这里最关键的运行时边界是：

- 当前不能再把这组旧名当作 2.1.197 的“待补工具类型”。
- 当前可实现边界应回到真实可见的 permission dialog registry 与 generic fallback。
- 对旧名的后续追踪只能作为历史对比或 bundle 外扩展调查，不进入当前确定性主链。

因此更稳的表述应改成：

- 这不是“四种本地还没追平的活工具”
- 也不再能在当前 bundle 中证明为“四对本地变量槽”
- 当前只保留为历史/扩展边界，不影响本地审批主链

若未来存在真实语义，更像来自：

- bundle 外 wiring
- build-time 条件编译/裁剪
- 更完整发行物里的可选 feature slice

“延迟注入”仍然可以作为候选解释之一，但不能再写成当前本地运行时的既成事实。

## 两类通用审批壳

### `IQ(...)`

这类壳主要服务“有路径、可预览 diff、可套 IDE diff 支持”的工具：

- 文件编辑
- 写文件
- notebook 编辑
- 通用 path-aware 工具

特点：

- 标题 / 副标题 / 问句 / content 区块分离
- 能接 IDE diff support
- 能按 path / operationType 记录 completion telemetry

### `HF8(...)`

这是通用选项确认器。

当前明确服务于：

- skill 使用
- 以及若干非文件型工具确认

特点：

- option 可带 `feedbackConfig`
- 可进入 accept/reject feedback mode
- 可绑定 keybinding

## Bash / PowerShell 的“建议规则”不是附属信息，而是一级交互

`xNz(...)` 和 `KB4(...)` 都会从 `permissionResult.suggestions` 中生成：

- `Yes`
- `Yes, and don't ask again ...`
- `No`

并支持：

- 直接应用建议规则
- 手动修改 command prefix
- 在 accept/reject 时附带 instructions

这意味着命令审批 UI 已经把“本次放行”和“未来规则写回”合到同一层。

## generic fallback：未命中特化 UI 时不是空白

旧名槽位在当前 bundle 中已经不能作为活证据；但 fallback 语义仍然能从通用审批链钉住：

- 如果某个 ask-capable tool 没有命中特化 dialog，审批链会退回完整的 generic option UI。
- generic fallback 不是空壳，它仍会展示工具名、工具请求摘要、permissionResult 和标准选择项。

generic fallback 至少会做这些事：

- 用 `tool.userFacingName(input)` 生成标题
- 打 `tool_use_single` telemetry
- 展示 `tool.renderToolUseMessage(...)`
- 展示 `permissionResult`
- 通过 `HF8(...)` 提供标准选项

当前能直接确认的标准选项至少包括：

- `Yes`
- `Yes, and don't ask again ...`
- `No`

其中“don't ask again”分支会写入：

- `addRules`
- `behavior: "allow"`
- `destination: "localSettings"`
- 规则主体只带 `toolName`

拒绝分支则会走：

- `toolUseConfirm.onReject(...)`

因此对当前本地 bundle 来说，即便旧名槽位完全不存在，审批主链也不会失效。  
更稳的运行时语义是：

```text
当前本地 bundle
-> tool-specific dialog registry
-> 命中特化 UI：走专用组件
-> 未命中特化 UI：走 generic fallback

旧名槽位:
-> 当前 bundle 无活字面命中
-> 只作为历史/扩展调查对象
```

所以这组黑箱更像：

- 专用审批体验增强槽位

而不是：

- 审批流程能否运行的必要依赖

## 当前已钉死的结论

- `tool-permission` 不是单一弹窗，而是一组专用审批组件的分派树。
- 文件编辑、文件写入、notebook 编辑、通用 path 工具共用 `IQ` 壳。
- Bash 与 PowerShell 是两条独立命令审批链，都支持 destructive warning 与 allow rule 写回。
- `ExitPlanMode`、`EnterPlanMode`、`AskUserQuestion`、`Skill` 都有各自专用 UI，不适合被简化成普通 `confirm(prompt)`。
- `rU / ru / __` 现在已经能点名为 `FindFiles / Grep / Read`。

## 仍未完全钉死

- 当前 ask-capable built-in 余集已经明显收缩：
  - `WebSearch`
    - `checkPermissions() -> passthrough + allow rule suggestion`
    - 但未见稳定专用审批 UI 分支
  - generic `MCPTool`
    - 静态 `mcp` singleton 与动态 MCP tool wrapper 都是 `checkPermissions() -> passthrough`
    - 同样未见稳定专用审批 UI 分支
  - `Xa1`
    - `isLsp`
    - `checkPermissions() -> n76(...)`
    - 有独立 `renderToolUseMessage`
    - 但也未见稳定专用审批 UI 分支
- `ListMcpResourcesTool` 与 `ReadMcpResourceTool` 当前都能直接看到 `checkPermissions() -> allow`。
  - 这说明 MCP 的 resource/read 支线不是 surviving ask-family。
- `hVz / SVz / bVz / xVz / RVz / CVz / IVz / uVz` 在当前 2.1.197 bundle 内没有活命中，不能再作为当前本地变量或当前最佳候选继续推断。
- generic fallback 本身已经会渲染：
  - `tool.userFacingName(input)`
  - `tool.renderToolUseMessage(...)`
  - `permissionResult`
  - `HF8(...)` 的标准选项/反馈壳
  因而像 `WebSearch`、generic `MCP` 这类 surviving family，即便没有专用组件，也不会在本地审批链上缺掉核心交互。
  这进一步说明旧名槽位最多是历史/扩展调查对象，而不是当前本地审批主链的必需拼图。

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
