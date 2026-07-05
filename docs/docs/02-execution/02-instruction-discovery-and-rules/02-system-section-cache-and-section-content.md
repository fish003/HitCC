# System section cache 与默认 section 内容

## 本页用途

- 整理 `systemPromptSectionCache`、`zL(...)` 默认 section 名单、section 内容边界与失效规则。
- 区分可缓存 section、常量/空值 section 与每轮强制重算的 `mcp_instructions`。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../04-non-main-thread-prompt-paths.md](../04-non-main-thread-prompt-paths.md)
- [../../05-appendix/01-glossary.md](../../05-appendix/01-glossary.md)

## `systemPromptSectionCache`：现在可以写成真正的 section cache

这块以前只写到“有 section 级缓存”，现在可以再收紧。

当前本地 bundle 直接可见：

- `systemPromptSectionCache` 是全局 `AppState` 上的一份 `Map`
- 统一入口是：
  - `Lq8()`：取整张 cache map
  - `Vc8(name, value)`：按 section 名写入
  - `Ec8()`：整张清空
- `zL(...)` 不直接逐段裸算，而是先构造一组 section descriptor，再交给 `kHq(...)`

`kHq(...)` 的本地语义现在可以直接写成：

```text
for each section descriptor:
  if !cacheBreak && cache.has(name)
    -> return cached value
  else
    -> recompute via compute()
    -> cache.set(name, value)
```

因此这里不是“整段 system prompt 做一个字符串缓存”，而是：

- **按 section name 做独立缓存**
- 命中粒度是 section，不是整份 prompt
- 即使本轮是 cache-break section，算完后的最新值仍会回写到 map

## `zL(...)` 当前已直接可见的 section 名单

`zL(...)` 现在可直接确认会交给 `kHq(...)` 的 section 包括：

- `memory`
- `ant_model_override`
- `env_info_simple`
- `language`
- `output_style`
- `mcp_instructions`
- `scratchpad`
- `frc`
- `summarize_tool_results`
- `brief`

其中当前本地 bundle 里，只有一项明确被标成 `cacheBreak: true`：

- `mcp_instructions`

也就是说，当前更稳的说法应是：

- **绝大多数默认 system sections 都允许按 name 复用**
- **MCP instructions 被视为跨 turn 易变段，每轮强制重算**

## 各 section 的当前内容边界

这一项现在也不必继续只写“名单”，可以把当前本地 bundle 已直接可见的 section 语义再压实。

- `memory`
  - 由 `fT8()` 生成
  - 内容不是单一文件正文，而是 auto/team memory 的组合提示块
  - 会按：
    - team memory 启用与否
    - auto memory 是否启用
    - extract mode 是否启用
    - 额外 guidelines
    这些条件切换不同模板
- `ant_model_override`
  - 由 `h8_()` 生成
  - 当前本地直接返回 `null`
  - 因而现在更像保留 section 名，而不是活注入源
- `env_info_simple`
  - 由 `Tvq(model, additionalWorkingDirs)` 生成
  - 当前是环境摘要块：工作目录、git repo 状态、平台、OS、模型家族信息、fast mode 说明等
- `language`
  - 只在 settings 里存在 `language` 时生成
  - 内容是“始终用某语言回复”的固定说明块
- `output_style`
  - 由当前 output style prompt 生成
  - 若没有选中的 style，则为 `null`
- `mcp_instructions`
  - 由连接中 MCP server 的 instructions 拼成
  - 当前唯一明确的 `cacheBreak: true` section
- `scratchpad`
  - 由 `d8_()` 生成
  - 只在 scratchpad 功能可用时出现
  - 内容是“必须使用 session scratchpad 目录而不是 `/tmp`”的说明块
- `frc`
  - 由 `c8_(model)` 生成
  - 当前本地直接返回 `null`
- `summarize_tool_results`
  - 当前是固定提醒文本
  - 核心语义是：tool result 之后可能被清空，重要信息要尽快写回自己输出
- `brief`
  - 只有 brief entitlement 存在且运行态 brief mode 已启用时才非空
  - 内容来自 `BRIEF_PROACTIVE_SECTION`

### 重点 section 变化条件表

| section | builder / selector | 何时非空 | 主要变化条件 | 额外说明 |
| --- | --- | --- | --- | --- |
| `memory` | `fT8()` | 只有 memory 功能链有可用来源时才非空 | `tengu_moth_copse` gate；team memory 是否启用；auto memory 是否启用；extract mode `hU6()`；`CLAUDE_COWORK_MEMORY_EXTRA_GUIDELINES` 是否存在 | team memory 开启时走 combined prompt；否则只走 auto memory；两者都不可用时返回 `null` |
| `output_style` | `yvq()` -> `S8_(style)` | 只有选中的 style 不是 `default/null` 时才非空 | plugin forced output style；`settings.outputStyle`；当前 cwd 下 output-style registry 变化 | `output_style` section 只负责 style prompt 本身；`keepCodingInstructions` 影响的是 `x8_()` 是否保留，不是这个 section 的非空条件 |
| `brief` | `i8_()` | 只有 `BRIEF_PROACTIVE_SECTION` 已装入且 `isBriefEnabled()` 为真时才非空 | brief entitlement 是否存在；运行态 brief mode toggle | 当前更像“运行态开关段”，不是固定 prompt 常量 |
| `language` | `R8_(settings.language)` | 只有 settings 里设置了 `language` 时才非空 | 用户设置变更 | 与 `brief/output_style` 不同，它不依赖插件 registry 或会话 mode |
| `mcp_instructions` | `C8_(mcpClients)` | 只有存在 `connected` 且带 `instructions` 的 MCP client 时才非空 | MCP client connect/disconnect；server instructions 变化 | 当前唯一明确 `cacheBreak: true` 的 section |

因此当前对 section cache 的更稳理解应改写为：

- **不是所有 section 都是“重内容动态块”**
- 其中有些其实是：
  - 恒为 `null` 的保留槽位
  - 固定文案提示
  - 受运行态开关控制的条件段

## `systemPromptSectionCache` 当前缓存什么，不缓存什么

把 section 名单与实现一起看后，当前本地可以直接分成三类：

1. **缓存且常常有实质内容**
   - `memory`
   - `env_info_simple`
   - `language`
   - `output_style`
   - `scratchpad`
   - `brief`
2. **缓存，但当前常量/空值化**
   - `ant_model_override`
   - `frc`
   - `summarize_tool_results`
3. **每轮强制重算**
   - `mcp_instructions`

这点很关键，因为它说明当前 section cache 的收益主要不在：

- `summarize_tool_results`
- `ant_model_override`
- `frc`

而主要在：

- memory prompt
- env / output style / language 这类稳定段
- scratchpad / brief 这类按会话状态变化、但不需要每轮重算的段

## `systemPromptSectionCache` 的失效边界

当前本地已直接看到的清理链有两类：

1. 显式清空 section cache
   - `Yn() -> Ec8()`
2. 与 instruction/user context 联动的上层失效
   - `O4t(loadReason)` 会通过 `Ek()` 把 `Qv.cache` 清掉
   - `Cn(querySource)` 在 compact 后会：
     - 主线程 / SDK 路径下清 `_$.cache`
     - 调 `O4t("compact")`
     - 再调 `Yn()` 清 `systemPromptSectionCache`

另外，session reset / clear conversation 路径本地也直接能看到：

- `_$.cache.clear?.()`
- `vO.cache.clear?.()`
- `_H.cache.clear?.()`
- `mjq(null)`
- `Cn()`
- `O4t("session_start")`

因此当前更准确的边界是：

- `systemPromptSectionCache` 是**会话内 section 复用层**
- 但不是长期稳定缓存
- compact、clear/reset、session-start 这类边界都会把它打掉

另外还有一个容易误判的点，现在也可以一起钉死：

- `currentDate` 不在 `systemPromptSectionCache` 这层
- 它来自 `hS(...) -> userContext.currentDate`
- 日期切换由 `y6z()` / `lastEmittedDate` 这套 attachment 链单独处理

因此当前没有看到：

- “跨天自动清 `systemPromptSectionCache`” 的专门逻辑
- 也没有必要用它来解释日期变化

更稳的理解应是：

- **日期变化属于 `userContext` / attachment 层问题**
- **不是 default system sections cache 的失效触发器**

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
