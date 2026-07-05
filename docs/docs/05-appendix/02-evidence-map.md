# 证据索引

> 本页是 evidence map 的导航入口。具体证据登记已经拆到子页，避免单个附录页继续膨胀。

## 本页用途

- 统一说明 evidence map 的维护入口与拆分边界。
- 保留当前发行版入口证据、登记规则、主题到证据映射、关键结论复核入口的阅读顺序。
- 新增运行时结论时，先改对应主题页，再在最贴近的 evidence map 子页登记。

## 相关文件

- [../00-overview/01-scope-and-evidence.md](../00-overview/01-scope-and-evidence.md)
- [../00-overview/02-document-style-and-structure-conventions.md](../00-overview/02-document-style-and-structure-conventions.md)
- [../04-rewrite/02-open-questions-and-judgment.md](../04-rewrite/02-open-questions-and-judgment.md)

## 拆分后的主题边界

### 入口、规则与 request snapshot 速查

见：

- [02-evidence-map/01-entry-rules-and-request-snapshot.md](02-evidence-map/01-entry-rules-and-request-snapshot.md)

集中放证据口径、当前发行版入口证据、断言登记规则，以及上下文缓存 / request snapshot 的专题速查。

### Overview 与 runtime 主题证据映射

见：

- [02-evidence-map/02-overview-and-runtime-topic-map.md](02-evidence-map/02-overview-and-runtime-topic-map.md)

集中放 overview 页面和 `01-runtime` 页面到直接证据来源的对应关系。

### Execution 主题证据映射

见：

- [02-evidence-map/03-execution-topic-map.md](02-evidence-map/03-execution-topic-map.md)

集中放 `02-execution` 页面到直接证据来源的对应关系。

### Ecosystem 与 rewrite 主题证据映射

见：

- [02-evidence-map/04-ecosystem-rewrite-topic-map.md](02-evidence-map/04-ecosystem-rewrite-topic-map.md)

集中放 `03-ecosystem`、`04-rewrite` 页面到直接证据来源的对应关系。

### 关键结论复核入口

见：

- [02-evidence-map/05-key-conclusion-review-and-next-evidence.md](02-evidence-map/05-key-conclusion-review-and-next-evidence.md)

集中放能否重写、网络闭环、prompt/context 主链、request snapshot 与 `contextLayers` 等强结论的复核入口。

## 建议阅读顺序

1. 先看 [01-entry-rules-and-request-snapshot.md](./02-evidence-map/01-entry-rules-and-request-snapshot.md)，确认证据口径和当前发行版入口。
2. 按专题所在目录进入 runtime、execution、ecosystem/rewrite 的证据映射页。
3. 需要复核强结论时，再看 [05-key-conclusion-review-and-next-evidence.md](./02-evidence-map/05-key-conclusion-review-and-next-evidence.md)。

## 与其它专题的边界

- 证据口径、置信度定义和当前恢复范围，仍以 [../00-overview/01-scope-and-evidence.md](../00-overview/01-scope-and-evidence.md) 为准。
- 具体运行机制、调用链和未决点写入对应正文页；evidence map 只做索引和复核入口。
- 阻塞级未知点和能否开工的判断，统一回到 [../04-rewrite/02-open-questions-and-judgment.md](../04-rewrite/02-open-questions-and-judgment.md)。

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
