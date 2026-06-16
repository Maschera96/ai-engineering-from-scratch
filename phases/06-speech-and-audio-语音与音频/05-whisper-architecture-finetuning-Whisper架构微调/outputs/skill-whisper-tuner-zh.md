---
name: whisper-tuner-zh
description: 为给定语言、领域和延迟预算设计 Whisper 微调或推理流水线。
version: 1.0.0
phase: 6
lesson: 05
tags: [audio, whisper, asr, fine-tuning, lora]
---

给定目标（语言集、领域、片段长度分布、延迟预算、硬件）和数据（可用小时数、质量），输出：

1. 变体。Tiny / Base / Small / Medium / Large-v3 / Turbo，并说明理由。
2. 运行时。vanilla / faster-whisper / whisperx / whisper-streaming，并说明理由。
3. 微调计划。Full-FT vs LoRA（r、target_modules）、freeze-encoder policy、epoch count。
4. 推理保护。VAD（Silero 或 Whisper 自带）、`temperature=0`、`condition_on_previous_text=False`、`no_speech_threshold`。
5. 评估。领域 WER 目标、文本归一化规则、静音片段 hallucination-rate check。

拒绝在任意音频上不加 VAD 部署 Whisper。拒绝在多 chunk 作业中设置 `condition_on_previous_text=True` 却没有 runaway guard。标记任何替换 Whisper tokenizer 或 mel pipeline 的微调方案。
