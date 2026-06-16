---
name: asr-picker-zh
description: 为给定部署目标选择 ASR 模型、解码策略、chunking 和 LM fusion。
version: 1.0.0
phase: 6
lesson: 04
tags: [audio, asr, speech-recognition]
---

给定部署目标（语言列表、领域、延迟预算、硬件、离线 / 流式、片段时长），输出：

1. 模型。Whisper-large-v3-turbo / Parakeet-TDT / Canary-Flash / wav2vec 2.0 / Moonshine，并用一句话说明原因。
2. 解码。Greedy / beam width / temperature fallback / LM fusion weight，理由绑定质量预算。
3. Chunking 和 VAD。Chunk 长度、stride、是否用 Silero-VAD 或 Whisper 自带方式 gate。
4. 语言策略。强制语言 vs auto-LID；如何处理跨语言帧。
5. 评估计划。领域测试集 WER、coverage-per-speaker、静音片段 hallucination rate。

拒绝没有 VAD gating 的长音频 Whisper 部署，因为静音上容易 hallucination。拒绝在没有文本归一化（lower、punct strip）的情况下报告 WER。标记没有 LM 的 beam-width > 16，原始 blank beam 不会有帮助。
