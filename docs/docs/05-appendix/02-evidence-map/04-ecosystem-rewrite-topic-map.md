# 证据索引：ecosystem 与 rewrite 主题

## 主题到证据的对应关系

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents.md)

- 主要证据：`03-ecosystem` 这一组主题的导航性总结
- 可信度：高
- 用途：把 Resume/Fork/Sidechain、Agent Team 模型、mailbox 协议、teammate runtime 分页组织

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md)

- 主要证据：`--resume / --fork-session / --resume-session-at / --reply-on-resume` CLI 分支、`tfe(...)` transcript load、`omr(...)` resume/fork processing、`subagents/agent-<id>.jsonl` 路径、`forkContextMessages`、`zN(...) / oRf(...)` 与 `_3(...)` 的当前执行骨架
- 可信度：高
- 后续补证据重点：subagent 边缘流程、CCR v2 hydrate 与 sidechain persistence 的低频冲突处理

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md)

- 主要证据：`TeamCreate / TeamDelete`、`AgentInput` teammate 分支、team `config.json`、`teamAllowedPaths`、`hiddenPaneIds`、TaskList 的 `pending / in_progress / completed / deleted` 状态和 owner/blockedBy 读写链
- 可信度：中高
- 后续补证据重点：team roster 字段全集、task list 更细状态机

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md)

- 主要证据：`TeammateMailbox` 读写日志、`inboxes/<agent>.json` 存储路径、`permission_request / permission_response`、`sandbox_permission_request / sandbox_permission_response`、`mode_set_request`、`plan_approval_request / plan_approval_response`、`idle_notification`、`shutdown_request` schema 与 permission-laundering 防护文案
- 可信度：中高
- 后续补证据重点：team auto-approve 边界、mailbox message schema 全量枚举

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md)

- 主要证据：导航页；负责 teammate runtime、backend 分流、idle wait、task claim、pane CLI 回接与 shutdown/hide-show 四条正文线的阅读顺序
- 可信度：中高
- 用途：把 teammate runtime 与任务调度主题拆成稳定阅读顺序，不单独承载直接证据

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md)

- 主要证据：`in_process_teammate` task 的真实 worker 与 pane leader task 壳两种形态、`Du()` teammate mode snapshot 判定、`BA6()` tmux/iTerm2 backend 检测矩阵、`wQ4()` 与 `wu_() / $u_()` spawn 分流、pane 子进程的 `--teammate-mode` / `--agent-id` / `--agent-name` / `--team-name` / `--parent-session-id` 注入
- 可信度：中高
- 后续补证据重点：pane backend 检测失败时的用户可见错误文案与 fallback telemetry

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md)

- 主要证据：`obf(...)` in-process agent loop、`rbf(...) / $j_()` idle/mailbox poll loop、固定 500ms tick、`Yj_()` 第一个 eligible pending task 选择、`rp1()` per-task lock claim、`dIq()` status 写入、`F$_()` 保守 claim 变体、`xA6(...)` owner 回收
- 可信度：中高
- 后续补证据重点：`F$_()` 是否出现新活调用点、`QD(...)` 文件系统顺序在不同平台上的稳定性

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md)

- 主要证据：pane / in-process 协作语义同构边界、`setDynamicTeamContext(...)`、`GU4(...)`、`WU4(...)`、`b5A(...)` startup 回接、pane 子 CLI 回到 `repl_main_thread -> zN(...) -> oRf(...)`、mailbox-first shutdown、pane kill bridge、tmux hide/show backend 与空壳上层函数
- 可信度：中高
- 后续补证据重点：pane 子 CLI worker loop 的直接分支证据、hide/show 执行链是否在后续版本恢复

### [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md)

- 主要证据：teammate runtime 的 spawn、worker turn、task claim、shutdown 总模型，以及 pane worker loop 复用和 `F$_()` 活调用点的保守结论
- 可信度：中高
- 后续补证据重点：用后续动态分析或新版本 bundle 复核 pane worker loop 与 `F$_()` 的接入状态

### [../../03-ecosystem/02-remote-persistence-and-bridge.md](../../03-ecosystem/02-remote-persistence-and-bridge.md)

- 主要证据：remote-control / `--remote` / `--sdk-url` 本地工作流矩阵、bridge credential handoff、remote transcript persistence、`409` 冲突还原、`bridge-kick` fault model 与其失联 handle 负证、`pending_action` 的只读不还原边界、VS Code、JetBrains、Windsurf / Devin Desktop、Remote Control、`CLAUDE_CLIENT_PRESENCE_FILE` marker 与 session presence pulse 的 host surface
- 可信度：高
- 后续补证据重点：服务端语义边界（`environment_secret` TTL / `worker_epoch` 正式定义），而不是本地工作流主干

### [../../03-ecosystem/03-plan-system.md](../../03-ecosystem/03-plan-system.md)

- 主要证据：导航页；负责 plan file 生命周期、enter/exit 与 `/plan`、退出审批 UI、attachments/持久化/team approval 四个拆分页入口
- 可信度：中高
- 用途：把 plan 主题拆成稳定阅读顺序，不单独承载直接证据

### [../../03-ecosystem/03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md](../../03-ecosystem/03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md)

- 主要证据：`prePlanMode / hasExitedPlanMode / needsPlanModeExitAttachment` 全局状态、`plansDirectory` project-root 边界、`plan_file_reference` attachment、plan file 的路径命名、resume/fork 的 plan 文件复制与还原链
- 可信度：中高
- 后续补证据重点：`plansDirectory` 越界保护的更多边界条件、subagent plan file 在更多还原分支里的命中覆盖面

### [../../03-ecosystem/03-plan-system/02-enter-exit-and-plan-command.md](../../03-ecosystem/03-plan-system/02-enter-exit-and-plan-command.md)

- 主要证据：`EnterPlanMode / ExitPlanMode` 工具、`prePlanMode` 还原分支、`--plan-mode-instructions`、`/plan` local-jsx 子命令、external editor 打开链与 `plan_mode_reentry / plan_mode_exit` bookkeeping
- 可信度：中高
- 后续补证据重点：`/plan` 各低频子命令与异常分支的输出细节、外部编辑器失败时的产品级降级提示

### [../../03-ecosystem/03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md](../../03-ecosystem/03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md)

- 主要证据：`ExitPlanMode` 的 `planWasEdited` 参数、plan file empty/missing guard、`set_permission_mode { mode:"plan", ultraplan:true }` 远端注入、`ultraplanSessionUrl / ultraplanPendingChoice / ultraplanLaunchPending` app state、`ultraplan-choice / ultraplan-launch` UI screen 与 `needsAttention` 链
- 可信度：中高
- 后续补证据重点：web 端 plan editor 的完整 dirty-state producer、`ultraplanPendingChoice` 的最终专属落地器

### [../../03-ecosystem/03-plan-system/04-attachments-persistence-and-team-approval.md](../../03-ecosystem/03-plan-system/04-attachments-persistence-and-team-approval.md)

- 主要证据：`plan_mode / plan_mode_reentry / plan_mode_exit / plan_file_reference` 的 producer 与 payload、compact/resume keep 链、teammate `plan_approval_request` mailbox 协议、remote ultraplan 的本地还原边界
- 可信度：中高
- 后续补证据重点：team approval 在更多 backend 变体下的分支差异、少量低频 attachment 的 transcript 可见性边界

### [../../03-ecosystem/04-mcp-system.md](../../03-ecosystem/04-mcp-system.md)

- 主要证据：`--mcp-config / --strict-mcp-config` CLI、`.mcp.json` 读写与 project approval 文案、`allowedMcpServers / deniedMcpServers` policy gate、`MCP-Protocol-Version` transport header、`resources/list / resources/read`、`notifications/claude/channel`、`ToolSearchTool / tool_reference / defer_loading / deferred_tools_delta`、SDK control channel 的 `mcp_call / mcp_reconnect`、`MCP_TOOL_TIMEOUT` 与 `${CLAUDE_PROJECT_DIR}` 替换边界
- 可信度：中高
- 边界：`resources/list_changed` 是本地 runtime snapshot refresh；`resources/updated` 与 `<mcp-resource-update>` / `<mcp-polling-update>` 当前只正证到 protocol / parser / renderer，未见 CLI 产品主链 producer；服务端侧额外 MCP prompt 拼装仍按黑箱处理

### [../../03-ecosystem/05-skill-system.md](../../03-ecosystem/05-skill-system.md)

- 主要证据：`SKILL.md` loader/cache 路径、`skill_listing / invoked_skills / dynamic_skill` attachment、`SkillTool` fork/inline 执行日志、`disableModelInvocation` 过滤、`commands_DEPRECATED` 兼容源、plugin/MCP skill builder 注册提示、`dynamicSkills / conditionalSkills` 运行态注册边界
- 可信度：中高
- 边界：当前 bundle 没有 `RC4 / discoveredSkillNames` 活字面命中；bundle 外 skill producer 只能作为 registry adapter 扩展位

### [../../03-ecosystem/06-plugin-system.md](../../03-ecosystem/06-plugin-system.md)

- 主要证据：导航页；负责 plugin 命令状态、marketplace/manifest/cache、runtime injection、validate/UI/safety 四个拆分页入口
- 可信度：中高
- 用途：把 plugin 子系统拆成稳定阅读顺序，不单独承载直接证据

### [../../03-ecosystem/06-plugin-system/01-command-state-and-source-resolution.md](../../03-ecosystem/06-plugin-system/01-command-state-and-source-resolution.md)

- 主要证据：`plugin validate/list/install/uninstall/enable/disable/update` 与 marketplace 子命令文案、`known_marketplaces.json / installed_plugins.json / enabledPlugins` 三层状态、runtime entry、source precedence
- 可信度：中高
- 边界：builtin plugin 机制消费链为 `hno -> yno/zXi/VXi`；当前 readable bundle 中 `hno` 初始化为空且未见 `set` producer，可见 builtin plugin entry set 为空

### [../../03-ecosystem/06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md](../../03-ecosystem/06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md)

- 主要证据：marketplace schema、plugin manifest schema、manifest 路径与兼容模式、versioned cache、`CLAUDE_CODE_SYNC_PLUGIN_INSTALL` cache/full install 分流、source 类型与 dependency resolution
- 可信度：中高
- 后续补证据重点：少量 source 类型在真实 marketplace 中的边缘字段

### [../../03-ecosystem/06-plugin-system/03-runtime-injection-settings-and-channels.md](../../03-ecosystem/06-plugin-system/03-runtime-injection-settings-and-channels.md)

- 主要证据：`strictKnownMarketplaces / blockedMarketplaces / disableSideloadFlags` policy gate、`--plugin-dir / --plugin-dir-no-mcp / --plugin-url` session overlay、plugin MCP `skipMcpDiscovery` 边界、skills/MCP/settings/channels/userConfig 注入
- 可信度：中高
- 边界：channel notification 已能闭环到 queued prompt；builtin plugin 的 `hno -> yno/zXi/VXi` consumer 侧完整，但当前 readable bundle 中 `hno` entry set 可见为空

### [../../03-ecosystem/06-plugin-system/04-validation-ui-and-safety-boundaries.md](../../03-ecosystem/06-plugin-system/04-validation-ui-and-safety-boundaries.md)

- 主要证据：`plugin validate`、marketplace manifest 校验、目录附属内容校验、`/plugin` 管理 UI 状态机、lock/downrank/telemetry 与下架处理
- 可信度：中高
- 后续补证据重点：`pip` 非对称支持之外的低频 validate 分支

### [../../03-ecosystem/07-tui-system.md](../../03-ecosystem/07-tui-system.md)

- 主要证据：导航页；负责 TUI 主题的子页入口、阅读顺序与跨专题边界，不单独承载直接证据
- 可信度：高
- 用途：把 root/render、transcript、input、dialogs、local JSX、permission dispatch、message renderer、tool-result renderer 八条正文线组织到同一目录

### [../../03-ecosystem/07-tui-system/01-repl-root-and-render-pipeline.md](../../03-ecosystem/07-tui-system/01-repl-root-and-render-pipeline.md)

- 主要证据：`f3A(...)` 主 REPL root、`screen === "transcript"` 的独立分支、`Obz(scrollable/bottom/overlay/modal)`、`Aj6/LOz` 的消息区标准化与 windowing、`TOz/scrollRef/renderRange` 的 dormant virtual-scroll 支线、`/tui default|fullscreen`、fullscreen/no-flicker、`axScreenReader`、scroll/footer badge、`/focus` 与主 REPL 中 `rL = false`
- 可信度：中高
- 后续补证据重点：brief/bash/collapsed read search 的更细视觉策略，以及 `rL` 被硬关闭的更上游原因

### [../../03-ecosystem/07-tui-system/02-transcript-and-rewind.md](../../03-ecosystem/07-tui-system/02-transcript-and-rewind.md)

- 主要证据：transcript 分支里的 `/ / n/N / q / v` handler、`Lx4()` 对 renderer search API 的桥接、renderer `scanElementSubtree(...)` 的离屏扫描、以及当前 `rL = false` 导致这组 handler 未接活；`r4A(...)` rewind/restore/summarize、`fVq(...)` summarize-from-here、`/clear` 后 `open_message_selector -> parentSessionId -> rewind_pre_clear` 恢复链
- 可信度：中高
- 后续补证据重点：renderer 搜索命中的几何结构、`rL` 的 build-time 来源、`message-selector` 下游 restore/fork 对 session 状态的更细粒度影响

### [../../03-ecosystem/07-tui-system/03-input-footer-and-voice.md](../../03-ecosystem/07-tui-system/03-input-footer-and-voice.md)

- 主要证据：`PF4/oLz(...)` 输入枢纽、`chat:*` / `attachments:*` / `footer:*` / `help:dismiss` 快捷键、stash/external-editor/image-paste 路径、`pCz(...)`/`uc4(...)` voice 锚点与 push-to-talk 处理
- 可信度：中高
- 后续补证据重点：history search 面板的几何/候选细节

### [../../03-ecosystem/07-tui-system/04-dialogs-and-approvals.md](../../03-ecosystem/07-tui-system/04-dialogs-and-approvals.md)

- 主要证据：`UA8()` 的 `focusedInputDialog` 优先级链、`IX()` 的分支取消逻辑、tool-permission overlay、sandbox/worker/elicitation/dialog/callout 组件族
- 可信度：中高
- 后续补证据重点：`ide-onboarding/plugin-hint/desktop-upsell` 细节，以及非审批类 callout 的完整文案边界

### [../../03-ecosystem/07-tui-system/05-help-settings-model-theme-and-diff.md](../../03-ecosystem/07-tui-system/05-help-settings-model-theme-and-diff.md)

- 主要证据：`WW4(...)` help overlay、`wL6(...)` ThemePicker、`x26(...)` ModelPicker、`BD4(...)` settings registry、`$4z(...)` DiffDialog，以及 `config/settings`、`theme`、`help`、`diff` local JSX 注册
- 可信度：中高
- 后续补证据重点：`Status` 具体字段来源、`Usage` 返回 schema、更多 settings 类 local JSX，以及早期 `option` 命中字符串的原始来源

### [../../03-ecosystem/07-tui-system/06-tool-permission-dispatch.md](../../03-ecosystem/07-tui-system/06-tool-permission-dispatch.md)

- 主要证据：tool-specific dialog / descriptor registry、file/edit/write/notebook/path-aware/command/fetch/plan/skill/AskUserQuestion 等审批组件、`IQ/HF8` 两类审批壳，以及 generic fallback 审批器
- 可信度：中高
- 边界：`hVz/SVz/bVz/xVz/RVz/CVz/IVz/uVz` 在当前 2.1.197 bundle 内没有活字面命中，只能作为旧恢复口径或历史混淆名边界；审批主链本身已不依赖这组旧名

### [../../03-ecosystem/07-tui-system/07-message-row-and-subtype-renderers.md](../../03-ecosystem/07-tui-system/07-message-row-and-subtype-renderers.md)

- 主要证据：`k8f(...)`、`o6l -> gVf(...) -> UQ/Xdf(...)`、`Jdf(...)`、`Qdf(...)`、`Hpl(...)`、`mpl(...)`、`cpl(...)`、lookup/search-text 提取链，`cj6(...)` / remote adapter 对 protocol-like system event 的放行边界，以及 SDK/headless schema 中 `model_consent_fallback / notification / thinking_tokens / permission_denied / local_command_output` 等非 TUI-row subtype 的分层边界
- 可信度：中高
- 后续补证据重点：是否还有其他工具实现 `renderGroupedToolUse(...)`；system/info 可见性主干与 SDK/headless/TUI 分层已基本闭环

### [../../03-ecosystem/07-tui-system/08-tool-result-renderers.md](../../03-ecosystem/07-tui-system/08-tool-result-renderers.md)

- 主要证据：`zpl / Fpl / a7n / Lpl / xpl`、`toolUseResult` sidecar 写回链、`lookup` 构建、`H2q/lv8/lv6` 的结果落盘预览、`Read/Bash/Edit/Write/NotebookEdit/WebFetch/WebSearch/MCP/agent` 的 `renderToolResultMessage(...)`，以及 `Fpl(...)` 没有全局 JSON fallback、低频 generic fallback 应作为重写 registry 默认项的边界
- 可信度：中高
- 后续补证据重点：bundle 外/remote 路径是否会再裁剪 `toolUseResult`；本地低频 renderer 已可按 tool-specific renderer + registry default fallback 保守实现

### [../../04-rewrite/01-rewrite-architecture.md](../../04-rewrite/01-rewrite-architecture.md)

- 主要证据：前面所有主题的归纳结果，不依赖单一函数；当前按“主干可直接实现 / 主链可实现但留扩展位 / 工程实现选择”三档约束候选架构，并把 `cli / settings / engine / session / prompt / tools / hooks / agents / mcp / plugins / skills / remote / ui / state / shared` 压成模块合同矩阵，明确职责、输入、输出、依赖方向，以及服务端黑箱、bundle 外 producer、灰度路径的 adapter / registry / opaque field 承接方式
- 可信度：中高
- 用途：作为高相似模块化重写的施工骨架页，固定可直接继承的职责边界、接口还原目标、extension registry、依赖方向、落地顺序与骨架维护约束；候选 `src/` 目录只表示后续实现建议，不承诺原始源码文件树

### [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)

- 主要证据：现有所有结论的汇总，以及尚未被 A 级证据完全钉死的部分；当前已分成阻塞级、边界级和后期对齐项，并说明模块合同、接口主线与弹性边界已经足以支撑开工
- 可信度：高
- 用途：决定“现在是否可以开工重写”，并集中维护服务端黑箱、bundle 外扩展面、灰度路径和后期对齐项的补证入口；当前骨架已作为开工基线收口，后续只把会改变接口弹性的未知项提升到本页

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
