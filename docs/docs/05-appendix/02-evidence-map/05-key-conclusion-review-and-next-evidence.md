# 证据索引：关键结论复核入口

## 关键结论复核入口

下面这些结论仍然重要，但它们各自已经有更合适的主承载页；本页只保留“先去哪里复核”的硬入口，不再在附录里重复展开主叙事。

### 1. 是否已经足以重写高相似版本

- 主判断页：
  - [../../00-overview/01-scope-and-evidence.md](../../00-overview/01-scope-and-evidence.md)
  - [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)
- 关键证据页：
  - [../../01-runtime/01-product-cli-and-modes.md](../../01-runtime/01-product-cli-and-modes.md)
  - [../../01-runtime/02-session-and-persistence.md](../../01-runtime/02-session-and-persistence.md)
  - [../../01-runtime/04-agent-loop-and-compaction/01-main-loop-state-caches-and-yield-surface.md](../../01-runtime/04-agent-loop-and-compaction/01-main-loop-state-caches-and-yield-surface.md)
  - [../../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md](../../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md)
  - [../../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md](../../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md)
  - [../../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md](../../02-execution/01-tools-hooks-and-permissions/03-permission-mode-and-classifier.md)
  - [../../03-ecosystem/01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md](../../03-ecosystem/01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md)
- 相关边界：
  - “不承诺还原原始源码文件结构 / 私有服务端实现 / 1:1 原版复刻” 统一看 `00-overview/01`
  - “当前阻塞级未决项与开工策略” 统一看 `04-rewrite/02`
  - “模块合同、接口主线、依赖方向、registry/adapter/opaque field 承接规则” 统一看 `04-rewrite/01`

### 2. 网络通信模块是否已经形成主分层闭环

- 主判断页：
  - [../../00-overview/01-scope-and-evidence.md](../../00-overview/01-scope-and-evidence.md)
- 关键证据页：
  - [../../01-runtime/06-stream-processing-and-remote-transport.md](../../01-runtime/06-stream-processing-and-remote-transport.md)
  - [../../01-runtime/09-api-lifecycle-and-telemetry.md](../../01-runtime/09-api-lifecycle-and-telemetry.md)
  - [../../01-runtime/10-control-plane-api-and-auxiliary-services/01-hosts-auth-and-data-plane.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/01-hosts-auth-and-data-plane.md)
  - [../../01-runtime/10-control-plane-api-and-auxiliary-services/03-remote-environments-and-sessions.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/03-remote-environments-and-sessions.md)
  - [../../01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/04-bridge-credentials-and-worker-lifecycle.md)
  - [../../01-runtime/10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md](../../01-runtime/10-control-plane-api-and-auxiliary-services/05-github-telemetry-and-mcp-proxy.md)
  - [../../01-runtime/11-non-llm-network-paths.md](../../01-runtime/11-non-llm-network-paths.md)
  - [../../03-ecosystem/02-remote-persistence-and-bridge.md](../../03-ecosystem/02-remote-persistence-and-bridge.md)
- 相关边界：
  - `environment_secret / session_ingress_token / worker_epoch` 等服务端语义黑箱，统一回到 `01-runtime/10` 与 `03-ecosystem/02`
  - shared TLS / CA trust 层当前已钉住默认 `bundled,system`、`CLAUDE_CODE_CERT_STORE` 与 `NODE_EXTRA_CA_CERTS` 行为，统一回到 `01-runtime/11`
  - telemetry/exporter、voice provider、stats 上游来源等剩余未知，统一回到对应 runtime 专题页

### 3. 是否还值得以“还原原源码”为主目标

- 主判断页：
  - [../../00-overview/01-scope-and-evidence.md](../../00-overview/01-scope-and-evidence.md)
  - [../../04-rewrite/01-rewrite-architecture.md](../../04-rewrite/01-rewrite-architecture.md)
  - [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)
- 复核重点：
  - 总览页负责说明“文档不承诺什么”
  - rewrite 页负责说明“为什么当前更适合按职责边界重写，而不是假设原始文件树已还原”
  - 候选 `src/` 目录只作为后续实现的模块拆分建议；没有明确新指令时，恢复文档维护阶段不创建实现代码

### 4. Prompt / Context 主链与未决边界

- 主判断页：
  - [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)
- 关键证据页：
  - [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)
  - [../../02-execution/03-prompt-assembly-and-context-layering/01-system-chain-default-sections-and-context-sources.md](../../02-execution/03-prompt-assembly-and-context-layering/01-system-chain-default-sections-and-context-sources.md)
  - [../../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md](../../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md)
  - [../../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md](../../02-execution/04-non-main-thread-prompt-paths/02-hook-and-compact-special-paths.md)
  - [../../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md](../../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md)
  - [../../01-runtime/05-model-adapter-provider-and-auth.md](../../01-runtime/05-model-adapter-provider-and-auth.md)
- 相关边界：
  - `_H(...)` 第二注入槽位、`O4t("compact")` 覆盖面、verification/compat/服务端二次注入等剩余问题，统一回到这些专题页与 `04-rewrite/02`

### 5. 上下文缓存 / request snapshot 是否已形成本地闭环

- 速查入口：
  - 先看 [01-entry-rules-and-request-snapshot.md](./01-entry-rules-and-request-snapshot.md) 里的“上下文缓存 / request snapshot 专题速查”
- 关键证据页：
  - [../../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md](../../01-runtime/04-agent-loop-and-compaction/02-compaction-pipeline-and-auto-compact-tracking.md)
  - [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)
  - [../../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md](../../02-execution/03-prompt-assembly-and-context-layering/03-api-payload-order-prompt-caching-and-final-boundary.md)
  - [../../02-execution/04-non-main-thread-prompt-paths/01-shared-merge-skeleton-and-overrides.md](../../02-execution/04-non-main-thread-prompt-paths/01-shared-merge-skeleton-and-overrides.md)
  - [../../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md](../../02-execution/04-non-main-thread-prompt-paths/03-fork-family-cache-safe-params-and-snapshot-reuse.md)
- 相关边界：
  - 这里的“闭环”只指本地 bundle 可见范围
  - 不等于服务端没有额外 cache layer

### 6. `contextLayers` / `ToolUseContext` 的真实地位

- 主判断页：
  - [../../02-execution/06-context-runtime-and-tool-use-context.md](../../02-execution/06-context-runtime-and-tool-use-context.md)
- 关键证据页：
  - [../../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md](../../02-execution/01-tools-hooks-and-permissions/01-tool-execution-core.md)
  - [../../02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md](../../02-execution/05-attachments-and-context-modifiers/03-context-modifier-and-executor-consumers.md)
  - [../../03-ecosystem/05-skill-system.md](../../03-ecosystem/05-skill-system.md)
- 相关边界：
  - 当前本地直接看到的 tool-returned concrete producer 至少包括 `SkillTool` 与 `EnterWorktree`
  - `contextModifier` 是早期文档概念名；当前 `2.1.197` 活实现应以 `contextLayers / permissionLayers / NYt(...)` 为准
  - 是否存在 bundle 外、远端或服务端侧的更多 producer，统一保持未决

## 后续补证据建议

- 若还要继续补 prompt/context 这条线，优先看 `_H(...)` 预留第二注入槽位与 verification 家族碎片的 build-time 来源。
- 再补 bundle 外 hook 扩展分支与少量流式取消边角。
- Prompt 相关补证据时，统一登记到最贴近的 evidence map 子页，避免同一问题在多份正文里重复写。

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
