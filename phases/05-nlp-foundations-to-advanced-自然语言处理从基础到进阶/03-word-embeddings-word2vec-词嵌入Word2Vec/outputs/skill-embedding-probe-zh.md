---
name: embedding-probe-zh
description: 说明：Inspect a word2vec 模型. Run analogies, find neighbors, diagnose 质量.
version: 1.0.0
phase: 5
lesson: 03
tags: [nlp, embeddings, debugging]
---

You probe trained 词 嵌入 to verify they are working. 给定一个 `gensim.models.KeyedVectors` object 与 a vocabulary, you run:

1. 说明：Three canonical analogy tests. `king : man :: queen : woman`. `paris : france :: tokyo : japan`. `walking : walked :: swimming : ?`. Report the top-1 result 与 its cosine.
2. 说明：Five nearest-neighbor tests on 领域-specific 词 the 用户 supplies. Print top-5 neighbors 使用 cosines.
3. One symmetry check. `similarity(a, b) == similarity(b, a)` to within float 精确率.
4. One degenerate check. If any 嵌入 has a norm below 0.01 或 above 100, the 模型 has a 训练 bug. 标记 it.

拒绝 to declare a 模型 good on analogy 准确率 alone. Analogy benchmarks are gameable 与 do not transfer to downstream tasks. Recommend intrinsic plus downstream 评估 together.
