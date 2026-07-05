# 证据索引：入口、规则与 request snapshot 速查

## 证据口径

证据来源、置信度标签与“已确认 / 高可信推断 / 待验证”的统一口径，统一以：

- [../../00-overview/01-scope-and-evidence.md](../../00-overview/01-scope-and-evidence.md)

为准，本页不再重复维护一套平行定义。

仍需外部补证、但会影响重写判断的未知点，统一以：

- [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)

作为入口。

本页只负责：

- 把主题文档和直接证据来源对应起来
- 在新增主题页后补登记对应证据入口
- 给强结论维护“断言 -> 直接证据点 -> 反证/边界 -> 校对入口”的硬索引

## 当前发行版入口证据

当前 `@anthropic-ai/claude-code@2.1.197` 的入口证据先按 package 形态分层校对：

- wrapper package：
  - `package/package.json`：`bin.claude` 指向 `bin/claude.exe`，`optionalDependencies` 列出各平台 native package，`files` 包含 `bin/claude.exe`、`install.cjs`、`cli-wrapper.cjs`、`sdk-tools.d.ts`。
  - `package/install.cjs`：postinstall 根据平台解析 optional native package，并把 native binary 放到 `bin/claude.exe`。
  - `package/cli-wrapper.cjs`：postinstall 没有放置 binary 时的手动 fallback，通过 Node 解析 native package 后 spawn binary。
- native package：
  - `package/preprocessed/native/package/package.json`：当前 linux-x64 native package 名为 `@anthropic-ai/claude-code-linux-x64`，版本为 `2.1.197`，文件入口是 `claude`。
  - `package/preprocessed/native/package/claude`：当前 native executable；`package/preprocessed/reports/file.txt` 标记为 x86-64 ELF executable。
- `.bun` section 与抽取 bundle：
  - `package/preprocessed/reports/readelf-sections.txt`：可见 `.bun` section。
  - `package/preprocessed/cli.extracted.bundle.js`：从 native executable `.bun` section 抽取出的 bundle。
  - `package/preprocessed/cli.extracted.bundle.pretty.js`：同一 bundle 的格式化版本，是人工阅读主入口。
  - `package/preprocessed/reports/ascii-spans.top80.json` 与 `package/preprocessed/cli.extracted.bundle.pretty.js` 开头均可见 `@bun @bytecode @bun-cjs`。
  - `package/preprocessed/cli.extracted.bundle.pretty.js` 开头包含 `Version: 2.1.197`。
  - `package/preprocessed/reports/symbol-index.json` 与 pretty bundle 中可定位当前顶层入口 `async function l3m()` 及末尾 `l3m();`。
- 路径边界：
  - `package/preprocessed/reports/bunfs-paths.txt` 的 `file:///$bunfs/root/src/entrypoints/cli.js` 是 Bun 内嵌文件系统路径线索。
  - 当前 wrapper package 下不存在 `package/cli.js` 或 `package/cli.readable.js`；正文应直接写“抽取 bundle”或 `package/preprocessed/cli.extracted.bundle.pretty.js`。
- 辅助报告：
  - `package/preprocessed/native/` 和 `package/preprocessed/reports/` 只作为 native 包、字符串、符号、section 与 keyword 辅助证据，不按手写源码目录解读。

## 断言登记规则

后续如果正文里出现强结论，优先在对应 evidence map 子页登记成下面这种结构，而不是只补一个“相关主题页”链接：

- 断言：
  - 只写可被校对的判断，不写泛化口号
- 直接证据点：
  - 优先登记 A 级证据对应的函数、分支、schema、帮助输出或文件
  - 若正文页已经把 bundle 命中点整理好，这里直接引用正文页与关键符号
- 交叉佐证：
  - 只放支撑断言稳定性的 B 级归纳，不拿它冒充直接证据
- 反证/边界：
  - 明确写出当前不能证明什么、哪些路径只看到负证、哪些部分仍依赖 bundle 外或服务端黑箱
- 校对入口：
  - 给出复核时应先看的正文页，避免重复扫 bundle

如果一个结论暂时只能写出“主题页导航”，还写不出“直接证据点/反证边界”，说明它还不该被当成强断言。

## 上下文缓存 / request snapshot 专题速查

这一组是当前“上下文缓存”与“可复用 request snapshot”文档线的直达索引。

### 主题页索引

- [../../01-runtime/04-agent-loop-and-compaction.md](../../01-runtime/04-agent-loop-and-compaction.md)
  - 专题导航；具体直达 `01-main-loop-state-caches-and-yield-surface`、`02-compaction-pipeline-and-auto-compact-tracking`、`03-no-tool-branch-recovery-stop-and-reactive-compact`、`04-tool-round-next-turn-and-terminal-reasons`
- [../../02-execution/02-instruction-discovery-and-rules.md](../../02-execution/02-instruction-discovery-and-rules.md)
  - 负责 `systemPromptSectionCache`、`O4t/Tpp`、`hS()/_H(...)`、section 变化条件
- [../../02-execution/03-prompt-assembly-and-context-layering.md](../../02-execution/03-prompt-assembly-and-context-layering.md)
  - 专题导航；具体直达 `01-system-chain-default-sections-and-context-sources`、`03-api-payload-order-prompt-caching-and-final-boundary`
- [../../02-execution/04-non-main-thread-prompt-paths.md](../../02-execution/04-non-main-thread-prompt-paths.md)
  - 专题导航；具体直达 `01-shared-merge-skeleton-and-overrides`、`02-hook-and-compact-special-paths`、`03-fork-family-cache-safe-params-and-snapshot-reuse`

### 关键函数索引

- section cache：
  - `Lq8 / Vc8 / Ec8 / kHq / Yn`
- default system sections：
  - `zL / fT8 / R8_ / S8_ / C8_ / d8_ / i8_ / h8_ / c8_`
- 主线程 request build / prompt cache：
  - `Hpc / Flm / Ak / Glm / Wlm / W9o`
- instruction load reason：
  - `O4t / Tpp / X5e`
- request snapshot 生命周期：
  - `ML / Cj4 / xe6 / I1z / lZ / ts6`
- compact 与失效：
  - `oRf / deps.autocompact / FOo / SHe / Z$o / G$o`

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
