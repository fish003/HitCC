# Teammate Task 形态、Backend 选择与 Spawn 分流

## 本页用途

- 单独整理 `in_process_teammate` 的两种运行时含义。
- 固定 `Du()`、`BA6()`、`wQ4()`、`wu_() / $u_()` 的 backend 选择和 spawn 边界。

## 相关文件

- [../04-teammate-runtime-and-backends.md](../04-teammate-runtime-and-backends.md)
- [../02-agent-team-and-task-model.md](../02-agent-team-and-task-model.md)
- [../03-agent-team-mailbox-and-approval.md](../03-agent-team-mailbox-and-approval.md)
- [../../../02-execution/04-non-main-thread-prompt-paths.md](../../../02-execution/04-non-main-thread-prompt-paths.md)

## `in_process_teammate` 是运行时实体，但有两种含义

之前最容易误判的一点，是把 `in_process_teammate` task type 直接等同于“工作一定在 leader 当前进程里跑”。  
本地 bundle 现在已经能把它拆成两种形态。

### 1. 真正的 in-process worker task

`jL8()` 注册的 task 字段当前至少包括：

- `type: "in_process_teammate"`
- `status: "running"`
- `identity`
  - `agentId`
  - `agentName`
  - `teamName`
  - `color`
  - `planModeRequired`
  - `parentSessionId`
- `prompt`
- `model`
- `abortController`
- `awaitingPlanApproval`
- `spinnerVerb`
- `pastTenseVerb`
- `permissionMode`
- `isIdle`
- `shutdownRequested`
- `lastReportedToolCount`
- `lastReportedTokenCount`
- `pendingUserMessages`
- `messages`
- `unregisterCleanup`

这才是由 `OL8() / Oj_()` 真正消费的本地 worker state。

### 2. pane teammate 的 leader 侧本地 task 壳

`Hq4()` 也会注册一个 `type: "in_process_teammate"` 的 task，但字段更少：

- 同样有 `identity`
- 同样有 `prompt / abortController / permissionMode / isIdle / shutdownRequested / pendingUserMessages`
- 但没有 `model`
- 没有 `messages`
- 没有 `spinnerVerb / pastTenseVerb`
- 它的 `abortController.abort()` 会转而 `killPane(...)`

因此要明确：

- `in_process_teammate` 是一个 **task/UI/bookkeeping 类型**
- 不是“执行位置一定在本进程”的充分条件

更准确的划分应是：

- `jL8() -> OL8() / Oj_()`：真正本进程 worker
- `Hq4() -> killPane hook`：pane worker 的 leader 侧本地镜像

## backend 选择：`Du()` 决定是否直接走 in-process，`BA6()` 决定 pane backend 类型

### `Du()`：是否启用 in-process

`Du()` 当前逻辑已经能直接写成：

1. 若当前是 non-interactive session：
   - 直接 `true`
2. 否则读取 teammate mode snapshot：
   - `in-process` -> `true`
   - `tmux` -> `false`
   - `auto` -> 继续判断
3. `auto` 下：
   - 若之前已被 `markInProcessFallback()` 标成 fallback
     - `true`
   - 否则：
     - `insideTmux === false`
     - 且 `inITerm2 === false`
     - 才走 `true`

因此 `auto` 的真实语义不是“随机挑一个 backend”，而是：

- 当前环境不像 pane-capable terminal 时直接 in-process
- 只要环境像 tmux / iTerm2，就优先尝试 pane backend
- pane backend 不可用时，才打 fallback 标记并回落 in-process

### `BA6()`：pane backend 检测矩阵

`BA6()` 当前可直接收敛为：

1. 已在 tmux 内
   - 直接选 `tmux`
   - `isNative: true`
   - `needsIt2Setup: false`
2. 在 iTerm2 内，且 `preferTmuxOverIterm2 !== true`
   - 若 `it2` CLI 可用
     - 选 `iterm2`
     - `isNative: true`
     - `needsIt2Setup: false`
3. 在 iTerm2 内，但 `it2` CLI 不可用
   - 若 tmux 可用
     - 退回 `tmux`
     - `isNative: false`
     - `needsIt2Setup: true`
4. 不在 tmux / iTerm2，但 tmux 可用
   - 选 `tmux`
   - `isNative: false`
   - `needsIt2Setup: false`
5. 否则
   - 抛“没有 pane backend 可用”

### snapshot 语义已经能写死

CLI 启动早期会 `captureTeammateModeSnapshot()`。  
pane teammate spawn 构造命令时，又会显式带：

- `--teammate-mode <snapshot>`

这说明：

- spawn 不是每次重新查配置
- 而是 leader 先截一份 mode snapshot，再传给 pane 子 CLI

## spawn 分流：真正的执行差异只在 `wQ4()` 与 `wu_() / $u_()`

顶层分流是：

```text
Ou_()
  -> if Du() === true
       wq4()
     else
       try BA6()
       catch (auto mode only)
         markInProcessFallback()
         wq4()
       else
         use_splitpane !== false ? wu_() : $u_()
```

### `wQ4()`：真正 in-process teammate

`wQ4()` 会：

1. 归一化 `name`
2. 算出 `agentId`
3. 调 `jL8()` 注册真正的 in-process task
4. 启动当前 in-process worker loop（本页下文按 `obf(...)` / `rbf(...)` 建模）
5. 回写：
   - `AppState.teamContext.teammates[...]`
   - team file `members.push({... backendType: "in-process" ...})`

### `wu_()` / `$u_()`：pane teammate

两条 pane 路径都遵循同一结构：

1. 创建 pane / window
2. 拼完整 CLI 命令
3. `sendCommandToPane(...)`
4. leader 本地调 `Hq4()` 注册 task 壳
5. team file `members.push({... backendType, tmuxPaneId, prompt, model ...})`
6. 再通过 mailbox 给新 teammate 塞初始 prompt

所以 pane teammate 不是“leader 持续在本地替它跑 agent loop”，而是：

- **leader 只负责 pane 创建、bookkeeping、kill 桥接与初始消息投递**
- **真正的 worker loop 在新 pane 里的 CLI 子进程执行**

## pane 子进程不是 dumb shell，而是完整 CLI teammate runtime

pane spawn 发进 pane 的命令当前已能写得很具体：

- 可执行文件：
  - `process.execPath` 或当前 CLI argv 路径
- flags：
  - `--agent-id`
  - `--agent-name`
  - `--team-name`
  - `--agent-color`
  - `--parent-session-id`
  - 条件性：
    - `--plan-mode-required`
    - `--agent-type`
    - `--model`
    - `--teammate-mode <snapshot>`
- env：
  - `CLAUDECODE=1`
  - `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
  - 以及透传的 provider / remote / config 相关 env

子进程启动后，本地还能确认：

1. CLI 参数会还原 teammate identity
2. 通过 `setDynamicTeamContext(...)` 注入动态 team context
3. startup / reconnect 阶段会再读 team file
4. 用 team file 还原：
   - `leadAgentId`
   - 自己在 `members[]` 中对应的 `agentId`
   - `teamAllowedPaths`
5. 再安装 teammate idle / Stop hook

因此 pane backend 并不是另一套弱化 runtime。  
更稳的说法是：

- **pane teammate 启动的是同一份 CLI**
- **身份通过 flags 注入**
- **运行态通过 team file + startup init 重新接回**

这也意味着：

- pane worker 的主执行逻辑不是另写一套
- 而是和 in-process teammate 共用同一套 teammate runtime 语义

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
