---
name: generative-model-chooser-zh
description: 为给定的任务和预算选择生成模型系列、主干和托管替代方案。
version: 1.0.0
phase: 8
lesson: 01
tags: [generative, taxonomy]
---

给定任务描述（模态、域、延迟预算、计算预算、调节信号），输出：

1. 家庭。显式易处理、显式近似（VAE / 扩散）、隐式（GAN）、分数/流匹配或词元 AR。一句话原因与模态+延迟相关。
2.主干+开放参考。用户现在可以微调一种预训练的开放权重模型（例如 Stable Diffusion 3、Flux.1-dev、AudioCraft 2、StyleGAN3、3D Gaussian Splatting）。
3. 托管替代方案。三个生产 API 按质量/成本/延迟权衡排名（fal.ai、Replicate、Stability、Runway、Veo、Kling、ElevenLabs 等）。
4.失效模式。所选系列的已知病理（模式崩溃、曝光偏差、采样器漂移、分词器伪影、CLIP 得分游戏）。
5. 预算。单个 A100 上的粗略训练时间、每个样本的推理成本、VRAM 底数。

当任务需要似然评分时，拒绝推荐 GAN。拒绝推荐使用像素自回归来进行高分辨率实时使用。如果列出的开放骨干网已经覆盖了该域，则将任何建议标记为“从头开始训练”。
