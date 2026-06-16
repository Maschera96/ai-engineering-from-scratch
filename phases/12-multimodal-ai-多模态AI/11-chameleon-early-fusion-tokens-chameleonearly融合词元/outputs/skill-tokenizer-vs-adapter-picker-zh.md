---
name: tokenizer-vs-adapter-picker-zh
description: 为 VLM 项目在 Chameleon 风格的early融合（共享词表分词器）与 LLaVA 风格的late融合（在冻结 LLM 上加适配器）之间做选择。
version: 1.0.0
phase: 12
lesson: 11
tags: [chameleon, early-fusion, vq-vae, late-fusion, adapter]
---

给定一份产品规格说明（仅理解，或理解+生成）、目标图像质量（社交帖文 / 杂志 / 印刷 / 广播）以及成本预算（训练 + 推理），推荐 Chameleon 系列或 LLaVA 系列，并给出具体的架构提纲。

产出：

1. 结论。early融合（Chameleon / Emu3 / AnyGPT）还是late融合（LLaVA / BLIP-2 / Qwen-VL）系列。
2. 分词器选择（针对early融合的结论）。VQ-VAE（Chameleon）、MAGVIT-v2、IBQ 或 SBER-MoVQGAN；引用以 PSNR 表示的预期重建上限。
3. 训练稳定性方案。大规模early融合中的 QK-Norm、dropout 放置位置、LayerNorm 排序。
4. 成本估算。训练 GPU 小时数，以及每张图像的推理延迟，与late融合方案对比。
5. 生成质量上限。用户可预期的 PSNR / FID 区间；用离散词元能否达到产品的质量门槛，还是需要连续（Transfusion 风格）生成。
6. 迁移路径。如果用户规模增长且late融合开始成为瓶颈（他们需要图像输出），迁移会是什么样子。

硬性拒绝：
- 为仅理解类产品推荐 Chameleon 风格。对于纯理解，late融合更简单、更便宜，且上限更高。
- 为生产级图像生成提出 K<4096 的 VQ-VAE。码本太小，伪影可见。
- 宣称early融合推理是免费的。VQ 解码器每生成一张图像会增加 50-200ms，往往超过 LLM 的输出时间。

拒绝规则：
- 如果用户想要前沿质量的图像生成（FID < 15，可印刷级），拒绝离散词元，并指向 Transfusion / Stable Diffusion 3 / MMDiT（课程 12.13）。
- 如果产品从不需要图像输出，拒绝early融合——其复杂性没有必要。
- 如果用户想接入现有的 Llama / Qwen LLM 权重，拒绝early融合——它需要从头预训练一个新模型。

输出：一页计划，包含结论、分词器选择、稳定性检查清单、成本估算、质量上限、迁移路径。结尾附上 arXiv 2405.09818（Chameleon）和 2408.11039（Transfusion）作为对比阅读。
