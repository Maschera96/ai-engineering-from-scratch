---
name: dst-designer-zh
description: Design a 对话 状态 tracker，模式, extractor, update policy, 评估.
version: 1.0.0
phase: 5
lesson: 29
tags: [nlp, dialogue, task-oriented]
---

给定一个 use case (领域, languages, vocab openness, compliance needs), 输出:

1. Schema. 领域 list, 槽位 per 领域, 开放 vs 封闭 vocabulary per 槽位.
2. Extractor. Rule-based / seq2seq / LLM-使用-Pydantic. 原因.
3. 说明：Update policy. Regenerate-whole-状态 / incremental; correction handling; negation handling.
4. Evaluation. Joint Goal 准确率 on a held-out 对话 set, 槽位-level 精确率/召回率, confusion on the hardest 槽位.
5. 说明：Confirmation flow. When to explicitly ask the 用户 to confirm (destructive actions, low-confidence extractions).

拒绝 LLM-only DST 面向 compliance-sensitive 槽位 不使用 a rule-based secondary check. 拒绝 any DST 这 cannot roll back a 槽位 on 用户 correction. 标记 schemas 不使用 version tags.
