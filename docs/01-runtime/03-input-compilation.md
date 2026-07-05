# 输入编译

## 本页用途

- 作为 `AU8(...)`、`ihz(...)`、block family、slash command 与 `UserPromptSubmit` hook 的导航页。
- 原正文已经拆成三个子页；本页只维护输入编译的阅读顺序和专题边界。

## 相关文件

- [04-agent-loop-and-compaction.md](./04-agent-loop-and-compaction.md)
- [../02-execution/01-tools-hooks-and-permissions.md](../02-execution/01-tools-hooks-and-permissions.md)
- [../02-execution/05-attachments-and-context-modifiers.md](../02-execution/05-attachments-and-context-modifiers.md)

## 拆分后的主题边界

### AU8 / ihz 预处理与主分流

见：

- [03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md](03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md)

集中放 `AU8(...)` 入参/返回、`ihz(...)` 归一化、remote slash gate、attachment loading 与三条主分流。

### Block family、BU4 与 slash command 编译结果

见：

- [03-input-compilation/02-block-family-bu4-and-slash-command-results.md](03-input-compilation/02-block-family-bu4-and-slash-command-results.md)

集中放本地 block producer 边界、`BU4(...)` 普通输入消息构造与 slash command 编译结果。

### UserPromptSubmit hook 与最终边界

见：

- [03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md](03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md)

集中放 `UserPromptSubmit` hook 的精确位置、输出合并、截断、去重、阻断规则与未决边界。

## 建议阅读顺序

1. 先看 [01-au8-ihz-preprocessing-and-main-branches.md](./03-input-compilation/01-au8-ihz-preprocessing-and-main-branches.md)，建立输入编译前段与主分流模型。
2. 再看 [02-block-family-bu4-and-slash-command-results.md](./03-input-compilation/02-block-family-bu4-and-slash-command-results.md)，确认消息构造与 slash command 结果形态。
3. 最后看 [03-userpromptsubmit-hooks-and-final-boundaries.md](./03-input-compilation/03-userpromptsubmit-hooks-and-final-boundaries.md)，复核 hook 时序和最终边界。

## 与其它专题的边界

- attachment payload 的生命周期与 materialize 细节，见 [../02-execution/05-attachments-and-context-modifiers.md](../02-execution/05-attachments-and-context-modifiers.md)。
- 工具执行、hook schema 与权限系统，见 [../02-execution/01-tools-hooks-and-permissions.md](../02-execution/01-tools-hooks-and-permissions.md)。
- 主循环如何消费编译后的用户输入，见 [04-agent-loop-and-compaction.md](./04-agent-loop-and-compaction.md)。

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
