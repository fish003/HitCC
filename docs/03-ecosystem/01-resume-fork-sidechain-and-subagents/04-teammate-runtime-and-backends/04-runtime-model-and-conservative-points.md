# Teammate Runtime 模型与保守结论

## 本页用途

- 保留 teammate runtime 的整体模型。
- 集中列出当前 bundle 证据仍需保守表述的点。

## 相关文件

- [../04-teammate-runtime-and-backends.md](../04-teammate-runtime-and-backends.md)
- [../02-agent-team-and-task-model.md](../02-agent-team-and-task-model.md)
- [../03-agent-team-mailbox-and-approval.md](../03-agent-team-mailbox-and-approval.md)
- [../../../02-execution/04-non-main-thread-prompt-paths.md](../../../02-execution/04-non-main-thread-prompt-paths.md)

## 当前结论

基于当前本地 bundle，已经可以把 teammate runtime 写成下面这个模型：

```text
spawn
  -> 选 in-process / pane backend
  -> 注册本地 teammate task（真实 worker 或 leader shell）
  -> pane 时额外启动一份完整 CLI 子进程

worker turn
  -> _3(...) 执行一轮
  -> turn 结束进入 idle
  -> 发 idle notification
  -> rbf(...) 以固定 500ms tick 等待：
       pendingUserMessages
       -> mailbox
       -> task auto-claim

task claim
  -> Yj_() 选第一个可做 pending task
  -> rp1() 原子写 owner
  -> ju() 写 status = in_progress
  -> 失败则仅记 log，下一次 500ms tick 再来

shutdown
  -> mailbox 协议优先
  -> pane 场景再通过本地 task 壳桥接 killPane
  -> xA6() 回收未完成任务 owner/status
```

## 仍需保守表述的点

- pane teammate 的 worker loop 虽然高度可判定为复用同一套 CLI teammate runtime，但当前直接抽到的轮询细节证据主要来自 `Oj_() / $j_()` 本身，而不是对子进程内部再次单独反编译出一条新分支
- `F$_()` 当前可非常强地判断为“未接当前主干”，但严格说仍属于**已实现、可被 future caller 激活**的分支，而不是可以写成“无意义死代码”

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
