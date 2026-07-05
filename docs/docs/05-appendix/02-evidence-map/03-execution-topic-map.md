# 证据索引：execution 主题

## 主题到证据的对应关系

### [../../02-execution/01-tools-hooks-and-permissions.md](../../02-execution/01-tools-hooks-and-permissions.md)

- 主要证据：导航页；负责工具执行内核、Hook 系统、permission 状态机、managed policy/sandbox/审批 backend 四个子专题与 Hook 拆分页入口
- 可信度：高
- 用途：把执行层的工具、Hook、权限三条线收束成稳定阅读顺序，不单独承载直接证据

### [../../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md](../../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md)

- 主要证据：`$Yt / nTf / oTf / KHe` 主执行链、deferred tools / `ToolSearch`、`tool_result` 两层形态、配对修复与落盘、`AskUserQuestion` 特例、Perforce mode、Git Bash/PowerShell selection、`CLAUDE_CODE_USE_POWERSHELL_TOOL`、embedded `bfs` / `ugrep`、background shell/orphan handling
- 可信度：高
- 后续补证据重点：`tool_reference` 到真实 tool schema 注入的最后桥接层

### [../../02-execution/01-tools-hooks-and-permissions/02-hook-system.md](../../02-execution/01-tools-hooks-and-permissions/02-hook-system.md)

- 主要证据：导航页；负责 hook schema、特殊输出分支、运行时时序、`InstructionsLoaded` 在非主线程的覆盖面四个拆分页入口
- 可信度：高
- 用途：把 Hook 系统从执行器正文里拆开，避免把父页误当作单一事件/函数的直接证据

### [../../02-execution/01-tools-hooks-and-permissions/02-hook-system/01-schema-instructionsloaded-and-event-inputs.md](../../02-execution/01-tools-hooks-and-permissions/02-hook-system/01-schema-instructionsloaded-and-event-inputs.md)

- 主要证据：hook schema、`InstructionsLoaded` 触发链、事件输入结构与参数边界
- 可信度：高
- 后续补证据重点：少量低频 event input 的字段覆盖面

### [../../02-execution/01-tools-hooks-and-permissions/02-hook-system/02-special-output-and-event-consumer-semantics.md](../../02-execution/01-tools-hooks-and-permissions/02-hook-system/02-special-output-and-event-consumer-semantics.md)

- 主要证据：`hookSpecificOutput` 分支、typed output channel、`Stop / SessionEnd / Worktree* / MessageDisplay / CwdChanged / FileChanged / Elicitation*` 等特殊事件的消费语义与保留行为、`terminalSequence` allowlist、`continueOnBlock` 阻断语义、`PreToolUse permissionDecision=defer` 与 `PermissionDenied` 复核边界
- 可信度：高
- 后续补证据重点：bundle 外是否还会追加新的特殊输出 producer

### [../../02-execution/01-tools-hooks-and-permissions/02-hook-system/03-runtime-order-and-cross-stage-timing.md](../../02-execution/01-tools-hooks-and-permissions/02-hook-system/03-runtime-order-and-cross-stage-timing.md)

- 主要证据：Hook 与 permission 相关阶段的顺序图、跨阶段 timing 与 merge 点、matcher 稳定顺序、hook 并发执行、结果完成序消费、command hook stdin / async / close 完成边界
- 可信度：高
- 后续补证据重点：bundle 外 producer 是否改变外层调度条件

### [../../02-execution/01-tools-hooks-and-permissions/02-hook-system/04-instructionsloaded-non-main-thread-coverage-and-dispatch-boundaries.md](../../02-execution/01-tools-hooks-and-permissions/02-hook-system/04-instructionsloaded-non-main-thread-coverage-and-dispatch-boundaries.md)

- 主要证据：`InstructionsLoaded` 在非主线程 prompt、fork-family、compact/hook 专用路径里的覆盖面与派发边界，`Qv()` discovery 数组决定的多文件本地 dispatch 顺序
- 可信度：高
- 后续补证据重点：bundle 外 source / producer 是否会改写 discovery 顺序

### [../../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md](../../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md)

- 主要证据：`D0z / YP` permission core、`za / Hy6 / oy6 / LqA / Qs6 / SV / IqA`、dangerous allow rules 剥离与还原、auto classifier 输入与失败语义、`Tool(param:value)` permission rule 的参数匹配合同
- 可信度：高
- 后续补证据重点：classifier 在更极端 tool schema 下的判定残差

### [../../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md](../../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md)

- 主要证据：`allowManagedPermissionRulesOnly`、`allowManagedDomainsOnly`、`allowManagedReadPathsOnly`、`RG8(...)` sandbox 合流、`apply-seccomp` helper、`--permission-prompt-tool`、`toolUseConfirmQueue`、remote/direct/ssh、headless/SDK/bridge 的 ask backend
- 可信度：高
- 后续补证据重点：`orphaned-permission` 与少量 approval backend 变体边界

### [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)

- 主要证据：导航页；负责 instruction source、system section cache、InstructionsLoaded load reason、userContext/systemContext 四个拆分页入口
- 可信度：高
- 用途：把 instruction discovery 与 rules 主题拆成稳定阅读顺序，不单独承载直接证据

### [../../02-execution/02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md](../../02-execution/02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md)

- 主要证据：`Qv()` 扫描链、`xy()`、`v16()`、`JI6()`、`HQ9() / aC1()`、instruction source 枚举、compat 与 `/init` 的边界
- 可信度：高
- 后续补证据重点：bundle 外 compat 输入是否还有其他迁移入口

### [../../02-execution/02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md](../../02-execution/02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md)

- 主要证据：`Lq8 / Vc8 / Ec8 / kHq / Yn`、`zL / fT8 / R8_ / S8_ / C8_ / d8_ / i8_ / h8_ / c8_`、section cache 粒度、默认 section 名单与失效边界
- 可信度：高
- 后续补证据重点：少量 section 在灰度配置下的非空条件

### [../../02-execution/02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md](../../02-execution/02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md)

- 主要证据：`O4t / Tpp / F4t / X5e`、`InstructionsLoaded` load reason 的一次性消费、`compact/session_start/include` 时序边界
- 可信度：高
- 后续补证据重点：非主线程路径是否会通过额外入口预置 load reason

### [../../02-execution/02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md](../../02-execution/02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md)

- 主要证据：`hS() / _H(...) / dlo()`、`userEmail / attachedProject / perforceMode` 的 request context 字段边界、`cachedClaudeMdContent`、`claudeMdExcludes`、`currentDate` 的 `qla(GSe()) / rdp() / odp() / Wla()` 标记链、domain list 147 项与 keyword list 11 项的用途边界、Claude API skill provider skip markers、`TRa(...) / vRa(...)` code indexing tool telemetry 归一化名单、同类 prompt 隐写载体的负面排查
- 可信度：高
- 后续补证据重点：`_H(...)` 预留第二注入槽位的原始用途；当前 `cacheBreakerPhrase` 只正证到 memo key / `has_injection` telemetry，不正证到返回字段

### [../../02-execution/03-prompt-assembly-and-context-layering.md](../../02-execution/03-prompt-assembly-and-context-layering.md)

- 主要证据：导航页；负责主线程 prompt 的 system 链、attachment 顺序、payload/cache、request-level 注入四个拆分页入口
- 可信度：中高
- 用途：把主线程 prompt 装配拆成稳定阅读顺序，不单独承载直接证据

### [../../02-execution/03-prompt-assembly-and-context-layering/01-system-chain-default-sections-and-context-sources.md](../../02-execution/03-prompt-assembly-and-context-layering/01-system-chain-default-sections-and-context-sources.md)

- 主要证据：`systemPrompt selection`、`CDl(...)`、`_H(...)`、`O4t("compact")`、`zL(...)`、skills 在 prompt 里的最小落点、`deferred_tools_delta` 与真实 tool schema 注入边界、`currentDate` 属于 `userContext -> otr(...) -> messages[]` 而不是 `payload.system` 的边界
- 可信度：高
- 后续补证据重点：`_H(...)` 第二注入槽位的原始用途

### [../../02-execution/03-prompt-assembly-and-context-layering/02-attachment-order-skill-plan-meta-and-message-merge.md](../../02-execution/03-prompt-assembly-and-context-layering/02-attachment-order-skill-plan-meta-and-message-merge.md)

- 主要证据：`invoked_skills` 在 compact/resume 里的顺序、当前轮 attachment 生成顺序、`skill_listing / plan_mode / critical_system_reminder` 位置、`claudeMd` 前缀链与 compat 当前边界
- 可信度：中高
- 后续补证据重点：少量低频 attachment 在 merge 前后的排序差异

### [../../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md](../../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md)

- 主要证据：`Hpc(...) -> Flm(...) -> Ak(...) -> Glm(...) -> Wlm(...)` 对象级顺序、cache breakpoint、`Glm(...)` 不再消费 `cache_reference / cache_edits` 的边界、system prompt 分块缓存、最终 `payload.system / payload.messages` 边界
- 可信度：高
- 后续补证据重点：prompt cache 在更多 provider/transport 组合里的剩余差异

### [../../02-execution/03-prompt-assembly-and-context-layering/04-request-level-injection-layers-and-local-server-boundary.md](../../02-execution/03-prompt-assembly-and-context-layering/04-request-level-injection-layers-and-local-server-boundary.md)

- 主要证据：`prompt-text`、`schema/request options`、`transport` 三层注入拆分，以及 `Hpc(...)` 完成 payload 后本地仍可证明的边界
- 可信度：中高
- 后续补证据重点：服务端是否还会追加黑箱 context

### [../../04-rewrite/01-rewrite-architecture.md](../../04-rewrite/01-rewrite-architecture.md)

- 主要证据：把 prompt / request 证据收束成 `PromptComposePipeline` 阶段表，标出 instruction discovery、user/system context、system prompt selection、message normalize、system blocks、request options 与 transport adapter 的输入输出和扩展位；把 Hook P1 证据收束成 `HookRuntimeInput -> resolveHookDefinitions -> matchHooks -> executeHooksConcurrent -> parseTypedOutput -> consumeByEventCallsite` 的候选接口
- 可信度：中高
- 用途：作为高相似模块化重写里 `prompt/compose` 与 `hooks/runtime` 的接口边界，不作为原始源码文件树还原证据

### [../../02-execution/04-non-main-thread-prompt-paths.md](../../02-execution/04-non-main-thread-prompt-paths.md)

- 主要证据：导航页；负责共享 merge 骨架、hook/compact 专用路径、fork-family snapshot 复用、compat/agent definitions 四个拆分页入口
- 可信度：中高
- 用途：把非主线程 prompt 相关证据拆成稳定子专题，不单独承载直接证据

### [../../02-execution/04-non-main-thread-prompt-paths/01-shared-merge-skeleton-and-overrides.md](../../02-execution/04-non-main-thread-prompt-paths/01-shared-merge-skeleton-and-overrides.md)

- 主要证据：非主线程三分类里的前两类共享 merge 骨架、`_3(...)`、`zN(...) / oRf(...)`、`omitClaudeMd`、`Explore/Plan` 裁剪、`SubagentStart` hook 注入位置
- 可信度：高
- 后续补证据重点：更多模式裁剪组合的边界

### [../../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md](../../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md)

- 主要证据：`hook_prompt`、`hook_agent`、verification 残留资产、compact summarize 旁路、shared-prefix/fallback 分支
- 可信度：中高
- 后续补证据重点：verification 家族在更完整 build 中是否仍有活 wiring

### [../../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md](../../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md)

- 主要证据：`vk(...)`、`cacheSafeParams`、`ML / Cj4 / xe6 / I1z` 生命周期、fork-family 的 snapshot producer、fresh-build 与 reuse 边界
- 可信度：高
- 后续补证据重点：reuse 失败时的回退条件与 cache invalidation 细节

### [../../02-execution/04-non-main-thread-prompt-paths/04-compat-agent-definitions-and-instruction-entry.md](../../02-execution/04-non-main-thread-prompt-paths/04-compat-agent-definitions-and-instruction-entry.md)

- 主要证据：compat 在非主线程里的三种 carrier、verification 本地反证闭环、agent definitions 来源与 winner 规则、祖先目录/worktree/external include
- 可信度：中高
- 后续补证据重点：agent definitions 在更多 host/runtime 变体下的来源优先级

### [../../02-execution/05-attachments-and-context-modifiers.md](../../02-execution/05-attachments-and-context-modifiers.md)

- 主要证据：导航页；负责 attachment 专题的子页入口与主题边界，不单独承载直接证据
- 可信度：高
- 用途：把 producer/lifecycle、payload/materialize、`contextLayers` consumer 三条正文线组织到同一目录

### [../../02-execution/05-attachments-and-context-modifiers/01-attachment-lifecycle-and-producers.md](../../02-execution/05-attachments-and-context-modifiers/01-attachment-lifecycle-and-producers.md)

- 主要证据：`JPl(...)` producer 矩阵、`KE6(...) / Nq(...)` attachment 包装、input compilation 的 loading gate、compact keep attachment 顺序、`Su_(...)` resume restore、`AVq(...) / mN8(...)` plan 还原分流
- 可信度：中高
- 后续补证据重点：少量 compact keep 边缘类型、UI 隐藏 attachment 与 transcript 渲染的更细边界

### [../../02-execution/05-attachments-and-context-modifiers/02-high-value-attachment-payloads-and-materialization.md](../../02-execution/05-attachments-and-context-modifiers/02-high-value-attachment-payloads-and-materialization.md)

- 主要证据：`dt1(...)` 的 prompt 归一化/丢弃规则、`queued_command` / `plan_file_reference` / `invoked_skills` / `relevant_memories` / `mcp_resource` / `task_status` / `async_hook_response` / `hook_additional_context` / usage attachments 的具体 materialize 逻辑
- 可信度：中高
- 后续补证据重点：低频 attachment payload 的完整字段族；`hook_*` 边缘类型在非主线程 prompt 中的精确覆盖面

### [../../02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md](../../02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md)

- 主要证据：`contextLayers / permissionLayers / NYt(...)` 的运行态接口，`SkillTool` 与 `EnterWorktree` concrete producer，执行器串行/并发 consumer、remote/bridge 直接缓存写入反证
- 可信度：中高
- 后续补证据重点：bundle 外/服务端侧是否存在更多 concrete producer，或是否仍有函数式 `contextModifier` 兼容入口

### [../../02-execution/06-context-runtime-and-tool-use-context.md](../../02-execution/06-context-runtime-and-tool-use-context.md)

- 主要证据：`ts6(...)` 的 `ToolUseContext` 克隆规则、`_3(...)` 的 request/tool context 裁剪、`Cx / Zjq / Ly6 / av6` 的 `readFileState` 基线重建、`contextLayers / permissionLayers` 的运行态边界、`dynamicSkillDirTriggers` 与 skill discovery trigger 的当前边界
- 可信度：中高
- 后续补证据重点：`_H(...)` 预留第二注入槽位的原始用途、远端服务端是否额外叠加 `systemContext`、远端是否存在本地 bundle 未暴露的额外 `contextLayers` producer

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
