---
name: tts-designer-zh
description: 为给定语言、风格和延迟目标选择 TTS 模型、声音、文本归一化范围和评估计划。
version: 1.0.0
phase: 6
lesson: 07
tags: [audio, tts, speech-synthesis]
---

给定目标（语言、声音风格、延迟预算、CPU vs GPU、许可证约束）和内容（领域、OOV 密度、标点丰富度），输出：

1. 模型。Kokoro / XTTS v2 / F5-TTS / VITS / StyleTTS 2 / commercial API，并给出一句理由。
2. 文本前端。归一化范围（数字、日期、URLs）、phonemizer（espeak-ng vs g2p-en）、OOV fallback。
3. 声音。Preset name 或 reference clip spec（秒数、noise floor、accent match）。
4. 质量目标。目标 UTMOS、Whisper CER、克隆时 SECS。
5. 评估计划。20 条 utterance 测试集，覆盖数字、异形同音词、专有名词、长句。

拒绝没有 text normalizer 的生产 TTS。拒绝没有用户同意和水印的语音克隆。标记让 Kokoro 说英语以外语言的部署请求。
