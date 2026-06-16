---
name: audio-brief-zh
description: 将音频简介转换为跨 TTS、音乐和 SFX 的模型 + 提示 + 评估计划。
version: 1.0.0
phase: 8
lesson: 11
tags: [audio, tts, music, sfx, codec]
---

给定音频简介（任务：TTS/音乐/SFX/语音克隆、持续时间、风格、语音或流派、许可证限制、实时或离线、质量栏），输出：

1.模型+托管。 ElevenLabs V3、OpenAI TTS、XTTS v2、Suno v4、Udio、Stable Audio 2.5、MusicGen 3.3B、AudioCraft 2 或 GPT-4o 实时。一句话理由。
2.提示格式。 TTS：文本+语音提示（3-10秒样本或语音ID）+情感/节奏标签。音乐：流派+乐器+情绪+BPM+结构标记。 SFX：象声词+来源+持续时间提示。
3.编解码器+生成器+声码器链。指定特定编解码器（编码解码器 32 kHz、DAC 44 kHz、自定义）和生成器选择（词元 AR 与流匹配）。
4.种子+再现性。种子 pin、版本 pin、提示哈希。
5. 评估。 TTS 的 MOS（平均意见分数）或 A/B、音乐的 CLAP 分数、TTS 转录的 CER、SFX 的用户听力测试。
6.护栏。语音克隆同意 + 水印（PerTh / SynthID-音频）、音乐输出版权扫描、训练数据策略检查。

未经所有者验证同意，拒绝克隆任何声音（卡带时代的“3秒提示”并不是同意）。拒绝运送带有未经许可的参考材料的音乐。标记任何实时目标<不使用流词元 AR 模型的 200 毫秒 - 基于扩散的音频无法满足 2026 年低于 300 毫秒的 TTFB。
