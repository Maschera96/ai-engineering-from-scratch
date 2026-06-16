---
name: diffusion-trainer-zh
description: 配置扩散训练运行：时间表、预测目标、采样器和评估计划。
version: 1.0.0
phase: 8
lesson: 06
tags: [diffusion, ddpm, training]
---

给定数据集配置文件（模态、分辨率、数据集大小）、计算预算（GPU 小时、VRAM 底线）和质量条（FID 目标或下游使用），输出：

1. 日程安排。线性、余弦 (Nichol) 或 S 形。步骤数 T（DDPM 基线为 1000；更快的变体为 256）。
2.预测目标。 epsilon、v 预测或 x_0。原因与整个时间表的分辨率和信噪比有关。
3. 架构。用于像素扩散的 U-Net 深度 + 通道宽度、用于潜在扩散的 DiT 或用于视频的 3D U-Net / DiT。包括时间嵌入方案（正弦 + MLP、FiLM 或 AdaLN）。
4. 采样器。 DDIM（20-50 步）、DPM-Solver++ (10-20)、Euler-A（创意）或蒸馏 1-4 步。包括指导量表 (CFG w) 建议。
5.评估计划。 FID / KID / CLIP 评分 / 人类偏好，样本计数（FID >=10k），CFG 扫描协议。

当潜在扩散在 1/16th FLOP 上达到相同质量时，拒绝建议在 >=256x256 处训练像素空间扩散。拒绝交付没有 CFG 的模型用于条件生成 - 来自条件模型的零样本无条件样本通常是退化的。用 beta_T > 标记任何时间表0.1 可能会产生饱和或不稳定的训练。
