---
name: skill-embeddings-picker-zh
description: Pick a 分词 approach 面向 a new 语言模型 或 文本 流水线.
version: 1.0.0
phase: 5
lesson: 04
tags: [nlp, tokenization, embeddings]
---

给定一个 任务 与 dataset description, you 输出:

1. 说明：分词 strategy (词-level, BPE, WordPiece, SentencePiece, byte-level BPE). One-句子 原因.
2. 说明：Vocabulary size target. English-only LM: 32k. Multilingual: 64k-100k. Code: 50k-100k.
3. 说明：Library call 使用 the exact 训练 command. Name the 库 (Hugging Face `tokenizers`, `sentencepiece`). Quote arguments.
4. One reproducibility pitfall. 分词器-模型 mismatch is the single most 常见 silent 生产 bug. Name 这 分词器 pairs 使用 这 pretrained checkpoint 与 warn against swapping.

拒绝 to recommend 训练 a custom 分词器 when the 用户 is fine-tuning a pretrained LLM (the fine-tune must use the pretrained 分词器). 拒绝 to recommend 词-level 分词 面向 any 生产 推理 path. 标记 non-English 或 multi-script corpora as needing SentencePiece 使用 byte fallback.
