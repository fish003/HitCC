# Validate、管理 UI 与安全边界

## 本页用途

- 整理 `plugin validate`、`/plugin` 管理 UI、下架/锁定/telemetry 处理与当前未钉死点。
- 保留 plugin 子系统可重写边界和剩余风险。

## 相关文件

- [../04-mcp-system.md](../04-mcp-system.md)
- [../05-skill-system.md](../05-skill-system.md)
- [../../01-runtime/01-product-cli-and-modes.md](../../01-runtime/01-product-cli-and-modes.md)
- [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)

## `plugin validate` 已基本闭环

`plugin validate` 现在已经可以写到输出格式和分支差异级别。

### 输入分流

- 不带参数时直接输出 usage/example 文案
- 目录输入时优先检查：
  - `.claude-plugin/marketplace.json`
  - 若不存在，再查 `.claude-plugin/plugin.json`
- 普通文件输入时：
  - 文件名是 `marketplace.json` -> marketplace 校验
  - 文件名是 `plugin.json` -> plugin 校验
  - 其他文件会先尝试按 marketplace 形状判断：
    - 若顶层存在 `plugins[]`，按 marketplace
    - 否则按 plugin

### CLI 输出样式

CLI handler `Mmz(...)` 的固定输出形状是：

```text
Validating <fileType> manifest: <filePath>

✗ Found N errors:
  • <path>: <message>

⚠ Found N warnings:
  • <path>: <message>

✓ Validation passed
```

退出码边界也已明确：

- 成功：`0`
- 校验失败：`1`
- 意外异常：`2`

### plugin manifest 校验

`w8A(...)` 当前至少会检查：

- 文件是否存在、是否可读、是否是合法 JSON
- `e36().strict()` schema
- `commands/agents/skills` 里的字符串路径是否包含 `..`
- `category/source/tags/strict/id` 这类 marketplace 字段是否误写进 `plugin.json`
  - 这类情况只报 warning
  - 文案明确说明运行时会忽略
- `name` 是否 kebab-case
  - 当前 CLI 接受
  - 但 claude.ai marketplace sync 要求 kebab-case
- 是否缺 `version/description/author`

### marketplace manifest 校验

`$8A(...)` 当前至少会检查：

- 文件存在性 / JSON 语法
- `Ve().extend({ plugins: R.array(jj1().strict()) }).strict()` schema
- 每个 plugin entry 的 `source`/`source.path` 是否包含 `..`
- 相对路径 source 的告警文案会明确提示：
  - 路径是相对 marketplace root
  - 不是相对 `marketplace.json` 自身
- marketplace 内是否存在重复 plugin name
- 本地 `./...` plugin entry 的 `version` 是否与目标 plugin 的 `.claude-plugin/plugin.json` 一致
  - 不一致只报 warning
  - 文案明确写出 install 时 **plugin.json version 优先**
- `metadata.description` 缺失 warning

### 目录内附属内容校验

若校验目标是 `.claude-plugin/plugin.json`，CLI 还会继续跑 `AG4(...)`：

- 递归检查 `skills/` 下的 `SKILL.md`
- 递归检查 `agents/`、`commands/` 下的 markdown 文件
- 单独检查 `hooks/hooks.json`

其中 markdown frontmatter 校验 `Z9z(...)` 至少覆盖：

- 是否存在 frontmatter
- YAML 是否能解析
- frontmatter 顶层是否是 mapping
- `description`
- `name`
- `allowed-tools`
- `shell`

并且文案会明确区分两类后果：

- 某些字段类型错：
  - runtime 会 silent drop
- hooks JSON 错：
  - runtime 会直接破坏整个 plugin load

因此 `plugin validate` 这一块已经不只是“能校验 schema”，而是：

- **manifest 识别**
- **plugin / marketplace 差异化检查**
- **附属 markdown / hooks 内容检查**
- **明确的 CLI 文本输出与退出码**

## `/plugin` 管理 UI 已能拆出明确状态机

本地 bundle 已经能把 `/plugin` overlay 的主状态写实成一组 view state，而不只是“有个 plugin 菜单”。

### 主列表态

管理面主列表当前会把条目混排成三类：

- plugin
- child MCP
- failed plugin / flagged plugin

并按 scope 分组显示：

- `flagged`
- `project`
- `local`
- `user`
- `enterprise`
- `managed`
- `builtin`
- `dynamic`

同时支持：

- `/` 或直接打字进入搜索
- `Space` toggle
- `Enter` 看详情
- `Esc` 返回

### 详情/子页态

当前已能直接确认的 state 包括：

- `plugin-list`
- `plugin-details`
- `plugin-options`
- `configuring-options`
- `configuring`
- `failed-plugin-details`
- `flagged-detail`
- `confirm-project-uninstall`
- `confirm-data-cleanup`
- `mcp-detail`
- `mcp-tools`
- `mcp-tool-detail`

### 已确认的交互分支

- plugin 明细页可进入：
  - enable
  - disable
  - update
  - uninstall
  - configure
- 若插件带 `userConfig`，启用后会进入 `plugin-options / configuring-options`
- 若插件的 MCP 配置需要单独走 `MCPB` 文件，则会进入 `configuring`
- 若 project scope 卸载会影响团队共享配置，会先进入 `confirm-project-uninstall`
- 若 data dir 非空，会进入 `confirm-data-cleanup`
- failed plugin 详情页提供 remove
- flagged plugin 详情页提供 dismiss
- child MCP 可以继续下钻到：
  - `mcp-detail`
  - `mcp-tools`
  - `mcp-tool-detail`

因此 `/plugin` 不是单页列表，而是：

- **一个把 plugin、plugin-sourced MCP、failed state、flagged state、config UI 串起来的 overlay 状态机**

## 安全与下架处理也是 plugin 子系统的一部分

plugin 当前还有一条明确的安全治理链：

- `blocklist.json`
- `flagged-plugins.json`
- 官方 security feed
- marketplace 的 `forceRemoveDeletedPlugins`

行为至少包括：

- 拉取官方安全消息并本地缓存
- 对被 marketplace 删除、且要求强制移除的 plugin 自动卸载
- 把已下架/已标记 plugin 记入 flagged 状态
- `/plugin` UI 中可单独展示 flagged plugin

因此 plugin 系统并不只负责“安装更多扩展”，还负责**下架、封禁、清理与用户提示**。

## 更稳的工程结论

基于当前本地 bundle，plugin 子系统已经可以收敛成：

1. 有独立命令树与 marketplace 子命令树。
2. 有 marketplace 声明层、安装记录层、enabledPlugins 选择层三层状态。
3. 有 `cacheOnly` 与全量安装两套 runtime 装配路径。
4. 有 session inline、marketplace、builtin 三类 source 与明确覆盖优先级。
5. 有 plugin manifest、marketplace entry，以及“`strict: true` 可 overlay、`strict: false` 会在双声明时冲突”的组合规则。
6. 有 versioned cache、zip cache、plugin data dir 与 installed ledger。
7. 有 dependency closure 安装与运行时 dependency demotion。
8. 能向 commands、skills、agents、hooks、MCP、LSP、settings 注入能力。
9. 自带 userConfig、channel、security/delist 这些治理层能力。

## 当前仍未完全钉死

- builtin plugin 的 `hno -> yno/zXi/VXi` consumer 侧已清楚；当前 readable bundle 中 `hno` 初始化为空且未见 `set` producer，因此可见 entry set 为空。bundle 外或灰度路径是否会追加条目仍无法正证。
- `/plugin` overlay 的主状态与主要交互已能写实，但个别组件内部展示细节还没必要继续穷举。
- `pip` source 的“不对称支持”已经明确，但未来版本是否会补上安装器，本地 bundle 无法判断。

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
