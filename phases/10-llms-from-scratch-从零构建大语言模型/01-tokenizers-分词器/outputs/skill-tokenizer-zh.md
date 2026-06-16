---
name: skill-tokenizer-zh
description: Choosing and building 分词器s for LLM projects
version: 1.0.0
phase: 10
lesson: 1
tags: [tokenizer, bpe, wordpiece, sentencepiece, llm, nlp]
---

# 分词器 Selection and Implementation

当starting an LLM project, apply this decision framework for 分词器 selection.

## When to use each 分词器

**Byte-level BPE (tiktoken):** You are building on or 微调 GPT-family 模型. You need guaranteed handling of any 输入 byte 序列. You want no unknown 词元.

**WordPiece (Hugging Face):** You are working with BERT-family 模型 for 分类, NER, or 嵌入 tasks. You need the "##" continuation prefix for downstream tasks that rely on word boundary signals.

**SentencePiece (BPE or Unigram):** You are 训练 from scratch. You need language-agnostic tokenization. Your 数据 includes CJK languages, Thai, or other scripts without whitespace word boundaries. LLaMA, T5, and most multilingual 模型 use this.

## 词表 size guidelines

- 32K 词元: good default for single-语言模型s, keeps 嵌入 层 small
- 50K-64K 词元: better for multilingual or code-heavy 模型
- 100K+ 词元: only when you have massive 训练 数据 and want short sequences

Larger 词表 means shorter sequences (cheaper 推理) but more 参数 in the 嵌入 matrix. For a 100K 词表 with 4096-dimensional 嵌入s, the 嵌入 层 alone is 400M 参数.

## Pre-tokenization rules that matter

1. Split on whitespace before BPE to prevent cross-word merges
2. Separate digits individually if you want the 模型 to learn arithmetic
3. Normalize Unicode (NFC) before tokenization for consistent behavior
4. Add special 词元 for your use case: `<pad>`, `<eos>`, `<bos>`, `<unk>`, and any task-specific markers

## Red flags in 分词器 behavior

- Fertility above 2.0 for your 目标 language: the 模型 wastes 上下文 window
- Common 领域 words splitting into 3+ 词元: retrain with 领域 数据
- Inconsistent tokenization of numbers: check digit-splitting rules
- Large 词表 with many single-use 词元: reduce 词表 size

## Building a custom 分词器 - checklist

1. Collect representative 训练 数据 (at least 1GB of 文本 in 目标 领域)
2. Choose algorithm: BPE for general use, Unigram for multilingual
3. Set 词表 size based on guidelines above
4. Configure pre-tokenization: whitespace splitting, digit handling, punctuation
5. Add special 词元
6. 训练 using Hugging Face 分词器s library (Rust backend, fast)
7. 验证: check fertility on held-out 文本 across all 目标 languages
8. Test 边 cases: empty string, very long 输入, binary 数据, emoji, RTL 文本
9. Save and version the 分词器 alongside 模型 checkpoints
