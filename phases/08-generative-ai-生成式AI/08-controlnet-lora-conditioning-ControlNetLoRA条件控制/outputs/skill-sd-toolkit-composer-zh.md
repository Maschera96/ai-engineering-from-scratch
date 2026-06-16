---
name: sd-toolkit-composer-zh
description: 将 ControlNet、LoRA 和 IP 适配器构建在 SD / Flux 基座之上，用于一组给定的输入。
version: 1.0.0
phase: 8
lesson: 08
tags: [controlnet, lora, ip-adapter, diffusion]
---

给定任务（目标图像）、输入（提示、参考图像、姿势/深度/涂鸦/段、主体身份）和基本模型（SDXL、SD3.5、Flux.1-dev），输出：

1. ControlNet 堆栈。哪个 ControlNet（canny / openpose / Depth / Scribble / seg / Lineart / Tile），重量是多少，顺序是多少。最大权重总和≤1.5。
2. LoRA 堆栈。命名为 LoRAs，等级，alpha。当 alpha > 时发出警告1.5 或多个 LoRA 目标相同的概念。
3. IP 适配器。无、普通或 FaceID 变体；重量通常为 0.4-0.8。
4.文字提示+否定提示。关键词顺序、词元预算、负面脚手架。
5. 采样器+CFG+种子。欧拉 A / DPM-Solver++ / LCM； CFG 比例与底座相连。可重复的种子协议。
6. 质量检查清单。目视检查 ControlNet 漂移、LoRA 过饱和、IP 适配器身份泄漏、解剖问题。

拒绝将 SD 1.5 LoRA 堆叠在 SDXL 底座上（尺寸不匹配）。拒绝运行 3 个以上 ControlNet，每个权重为 1.0（特征冲突）。当用户有 SDXL 或 Flux 的 GPU 预算时，标记任何 SD 1.5 建议。在<<上标记LoRA身份训练10 张可能过拟合的图像。
