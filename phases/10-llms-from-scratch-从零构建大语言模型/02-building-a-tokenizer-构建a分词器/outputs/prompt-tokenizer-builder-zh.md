---
name: prompt-tokenizer-builder-zh
description: 构建and 调试 production-quality 分词器s for LLM projects
version: 1.0.0
phase: 10
lesson: 2
tags: [tokenizer, bpe, byte-level, special-tokens, chat-template, multilingual]
---

# 生产 分词器 Builder

当building or 调试 a 分词器 for an LLM project, follow this framework.

## 流水线 Checklist

每个生产 分词器 needs these five stages. If one is missing, you will hit 边 cases in 生产.

1. **Normalize** -- Apply NFKC Unicode 归一化. This collapses ligatures ("fi" -> "fi"), normalizes fullwidth characters, and standardizes whitespace. Skip this and the same word gets different 词元 IDs depending on how it was typed.

2. **Pre-Tokenize** -- Split 文本 into chunks before BPE. Use GPT-2's regex pattern for English-centric 模型. Use SentencePiece's raw-byte approach for multilingual 模型. The choice determines whether BPE can merge across word boundaries.

3. **BPE Merge** -- Apply the learned merge table to byte sequences within each 分块. The merge table IS the 分词器's learned knowledge. Everything else is plumbing.

4. **Special 词元 Injection** -- Match special 词元 exactly before BPE runs. [BOS], [EOS], [PAD], chat template markers get fixed IDs. They never participate in merges.

5. **ID Mapping** -- Convert 词元 strings to integers. The 模型 sees integers only.

## 调试 分词器 Issues

**Symptom: 模型 produces garbage on chat 输入**
- Check the chat template. Every 模型 has a different format. Llama 3 uses `<|start_header_id|>` markers. ChatGPT uses `<|im_start|>` markers. A wrong template puts 输入 outside the 训练 分布.

**Symptom: non-English 文本 uses too many 词元**
- Check fertility (词元 per word). Above 2.0 means the 分词器 wastes 上下文 window on that language. Solutions: retrain with more multilingual 数据, increase 词表 size, or use SentencePiece with Unigram.

**Symptom: numbers and arithmetic fail**
- Check how digits are tokenized. "1234" as one 词元 means the 模型 cannot do digit-level operations. Split digits individually during pre-tokenization.

**Symptom: code 词元 are inefficient**
- Check how indentation is handled. GPT-2's 分词器 wastes 词元 on spaces. Codex and StarCoder use special indentation 词元 (4 spaces = 1 词元).

## 词表 Size Decision

- 32K 词元: single-language, small 模型, limited 计算. 嵌入 层 is 32K * d_model 参数.
- 50K-64K: multilingual or code-heavy. Good balance for most projects.
- 100K+ (GPT-4, Llama 3): only with massive 训练 数据. Shorter sequences but 100K * d_model 嵌入 参数.

For a 4096-dimensional 模型: 32K vocab = 131M 嵌入 params. 128K vocab = 524M 嵌入 params. That is 400M 参数 just in the 嵌入 层.

## Speed Requirements

- 训练 数据 tokenization: use Rust-backed libraries (tiktoken, HuggingFace 分词器s). Pure Python is 10-100x slower.
- 推理 tokenization: 延迟 matters less (single 序列), but still use compiled implementations.
- 基准: tokenize 1GB of 文本 and measure wall clock time. If it takes more than 60 seconds, switch to a Rust backend.

## Chat Template 验证

Before deploying any chat 模型, verify the template:

1. Encode a known conversation with the 分词器
2. Decode it back to 文本
3. 比较character-by-character with the expected format from the 模型's documentation
4. Pay 注意力 to: newlines after header 词元, spaces before content, end-of-turn markers
5. Test 边 cases: empty 系统 消息, very long 用户 消息, multiple 助手 turns

Getting the chat template wrong is the most common 来源 of degraded chat 模型 performance.
