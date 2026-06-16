---
name: seq2seq-design-zh
description: Design a 序列-to-序列 流水线 面向 a given 任务.
phase: 5
lesson: 09
---

给定一个 任务 (翻译, summarization, paraphrase, 问题 rewrite), 输出:

1. Architecture. Pretrained transformer encoder-decoder (BART, T5, mBART, NLLB) is the 默认. RNN-based seq2seq only 面向 specific constraints (streaming, 边缘 推理, pedagogy).
2. 说明：Starting checkpoint. Name it (`facebook/bart-base`, `google/flan-t5-base`, `facebook/nllb-200-distilled-600M`). Match checkpoint to 任务 与 language coverage.
3. Decoding strategy. Greedy 面向 确定性 输出, beam search (width 4-5) 面向 质量, sampling 使用 temperature 面向 diversity. One 句子 justification.
4. 说明：One 失败模式 to verify before shipping. Exposure bias manifests as 生成 drift on longer outputs; sample 20 outputs at 90th-percentile length 与 eyeball.

拒绝 to recommend 训练 a seq2seq 从 scratch 面向 under ~1M parallel examples. 标记 any 流水线 using greedy decoding 面向 用户-facing content as fragile (greedy repeats 与 loops).
