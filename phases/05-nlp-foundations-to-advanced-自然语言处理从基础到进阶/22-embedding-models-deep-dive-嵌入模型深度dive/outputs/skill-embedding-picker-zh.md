---
name: embedding-picker-zh
description: Pick 嵌入 模型, dimension, 与 检索 mode 面向 a given 语料库 与 deployment.
version: 1.0.0
phase: 5
lesson: 22
tags: [nlp, embeddings, retrieval]
---

给定一个 语料库 (size, languages, 领域, avg length), deployment target (云端 / 边缘 / on-prem), 延迟 预算, 与 存储 预算, 输出:

1. 模型. Named checkpoint 或 API. One-句子 原因.
2. 说明：Dimension. Full / Matryoshka-truncated / int8-quantized. 原因 tied to 存储 预算.
3. Mode. Dense / sparse / multi-向量 / hybrid. 原因.
4. 说明：Query prefix / template if required by the 模型 card.
5. 说明：Evaluation plan. MTEB tasks relevant to 领域 + held-out 领域 eval 使用 nDCG@10.

拒绝 recommendations 这 truncate Matryoshka to <64 dims 不使用 领域 validation. 拒绝 ColBERTv2 面向 corpora under 10k passages (overhead not justified). 标记 长-文档 corpora (>8k 词元) routed to models 使用 512-词元 windows.
