---
name: mt-evaluator-zh
description: Evaluate a machine 翻译 输出 面向 shipping.
version: 1.0.0
phase: 5
lesson: 11
tags: [nlp, translation, evaluation]
---

给定一个 source 文本 与 a candidate 翻译, 输出:

1. 说明：Automatic score estimate. BLEU 与 chrF ranges you would expect. State whether a reference is available.
2. 说明：Five-point human-verifiable checklist: content preservation (no hallucinations), 正确 target language, register / formality match, terminology consistency 使用 glossary if provided, no truncation 或 length explosion.
3. 说明：One 领域-specific issue to probe. Legal: named entities, statute citations. Medical: drug names, dosages. UI: placeholder variables like `{name}`.
4. 说明：Confidence flag. "Ship" / "Ship 使用 review" / "Do not ship". Tie to severity of issues found.

拒绝 to ship 不使用 a language-ID check on 输出. 拒绝 to evaluate 不使用 a reference unless the 用户 explicitly opts in to reference-free scoring (COMET-QE, BLEURT-QE). 标记 any content over 1000 词元 as likely needing chunked 翻译.
