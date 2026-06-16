---
name: structured-output-designer-zh
description: 为自由文本抽取目标设计一个兼容 strict 模式的 JSON Schema 和 Pydantic 模型，并预置带类型的拒绝与重试处理。
version: 1.0.0
phase: 13
lesson: 04
tags: [structured-output, json-schema, pydantic, strict-mode, extraction]
---

给定一个自由文本抽取目标（发票、简历、客服工单、研究摘要），产出一份生产可用的抽取契约：JSON Schema 2020-12、Pydantic 模型、拒绝处理器和重试策略。

产出内容：

1. JSON Schema 2020-12。每个属性都有类型。`required` 列出每一个属性。每个对象都设置 `additionalProperties: false`。对闭合取值集合使用枚举。不使用 `$ref`。不使用含糊的 `oneOf` / `anyOf`。需通过 OpenAI strict 模式要求的校验。
2. Pydantic v2 BaseModel。用 Python 类型镜像该模式。`model_json_schema()` 必须产出与 (1) 等价的模式。
3. 拒绝处理器。带类型的 `Refusal(reason: str, category: str)` 结果。列出类别：`safety`、`input_mismatch`、`insufficient_info`。
4. 重试策略。三种重试形态：(a) 注入校验错误并重试一次（在 strict 模式之外）；(b) 将拒绝接受为最终结果（strict 模式）；(c) 在反复拒绝时升级到更强的模型。
5. 测试向量。十个输入，覆盖正常路径、对抗性字段、部分输入，以及一个触发拒绝的用例。每个都附带预期结果。

硬性拒绝项：
- 任何含未定类型字段的模式。strict 模式和校验器都会失败。
- 任何缺少 `additionalProperties: false` 的模式。会泄漏幻觉内容。
- 任何使用 `oneOf` 却没有判别字段（discriminator）的模式。解码含糊。
- 任何未检查其 JSON Schema 往返一致性的 Pydantic 模型。

拒绝规则：
- 如果目标领域在没有书面用途说明的情况下包含个人可识别数据，拒绝并转交 Phase 18（ethics）处理合法依据论证。
- 如果用户要求一个无法用 JSON Schema 2020-12 表达的模式（例如递归的任意图结构），拒绝并提出最接近的可表达放宽方案。
- 如果抽取目标是"从任意内容中抽取结构化数据"，拒绝并询问具体领域。

输出：一页契约，包含模式 JSON、Pydantic 类、拒绝与重试策略，以及十个测试向量。结尾附一条说明：首先针对哪个 provider 以及原因。
