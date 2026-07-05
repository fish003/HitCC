# 范围、证据与结论

## 本页用途

- 用来界定这套重建文档“已经知道什么”和“明确还不知道什么”。
- 用来统一证据等级与判断口径，避免后续补证据时混淆“已确认”和“高可信推断”。

## 相关文件

- [00-index.md](./00-index.md)
- [../05-appendix/01-glossary.md](../05-appendix/01-glossary.md)
- [../05-appendix/02-evidence-map.md](../05-appendix/02-evidence-map.md)
- [../04-rewrite/02-open-questions-and-judgment.md](../04-rewrite/02-open-questions-and-judgment.md)

## 文档范围与结论

### 当前发行版包形态

当前分析对象是 `@anthropic-ai/claude-code@2.1.197`。它不是旧版的 `package/cli.js` 单文件 JS bundle 包，而是 wrapper package 加平台 native binary 的发行形态：

- `package/package.json` 的 `bin.claude` 指向 `bin/claude.exe`。
- `install.cjs` 在 postinstall 阶段从对应平台的 optional native package 复制或 hardlink binary 到 `bin/claude.exe`。
- `cli-wrapper.cjs` 只是在 postinstall 没有放置 native binary 时的手动 fallback，它通过 Node 找到 native package 后 spawn binary，本身不是产品主逻辑。
- 当前本机 native package 为 `@anthropic-ai/claude-code-linux-x64@2.1.197`，其 `package/preprocessed/native/package/claude` 是 ELF native executable。
- `package/preprocessed/cli.extracted.bundle.js` 是从该 native executable 的 `.bun` section 抽取出的 JS bundle；`package/preprocessed/cli.extracted.bundle.pretty.js` 是同一 bundle 的格式化版本，也是人工阅读主入口。
- 后续文档应直接写“抽取 bundle”或 `package/preprocessed/cli.extracted.bundle.pretty.js`；它不表示当前 package 下真实存在 `package/cli.js` 或 `package/cli.readable.js`。
- `package/preprocessed/native/` 和 `package/preprocessed/reports/` 是 native 包、ELF/Bun section、字符串索引、符号索引与抽取报告等辅助证据，不是手写源码目录。

### 本文档覆盖什么

本文档覆盖的内容包括：

- CLI 入口与命令树
- 运行模式分流
- 全局状态与 Session State Store
- Session/Transcript 持久化、还原、分叉、Sidechain/Subagent
- 输入预处理链
- 主 Agent Loop / 多轮工具循环
- 模型调用适配层
- 网络通信模块
- 工具执行器、并发工具执行器、Hook/Permission 交互
- Prompt 装配与 Instruction / Rules / Skills 发现系统
- 远程控制 / 远程持久化
- 附件类型与 Context Layers
- 可重写的模块结构与类型接口

### 本文档不承诺什么

本文档**不承诺**还原：

- 原始源代码文件结构
- 原版完整 System Prompt 文本
- 私有服务端 API/端点的服务端实现细节
- Marketplace、Feature Flag、Telemetry 的上游数据生产与服务端内部逻辑
- 与原版所有边角行为 1:1 一致

### 当前最核心结论

基于当前 `2.1.197` wrapper package、native binary 与抽取 bundle 的分析，Claude Code CLI 的**核心可执行逻辑**已经还原到足以指导高相似版本重写的程度。  
这里更适合保留“证据边界已经覆盖到哪”，而不再重复维护各专题页里的正文结论清单。

当前已经有稳定结论的主题簇至少包括：

- CLI 入口、启动分流、命令树与运行模式
- Session / Transcript / Resume / Fork / Sidechain 主模型
- 输入编译链、主循环、多轮工具执行、Hook / Permission 主骨架
- prompt layering、context source 与主线程 / 非主线程边界
- provider / auth / stream fallback / remote ingress transport 的主分层
- 非模型网络链、control plane、remote persistence、MCP / plugin / skill / TUI 等外围系统

`04-rewrite/01` 在这里应被理解为“候选架构页”，不是“原始文件树还原页”。

“是否已经足以开工重写”以及“当前阻塞级未决项”的细化判断，统一收口到：

- [../04-rewrite/02-open-questions-and-judgment.md](../04-rewrite/02-open-questions-and-judgment.md)

本页只保留证据边界本身，不再和重写判断页重复维护同一组未决项总表。

另外，发行版 README 已明确给出一条用户可见的数据采集边界：

- usage data（例如 code acceptance / rejection）
- associated conversation data
- 通过 `/bug` 提交的用户反馈

这条边界更适合被当作“产品承诺侧证据”，而不是 event payload 完整 schema 的直接证明。

### 结论量化

按“**是否足以指导重写**”计算：

- **可运行替代品**：90%+ 可行
- **功能高相似版本**：75%~85% 可行
- **1:1 原版复刻**：低于 30%

---

## 证据来源与置信度说明

### 证据来源

本次逆向主要依赖：

1. `package/preprocessed/cli.extracted.bundle.pretty.js`：从 native executable 的 `.bun` section 抽取并格式化后的主阅读入口。
2. `package/preprocessed/cli.extracted.bundle.js`：同一抽取 bundle 的原始形态，用于校对格式化版本。
3. `package/preprocessed/native/package/claude` 与 `package/preprocessed/native/package/package.json`：当前 linux-x64 native executable 与 native package 元数据。
4. `package/package.json`、`package/install.cjs`、`package/cli-wrapper.cjs`：wrapper package、bin 指向、postinstall 放置 binary 与 fallback launcher 的直接证据。
5. `package/preprocessed/reports/`：`.bun` section、ELF section、字符串、符号与 keyword 抽取报告，用于快速定位和交叉校对。
6. `package/sdk-tools.d.ts`、发行版 README/License、CLI 帮助输出及部分子命令 `--help` 探测记录。
7. 对抽取 bundle 中关键函数、状态对象、路径函数、事件分支、attachment 类型、hook schema、tool runner、retry 逻辑的静态抽取。

其中，`package/preprocessed/reports/bunfs-paths.txt` 里的 `file:///$bunfs/root/src/entrypoints/cli.js` 是 Bun 内嵌文件系统路径线索，只能说明 bundle 原始入口名，不等于 wrapper package 里存在 `package/cli.js`。

### 置信度标签

本文档中使用以下术语：

- **已确认**：可从 bundle 直接看到定义、调用点、结构或运行结果，可信度最高。
- **高可信推断**：未直接看到完整定义，但从多个调用点与分支能高度确定其职责与接口。
- **待验证**：存在明显线索，但还不能确保 1:1 细节。

### 解读原则

压缩 bundle 中变量名多数无语义，因此本文档遵循：

- 优先以**职责**命名，而不是迷信压缩名。
- 优先还原**边界与接口**，而不是追求逐字符翻译。
- 对未知细节明确标注，不虚构。

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
