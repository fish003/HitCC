# InstructionsLoaded load reason 与 hook 时序

## 本页用途

- 整理 `O4t(...)`、`Tpp()`、`X5e(...)` 的真实关系。
- 说明 `compact/session_start/include` 等 load reason 的消费时机与一次性边界。

## 相关文件

- [../03-prompt-assembly-and-context-layering.md](../03-prompt-assembly-and-context-layering.md)
- [../04-non-main-thread-prompt-paths.md](../04-non-main-thread-prompt-paths.md)
- [../../05-appendix/01-glossary.md](../../05-appendix/01-glossary.md)

## `O4t(...)` 不是立即触发 hook，而是为“下一次主扫描”预置 load reason

这一点现在也可以从 bundle 里写得更硬。

`O4t(reason)` 当前本地只有三件事：

```text
sC1 = reason
tC1 = true
Qv.cache?.clear?.()
```

也就是说它不会直接执行 `X5e(...)`。  
真正消费这个 reason 的地方，在 `Qv()` 主扫描尾部：

```text
let f = Tpp()
if (f !== undefined && F4t()) {
  for (W of K) {
    if (!fQ9(W.type)) continue
    let G = W.parent ? "include" : f
    X5e(W.path, W.type, G, { globs: W.globs, parentFilePath: W.parent })
  }
}
```

这能直接推出三条关键时序结论：

1. `compact` 不是 compact 完成时立即发 `InstructionsLoaded`
   - 它只是先通过 `O4t("compact")` 预置“下一次主扫描的默认 reason”
2. 真正发 hook，要等下一次普通 `Qv()` 主扫描完整跑完
   - 收集完 `K` 之后才统一逐文件 `X5e(...)`
3. 即使默认 reason 已是 `compact`
   - 只要某个文件是通过 `@include` 进入
   - 它最后发出的 `load_reason` 仍会被覆写成 `include`

这里还要再补一个容易漏掉的细节：

- `Tpp()` 的调用发生在 `F4t()` 判断之前
- 也就是当前 reason 槽位是**一次性消费**
- 即使当前根本没有 `InstructionsLoaded` hook，`Tpp()` 也会把它读走并重置回 `session_start`

因此当前本地更稳的说法应是：

- **`O4t(...)` 是“下一次主扫描的 load-reason setter”**
- **不是 hook dispatcher**
- **`compact` / `session_start` 这类主 reason 都是 `Qv()` 末尾一次性消费的**

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
