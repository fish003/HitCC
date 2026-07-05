[English](./README.md) | **中文**

# HitCC

## Claude Code CLI v2.1.197 完整逻辑逆向文档

HitCC 不是源码仓库，而是一套面向学习、分析与重写的文档库。  
项目目标不是复刻原始文件树，而是尽可能还原 Claude Code CLI 的核心运行逻辑、模块边界、配置系统和外围生态，从而为“可运行替代品”或“高相似版本重写”提供稳定参考。

本项目与 Anthropic PBC 无任何关系。仓库中不包含 Claude Code 原始源码，不包含破解内容，也不包含绕过产品策略或权限机制的实现。

## 说明

获取本仓库研究的 Claude Code CLI：
```sh
npm pack @anthropic-ai/claude-code@2.1.197
```

`recovery_tools/` 下面的 Python 脚本可以对混淆加密后的源代码做初步整理。

本仓库基于上述方法获得的 Claude Code CLI v2.1.197 本地包，以及从 native binary 中抽取出的 bundle 进行分析；不依赖 Anthropic PBC 提供的任何包括 LLM 推理服务在内的网络服务。

## v2.1.197 更新说明

`docs/` 知识库已经从之前的 v2.1.84 基线更新到 Claude Code CLI v2.1.197。当前证据口径已改为新的发行形态：wrapper package、平台 native binary package，以及从 native executable 的 `.bun` section 中抽取出的 JavaScript bundle。

这次更新同步刷新了运行时入口、命令面、后台/远端能力、Plugin / Skill、TUI、控制面、telemetry、abuse/risk/refusal、tool execution 和 `contextLayers` 等主题。部分证据密集的大页已经拆成“父导航页 + 子专题页”，方便按运行链追证据。

本次也新增了独立专题 [docs/standalone/abuse-control-and-telemetry.md](./docs/standalone/abuse-control-and-telemetry.md)，集中说明客户端侧可见的 abuse/risk control 与 telemetry 证据。该专题把 prompt marker、request identity、telemetry/logging 通道、managed policy limits、执行 gate、remote-control trust、gateway 防护和 refusal/fallback 处理放在同一张图里，同时把私有服务端处罚、审核和模型内部判定保留在证据边界之外。

本仓库文档内容可能偏多，目前统计的 `docs/`（下列数据可能不会及时更新）：
```
TOTAL
  Files: 118
  Lines: 32984
  Chars: 990580
  Bytes: 1319672
```

## 这个仓库提供什么

- 基于 Claude Code CLI v2.1.197 本地包和 extracted bundle 的结构化逆向文档
- 按 runtime、execution、ecosystem、rewrite、appendix 组织的主题化证据库
- 面向学习、分析和高相似重写的边界判断、候选架构与后续补证方向

## 这个仓库不提供什么

- 原始源码文件结构还原
- 私有服务端实现细节
- 1:1 行为复刻保证
- 可直接运行的 CLI、SDK 或安装脚本

## 文档覆盖范围

当前知识库不是按原始源码文件树组织，而是按“可稳定还原的运行时主题”组织。主知识库位于 `docs/docs/`，独立专题位于 `docs/standalone/`。截至目前可以按目录理解为：

- `00-overview`
  - 范围边界、证据来源、置信度口径、文档结构与维护约定
- `01-runtime`
  - CLI 入口、命令树、运行模式分流、Session / Transcript 持久化与恢复
  - 输入编译链、主 Agent Loop、compact 分支、model adapter、provider 选择与 auth
  - stream event、request telemetry/refusal、remote transport、request build boundary
  - `web-search`、`web-fetch` 两类内建网络工具
  - API lifecycle、startup telemetry、1P event logging、非必要流量开关、control plane 与非 LLM 网络链路
  - settings source、路径、合并、缓存、写回、CLI 注入 schema、关键消费面与 abuse/risk/refusal controls
- `02-execution`
  - Tool execution core、并发执行、Hook runtime、Permission / Sandbox / Approval
  - instruction discovery、rules、prompt assembly、context layering、request-level injection boundary
  - 非主线程 prompt 路径、compat agent definition、attachment 生命周期、context runtime 与 tool-use context
- `03-ecosystem`
  - Resume、Fork、Sidechain、Subagent、agent team
  - Remote persistence、bridge、bridge credentials、Plan system
  - MCP、Skill、Plugin、TUI、Monitor / Advisor 及其运行时交互边界
- `04-rewrite`
  - 面向重写工程的候选分层、目录骨架、未决项分级、阻塞判断与后续补证方向
- `05-appendix`
  - glossary、evidence map、topic map、关键结论复核等统一术语和证据索引
- `standalone`
  - 跨目录专题，不挂入主导航树，用于承载横跨 runtime、execution、ecosystem 的证据整理
  - [封控与遥测专题](./docs/standalone/abuse-control-and-telemetry.md)，串联 prompt marker、request identity、telemetry/logging、policy/gate、remote-control trust、gateway 与 refusal/fallback；只描述本地 bundle 可证明的客户端边界

## 当前结论

当前证据边界、重写可行性、未知点和后续补证方向见：

- [范围、证据与结论](./docs/docs/00-overview/01-scope-and-evidence.md)
- [重写判断、未决项分级与后续补证](./docs/docs/04-rewrite/02-open-questions-and-judgment.md)

## 维护原则

本仓库当前采用“知识库”而不是“长篇单文档”维护方式，核心原则如下：

- `00-overview/00-index.md` 只做全局入口，不重复正文
- 父页只做导航与边界说明，不承载长篇机制分析
- 子页负责正文、证据落点、反证与未决项
- 未知点统一收口到范围页与重写判断页，避免在多个目录页重复维护
- 新增证据时，优先更新对应主题页，再回补 glossary 或 evidence map

详细约定见：

- [docs/docs/00-overview/02-document-style-and-structure-conventions.md](./docs/docs/00-overview/02-document-style-and-structure-conventions.md)

## 授权声明

Copyright (c) 2026 Hitmux contributors

本项目采用 [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) 许可协议。

这意味着你可以：

- 复制与传播本项目内容
- 修改、改编并再发布
- 在商业或非商业场景中使用

但你必须：

- 保留原作者或来源署名
- 附上许可证链接
- 说明是否做过修改

正式许可证文本见仓库根目录的 [LICENSE](./LICENSE)。

## 致谢
感谢[Linux Do](https://linux.do)社区的支持

## 免责条款

请仅将本项目用于学习、研究、教学或正当工程分析用途。  
任何将本项目用于侵犯 Anthropic PBC 合法权益或规避产品政策的行为，均与本项目无关，风险自负。

作者不对文档内容的准确性、完整性或因使用该文档导致的任何直接或间接损失（包括但不限于法律风险、技术故障、数据丢失）承担法律责任。
