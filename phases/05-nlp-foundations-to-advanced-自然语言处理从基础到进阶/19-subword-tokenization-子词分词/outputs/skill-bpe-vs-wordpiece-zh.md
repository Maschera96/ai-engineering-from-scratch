---
name: skill-bpe-vs-wordpiece-zh
description: Pick 分词器 算法, vocab size, 库 面向 a given 语料库 与 deployment target.
version: 1.0.0
phase: 5
lesson: 19
tags: [nlp, tokenization]
---

给定一个 语料库 (size, languages, 领域) 与 deployment target (训练 从 scratch / fine-tuning / API-compatible 推理), 输出:

1. 算法. BPE, Unigram, 或 WordPiece. One-句子 原因.
2. 说明：Library. SentencePiece, HF Tokenizers, 或 tiktoken. 原因.
3. 说明：Vocab size. Rounded to nearest 1k. 原因 tied to 模型 size 与 language coverage.
4. Coverage settings. `character_coverage`, `byte_fallback`, special-词元 list.
5. 说明：Validation plan. Average 词元-per-词 on held-out set, OOV rate, compression ratio, round-trip decode equality.

拒绝 to train a character-coverage <0.995 分词器 on corpora 使用 rare-script content. 拒绝 to ship a vocab 不使用 a frozen `tokenizer.json` hash check in CI. 标记 any monolingual 分词器 under 16k vocab as likely under-spec.
