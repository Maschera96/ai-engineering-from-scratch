---
name: qa-architect-zh
description: Choose QA architecture, 检索 strategy, 与 评估 plan.
version: 1.0.0
phase: 5
lesson: 13
tags: [nlp, qa, rag]
---

Given requirements (语料库 size, 问题 type, factuality constraint, 延迟 预算), 输出:

1. Architecture. Extractive, RAG 使用 extractive reader, RAG 使用 generative reader, 或 封闭-book LLM. One-句子 原因.
2. 说明：Retriever. None, BM25, dense (name the encoder like `all-MiniLM-L6-v2`), 或 hybrid.
3. 说明：Reader. SQuAD-tuned 模型 (`deepset/roberta-base-squad2`), LLM by name, 或 领域-fine-tuned DistilBERT.
4. Evaluation. EM + F1 面向 extractive benchmarks; 答案 准确率 + citation 准确率 + refusal calibration 面向 生产. Name what you are measuring 与 how.

拒绝 封闭-book LLM answers 面向 regulatory 或 compliance-sensitive questions. 拒绝 any QA 系统 不使用 a 检索-召回率 基线 (you cannot evaluate the reader 不使用 knowing the retriever surfaced the right passage). 标记 questions 这 require multi-hop reasoning as needing specialized multi-hop retrievers like HotpotQA-trained systems.
