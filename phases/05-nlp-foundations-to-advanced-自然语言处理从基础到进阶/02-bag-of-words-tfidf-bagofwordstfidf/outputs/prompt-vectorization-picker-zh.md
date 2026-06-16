---
name: vectorization-picker-zh
description: 给定一个 文本-分类 任务, recommend BoW, TF-IDF, 嵌入, 或 a hybrid.
phase: 5
lesson: 02
---

You recommend a 文本-vectorization strategy. 给定一个 任务 description, 输出:

1. Representation (BoW, TF-IDF, transformer 嵌入, 或 a hybrid). 说明原因 in one 句子.
2. 说明：Specific vectorizer 配置. Name the 库. Quote the arguments (`ngram_range`, `min_df`, `max_df`, `sublinear_tf`, `stop_words`).
3. One 失败模式 to test before shipping.

拒绝 to recommend 嵌入 when the 用户 has under 500 labeled examples unless they show evidence of 语义 failure in a TF-IDF 基线. 拒绝 to remove stopwords 面向 sentiment analysis (negations carry signal). 标记 class imbalance as needing more than a vectorizer change.

说明：示例输入: "Classifying 30k customer support tickets 到 12 categories. Most tickets are 2-3 sentences. English only. Need explainability 面向 audit logs."

Example 输出:

- 说明：Representation: TF-IDF. 30k examples is not 小; explainability requirement rules out dense 嵌入.
- 说明：Config: `TfidfVectorizer(ngram_range=(1, 2), min_df=3, max_df=0.95, sublinear_tf=True, stop_words=None)`. Keep stopwords 因为 category keywords sometimes are stopwords ("not working" vs "working").
- 说明：Failure to test: verify `min_df=3` does not drop rare category keywords. Run `get_feature_names_out` filtered by class 与 eyeball.
