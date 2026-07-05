# Resume、Fork、Sidechain、Subagent 与 Agent Team

## 本页用途

- 这页不再承载全部细节，而改成 `03-ecosystem` 下这一组主题的总览与导航。
- 原先混在一页里的内容，已经拆成 Resume/Fork/Sidechain、Agent Team 模型、mailbox 协议、teammate runtime 四个专题页。

## 相关文件

- [01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md](./01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md)
- [01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md](./01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md)
- [01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md](./01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md)
- [01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md)
- [01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md)
- [01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md)
- [01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md)
- [01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md)
- [02-remote-persistence-and-bridge.md](./02-remote-persistence-and-bridge.md)
- [../04-rewrite/01-rewrite-architecture.md](../04-rewrite/01-rewrite-architecture.md)

## 拆分后的主题边界

### Resume / Fork / Sidechain / Subagent

见：

- [01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md](./01-resume-fork-sidechain-and-subagents/01-resume-fork-sidechain-and-subagent-core.md)

### Agent Team / TaskList / Roster

见：

- [01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md](./01-resume-fork-sidechain-and-subagents/02-agent-team-and-task-model.md)

### Mailbox / Permission / Plan Approval / Idle / Shutdown

见：

- [01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md](./01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md)

### Teammate Runtime / Backend / Scheduling

见：

- [01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends.md)

这一组子页集中放：

- [task 形态、backend 选择与 spawn 分流](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md)
- [in-process loop、idle wait 与 task claim](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md)
- [pane CLI、REPL 回接、shutdown 与 hide/show](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md)
- [runtime 模型与保守结论](./01-resume-fork-sidechain-and-subagents/04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md)

## Agent View、background session 与 worktree 横切边界

当前 2.1.197 把 `claude agents`、`--bg`、background job state、pinned notifications、workflow task 和 worktree isolation 接在同一组能力附近，但它们不属于同一个文件格式：

- `Agent` tool 输入侧直接支持 `run_in_background`、`name`、`mode` 与 `isolation: "worktree" | "remote"`，证据在 `package/sdk-tools.d.ts:431-449`。
- background job 本地状态含 `state / tempo / inFlight / sessionId / cwd / worktreePath / backend` 等字段，归属于 daemon/background handler；`claude agents --json` 还会在 waiting 状态下输出 `waitingFor`，用于区分 permission prompt、worker request、sandbox request、dialog open 或 input needed，证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:443306-443307`、`823745`、`866317-866336`。
- pinned background notifications 是 Agent View UI 状态，不是 transcript 或 TaskList 的 canonical state，证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:211760-211823`。
- workflow 输出的 `taskType / workflowName / runId / transcriptDir / scriptPath / sessionUrl` 会进入 background task surface，证据在 `package/sdk-tools.d.ts:3416-3445`。
- worktree isolation 既服务普通 subagent，也服务 `--bg`、Workflow 与 `EnterWorktree`，settings gate 在 `package/preprocessed/cli.extracted.bundle.pretty.js:66132-66157`。
- 普通 `Agent` tool 有 subagent depth cap：启动前会读取当前 `agentContext.depth`，超过上限时记录 `subagent_launch / subagent_depth_cap` 并提示直接使用当前工具完成任务；子 agent 启动时会把 depth 加一，证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:514304-514315`、`514567`。

因此这一组文档的恢复边界是：Team/TaskList 负责 agent/team source-of-truth；Agent View 负责可见调度面；daemon/background state 负责本地生命周期；worktree 负责隔离执行环境。

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
