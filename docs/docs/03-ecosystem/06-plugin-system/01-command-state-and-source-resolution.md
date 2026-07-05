# 命令面、状态层与 source resolution

## 本页用途

- 整理 plugin 命令树、marketplace/install/enable 三层状态、runtime 入口与 source precedence。
- 说明 session plugin、installed plugin 与 builtin plugin 进入运行时物化链前的边界。

## 相关文件

- [../04-mcp-system.md](../04-mcp-system.md)
- [../05-skill-system.md](../05-skill-system.md)
- [../../01-runtime/01-product-cli-and-modes.md](../../01-runtime/01-product-cli-and-modes.md)
- [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)

## 一句话结论

plugin 在 Claude Code CLI 中不是“附带几条 skill 的目录”，而是贯穿 **marketplace 分发、安装缓存、settings 启停、运行时装配、能力注入、策略管控** 的扩展控制面。

---

## 结构总览

当前更稳的 plugin 分层应写成：

```text
CLI / management layer
  -> plugin validate/list/install/uninstall/enable/disable/update
  -> plugin marketplace add/list/remove/update

marketplace / distribution layer
  -> known_marketplaces.json
  -> marketplace cache / refresh / seed / official fallback
  -> plugin entry source resolution

installation / state layer
  -> installed_plugins.json
  -> enabledPlugins across managed/user/project/local/flag
  -> versioned cache / zip cache / plugin data dir

runtime materialization layer
  -> session --plugin-dir
  -> marketplace plugins
  -> builtin plugins
  -> dependency demotion / managed blocking / precedence merge

capability injection layer
  -> commands / skills / agents / output styles / hooks
  -> mcpServers / lspServers / settings / channels / userConfig
```

## 命令面已经能单独成系统

`plugin` 命令树当前已能直接确认包括：

- `validate`
- `list`
- `install`
- `uninstall` / `remove` / `rm`
- `enable`
- `disable`
- `update`
- `marketplace add`
- `marketplace list`
- `marketplace remove`
- `marketplace update`

其中有几个边界很关键：

- `validate <path>` 明确是校验 **plugin 或 marketplace manifest**
- `list --available --json` 会同时列出“已安装”和“marketplace 中可安装”的插件
- `uninstall --keep-data` 说明 plugin 还有独立持久化数据目录
- `marketplace` 子树证明 plugin 的正式分发单位不是单个 repo，而是 **marketplace**

因此 CLI 对 plugin 的定位不是“扫描本地扩展目录”，而是**内建完整安装与分发管理面**。

## plugin 至少有三层状态，而不是一张表

当前本地 bundle 已能把 plugin 状态拆成三层：

### 1. marketplace 声明层

- `known_marketplaces.json`
- `marketplaces/<name>.json` 或 marketplace 目录缓存
- seed marketplace / official marketplace / settings-sourced marketplace

这层解决的是：

- 从哪里发现 plugin
- 每个 marketplace 的 source 是什么
- 本地 cache 安装位置在哪里
- 是否允许 autoUpdate

### 2. 安装记录层

- `installed_plugins.json`
- 记录 `pluginId -> [installations...]`
- 每条 installation 至少带：
  - `scope`
  - `installPath`
  - `version`
  - `installedAt`
  - `lastUpdated`
  - `gitCommitSha`
  - `projectPath`（非 user scope 时）

这层解决的是：

- 哪些 plugin 已经物化到本地磁盘
- 不同 scope 是否有各自安装副本
- 当前缓存路径和版本是什么

### 3. 启停选择层

- `settings.json` 的 `enabledPlugins`
- 来源至少包括：
  - `policySettings`
  - `userSettings`
  - `projectSettings`
  - `localSettings`
  - `flagSettings`

这层解决的是：

- 哪些已安装 plugin 当前应视为 enabled
- scope 优先级如何覆盖
- builtin plugin 是否被显式关闭

因此更稳的理解不是“plugin 安装后就生效”，而是：

```text
marketplace declaration
-> installation ledger
-> enabledPlugins decision
-> runtime load
```

## runtime 真实入口：`AM / mD / bJ4 / SJ4`

plugin 运行时装配主链已经比较清楚：

- `SJ4({ cacheOnly })`
  - 从 `enabledPlugins + flag settings` 收集目标 plugin id
  - 先按 marketplace / policy / cache 解析每个 plugin
- `bJ4(loadMarketplacePlugins)`
  - 再并入 `--plugin-dir` session plugins
  - 再并入 builtin plugins
  - 再做 managed blocking、session override、dependency demotion
- `AM()`
  - 走 `SJ4({ cacheOnly: false })`
  - 允许下载/缓存真实 plugin
- `mD()`
  - 默认走 `SJ4({ cacheOnly: true })`
  - 只使用现有 cache
  - 若 `CLAUDE_CODE_SYNC_PLUGIN_INSTALL` 开启则直接退回 `AM()`

这说明 plugin 在启动期至少分成两种装配模式：

- **cache-only 启动路径**
- **允许同步安装的全量路径**

## source precedence：session plugin 会盖掉已安装 plugin

`qe_ / Ke_ / bJ4` 这一组当前已经能把优先级收紧为：

```text
session --plugin-dir plugins
-> installed / marketplace-backed plugins
-> builtin plugins
```

更具体地说：

- `--plugin-dir` 产物会被标成 `name@inline`
- builtin plugin 会被标成 `name@builtin`
- 若 session plugin 与已安装 plugin 同名：
  - session plugin 覆盖 marketplace-backed 版本
- 若 managed settings 锁住该插件名：
  - `--plugin-dir` 同名副本会被直接挡掉

因此 `--plugin-dir` 不是简单“额外追加”，而是**session scope overlay**。

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
