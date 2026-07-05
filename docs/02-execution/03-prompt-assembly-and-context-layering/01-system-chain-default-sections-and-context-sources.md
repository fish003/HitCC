# System 链、默认 sections 与上下文来源

## 本页用途

- 单独承接主线程 prompt 的 `system` 链装配、默认 sections 和 request-level 上下文来源。
- 把 `zL / CDl / _H / O4t("compact")` 这组结论固定到同一页，并把 skills 与 `deferred_tools_delta` 在 prompt 里的最小落点收口。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../02-instruction-discovery-and-rules.md](../02-instruction-discovery-and-rules.md)
- [../04-non-main-thread-prompt-paths.md](../04-non-main-thread-prompt-paths.md)
- [../06-context-runtime-and-tool-use-context.md](../06-context-runtime-and-tool-use-context.md)
- [../../03-ecosystem/05-skill-system.md](../../03-ecosystem/05-skill-system.md)
- [../../04-rewrite/02-open-questions-and-judgment.md](../../04-rewrite/02-open-questions-and-judgment.md)

## 主线程 Prompt 装配与 Context Layering

### `zL(...)` 与外层 System Prompt 优先级

已确认：

```text
overrideSystemPrompt
> active agent prompt
> customSystemPrompt
> default system sections
> appendSystemPrompt
```

这是外层 `systemPrompt` 选择的 priority。  
但这里的 `default system sections` 现在应更具体地理解成：**`zL(...)` 产出的默认 system sections**，而不是 `Qv() / hS()` 那条 CLAUDE.md 扫描链。

当前能确认：

- `overrideSystemPrompt` 直接短路
- 否则若有 active agent prompt，则压过 `customSystemPrompt`
- `appendSystemPrompt` 总是尾部追加

### 主线程最终装配：必须拆成两条链理解

当前最稳的主线程近似应是：

```text
API request
  system:
    tTn(...)(attribution/runtime billing header)
    -> $Rn(...)(product identity header)
    -> systemPrompt selection(
         overrideSystemPrompt
         > active agent prompt
         > customSystemPrompt
         > zL(...) default system sections
         > appendSystemPrompt
       )
    -> CDl(..., systemContext)
      where systemContext currently defaults to:
        {
          gitStatus?: dlo() multi-line snapshot string,
          perforceMode?: CLAUDE_CODE_PERFORCE_MODE workspace hint
        }
    -> advisor / request-level explicit additions when applicable

  messages:
    F
      = Ak(transcript-like messages, availableTools)
      where transcript-like messages for current turn are first compiled as:
        [current turn user, ...attachments]
    -> Ak(...) linearizes attachments into user meta content
    -> adjacent user messages may be merged by rdr(...) / Qcm(...)
    -> attachment-derived meta content for current turn
    -> current turn user content
    -> otr(F, userContext)
      where userContext includes:
        claudeMd = B4t(N4t(await Qv()))
        userEmail?
        attachedProject?
        currentDate
      and otr(...) prepends one <system-reminder> user meta message at the very front
```

### `CDl(...)` 对 `systemContext` 的消费方式

`CDl(...)` 当前已可直接写成：

```text
return [
  ...systemPromptSections,
  Object.entries(systemContext)
    .map(([key, value]) => `${key}: ${value}`)
    .join("\n")
].filter(Boolean)
```

因此这里还应明确两点：

- `CDl(...)` 不是 `gitStatus` special-case 拼装器
- 它是一个 **泛型 `Object.entries(systemContext)` 序列化器**

这意味着：

- 如果未来上游真的给 `systemContext` 注入新字段，`CDl(...)` 会原样把它们拼到 system prompt 末尾
- 但就当前本地 bundle 而言，主线程默认自动传入的字段只看到 `gitStatus? / perforceMode?`
- 这也反过来说明：`_H(...)` 里那个预留第二槽位一旦被激活，`CDl(...)` 不需要改代码就能把它并入最终 `system`

这里最值得注意的是：

- `CLAUDE.md / .claude/rules / claude.local.md / AutoMem / TeamMem` 这条扫描链，当前更像 **messages 前缀上下文**
- 而不是 request-level `system` 主链
- `otr(...)` 不是 `Ak(...)` 内部阶段，而是**更晚一步**在调用模型前把 `userContext` 整体前插到 `messages[]` 最前
- `currentDate` 也走这条 `userContext -> otr(...) -> messages[]` 前缀链；当前 `2.1.197` 中它由 `qla(GSe())` 生成，会在非 first-party `ANTHROPIC_BASE_URL` 下把 base URL host 命中状态和 `Asia/Shanghai / Asia/Urumqi` timezone 状态编码到日期句子的撇号与日期分隔符里
- “attachment 在前、当前 user 在后”说的是**最终内容线性顺序**；至于它们最终是否仍是多个独立 `message object`，要看 `Ak(...)` 是否把相邻 user message 合并

### `_H(...)` 第二槽位：当前只能证明“预留”，不能证明“在本地活着”

`_H(...)` 的本地定义当前已能直接收紧成：

```text
let gitStatus =
  remote mode or git instructions disabled
    ? null
    : await dlo()

return {
  ...(gitStatus ? { gitStatus } : {}),
  ...(CLAUDE_CODE_PERFORCE_MODE ? { perforceMode } : {}),
  ...{}
}
```

同时它的完成日志仍会上报：

```text
system_context_completed {
  has_git_status,
  has_injection
}
```

这说明：

- `has_injection` 不是文档猜测，而是当前代码里真实保留的 telemetry 位，入参也参与 `_H(...)` 的 memo key
- 但本地可执行路径里，与之配套的返回对象当前仍是空 spread
- `perforceMode` 是单独的 env 条件字段，不是这个第二注入槽位
- 因此这更像 **被裁掉、灰度关闭，或等待外部配套接回的预留槽位**
- 还不能把它写成“当前本地 runtime 仍会注入第二段 systemContext”

### `O4t("compact")` 的真实覆盖面：不是所有 compact 都会预置 `load_reason`

这一点之前在本页还没有单独钉死，但代码现在已经足够直接：

```text
function O4t(reason = "session_start") {
  Yao = reason
  Xao = true
  Qv.cache.clear?.()
}
```

同时 memory scan 尾部对它的消费仍是：

```text
let reason = Tpp()
if (reason !== undefined && F4t()) {
  for (entry of collectedMemoryFiles) {
    let loadReason = entry.parent ? "include" : reason
    X5e(entry.path, entry.type, loadReason, ...)
  }
}
```

因此这里应明确写成三个层次：

- `O4t("compact")` 当前本地活入口在 compact 成功后的 cache reset / memory scan reason 预置路径
- compact 成功路径会清空 `hS.cache` 与 memory scan cache，并预置下一次扫描 reason
- 所以它覆盖的是“下一次 fresh memory scan”，不是 compact 结果生成当场

这里还必须再压实一个时序边界：

- `O4t("compact")` **不会立刻重新跑 `Qv()`**
- 它只是在本地把下一次 memory scan 的默认 `load_reason` 预置成 `compact`
- 真正发到 `InstructionsLoaded` hook 的时刻，仍然要等后续某次 fresh-run `Qv()` 发生
- 若某条 memory file 是通过 `@include` 进入，最终 `load_reason` 仍会被覆写成 `include`

因此本页现在不应再把 `compact -> O4t("compact")` 写成“所有 compact 都会立即刷新 instructions”，更准确的说法是：

- **compact success 预置的是下一次主扫描的默认 reason**
- **而且这只是 invalidation + retag，不是即时重扫**

### `zL(...)` 的默认 system sections

从 `zL(...)` 当前能直接确认的组成包括：

- 产品/身份类 system section
- tool / environment / language / output-style 相关 section
- MCP instructions section
- scratchpad / brief / summarize-tool-results 等内建 section

因此主线程里更稳的分类是：

- `zL(...)` = 默认 system sections
- `Qv() / hS()` = 默认 CLAUDE.md / memory discovery

这两者都属于“instructions 来源”，但**运行时落点不同**。

### Skills 的参与方式

本页只保留 skills 在 **prompt / attachment 顺序** 里的落点。

已确认与本页直接相关的只有：

- 文件读写相关路径会触发 `dynamicSkillDirTriggers`
- `sDf()` 产出 `dynamic_skill`
- `D7n()` 产出 `skill_listing`
- `invoked_skills` 属于 compact/resume 保留附件，而不是普通当前轮的发现附件

skills 的 registry、多源装载、动态注册、`SkillTool`、`invokedSkills` 运行态还原等内容，已拆到 [../../03-ecosystem/05-skill-system.md](../../03-ecosystem/05-skill-system.md)。

### `deferred_tools_delta` 的定位

这一层现在也可以和真正的 tool schema 注入拆开写。

当前更稳的分层是：

```text
attachments / meta messages
  -> deferred_tools_delta
  -> 告诉模型：当前有哪些 deferred tool 名字可通过 ToolSearch 获取

request body tools array
  -> 真正可调用的 tool schema
  -> 只包含：
       非 deferred tools
       + ToolSearch
       + 已在历史里被 tool_reference 命中过的 deferred tools
```

也就是说：

- `deferred_tools_delta` 或 `<available-deferred-tools>` 只负责暴露“名字列表”
- 真正让某个 deferred tool 变成 callable 的，不是 attachment 文本本身
- 而是下一轮 request build 时，query builder 根据历史 `tool_reference` 把该 tool 重新放进 `tools` 数组

因此 prompt layering 页里看到 `deferred_tools_delta`，不能直接等同于“schema 已经注入”。

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
