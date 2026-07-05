# 运行时注入、settings 与 channels

## 本页用途

- 整理 scope-aware 启停、commands/skills/agents/hooks/MCP/LSP/settings/channels 注入边界。
- 说明 plugin 与 skills、MCP、userConfig、builtin plugin 的接线关系。

## 相关文件

- [../04-mcp-system.md](../04-mcp-system.md)
- [../05-skill-system.md](../05-skill-system.md)
- [../../01-runtime/01-product-cli-and-modes.md](../../01-runtime/01-product-cli-and-modes.md)
- [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)

## 启停规则是 scope-aware 的

plugin enable/disable/uninstall 当前不是只改一个布尔值，而是带 scope 语义：

- `user`
- `project`
- `local`
- `managed`
- `flag`
- builtin 走 `userSettings`

其中几个行为已经能直接写实：

- `enable/disable` 若不指定 scope，会自动探测已有可编辑 scope
- `uninstall` 会校验当前安装副本是否真的在目标 scope
- 若插件在 project scope 启用，用户想“只对自己关闭”，会被引导改用 local scope
- `disable --all` 是独立批量路径

因此 plugin scope 不是安装时附带标签，而是**设置层与磁盘层共同参与的状态维度**。

## plugin 会向运行时注入多类能力

基于 `RJ4 / G26 / zt1 / K68 / mF / ZA6 / bt6`，plugin 当前至少可注入：

- commands
- skills
- agents
- output styles
- hooks
- MCP servers
- LSP servers
- settings overlay

其中：

- commands / skills / agents / hooks / MCP / LSP 都有独立 loader
- `plugin settings` 会在 enabled plugin 集合上做 merge
- 若多个 plugin 覆盖同一 setting key，日志会记 override

因此 plugin 是真正的**多能力装配容器**。

## 与 skills 的接线点

plugin 对 skills 的影响当前至少分两层：

- plugin commands：进入 `jKe()`
- plugin skills：进入 `X1o()`

然后在 skills 总注册表里，顺序可确认是：

```text
skill dir commands
-> workflow commands
-> plugin commands
-> plugin skills
-> bundled skills
-> builtin plugin skills
-> built-in local command set
```

因此 plugin 不只是“给 skill 提供额外目录”，而是直接占据 skills registry 的两个独立注入槽位。  
skills 更细执行与公告边界见 [05-skill-system.md](../05-skill-system.md)。

## 与 MCP 的接线点

plugin manifest 中的 `mcpServers`、`channels`、`userConfig` 共同构成了 plugin->MCP 的装配桥：

- plugin 可直接携带 MCP server config
- plugin 可声明 channel 绑定到某个 MCP server
- channel 注册要求该 server 来自 **plugin-sourced 且 marketplace 可识别** 的插件
- plugin 的 userConfig 会参与 `${user_config.KEY}` 替换

因此 plugin 在 MCP 侧不是“发现来源”这么简单，而是：

- **MCP server producer**
- **channel capability carrier**
- **MCP 配置 UI 的上游**

MCP 连接、resource、channel push 更细行为见 [04-mcp-system.md](../04-mcp-system.md)。

## `userConfig` 与 plugin settings 说明 plugin 自带配置面

plugin 配置当前至少有两条线：

### 1. `userConfig`

- manifest 可声明用户可配置项
- sensitive 值不会和普通值走同一持久化位置
- 非敏感值可注入 skills / agents / hooks / MCP env

### 2. `settings`

- plugin 可直接提供 allowlisted settings overlay
- enabled plugin 集合最终会把这些设置 merge 进运行态

这说明 plugin 不只是“静态能力包”，而是**带配置 schema 的扩展模块**。

## channels 证明 plugin 还承担消息入口角色

`channels` 字段当前至少能确认：

- 每个 channel 绑定某个 plugin 内的 MCP server
- 可附带 channel 级 `userConfig`
- 运行时 `channel_enable` 会验证该 server 是否真来自 marketplace plugin
- 通过验证后，channel notification 会被回注成 queued prompt

因此某些 plugin 的本质不是“给模型多一个工具”，而是**提供一条入站消息通道**。

## builtin plugin 不是普通已安装 plugin

builtin plugin 当前已有独立 sentinel：

- source：`name@builtin`
- marketplace：`builtin`

其特征至少包括：

- 来自内存里的 `hno` 注册表
- 不走 marketplace 下载
- 不走普通 uninstall/update
- 可以通过 `enabledPlugins` 显式 enable/disable
- 还能额外产出 builtin plugin skills

从 `yno / zXi / VXi` 反推，内建 registry 条目至少可带：

- `description`
- `version`
- `defaultEnabled`
- `isAvailable`
- `hooks`
- `mcpServers`
- `skills`

因此 builtin plugin 更像：

- **内建扩展包 registry**

而不是：

- 放在某个 cache 目录里的普通插件

但这里已经能把“不确定点”继续收紧：

- `hno` 的消费侧非常清楚：
  - `yno()` 把条目转成 `name@builtin`
  - `zXi()` 从启用的 builtin 条目里再抽 `skills`
  - `VXi(name)` 提供按名读取
- `hno` 的初始化侧在本地 bundle 中只看到：
  - `Sct()` 里 `hno = new Map()`
- 对整份 bundle 继续追踪，没有找到：
  - `hno.set(...)`
  - `hno = new Map([...])`
  - 其他显式 producer

因此本地发行版能正证的是：

- **builtin plugin 机制存在且消费链完整**
- **当前 readable bundle 中的 builtin plugin entry set 可见为空**
- **builtin plugin skills 已作为 skills registry 的一个输入槽位存在，但当前槽位没有可见条目**

但还不能正证的是：

- **bundle 外、灰度或裁剪路径是否会注册额外 builtin plugin 条目**

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
