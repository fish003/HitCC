# Instruction source、Qv 扫描与 compat 边界

## 本页用途

- 整理 `Qv()` 主扫描顺序、`CLAUDE.md` / `.claude/rules/` 来源、compat 文件与 `/init` 迁移边界。
- 保留“`Qv()` 产物先进入 `userContext`，不是直接写入 `payload.system`”这一当前 2.1.197 边界。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../04-non-main-thread-prompt-paths.md](../04-non-main-thread-prompt-paths.md)
- [../../05-appendix/01-glossary.md](../../05-appendix/01-glossary.md)

## Instruction 发现、扫描与 Rules/Skills 来源

这一条已经从“仅还原骨架”推进到“**主扫描顺序与本地 compat 结论基本钉死**，剩余边界主要落在 attachment 顺序细节和 bundle 外行为”。

### 明确存在的 prompt/instruction sources

source 枚举至少包括：

- `local`
- `user`
- `project`
- `dynamic`
- `enterprise`
- `claudeai`
- `managed`

说明 prompt 不是单一来源，而是分层拼装。

### 一个关键修正：`Qv()` 不应再直接归入 system prompt 主链

补完直接证据后，必须修正此前文档里的一个核心判断：

- `Qv()` 仍然是最重要的 instructions / memory 扫描器
- 但在**主线程运行时**，它的产出首先进入 `hS(...) -> userContext.claudeMd`
- 随后再经 `otr(...)` 变成前置 `<system-reminder>` user meta message
- 也就是说，`Qv()` 主链更接近 **userContext/messages 注入链**
- 而不是最终 API request 的 `system` 字段主链

当前更稳的职责分层应改写为：

- `system` 链：运行时 header + `systemPrompt selection` 外层 system prompt 合成 + `systemContext`
- `messages` 链：`userContext`、attachment/meta message、transcript 正常消息一起归一化后下发

### 已确认的 `Qv()` 主扫描文件与目录

当前可见证据已从 `Qv()` 主扫描函数直接确认，它至少会扫描：

- `Managed` 级 `CLAUDE.md`
- `Managed` 级 `.claude/rules/`
- `User` 级 `CLAUDE.md`
- `User` 级 `.claude/rules/`
- 沿项目祖先目录向上查找的 `<dir>/CLAUDE.md`
- 沿项目祖先目录向上查找的 `<dir>/.claude/CLAUDE.md`
- 沿项目祖先目录向上查找的 `<dir>/.claude/rules/`
- 沿项目祖先目录向上查找的 `<dir>/claude.local.md`
- `additionalDirectoriesForClaudeMd` 指向目录下的 `CLAUDE.md / .claude/CLAUDE.md / .claude/rules/`
- 追加到末尾的 `AutoMem`
- 追加到末尾的 `TeamMem`

但这里必须再次强调：

- 这条顺序是 **`Qv()` 的扫描顺序**
- 不应直接等同于“最终 API request 的 system sections 顺序”

### 关于 compat 文件

此前文档写到：

- `AGENTS.md`
- `.cursor/rules`
- `.cursorrules`
- `.github/copilot-instructions.md`
- `.windsurfrules`
- `.clinerules`
- `.mcp.json`

现在可以把 compat 相关结论再收紧一层。

当前可见证据**没有在 `Qv()` 这条主扫描链里直接看到这些文件名常量**；它们目前更像是：

- `/init`/初始化类 prompt 明确提及的“应参考文件”
- 某些 IDE/tool ecosystem 兼容逻辑中的概念来源

额外可确认的是：

- `AGENTS.md / .cursor/rules / .cursorrules / .github/copilot-instructions.md / .windsurfrules / .clinerules` 目前看到的命中基本都落在 `/init` 类提示词文案里
- `.mcp.json` 确实有独立的 MCP 配置读写路径，但这属于 MCP config 子系统证据，**不是 prompt 注入证据**
- `init` builtin prompt 明确要求子代理在初始化时读取这些 compat 文件，并把其中重要部分写入生成的 `CLAUDE.md`
- `init` 本身只是一次性 prompt command：其 `getPromptForCommand()` 返回的长文本会在执行 `/init` 的那一轮被包成 meta user message 送入模型，而不是注册成常驻 system/userContext 来源
- `subagent / fork / compact / hook agent / sdk-url / bridge` 当前也**没有看到第二套 compat discovery**
- 但它们进入 non-main-thread 的载体不能再一概写成“复用现成上下文”
  - `_3(...)` 默认分支会 fresh-build `userContext/systemContext`
  - `vk(...)` 只消费外部给的 `cacheSafeParams`，是否复用旧 snapshot 取决于 producer
  - `hook_agent` / dedicated compact summarize 这类专用路径则是旁路主线程 layering

因此当前更稳的本地闭环应改写为：

- `compat 文件 -> /init 读取 -> 写入 CLAUDE.md / claude.local.md -> Qv() 扫描 -> userContext.claudeMd -> 运行时`
- 而不是：`compat 文件 -> CLI 本地主线程自动兼容注入 prompt`

因此更稳妥的结论应是：

- `CLAUDE.md / .claude/rules / claude.local.md / AutoMem / TeamMem` 的扫描已确认
- compat 文件**不会在当前已确认的 CLI 本地链路中被自动注入** `system` 或 `messages`
- 若 `remote-control / sdk-url` 对接的远端服务端在服务端侧再做 compat 拼装，本地 bundle 无法直接证明或反证

### 已确认的缓存、函数与状态

- `cachedClaudeMdContent`
- `systemPromptSectionCache`
- `additionalDirectoriesForClaudeMd`
- `Qv()`：主 instructions/memory 文件收集器
- `xy()`：单文件处理与递归 include
- `v16()`：`.claude/rules/` 规则文件收集器
- `JI6()`：conditional rule 过滤层
- `HQ9() / aC1()`：文件读取、注释剥离、`@include` 解析
- `hS(...)`：把 `Qv()` 扫描结果装配进 `userContext`
- `_H(...)`：systemContext 生成器
- `otr(...)`：把 `userContext` 前插为 `<system-reminder>` user meta message
- `CDl(...)`：把 `systemContext` 追加到 system prompt sections 末尾
- `zL(...)`：默认 system sections 生成器
- `Ak(...)`：消息归一化与 attachment -> message 线性化入口
- `invokedSkills`
- `planSlugCache`

### 直接可得的结论

这说明 prompt/discovery 系统：

1. 有 section 级缓存
2. 支持从额外目录加载 `CLAUDE.md`
3. `.claude/rules/` 至少区分 unconditional 与 conditional 两类
4. Skills 可以被调用后纳入 prompt/context
5. 主扫描函数不是平铺读目录，而是带递归 include、去重、深度限制与路径排除的收集器
6. `Qv()` 的结果在主线程里首先表现为 `userContext.claudeMd`，不是直接写进 `system`

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
