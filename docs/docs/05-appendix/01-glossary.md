# 术语索引

> 本页用于统一“压缩名”“职责名”“文档里反复出现的运行时术语”。
>
> 原则是优先按职责理解，而不是执着于压缩变量名本身。

## 使用说明

- “压缩名”表示从 bundle 里看到的符号名。
- “建议职责名”表示重写时更适合采用的工程命名。
- “当前判断”表示基于现有证据对它的职责总结，不等于 1:1 原始实现名。

## 主执行链

### `l3m`

- 建议职责名：顶层快速分流入口
- 当前判断：当前 2.1.197 bundle 的可见顶层入口；处理 `--version`、Chrome/native-host、computer-use MCP、daemon worker、background agents、fleet view、tmux/worktree、`--bare` 等 fast-path，再懒加载主 `main`

### `l4m`

- 建议职责名：主入口 `main`
- 当前判断：主模块导出的 `main` 函数；负责信号、deep-link、非交互判定、client type、bridge 环境标记，再进入 Commander 程序

### `u4m`

- 建议职责名：CLI 程序装配器
- 当前判断：构建顶层命令树，并在 `preAction` 中统一做初始化接线

### `jBz / _Bz / YBz`

- 建议职责名：旧文档残留入口名
- 当前判断：这组三段名来自旧版恢复文档，不应作为当前 `@anthropic-ai/claude-code@2.1.197` 的入口证据。当前入口口径以 `l3m() -> main(l4m) -> commander(u4m)` 为准

### `AU8`

- 建议职责名：输入编译器入口
- 当前判断：把原始输入编译成可进入主循环的 `messages + options` 结构

### `ihz`

- 建议职责名：输入预处理器
- 当前判断：处理图片、附件、slash command、remote 限制、本地命令分流

### `BU4`

- 建议职责名：普通输入消息构造器
- 当前判断：把普通用户输入包装成最终 user message，并返回 `shouldQuery`

### `CC`

- 建议职责名：主循环外壳
- 当前判断：旧文档里的主循环外壳压缩名，只适合作为历史检索线索；当前 2.1.197 对应主链以 `zN(...) / oRf(...)` 为准

### `po_`

- 建议职责名：多轮 agent 状态机
- 当前判断：旧文档里的多轮 agent 状态机压缩名；当前 2.1.197 的 turn loop 口径以 `oRf(...)` 为准

### `HSt`

- 建议职责名：高层模型调用适配器
- 当前判断：当前 2.1.197 的高层模型调用流接口，薄包装 `m1o(...)` 后进入 `Hpc(...)`

### `Hpc`

- 建议职责名：模型请求与 streaming adapter 主体
- 当前判断：负责 request body 条件装配、streaming event 累积、non-streaming fallback、request lifecycle telemetry 与最终 `assistant` 事件产出

### `Kur`

- 建议职责名：模型调用重试/降级包装层
- 当前判断：处理 retry、fallback、流式失败还原等模型调用防护逻辑

## 工具与权限链

### `he6`

- 建议职责名：单工具调度入口
- 当前判断：旧版压缩名；当前 2.1.197 工具执行主链以 `$Yt(...) -> nTf(...) -> oTf(...)` 与 `KHe` runner 为准

### `Mo_`

- 建议职责名：单工具执行包装器
- 当前判断：旧版压缩名；当前 2.1.197 对应职责已并入 `$Yt(...) -> nTf(...) -> oTf(...)` 主链说明

### `Re6`

- 建议职责名：流式并发工具执行器
- 当前判断：旧版压缩名；当前只作为历史检索线索保留

### `Zx8`

- 建议职责名：传统批次工具执行器
- 当前判断：旧版压缩名；当前只作为历史检索线索保留

## Session 与还原

### `I76`

- 建议职责名：Resume 还原器
- 当前判断：还原 transcript、plan、file-history backups，并处理 interrupted turn

### `hq4`

- 建议职责名：中断回补处理器
- 当前判断：识别 interrupted turn，必要时插入 continuation，并修正最后消息边界

## Prompt、Skill 与压缩

### `sj`

- 建议职责名：CLAUDE.md / memory 扫描主链
- 当前判断：扫描 `CLAUDE.md`、`.claude/rules/`、`CLAUDE.local.md`、AutoMem、TeamMem 等来源
- 当前更稳的落点：主线程里更像 `userContext.claudeMd` 的来源，而不是最终 request `system` 字段本身

### `_$(...)`

- 建议职责名：userContext 生成器
- 当前判断：读取 `sj()` 结果并装配主线程 `userContext`
- 当前已直接确认的字段：
  - `claudeMd?`
  - `userEmail?`
  - `attachedProject?`
  - `currentDate`

### `vO`

- 建议职责名：systemContext 生成器
- 当前判断：主线程 `systemContext` 生成器
- 当前已直接确认的字段：
  - `gitStatus?`
  - `perforceMode?`
- 当前更精确的判断：
  - `gitStatus` 不是对象，而是多行字符串快照
  - `perforceMode` 由 `CLAUDE_CODE_PERFORCE_MODE` 控制，不是第二注入槽位
  - `vO()` 内部还残留一个未启用的第二注入槽位形状（入参参与 memo key，telemetry 上报 `has_injection`，返回对象对应 spread 为空）

### `wb1`

- 建议职责名：git 状态快照构造器
- 当前判断：生成 `systemContext.gitStatus`
- 当前已直接确认的组成：
  - 当前分支
  - 主分支
  - `git status --short`
  - 最近 5 条提交

### `Lx8`

- 建议职责名：userContext 前置注入器
- 当前判断：把 `userContext` 包成 `<system-reminder>` user meta message，前插到消息链

### `dj4`

- 建议职责名：systemContext 末尾拼接器
- 当前判断：对 `Object.entries(systemContext)` 做 `key: value` 串接后，追加到 system prompt sections 末尾

### `sDf`

- 建议职责名：动态 skill 扫描器
- 当前判断：扫描触发目录下的 `SKILL.md`，产出 `dynamic_skill` attachment

### `D7n`

- 建议职责名：skill 列表生成器
- 当前判断：根据当前 registry 生成 `skill_listing` attachment

### `Su_`

- 建议职责名：已调用 skill 还原器
- 当前判断：在 resume 时把 `invoked_skills` attachment 重新装回运行态

### `Mi6`

- 建议职责名：compact 指令加载入口
- 当前判断：与 compact 相关的 instructions 加载有关，但完整覆盖面仍待验证

## 运行时术语

### Transcript

- 当前判断：不是聊天记录，而是事件日志
- 包含内容：message、summary、title、task summary、mode、queue operation、content replacement、file-history snapshot 等

### Session State Store

- 当前判断：全局 app state 与 per-session state 的混合存储
- 作用：共享当前 cwd、sessionId、缓存、模型消耗、invoked skills、prompt cache 等信息

### Context Layers

- 当前判断：工具执行后的上下文 overlay，不只是 UI 附件；早期文档中的 `ContextModifier` 在当前 `2.1.197` 活实现里主要落成 `contextLayers / permissionLayers`
- 已确认影响面：权限规则、模型选择、推理强度、工作目录 overlay

### Sidechain

- 当前判断：与主会话分开的执行链路，可复用主循环，但 transcript 独立落盘

### Subagent

- 当前判断：Sidechain 的具体实例形态之一，落盘到 `subagents/agent-<id>.jsonl`

### Headless

- 当前判断：一等运行模式，不是 TUI 的简化输出
- 典型用途：`print/json/stream-json`
- 边界：可与 Bridge 共用部分主执行链，但不应把 `--sdk-url` 或 `remote-control` 直接并入 Headless 术语

### Bridge

- 当前判断：`remote-control`、`--remote`、`--sdk-url + stream-json` 这一组远程控制 / SDK 接入 / 远程 transport 模式族
- 边界：和本地 TUI/Headless 共享核心循环，但 transport、接入方与产品语义不同

## 命名建议

- 文档与代码中优先使用职责名，压缩名只作为注释或证据引用保留。
- 如果一个压缩名职责仍有争议，先在本页标成“待验证”，不要直接在正文里硬编码为稳定接口名。

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
