# 重写候选架构、职责边界与落地顺序

## 本页用途

- 用来把前面已经还原出来的运行时职责边界，转成可执行的重写约束与候选架构。
- 用来说明哪些内容可以直接据此开工，哪些内容只能先预留扩展位，不能在这里提前写死。

## 相关文件

- [../00-overview/01-scope-and-evidence.md](../00-overview/01-scope-and-evidence.md)
- [../01-runtime/04-agent-loop-and-compaction.md](../01-runtime/04-agent-loop-and-compaction.md)
- [../01-runtime/05-model-adapter-provider-and-auth.md](../01-runtime/05-model-adapter-provider-and-auth.md)
- [../01-runtime/12-settings-and-configuration-system.md](../01-runtime/12-settings-and-configuration-system.md)
- [../02-execution/03-prompt-assembly-and-context-layering.md](../02-execution/03-prompt-assembly-and-context-layering.md)
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents.md)
- [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)
- [02-open-questions-and-judgment.md](./02-open-questions-and-judgment.md)

## 使用边界

- 本页不是“原始源码目录结构还原页”。
- 本页给出的内容是**候选架构**，不是“已确认原版就这样分文件”。
- 若后续正文证据与本页候选设计冲突，以正文证据页为准，并回改本页。
- 本页只把当前 evidence map、主题矩阵和正文专题已经支撑住的边界转成重写建议；服务端黑箱、bundle 外 producer 和灰度路径只进入扩展位，不进入确定性设计。

## 当前证据支撑程度

当前已经能支撑“职责边界”的模块，不等于已经还原了原始源码文件、类型名或目录组织。更稳的阅读方式是按下面三档处理：

| 档位 | 可用于重写的结论 | 典型模块 |
| --- | --- | --- |
| 主干可直接实现 | 本地 bundle 已能证明主调用链、状态边界或输入输出合同 | `cli`、`settings`、`engine`、`session`、`prompt`、`tools`、`permissions` |
| 主链可实现，边角留扩展位 | 主 workflow 已可见，但仍有服务端、bundle 外或低频分支 | `hooks`、`agents`、`mcp`、`plugins`、`skills`、`remote / bridge`、`ui` |
| 工程实现选择 | 只是重写时的组织便利，不代表原版就有同名模块 | `state`、`shared`、具体目录和文件名 |

## 当前可以直接继承的职责边界

基于现有 runtime、execution、ecosystem 三组正文，当前已经足以直接固定下面这些模块级职责：

- `cli`
  - 负责入口分流、Commander 命令树、非交互模式判定、启动前 settings 预加载
- `settings`
  - 负责 source/path、effective merge、cache/watcher、flag 注入、write-back，以及对 permission / plugin / MCP / model / env 的运行时配置分发
- `engine`
  - 负责 headless / repl 运行路径、输入编译、主 turn loop、模型调用门面
- `session`
  - 负责 transcript、resume、fork、sidechain、file history、统计与本地持久化
- `prompt`
  - 负责 instruction discovery、rules、skills、prompt compose、snapshot/cache
- `tools`
  - 负责 registry、执行器、并发工具调度、permission merge、tool result 回写
- `hooks`
  - 负责 hook schema、hook runtime、pre/post tool 与 session 级事件桥接
- `agents`
  - 负责 subagent、agent team、mailbox、teammate runtime 与约束裁剪
- `mcp / plugins / skills`
  - 负责 MCP server/source 管理、plugin marketplace、skill registry、安装缓存、能力注入与 policy gate
- `remote / bridge`
  - 负责 remote-control、remote session、direct connect、bridge transport 与冲突还原
- `ui`
  - 负责 TUI root、transcript、dialog、approval、tool result renderer 等前台表现

这些是当前能从发行版分析中稳定还原的**职责边界**。  
更细的文件拆分、类型命名与目录组织，仍属于工程实现选择。

## 模块合同矩阵

下面的矩阵把候选目录骨架进一步压成施工合同。它约束的是模块之间的输入输出和依赖方向，不代表原始源码存在同名包或同名文件。

| 模块 | 主要职责 | 主要输入 | 主要输出 | 依赖方向 |
| --- | --- | --- | --- | --- |
| `cli` | 入口分流、Commander 命令树、启动形态选择、启动前配置预热 | argv、env、cwd、wrapper/native 启动信息 | `RunMode`、初始 settings source、headless/repl/daemon/remote 启动请求 | 只编排 `settings / engine / session / remote / ui`，不持有业务状态 |
| `settings` | settings source、merge、cache、watcher、flag 注入、writeback、policy limits | settings 文件、CLI flag、env、enterprise policy、remote managed settings | `EffectiveSettings`、policy gate、consumer-specific config view | 被多数模块读取；不反向调用 `engine`、`ui` 或具体 tool |
| `engine` | headless/repl turn loop、输入编译、模型调用门面、tool round 调度 | compiled input、session state、effective settings、prompt/tool registries | assistant events、tool use request、turn result、terminal reason | 依赖 `prompt / tools / session / hooks` 的接口，不依赖具体 UI renderer |
| `session` | transcript、resume、fork、sidechain、file history、统计、本地持久化 | session id、project/worktree、transcript events、compact/fork 请求 | persisted transcript、resume/fork state、file-history snapshot、stats | 向 `engine / ui / remote` 提供状态接口；不直接执行 tool 或构造 prompt |
| `prompt` | instruction discovery、rules、context layering、request payload compose、prompt cache | user input、transcript slice、attachments、settings、tool registry view、skills/MCP deltas | `payload.system`、`payload.messages`、request options、transport metadata | 被 `engine` 调用；通过 registry view 读取扩展能力，不直接启动 MCP/plugin/skill |
| `tools` | builtin/deferred tool registry、执行器、并发调度、permission merge、tool result 写回 | tool use blocks、`ToolUseContext`、settings/policy、hook decisions、MCP/tool schemas | transcript messages、`ToolExecutionOutput`、`contextLayers`、sidecar result、stop signal | 依赖 `hooks / permissions / mcp / skills` 的接口；不直接改 prompt payload |
| `hooks` | hook schema、matcher、runtime、typed output channel、event consumer | hook definitions、event input、tool/session/prompt callsite | typed hook outputs、permission decision、display-only output、side effects | 由 `engine / tools / prompt / session` 在明确 callsite 调用，不拥有主循环 |
| `agents` | subagent、agent team、mailbox、teammate runtime、权限与路径裁剪 | task/agent config、session/fork context、mailbox messages、team state | subagent run request、mailbox events、team task state、permission relay | 通过 `engine / session / tools` 接入；不能绕开 permission/tool context |
| `mcp` | MCP server/source 管理、resources、deferred tool、SDK control channel | `.mcp.json`、settings/policy、server transport、resource notifications | MCP client registry、resource snapshot、tool schema reference、instructions delta | 向 `prompt / tools / ui` 暴露 registry，不把协议事件直接塞进 engine |
| `plugins` | marketplace、manifest、install cache、enabled state、runtime injection、policy gate | marketplace manifest、plugin directory/cache、settings/policy | plugin-provided MCP/skills/settings/channels、validation result | 通过 `mcp / skills / settings` adapter 注入，不直接写主循环 |
| `skills` | `SKILL.md` loader/cache、skill listing、dynamic skill、SkillTool fork/inline 执行 | skill dirs、plugin/MCP skill builders、dynamic triggers、tool input | skill listing attachment、invoked skill record、`contextLayers`、subagent request | 作为 `prompt / tools / agents` 的 registry provider，不直接改 session store |
| `remote` | remote-control、bridge、direct connect、credential handoff、conflict recovery | remote flags、control-plane responses、ingress token、bridge stream | remote session state、transport adapter、opaque credential fields、replay events | 包装 transport/session 边界；不在本地重写 `payload.system/messages` |
| `ui` | TUI root、input footer、dialog/approval、message row、tool result renderer | app state、transcript/events、approval requests、renderer sidecar | rendered screens、user actions、approval responses、display-only status | 观察 `session / engine / tools` 事件；不直接执行 tool 或写持久化 |
| `state` | 运行期 app/session/tool/ui 状态容器与快照边界 | reducer events、session snapshots、remote replay、UI actions | typed state view、snapshot、selector result | 工程便利层；保持无业务副作用，避免变成全局大对象 |
| `shared` | ids、paths、env、logger、schema utils、公共错误类型 | platform/env、project paths、raw values | normalized primitives、helpers、error/result types | 只能放低层通用能力；不依赖任何业务模块 |

## 依赖方向与扩展承接

重写时应保持下面这组方向约束：

- `cli` 只做启动编排；进入 `engine` 后不再让命令处理函数直接持有 session、prompt 或 tool 状态。
- `settings` 是横向输入层；consumer 可以请求裁剪后的 view，但 settings 不主动调用 consumer。
- `engine` 只认识 `PromptComposePipeline`、`ToolRunner`、`HookRuntime`、`SessionStore` 这类接口，不认识 TUI 组件、MCP transport 细节或 plugin manifest 细节。
- `prompt` 输出分成 `system/messages/requestOptions/transportMetadata`；服务端追加、灰度包装、verification / compat policy wrapping 只进入 transport adapter 或 opaque extension。
- `tools` 返回 `ToolExecutionOutput`，由 executor 统一提交 transcript、sidecar、`contextLayers` 与 stop signal；tool 本身不直接改主循环状态。
- `hooks` 的 parser 与 callsite consumer 分离；新增 hook 字段可以先被解析，但只有对应事件消费的 channel 才生效。
- `mcp / plugins / skills / agents` 通过 registry 或 adapter 接入 `prompt / tools / ui`，不把具体 producer 写死在 `engine` 内。
- `remote` 只包装 transport、credential 和 replay/conflict recovery；本地可见 payload build 完成后不假设 remote 会改写 prompt。
- `ui` 只消费状态和 sidecar；message subtype、tool result、approval descriptor 都走 registry/fallback，而不是把低频 tool 分支散落到组件树里。
- `shared` 与 `state` 只放跨模块基础设施。发现它们开始持有业务决策时，应把决策移回对应模块。

服务端黑箱、bundle 外 producer 与灰度路径统一按下面三类承接：

| 类型 | 承接位置 | 写法要求 |
| --- | --- | --- |
| 服务端黑箱 | transport adapter、opaque response/request metadata、remote/control-plane adapter | 记录本地可见字段和错误恢复分支，不声明服务端内部策略 |
| bundle 外 producer | registry item、adapter extension、optional typed channel | parser/类型可接收，主链只消费当前本地可证的 channel |
| 灰度 / dormant 路径 | strategy registry、feature flag provider、telemetry-only marker | 保留后期收窄入口，不把未启用分支写成活控制流 |

## 候选分层

当前更稳的重写思路，不是先追求文件树长什么样，而是先保持下面这组层次分离：

### 1. 启动与命令层

- `cli` 只负责命令面、参数解析、启动分流与初始化时机。
- 不把 session、prompt、tool、remote 的实际运行逻辑继续塞回 CLI 命令处理函数。

### 2. 配置与策略层

- `settings` 应保持成独立层，而不是散落在 CLI、全局 store、UI toggle 和各个 consumer 里各自拼 effective config。
- permission、model、plugin、MCP、remote、env 这些消费面都很多，但 settings 仍应是单独模块，而不是 `shared/config.ts` 一类薄包装。

### 3. 主执行层

- `engine` 负责“输入进入系统后如何跑完整个 turn”。
- `headless` 与 `repl` 应共享 turn loop 和 model call 的核心运行骨架，只在 I/O 形态、前台 UI 和审批交互上分叉。

### 4. 持久化与工作现场层

- `session` 负责持久化模型和还原能力。
- 这一层应继续把 transcript、resume/fork、file-history、plan restore 等内容拆开，而不是重新塞成单个大 store。

### 5. Prompt 与上下文层

- `prompt` 负责 discovery、合成、cache、snapshot 和 request 边界。
- 与 `engine` 的边界应保持为：
  - `engine` 决定何时需要 prompt
  - `prompt` 决定 prompt 如何被构造

### 6. Tool / Permission / Hook 层

- `tools` 负责执行与结果提交。
- `permissions` 与 `hooks` 虽然强耦合 tool round，但不应直接埋在单一 tool executor 内。
- 保持这三者拆开，后续补齐 ask backend、managed policy、hook runtime 时才不会回到大函数。

### 7. 生态扩展层

- `agents`、`mcp`、`plugins`、`skills` 属于扩展能力面。
- 这些能力都需要接入 prompt、tool、settings、runtime，但不应反过来污染主循环的数据结构定义。
- MCP resource、deferred tool、plugin MCP、SkillTool 和 team mailbox 都应通过 registry / adapter 接入，不把具体 producer 写死在 engine 内。

### 8. 远端与控制面层

- remote-control、bridge、direct connect、control-plane API 应独立成层。
- 它和 provider/model call 是正交关系，不应混为“模型请求的一个特殊分支”。

### 9. 表现层

- `ui` 只解决 TUI / dialog / renderer / input footer / transcript 这些前台表现问题。
- `ui/renderers` 至少应有 message subtype registry、tool result registry 和 default JSON/text/status fallback；具体 tool 自定义 renderer 作为 registry item 接入，不要写成 transcript block 的字符串打印。这个 default fallback 只用于 unknown / bundle 外 / deferred tool，已知 builtin tool 仍以自己的 `renderToolResultMessage(...)` 为准。
- 不把状态机、tool 执行器、bridge 协议消费重新塞进组件树里。

## 一份可施工的目录骨架

下面这份目录骨架只是**一个可行拆法**，用来约束重写时的分层，不代表原始源代码文件树就是这样：

```text
src/
  cli/
    entry.ts
    program.ts
    commands/
  settings/
    sources.ts
    loader.ts
    merge.ts
    cache.ts
    writeback.ts
  engine/
    headless.ts
    repl.ts
    turn-loop.ts
    call-model.ts
    input-compiler.ts
  session/
    transcript.ts
    persistence.ts
    resume.ts
    fork.ts
    file-history.ts
    stats.ts
  prompt/
    discovery.ts
    compose.ts
    cache.ts
    rules.ts
    skills.ts
  tools/
    registry.ts
    executor.ts
    concurrent-runner.ts
    permissions.ts
    builtins/
  hooks/
    runtime.ts
    schema.ts
  agents/
    registry.ts
    launch.ts
    mailbox.ts
  mcp/
    manager.ts
    client.ts
  plugins/
    manager.ts
    manifest.ts
  skills/
    registry.ts
    executor.ts
  remote/
    control-plane.ts
    bridge.ts
    direct-connect.ts
  ui/
    app.tsx
    transcript/
    dialogs/
    renderers/
      message-registry.ts
      tool-result-registry.ts
  state/
    app-state.ts
    session-state.ts
  shared/
    ids.ts
    paths.ts
    env.ts
    logger.ts
```

如果开始真正施工，建议优先保证：

- 单个文件只负责一个稳定问题域
- 大页里已经独立成专题的职责，不要在实现时再重新耦合
- 不要为了“像原版”而把已经拆清的边界重新揉回单文件
- 对服务端黑箱、bundle 外 producer、灰度路径使用 adapter / registry / opaque field，而不是在类型里假装已经穷尽

## 骨架维护约束

这份骨架的维护目标是让后续实现有稳定施工合同，而不是在恢复文档里提前生成代码。当前阶段不创建 `src/**`，也不把上面的候选目录写成“原版源码文件树”。

后续任何实现或补证据若要改动骨架，应按下面顺序判断：

- 先看模块合同矩阵：职责、输入、输出或依赖方向是否真的变化。
- 再看接口主线：`PromptComposePipeline`、`ToolExecutionOutput`、`HookRuntimeInput`、`Extension Registry`、`Turn Loop State` 是否需要新增字段或分支。
- 最后看扩展承接：新增事实属于服务端黑箱、bundle 外 producer、灰度路径，还是当前本地确定性主链。

稳定边界应保持：

- `engine` 不直接依赖 TUI 组件、MCP transport、plugin manifest 或 remote control-plane 细节。
- `prompt` 输出继续分成 `payload.system`、`payload.messages`、request options 与 transport metadata。
- `tools` 只通过 executor 统一提交 transcript、sidecar、`contextLayers` 与 stop signal。
- `hooks` 保持 parser 与 callsite consumer 分离，不能把所有输出压成单一 string。
- `mcp / plugins / skills / agents` 继续通过 registry 或 adapter 进入主链。
- `ui` 只消费 session / engine / tools 状态与 sidecar，renderer fallback 只用于 unknown、bundle 外或 deferred tool。
- `state` 与 `shared` 只能承接跨模块基础能力，不能变成业务决策集中地。

如果后续专题页证明某个未知点进入本地活控制流，应先更新对应专题和 evidence map，再回改本页的合同或接口表；不要只在本页单独追加判断。

## 接口还原目标

这一节只描述**接口边界**，不把当前还没证死的字段细节提前写成最终类型定案。

### Transcript / Message 族

至少应保留：

- `user`
- `assistant`
- `system`
- `progress`
- `attachment`

并且要支持：

- block-based content
- tool use / tool result
- compact 与 retry 相关 system subtype
- tool use 关联关系

当前不宜提前写死的部分：

- 每个 `system` subtype 的完整字段族
- 远端协议型 system 行在不同前台路径中的最终可见性

### Session 持久化对象

至少应覆盖：

- session identity
- project/worktree 关联
- transcript 消息集合
- parent / fork 关系
- file-history / attribution / content replacement / compact snapshot 之类的工作现场

当前不宜提前写死的部分：

- 所有附属 snapshot 的完整字段结构
- 远端 session 相关扩展字段是否全部本地可见

### Compiled Input

至少应覆盖：

- 编译后的 messages
- 是否触发模型请求
- 本轮结果文本与输出模式
- 工具 allowlist / model override / plan 之类的请求级覆盖
- request-level prompt 本地分层和 transport metadata

当前不宜提前写死的部分：

- web/editor 侧 plan 注入的完整前端载荷
- 服务端收到 payload 后是否还会做额外 context 注入
- `_H(...)` 预留第二注入槽位的原始用途

### PromptComposePipeline

当前本地证据已经足以把 `prompt` 模块落成分阶段 pipeline。这里的阶段表描述的是 **本地 request build 到 provider call 前的接口边界**，不是服务端最终 prompt 的完整复刻。

| 阶段 | 主要输入 | 本地输出 | 已确认字段 / 顺序 | 保留扩展位 |
| --- | --- | --- | --- | --- |
| `discoverInstructions` | cwd、settings、managed/user/project/local rules、additional directories | `userContextSeed` | `Qv() -> B4t(...)` 生成 `claudeMd`；`O4t("compact")` 只预置下一次 fresh scan 的 `load_reason`，不会立即重扫 | bundle 外 rules producer、服务端 policy 二次下发 |
| `buildUserContext` | `userContextSeed`、account、attached project、current date marker | `userContext` | 默认可见为 `claudeMd? / userEmail? / attachedProject? / currentDate`；经 `otr(...)` 前插到 `messages[]` 最前 | 额外 user meta fields |
| `buildSystemContext` | git/perforce/workspace 状态、cache breaker phrase | `systemContext` | `_H(...)` 默认返回 `gitStatus? / perforceMode?`；`CDl(...)` 用 `Object.entries(systemContext)` 拼到 system 末尾 | `_H(...)` 预留的第二注入槽位和未来 systemContext 字段 |
| `selectSystemPrompt` | override、active agent、custom prompt、`zL(...)`、append prompt | `systemSections` | 优先级为 `overrideSystemPrompt > active agent prompt > customSystemPrompt > zL(...) default sections > appendSystemPrompt` | advisor / agent 专用 system section |
| `compileTurnMessages` | transcript、当前 user、attachments、hook results、skill/plan/meta attachments | transcript-like messages | 普通当前轮近似为 `user -> attachments -> UserPromptSubmit hook attachments`；`skill_listing / plan_mode / critical_system_reminder` 保持 attachment 顺序 | 低频 attachment subtype 的 renderer-only sidecar |
| `normalizeMessages` | transcript-like messages、available tools、model | API `messages` source | `Flm(...) -> Ak(...)` 先移动 attachment，再合并相邻 user；`otr(userContext)` 作为前置 meta user message | fork / compact helper 的 snapshot reuse 参数 |
| `buildSystemBlocks` | billing/header、identity、`systemSections`、`systemContext` | API `system` blocks | `Hpc(...) -> Ec(...) -> Wlm(...)`；按 `null/org/global` scope 拆成多 block，MCP tool schema 可触发 global cache 降级 | 服务端 system augmentation |
| `buildRequestOptions` | model、tools、permission context、thinking、betas、output config | request options | `tools / tool_choice / betas / metadata / max_tokens / thinking / context_management / output_config / speed` 不属于 prompt 文本 | provider-specific body params、server fallback / fallback credit fields |
| `sendViaTransport` | payload、provider client、auth、remote/bridge ingress | provider request | 当前本地可见 transport 不在 `Hpc(...)` 后改写 `system/messages` | server-side opaque extension、verification / compat / policy wrapping |

重写时可以把这个表直接转成 `prompt/compose.ts` 的接口边界：

```text
PromptComposeInput
  -> discoverInstructions()
  -> buildUserContext()
  -> buildSystemContext()
  -> selectSystemPrompt()
  -> compileTurnMessages()
  -> normalizeMessages()
  -> buildSystemBlocks()
  -> buildRequestOptions()
  -> sendViaTransport()
```

其中 `buildSystemBlocks()` 和 `normalizeMessages()` 的输出已经能分别对应 `payload.system` 与 `payload.messages`；`buildRequestOptions()` 只负责 schema / options，不应混入 prompt 文本。服务端追加、灰度包装、verification / compat 这类不可见内容只进入 `serverSideOpaqueExtension` 或 transport adapter，不作为本地确定性阶段。

### Tool Execution Output

至少应支持：

- 单条或多条 transcript 写回
- `contextLayers` / context overlay
- structured output / sidecar payload
- MCP 相关附加元数据
- 停止后续 continuation 的信号

当前不宜提前写死的部分：

- bundle 外 / 服务端侧是否还有额外 producer
- 一些低频工具的 sidecar 形状

当前本地 `2.1.197` 可见实现应把早期概念名 `contextModifier` 落成更窄的 layer 接口：

```text
ToolExecutionOutput
  -> message / messages
  -> attachments
  -> contextLayers?: ToolContextLayer[]
  -> mcpMeta?
  -> stop / preventContinuation?

ToolContextLayer
  -> allowed_tools / disallowed_tools
  -> model / effort / max_thinking_tokens
  -> permission_mode / working_directory / flag_settings

NYt(toolUseContext, layers)
  -> append permissionLayers
  -> update options.mainLoopModel / options.thinkingConfig when needed
```

已确认的 tool-returned producer 至少包括：

- `SkillTool` inline：`allowed_tools / model / effort`
- `EnterWorktree`：`working_directory`

fork skill、fork slash command、remote / bridge 和 SDK 入口也会预置或覆盖上下文，但它们不是同一种“父线程 tool result 返回后提交”的协议。重写时可以把 `contextLayers` 做成正式扩展字段；若要兼容旧设计名，可在 adapter 层把 `contextModifier` 视为历史别名，而不要在核心 executor 里实现任意函数式 context mutation。

### Hook Runtime Event

至少应支持：

- event registry：
  - `PreToolUse / PostToolUse / PostToolBatch / UserPromptSubmit / SessionStart / Stop / SubagentStop / PermissionRequest / PermissionDenied / InstructionsLoaded / MessageDisplay / Worktree*` 等当前本地可见事件
- hook def registry：
  - `command / prompt / agent / http / mcp_tool / callback / function`
- typed output channel：
  - transcript message：`hook_success / hook_non_blocking_error / hook_system_message / hook_stopped_continuation`
  - permission：`permissionBehavior / permissionRequestResult / retry`
  - input/output rewrite：`updatedInput / updatedToolOutput / updatedMCPToolOutput`
  - prompt/context：`additionalContext / initialUserMessage / sessionTitle / reloadSkills`
  - control：`blockingError / preventContinuation / stopReason`
  - display-only：`terminalSequence / displayContent`
  - lifecycle side-effect：`watchPaths / worktreePath`
- runtime ordering：
  - matcher 顺序稳定
  - hook 执行并发
  - 结果按完成顺序消费
  - command hook 支持 async 首行握手与 `forceSyncExecution`

当前不宜提前写死的部分：

- bundle 外是否还有额外事件分支
- 服务端或插件外层是否会追加新的 event / output producer

候选接口可按下面拆：

```text
HookRuntimeInput
  -> resolveHookDefinitions()
  -> matchHooks(event, matchQuery)
  -> emitProgress()
  -> executeHooksConcurrent()
  -> parseTypedOutput()
  -> consumeByEventCallsite()
```

其中 `parseTypedOutput()` 只负责把 stdout / HTTP body / callback result 变成 typed channel；`consumeByEventCallsite()` 决定哪些 channel 对当前事件真正生效。这样可以保留 `MessageDisplay` 的 display-only 边界、`PermissionRequest` 的 ask 后语义、`PreToolUse defer` 的 print-mode 限制，以及 `SessionEnd / WorktreeRemove` 这类只消费成功失败状态的路径。

### Extension Registry

至少应支持：

- MCP server / resource / deferred tool 注入
- plugin marketplace / installed plugin / policy gate
- skill listing / dynamic skill / SkillTool
- agent team / mailbox / teammate backend

当前不宜提前写死的部分：

- `resources/updated` 与 `<mcp-resource-update>` / `<mcp-polling-update>` 的产品级 producer
- bundle 外是否会追加 builtin plugin 条目；当前 readable bundle 的 builtin plugin registry 可见为空
- bundle 外 skill producer
- team backend 的更多运行形态

### Turn Loop State

至少应跟踪：

- 当前消息集
- tool use context
- compact / recovery 状态
- pending tool summary

当前不宜提前写死的部分：

- `transition` 一类 branch marker 是否保留成调试/快照字段；就当前 bundle 而言，它不该被当成活控制流输入
- 所有 retry / telemetry / header 的边缘状态位
- 远端路径下服务端 sideband 行为的完整镜像

## 落地顺序

### Phase 1：可运行的 Headless 主干

目标：

- 跑通 `prompt -> model -> tool use -> transcript -> result`
- 支持 `json / jsonl / stream-json`
- 支持基础 resume / fork
- 支持显式 provider/auth adapter、settings source merge、tool output persistence / truncation、`PostToolBatch` 与 compact producer adapter 的 headless 行为

优先实现的层：

- `cli`
- `settings`
- `engine`
- `session`
- `prompt`
- `tools`
- `hooks`

### Phase 2：还原本地工作现场

补齐：

- file-history
- plan restore
- content replacement
- queued commands
- invoked skills
- attachment 体系

### Phase 3：还原外围能力

补齐：

- hooks
- MCP
- plugins
- skills
- agents / subagents
- remote-control / bridge
- TUI

### Phase 4：对齐边缘行为

逐步校对：

- request-level prompt 最终线性顺序
- `Mi6("compact")` 与相关失效链的完整覆盖面
- model fallback / retry 的全部边角
- 服务端或 bundle 外追加的 context / compat / verification 行为
- telemetry / cache / header 的边缘行为

## 暂不应提前定案的部分

下面这些点不妨碍开工，但不适合在本页里写成“最终设计已经确定”：

- request-level prompt 的最后顺序，以及本地/服务端边界后的残余注入点
- Hook 的 bundle 外扩展分支与少量边角时序
- 是否还有额外 bundle 外 `contextLayers` producer，或历史 `contextModifier` 兼容入口
- bridge / worker 的服务端正式语义
- 少量只影响 1:1 复刻的 TUI 微观交互细节，例如 fullscreen virtual-scroll/search geometry

更稳的工作方式是：

- 先按已确认职责边界拆模块
- 对未决区保留扩展位
- 等对应专题页继续补证据后，再收窄实现

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
