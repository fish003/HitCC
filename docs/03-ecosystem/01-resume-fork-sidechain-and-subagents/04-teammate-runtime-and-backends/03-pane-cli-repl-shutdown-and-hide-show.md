# Pane CLI、REPL 回接、Shutdown 与 Hide/Show

## 本页用途

- 单独整理 pane teammate 与 in-process teammate 的运行差异。
- 固定 pane 子 CLI 从 argv identity 到标准 REPL 主循环的回接路径，以及 shutdown / hide-show 的当前边界。

## 相关文件

- [../04-teammate-runtime-and-backends.md](../04-teammate-runtime-and-backends.md)
- [../02-agent-team-and-task-model.md](../02-agent-team-and-task-model.md)
- [../03-agent-team-mailbox-and-approval.md](../03-agent-team-mailbox-and-approval.md)
- [../../../02-execution/04-non-main-thread-prompt-paths.md](../../../02-execution/04-non-main-thread-prompt-paths.md)

## pane / in-process 的运行差异边界

### 真正的差异

- in-process
  - worker loop 直接在 leader 当前 Node 进程里跑
- pane
  - worker loop 在新 pane 的 CLI 子进程里跑

### 不该误判成差异的东西

- 两者都通过 team file 还原 identity / roster / permissions
- 两者都通过 mailbox 收发 teammate 消息
- 两者都通过 shared task list auto-claim
- 两者都由同一套 CLI teammate runtime 语义驱动

因此更准确的结论是：

- pane 与 in-process 的**协作语义同构**
- 真正的差别在：
  - 执行进程位置
  - 终止桥接方式
  - leader 是否额外持有一个本地 task 壳

## pane 子 CLI 在注入 teammate identity 后，会回到标准 REPL 主循环

之前这部分只能写到“同一份 CLI + startup init 重新接回”。  
现在还能再往前接一跳。

### 1. pane spawn 传入的不是内部 IPC 协议，而是标准 CLI flags

pane backend 构造命令时，实际塞进 pane 的是：

- 同一个 CLI 可执行文件
- 再加：
  - `--agent-id`
  - `--agent-name`
  - `--team-name`
  - `--agent-color`
  - `--parent-session-id`
  - 条件性：
    - `--plan-mode-required`
    - `--agent-type`
    - `--model`
    - `--teammate-mode`

这说明 pane worker 首先不是通过某个专用 worker entrypoint 启动，而是重走主 CLI 入口。

### 2. CLI 启动早期只把这些 flags 落成 dynamic teammate context

主 CLI 解析 argv 后，会先做一层强校验：

- `--agent-id / --agent-name / --team-name` 必须三者同时提供

然后调用：

- `setDynamicTeamContext({ agentId, agentName, teamName, color, planModeRequired, parentSessionId })`

也就是说，pane 子进程刚启动时拿到的不是完整运行时 state，而只是一份：

- dynamic teammate identity snapshot

### 3. 进入 App / REPL 后，再由 startup hook 把这份 snapshot 接回 team runtime

`f3A(...)` 这个主 REPL 组件初始化时会直接调用：

- `GU4(setAppState, initialMessages, { enabled: !remote })`

而 `GU4(...)` 内部会：

1. 从 session resume 数据里尝试拿 `teamName / agentName`
2. 否则退回 `getDynamicTeamContext()`
3. 调 `WU4(...)` 把 `teamContext` 写进 app state
4. 再调 `b5A(...)`
   - 读取 team file
   - 应用 `teamAllowedPaths`
   - 给 teammate 安装 Stop/idle notification hook

因此 pane 子 CLI 在 runtime 上不是“只靠 argv 直接开始跑”，而是：

```text
argv teammate flags
  -> dynamic teammate context
  -> App/f3A mount
  -> GU4()
     -> WU4() 初始化 teamContext
     -> b5A() 安装 teammate startup hooks / team permissions
```

### 4. 真正发起模型请求时，走的仍是标准 main-thread query path

`f3A(...)` 里后续正常输入、prompt queue、queued command 最终都会落到：

- `repl_main_thread` query source
- `zN(...)`
- `oRf(...)`

当前可见的主线程 query source 是：

- `repl_main_thread`
  - 或带 output-style 后缀的 `repl_main_thread:...`

这说明 pane 子 CLI 在注入 teammate identity 之后，最后并不是跳进某条单独的 “pane teammate main loop”：

- 它回到的是**标准 REPL main-thread query path**

更准确的说法应是：

- pane teammate 用标准 CLI 入口启动
- 用 dynamic teammate context 把身份注入进去
- 再在 App 初始化阶段把身份接回 team runtime
- 最后仍通过标准 `repl_main_thread` 主查询链跑模型与工具

## shutdown / kill 的运行时边界

### in-process

- `terminate()`
  - 发 mailbox `shutdown_request`
  - 并标记 `shutdownRequested`
- `kill()`
  - 走 `HL8()`
  - `abortController.abort()`
  - `unregisterCleanup()`
  - 从 `teamContext.teammates` 移除
  - 把 task 状态改成 `killed`

### pane

- `terminate()`
  - 也是先发 `shutdown_request`
- 本地 task 壳的 `abortController.abort()`
  - 会转成 `killPane(...)`
- cleanup 时也会根据 `backendType / tmuxPaneId` 去 kill pane

所以 shutdown 协议仍然是 mailbox first，不是直接硬杀；  
但 pane backend 额外有一个 leader 本地 kill bridge。

## pane hide/show 当前更像残留半实现能力

这部分现在可以比“`hiddenPaneIds` 还没完全收口”写得更硬。

### 后端能力仍然存在

tmux backend `Cg1` 已经实现了：

- `hidePane(paneId)`
  - 通过 `break-pane` 把 pane 移到隐藏 session
- `showPane(paneId, target)`
  - 通过 `join-pane` 把 pane 接回目标 window

同时 backend capability 也显式区分：

- tmux:
  - `supportsHideShow = true`
- iTerm2:
  - `supportsHideShow = false`

因此底层不是“完全没有 hide/show 实现”。

### UI 入口与帮助文案也还在

TeamsDialog 仍保留：

- `h`
  - 当前成员 hide/show
- `H`
  - 批量 hide/show

帮助文案里也仍然直接显示：

- `h hide/show`
- `H hide/show all`

入口函数链也还能直接写出：

```text
TeamsDialog keybinding
  -> _Lz(teammate, teamName)
     -> if isHidden
          vg4(...)
        else
          Gg4(...)
```

### 但真正执行 hide/show 的上层函数在当前 bundle 里是空壳

当前 bundle 内：

- `GG4()` 函数体为空
- `VG4()` 函数体为空

同时全 bundle 也没看到任何活调用点去调用：

- backend `hidePane()`
- backend `showPane()`

也没看到当前活路径去调用：

- `Lj_(team, paneId)`
- `hj_(team, paneId)`

### 因此当前最稳判断

`hiddenPaneIds` / pane hide-show 这条链现在应拆成四层看：

1. team file 字段还在
2. tmux backend 能力还在
3. UI 快捷键与帮助文案还在
4. 但上层动作函数没有真正接到 backend / team-file helpers

因此更稳的结论不是“hide/show 功能完整但文档没补齐”，而是：

- **当前发行版里，pane hide/show 很可能是残留半实现功能**
- **展示层与后端层都还活着，但真正执行链大概率已经断线**

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
