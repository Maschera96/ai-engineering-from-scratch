---
name: feature-extractor-zh
description: 选择特征类型、mel 数、帧长、hop 和归一化方式，以匹配下游音频模型。
version: 1.0.0
phase: 6
lesson: 02
tags: [audio, features, spectrogram, mel]
---

给定目标模型（ASR / TTS / classifier / speaker / music）和输入音频（采样率、领域），输出：

1. 特征类型。Log-mel、mel、MFCC、raw waveform 或 discrete codec（EnCodec、SoundStream），并给出一句理由。
2. Mel 数和频率范围。`n_mels`、`fmin`、`fmax`，理由要绑定领域（语音 vs 音乐）和模型目标。
3. 帧与 hop。`frame_len`、`hop_len`、窗口类型，理由要绑定所需时间分辨率。
4. 归一化。逐 utterance mean/var、全局统计，或固定 reference 的 dB，说明在特征化前还是后。
5. 验证片段。Python 打印 1 秒参考片段的形状、min/max、mean/std，并断言匹配训练配置。

拒绝交付 frame/hop/mel count 偏离目标模型公开训练配置的特征流水线。标记任何给 Whisper 或 Parakeet 使用 MFCC 的方案为错误，这些模型消费 log-mel。标记没有归一化断言的特征提取器。
