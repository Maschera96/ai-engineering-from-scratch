---
name: unified-gen-model-picker-zh
description: 为既需要多模态理解又需要生成、且要求开放权重的产品，在 Show-o / Transfusion / Emu3 / Janus-Pro 系列之间做出选择。
version: 1.0.0
phase: 12
lesson: 14
tags: [show-o, masked-diffusion, unified, t2i, inpainting]
---

给定一个需要统一理解 + 生成（VQA、图像描述、T2I，可选 inpainting）的产品，并带有开放权重约束和延迟预算，选择一个模型系列并输出一份参考配置。

产出：

1. 系列判定。Show-o（掩码离散扩散）、Transfusion / MMDiT（连续扩散）、Emu3 / Chameleon（自回归离散）或 Janus-Pro（解耦编码器）。
2. 推理步数预算。Show-o 为 16 步，Transfusion 为 20 步，Emu3 为 1024+ 步。结合用户的延迟预算论证该选择。
3. inpainting 支持。Show-o 免费支持；Transfusion 需增加一个 mask 通道；Emu3 需要单独微调。向用户标明这一点。
4. tokenizer 选择。对于离散系列，推荐 IBQ / MAGVIT-v2 / SBER；对于连续系列，推荐 SD3 的 VAE。
5. 训练稳定性。双损失（Transfusion）需要调权重；Show-o 的单损失更干净。
6. 用户规模增长时的迁移路径。当质量成为瓶颈时，从 Show-o 迁移到 Transfusion。

强制拒绝：
- 在每张图像推理延迟 <10s 时提议 Emu3 / Chameleon。对约 1024 个 token 做自回归太慢了。
- 声称 Show-o 在前沿图像质量上能匹敌 Transfusion。它做不到。tokenizer 是上限。
- 为需要 VQA 的产品推荐 Stable Diffusion。SD 无法对图像进行推理。

拒绝规则：
- 如果用户希望每张图像生成 <2s，拒绝 Show-o，并推荐 Stable Diffusion + 单独的 VLM 来做理解。接受多模型带来的复杂度。
- 如果用户希望在开放权重下获得「业界顶级质量」，拒绝 Show-o / Emu3，并推荐 Transfusion 系列（MMDiT）或 JanusFlow。
- 如果用户无法在某个 tokenizer 上做出承诺（担心授权、质量上限），拒绝纯离散系列，并推荐 Transfusion。

输出：一页式选择结论，包含系列判定、步数预算、inpainting 支持、tokenizer 推荐、稳定性方案和迁移路径。结尾附上 arXiv 2408.12528（Show-o）、2408.11039（Transfusion）、2501.17811（Janus-Pro）。
