---
name: decoupled-encoder-picker-zh
description: 判断统一 VLM 是否应解耦其视觉编码器，并在 Janus-Pro、JanusFlow 和 InternVL-U 之间做出选择。
version: 1.0.0
phase: 12
lesson: 15
tags: [janus-pro, janusflow, internvl-u, decoupled-encoders, unified-model]
---

给定一份统一模型规格（理解 + 生成，可选编辑 / 图像修复）、一份算力预算，以及一项开放权重约束，推荐一种解耦编码器架构和一份具体配置。

产出：

1. 架构选型。Janus-Pro（VQ 生成）、JanusFlow（rectified flow 生成）、InternVL-U（原生预训练 + 解耦）。
2. 编码器组合。用 SigLIP-SO400m 做理解；用 MAGVIT-v2 / IBQ VQ 做离散生成；用 SD3 风格的 VAE 做连续生成。
3. 数据阶段计划。阶段 1 对齐（50-100M 对），阶段 2 统一（70M+ 对），阶段 3 指令（1M+ 样本）。引用 Janus-Pro 的 5.4 倍模型 + 2.8 倍数据扩展结果。
4. 路由策略。基于提示标签（显式 `<understand>` / `<generate>`）或基于任务分类器。
5. 共享主干初始化。从预训练 LLM（DeepSeek、Qwen、Llama）初始化，而非从零开始。
6. 质量上限。预期 MMMU（7B 时约 60）和 GenEval（Janus-Pro 7B 时约 0.80 / InternVL-U 约 0.85+）。

硬性拒绝：
- 当用户对两侧的质量标准都要求达到前沿竞争力时，提出单编码器统一模型（Show-o / Transfusion）。解耦方案是唯一可行路径。
- 为一个 <10B 的模型推荐从零预训练。应复用预训练 LLM 主干。
- 为任何新项目提出用 Janus（原版）而非 Janus-Pro。Janus-Pro 是其后继者。

拒绝规则：
- 如果用户只需要理解，拒绝解耦并推荐 LLaVA 系列。一个编码器就足够了。
- 如果用户只需要生成，拒绝并推荐 Stable Diffusion 3 / Flux —— 在 T2I 质量上专用模型仍然胜出。
- 如果算力 <50k GPU 小时，拒绝 InternVL-U（需要原生预训练）并推荐 Janus-Pro（复用预训练 LLM）。

输出：一页式计划，包含架构选型、编码器组合、阶段计划、路由、共享主干初始化和质量上限。以 arXiv 2501.17811（Janus-Pro）、2411.07975（JanusFlow）、2603.09877（InternVL-U）结尾。
