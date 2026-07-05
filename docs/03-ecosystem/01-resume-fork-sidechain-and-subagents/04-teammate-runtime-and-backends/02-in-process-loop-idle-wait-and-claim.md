# In-process Loop、Idle Wait 与 Task Claim

## 本页用途

- 单独整理 `obf(...)` turn executor、`rbf(...) / $j_()` idle wait loop 和固定 500ms tick。
- 固定 task auto-claim、owner 写入、`F$_()` 保守变体和 shutdown reclaim 的真实边界。

## 相关文件

- [../04-teammate-runtime-and-backends.md](../04-teammate-runtime-and-backends.md)
- [../02-agent-team-and-task-model.md](../02-agent-team-and-task-model.md)
- [../03-agent-team-mailbox-and-approval.md](../03-agent-team-mailbox-and-approval.md)
- [../../../02-execution/04-non-main-thread-prompt-paths.md](../../../02-execution/04-non-main-thread-prompt-paths.md)

## in-process worker loop：`obf(...)` 是 turn executor，`rbf(...)` 是 idle wait loop

### `obf(...)` 的主骨架

`obf(...)` 当前可直接概括为：

```text
初始化 system prompt / agentDefinition / tool set
-> 先尝试一次 dIq() 自动认领 task
-> 把初始 prompt 写入 teammate transcript
-> while not aborted:
     设 currentWorkAbortController
     必要时 compact 历史
     _3(...) 跑 agent turn
     把流事件/assistant/tool结果回写到 teammate task state
     结束后设 isIdle = true
     发 idle notification
     调 rbf(...) 进入等待
     收到下一条 message / shutdown_request / task prompt 后继续
```

它不是单次 `prompt -> response` 函数，而是完整的多轮 teammate event loop。

### `$j_()`：精确等待顺序

`$j_()` 的顺序现在已经可以写死：

1. 先读本地 task state
   - `pendingUserMessages`
   - 若有，直接弹出第一条并返回
2. 若不是首次轮询
   - `await C_(500)`
3. 检查 abort
4. 查 mailbox
   - 先扫 unread structured shutdown message
   - 找到就优先返回 `shutdown_request`
5. 再扫 leader 发来的 unread message
6. 若没有，再扫任意 unread message
7. 最后尝试 `dIq()` 从 shared task list auto-claim
8. 还没有就继续下一次轮询

因此这个 wait loop 的 wakeup source 有且只有三类：

- 本地 `pendingUserMessages`
- mailbox unread 消息
- task list 中新出现的可 claim task

### tick / sleep / backoff

这部分现在已经能写得比“500ms 级轮询”更硬：

- `pollCount === 0`
  - 不 sleep，立即检查一次
- `pollCount > 0`
  - 每轮固定 `await C_(500)`

也就是说：

- **in-process teammate 的 idle tick 是固定 500ms**
- **没有看到指数退避**
- **没有看到按失败原因拉长间隔**
- **claim 失败、mailbox 空、没有 pendingUserMessages，都会统一回到下一次 500ms tick**

换句话说，当前本地实现是：

- `fixed-interval polling`

而不是：

- `exponential backoff`
- `event-driven blocking wait`

## claim 竞争与原子性：当前只有 per-task lock，没有全局调度器

### `Yj_()`：挑“第一个可做的 pending task”

`Yj_()` 的选择条件非常直接：

- `status === "pending"`
- `owner` 为空
- `blockedBy` 里没有未完成前置任务

然后：

- 直接对 `QD(taskList)` 返回数组做 `find(...)`
- 拿到**第一个**符合条件的 task

因此自动调度的真实策略不是复杂打分，而是：

- **first eligible pending task wins**

### `QD()` 的顺序不是“低 ID 优先”的实现真相

这里现在可以再收紧一层。  
`QD(taskList)` 当前底层就是：

```text
readdir(taskDir)
  -> 过滤 *.json
  -> 去掉扩展名
  -> Promise.all(KU(taskId))
```

中间没有显式：

- `sort((a, b) => ...)`
- 按数字 ID 重排
- 按 mtime / ctime 重排

因此 `Yj_()` 的“第一个可做 task”其实不是：

- “数值最小的 id”

而是：

- **`QD(...)` 当前返回数组里的第一个 eligible task**

也正因为这样，下面三层要分开看：

- auto-claim 主干
  - 吃的是 `QD(...)` 原始顺序
- `TaskList` tool
  - 也只是直接映射 `QD(...)` 结果，不额外排序
- TUI tasks 面板
  - `gy8()` 才会按 `Po6(...)` 对 `completed / in_progress / pending` 分桶后重排
  - `pending` 里还会先把未阻塞项排前，再按 `id` 排

所以当前 bundle 里真正存在的是：

- **claim 顺序与 tool 输出顺序，共同依赖 `QD(...)`**
- **TUI 面板顺序，是单独的 display 重排**

至于“为什么文案总强调低 ID 优先”：

- prompt 文案明确这样要求
- task 相关辅助排序里也有按 `id` 数值/字典序比较的 `Po6(...)`

但 auto-claim 主干本身并没有额外重排逻辑；它吃的是 `QD(...)` 当前顺序。  
因此若以后要复刻运行时，`QD(...)` 是否稳定排序是一个必须单独决策的实现点，不能被 prompt 文案掩盖掉。

### `rp1()`：当前主 claim 路径

`rp1(taskList, taskId, agentName)` 的执行顺序是：

1. 先确认 task 还存在
2. 给该 task 的 lock path 上锁
3. 锁内重新读取 task
4. 拒绝：
   - task 不存在
   - 已被其他 owner 占用
   - 已 `completed`
   - `blockedBy` 仍含未完成前置任务
5. 通过后原子写：
   - `owner: agentName`

关键点在于：

- claim 竞争是靠 **per-task lock** 解决的
- 不是 leader 侧 centralized scheduler

### `dIq()`：claim 成功后的第二步

`dIq()` 在 `rp1()` 成功后还会再做一次：

- `ju(taskId, { status: "in_progress" })`

因此 auto-claim 真实是两段式：

```text
选 task
-> rp1() 原子写 owner
-> ju() 再写 status = in_progress
-> 生成 prompt: "Complete all open tasks. Start with task #..."
```

### claim 失败后的竞争表现

若 `rp1()` 失败：

- `dIq()` 只记 log
- 不会本轮重试其他 task
- 不会立刻 fallback 到 `F$_()`
- 也不会进入更慢 backoff

之后的行为就是：

- 回到 `$j_()` 外层
- 下一次 500ms tick 再重新跑一遍

因此当前 bundle 中 claim 竞争的保守结论是：

- 有原子 owner claim
- 但没有更高层的 retry / rebalance / work stealing 逻辑

## `F$_()` 的真实地位：当前更像未接主干的保守变体

之前只能说“像预留分支”；现在可以进一步收紧。

### `F$_()` 做了什么

`F$_(taskList, taskId, agentName)` 比 `rp1()` 多两件事：

1. 它拿的不是 per-task lock，而是 tasklist 高水位 lock
2. 它会额外检查：
   - 当前 agent 是否已经拥有其他未完成任务
   - 若有则返回：
     - `reason: "agent_busy"`
     - `busyWithTasks: [...]`

### 但它怎么被接入

`F$_()` 的唯一入口是：

- `rp1(..., { checkAgentBusy: true })`

而当前本地 bundle 内：

- `checkAgentBusy` 只命中这一处判断
- 没看到任何活调用点传入 `{ checkAgentBusy: true }`
- auto-claim 的 `dIq()` 用的是普通 `rp1()`

因此这里可以比原文档写得更硬：

- `F$_()` 不是死代码
  - 因为 `rp1()` 确实能分流进去
- 但在**当前本地 bundle 可见活路径**里：
  - 没有调用者
  - 没有接入 auto-claim 主干
  - 没有接入 leader 调度主干

所以当前最稳判断应是：

- **`F$_()` 是已实现但未接入当前主调度链的保守 claim 变体**

若以后要证明它不是预留分支，必须找到新的活调用点；目前本地没有。

## owner 写路径与回收路径

真正会改 task `owner` 的入口，目前可收敛为：

1. `TaskCreate`
   - 初始 `owner: undefined`
2. `TaskUpdate(owner=...)`
   - 显式分配
3. `TaskUpdate(status="in_progress")`
   - team 场景下可能隐式补 `owner = 当前 agent name`
4. auto-claim
   - `rp1()` 写入 `owner = agentName`
5. shutdown reclaim
   - `xA6(taskList, agentId, teammateName, reason)`
   - 把尚未完成的任务改回：
     - `owner: undefined`
     - `status: "pending"`
   - 匹配时同时兼容：
     - `owner === agentId`
     - `owner === teammateName`

因此 leader 在 owner 上的主动行为其实很有限：

- 显式 `TaskUpdate`
- shutdown 后回收

当前没看到 leader 常驻调度器去持续 rebalance 任务。

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
