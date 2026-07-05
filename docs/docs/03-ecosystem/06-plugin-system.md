# Plugin 系统

## 本页用途

- 作为 plugin marketplace、安装缓存、启停状态、运行时注入与管理 UI 的导航页。
- 原正文已经拆成四个子页；本页只保留主题地图和相邻系统边界。

## 相关文件

- [04-mcp-system.md](./04-mcp-system.md)
- [05-skill-system.md](./05-skill-system.md)
- [../01-runtime/01-product-cli-and-modes.md](../01-runtime/01-product-cli-and-modes.md)
- [../02-execution/02-instruction-discovery-and-rules.md](../02-execution/02-instruction-discovery-and-rules.md)

## 拆分后的主题边界

### 命令面、状态层与 source resolution

见：

- [06-plugin-system/01-command-state-and-source-resolution.md](06-plugin-system/01-command-state-and-source-resolution.md)

集中放 plugin 命令树、marketplace/install/enable 三层状态、runtime 入口与 source precedence。

### Marketplace、manifest、安装缓存与依赖闭包

见：

- [06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md](06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md)

集中放 marketplace schema、plugin manifest、compat/strict 组合、versioned cache、source 类型与 dependency resolution。

### 运行时注入、settings 与 channels

见：

- [06-plugin-system/03-runtime-injection-settings-and-channels.md](06-plugin-system/03-runtime-injection-settings-and-channels.md)

集中放启停规则、能力注入、skills/MCP/settings/channels/userConfig 与 builtin plugin 边界。

### Validate、管理 UI 与安全边界

见：

- [06-plugin-system/04-validation-ui-and-safety-boundaries.md](06-plugin-system/04-validation-ui-and-safety-boundaries.md)

集中放 `plugin validate`、`/plugin` UI 状态机、安全/下架处理、结论与未决点。

## 建议阅读顺序

1. 先看 [01-command-state-and-source-resolution.md](./06-plugin-system/01-command-state-and-source-resolution.md)，建立命令面和状态层模型。
2. 再看 [02-marketplace-manifest-install-cache-and-dependencies.md](./06-plugin-system/02-marketplace-manifest-install-cache-and-dependencies.md)，确认分发、manifest 和本地物化边界。
3. 然后看 [03-runtime-injection-settings-and-channels.md](./06-plugin-system/03-runtime-injection-settings-and-channels.md)，理解运行时注入面。
4. 最后看 [04-validation-ui-and-safety-boundaries.md](./06-plugin-system/04-validation-ui-and-safety-boundaries.md)，复核 validate、管理 UI 与安全边界。

## 与其它专题的边界

- MCP server/tool/resource 的连接和协议层，见 [04-mcp-system.md](./04-mcp-system.md)。
- SKILL.md loader、runtime skill listing 与 SkillTool 执行，见 [05-skill-system.md](./05-skill-system.md)。
- plugin 命令入口在 CLI 命令树里的位置，见 [../01-runtime/01-product-cli-and-modes.md](../01-runtime/01-product-cli-and-modes.md)。

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
