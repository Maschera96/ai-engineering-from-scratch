---
name: structured-output-picker-zh
description: 说明：Choose a structured 输出 approach, 模式 design, 与 validation plan.
version: 1.0.0
phase: 5
lesson: 20
tags: [nlp, llm, structured-output]
---

给定一个 use case (provider, 延迟 预算, 模式 complexity, failure tolerance), 输出:

1. 说明：Mechanism. Native vendor structured 输出, Instructor retries, Outlines FSM, 或 XGrammar CFG. One-句子 原因.
2. 说明：Schema design. Field order (reasoning first, 答案 last), nullable fields 面向 "unknown", enum vs regex, required fields.
3. 说明：Failure strategy. Max retries, fallback 模型, graceful `null` handling, out-of-分布 refusal.
4. 说明：Validation plan. Schema compliance rate (target 100%), 语义 validity (LLM-judge), field-coverage rate, 延迟 p50/p99.

拒绝 any design 这 puts `answer` 或 `decision` before reasoning fields. 拒绝 to use bare JSON mode 不使用 a 模式. 标记 recursive schemas behind an FSM-only 库.
