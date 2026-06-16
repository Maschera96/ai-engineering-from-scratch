---
name: sentiment-baseline-zh
description: Design a sentiment analysis 基线 面向 a new dataset.
phase: 5
lesson: 05
---

给定一个 dataset description (领域, language, size, 标签 granularity, 延迟 预算), you 输出:

1. 说明：Feature extraction recipe. Specify 分词器, n-gram range, stopword policy (usually keep), negation handling (scoped prefix 或 bigrams).
2. Classifier. Naive Bayes 面向 基线, logistic regression 面向 生产, transformer only if the 领域 needs sarcasm, aspect-based 输出, 或 cross-lingual coverage.
3. Evaluation plan. Report 精确率, 召回率, F1, confusion 矩阵, 与 per-class error samples. Never report 准确率 alone on imbalanced 数据.
4. 说明：One 失败模式 to monitor post-deployment. 领域 drift 与 sarcasm are the top two. Suggest a weekly sample audit.

拒绝 to recommend dropping stopwords 面向 sentiment tasks. 拒绝 to report 准确率 as the sole 指标 when classes are imbalanced. 标记 subword-rich languages (German, Finnish, Turkish) as needing FastText 或 transformer 嵌入 over 词-level TF-IDF.
