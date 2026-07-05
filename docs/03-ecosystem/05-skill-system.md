# Skills 系统深挖

## 本页用途

- 单独整理 skills 的注册源、动态发现、prompt 注入、invokedSkills 持久化、SkillTool 运行时修改与 fork 路径。
- 把 “技能只是 slash command” 的表层认识收紧成完整运行时模型。

## 相关文件

- [../02-execution/03-prompt-assembly-and-context-layering.md](../02-execution/03-prompt-assembly-and-context-layering.md)
- [../02-execution/05-attachments-and-context-modifiers.md](../02-execution/05-attachments-and-context-modifiers.md)
- [../02-execution/06-context-runtime-and-tool-use-context.md](../02-execution/06-context-runtime-and-tool-use-context.md)
- [01-resume-fork-sidechain-and-subagents.md](./01-resume-fork-sidechain-and-subagents.md)

## 一句话结论

skills 在 Claude Code CLI 中至少分成 **注册层、动态发现层、prompt 注入层、调用执行层、持久化还原层** 五层，远不只是 `/xxx` 命令别名。

---

## 技能系统的五层结构

当前更稳的近似是：

```text
registry layer
  -> bundled / project skills / plugin skills / builtin plugin skills / commands_DEPRECATED / MCP prompt commands

dynamic discovery layer
  -> file operations trigger skill dirs
  -> sDf() emits dynamic_skill
  -> p_t()/f_t() mutate runtime registry

prompt injection layer
  -> skill_listing
  -> invoked_skills

execution layer
  -> slash command expansion
  -> SkillTool inline / fork execution
  -> contextLayers / permissionLayers

persistence / resume layer
  -> invokedSkills map
  -> compact keep attachments
  -> resume restore
```

## 注册源不止一个

当前 2.1.197 的 skills / slash command 注册链可以按 `wsm -> Jcr -> gA / uC / y0o` 来理解。skills 的来源至少包括：

- skill dir commands：项目 / 用户 skill directory 中的 `SKILL.md`
- workflow commands
- plugin commands
- plugin skills
- bundled skills
- builtin plugin skills
- built-in local command set
- MCP prompt commands

这里有一个容易混淆但很关键的点：

- `uC(projectRoot)` 返回的是适合 `SkillTool` / `skill_listing` 面向模型公告的 prompt-skill 子集
- `y0o(ctx)` 会在 `gA(projectRoot)` 基础上，再把 `appState.mcp.commands` 中：
  - `type === "prompt"`
  - `loadedFrom === "mcp"`
  的命令并进来

因此 SkillTool 可见的“skill 列表”里，实际上混入了 **MCP prompt command**。

## `Jcr -> gA -> uC / y0o`：这不是同一份列表

skills 当前至少存在四套相关但不相同的候选集：

### 1. `Jcr(projectRoot)`：基础总注册表

`Jcr(...)` 当前把多源命令合成基础总表，顺序可直接写成：

```text
skill dir commands
-> workflow commands
-> plugin commands
-> plugin skills
-> bundled skills
-> builtin plugin skills
-> built-in local command set
```

这里先得到的是“总 registry”，还没做 availability / dynamic merge / SkillTool 过滤。

旧文档里的 `RC4` 额外注入槽在当前 bundle 已没有字面命中；不能再把它写成本地可见的活 producer。若后续需要兼容旧口径，最多把它放进 bundle 外 / adapter 扩展位。

### 2. `gA(projectRoot)`：运行态可用命令表

`gA(...)` 会在 `Jcr(...)` 之上继续做几步：

- 先按 `availability` 与 `isEnabled()` 过滤
- 再把动态 registry `vxl()` 合进来
- 但动态项只在**名字不冲突**时追加，不会覆盖已存在静态命令
- 还会处理目录作用域命名与 fallback skill 去重

更关键的是动态项的插入位置：

- 若列表里能找到第一段 built-in 本地命令集合 `yen()`
- 动态 skills 会被插在它前面

所以更稳的排序结论是：

- 同名时，静态 registry 优先，动态 skill 不覆盖
- 无冲突时，动态 skill 会排在 built-in 本地命令之前
- `gA(...)` 才是“当前运行态真正能 resolve 到的命令总表”

### 3. `uC(projectRoot)`：适合向模型公告的 prompt-skill 子集

`uC(...)` 不是总表，而是从 `gA(...)` 里再筛一次：

- 只保留 `type === "prompt"`
- 排除 `disableModelInvocation`
- 过滤 `skillOverrides` 关闭或 `user-invocable-only` 的项
- 接受 built-in / bundled / skills / `commands_DEPRECATED`、有用户描述或 `whenToUse` 的 prompt skill

因此 `uC(...)` 更像：

- **SkillTool / skill_listing 面向模型的“可公告 prompt-skill 子集”**

### 4. `y0o(ctx)`：SkillTool 实际 resolve 的执行集合

`y0o(...)` 会在 `gA(projectRoot)` 基础上，再把当前 `appState.mcp.commands` 中：

- `type === "prompt"`
- `loadedFrom === "mcp"`

并进来，再按 `name` 去重。

因此：

- `SkillTool.validateInput / checkPermissions / call` 真正看的，是 `y0o(ctx)`
- 它比 `uC(projectRoot)` 更大
- 它明确包含 runtime MCP prompt commands

这也是为什么“模型被公告到的 skill 列表”和“SkillTool 真的能执行的候选集”不能直接画等号。

## `SKILL.md` 的主装载路径

当前能直接确认两类读取器：

### 1. 目录型 skill loader

`IE6(dir, source)` 会：

- 枚举目录下子目录
- 找每个子目录里的 `SKILL.md`
- 解析 frontmatter + markdown content
- 产出 `loadedFrom: "skills"` 的 prompt skill

### 2. 单文件/目录兼容 loader

`FH4(path, source, ...)` 会：

- 若目录根直接存在 `SKILL.md`，按单 skill 处理
- 否则枚举一级子目录里的 `SKILL.md`
- 生成 `isSkillMode: true` 的命令对象

这说明 skills 既支持：

- `.../<name>/SKILL.md`
- 也支持某些“目录根就是单 skill”的读取模式

## 动态发现：由文件操作触发，而不是全局轮询

当前最关键的本地闭环是：

1. 文件读取/写入类路径在访问文件后，会通过 `d_t(paths, cwd)` 反推出相关 `.claude/skills` 目录
2. 这些目录被写入 `dynamicSkillDirTriggers`
3. 同时后台触发 `p_t(dirs)` 直接把新 skills 装进运行时 registry
4. attachment 生成阶段再由 `sDf(ctx)` 把“发现到哪些目录里有哪些 skill 名字”包装成 `dynamic_skill`

`sDf(...)` 的行为现在已直接可写成：

- 对每个触发目录，列出一级子目录
- 检查 `<dir>/<subdir>/SKILL.md` 是否存在
- 生成：
  - `skillDir`
  - `skillNames`
  - `displayPath`
- 最后清空 `dynamicSkillDirTriggers`

因此 `dynamic_skill` 的本质是：

- **当前 turn 的发现提示**
- 不是完整 skill 内容本身

## 动态注册表：`dynamicSkillDirs / dynamicSkills / conditionalSkills`

动态 skills 注册状态至少能确认有四组：

- `dynamicSkillDirs`：已处理过的动态 skill 目录集合
- `dynamicSkills`：当前激活 registry
- `conditionalSkills`：带 `paths` 条件的待激活 skill 池
- `activatedConditionalSkillNames`：已被条件路径激活过的名字集合

其中：

- `p_t(...)` 会把 file-operation 新发现的 skills 放进 `dynamicSkills`
- `f_t(changedPaths, cwd)` 会按 `paths` 条件从 `conditionalSkills` 激活到 `dynamicSkills`
- `vxl()` 返回 `dynamicSkills` 当前值
- `Cxl()` 会清空动态状态

这说明 skills 不是静态启动快照，而是**可在会话内继续扩展的 registry**。

## `dynamic_skill`、`skill_listing`、`invoked_skills` 三者职责完全不同

### `dynamic_skill`

- 来源：`sDf()`
- 内容：目录与名字
- 定位：发现提示
- 注入特点：默认不直接变成 meta prompt

### `skill_listing`

- 来源：`D7n()`
- 内容：当前可用 skills 列表
- 定位：向模型宣布“现在有哪些 skill 可以调用”
- 注入特点：会变成技能列表 meta message

### `invoked_skills`

- 来源：`Vvq()` compact 保留附件
- 内容：本 session 已调用 skill 的内容摘要
- 定位：要求后续继续遵循这些 skill 指南
- 注入特点：强于普通列表，属于“已生效技能”的再注入

这三者不能混写成一个“skills attachment”概念。

## `skill_listing` 不是每轮全量重发

`D7n(ctx)` 的运行时边界已经很清楚：

- 先检查当前 tools 中是否存在 `SkillTool`
- 再取 `uC(projectRoot)` 形成当前可宣布 skills
- 若 MCP skill discovery gate 开启，还会把 `BKe(ctx.getMcp().commands)` 里可模型调用的 MCP prompt commands 并入公告集合
- 通过全局 `Z7t` 记录“已经发过哪些 skill 名字”
- 只发送新增项

这里有一个必须单独写清的边界：

- `D7n()` 用的是 `uC(projectRoot) + BKe(mcp commands)`
- 不是 `y0o(ctx)` 的完整执行集合

因此 `skill_listing` 当前声明给模型的，是 **公告子集**，不是 SkillTool 的完整执行集合。  
尤其是：

- 某些 `disableModelInvocation` 命令
- 某些只允许用户显式 slash 调用的命令

并不会自动出现在 `skill_listing` attachment 里。

另外还有一个很关键的还原位：

- `Wtr` 为真时，`cMl()` 会把当前列表全部记入 `Z7t`，但**不再发送 attachment**
- resume 时一旦看到历史里存在 `skill_listing`，会触发这一“已公告”还原路径，避免重复注入

因此 resume 后的正确理解应是：

- 不是把旧 `skill_listing` 再发一遍
- 而是先把“已发过 skill 列表”这件事还原回运行态，避免重复注入

## `invokedSkills`：全局运行态与 compact/resume 闭环

全局状态里直接有：

- `invokedSkills: Map`

`NJ6(name, path, content, agentId?)` 的行为也已直接确认：

- key = `${agentId ?? ""}:${name}`
- value 至少包括：
  - `skillName`
  - `skillPath`
  - `content`
  - `invokedAt`
  - `agentId`

围绕这张表，至少有三条直接闭环：

1. `yq8(agentId)`：按 agent 过滤 invoked skills
2. `vVq(agentId)`：compact 时导出 `invoked_skills` attachment
3. `Su_(messages)`：resume 时扫描历史 attachment，把 `invoked_skills` 重新装回运行态

另外还有清理逻辑：

- `_s(agentId)`：删除某个 agent 的 invoked skills
- `kc8(activeAgentSet)`：按存活 agent 集合清理

因此 invoked skill 不是 prompt 侧临时文本，而是**全局会话状态的一部分**。

## 技能真正生效的入口：`r74(...)` 与 `SkillTool.call(...)`

### slash command / prompt command 路径

`r74(...)` 执行 prompt skill 时会：

1. 调 `getPromptForCommand(args, ctx)`
2. 把纯文本内容拼成完整 skill content
3. 调 `NJ6(...)` 记录到 invokedSkills
4. 生成：
   - 一个 user message，标记 skill/load 语义
   - 一个 meta user message，承载 skill 展开内容
   - `command_permissions` attachment

因此 slash command 的本质不是“本地直接跑完”，而是**把 skill 展开成一段会进入主对话的提示链**。

### SkillTool 路径

`SkillTool.call(...)` 有两种模式：

- `inline`
- `fork`

若 skill frontmatter 指定 `context === "fork"`，则走 `Em_(...)`：

- 通过 `ZC8(...)` 生成 promptMessages
- 选定 agent
- 调非主线程 agent 执行入口，在子 agent 上执行
- 结果以 `status: "forked"` 返回

否则走 inline 路径：

- 仍然先做 slash-command prompt 处理
- 然后把新消息回注入主线程

## `ZC8(...)`：fork skill 的核心预处理器

`ZC8(skill, args, ctx)` 当前已能直接写实为：

- 取 `getPromptForCommand(args, ctx)` 的纯文本拼接结果
- 计算 `allowedTools`
- 用 `co_(getAppState, allowedTools)` 包装权限态
- 选择目标 agent：
  - skill 指定 agent 优先
  - 否则回退 `general-purpose`
- 返回：
  - `skillContent`
  - `modifiedGetAppState`
  - `baseAgent`
  - `promptMessages`

这说明 fork skill 不是把原命令字符串直接丢给子 agent，而是**先编译成完整 prompt 再 fork**。

## SkillTool 的 `contextLayers` 已经可以写实

SkillTool 的 inline concrete `contextLayers` 当前已直接确认会做三件事：

1. 追加 `allowed_tools`
2. 追加 `model`
3. 追加 `effort`

更具体地说：

- `allowedTools` 会以 `{ kind: "allowed_tools", allowedTools }` 进入后续 `permissionLayers`，再由 `Nr(ctx)` 折算成有效 permission context
- `model` 会经归一化后以 `{ kind: "model", mainLoopModel }` 进入 layer，并由 `NYt(...)` 同步更新 `options.mainLoopModel`
- `effort` 会以 `{ kind: "effort", effort }` 进入 layer，再由 `bg(ctx)` 折算为有效 effort

所以 skill 生效不只是“告诉模型照着做”，还会直接**改后续运行态**。

fork skill 和 fork slash command 的边界不同：它们会先用 `Dzt(...)` 生成 `allowed_tools / disallowed_tools` layer，再把这些 layer 合进传给 `_3(...)` 的子 agent `permissionLayers`。这属于 fork 执行前的上下文预置，不是父线程等待 tool result 后再提交。

## `r74(...)` 与 `SkillTool.call(...)` 的候选解析规则不同于 attachment

技能真正执行时的解析规则也已经可以收紧：

- `/name`
- `name`
- `aliases`

都会通过 `UU(...)` 在候选集中 resolve。

而这个候选集在两条执行入口里都来自 `y0o(ctx)`：

- `SkillTool.validateInput`
- `SkillTool.checkPermissions`
- `SkillTool.call`

也就是说：

- attachment 侧的 `skill_listing` 只负责“向模型公告”
- 真正执行时看的却是 `gA + runtime MCP prompt commands`

这两套集合只部分重叠。

## 相同文件与条件 skills 的去重/激活顺序

`_r1(...)` 这一层还能再收紧两个点：

### 1. 同文件 dedupe 早于运行态合并

- 先按真实文件归并
- 同一文件若被多个 source 命中，只保留一份
- 日志里会写 `Deduplicated ... skills (same file)`

所以“同文件重复装载”不会进入后续 `gA(...)` 排序阶段。

### 2. 条件 skills 默认不进主表

- 带 `paths` 的 prompt skill 先进入 `conditionalSkills`
- 只有 `f_t(changedPaths, cwd)` 命中后才移入 `dynamicSkills`
- 未激活前，它们不会进入 `gA(...)` 主结果

因此条件 skill 不是弱提示，而是**真正的延迟激活 registry 项**。

## fork/subagent 不继承动态发现触发器

`ts6(parentCtx, overrides)` 当前已经能直接确认会重建：

- `nestedMemoryAttachmentTriggers = new Set()`
- `dynamicSkillDirTriggers = new Set()`

这意味着：

- fork/subagent 不会沿用父线程已经触发过的动态 skill 发现状态
- skill 运行态是“部分继承、部分重建”的 capability object，而不是完整复制

## settings policy 会直接改写 skill 暴露面

当前 2.1.197 的 settings schema 已经把 skill policy key 写得比较明确，证据在 `package/preprocessed/cli.extracted.bundle.pretty.js:66106-66117`、`66196`，以及 tool gating 相关路径：

| key / surface | 当前语义 | 边界 |
| --- | --- | --- |
| `skillOverrides` | 按 skill name 设置 `on / name-only / user-invocable-only / off` | 改的是模型公告与用户可调用面，不等价于删除磁盘 skill |
| `disableBundledSkills` | 移除随 Claude Code 分发的 bundled skills / workflows | built-in slash commands 仍可输入，但会从模型可见面隐藏；plugin、项目 `.claude/skills/`、`.claude/commands/` 不受这个 key 直接影响 |
| `disableSkillShellExecution` | 禁止 skill shell execution | 这是 skill runtime 的安全 gate，不是普通 Bash tool permission rule |
| `disallowed-tools` | skill frontmatter / skill tool policy 可声明禁用工具 | 它收紧 skill 执行环境，不代表全局工具不可用 |

因此 skills 的“是否存在”“是否给模型公告”“用户能否 slash 调用”“执行时能否跑 shell/工具”是四个不同面，不能只用 enabled/disabled 一个布尔值概括。

## `discoveredSkillNames / RC4` 是历史口径，不是当前主链

当前 `package/preprocessed/cli.extracted.bundle.pretty.js` 里没有 `discoveredSkillNames` 或 `RC4` 的活字面命中。旧文档曾把它们写成预留字段 / 额外 producer 槽位，但按当前 2.1.197 bundle，应收紧为：

- `skill_listing` 增量发送由 `Z7t / cMl(...)` 控制
- 动态发现由 `dynamicSkillDirTriggers -> sDf(...)` 与 `dynamicSkills` 控制
- bundle 外若仍有其它 producer，应进入 registry adapter 扩展位，不进入本地确定性主链

## 动态 producer 边界

当前本地能闭环的 skill producer 可以分成三类：

- 启动期 / reload 期的静态注册源：skill dir commands、workflow commands、plugin commands、plugin skills、bundled skills、builtin plugin skills、built-in local command set
- 会话内动态注册源：file-operation 触发的 `dynamicSkillDirTriggers -> p_t(...) -> dynamicSkills`
- 条件激活源：`conditionalSkills -> f_t(changedPaths, cwd) -> dynamicSkills`

这些 producer 最终都进入 `Jcr / gA / uC / y0o` 这组 registry / filter / resolve 链。当前没有证据显示还有一个独立的 `discoveredSkillNames` 字段或 `RC4` 槽位会直接绕过这条链路给模型注入 skill。

## 更稳的工程结论

基于当前本地 bundle，skills 子系统已经可以收敛成：

1. skill 来自多源 registry，而不止 project `SKILL.md`
2. `Jcr / gA / uC / y0o` 分别对应总表、运行态表、公告子集、执行集合
3. 文件读写会触发动态 skill 发现与注册
4. 条件 skills 会先进入待激活池，而不是直接进主表
5. prompt 注入明确拆成 `dynamic_skill / skill_listing / invoked_skills`
6. `skill_listing` 公告子集不等于 SkillTool 的完整执行集合
7. invoked skill 有全局 Map、compact 导出、resume 还原闭环
8. SkillTool 会改权限、模型、effort，而不只是扩展消息
9. fork/subagent 默认不会继承父链 discovery triggers

## 当前仍未完全钉死

- bundle 外若存在远端/服务端 skill producer，本地代码无法正证；候选实现应通过 registry adapter 接入

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
