# Claude Code CLI 重建知识库

## 使用方式

- 目录化版本：`docs/`
- 维护原则：优先维护拆分后的文档；补证据、纠错和重写规划都在这里持续推进

## 状态边界

- 本页是知识库总入口，按当前 `docs/` 目录、主题矩阵和 evidence map 维护导航关系。
- 本页不承载当前 bundle 直接证据；新增运行时结论应进入对应主题页，并在 [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md) 登记。
- 当前发行版入口、证据口径和置信度定义以 [01-scope-and-evidence.md](./01-scope-and-evidence.md) 为准。

## 入口导航

### 00-overview

- [01-scope-and-evidence.md](./01-scope-and-evidence.md)
  - 文档边界、当前结论、证据来源、置信度定义
- [02-document-style-and-structure-conventions.md](./02-document-style-and-structure-conventions.md)
  - 文档风格、父页/子页职责、拆分与迁移约定

### 01-runtime

- [../01-runtime/01-product-cli-and-modes.md](../01-runtime/01-product-cli-and-modes.md)
  - 产品形态、入口分流、命令树、运行模式
- [../01-runtime/02-session-and-persistence.md](../01-runtime/02-session-and-persistence.md)
  - 全局状态、Session、Transcript、持久化策略
- [../01-runtime/03-input-compilation.md](../01-runtime/03-input-compilation.md)
  - 输入本地编译、block/slash command 与 `UserPromptSubmit` hook 的总览与拆分导航
- [../01-runtime/03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md](../01-runtime/03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md)
  - `AU8(...)`、`ihz(...)`、输入归一化与三条主分流
- [../01-runtime/03-input-compilation/02-block-family-bu4-and-slash-command-results.md](../01-runtime/03-input-compilation/02-block-family-bu4-and-slash-command-results.md)
  - 输入 block family、`BU4(...)` 与 slash command 编译结果
- [../01-runtime/03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md](../01-runtime/03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md)
  - `UserPromptSubmit` hook 位置、输出合并、截断、去重与阻断
- [../01-runtime/04-agent-loop-and-compaction.md](../01-runtime/04-agent-loop-and-compaction.md)
  - 主循环与 compact 主题的总览与拆分导航
- [../01-runtime/04-agent-loop-and-compaction/01-main-loop-state-caches-and-yield-surface.md](../01-runtime/04-agent-loop-and-compaction/01-main-loop-state-caches-and-yield-surface.md)
  - 主循环骨架、运行态缓存与对外产出面
- [../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md](../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md)
  - compact 管线、触发来源与跟踪状态
- [../01-runtime/04-agent-loop-and-compaction/03-no-tool-branch-recovery-stop-and-reactive-compact.md](../01-runtime/04-agent-loop-and-compaction/03-no-tool-branch-recovery-stop-and-reactive-compact.md)
  - 无工具分支、reactive compact 与 stop 相关收尾
- [../01-runtime/04-agent-loop-and-compaction/04-tool-round-next-turn-and-terminal-reasons.md](../01-runtime/04-agent-loop-and-compaction/04-tool-round-next-turn-and-terminal-reasons.md)
  - 工具轮续转、延后摘要与终止原因
- [../01-runtime/05-model-adapter-provider-and-auth.md](../01-runtime/05-model-adapter-provider-and-auth.md)
  - 模型门面、provider 选择与鉴权来源
- [../01-runtime/06-stream-processing-and-remote-transport.md](../01-runtime/06-stream-processing-and-remote-transport.md)
  - stream event、fallback、request telemetry 与 remote transport 的总览与拆分导航
- [../01-runtime/06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md](../01-runtime/06-stream-processing-and-remote-transport/01-hpc-stream-events-and-non-streaming-fallback.md)
  - `Hpc(...)` stream events、assistant 增量与 non-streaming fallback
- [../01-runtime/06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md](../01-runtime/06-stream-processing-and-remote-transport/02-request-telemetry-refusal-and-stream-status.md)
  - `NDl / KOo / UDl`、refusal/model block 与 `stream_request_start`
- [../01-runtime/06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md](../01-runtime/06-stream-processing-and-remote-transport/03-remote-transport-and-request-build-boundary.md)
  - `sdk-url` / bridge / ingress transport 与 request build 边界
- [../01-runtime/07-web-search-tool.md](../01-runtime/07-web-search-tool.md)
  - Web 搜索工具包装、服务端调用链与返回边界
- [../01-runtime/08-web-fetch-tool.md](../01-runtime/08-web-fetch-tool.md)
  - Web 获取工具、本地抓取链与服务端边界
- [../01-runtime/09-api-lifecycle-and-telemetry.md](../01-runtime/09-api-lifecycle-and-telemetry.md)
  - telemetry 初始化、startup event、1P event logging 与流量开关的总览与拆分导航
- [../01-runtime/09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md](../01-runtime/09-api-lifecycle-and-telemetry/01-initialization-and-remote-settings-gating.md)
  - `Rg8 / H4A / qvz / initializeTelemetry` 与 remote managed settings gating
- [../01-runtime/09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md](../01-runtime/09-api-lifecycle-and-telemetry/02-counters-startup-events-and-github-status.md)
  - counter 注册、startup telemetry 与 GitHub auth status
- [../01-runtime/09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md](../01-runtime/09-api-lifecycle-and-telemetry/03-event-logging-config-and-telemetry-settings.md)
  - 1P event logging 配置、endpoint 与 telemetry 设置矩阵
- [../01-runtime/09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md](../01-runtime/09-api-lifecycle-and-telemetry/04-traffic-switches-request-lifecycle-and-boundaries.md)
  - `DISABLE_TELEMETRY`、nonessential traffic、request lifecycle telemetry 与边界
- [../01-runtime/10-control-plane-api-and-auxiliary-services.md](../01-runtime/10-control-plane-api-and-auxiliary-services.md)
  - data plane / control plane 分层、OAuth/account、remote environment/session、GitHub 接入、1P telemetry、MCP proxy
- [../01-runtime/10-control-plane-api-and-auxiliary-services/01-hosts-auth-and-data-plane.md](../01-runtime/10-control-plane-api-and-auxiliary-services/01-hosts-auth-and-data-plane.md)
  - host、认证来源、数据面 endpoint 与请求边界
- [../01-runtime/10-control-plane-api-and-auxiliary-services/02-oauth-and-account-control-plane.md](../01-runtime/10-control-plane-api-and-auxiliary-services/02-oauth-and-account-control-plane.md)
  - OAuth、account control plane 与本地凭据衔接
- [../01-runtime/10-control-plane-api-and-auxiliary-services/03-remote-environments-and-sessions.md](../01-runtime/10-control-plane-api-and-auxiliary-services/03-remote-environments-and-sessions.md)
  - remote environment、session lifecycle 与远程入口
- [../01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md](../01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md)
  - bridge 凭据、worker lifecycle 与远端运行边界
- [../01-runtime/10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md](../01-runtime/10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md)
  - GitHub 接入、telemetry 旁路与 MCP proxy
- [../01-runtime/11-non-llm-network-paths.md](../01-runtime/11-non-llm-network-paths.md)
  - voice WebSocket、plugin install counts、GrowthBook、transcript share 与其他非模型出网链
- [../01-runtime/12-settings-and-configuration-system.md](../01-runtime/12-settings-and-configuration-system.md)
  - settings source、路径、优先级、缓存、写回、CLI 注入与 schema 轮廓
- [../01-runtime/12-settings-and-configuration-system/01-source-model-and-paths.md](../01-runtime/12-settings-and-configuration-system/01-source-model-and-paths.md)
  - settings source 模型、路径体系与层级来源
- [../01-runtime/12-settings-and-configuration-system/02-loading-policy-and-merge.md](../01-runtime/12-settings-and-configuration-system/02-loading-policy-and-merge.md)
  - settings 读取策略、remote managed settings 与 merge 规则
- [../01-runtime/12-settings-and-configuration-system/03-cache-refresh-and-writeback.md](../01-runtime/12-settings-and-configuration-system/03-cache-refresh-and-writeback.md)
  - settings cache、热刷新、状态退化与写回语义
- [../01-runtime/12-settings-and-configuration-system/04-cli-injection-schema-and-migration.md](../01-runtime/12-settings-and-configuration-system/04-cli-injection-schema-and-migration.md)
  - CLI 注入、schema preprocess 与迁移兼容层
- [../01-runtime/12-settings-and-configuration-system/05-key-consumers-and-rewrite-boundaries.md](../01-runtime/12-settings-and-configuration-system/05-key-consumers-and-rewrite-boundaries.md)
  - key consumer 索引、外围合同与重写边界
- [../01-runtime/13-abuse-risk-and-refusal-controls.md](../01-runtime/13-abuse-risk-and-refusal-controls.md)
  - 反滥用、风控、反蒸馏相关 category、policy、Gateway 防护与 refusal/fallback 边界

### 02-execution

- [../02-execution/01-tools-hooks-and-permissions.md](../02-execution/01-tools-hooks-and-permissions.md)
  - 工具执行器、Hook、权限系统的总览与拆分导航
- [../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md](../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md)
  - 工具调度、执行包装、结果回写与并发路径
- [../02-execution/01-tools-hooks-and-permissions/02-hook-system.md](../02-execution/01-tools-hooks-and-permissions/02-hook-system.md)
  - Hook 主题的总览与拆分导航
- [../02-execution/01-tools-hooks-and-permissions/02-hook-system/01-schema-instructionsloaded-and-event-inputs.md](../02-execution/01-tools-hooks-and-permissions/02-hook-system/01-schema-instructionsloaded-and-event-inputs.md)
  - hook schema、InstructionsLoaded 与事件输入
- [../02-execution/01-tools-hooks-and-permissions/02-hook-system/02-special-output-and-event-consumer-semantics.md](../02-execution/01-tools-hooks-and-permissions/02-hook-system/02-special-output-and-event-consumer-semantics.md)
  - 特殊输出分支与关键事件消费语义
- [../02-execution/01-tools-hooks-and-permissions/02-hook-system/03-runtime-order-and-cross-stage-timing.md](../02-execution/01-tools-hooks-and-permissions/02-hook-system/03-runtime-order-and-cross-stage-timing.md)
  - hook 与权限相关阶段的顺序图
- [../02-execution/01-tools-hooks-and-permissions/02-hook-system/04-instructionsloaded-non-main-thread-coverage-and-dispatch-boundaries.md](../02-execution/01-tools-hooks-and-permissions/02-hook-system/04-instructionsloaded-non-main-thread-coverage-and-dispatch-boundaries.md)
  - InstructionsLoaded 在非主线程的覆盖面与派发边界
- [../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md](../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md)
  - permission mode 状态机、auto mode gate 与 classifier
- [../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md](../02-execution/01-tools-hooks-and-permissions/04-policy-sandbox-and-approval-backends.md)
  - managed policy、sandbox 合流与审批后端
- [../02-execution/02-instruction-discovery-and-rules.md](../02-execution/02-instruction-discovery-and-rules.md)
  - instruction discovery、system section cache、InstructionsLoaded 与 context 边界的总览与拆分导航
- [../02-execution/02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md](../02-execution/02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md)
  - `Qv()` 主扫描、instruction sources、compat 文件与 `/init` 迁移边界
- [../02-execution/02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md](../02-execution/02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md)
  - `systemPromptSectionCache`、`zL(...)` 默认 section 与失效边界
- [../02-execution/02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md](../02-execution/02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md)
  - `O4t(...)` load reason、`InstructionsLoaded` hook 派发与时序
- [../02-execution/02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md](../02-execution/02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md)
  - `hS()`、`_H(...)`、`currentDate`、`gitStatus` 与 `claudeMdExcludes`
- [../02-execution/03-prompt-assembly-and-context-layering.md](../02-execution/03-prompt-assembly-and-context-layering.md)
  - 主线程 prompt 主题的总览与拆分导航
- [../02-execution/03-prompt-assembly-and-context-layering/01-system-chain-default-sections-and-context-sources.md](../02-execution/03-prompt-assembly-and-context-layering/01-system-chain-default-sections-and-context-sources.md)
  - system sections、上下文来源与 compact 相关失效边界
- [../02-execution/03-prompt-assembly-and-context-layering/02-attachment-order-skill-plan-meta-and-message-merge.md](../02-execution/03-prompt-assembly-and-context-layering/02-attachment-order-skill-plan-meta-and-message-merge.md)
  - attachment 顺序、skill/plan 注入与消息合并
- [../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md](../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md)
  - prompt caching 与最终 API payload 边界
- [../02-execution/03-prompt-assembly-and-context-layering/04-request-level-injection-layers-and-local-server-boundary.md](../02-execution/03-prompt-assembly-and-context-layering/04-request-level-injection-layers-and-local-server-boundary.md)
  - request 级注入层与本地/服务端边界
- [../02-execution/04-non-main-thread-prompt-paths.md](../02-execution/04-non-main-thread-prompt-paths.md)
  - 非主线程 prompt 主题的总览与拆分导航
- [../02-execution/04-non-main-thread-prompt-paths/01-shared-merge-skeleton-and-overrides.md](../02-execution/04-non-main-thread-prompt-paths/01-shared-merge-skeleton-and-overrides.md)
  - 共用 merge 骨架、模式裁剪与子代理注入位置
- [../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md](../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md)
  - hook/compact 特殊路径与 verification 残留边界
- [../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md](../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md)
  - fork-family snapshot 复用与 cacheSafeParams 边界
- [../02-execution/04-non-main-thread-prompt-paths/04-compat-agent-definitions-and-instruction-entry.md](../02-execution/04-non-main-thread-prompt-paths/04-compat-agent-definitions-and-instruction-entry.md)
  - compat 载体、agent definitions 来源与优先级
- [../02-execution/05-attachments-and-context-modifiers.md](../02-execution/05-attachments-and-context-modifiers.md)
  - attachment 与 context layers 主题导航
- [../02-execution/05-attachments-and-context-modifiers/01-attachment-lifecycle-and-producers.md](../02-execution/05-attachments-and-context-modifiers/01-attachment-lifecycle-and-producers.md)
  - attachment lifecycle、producer/consumer 与生成边界
- [../02-execution/05-attachments-and-context-modifiers/02-high-value-attachment-payloads-and-materialization.md](../02-execution/05-attachments-and-context-modifiers/02-high-value-attachment-payloads-and-materialization.md)
  - 高价值 attachment payload、materialization 与落地形态
- [../02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md](../02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md)
  - `contextLayers`、历史 `contextModifier` 口径与 executor consumer
- [../02-execution/06-context-runtime-and-tool-use-context.md](../02-execution/06-context-runtime-and-tool-use-context.md)
  - 运行时上下文分层、继承与裁剪规则

### 03-ecosystem

- [../03-ecosystem/01-resume-fork-sidechain-and-subagents.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents.md)
  - Resume/Fork/Sidechain、Subagent、Agent Team 的总览与拆分导航
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md)
  - Resume、Fork、Sidechain、Subagent 核心路径
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md)
  - Agent Team、TaskList、Roster 与协作模型
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md)
  - mailbox 协议、审批流与生命周期控制
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md)
  - teammate runtime、backend 分流与任务调度导航
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md)
  - teammate task 形态、backend 选择与 spawn 分流
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md)
  - in-process loop、idle wait、task claim 与 owner 回收
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md)
  - pane 子 CLI 回接、shutdown 与 hide/show 边界
- [../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md](../03-ecosystem/01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md)
  - teammate runtime 整体模型与保守结论
- [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)
  - remote-control、bridge、远程 transcript 持久化与冲突还原
- [../03-ecosystem/03-plan-system.md](../03-ecosystem/03-plan-system.md)
  - Plan 主题的总览与拆分导航
- [../03-ecosystem/03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md](../03-ecosystem/03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md)
  - plan runtime object、plan file 生命周期与 fork 复制
- [../03-ecosystem/03-plan-system/02-enter-exit-and-plan-command.md](../03-ecosystem/03-plan-system/02-enter-exit-and-plan-command.md)
  - plan 模式的进入、退出与命令入口
- [../03-ecosystem/03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md](../03-ecosystem/03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md)
  - 退出审批 UI、编辑状态与 bridge 回传
- [../03-ecosystem/03-plan-system/04-attachments-persistence-and-team-approval.md](../03-ecosystem/03-plan-system/04-attachments-persistence-and-team-approval.md)
  - plan 相关 attachment、持久化与 team approval
- [../03-ecosystem/04-mcp-system.md](../03-ecosystem/04-mcp-system.md)
  - MCP 配置源、连接层、指令增量、resource、deferred tool
- [../03-ecosystem/05-skill-system.md](../03-ecosystem/05-skill-system.md)
  - skills 注册源、动态发现、运行时修改与 fork 语义
- [../03-ecosystem/06-plugin-system.md](../03-ecosystem/06-plugin-system.md)
  - plugin marketplace、安装缓存、启停状态、运行时注入与管理 UI 的总览与拆分导航
- [../03-ecosystem/06-plugin-system/01-command-state-and-source-resolution.md](../03-ecosystem/06-plugin-system/01-command-state-and-source-resolution.md)
  - plugin 命令面、三层状态与 source precedence
- [../03-ecosystem/06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md](../03-ecosystem/06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md)
  - marketplace、manifest、安装缓存、source 类型与依赖闭包
- [../03-ecosystem/06-plugin-system/03-runtime-injection-settings-and-channels.md](../03-ecosystem/06-plugin-system/03-runtime-injection-settings-and-channels.md)
  - plugin 的运行时能力注入、settings、channels 与 builtin 边界
- [../03-ecosystem/06-plugin-system/04-validation-ui-and-safety-boundaries.md](../03-ecosystem/06-plugin-system/04-validation-ui-and-safety-boundaries.md)
  - `plugin validate`、`/plugin` 管理 UI、安全与下架处理
- [../03-ecosystem/07-tui-system.md](../03-ecosystem/07-tui-system.md)
  - TUI root、状态轴、对话框体系与 keybinding context
- [../03-ecosystem/07-tui-system/01-repl-root-and-render-pipeline.md](../03-ecosystem/07-tui-system/01-repl-root-and-render-pipeline.md)
  - 主 REPL root 与渲染管线
- [../03-ecosystem/07-tui-system/02-transcript-and-rewind.md](../03-ecosystem/07-tui-system/02-transcript-and-rewind.md)
  - transcript 搜索/导出、show-all、message-selector 与 rewind/summarize
- [../03-ecosystem/07-tui-system/03-input-footer-and-voice.md](../03-ecosystem/07-tui-system/03-input-footer-and-voice.md)
  - 输入区、attachments、footer、history/help 与 voice
- [../03-ecosystem/07-tui-system/04-dialogs-and-approvals.md](../03-ecosystem/07-tui-system/04-dialogs-and-approvals.md)
  - 对话框优先级、审批弹层与 callout 分支
- [../03-ecosystem/07-tui-system/05-help-settings-model-theme-and-diff.md](../03-ecosystem/07-tui-system/05-help-settings-model-theme-and-diff.md)
  - help、settings、theme/model picker 与 diff 对话框
- [../03-ecosystem/07-tui-system/06-tool-permission-dispatch.md](../03-ecosystem/07-tui-system/06-tool-permission-dispatch.md)
  - tool-specific 审批分派表
- [../03-ecosystem/07-tui-system/07-message-row-and-subtype-renderers.md](../03-ecosystem/07-tui-system/07-message-row-and-subtype-renderers.md)
  - 消息行渲染链、message type 与 block subtype
- [../03-ecosystem/07-tui-system/08-tool-result-renderers.md](../03-ecosystem/07-tui-system/08-tool-result-renderers.md)
  - tool result 渲染、sidecar 与错误/拒绝分支

### 04-rewrite

- [../04-rewrite/01-rewrite-architecture.md](../04-rewrite/01-rewrite-architecture.md)
  - 候选架构、职责边界、接口还原目标与落地顺序
- [../04-rewrite/02-open-questions-and-judgment.md](../04-rewrite/02-open-questions-and-judgment.md)
  - 能否开工的判断、阻塞级未决项与后续补证方向

### 05-appendix

- [../05-appendix/01-glossary.md](../05-appendix/01-glossary.md)
  - 压缩名、运行时术语、重写时建议命名
- [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md)
  - evidence map 导航入口；主题证据映射和关键结论复核入口已拆到子页
- [../05-appendix/02-evidence-map/01-entry-rules-and-request-snapshot.md](../05-appendix/02-evidence-map/01-entry-rules-and-request-snapshot.md)
  - 证据口径、当前发行版入口证据、断言登记规则与 request snapshot 速查
- [../05-appendix/02-evidence-map/02-overview-and-runtime-topic-map.md](../05-appendix/02-evidence-map/02-overview-and-runtime-topic-map.md)
  - overview 与 runtime 主题到证据来源的对应关系
- [../05-appendix/02-evidence-map/03-execution-topic-map.md](../05-appendix/02-evidence-map/03-execution-topic-map.md)
  - execution 主题到证据来源的对应关系
- [../05-appendix/02-evidence-map/04-ecosystem-rewrite-topic-map.md](../05-appendix/02-evidence-map/04-ecosystem-rewrite-topic-map.md)
  - ecosystem 与 rewrite 主题到证据来源的对应关系
- [../05-appendix/02-evidence-map/05-key-conclusion-review-and-next-evidence.md](../05-appendix/02-evidence-map/05-key-conclusion-review-and-next-evidence.md)
  - 强结论复核入口与后续补证据建议

## 建议阅读顺序

1. 先看 [01-scope-and-evidence.md](./01-scope-and-evidence.md)，确认当前结论和可信度边界。
2. 再读 runtime 与 execution 两组文件，建立主执行链。
3. 接着读 ecosystem，补齐 Resume、Remote、MCP、TUI 等外围系统。
4. 最后读 rewrite 与 appendix，把“知道什么”和“该怎么重写”接起来。

## 维护约定

- 新增证据时，优先改对应主题文件，再在 evidence map 中登记。
- 如果某个压缩名的职责判断发生变化，先改 glossary，再回改正文引用。
- 结构调整、父页收缩和新增拆分页时，先遵守 [02-document-style-and-structure-conventions.md](./02-document-style-and-structure-conventions.md)。
- 不把历史整理过程和旧目录痕迹重新带回知识库首页。

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
