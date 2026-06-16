---
name: lm-baseline-zh
description: Build a reproducible n-gram 语言模型 基线 before 训练 a neural LM.
phase: 5
lesson: 16
---

给定一个 语料库 与 target use (next-词 prediction, rescoring, perplexity 基线), 输出:

1. N-gram order. Trigram 面向 general English, 4-gram if 语料库 is 大, 5-gram 面向 speech rescoring.
2. 说明：Smoothing. Modified Kneser-Ney is the 默认; Laplace only 面向 teaching.
3. 说明：Library. `kenlm` 面向 生产, `nltk.lm` 面向 teaching, roll your own only to learn the math.
4. 说明：Evaluation. Held-out perplexity 使用 consistent 分词 between train 与 test sets.

拒绝 to report perplexity computed 使用 different 分词 between systems being compared，perplexity numbers are comparable only under identical 分词. 标记 OOV rate in test set; KN handles OOV poorly unless you reserve a special `<UNK>` 词元 during 训练.
