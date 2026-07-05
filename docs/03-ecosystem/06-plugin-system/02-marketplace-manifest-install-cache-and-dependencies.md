# Marketplace、manifest、安装缓存与依赖闭包

## 本页用途

- 整理 marketplace schema、plugin manifest、strict/compat 组合、versioned cache 与 source 类型。
- 说明 dependency resolution 为什么是硬约束，以及安装/缓存路径如何分层。

## 相关文件

- [../04-mcp-system.md](../04-mcp-system.md)
- [../05-skill-system.md](../05-skill-system.md)
- [../../01-runtime/01-product-cli-and-modes.md](../../01-runtime/01-product-cli-and-modes.md)
- [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)

## marketplace 不是简单仓库地址，而是正式 schema

marketplace schema 当前已能直接确认至少包括：

- `name`
- `owner`
- `plugins[]`
- `forceRemoveDeletedPlugins`
- `metadata`
- `allowCrossMarketplaceDependenciesOn`

source 类型也已比较完整：

- `github`
- `git`
- `url`
- `file`
- `directory`
- `npm`
- `settings`
- policy only:
  - `hostPattern`
  - `pathPattern`

其中几个边界很关键：

- `settings` source 不是远端拉取，而是把 settings 内联 marketplace 写成 synthetic `marketplace.json`
- `inline` 名字保留给 `--plugin-dir`
- `builtin` 名字保留给内建插件
- 一批官方 marketplace 名字被保留，并要求 source 必须来自 `anthropics/*`

因此 marketplace 不是“下载一份列表”那么简单，而是**受命名规则、source 类型、enterprise policy 共同约束的分发目录**。

## plugin manifest 比 skills/mcp 页面里的“plugin skill”要大得多

plugin manifest schema 当前已能直接确认至少覆盖：

- `name`
- `version`
- `description`
- `author`
- `homepage`
- `repository`
- `license`
- `keywords`
- `dependencies`
- `commands`
- `agents`
- `skills`
- `outputStyles`
- `hooks`
- `mcpServers`
- `lspServers`
- `channels`
- `userConfig`
- `settings`
- `defaultEnabled`
- `monitors`
- `binaries`
- marketplace source `skipLfs`

因此 plugin 更稳的定义应是：

- **一组可被统一分发、安装、启停、配置与治理的扩展能力包**

而不是：

- skill 的目录
- MCP 的包装壳
- 单独一份 hooks 配置

当前 2.1.197 可见的新增 manifest / marketplace 能力可以按下表落地，证据来自 `package/preprocessed/cli.extracted.bundle.pretty.js:64314-64876`：

| 能力 | 字段 | 当前语义 |
| --- | --- | --- |
| 默认启用 | `defaultEnabled` | 用户没有显式 enable/disable 记录时是否默认启用；显式 `enabledPlugins` 与被 dependency 要求启用仍优先 |
| 依赖闭包 | `dependencies` | 插件依赖必须启用；裸名按声明插件自己的 marketplace resolve |
| persistent monitors | `monitors` / `monitors/monitors.json` | 声明 session 生命周期内的后台 monitor；stdout line 会作为 `<task_notification>` 送回模型，信任层级接近 hooks |
| 二进制分发 | `binaries` | 安装时把 sha256-pinned 文件抓取到 `bin/`，key 是目标 basename |
| marketplace sparse / LFS 控制 | `sparsePaths` / `skipLfs` | GitHub/git source 可 sparse checkout；`skipLfs` 设置 `GIT_LFS_SKIP_SMUDGE=1`，避免下载大 LFS 内容 |
| source 形态 | `url / github / git / npm / file / directory / settings` | source resolution 是 marketplace 级 schema，不是单一 URL 字符串 |

## manifest 路径与兼容模式

当前读取路径至少有两层：

- 首选：`.claude-plugin/plugin.json`
- 兼容：根目录 `plugin.json`

而且 manifest 之外还存在一套默认目录约定：

- `commands/`
- `agents/`
- `skills/`
- `output-styles/`
- `hooks/hooks.json`

也就是说，plugin 即使不在 manifest 里显式列出所有组件，也可能通过默认目录被自动装载。

## marketplace entry 与 plugin.json 不是简单二选一

`CJ4(...)` 当前已经能把两者的关系拆成三种：

### 1. 只有 marketplace entry

- 允许直接由 marketplace entry 提供 `commands/agents/skills/hooks/outputStyles`
- 即使 plugin 根里没有 `.claude-plugin/plugin.json` 也能装起来

### 2. 有 plugin.json，且 marketplace entry 保持默认 / 显式 `strict: true`

- 两边的组件定义会做增量合并
- marketplace entry 可继续补充 paths / hooks / metadata

### 3. 有 plugin.json，且 marketplace entry `strict: false` 同时再声明组件

- 会被判成冲突
- 文案直接提示这是 conflicting manifests
- 报错还会明确提示：
  - 要么把 marketplace entry 改成 `strict: true`
  - 要么删除其中一侧的组件声明

因此 plugin 的真实语义不是“manifest 只有一份”，而是：

- **plugin.json 是主 manifest**
- **marketplace entry 在某些模式下还能当 overlay / fallback manifest**

## 安装与缓存不是平铺目录，而是 versioned cache

plugin 安装路径当前已能直接确认至少经过：

1. 解析 source
2. 拉取或复制到临时目录
3. 解析 manifest
4. 计算 version
5. 物化到 versioned cache
6. 写入 `installed_plugins.json`

version 的优先顺序也已经可写实：

- manifest `version`
- 调用方提供的 version
- git SHA / pre-resolved SHA
- `unknown`

本地缓存形状则至少包括：

- `~/.claude/plugins/cache/...`
- 可选 zip cache
- `npm-cache`
- plugin data dir：`~/.claude/plugins/data/{id}/`

这说明 plugin 的安装结果不是“把仓库 clone 到某个目录”，而是**带版本名、可迁移、可清理、可回退的本地缓存对象**。

## source 类型已经超出“本地目录 + GitHub”

plugin source 当前已能直接确认包括：

- 相对路径字符串
- `github`
- `url`
- `git-subdir`
- `npm`
- `pip`

但实现状态并不完全对称：

- `github / url / git-subdir / 相对路径` 已有明确装载链
- `npm` 已有安装到本地 cache 的实现
- `pip` schema 已存在，但当前直接报 `not yet supported`

因此 schema 支持面大于当前完整实现面，这点在重写时要保留。

更具体一点：

- `pip` 不是“暂时没走到”。
- `se6(...)` 在 source switch 里有明确 `case "pip"`，但分支主体直接抛：
  - `Python package plugins are not yet supported`
- UI 安装详情页仍会把 `pip` 视为 remote plugin source，与 `github/url/npm` 一样显示“组件摘要需安装后发现”

因此当前更稳的结论不是“pip 可能有隐藏实现”，而是：

- **schema / UI 已知晓 `pip`**
- **实际安装器仍未落地**

## 依赖解析不是建议，而是硬约束

plugin 依赖链当前已经闭环到安装与运行时两层：

### 安装期

- `Lt1(...)` 会先做 dependency closure 解析
- 再把整条闭包一起写入 `enabledPlugins`
- 然后按闭包顺序安装依赖

### 运行期

- `JJ4(...)` 会复检 enabled plugin 的依赖是否满足
- 若缺依赖、依赖未启用、跨 marketplace 不被允许：
  - 当前 plugin 会被 demote 成 disabled
  - 并生成 `dependency-unsatisfied` 错误

跨 marketplace 依赖还有一条硬边界：

- 只有**根 marketplace** 的 `allowCrossMarketplaceDependenciesOn` 生效
- 不存在“传递信任”

因此 plugin dependency 不是 UI 提示，而是**正式的启用前提与运行时 gate**。

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
