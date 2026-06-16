---
name: topic-picker-zh
description: Pick LDA 或 BERTopic 面向 a 语料库. Specify 库, knobs, 评估.
version: 1.0.0
phase: 5
lesson: 15
tags: [nlp, topic-modeling]
---

给定一个 语料库 description (文档 count, avg length, 领域, language, 计算 预算), 输出:

1. 算法. LDA / NMF / BERTopic / Top2Vec / FASTopic. One-句子 原因.
2. 配置. Number of 主题 (start at ~sqrt(n_docs)), `min_df` / `max_df` filters, 嵌入 模型 面向 neural approaches.
3. 说明：Evaluation. 主题 coherence (c_v) via `gensim.models.CoherenceModel`, 主题 diversity, plus a 20-sample human read.
4. 失败模式 to probe. 面向 LDA, "junk 主题" absorbing stopwords 与 frequent terms. 面向 BERTopic, -1 离群点 聚类 swallowing ambiguous 文档.

拒绝 BERTopic on 文档 longer than the 嵌入 模型's context window 不使用 a 分块 strategy. 拒绝 LDA on very 短 文本 (tweets, reviews under 10 词元) as coherence collapses. 标记 any n_topics choice below 5 或 above 200 as likely 错误 面向 real 数据.
