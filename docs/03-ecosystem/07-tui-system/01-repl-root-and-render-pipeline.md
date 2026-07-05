# REPL Root 与 Render Pipeline

## 本页用途

- 把 `f3A(...)` 这一层主 REPL root 的状态骨架、render 分支和布局槽位写实。
- 固定 `Aj6` 消息区不是“把 transcript 逐条打印出来”，而是经过筛选、折叠、截断后的渲染流水线。
- 区分 `Aj6` 内部预留的 virtual-scroll/search 接口，与当前主 REPL 路径实际上是否接活。

## 相关文件

- [../07-tui-system.md](../07-tui-system.md)
- [02-transcript-and-rewind.md](./02-transcript-and-rewind.md)
- [03-input-footer-and-voice.md](./03-input-footer-and-voice.md)
- [04-dialogs-and-approvals.md](./04-dialogs-and-approvals.md)
- [07-message-row-and-subtype-renderers.md](./07-message-row-and-subtype-renderers.md)

## 主 REPL root：`f3A(...)`

`f3A(...)` 不是单纯“输入框 + 消息列表”，而是 TUI 主工作台。  
当前能直接看到它同时管理：

- `screen`：`B6`
- `showAllInTranscript`：`N6`
- `streamMode`：`F4`
- `isLoading`：`i4`
- `messages`：`O4`
- `inputValue`：`V2`
- `inputMode`：`cV`
- `toolJSX`：`s1`
- `toolUseConfirmQueue`：`iq`
- `sandbox permission queue`：`Lq`
- `worker sandbox queue`：`$6.queue`
- `elicitation queue`：`T6.queue`
- `message selector open`：`cQ`
- `helpOpen`：`$H6`
- `history search open`：`YR6`
- `vimMode`：`bX`
- `conversationId`：`Pb`
- `streamingToolUses`：`Mw`
- `streamingThinking`：`U4`
- `streamingText`：`lV / Sa`

这说明 REPL 本体本身就是一台多轴状态机，不存在一个很薄的“页面壳层”。

## 顶层 render 先分 transcript，再分普通 REPL

### transcript 分支

当 `screen === "transcript"` 时，直接走专门分支：

- 主体是 `Aj6(...)`
- 可选挂一个 local JSX 片段
- 底部固定接 `Pbz(...)` transcript 状态条

这个分支不走普通 REPL 的 `Obz(...)` 四槽位拼装。  
因此 transcript 不是“主界面的一个展开模式”，而是**独立视图树**。

### 普通 REPL 分支

非 transcript 时，主界面通过：

```text
Obz({
  scrollable,
  bottom,
  overlay,
  modal
})
```

来拼装四个槽位。

当前已经能直接写死的分配：

- `scrollable`
  - `$Q4`
  - `Aj6`
  - prompt suggestion / streaming spinner / 空闲占位
- `bottom`
  - `E4`
  - immediate local JSX
  - task/permission/prompt/elicitation/cost/idle-return 等下半区组件
  - `PF4`
  - `EQ4`
- `overlay`
  - 当前直接看到 `tool-permission`
- `modal`
  - 代码里作为独立 slot 预留，但当前主路径未看到持续占用

## `Aj6(...)`：消息区不是 transcript 原样直出

`Aj6` 对输入消息做的事情至少包括：

1. 先把 `messages` 过 `JP(...)` 一层标准化。
2. 把未完成的 `streamingToolUses` 转成临时 assistant/tool-use 伪消息，拼回消息流。
3. 过滤不该显示的 `progress` / 非 transcript-only / 部分系统项。
4. 按 `brief`、`hidePastThinking`、`showAllInTranscript` 决定保留多少历史。
5. 通过 `cT4(...) / oT4(...) / nT4(...) / HNq(...)` 再做一层 renderable message 变换。
6. 预留 transcript 搜索定位与 virtual-scroll 切窗接口。

所以 `Aj6` 更接近：

```text
transcript-like messages
-> normalize
-> merge streaming tool-use placeholders
-> filter / collapse
-> brief/show-all windowing
-> virtual-scroll slice
-> render rows
```

## `Aj6` 已确认的几条重要行为

### 1. `showAllInTranscript` 关闭时不是全量显示

当前常量下界是：

- `C1A = 30`

也就是 transcript 视图在 `showAllInTranscript === false` 时，默认只保留最近一段消息窗口；前面的会变成“可展开的旧消息计数”。

### 2. brief 模式会改变消息区裁剪方式

`Aj6` 会根据：

- `isBriefOnly`
- `hidePastThinking`
- `streamingThinking`

来决定过去的 thinking 块是否显示，以及 brief 工具结果是否折叠。

### 3. streaming text 与 streaming thinking 有单独尾部渲染

除了稳定消息对象外，`Aj6` 还会单独渲染：

- 正在流式输出的文本块
- 仍处于 streaming 状态的 thinking 块

因此“assistant 最后一条消息”与“当前正在流的尾巴”不是完全同一个渲染对象。

### 4. 有 unseen divider

消息区会根据 `unseenDivider` 在指定 message 前插入：

- `N new messages`

这说明 TUI 维护了“用户上次看到哪里”的位置状态，而不只是顺序 append。

### 5. virtual-scroll 能力在 `Aj6` 内存在，但当前主 REPL 路径未真正接活

`Aj6` 内部仍保留了 virtual-scroll 相关支点：

- `TOz != null`
- `scrollRef != null`
- 没有 `CLAUDE_CODE_DISABLE_VIRTUAL_SCROLL`

并且会基于 `renderRange` / `yOz(...)` 做切窗。

但对当前主 REPL 分支继续往上追，可以把边界再收紧：

- transcript 分支里 `rL = false`
- 传给 `Aj6(...)` 的 `scrollRef` 是 `void 0`
- 普通 REPL 分支里传给 `Aj6(...)` 的 `scrollRef` 也是 `void 0`
- `R5A(...)` 收到的 `virtualScrollActive` 也是这个 `false`

因此更稳的结论不是“当前 transcript 正在用 virtual scroll”，而是：

- **`Aj6` 里保留了 virtual-scroll/search 的接口层**
- **但这份本地 bundle 的主 REPL/transcript 活路径并没有把它接通**

也就是说，virtual scroll 现在更像：

- 曾经存在或预留的渲染优化层
- 当前 build 中处于未接线 / 被裁剪 / 被硬关闭状态

## `Aj6` 的输入并不只来自主 transcript

消息区实际会混合这些来源：

- 已持久化/已存在 transcript 消息
- 本轮 streaming 中的临时 tool-use 占位
- 当前流式文本尾巴
- 当前流式 thinking 尾巴
- 可能的 tool JSX 插片周边状态

这也是为什么“只还原 transcript schema”还不等于还原 TUI 的显示逻辑。

## keybinding context 说明 TUI 是上下文敏感路由

从 keybindings schema 当前已经能直接反推出一组正式 UI context：

- `Global`
- `Chat`
- `Autocomplete`
- `Confirmation`
- `Help`
- `Transcript`
- `HistorySearch`
- `Task`
- `ThemePicker`
- `Settings`
- `Tabs`
- `Attachments`
- `Footer`
- `MessageSelector`
- `DiffDialog`
- `ModelPicker`
- `Select`
- `Plugin`

这组枚举的重要性不在于“名字很多”，而在于它说明：

- 快捷键不是全局硬编码
- 焦点域是正式状态，而不是组件内部私有开关
- `Help / ThemePicker / ModelPicker / Settings / DiffDialog / Attachments / Footer` 这类表面上分散的 UI，都挂在同一套路由模型下

因此从 root 角度看，TUI 的真实结构不是“一个 REPL 页面”，而是：

```text
App / REPL root
  -> current screen
  -> current focused dialog
  -> current keybinding context
  -> current layout slot tree
```

## fullscreen / no-flicker / footer link 的配置面

当前 bundle 里 TUI renderer 已经有明确 settings/env 合同：

- `tui: "default" | "fullscreen"`
  - `fullscreen` 使用 flicker-free alternate-screen renderer，并带 virtualized scrollback 语义。
  - `default` 保留 classic main-screen renderer。
- `CLAUDE_CODE_NO_FLICKER=1`
  - 和 fullscreen renderer 关系紧密；文案把 fullscreen 描述成等价于 no-flicker 路径。
- `CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN`
  - 禁用 alternate screen；changelog 里若写 `DISABLE_ALTERNATE_SCREEN`，恢复文档应改成当前完整 env 名。
- `autoScrollEnabled`
  - fullscreen mode 的自动滚到底部开关。
- `wheelScrollAccelerationEnabled`
  - fullscreen mode 下鼠标滚轮快速滚动加速。
- `/scroll-speed`
  - 调整每个滚轮 notch 的行数；和 `wheelScrollAccelerationEnabled` 不是同一层。
- `footerLinksRegexes`
  - 为 tool result / assistant response 中匹配到的内容生成额外 footer badge。
  - 只读取 user / flag / managed settings，忽略 project `.claude/settings.json` 和 local `.claude/settings.local.json`。
  - 最多保留 5 个 badge；`/clear` 会清掉。
- `prUrlTemplate`
  - 控制 detected git PR 在 footer badge / inline message 里的 URL 模板。

这组配置说明 fullscreen/no-flicker 不是单纯 UI 文案，而是 renderer 与 terminal 输出策略。重写 TUI 时应把 renderer mode、alternate-screen env、scroll 行为和 footer badge 分开实现，不能压成一个“主题/外观”开关。

## `/tui`、screen reader 与 focus view 的入口边界

`/tui` 是一个会写 user settings 的 renderer 切换命令，不只是提示用户改配置：

- 空参数时只回显当前 renderer：`default` 或 `fullscreen`
- 参数只接受 `default | fullscreen`
- 命中 background session 时不会改设置；background session 固定使用 fullscreen renderer，便于 attach 后滚动和鼠标工作
- screen-reader mode 开启时，`/tui` 设置不会生效，因为 screen-reader mode 固定走 classic renderer
- 切换前如果有仍在运行的普通 background task，会拒绝切换，避免 renderer restart/resume 与在途任务冲突
- 成功切换会写入 user settings 的 `tui`，随后尝试 restart/resume；若 restart 失败，设置仍已保存，需要重启 Claude Code 才应用

screen reader 是独立可访问性模式，不属于 fullscreen 的一个主题：

- CLI flag 是 `--ax-screen-reader`
- settings key 是 `axScreenReader`
- env override 是 `CLAUDE_AX_SCREEN_READER`
- 开启后启动时会输出 `[Accessible screen reader mode: on]`
- schema 描述为 flat text、无装饰边框、无动画

因此 TUI 重写时应把 `tui` renderer、`CLAUDE_CODE_NO_FLICKER`、`CLAUDE_CODE_DISABLE_ALTERNATE_SCREEN` 和 `axScreenReader` 分开建模。screen-reader mode 可以覆盖 renderer 选择，不能简单等同于 fullscreen 的低动效版本。

`/focus` 也是 renderer-dependent 入口：

- 它切换的是 app state / flag settings 中的 brief transcript / focus view
- focus view 需要 fullscreen renderer
- 若 `viewMode: "focus"` 已经来自 settings，命令会提示必须从 settings 删除并重启，不能在运行时直接关掉
- 远端 transport 能力不足时，本地可能只能应用当前端状态，不能保证同步更新远端 session

这说明 focus view 的最小恢复模型不是“隐藏旧消息”的 UI 开关，而是：

```text
fullscreen renderer 可用
  -> /focus 切换 brief/focus view 状态
  -> 必要时通过 control request 同步远端 flag settings
```

## 当前收束边界

- 单条消息 renderer 已经拆到 current bundle 的 `o6l -> gVf -> UQ/Xdf`，并继续分到 `Jdf`、`Qdf`、`Hpl`、`zpl`、`cpl`、`mpl` 等 subtype / tool-result / grouped renderer。具体分派表见 [07-message-row-and-subtype-renderers.md](./07-message-row-and-subtype-renderers.md) 与 [08-tool-result-renderers.md](./08-tool-result-renderers.md)。
- brief / focus view 的主策略已经能写成：先在 `k8f(...)` 内按 `briefTranscript / isBriefOnly / focus view` 改写 message window，再把 read/search/bash/MCP/agent/edit/memory/team-memory 等活动合并成 `collapsed_read_search` 摘要行；active group 额外显示 latest hint、thinking summary 或 bash elapsed/line count。
- tool result renderer 的主 fallback 已经能写成 `zpl(...)` 四分支：interrupt、rejected、error、normal sidecar；正常分支再由 `Fpl(...)` 调 tool 自己的 `renderToolResultMessage(...)`。
- `TOz / renderRange / yOz(...)` 对应的 virtual-scroll/search 支线仍是 dormant 接口层；若要 1:1 复刻，需要继续补它在 fullscreen renderer 里的坐标与挂载细节。
- 搜索命中跳转的更细几何结构仍依赖 renderer 内部坐标对象；这属于后期显示一致性，不阻塞候选 TUI 主架构。

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
