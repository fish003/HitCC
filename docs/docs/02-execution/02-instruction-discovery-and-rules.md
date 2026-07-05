# Instruction 发现与扫描规则

## 本页用途

- 作为 `Qv()`、`CLAUDE.md`、`.claude/rules/`、`systemPromptSectionCache` 与 `InstructionsLoaded` 主题的导航页。
- 原正文已经拆到四个子页；本页只维护阅读边界和相邻专题关系。

## 相关文件

- [01-tools-hooks-and-permissions.md](./01-tools-hooks-and-permissions.md)
- [03-prompt-assembly-and-context-layering.md](./03-prompt-assembly-and-context-layering.md)
- [04-non-main-thread-prompt-paths.md](./04-non-main-thread-prompt-paths.md)
- [../05-appendix/01-glossary.md](../05-appendix/01-glossary.md)

## 拆分后的主题边界

### Instruction source / Qv 扫描 / compat 边界

见：

- [02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md](02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md)

集中放 `Qv()` 主扫描来源、compat 文件、`.claude/rules/`、`@include` 与 `/init` 迁移输入边界。

### System section cache 与默认 section

见：

- [02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md](02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md)

集中放 `systemPromptSectionCache`、`zL(...)` section 名单、缓存粒度、内容边界与失效规则。

### InstructionsLoaded load reason 与 hook 时序

见：

- [02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md](02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md)

集中放 `O4t(...)`、`Tpp()`、`X5e(...)` 的 load reason setter / consumer / dispatcher 边界。

### userContext / systemContext / ClaudeMd 边界

见：

- [02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md](02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md)

集中放 `hS()`、`_H(...)`、`currentDate` 标记链、`cachedClaudeMdContent`、`gitStatus / perforceMode` 与 `claudeMdExcludes`。

## 建议阅读顺序

1. 先看 [01-sources-qv-scan-and-compat-boundaries.md](./02-instruction-discovery-and-rules/01-sources-qv-scan-and-compat-boundaries.md)，确认 instruction 来源与扫描顺序。
2. 再看 [02-system-section-cache-and-section-content.md](./02-instruction-discovery-and-rules/02-system-section-cache-and-section-content.md)，理解默认 system sections 与缓存层。
3. 然后看 [03-instructionsloaded-load-reason-and-hook-timing.md](./02-instruction-discovery-and-rules/03-instructionsloaded-load-reason-and-hook-timing.md)，把 instruction load hook 时序补齐。
4. 最后看 [04-user-system-context-and-claude-md-boundaries.md](./02-instruction-discovery-and-rules/04-user-system-context-and-claude-md-boundaries.md)，区分 userContext、systemContext、auto-mode classifier 旁路与 exclude 规则。

## 与其它专题的边界

- 最终 API payload 的 system/messages 排序与 prompt cache 边界，见 [03-prompt-assembly-and-context-layering.md](./03-prompt-assembly-and-context-layering.md)。
- Hook schema、工具权限阶段与 hook 执行器，见 [01-tools-hooks-and-permissions.md](./01-tools-hooks-and-permissions.md)。
- 非主线程 prompt 路径中的 compat agent definitions 与 instruction 入口，见 [04-non-main-thread-prompt-paths.md](./04-non-main-thread-prompt-paths.md)。

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
