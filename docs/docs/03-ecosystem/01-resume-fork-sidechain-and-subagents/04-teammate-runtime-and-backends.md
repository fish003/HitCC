# Teammate Runtime、Backend 与任务调度

## 本页用途

- 这页不再承载全部细节，而改成 teammate runtime / backend / task scheduling 的总览与导航。
- 原正文已经拆成 task 形态与 spawn 分流、in-process loop 与 task claim、pane CLI 与 shutdown/hide-show、整体模型与保守结论四个专题页。

## 相关文件

- [../01-resume-fork-sidechain-and-subagents.md](../01-resume-fork-sidechain-and-subagents.md)
- [02-agent-team-and-task-model.md](./02-agent-team-and-task-model.md)
- [03-agent-team-mailbox-and-approval.md](./03-agent-team-mailbox-and-approval.md)
- [../../02-execution/04-non-main-thread-prompt-paths.md](../../02-execution/04-non-main-thread-prompt-paths.md)

## 拆分后的主题边界

### Task 形态 / Backend 选择 / Spawn 分流

见：

- [04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md](./04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md)

这一页集中放：

- `in_process_teammate` 作为真实 in-process worker task 与 pane leader 侧 task 壳的区别
- `Du()` 的 in-process 判定、`BA6()` 的 pane backend 检测矩阵
- `wQ4()` 与 `wu_() / $u_()` 的 spawn 分流
- pane 子进程作为完整 CLI teammate runtime 的启动参数与 team file 回接

### In-process Loop / Idle Wait / Task Claim

见：

- [04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md](./04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md)

这一页集中放：

- `obf(...)` turn executor 与 `rbf(...) / $j_()` idle wait loop
- `pendingUserMessages`、mailbox、task auto-claim 三类 wakeup source
- 固定 500ms tick 与没有指数退避的边界
- `Yj_()`、`rp1()`、`dIq()`、`F$_()` 和 owner reclaim 路径

### Pane CLI / REPL 回接 / Shutdown / Hide-Show

见：

- [04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md](./04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md)

这一页集中放：

- pane 与 in-process 的协作语义同构和运行位置差异
- pane 子 CLI 从 teammate flags 到 `setDynamicTeamContext(...)`、`GU4()`、`b5A(...)` 的回接
- 标准 `repl_main_thread -> zN(...) -> oRf(...)` 查询链
- mailbox-first shutdown、pane kill bridge 与 hide/show 残留半实现边界

### 整体模型 / 保守结论

见：

- [04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md](./04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md)

这一页集中放：

- spawn、worker turn、task claim、shutdown 的整体模型
- pane worker loop 复用证据与 `F$_()` 活调用点的保守表述

## 建议阅读顺序

1. 先看 [01-task-shapes-backend-selection-and-spawn.md](./04-teammate-runtime-and-backends/01-task-shapes-backend-selection-and-spawn.md)，明确 teammate task 形态和 backend 分流。
2. 再看 [02-in-process-loop-idle-wait-and-claim.md](./04-teammate-runtime-and-backends/02-in-process-loop-idle-wait-and-claim.md)，理解 idle tick、mailbox wakeup 与 task claim。
3. 然后看 [03-pane-cli-repl-shutdown-and-hide-show.md](./04-teammate-runtime-and-backends/03-pane-cli-repl-shutdown-and-hide-show.md)，把 pane 子 CLI 回接和生命周期边界补上。
4. 最后看 [04-runtime-model-and-conservative-points.md](./04-teammate-runtime-and-backends/04-runtime-model-and-conservative-points.md)，复核整体模型和仍需保守表述的点。

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
