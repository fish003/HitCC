# 重写判断、未决项分级与后续补证

## 本页用途

- 用来回答：基于当前发行版分析，是否已经足以开始重写高相似版本。
- 用来只保留那些会直接影响重写策略、模块边界、接口弹性或后期对齐的未决项。
- 用来把“本地 bundle 能证明什么”和“仍属服务端 / bundle 外黑箱的内容”分开。

## 相关文件

- [01-rewrite-architecture.md](./01-rewrite-architecture.md)
- [../00-overview/01-scope-and-evidence.md](../00-overview/01-scope-and-evidence.md)
- [../02-execution/03-prompt-assembly-and-context-layering.md](../02-execution/03-prompt-assembly-and-context-layering.md)
- [../01-runtime/05-model-adapter-provider-and-auth.md](../01-runtime/05-model-adapter-provider-and-auth.md)
- [../01-runtime/12-settings-and-configuration-system.md](../01-runtime/12-settings-and-configuration-system.md)
- [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)
- [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md)

## 当前判断

### 已经可以做什么

在当前证据边界下，已经足以开始：

- 可运行替代品的重建
- 高相似版本的模块化重写
- 按职责边界拆分 `cli / settings / engine / session / prompt / tools / hooks / agents / mcp / plugins / skills / remote / ui`
- 按 [01-rewrite-architecture.md](./01-rewrite-architecture.md) 的模块合同矩阵建立初版目录、接口和 registry/adapter 扩展位

但仍然不足以承诺：

- 原始源码文件结构还原
- 私有服务端实现还原
- 1:1 原版复刻

### 这个判断成立的前提

当前已经还原到足以支撑重写的主干包括：

- CLI 启动分流与命令树
- Session / Transcript / Resume / Fork 主模型
- settings/config source、merge、cache 与 key consumer 主边界
- 输入编译链
- 主循环与工具轮
- prompt discovery 与主线程/非主线程 prompt 主分层
- model/provider/auth/stream fallback/remote ingress 的主结构
- 工具执行、permission merge、hook 主骨架
- remote transcript persistence、bridge credential handoff 与本地主工作流
- plan / MCP / skill / plugin / TUI 中会影响接口边界的主链

### 需要避免的误读

- 这不等于“原版工程结构已经还原”。
- 这不等于“服务端黑箱行为已经完全知道”。
- 这也不等于“可以把工程实现选择写成还原结论”。
- 当前本地 bundle 只能证明客户端可见行为、接口载荷和本地状态机；服务端追加、灰度下发和 bundle 外 producer 必须保留为边界。

## 阻塞级未决项

按“能否开始高相似版本模块化重写”这个标准，目前没有继续阻塞开工的本地未决项。

仍然阻塞的是另一类目标：

| 目标 | 当前结论 | 原因 |
| --- | --- | --- |
| 原始源码文件树还原 | 不可承诺 | 当前证据来自 wrapper、native executable 的 `.bun` section 抽取、字符串和报告，不是原始仓库。 |
| 私有服务端实现还原 | 不可承诺 | 本地 bundle 只能看到请求、响应、header、token handoff、feature flag 和 telemetry 消费面，不能证明服务端内部策略。 |
| 1:1 原版复刻 | 不可承诺 | 仍有服务端黑箱、bundle 外 producer、feature flag/灰度路径和低频前台细节。 |

因此后续 rewrite 的开工标准应是：

- 主干行为先按当前可证职责边界实现。
- 未决点进入显式扩展位。
- 后期对齐用专题页继续收窄，而不是把未知点混成“不能开工”。

## 骨架收口状态

当前可施工骨架已经收束为三层约束：

- 模块合同：`cli / settings / engine / session / prompt / tools / hooks / agents / mcp / plugins / skills / remote / ui / state / shared` 均已在架构页给出职责、输入、输出和依赖方向。
- 接口主线：`PromptComposePipeline`、`ToolExecutionOutput.contextLayers`、`HookRuntimeInput`、`Extension Registry`、`Turn Loop State` 已能作为第一批重写接口。
- 弹性边界：服务端黑箱、bundle 外 producer、灰度或 dormant 路径统一由 adapter / registry / opaque field 承接，不进入确定性主链。

这组骨架已经可以作为高相似模块化重写的开工基线。候选 `src/` 目录只表示后续实现时的职责拆分建议，不是当前恢复文档要创建的代码树，也不代表原版源码文件树已经还原。

因此本页后续只维护会改变接口弹性的未决项；纯 UI 视觉细节、telemetry 边角、低频服务端字段和原始命名问题不再提高到架构阻塞级。

## 边界级未决项

这些问题不会阻止重写主干开工，但会直接影响接口设计和扩展位。

### 1. Request-level prompt 最终顺序与本地/服务端边界

本地 bundle 能证明：

- 主线程 request build 已按 `Hpc(...) -> Flm(...) -> Ak(...) -> Glm(...) -> Wlm(...)` 对象级顺序落到 `payload.system / payload.messages`。
- `systemPrompt selection`、`hS() / _H(...)`、`zL(...)`、skills、attachments、hook additional context、`deferred_tools_delta` 与 `currentDate` 的本地位置已经可分层说明。
- 非主线程 prompt 复用 `_3(...)`、`vk(...)`、`zN(...) / oRf(...)` 等骨架，并按 fork/compact/hook/agent 场景裁剪。
- `PromptComposePipeline` 可以按 `discoverInstructions -> buildUserContext -> buildSystemContext -> selectSystemPrompt -> compileTurnMessages -> normalizeMessages -> buildSystemBlocks -> buildRequestOptions -> sendViaTransport` 落成接口。

当前不能证明：

- `_H(...)` 预留注入槽位的原始用途；当前本地活路径只能证明 `cacheBreakerPhrase` 参与 memo key / `has_injection` telemetry，不能证明具体 producer 或返回字段。
- 服务端收到本地 payload 后是否还会额外拼装 context / compat / verification。

重写时应保留：

- `PromptComposePipeline` 的分阶段 hook：local instruction discovery、local user/system context、local system blocks、local messages、request options、transport metadata、server-side opaque extension。
- `systemContext` / request context 的可选扩展字段，不把当前 `_H(...)` 的可见字段当成永久全集。
- `payload.system`、`payload.messages` 与 schema/request options 三类输出要在类型上分开，不把 `tools / betas / context_management / output_config` 混写成 prompt 文本。

继续补证据应看：

- `02-execution/02-instruction-discovery-and-rules.md`
- `02-execution/03-prompt-assembly-and-context-layering/**`
- `02-execution/04-non-main-thread-prompt-paths/**`
- `01-runtime/06-stream-processing-and-remote-transport.md`

### 2. Hook 扩展面与少量边角时序

本地 bundle 能证明：

- hook schema、事件输入、`InstructionsLoaded` 触发、特殊输出、permission 结果、`Stop / SessionEnd / Worktree* / MessageDisplay / CwdChanged / FileChanged / Elicitation*` 等消费语义已经有当前路径。
- Hook 与 permission、tool executor、`PostToolBatch`、非主线程 prompt 的主时序已经足够实现。
- command hook 的 stdin / async 首行握手 / close 边界、并发执行与完成序消费已经能转成候选接口。
- `PreToolUse permissionDecision=defer`、`continueOnBlock`、`terminalSequence` 已收紧到各自的窄语义，不能再写成通用 permission mode 或 prompt 注入。
- `InstructionsLoaded` 多文件顺序在本地 bundle 内等于 `Qv()` discovery 数组顺序；当前没有看到额外本地排序器。

当前不能证明：

- bundle 外是否还有额外 hook 事件或输出分支
- bundle 外或服务端侧是否会追加 producer，改变 `InstructionsLoaded` 的外层 source 或 hook event registry

重写时应保留：

- 可扩展的 hook event registry。
- 输出通道不要只做成单一 string；至少保留 message、permission decision、input/output update、additional context、stop/prevent continuation、display-only、watch/worktree side-effect 等 typed channel。
- event callsite consumer 与 parser 分离；parser 可以识别更多字段，但调用方只消费该事件真正支持的 channel。

继续补证据应看：

- `02-execution/01-tools-hooks-and-permissions/02-hook-system/**`
- `02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md`
- `02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md`

### 3. `contextLayers` 的 bundle 外 producer

本地 bundle 能证明：

- 当前 `2.1.197` 活实现没有字面量 `contextModifier`；tool-returned 协议应按 `contextLayers / permissionLayers / NYt(...)` 理解。
- 当前直接确认的 tool-returned concrete producer 至少包括：
  - `SkillTool` inline：`allowed_tools / model / effort`
  - `EnterWorktree`：`working_directory`
- executor consumer、tool result 写回、`ToolUseContext` 克隆与 remote/bridge 直接缓存写入反证已经形成边界。
- fork skill / fork slash command 会把 layer 预置进子 agent `permissionLayers`，不是父线程 tool result 返回后再提交。

当前不能证明：

- 是否还存在 bundle 外、远端路径或未启用分支中的更多 producer。
- 历史上更泛化的函数式 `contextModifier` 是否在服务端或裁剪路径仍有兼容入口。

重写时应保留：

- `ToolExecutionOutput.contextLayers` 作为正式扩展字段。
- consumer 支持多个 producer 的合并或顺序应用；实现初期至少启用 `SkillTool` 与 `EnterWorktree`。
- `contextModifier` 若需要保留，只作为 adapter / 历史别名，不作为当前 executor 的核心字段名。

继续补证据应看：

- `02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md`
- `02-execution/06-context-runtime-and-tool-use-context.md`
- `03-ecosystem/05-skill-system.md`

### 4. bridge / worker 的服务端正式语义

本地 bundle 能证明：

- remote-control、`--remote`、`--sdk-url`、bridge credential handoff、remote transcript persistence、`409` 冲突还原和 `pending_action` 的本地只读边界。
- `environment_secret -> work secret -> session_ingress_token`、`worker_jwt` 刷新、`worker_epoch` 字段、Trusted Device gate 等本地消费面。

当前不能证明：

- `environment_secret` 的失效条件
- `worker_epoch` 的正式服务端含义
- `pending_action` 在远端 UI / 服务端协调层中的真实消费方式

重写时应保留：

- remote / bridge 状态对象中的 opaque credential fields。
- `worker_epoch`、`pending_action`、remote task state 的向后兼容字段。
- credential refresh、conflict recovery、server-side denial 的可插拔错误还原分支。

继续补证据应看：

- `01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md`
- `01-runtime/06-stream-processing-and-remote-transport.md`
- `03-ecosystem/02-remote-persistence-and-bridge.md`

### 5. MCP / plugin / skill 的动态注入边界（已收口为扩展槽）

本地 bundle 能证明：

- MCP config、project approval、policy gate、resources read、deferred tool、ToolSearch、MCP instructions delta、SDK control channel 的主链。
- plugin marketplace、install cache、runtime load、`enabledPlugins`、strict policy gate、channel prompt injection 与 builtin plugin 空 registry 边界。
- skill registry、`SKILL.md`、`dynamicSkills / conditionalSkills`、`skill_listing / invoked_skills / dynamic_skill`、`SkillTool` fork/inline 执行边界。

本地边界：

- `resources/list_changed` 会刷新本地 resource snapshot；`resources/updated` 与 `<mcp-resource-update>` / `<mcp-polling-update>` 只正证到协议 / parser / renderer，未见 CLI 产品主链 producer。
- builtin plugin registry 当前是 `hno = new Map()` 且未见 `hno.set(...)`，可见 entry set 为空。
- 当前 bundle 没有 `RC4 / discoveredSkillNames` 活字面命中；bundle 外 skill producer 不能写进确定性主链。

重写时应保留：

- 动态 tool/resource/skill/plugin registry。
- deferred tool schema 与 prompt injection 的分层接口。
- policy gate 在 settings、MCP、plugin、skill 注入前后的统一拦截点。

## 后期对齐项

这些问题主要影响高相似度细节，不应影响主架构开工。

| 项目 | 本地 bundle 已能证明 | 仍不能证明 | 保留扩展位 | 继续看 |
| --- | --- | --- | --- | --- |
| plan web editor / dirty-state | 本地 `ExitPlanMode.planWasEdited`、plan file guard、`set_permission_mode { mode:"plan", ultraplan:true }`、`ultraplanPendingChoice` 弱消费与 `needsAttention` 链已可见 | web 端 plan editor 的完整 dirty-state producer、`ultraplanPendingChoice` 专属确认器 | plan input source、remote plan approval adapter、pending choice renderer | `03-ecosystem/03-plan-system/**` |
| `Mi6("compact")` 与 compact 灰度路径 | compact 主管线、`FOo(...) / SHe(...)`、autocompact tracking 已可实现 | bundle 外或灰度路径覆盖面 | compact strategy registry、telemetry-only branch marker | `01-runtime/04-agent-loop-and-compaction/**` |
| GrowthBook / feature flag 上游语义 | 本地初始化、SSE、event logging、off-switch consumer 已可见 | 实验配置上游、server-side targeting 与真实 rollout 规则 | feature flag provider interface、local fallback | `01-runtime/09-api-lifecycle-and-telemetry.md`、`01-runtime/11-non-llm-network-paths.md` |
| TUI 低频显示策略 | approval renderer、message subtype、tool result renderer 主分派、SDK/headless/TUI subtype 分层、`toolUseResult` sidecar 与低频 renderer 共同形态可见 | fullscreen virtual-scroll / search geometry 的 1:1 坐标细节、bundle 外 UI 扩展、remote 路径是否二次裁剪 `toolUseResult` | message renderer registry、unknown subtype fallback、tool result registry default fallback | `03-ecosystem/07-tui-system/**` |
| WebFetch server-side wrapper | 当前 CLI `WebFetch` 本地实现和 server-side `web_fetch` 概念都可见 | 当前 CLI 是否有未命中 feature flag 切到 server-side wrapper | web tool backend adapter | `01-runtime/08-web-fetch-tool.md` |
| 低频 telemetry / cache / header | 主 telemetry、stream fallback、gateway classifier、prompt cache 主链可见 | 少量 header/cache/provider 组合边角 | request metadata bag、provider-specific options | `01-runtime/05-model-adapter-provider-and-auth.md`、`01-runtime/06-stream-processing-and-remote-transport.md` |

## 已降级为非阻塞项的内容

下面这些点可以继续记录，但不该再当成“是否能开工”的关键阻塞项：

- 早期 `option` 命中字符串的历史来源
- 少量低频 protocol-like `system` subtype / producer 的调用表与边缘可见性
- transcript 搜索 / 导出 / virtual scroll 的 dormant 支线原因
- `bridge-kick` 是否在上游曾有完整注册链

这些问题值得继续补证据，但它们已经不再决定“主逻辑是否足够重写”。

## 建议的工程动作

### 1. 以候选架构开工，而不是等待全部未知点归零

当前更稳的做法是：

- 用 [01-rewrite-architecture.md](./01-rewrite-architecture.md) 里的候选分层启动工程
- 先建立模块目录、接口文件、registry/adapter 空实现和主干 smoke path
- 把未决区实现成可收缩的扩展位
- 避免为了等待 1:1 证据而阻塞主干重建

在恢复文档维护阶段，上述动作只作为后续工程建议保留；没有明确新指令时，不在本文档任务中创建实现代码。

### 2. 在未决区主动保留弹性

建议保留扩展位的地方包括：

- prompt compose pipeline
- hook event / hook output runtime
- `ToolExecutionOutput.contextLayers`
- bridge / worker 状态对象与错误分支
- dynamic MCP/plugin/skill registry
- plan/ultraplan remote editor adapter
- TUI renderer fallback registry

### 3. 继续补证据时，回到专题页而不是在本页继续堆正文

后续补证据应回到各自主题页：

- prompt 与 request boundary：`02-execution/03*`
- hook / permission：`02-execution/01*`
- remote / bridge：`03-ecosystem/02`
- plan / ultraplan：`03-ecosystem/03*`
- MCP / skill / plugin：`03-ecosystem/04*`、`03-ecosystem/05*`、`03-ecosystem/06*`
- TUI：`03-ecosystem/07*`
- model/provider/auth：`01-runtime/05` 与 `01-runtime/06`

本页只保留：

- 当前能否开工的判断
- 阻塞级、边界级、后期对齐项的分级
- 下一步该去哪里补证据

## 一句话结论

当前发行版已经足以支撑高相似版本的模块化重写；剩余未知主要集中在服务端黑箱、bundle 外扩展面和少量后期对齐细节，应通过接口扩展位承接，而不是继续阻塞主干开工。

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
