# Telemetry 初始化与 remote settings gating

## 本页用途

- 整理 `Rg8 -> H4A -> qvz -> initializeTelemetry` 启动链。
- 说明 remote managed settings 为什么会延迟 telemetry 初始化，以及哪些入口会触发 eager init。

## 相关文件

- [../05-model-adapter-provider-and-auth.md](../05-model-adapter-provider-and-auth.md)
- [../06-stream-processing-and-remote-transport.md](../06-stream-processing-and-remote-transport.md)
- [../../02-execution/06-context-runtime-and-tool-use-context.md](../../02-execution/06-context-runtime-and-tool-use-context.md)
- [../../05-appendix/02-evidence-map.md](../../05-appendix/02-evidence-map.md)

## 一句话结论

Claude Code CLI 里的 telemetry 不是一个单开关系统，而是至少分成四层：

- request lifecycle telemetry：`NDl / KOo / UDl / F7t`
- startup telemetry：`tengu_startup_telemetry`
- 3P OTEL metrics / logs / traces：`initializeTelemetry()`
- 1P event logging / GrowthBook event logger：独立于 `initializeTelemetry()` 的另一条链

因此 `DISABLE_TELEMETRY` 与 `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` 的效果并不相同，也不等于“所有 telemetry 都停了”。

## 初始化入口：`Rg8 -> H4A -> qvz -> initializeTelemetry`

当前本地 bundle 里，3P telemetry 初始化主入口已经可以写成：

```text
startup init
  -> Rg8()
    -> if remote managed settings gating active:
         maybe eager-init beta tracing
         wait tL8()
         -> SQ()
         -> H4A()
       else:
         -> H4A()

H4A()
  -> guard UI4
  -> qvz()
    -> import { initializeTelemetry } from Nl1
    -> meter = await initializeTelemetry()
    -> if meter:
         Fd8(meter, createCounter)
```

这里能直接确认：

- `H4A()` 只是一次性初始化 guard，不负责具体 exporter 组装
- `qvz()` 才是真正调用 `initializeTelemetry()` 的地方
- `initializeTelemetry()` 返回的是 meter；只有 meter 存在时，`Fd8(...)` 才会注册 counters

## `Rg8()` 在启动期何时触发

这一点现在也可以写得更实，不必只停留在“startup init”。

- TUI 主启动路径里，会先做 `SQ()`，再调用 `Rg8()`
- headless / `--print` / `stream-json` 主路径里，同样会先做 `SQ()`，再调用 `Rg8()`

因此更稳的结论是：

- telemetry 初始化属于启动早期动作
- 发生在 settings 已经同步到本地运行态之后
- 不依赖主对话循环已经开始

另外 `Rg8()` 在 remote managed settings 分支里还有一个容易漏掉的细节：

- 若 `CF1()` 为真，且 `lA() && rj()` 成立，会先尝试一次 `H4A()`
- 这次 eager init 失败只记日志，不会阻止后续 `tL8() -> SQ() -> H4A()` 的正式路径

所以 beta tracing 在特定入口下并不一定严格等到 remote settings 完成后才首次初始化。

## remote managed settings 为什么会阻塞 telemetry init

这部分现在已经不再只是现象描述。

### `Tu()`：是否需要 remote managed settings

`CF1()` 只是 `Tu()` 的薄包装。  
`Tu()` 当前至少要求：

- provider 是 `firstParty`
- 当前 host 仍是 Anthropic first-party host
- 不是 `local-agent` 入口
- 且满足下列任一条件：
  - OAuth token 存在，但 `subscriptionType === null`
  - OAuth token 存在，且 scopes 包含 enterprise 相关 scope，并且 `subscriptionType` 是 `enterprise / team`
  - 存在 API key

因此更稳的理解不是“登录了就一定等 remote settings”，而是：

- 只有 first-party 且具备 enterprise/team/待判定账户语义时，才进入 remote managed settings 路径

### `EBq()` / `tL8()` / `yBq()`

一旦 `CF1()` 为真，启动期会先执行 `EBq()`：

- 若还没有 loading promise，则创建 `C$6`
- 同时保存 resolver `TC`
- 超时后会强制 resolve，避免无限阻塞

之后：

- `tL8()` 只是 `await C$6`
- `yBq()` / `eL8()` 在 remote settings 刷新完成后 resolve `TC`
- `policySettings` 改变时会触发 `qX.notifyChange("policySettings")`

所以 telemetry 被“延迟初始化”的真实原因不是初始化函数内部主动等待，而是：

- 外层 `Rg8()` 先等待 remote settings loading promise
- promise resolve 后调用 `SQ()` 重刷 `process.env`
- 然后才进入 `H4A()`

### `SQ()` 在这里的作用

`SQ()` 当前可直接确认会：

- 把 `P8().env` 合回 `process.env`
- 把 `$A()?.env` 合回 `process.env`
- 然后刷新若干依赖 `process.env` 的运行态缓存

所以 remote managed settings 对 telemetry 初始化的影响，不是“传一个 settings 对象进去”，而是：

- 先把远端/本地 settings 合并进 `process.env`
- 再让 `initializeTelemetry()` 读取已经更新后的 OTEL / auth / proxy 相关环境变量

## `initializeTelemetry()`：实际做了什么

`initializeTelemetry()` 对应 `Wb_()`。  
它至少做四组事：

### 1. 预处理 env

- `bootstrapTelemetry()` / `_14()` 会默认把 `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` 设成 `delta`
- 若 `B$A()` 条件成立，会把 console exporter 从 `OTEL_*_EXPORTER` 中剔掉
- 安装 OTEL diag logger：`YO6.diag.setLogger(new bQ1, ERROR)`

### 2. 构造 meter provider

无论 `CLAUDE_CODE_ENABLE_TELEMETRY` 是否开启，`Wb_()` 都会创建 `MeterProvider`。

区别只在 reader/exporter 来源：

- `q = yXa()` 只看 `CLAUDE_CODE_ENABLE_TELEMETRY`
- `q === true`
  - 加入 `Jb_()` 产出的标准 metrics exporters
- `Db_() === true`
  - 再额外加入 `Xb_()` reader

这里最重要的结论是：

- `CLAUDE_CODE_ENABLE_TELEMETRY` 控制的是 OTEL exporter 组装
- 不是“是否创建 telemetry runtime 对象”的总开关

### 3. 条件性创建 event logger

标准路径里，只有 `q === true` 且 `Mb_()` 返回了至少一个 log exporter，才会：

- 创建 `LoggerProvider`
- `Dq8(loggerProvider)`
- `fq8(logger)` 设置全局 event logger

而 `GO(eventName, attrs)` 是否真的能发出事件，完全取决于：

- `ld8()` 是否返回已初始化的 event logger

如果没有 logger：

- `GO(...)` 会直接丢事件，并打印一次 `Event dropped (no event logger initialized)`

### 4. 条件性创建 tracer provider

traces 有两条独立路径：

- 标准 OTEL traces
  - 条件：`q === true && HS1()`
- beta tracing 旁路
  - 条件：`rj() === true`
  - 依赖 `ENABLE_BETA_TRACING_DETAILED + BETA_TRACING_ENDPOINT`

beta tracing 旁路 `fb_(resource)` 会直接：

- 建 `OTLPTraceExporter`
- 建 `OTLPLogExporter`
- 设置全局 tracer provider
- 设置全局 logger provider
- `fq8(logger)` 直接装 event logger

因此更稳的理解是：

- beta tracing 可以绕过标准 `CLAUDE_CODE_ENABLE_TELEMETRY` 开关，单独把 tracer/logger 拉起来

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
