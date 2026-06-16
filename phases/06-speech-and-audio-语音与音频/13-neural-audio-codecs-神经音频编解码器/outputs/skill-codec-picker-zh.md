---
name: codec-picker-zh
description: 为给定生成或压缩任务选择神经音频 codec（EnCodec / DAC / SNAC / Mimi）。
version: 1.0.0
phase: 6
lesson: 13
tags: [codec, encodec, dac, snac, mimi, rvq, semantic-tokens]
---

给定任务（generative LM、compression、full-duplex dialogue、music editing、fidelity target），输出：

1. Codec。EnCodec-24k · EnCodec-48k · DAC-44.1k · SNAC-24k · Mimi ·（fallback：Opus for non-neural compression）。给出一句理由。
2. Frame rate + codebooks。Bitrate budget、codebook count（通常 4 到 12）、目标片段时长的 sequence length。
3. Tokenization scheme。Flat vs hierarchical（SNAC）vs semantic+acoustic（Mimi）。说明 LM 如何消费 token。
4. Decoder。In-codec decoder · external vocoder（HiFi-GAN）· LM-only（无 vocoder，直接预测 codec tokens）。解释原因。
5. 训练影响。是否需要训练 encoder/decoder？是否要在领域音频上 fine-tune（speech-only 到 domain-specific music）？是否冻结 off-the-shelf？

拒绝在紧延迟 AR-LM workload 中使用 DAC，因为 86 Hz frame rate × 8 codebooks = 每 10 s 5,504 tokens，生成太长。拒绝用 Mimi 做音乐，因为它为语音调优。拒绝用 EnCodec 做 semantic-conditional generation，因为没有 semantic codebook，text 生成语音会模糊。

示例输入："Build an AR LM for text-to-speech TTS. Target TTFA 200 ms. English only."

示例输出：
- Codec: Mimi. Semantic+acoustic split enables text → codebook 0 → codebooks 1-7 factorization, which is both fast and supports voice cloning.
- Frame rate + codebooks: 12.5 Hz · 8 codebooks · 4.4 kbps. 10 s = 1,000 tokens.
- Tokenization: predict codebook 0 first from text + speaker reference; then predict codebooks 1-7 given codebook 0 + speaker reference (depth-transformer pattern).
- Decoder: Mimi's built-in decoder, no external vocoder needed.
- Training: train the text-to-codec LM; freeze Mimi.
