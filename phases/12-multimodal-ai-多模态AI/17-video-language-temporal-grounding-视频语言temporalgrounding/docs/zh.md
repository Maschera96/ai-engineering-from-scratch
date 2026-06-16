# 视频语言模型：时序 token 与 grounding

> 视频不是一叠照片。一段 5 秒的片段具有因果顺序、动作动词和事件时序，而图像模型无法表示这些。Video-LLaMA（Zhang 等，2023 年 6 月）发布了首个带音视频 grounding 的开源视频 LLM。VideoChat 和 Video-LLaVA 扩大了这一范式。到 2025 年，Qwen2.5-VL 的 TMRoPE 缩小了与前沿专有模型的差距。每个系统解决时序 token 的方式各不相同——Video-LLaMA 用每片段一个 Q-former，Video-LLaVA 用每帧 concat-pool，Qwen2.5-VL 用每 token 一个 TMRoPE。本课解读这些范式，构建一个均匀采样 vs 动态采样的帧采样器，并在时序 grounding 任务上做评估。

**类型：** 构建
**语言：** Python（标准库，帧采样器 + 时序 grounding 评估器）
**前置要求：** 阶段 12 · 08（LLaVA-OneVision）
**时长：** ~180 分钟

## 学习目标

- 解释为什么时序位置编码会独立于视觉编码器而改变视频 VLM 的性能。
- 在每秒 token 数 vs grounding 准确率上对比均匀采样、动态 FPS 和事件驱动的帧采样。
- 描述每片段 Q-former（Video-LLaMA）vs 每帧 pooled（Video-LLaVA）vs 每 token M-RoPE（Qwen2.5-VL）的设计。
- 说出四个视频基准的名称：VideoMME、TempCompass、EgoSchema、Video-MMMU。

## 问题

一段 1 分钟、30 FPS 的视频是 1800 帧。每帧 196 个视觉 token（224 分辨率下的 ViT-B），那就是 35.2 万个 token——比任何 2024 时代的 LLM 上下文都大。

存在三种缩减策略：

1. 对帧做子采样（根据内容取 1-8 FPS）。
2. 对每帧的 patch token 做激进 pooling（3x3 或 4x4 双线性 pool）。
3. 通过一个 Q-former 压缩，它接收 16 帧片段并输出 64 个 token。

每种权衡都不同。子采样会损失时序细节。Pooling 会损失空间细节。Q-former 两者都损失一点，但节省 token。

时序位置编码是另一个维度：模型如何知道第 5 帧在第 6 帧之前？选项包括简单的一维时序 RoPE（Video-LLaMA）、学习得到的时序嵌入（Video-LLaVA），以及 TMRoPE（Qwen2.5-VL，完整 3D）。

## 概念

### Video-LLaMA：每片段 Q-former + 音频分支

Video-LLaMA（2023）是首个开源视频 LLM。架构：

- 2 FPS 下的 16 帧片段（即 8 秒）。
- 每帧 ViT 特征 -> Video Q-former，它在全部 16 帧上做 cross-attend -> 32 个学习得到的查询 -> LLM。
- 并行音频分支：波形 -> ImageBind 音频编码器 -> Audio Q-former -> 32 个查询 -> LLM。

优势：音视频联合推理。劣势：固定片段长度，无法做任意时间 grounding。

### VideoChat 和 Video-LLaVA

VideoChat 保留了 Video-LLaMA 的思路，但去掉了音频并做了简化。Video-LLaVA（Lin 等，2023）在图像和视频帧上训练单个视觉编码器（"投影前先对齐"），给出统一的表示。两者都是冻结的 CLIP 编码器 + MLP + LLM。

二者都无法处理长视频。两者都是 8-16 帧的系统。

### Qwen2.5-VL 和 TMRoPE

Qwen2.5-VL 引入了 TMRoPE——Temporal-Modality Rotary Position Embedding（时序-模态旋转位置嵌入）。每个 patch token 携带一个 (t, h, w) 位置，其中 t 是实际时间戳（而非帧索引）。

与简单时序嵌入的关键区别：

- 绝对时间，而非索引。模型看到的是"在 4.2 秒处"，而不是"在第 15 帧"。
- 每 token 旋转，而非每片段。每个视觉 token 按其时间戳独立旋转。
- 兼容动态 FPS。如果你这里以 2 FPS、那里以 4 FPS 采样，TMRoPE 原生处理这种不均匀间隔。

TMRoPE 支持"猫在第几秒跳起来？"这类查询。模型可以输出"在 4.2 秒处"。Video-LLaMA 只能说"在片段的早段"。

### 帧采样策略

均匀采样：在时长上均匀采样 N 帧。简单，但会丢失运动峰值。

动态 FPS：根据运动强度自适应采样。光流或帧间差分挑出高运动段做更密集的采样。Qwen2.5-VL 就以此训练。

事件驱动：运行一个轻量检测器，在有动作发生的地方采样更多。VideoAgent 使用此方法。

关键帧 + 上下文：在镜头边界处加上几个相邻帧采样。用于影视类内容。

### 每帧 pooling

在 1 FPS、每帧 576 token 时，一段 5 分钟的片段是 172,800 个 token。Qwen2.5-VL-72B 的 128k 上下文能做到，但代价高昂。

3x3 双线性 pool 将其缩减到每帧 64 个 token -> 5 分钟 19,200 个 token。对大多数任务而言是甜点位。

在空间细节不那么重要的智能体工作流中，做更激进的 pooling（6x6 -> 每帧 16 个 token）。

### 四个视频基准

- VideoMME：全面的视频理解，短 + 中 + 长。
- TempCompass：细粒度时序推理，"之前" / "之后" 类问题。
- EgoSchema：长程第一人称视频。
- Video-MMMU：多模态多学科视频问题。

一次完整的视频 VLM 评估会覆盖这全部四个。它们考验不同的维度——TempCompass 完全关乎排序，EgoSchema 关乎 3 分钟以上的推理，VideoMME 跨越各种时长。

### Grounding 输出格式

时序 grounding 的输出格式：

- 自由文本："猫在 4 秒左右跳起来。"易于解析但不精确。
- 结构化 JSON：`{"event": "jump", "start": 4.1, "end": 4.3}`。Qwen2.5-VL 以此训练。
- 基于 token：与答案交错的特殊 `<time>4.1</time>` token。Qwen2.5-VL 的内部格式。

基于 token 的格式对下游使用最精确。Qwen2.5-VL 的 JSON 输出格式可直接解析。

### 2026 最佳实践

2026 年的视频 VLM：

- 编码器：带 M-RoPE 或 TMRoPE 的 SigLIP 2（Qwen2.5-VL）。
- 帧采样：动态 FPS（根据运动取 1-4）并设最大帧数上限。
- 每帧 pooling：3x3 双线性。
- 输出：带 time + event 字段的结构化 JSON。
- 基准：通用用 VideoMME + TempCompass；长程用 EgoSchema。

## 动手用

`code/main.py` 包含：

- 均匀采样和动态 FPS 帧采样器。
- 一个玩具级时序 grounding 评估器：给定时间 T 处的"真值"事件和模型输出，按容差为准确率打分。
- Video-LLaMA（16 帧，Q-former）、Video-LLaVA（8 帧，MLP）、Qwen2.5-VL（动态 FPS + TMRoPE）之间的对比。

## 交付

本课产出 `outputs/skill-video-vlm-frame-planner.md`。给定一个视频任务（监控、动作识别、时序 grounding、摘要），它会挑选帧采样器、pooling 因子、输出格式和预期准确率档位。

## 练习

1. 对于一段 3 分钟的烹饪演示，选择均匀采样还是动态 FPS。用 token 数加以论证。

2. TMRoPE 具体增加了哪些简单时序嵌入表无法做到的能力？

3. 为时序 grounding 写一个 VLM 可以学会输出的 JSON 模式。包含错误情形。

4. 阅读 Video-LLaVA 第 3 节"投影前先对齐（Alignment Before Projection）"。为什么这比训练分离的图像和视频编码器更好？

5. 给定 VideoMME 排行榜，截至 2026 年，顶级开源模型与顶级专有模型之间的差距是多少？其中有多少差距可归因于时序编码 vs 基础 LLM 规模？

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|-----------------|------------------------|
| 时序 grounding | "时间定位的答案" | VLM 为事件发生时刻输出一个具体的时间戳范围 |
| TMRoPE | "Time-Multimodal RoPE" | 带绝对时间戳的 3D 旋转位置，Qwen2.5-VL 使用 |
| 动态 FPS | "运动感知采样" | 在高运动段采样更多帧，在静态段采样更少 |
| 帧 pooling | "每帧空间压缩" | 在送入 LLM 前用双线性插值缩减每帧的 patch 数 |
| Video Q-former | "片段压缩器" | 将 N 帧映射到 K 个学习查询的 cross-attention 瓶颈 |
| VideoMME | "视频基准" | 全面的短/中/长视频基准，2500+ 样本 |

## 延伸阅读

- [Zhang et al. — Video-LLaMA (arXiv:2306.02858)](https://arxiv.org/abs/2306.02858)
- [Li et al. — VideoChat (arXiv:2305.06355)](https://arxiv.org/abs/2305.06355)
- [Lin et al. — Video-LLaVA (arXiv:2311.10122)](https://arxiv.org/abs/2311.10122)
- [Qwen Team — Qwen2.5-VL (arXiv:2502.13923)](https://arxiv.org/abs/2502.13923)
- [Lin et al. — VILA-1.5 (arXiv:2312.07533)](https://arxiv.org/abs/2312.07533)
