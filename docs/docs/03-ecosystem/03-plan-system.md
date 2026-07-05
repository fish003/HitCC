# Plan 系统

## 本页用途

- 这页不再承载全部细节，而改成 `03-ecosystem` 下 plan 主题的总览与导航。
- 原先混在一页里的内容，已经拆成文件与状态对象、enter/exit 与 `/plan`、审批 UI 与 edited-plan 回传、attachments/持久化/团队审批四个专题页。

## 相关文件

- [03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md](./03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md)
- [03-plan-system/02-enter-exit-and-plan-command.md](./03-plan-system/02-enter-exit-and-plan-command.md)
- [03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md](./03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md)
- [03-plan-system/04-attachments-persistence-and-team-approval.md](./03-plan-system/04-attachments-persistence-and-team-approval.md)
- [01-resume-fork-sidechain-and-subagents.md](./01-resume-fork-sidechain-and-subagents.md)
- [01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md](./01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md)
- [../01-runtime/02-session-and-persistence.md](../01-runtime/02-session-and-persistence.md)
- [../02-execution/05-attachments-and-context-modifiers.md](../02-execution/05-attachments-and-context-modifiers.md)
- [07-tui-system.md](./07-tui-system.md)

## 拆分后的主题边界

### 运行时对象模型 / plan file 生命周期

见：

- [03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md](./03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md)

这一页集中放：

- plan runtime object 模型
- `plansDirectory`、slug、per-agent plan file 路径
- plan file 的读写、还原、snapshot 与 fork 复制链

### `EnterPlanMode` / `ExitPlanMode` / `/plan`

见：

- [03-plan-system/02-enter-exit-and-plan-command.md](./03-plan-system/02-enter-exit-and-plan-command.md)

这一页集中放：

- plan mode 的正式状态迁移
- `ExitPlanMode` 工具输出 schema
- `/plan` 作为 mode entry、viewer 与 external-editor launcher 的本地入口

### Exit 审批 UI / `planWasEdited` / CCR-ultraplan 回传

见：

- [03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md](./03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md)

这一页集中放：

- `Cm4(...)` 审批 UI 状态机
- clear-context 与 keep-context 后继链
- `planWasEdited` 的真实判定条件
- `Ctrl+G`、CCR/web、`ultraplanPendingChoice` 与 `needsAttention` 链

### attachments / compact-resume / teammate approval

见：

- [03-plan-system/04-attachments-persistence-and-team-approval.md](./03-plan-system/04-attachments-persistence-and-team-approval.md)

这一页集中放：

- `plan_mode*` 与 `plan_file_reference`
- compact / resume / clear-context 的保留链
- teammate -> leader plan approval mailbox 协议
- 当前已钉死与仍未完全钉死的边界

## 建议阅读顺序

1. 先看 [01-runtime-objects-and-plan-file-lifecycle.md](./03-plan-system/01-runtime-objects-and-plan-file-lifecycle.md)，建立 plan file 与 mode state 的核心模型。
2. 再看 [02-enter-exit-and-plan-command.md](./03-plan-system/02-enter-exit-and-plan-command.md)，补齐正式状态迁移与 `/plan` 入口。
3. 然后看 [03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md](./03-plan-system/03-exit-approval-ui-planwasedited-and-ultraplan-bridge.md)，理解退出审批和 edited-plan 回传链。
4. 最后看 [04-attachments-persistence-and-team-approval.md](./03-plan-system/04-attachments-persistence-and-team-approval.md)，把 attachment、还原与 team approval 合回同一条运行链。

## 与其它专题的边界

- teammate runtime、mailbox 与 approval backend 更广义的 team 机制，见 [01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md](./01-resume-fork-sidechain-and-subagents/03-agent-team-mailbox-and-approval.md)。
- TUI 里的 tasks、dialogs、plan detail 与 remote ultraplan 呈现，见 [07-tui-system.md](./07-tui-system.md)。
- attachment payload 的 materialize 细节，见 [../02-execution/05-attachments-and-context-modifiers.md](../02-execution/05-attachments-and-context-modifiers.md)。

## `/goal`、Workflow 与 worktree 的横切入口

`/goal` 不是 `/plan` 的别名。当前 2.1.197 的可见实现把它做成 session-scoped Stop hook：

- `/goal` 空参数读取 `options.activeGoal`，显示当前条件、检查轮次和最近原因。
- `/goal clear` 清除 active goal。
- `/goal <condition>` 会设置 active goal，并把条件转成停止前检查的 hook prompt。
- goal 状态会通过 `active_goal` 事件、`goal_status` attachment 和 resume restore 回到运行态。
- 受 trust / hook policy 约束：未受信 workspace、`disableAllHooks` 或 `allowManagedHooksOnly` 命中时不能启用。

这条链的证据点在 `package/preprocessed/cli.extracted.bundle.pretty.js:554237-554287`、`688617-688676`、`771580-771613`、`839112-839123`。因此 `/goal` 应归到“停止条件 / session runtime state”，而不是 plan file 生命周期。

Workflow、scheduled task 与 worktree 也横跨 plan / agent / background 三页：

| 面 | 当前可见字段或入口 | 归属边界 |
| --- | --- | --- |
| `WorkflowInput` | `script / name / args / scriptPath / resumeFromRunId` | workflow 自带可恢复 run id 与持久化 script path，不只是普通 prompt wrapper |
| `WorkflowOutput` | `status / taskId / taskType / workflowName / runId / transcriptDir / scriptPath / sessionUrl` | 结果天然进入 background task / remote session 体系 |
| `CronCreateInput` | `cron / prompt / recurring / durable` | scheduled task 可内存态或 `.claude/scheduled_tasks.json` durable |
| `EnterWorktree` / worktree settings | `baseRef / sparsePaths / symlinkDirectories / bgIsolation` | worktree 是 background agent、workflow 和手动隔离共用的执行边界 |

这些结论来自 `package/sdk-tools.d.ts:2449-2498`、`3275-3284`、`3416-3445` 与 settings schema `package/preprocessed/cli.extracted.bundle.pretty.js:66132-66157`、`66171-66194`。服务端是否默认开放 workflows 或 scheduled tasks，当前本地 bundle 不能证明。

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
