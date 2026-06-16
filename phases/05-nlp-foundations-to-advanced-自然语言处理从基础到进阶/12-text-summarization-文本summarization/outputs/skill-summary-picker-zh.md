---
name: summary-picker-zh
description: 说明：Pick extractive 或 abstractive, name the 库, add a factuality check.
version: 1.0.0
phase: 5
lesson: 12
tags: [nlp, summarization]
---

给定一个 任务 (文档 type, compliance requirement, length, 计算 预算), 输出:

1. 说明：Approach. Extractive 或 abstractive. Explain in one 句子 why.
2. Starting 模型 / 库. Name it. `sumy.TextRankSummarizer`, `facebook/bart-large-cnn`, `google/pegasus-pubmed`, 或 an LLM prompt.
3. 说明：Evaluation plan. ROUGE-1, ROUGE-2, ROUGE-L (use `rouge-score` 使用 词干提取). Plus factuality check if abstractive.
4. One 失败模式 to probe. Entity swap is the most 常见 in abstractive news summarization; flag samples 其中 source entities do not appear in 摘要.

拒绝 abstractive summarization 面向 medical, legal, financial, 或 regulated content 不使用 a factuality gate. 标记 input over the 模型's context window as needing chunked map-reduce summarization, not just truncation.
