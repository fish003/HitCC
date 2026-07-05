# 产品形态、入口、命令树与运行模式

## 本页用途

- 用来快速理解 Claude Code CLI 是什么产品，以及 CLI 如何从顶层入口分流到不同运行模式。
- 用来建立当前 2.1.197 bundle 中 `l3m() -> main(l4m) -> commander(u4m)` 这一段启动链的整体认识。

## 相关文件

- [../00-overview/01-scope-and-evidence.md](../00-overview/01-scope-and-evidence.md)
- [02-session-and-persistence.md](./02-session-and-persistence.md)
- [03-input-compilation.md](./03-input-compilation.md)
- [04-agent-loop-and-compaction.md](./04-agent-loop-and-compaction.md)
- [../05-appendix/01-glossary.md](../05-appendix/01-glossary.md)

## 产品形态总览

从发行版可以明确看出，Claude Code CLI 不是“问答命令行”，而是一个完整的 **Agentic Coding Shell**。

其能力至少包括：

- Commander 风格 CLI 程序
- React/Ink 风格 TUI
- Headless/SDK/JSON 流模式
- 会话持久化与还原
- 分叉会话 / 子 Agent / Sidechain
- 工具执行与权限系统
- MCP server/client 管理
- 插件管理与 Marketplace
- Chrome/native-host / computer-use / remote-control bridge
- Hook 系统
- Rules / Skills / Memory / Plan / File History
- 远程 transcript 同步

### 产品本质

其产品本质可以理解为：

```text
CLI/TUI 外壳
  -> 输入编译器
    -> 多轮 Agent Loop
      -> 模型调用适配层
      -> 工具执行器
      -> Hook/Permission
      -> Session/Transcript/Plan/FileHistory
```

### 与一般聊天 CLI 的根本差异

Claude Code CLI 与普通聊天 CLI 的最大区别：

1. **会话不是单一消息数组**，而是带 file-history、content-replacement、plan、attribution、queued commands 的“工作现场”。
2. **工具不是插件式附加物**，而是主循环的一等公民。
3. **Prompt 不是固定字符串**，而是多源、多层、带缓存的 system sections。
4. **Resume/Fork** 还原的是“工作状态”，不仅是聊天记录。
5. **Headless 与 TUI 共享核心循环**，只是外层 transport/render 不同。

---

## 顶层入口与启动分流

### 顶层入口：`l3m()`

当前 2.1.197 的最外层入口是 `async function l3m()`，bundle 末尾直接调用 `l3m();`。它不是旧文档里记录的 `jBz()`。

`l3m()` 的职责不是直接进入 Commander 主程序，而是先完成参数合法性检查、版本输出、多个旁路模式和 background/daemon/fleet fast-path 分流，最后才懒加载主 `main`。

### 已确认分支

按大致顺序：

1. `--version` / `-v` / `-V`：直接输出 `2.1.197 (Claude Code)`；带 `--verbose` 时额外输出 `Commit: c8fd8048f30950a21d28734718275aa7e97f5143`。
2. `--claude-in-chrome-mcp`：加载 `runClaudeInChromeMcpServer()`，启动 Chrome MCP server。
3. `--chrome-native-host`：加载 `runChromeNativeHost()`，启动 Chrome native host。
4. `--computer-use-mcp`：加载 `runComputerUseMcpServer()`，启动 computer-use MCP server。
5. `--daemon-worker <kind>`：先走 `loadFastPathPolicy()`，再加载 `runDaemonWorker()`。
6. `--bg-pty-host ...`：先走 `ensureFastPathSettingsLoaded()`，再加载 `runPtyHost()`。
7. `--bg-spare ...`：先走 `ensureFastPathSettingsLoaded()`，再加载 `runBgSpare()`。
8. `--preload ...`：先走 `ensureFastPathSettingsLoaded()`，再加载 `runPreload()`。
9. `remote-control | rc | remote | sync | bridge`：进入 bridge fast-path。该路径会加载 fast-path policy、bridge disable reason、bridge minimum version、OAuth token、trusted device 和 `allow_remote_control` policy，最后调用 `bridgeMain()`。
10. `daemon ...`：加载 fast-path settings、初始化 log sinks，再调用 `daemonMain()`。
11. `logs | attach | stop | kill | respawn | rm` 或任意 `--bg | --background`：进入 background agents fast-path。该路径先检查 policy 与 agents fleet gate，再按命令调用 background handlers 或 `handleBgFlag()`。
12. `agents` positional / 默认 agents view：当命中 agents positional 或 `defaultToAgentsView` 且 stdin/stdout 是 TTY 时，入口会提前捕获输入、加载 settings/plugin flags、检查 fleet gate，直接 mount fleet view。
13. `--tmux + --worktree`：命中 tmux/worktree fast-path；在 worktree mode 开启时调用 `execIntoTmuxWorktree()`，成功处理后不再进入主 Commander。
14. 单独 `--update` / `--upgrade`：把 `process.argv` 改写成 `update` 子命令。
15. `--bare`：在 `--` 之前出现时设置 `CLAUDE_CODE_SIMPLE=1`。
16. 非 `NON_REPL_SUBCOMMANDS`：调用 `startCapturingEarlyInput()`，在主模块加载前捕获用户输入。
17. 并行启动 `startMdmRawRead()` 与 `startKeychainPrefetch()`。
18. 动态 import 主模块，取出 `main` 并执行。

### 设计意图

这是典型的“**薄入口 + 重模式提前分流 + 主模块延迟加载**”设计。

优点：

- 启动更快
- Bridge/Chrome/Native Host 模式不必加载整套 TUI/Agent Loop
- 某些特殊子进程可以保持较小内存占用

### 重写建议

重写时保留这个分层：

```ts
entry.ts
  -> fastPathDispatch(argv)
  -> mainCli()
```

---

## 主 CLI 程序与命令树

### `main` / `l4m()`

当前 bundle 中主模块导出 `main`，其压缩函数名可见为 `l4m()`。`l3m()` 在所有 fast-path 之后动态加载该导出。

`main(l4m)` 的职责是：

- 初始化信号处理
- 处理 deep-link/deeplink URI
- 判断是否非交互模式
- 设置 client type / entrypoint
- 设置 question preview format
- 在 bridge 环境下标记 `remote-control`
- 进入 Commander 程序构建与执行

### 非交互模式判定

至少会触发 non-interactive 的条件：

- `-p` / `--print`
- `--init-only`
- `--sdk-url`
- `!stdout.isTTY`

### 推论

Claude Code CLI 的非交互模式不只是一种“纯文本打印”，而是一个独立的一等运行模式，Headless/SDK/Bridge 都依赖此路径。

### Commander 装配：`u4m()`

当前 Commander 装配函数可见为 `u4m()`。它负责两件事：

1. 构建 Commander 命令树
2. 设置 `preAction` 初始化逻辑

### 已确认的顶层命令

- `claude [prompt]`
- `auth`
- `agents`
- `auto-mode`
- `doctor`
- `gateway`
- `install`
- `project`
- `mcp`
- `plugin` / `plugins`
- `setup-token`
- `ultrareview`
- `update` / `upgrade`
- 隐藏：`remote-control` / `rc`
- 隐藏：`import-conversations`

### 已确认的子命令

#### auth
- `login`
- `logout`
- `status`

#### mcp
- `add`
- `add-from-claude-desktop`
- `add-json`
- `get`
- `list`
- `login`
- `logout`
- `remove`
- `reset-project-choices`
- `serve`

`claude mcp login <name>` 与 `claude mcp logout <name>` 是当前命令树里的显式子命令。`logout` 的本地行为不是简单删除配置：claude.ai connector 会提示到 claude.ai 侧断开，非 OAuth transport 会提示没有本地 credential 可清，hosted/OAuth server 则清本地 credential 或 revoke 后提示重新 `mcp login`。证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:737215-737241`、`737355-737363`。

#### plugin
- `init` / `new`
- `install`
- `uninstall` / `remove`
- `enable`
- `disable`
- `update`
- `list`
- `details`
- `prune` / `autoremove`
- `tag`
- `marketplace`
- `validate`

#### command / slash command surface

当前 2.1.197 还能把几个容易被遗漏的产品入口写实：

- `/goal [<condition> | clear]`：session-scoped goal 入口；空参数显示当前 active goal，`clear` 清除目标，带条件时创建会在停止前检查的 goal。
- `/review`：当前注册为 slash command，和 code review 能力共用本地 review surface。
- `/code-review ultra`：面向 cloud multi-agent review 的用户触发入口；本地提示明确 `/ultrareview` 是 deprecated alias。
- `/ultrareview` 与 `claude ultrareview [target]`：仍保留为 cloud review 启动面，包含 preflight、登录/entitlement 检查与远端任务 launch/poll 路径。
- `/recap`：生成当前 session 的 one-line recap；没有可总结 turn 时返回 `Nothing to recap yet`。
- `/release-notes`、`/terminal-setup`、`/install-github-app`、`/scroll-speed`、`/color`、`/theme`、`/powerup`、`/team-onboarding`：当前 bundle 均有 slash command 定义，但它们主要属于 release notes、终端设置、GitHub App onboarding、滚轮速度、外观/提示栏颜色、交互教程和团队 onboarding guide。除非要恢复 TUI/配置 UI 细节，否则应作为 low-impact command surface 记录，不应展开成核心 runtime 逻辑。

这些低影响入口的证据点在 `package/preprocessed/cli.extracted.bundle.pretty.js:688339`、`633783`、`654525`、`611686`、`647131`、`582579-582589`、`655869-655872`、`633462`、`690054`。

这里的边界是：文档可以写入口、参数与本地 preflight/dispatch；不能写成“本地代码一定能替用户启动 cloud review”，因为相关提示明确要求用户触发且受 auth、account entitlement、policy 与远端可用性约束。

#### low-impact command 的恢复价值筛选

不是所有 slash command 都值得展开成 runtime 主逻辑。当前更适合作为合同记录的低影响入口，是会改变 session 状态、配置、排障或外部 review 面的命令：

- `/cd <path>`
  - 会实际 `process.chdir(...)`，刷新 cwd / git branch / config cache，并追加 meta message 提醒开场 environment block 已过期
  - 目标路径必须存在且是目录
  - 还会经过 `/cd` permission rule 检查；命中 block 或 allow pattern 不匹配时拒绝移动
  - 若目标不在已信任目录，会先进入确认 UI
- `/config`
  - TUI 中空参数打开 Settings overlay
  - `key=value` 形式直接写设置
  - `/config help` / `/config --help` / 类似 help token 回显 usage 与可用 key，不是另一个独立子命令
  - print / non-interactive 侧也有 local 版本，只支持 `key=value`
- `/doctor`
  - 既是顶层 Commander 命令，也进入 TUI 的 Doctor context
  - 当前多处错误提示会把它作为排障入口，例如 MCP settings、sandbox、keybinding、GitHub CLI、update lock 等
- `/plugin list`
  - plugin 主命令树里有 `list`
  - 完整的 `/plugin` overlay 状态机在 plugin 专题页维护；CLI 入口只需要说明它是管理面的一部分
- `/focus`、`/rewind`
  - 分别影响 focus view 和 message-selector 恢复工作流，详见 TUI 专题页

这组入口的共同点是：它们虽然不是 agent loop 主干，却会改变后续工具路径、UI 状态、配置面或排障判断。其它纯介绍、onboarding、颜色、release notes 类命令只保留入口名即可。

#### PR / review platform 入口

当前本地 PR 面可以分三层写：

- `--from-pr [value]`
  - 是主 `claude [prompt]` action 的 CLI option
  - 无值时进入交互选择；有值时可按 PR number / URL / 搜索词继续 resume 关联 session
  - 它和 `--resume` 在后续分支合流，但 `fromPr` 会先转换成 PR resume target，不等价于普通 session id
- `/review [pr number]`
  - 是 prompt slash command，描述明确是 review GitHub pull request
  - 无参数时提示先用 `gh pr list` 选择 open PR
  - 对当前 working diff，应使用 `/code-review`
- `/code-review ultra` / `claude ultrareview`
  - 是 cloud multi-agent review 启动面；`/ultrareview` 属于 deprecated alias
  - 本地负责 preflight、entitlement、launch/poll 与提示；是否真正产出结果仍取决于远端任务

GitLab / Bitbucket / GHE 的边界需要小心：

- GitLab / Bitbucket 当前主要出现在 MCP discovery、plugin customization、external source-control 文案、registry keyword 或 URL allowlist 一类表面
- `GH_HOST` / GitHub Enterprise guidance 能证明 GitHub host 可配置
- 当前还不能把 GitLab / Bitbucket 写成已经接入 `--from-pr` 的同一 runtime path
- `prUrlTemplate` 已是 settings 合同，可影响 PR URL 展示；它不是外部平台 API connector

因此恢复实现时应把 GitHub PR resume / review、MCP connector discovery、footer PR URL 模板、cloud code review 分开建模。

#### Advisor 与 Monitor 的产品层级

`Advisor` 当前不是本地普通 tool implementation，而是 server-side advisor tool 的客户端装配面：

- settings key `advisorModel` 描述为 server-side advisor tool 的 model
- CLI option `--advisor <model>` 用于启用 server-side advisor tool
- 本地会校验 main model 是否支持 advisor、advisor model 是否可作为 advisor、advisor 是否至少和 main model 同级
- request tools 注入时会加入 `{ type: "advisor_20260301", name: "advisor", model }`
- bundle 中也保留了 advisor tool result / beta header 相关文本

`Monitor` 则是另一层：

- system guidance 会把 `Monitor` 描述成流式监控后台进程输出的工具；每条 stdout line 作为 notification
- plugin manifest 的 `experimental.monitors` / `monitors/monitors.json` 可由 host 在 session start 或 skill invoke 时 arm
- 任务类型里能看到 `monitor_mcp` 与 `monitor_ws`
- schema 文案明确这些 persistent Monitor tasks 是 unsandboxed，信任级别和 hooks 相同

所以两者不能混写：

- `Advisor`：模型请求侧的 server-side tool / beta tool
- `Monitor`：host/plugin/background task 侧的 persistent watch task

#### daemon

`daemon` 在当前版本是 `l3m()` fast-path，不等到主 Commander 注册。它自己的 dispatcher 支持：

- `run`
- `status`
- `logs` / `log`
- `stop`
- `uninstall`
- 受 gate 限制的 `list`
- 受 gate 限制的 `scheduled`
- 受 gate 限制的 `remote-control`
- 受 gate 限制的 `hub`

`install` / `start` / `restart` 也有 dispatcher case，但当前 Linux 形态会提示 service install disabled 或 no launchd/systemd；daemon 按 on-demand 方式运行。

### 关键选项

从主 action 入口与 help 文本可确认常见选项至少包括：

- `--print`
- `--output-format`
- `--input-format`
- `--json-schema`
- `--continue`
- `--resume`
- `--fork-session`
- `--session-id`
- `--name`
- `--model`
- `--effort`
- `--fallback-model`
- `--tools`
- `--allowedTools`
- `--disallowedTools`
- `--permission-mode`
- `--dangerously-skip-permissions`
- `--system-prompt`
- `--append-system-prompt`
- `--settings`
- `--setting-sources`
- `--mcp-config`
- `--plugin-dir`
- `--strict-mcp-config`
- `--worktree`
- `--tmux`
- `--ide`
- `--bare`
- `--sdk-url`
- `--teleport`
- `--cloud`
- `--remote`
- `--remote-control`
- `--rc`
- `--bg` / `--background`
- `--brief`
- `--safe-mode`

### `preAction`

统一初始化层，而不是每个命令各自初始化。已确认其至少会做：

- terminal title
- log sink
- migration
- plugin-dir 注入
- plugin-url / agents sideload policy 检查
- settings 同步
- remote settings / managed settings 处理
- policy settings / gateway settings 刷新
- dynamic MCP config / project config 检查

### 设计特点

这说明原工程非常强调：

- **命令层统一生命周期**
- 配置、日志、插件、远端设置在 action 前完成接线

---

## update / distribution 合同

当前包形态是 wrapper package + native binary + extracted Bun bundle，因此 `update` 入口不能按旧 `package/cli.js` 单文件发行方式理解。

已经能确认的本地合同：

- 单独 `--update` / `--upgrade` 会在最外层入口被改写成 `update` 子命令。
- `update` / `upgrade` 是主 Commander 命令树里的显式命令。
- `DISABLE_UPDATES` 是当前 bundle 可见的 update gate。
- `downloads.claude.ai` 是 release download 相关 host；发行下载地址不能继续写成旧 npm-only 口径。
- native binary staged package 会检查 binary 是否存在；缺失时有 `Native binary not found in staged package` / `Native binary not found` 这一类失败路径。
- lockless update 当前在 env/feature flag 面可见为 `ENABLE_LOCKLESS_UPDATES`，但具体启用条件仍应按 gate 处理，不能写成默认行为。

这组事实的边界是：

- `DISABLE_UPDATES` 只说明客户端 update 入口被 gate，不代表 npm wrapper、native optional package、download CDN 和 staged install 是同一层。
- `downloads.claude.ai` 只证明本地 bundle 有 release download host，不证明服务端 rollout 策略或版本投放规则。
- native binary 更新失败处理是发行版外壳行为，不属于 agent loop 或 TUI renderer。

因此恢复 update 逻辑时应分四层建模：

```text
wrapper CLI command
  -> update / upgrade dispatcher
  -> release download / staged package
  -> native binary presence and replacement
```

不要把它压成“npm install 新版本”或“bundle 自更新”。

## 运行模式：Headless / TUI / Bridge / Chrome / MCP

### 模式总览

可以明确分出：

1. **TUI 模式**：交互式终端 UI
2. **Headless 模式**：非交互/print/json/stream-json
3. **Bridge 模式**：remote-control / sdk-url / stream-json
4. **Chrome/Native Host 模式**
5. **MCP serve 模式**
6. **computer-use-mcp 模式**
7. **Daemon / background agents 模式**
8. **Agents fleet view 模式**

### Headless

Headless 由主 action 的非交互路径承担；旧文档中把这一段标成 `runHeadless()` / `cuz(...)` / `luz(...)`，入口层复核暂不把这些旧压缩名当作当前 2.1.197 的已确认命名。

其职责：

- 构造 IO bridge
- 初始化 sandbox
- 统一处理 continue/resume/teleport
- 构造工具集与 permission gate
- 根据 `text/json/stream-json` 输出结果

### TUI

TUI 最终会进入 React/Ink root 一类渲染入口；具体压缩名留给 TUI 专题复核。

典型入口：

- 新会话
- resume/continue/teleport 后进入主 App
- 仅 `--resume` 无 session 时进入 ResumeConversation 选择器

### Bridge

已确认支持：

- `--input-format stream-json`
- `--output-format stream-json`
- `--sdk-url`
- `remote-control` / `rc` / `remote` / `sync` / `bridge`

说明 Headless path 可以被外部前端/控制器嵌套调用，作为“子 agent 进程”或本地执行后端。

但 `remote-control`、`--remote`、`--sdk-url + stream-json` 三条入口的产品语义并不相同：

- `remote-control`
  - 把当前本地会话开放给远端接入
- `--remote`
  - 让 CLI 充当远端 session 前端
- `--sdk-url + stream-json`
  - 让 CLI 充当可嵌入的 agent backend

更细的工作流矩阵、bridge transport、远程 transcript 持久化与冲突还原，统一见：

- [../03-ecosystem/02-remote-persistence-and-bridge.md](../03-ecosystem/02-remote-persistence-and-bridge.md)

### Chrome / Native Host / computer-use

这几条都是启动时 fast-path，独立于主 CLI；因此可以被视为“旁路模式”。

### Daemon / Background Agents / Fleet View

当前入口层把 daemon 与 background agents 作为一等 fast-path 处理，而不是普通主 CLI 内部细节：

- `--daemon-worker` 是 daemon supervisor 派生 worker 的入口。
- `--bg-pty-host` 是 background PTY host 入口。
- `--bg-spare` 是 daemon spare worker 入口。
- `--preload` 是预加载入口。
- `daemon` 直接进入 daemon dispatcher。
- `logs | attach | stop | kill | respawn | rm` 和 `--bg | --background` 直接进入 background agents handler。
- `agents` positional / `defaultToAgentsView` 可以在主 Commander 之前 mount fleet view。

这些路径都在 `l3m()` 中先于 `main` import 执行，因此恢复或重写时应单独建模为启动层模式，而不是塞进 TUI 或 Headless 的分支里。

#### Background agents handler 边界

`l3m()` 命中 `logs / attach / stop / kill / respawn / rm` 或 `--bg / --background` 后，会先跑 fast-path policy，再通过 `ensureFleetGateHydrated()` 和 `isAgentsFleetEnabled()` 检查 agent view gate。gate 的本地可见 deny 来源包括：

- `CLAUDE_CODE_DISABLE_AGENT_VIEW`
- managed/local effective settings 里的 `disableAgentView`

通过 gate 后，入口动态加载 `(len(), Glc)` 这一组 background handlers。当前可直接确认的 handler 表包括：

- `logsHandler(id)`：按 job id/prefix 读取背景 session 的近期 terminal output。
- `attachHandler(id)`：要求 daemon 可达；遇到 `ENOJOB` 会尝试 wake session；遇到断连会尝试 transient daemon reconnect；对 respawning / stalled / exited 做专门错误提示。
- `stopHandler(id)`：停止背景 session，确认后把 job state 写成 `stopped`，并在 worktree 存在时提示 `claude rm <id>`。
- `respawnHandler(id | --all)`：让背景 session 使用当前 Claude binary 重启，支持单个 id/prefix 或 live jobs 全部重启。
- `rmHandler(id)`：删除背景 session 和 worktree；如果 worktree dirty、branch mismatch 或删除失败，会保留 worktree 并说明原因。
- `handleBgFlag(argv)`：处理 `--bg / --background`，支持 `--exec` shell job；非 TTY stdin 会在 3 秒窗口内读取，最多 1 MiB，随后拼入 positional prompt；再剥掉 bg flags、分配/管理 session id，并调用 `spawnBgSession(...)`。

背景 job 的本地状态围绕 `jobs/<short>/state.json` 维护。schema 里可见 `state / tempo / inFlight / respawnFlags / providerEnv / sessionPermissionRules / sessionId / cwd / worktreePath / backend` 等字段；持久化读写会做 schema 校验、大小限制、mtime cache 和 allowlist 清洗。也就是说，background agents 不是简单 `spawn` 一个子进程，而是“daemon dispatch + job state + attach/respawn/rm 生命周期”的组合系统。

#### Fleet view mount 边界

`agents` positional 或 `defaultToAgentsView` 命中时，`l3m()` 会在主 Commander 前提前进入 fleet view：

- 先 `startCapturingEarlyInput()`，避免主模块加载期间丢输入。
- 读取 global config 的 `defaultToAgentsView`，并处理 `--settings`、`--plugin-dir`、`--plugin-dir-no-mcp`。
- 跑 fast-path policy 与 sideload flag policy；命中组织策略时直接退出。
- 通过 `ensureFleetGateHydrated({ kickGrowthBook: false })` 和 `isAgentsFleetEnabled()` 检查 fleet gate；若是显式 `claude agents` 且 gate 关闭，走 `fleetGateRejected("claude agents")`。
- 创建 Ink root，设置 `interactive` 与 `sessionStartType = "agents_view"`，后台异步初始化 graceful shutdown、error log sink、analytics、1P event logging、GrowthBook、trust 后 telemetry 和 teammate snapshot。
- 将 `cwdFilter`、从 CLI flags 派生的 `dispatchExtraArgs`、以及 permission/model/effort/agent 这类 `dispatchDefaults` 传给 `mountFleetViewWithComposerBack(...)`。

因此 fleet view 是启动层的一等 TUI，而不是主 REPL 里普通的一个 dialog。它共享 settings、plugin、policy、telemetry 初始化链，但在 `main(l4m)` 之前完成 mount。

#### Agent View / background status 字段

`claude agents` 与 background agents 的可见状态不是只靠终端输出拼字符串。本地类型和持久化状态里能确认这些字段族：

| 字段族 | 语义 | 证据来源 |
| --- | --- | --- |
| `run_in_background` / `name` / `mode` / `isolation` | `Agent` tool 输入侧直接支持后台运行、命名、权限模式和 worktree / remote 隔离 | `package/sdk-tools.d.ts:431-449` |
| `state / tempo / inFlight / respawnFlags / providerEnv / sessionPermissionRules / sessionId / cwd / worktreePath / backend` | 本地 background job state 的核心字段 | `package/preprocessed/cli.extracted.bundle.pretty.js` background job schema |
| `waitingFor` | `claude agents --json` 只在 status 为 waiting 且存在等待原因时输出，来源可以是 permission prompt、worker request、sandbox request、dialog open 或 input needed | `package/preprocessed/cli.extracted.bundle.pretty.js:443306-443307`、`823745`、`866317-866336` |
| pinned notifications | Agent View 可把 background 通知 pin 到状态区，避免完成/等待类提醒被普通通知滚走 | `package/preprocessed/cli.extracted.bundle.pretty.js:211760-211823` |
| workflow task fields | workflow launch 结果会落 `taskType / workflowName / runId / transcriptDir / scriptPath / sessionUrl` | `package/sdk-tools.d.ts:3416-3445` |

因此重写时应把 Agent View 视作 background session / workflow / notification 的汇合面，而不是单独的 session 列表页。

---

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
