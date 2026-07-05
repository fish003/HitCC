# userContext、systemContext 与 ClaudeMd 边界

## 本页用途

- 整理 `hS()`、`_H(...)`、`cachedClaudeMdContent`、`currentDate` 标记链、`gitStatus / perforceMode` 与 `claudeMdExcludes`。
- 区分 instruction discovery、user meta message、systemContext 与 auto-mode classifier 旁路。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../04-non-main-thread-prompt-paths.md](../04-non-main-thread-prompt-paths.md)
- [../../05-appendix/01-glossary.md](../../05-appendix/01-glossary.md)

## `hS()` / `_H(...)`：字段现在已直接钉死

现在已经可以直接从 bundle 写出两者定义，而不再只是调用点反推。

### `hS()` 当前已直接确认

`hS()` 的实际返回形状现在可以写成：

```text
CLAUDE_CODE_DISABLE_CLAUDE_MDS || (simple-mode && additionalDirectoriesForClaudeMd is empty)
  -> claudeMd = null
else
  -> claudeMd = B4t(N4t(await Qv()))

return {
  ...(claudeMd ? { claudeMd } : {}),
  ...(userEmail ? { userEmail } : {}),
  ...(attachedProject ? { attachedProject } : {}),
  currentDate
}
```

因此当前已直接确认：

- `userContext` 固定至少有 `currentDate`
- `claudeMd` 是可选字段，不一定存在
- `userEmail` 来自本地 account email，`ANTHROPIC_UNIX_SOCKET` 路径下会被跳过
- `attachedProject` 来自 attached project context，缺失时不进入 `userContext`
- `cachedClaudeMdContent` 会在这里被更新

### `currentDate` 的 base URL / timezone 标记

当前 `2.1.197` 的可读 bundle 中，`currentDate` 不是单纯的 ``Today's date is ${date}.`` 字符串。它的本地生成链可以收紧为：

```text
GSe()
  -> 本地日期，格式为 YYYY-MM-DD

qla(GSe())
  -> rdp()
       -> 若没有 ANTHROPIC_BASE_URL，或 host 是 api.anthropic.com，则返回 null
       -> 读取 new URL(ANTHROPIC_BASE_URL).hostname.toLowerCase()
       -> known = edp().some(host === d || host.endsWith("." + d))
       -> labKw = tdp().some(host.includes)
       -> cnTZ = timezone 是 Asia/Shanghai 或 Asia/Urumqi
  -> odp(known, labKw)
       false, false -> U+0027 '
       true,  false -> U+2019 ’
	       false, true  -> U+02BC ʼ
	       true,  true  -> U+02B9 ʹ
	  -> cnTZ 为 true 时把日期分隔符从 "-" 换成 "/"
	  -> `Today${apostrophe}s date is ${date}.`
```

这条链最多形成 3 bit marker：`known` 与 `labKw` 通过 apostrophe 变体编码，`cnTZ` 通过日期分隔符编码。它可以被描述为 `currentDate` 的自然语言 prompt marker，但不能直接写成完整封控、反蒸馏或账号处罚机制。

名单本身不是明文数组，而是：

```text
Wla(base64)
  -> Buffer.from(base64, "base64")
  -> 每个 byte XOR 91
  -> split(",")
```

当前本地实测解出的数量：

- domain list：147 项，开头包括 `cn / sankuai.com / netease.com / 163.com / baidu-int.com / baidu.com / alibaba-inc.com / alipay.com / antgroup-inc.cn / kuaishou.com / bytedance.net`
- keyword list：11 项，分别是 `deepseek / moonshot / minimax / xaminim / zhipu / bigmodel / baichuan / stepfun / 01ai / dashscope / volces`

完整解码结果如下，顺序保持 bundle 中的原始顺序：

```text
# domain list
cn
sankuai.com
netease.com
163.com
baidu-int.com
baidu.com
alibaba-inc.com
alipay.com
antgroup-inc.cn
kuaishou.com
bytedance.net
xiaohongshu.com
ctripcorp.com
jd.com
jdcloud.com
bilibili.co
iflytek.com
stepfun-inc.com
aliyuncs.com
cn-shanghai.fcapp.run
cn-beijing.fcapp.run
xaminim.com
moonshot.ai
anyrouter.top
packyapi.com
aicodemirror.com
aigocode.com
hongshan.com
iwhalecloud.com
dhcoder.net
lemongpt.top
zhihuiapi.top
intsig.net
high-five-ai.xyz
cloudsway.net
4sapi.com
529961.com
88996.cloud
88code.ai
88code.org
91code.pro
992236.xyz
ai.codeqaq.com
ai.hybgzs.com
ai.kjvhh.com
aicanapi.com
aicoding.sh
aifast.site
aihubmix.com
anmory.com
api.5202030.xyz
api.ablai.top
api.bianxie.ai
api.bltcy.ai
api.cpass.cc
api.dev88.tech
api.dreamger.com
api.expansion.chat
api.gueai.com
api.holdai.top
api.ikuncode.cc
api.lconai.com
api.linkapi.org
api.mkeai.com
api.nekoapi.com
api.oaipro.com
api.ruyun.fun
api.ssopen.top
api.tu-zi.com
api.uglycat.cc
api.v3.cm
api.whatai.cc
api.wpgzs.top
api.xty.app
api.yuegle.com
api.zzyu.me
apimart.ai
apipro.maynor1024.live
apiyi.com
applyj.hiapi.top
augmunt.com
b4u.qzz.io
clauddy.com
claude-code-hub.app
claude-opus.top
claudeide.net
co.yes.vg
code.wenwen-ai.com
code.x-aio.com
codeilab.com
cubence.com
deeprouter.top
dimaray.com
dmxapi.com
docs.aigc2d.com
duckcoding.com
fk.hshwk.org
flapcode.com
foxcode.hshwk.org
foxcode.rjj.cc
fuli.hxi.me
getgoapi.com
gpt.zhizengzeng.com
gptgod.cloud
gptkey.eu.org
gptpay.store
hdgsb.com
henapi.top
instcopilot-api.com
jeniya.top
jiekou.ai
kg-api.cloud
n1n.ai
new-api.u4vr.com
new.xychatai.com
one-api.bltcy.top
one.ocoolai.com
oneapi.paintbot.top
open.xiaojingai.com
openclaude.me
opus.gptuu.com
poloai.top
poloapi.top
privnode.com
proxyai.com
qinzhiai.com
right.codes
runanytime.hxi.me
sssaicode.com
store.zzyus.top
tiantianai.pro
uiuiapi.com
uniapi.ai
vip.undyingapi.com
wolfai.top
wzw.de5.net
wzw.pp.ua
xairouter.com
xaixapi.com
xiaohuapi.site
xiaohumini.site
xy.poloapi.com
yansd666.com
yansd666.top
yunwu.ai
yunwu.zeabur.app
zenmux.ai

# keyword list
deepseek
moonshot
minimax
xaminim
zhipu
bigmodel
baichuan
stepfun
01ai
dashscope
volces
```

这份名单只在 `rdp()` 读取 `ANTHROPIC_BASE_URL` 的 hostname 后参与 `known / labKw` 判定；当前没有看到它被 WebFetch、WebSearch、MCP、plugin marketplace 或普通 HTTP URL 访问路径复用。因此它不应被写成“危险第三方 URL 阻断名单”。更准确的说法是：当前 bundle 内置了一份第三方/国内相关 domain 与 keyword 识别表，用于给 `currentDate` 的自然语言文本选择不同 apostrophe 与日期格式。

这个标记位于 `userContext.currentDate`，随后由 `otr(...)` 包成前置 `<system-reminder>` user meta message。就本地代码证据而言，它不是 `payload.system` 字段里的直接拼接，也不能单独证明服务端如何使用该标记、是否关联任何封号动作或反蒸馏策略。

版本边界要保守写：

- `2.1.197` 的当前本地 bundle 已确认存在这条链。
- `2.1.201` 的 npm native 包抽样核对已看不到 `Asia/Shanghai / Asia/Urumqi` 字面量，`currentDate` 附近恢复成普通 `Today's date is`。
- `2.1.91` 起点尚未逐版本验证，不能作为已证实结论写入恢复口径。

### 和模型名名单的边界

模型名相关名单属于另一条链，不和 `Wla(base64) -> XOR 91` 的 domain/keyword 表共享：

- `availableModels` 是 settings / managed policy 里的可选 allowlist，用来限制用户可选模型。
- `modelOverrides` 是 provider model id 映射层，用来把 Anthropic-style model ID 映射到 provider-specific model ID。
- Claude Code Gateway 的 `models[].upstream_model` 也是显式配置的上游模型映射。

因此“内置第三方 base URL 识别表”和“模型名 allowlist / provider remap / gateway upstream model”需要分开描述：前者只影响 `currentDate` 标记，后者才影响 `--model`、`--fallback-model`、capability 检查和 gateway 转发。

### 其他已发现的 provider / 品牌名单

下面这些名单也在当前 bundle 中成组出现，但不属于 `currentDate` 的 `ANTHROPIC_BASE_URL` 标记链。它们需要单独记录，避免以后把所有 provider / 品牌词都误并到同一种机制里。

1. Claude API bundled skill 的第三方 provider skip markers

这组字符串位于内置 Claude API skill 的触发/跳过条件里。语义是：当任务明确在处理其他 provider 时，不使用这份 Claude/Anthropic API skill 指南。

```text
# provider names in query
OpenAI
GPT
Gemini
Llama
Mistral
Cohere
Ollama

# grep markers in project
openai
langchain_openai
google.generativeai
genai
mistralai
cohere
ollama
```

这不是 runtime provider 选择，也不是 URL 阻断。它是 bundled skill 的任务路由边界：避免在非 Anthropic provider 项目里强行套用 Claude API SDK 写法。

2. code indexing tool telemetry 的工具/品牌归一化名单

`TRa(...)` 会从 CLI 命令名中识别 code indexing / code search 工具；`vRa(...)` 会从 MCP server name 中识别同类工具。命中后只看到写入 `tengu_code_indexing_tool_used` telemetry，区分 `source: cli / mcp` 与 `success`。这说明它是工具使用归因，不是权限、模型或 base URL 风险判定。

CLI command / `npx` / `bunx` shortcut 映射：

```text
src -> sourcegraph
cody -> cody
aider -> aider
tabby -> tabby
tabnine -> tabnine
augment -> augment
pieces -> pieces
qodo -> qodo
aide -> aide
hound -> hound
seagoat -> seagoat
bloop -> bloop
gitloop -> gitloop
q -> amazon-q
gemini -> gemini
```

MCP server name pattern 映射：

```text
sourcegraph -> sourcegraph
cody -> cody
openctx -> openctx
aider -> aider
continue -> continue
github-copilot / copilot -> github-copilot
cursor -> cursor
tabby -> tabby
codeium -> codeium
tabnine -> tabnine
augment-code / augment -> augment
windsurf -> windsurf
aide / codestory -> aide
pieces -> pieces
qodo -> qodo
amazon-q -> amazon-q
gemini-code-assist / gemini -> gemini
hound -> hound
seagoat -> seagoat
bloop -> bloop
gitloop -> gitloop
claude-context / code-context -> claude-context
code-index-mcp / code-index -> code-index-mcp
local-code-search -> local-code-search
codebase / autodev-codebase -> autodev-codebase
```

### 同类 prompt 隐写载体的排查边界

以当前 `2.1.197` 的可读 bundle 为准，`currentDate` 是目前唯一能直接闭环到“自然语言 prompt 文本 + 近似不可见字符/同形字符 + `ANTHROPIC_BASE_URL` 条件”的标记链。

已做过的负面排查点包括：

- `U+02B9 / U+02BC` 在可读 bundle 中只出现在 `odp(...)` 的 apostrophe 选择逻辑
- `Today${...}s date is ...` 只看到 `qla(GSe()) -> currentDate` 这一条用户上下文入口
- `\u200B / \u200C / \u200D / \u2066-\u2069 / Default_Ignorable_Code_Point` 的其他命中主要落在第三方解析库、HTML entity、emoji/Unicode 处理、路径/deep-link 清洗或拒绝逻辑里
- `Lkp(...)`、`Z7(...)` 与 deep-link cwd 校验会清理或拒绝 invisible / bidirectional control characters；这类代码更像防输入混淆，不是新增水印载体
- `Buffer.from(..., "base64")` 与 `String.fromCharCode(... ^ ...)` 的其他命中大多是认证、JWT/base64、资源解码、压缩 blob 或第三方库逻辑；当前只看到 `Wla(...)` 这条名单解码链同时服务于 `currentDate` 标记

因此可以把本地结论写成：**同类 prompt 内隐写，目前只确认 `currentDate` 这一处**。请求头、telemetry、policy limits、trusted device、model off-switch 等机制都和反滥用有关，但它们不是同一种“藏进自然语言 prompt 的标记”。这条 marker 具备随正常 prompt 发送并被服务端解码的条件；服务端是否实际解码、如何用于风控或蒸馏识别，仍是 bundle 外黑箱。

## `cachedClaudeMdContent` 不是扫描缓存，而是“渲染后快照”

这项以前容易被误读成：

- `Qv()` 的文件级结果缓存
- 或 `CLAUDE.md` 原始内容缓存

现在可以收紧为：

- `hS()` 在得到 `K = B4t(N4t(await Qv()))` 之后
- 直接执行 `$c8(K || null)`
- 也就是把**最终渲染完成、准备用于 `userContext.claudeMd` 的整段字符串**写入全局状态

因此它缓存的不是：

- 文件数组
- section 列表
- 未拼装的原始 markdown

而是：

- **主线程最近一次生成出来的完整 CLAUDE.md 渲染文本**

当前本地直接可见的消费点也只有一条核心链：

- `HK_()` 读取 `Oc8()`
- 包成一段 auto-mode classifier 的 `<user_claude_md>` user message
- 并给这段 classifier 输入再打上 `cache_control`

因此更稳的结论是：

- `cachedClaudeMdContent` 更像**给 auto-mode / classifier 旁路复用的用户指令快照**
- 不是 `Qv()` 的替代缓存层

### `_H(...)` 当前已直接确认

`_H(...)` 的实际返回形状目前可以写得更精确：

```text
gitStatus =
  remote mode or git instructions disabled or unsupported repo state
    ? null
    : await dlo()

injection = null

return {
  ...(gitStatus ? { gitStatus } : {}),
  ...(CLAUDE_CODE_PERFORCE_MODE ? { perforceMode } : {}),
  ...{}
}
```

因此当前已直接确认：

- `systemContext` 当前本地 bundle 里直接实锤的默认字段是 `gitStatus? / perforceMode?`
- `perforceMode` 只受 `CLAUDE_CODE_PERFORCE_MODE` 控制，文本提示只读文件要通过 `p4 edit` checkout，不是 `_H(...)` 的第二注入槽位
- `_H(...)` 仍残留一个“第二动态槽位”形状：
  - 入参会进入 memo cache key
  - 完成日志里会上报 `has_injection: e !== void 0`
  - 但当前返回对象中对应 spread 仍是空对象
- 因此更严格的结论不是“设计上永远只有 `gitStatus`”，而是“**当前本地可执行路径只看到 `gitStatus? / perforceMode?`；第二注入槽位目前没有活字段写入返回值**”
- 就这一页最关心的 discovery 语义来说，还能再补一句负面证据：
  - 当前没有看到 compat 文件扫描结果、`Qv()` 产物，或其他 instruction source 流向这个预留槽位
  - 因此 `_H(...)` 的预留槽位目前**看不出与 compat discovery 有直接关系**
- `Explore / Plan` 在 `_3(...)` 里裁掉的也正是这个 `gitStatus`
- 本地 bundle 中 `currentDate / gitStatus / perforceMode / claudeMd` 的后续命中已经基本穷尽
  - `currentDate` 只看到 `hS()` 生成，没有看到后续专门裁剪
  - `gitStatus` 只看到 `_H(...)` 生成与 `_3(...)` 中 `Explore / Plan` 裁剪
  - `perforceMode` 只看到 `_H(...)` 按 env 生成，以及 `Explore / Plan` 裁剪 `gitStatus` 后随 `rest` 继续保留
  - `claudeMd` 只看到 `hS()` 生成与 `_3(...)` 中 `omitClaudeMd` 裁剪

### `dlo()` / `gitStatus` 当前已直接确认

`dlo()` 返回的不是结构化对象，而是一段多行字符串快照。其固定骨架现在可以写成：

```text
This is the git status at the start of the conversation. Note that this status is a snapshot in time, and will not update during the conversation.
Current branch: <current branch>

Main branch (you will usually use this for PRs): <main branch>

Status:
<git status --short output or "(clean)">

Recent commits:
<git log --oneline -n 5 output>
```

因此 `gitStatus` 当前更精确的语义应改写为：

- 它是 **systemContext 里的单个字符串字段**
- 但这个字符串本身封装了 4 段 git 快照信息

目前已确认的 4 段来源为：

- `Current branch`
  - `vM() -> NkA() -> QGK()`
  - 读取当前 HEAD 所在分支；失败时回退成 `"HEAD"`
- `Main branch`
  - `EE() -> ykA() -> lGK()`
  - 优先级为：
    - `refs/remotes/origin/HEAD`
    - `origin/main`
    - `origin/master`
    - 最后回退 `"main"`
- `Status`
  - 直接执行 `git --no-optional-locks status --short`
- `Recent commits`
  - 直接执行 `git --no-optional-locks log --oneline -n 5`

额外还可直接确认：

- `Status` 文本超过 `40000` 字符会被截断
- 截断后会附加“如需更多信息请自己运行 `git status`”的提示
- 因此 `tb4(...)` 一类统计代码读取 `Y.gitStatus?.length` 时，拿到的是**字符串长度**，不是结构化对象大小

## `claudeMdExcludes`

设置中存在：

- `ClaudeMdExcludes: string[]`

语义明确：

- 对绝对路径或 glob 做排除
- 只影响 `User / Project / Local`
- 不影响 `Managed / policy`
- 匹配基于归一化后的绝对路径，并会补充 realpath 变体

## 更精确的作用点

`claudeMdExcludes` 是在 `xy()` 递归处理单个 memory 文件时生效的。  
因此它不只是“最终展示前过滤”，而是会直接阻止某些 `CLAUDE.md` / `CLAUDE.local.md` / `.claude/rules/*` 进入收集链。

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
