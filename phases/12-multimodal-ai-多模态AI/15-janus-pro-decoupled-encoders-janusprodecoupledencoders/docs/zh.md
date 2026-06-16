# Janus-Pro：用于统一多模态模型的解耦编码器

> 统一多模态模型存在一种无法回避的张力。理解任务需要语义特征——SigLIP 或 DINOv2 输出富含概念级信息的向量。生成任务需要利于重建的编码——能够重新组合成清晰像素的 VQ token。这两个目标在单一编码器中无法兼容。Janus（DeepSeek，2024 年 10 月）与 Janus-Pro（DeepSeek，2025 年 1 月）认为解决之道是不再勉强：解耦这两个编码器。在任务之间共享 transformer 主体，但让理解任务经过 SigLIP 路由、生成任务经过 VQ 分词器路由。在 7B 规模下，Janus-Pro 在 GenEval 上击败 DALL-E 3，同时在 MMMU 上与 LLaVA 持平。本课讲解为何在单编码器失败之处，两个编码器却能奏效。

**类型：** 构建
**语言：** Python（标准库，双编码器路由 + 共享主体信号）
**前置：** Phase 12 · 13（Transfusion）、Phase 12 · 14（Show-o）
**时长：** 约 120 分钟

## 学习目标

- 解释为何单一共享编码器会牺牲理解或生成中某一方的质量。
- 描述 Janus-Pro 的路由方式：理解任务在输入侧使用 SigLIP 特征，生成任务在输入和输出两侧都使用 VQ token。
- 追溯使 Janus-Pro 成功（而 Janus 未能成功）的数据混合规模扩展。
- 比较解耦式（Janus-Pro）、耦合连续式（Transfusion）和耦合离散式（Show-o）架构。

## 问题

统一模型在理解和生成任务之间共享一个 transformer 主体。此前的尝试（Chameleon、Show-o、Transfusion）都对两个方向使用同一个视觉分词器。这个分词器是一种折中：

- 为重建（生成）优化：VQ-VAE 捕捉细粒度的像素细节，但产生的 token 语义连贯性较弱。
- 为语义（理解）优化：SigLIP 嵌入会把"猫"的图像聚集在"猫"的 token 附近，但不允许良好的重建。

Show-o 和 Transfusion 为此在某一个方向上付出了明显可见的质量代价。Janus-Pro 提出疑问：既然任务的需求不同，为何还要求只用一个分词器？

## 概念

### 解耦视觉编码

Janus-Pro 的架构将两个编码器分开：

- 理解路径。输入图像 → SigLIP-SO400m → 2 层 MLP → transformer 主体。
- 生成路径。输入图像（若以已有图像为条件）→ VQ 分词器 → token ID → transformer 主体。
- 输出生成。transformer 预测的图像 token → VQ 解码器 → 像素。

transformer 主体是共享的。主体上游和下游的所有部分都是任务专属的。

输入通过提示格式来消歧：`<understand>` 标签经由 SigLIP 路由；`<generate>` 经由 VQ 路由。或者路由由任务隐式决定。

### 为何这样有效

理解任务的损失使用 SigLIP 特征，而 CLIP 式预训练已将其调优以适配语义相似度。模型的感知基准相比 Show-o / Transfusion 有所提升，因为输入特征更契合该任务。

生成任务的损失使用 VQ token，而分词器已将其调优以适配重建。图像质量相比 Show-o 有所提升，因为 VQ 编码能够干净地组合回像素。

共享的 transformer 主体看到两种输入分布（SigLIP 和 VQ），并学会同时处理两者。其主张是：只要有足够的数据 + 足够的参数，主体便能吸收这种切换。

### 数据规模扩展——Janus 对比 Janus-Pro

Janus（最初版，arXiv 2410.13848）引入了解耦，但规模较小（1.3B 参数，数据有限）。Janus-Pro（arXiv 2501.17811）进行了规模扩展：

- 7B 参数（对比 1.3B）。
- 阶段 1（对齐）使用 90M 图文对，从 72M 提升而来。
- 阶段 2（统一）使用 72M，从 26M 提升而来。
- 为阶段 3 新增了 200k 图像生成指令样本。

结果是：Janus-Pro-7B 在 MMMU 上与 LLaVA 持平（60.3 对比约 58），在 GenEval 上击败 DALL-E 3（0.80 对比 0.67）。一个开放模型，在统一光谱的两端都具备竞争力。

### JanusFlow——修正流变体

JanusFlow（arXiv 2411.07975）将 VQ 生成路径替换为修正流生成路径（连续）。这样划分就变成了 SigLIP 用于理解 + 修正流用于生成。质量上限进一步抬升。架构仍然是解耦编码器、共享主体。

### 共享主体的职责

transformer 主体处理一个统一序列，但有两种输入分布。它的职责是：

- 对理解任务：消费 SigLIP 特征 + 文本 token → 自回归地输出文本。
- 对生成任务：消费文本 token +（可选的图像 VQ token）→ 自回归地输出图像 VQ token。

主体每个块中都没有模态专属的权重。它就是你预期会在 Qwen 或 Llama 内部看到的那种文本式 transformer，再加上两个输入适配器。

有趣的是，这意味着 Janus-Pro 的主体可以从一个预训练 LLM 初始化。Janus-Pro 确实从 DeepSeek-MoE-7B 初始化。这个选择很重要：LLM 贡献了纯从零开始的统一模型难以企及的推理能力。

### 与 InternVL-U 的比较

InternVL-U（第 12.10 课）是 2026 年的后续工作。它结合了：

- 原生多模态预训练（InternVL3 骨干）。
- 解耦编码器路由（SigLIP 输入，VQ + 扩散头输出）。
- 统一的理解 + 生成 + 编辑。

InternVL-U 将 Janus-Pro 的架构选择纳入一个更大的框架。解耦编码器的思路如今已成为大规模统一模型的默认方案。

### 局限

解耦编码器带来架构复杂度。要训练两个分词器、维护两条输入路径、应对两套失败模式。对于不需要生成的产品，Janus-Pro 是过度设计——选一个 LLaVA 系列的理解模型即可。

对于不需要理解的产品，Janus-Pro 是大材小用——选一个 Stable Diffusion 3 / Flux 模型即可。

对于两者都需要的产品，Janus-Pro 如今是参考性的开放架构。

## 动手用

`code/main.py` 模拟 Janus-Pro 的路由：

- 两个模拟编码器：SigLIP 式（产生 256 维语义向量）和 VQ 式（产生整数编码）。
- 一个提示路由器，根据任务标签选择编码器。
- 一个共享主体（替身），无论 token 序列由哪个编码器产生，都对其进行处理。
- 一个从阶段 1（对齐）到阶段 3（指令微调）的加权采样切换调度。

为 3 个示例打印路由路径：图像问答、T2I、图像编辑。

## 交付

本课产出 `outputs/skill-decoupled-encoder-picker.md`。给定一个想要在接近前沿质量下实现统一生成 + 理解的产品，它会在 Janus-Pro、JanusFlow 或 InternVL-U 之间做出选择，并给出具体的数据规模建议。

## 练习

1. Janus-Pro-7B 在 GenEval 上击败 DALL-E 3。解释为何一个 7B 开放模型能在生成上匹敌前沿专有模型，却无法在理解上做到。

2. 实现一个路由函数：给定提示文本，将其分类为 `understand` 或 `generate`。你如何处理像"描述然后画出草图"这样的模糊提示？

3. JanusFlow 用修正流替换了 VQ 路径。transformer 主体现在输出什么，损失又发生了什么变化？

4. 提出 Janus-Pro 架构可以再用一个解耦编码器处理的第四种任务。例如：图像分割（DINO 式）、深度估计（MiDaS 式）。

5. 阅读 Janus-Pro 第 4.2 节关于数据规模扩展的内容。相比 Janus，哪个数据阶段对 T2I 质量提升贡献最大？

## 关键术语

| 术语 | 人们的说法 | 实际含义 |
|------|-----------------|------------------------|
| 解耦编码 | "两个视觉编码器" | 每个方向使用各自的分词器或编码器：理解用语义型，生成用重建型 |
| 共享主体 | "一个 transformer" | 单个 transformer 处理任一编码器的输出；没有模态专属权重 |
| SigLIP 用于理解 | "语义特征" | CLIP 系列的视觉塔，提供丰富的概念特征但重建能力差 |
| VQ 用于生成 | "重建编码" | 向量量化 token，能够干净地解码回像素 |
| JanusFlow | "修正流变体" | 用连续的流匹配生成头替代 VQ 的 Janus-Pro |
| 路由标签 | "任务标签" | 选择输入编码器的提示标记（`<understand>` / `<generate>`） |

## 延伸阅读

- [Wu et al. — Janus (arXiv:2410.13848)](https://arxiv.org/abs/2410.13848)
- [Chen et al. — Janus-Pro (arXiv:2501.17811)](https://arxiv.org/abs/2501.17811)
- [Ma et al. — JanusFlow (arXiv:2411.07975)](https://arxiv.org/abs/2411.07975)
- [InternVL-U (arXiv:2603.09877)](https://arxiv.org/abs/2603.09877)
- [Dong et al. — DreamLLM (arXiv:2309.11499)](https://arxiv.org/abs/2309.11499)
