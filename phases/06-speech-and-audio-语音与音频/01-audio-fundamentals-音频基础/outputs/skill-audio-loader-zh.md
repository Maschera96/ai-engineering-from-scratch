---
name: audio-loader-zh
description: 根据目标模型期望校验原始音频文件，并安全重采样。
version: 1.0.0
phase: 6
lesson: 01
tags: [audio, speech, preprocessing]
---

给定音频文件（路径、通道、采样率、位深、codec）和目标模型（ASR / TTS / classifier，含所需采样率与通道数），输出：

1. 不匹配项。列出文件与目标不一致的维度（sr、channels、duration floor、clipping check）。
2. 重采样计划。源 sr、目标 sr、重采样库（`torchaudio.transforms.Resample` 或 `librosa.resample`）、抗混叠滤波器类型。
3. 通道计划。Mono fold 策略（mean vs left-only），或模型支持多通道时直接透传。
4. 归一化。Peak vs RMS normalization、dBFS target、clipping guard。
5. 验证片段。Python 加载文件、执行变换，并断言最终数组匹配 `(target_sr, dtype, channel_count, range)`。

拒绝在没有抗混叠滤波器时降采样。拒绝在没有重建滤波器时上采样超过 2x。标记任何峰值超过 ±0.999 或 DC offset 高于 ±0.01 的输入文件。
