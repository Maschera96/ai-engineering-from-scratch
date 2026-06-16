# Emu3：用于图像与视频生成的下一词元预测

> BAAI 的 Emu3（Wang et al.，2024 年 9 月）是本应终结扩散模型与自回归之争的 2024 年成果。一个 Llama 风格的纯解码器 transformer，仅以下一词元预测目标训练，跨越文本 + VQ 图像词元 + 3D VQ 视频词元的统一词表，在图像生成上击败 SDXL，在感知上击败 LLaVA-1.6。没有 CLIP 损失。没有扩散调度。推理阶段使用无分类器引导来提升质量，但核心训练目标是基于 teacher forcing 的下一词元预测。论文发表于 Nature。本课研读 Emu3 的论点——为何更好的 tokenizer 加上规模就是你所需要的一切——并与扩散方法做对比。

**类型：** 学习
**语言：** Python（标准库，3D 视频 tokenizer 数学 + 自回归采样器骨架）
**前置：** Phase 12 · 11（Chameleon）
**时间：** ~120 分钟

## 学习目标

- 解释为何 Emu3 的单损失下一词元目标能奏效，尽管长期以来人们认为图像质量必须依赖扩散。
- 描述 3D 视频 tokenizer：时空 VQ 码本长什么样，为何 patch 要跨越时间。
- 在（训练算力、推理成本、质量上限）三方面对比 Emu3 与 Stable Diffusion XL。
- 说出同一个 Emu3 模型扮演的三种角色：Emu3-Gen（图像生成）、Emu3-Chat（感知）、Emu3-Stage2（视频生成）。

## 问题

截至 2024 年的传统观点：图像生成需要扩散。论据是：离散图像词元丢失了太多信息以致无法重建细节，而自回归采样会在数千个词元间累积误差。Stable Diffusion、DALL-E 3、Imagen、Midjourney 都使用某种形式的扩散。Chameleon（第 12.11 课）在小规模上部分推翻了这一点，但在质量上未能匹敌 SDXL。

Emu3 正面迎击了这个论据。其主张是：更好的视觉 tokenizer + 足够的规模 + 下一词元损失 = 在同一个也能做感知的模型里实现击败扩散的图像生成。

这个赌注在发表时颇具争议。两年过去，开源统一生成家族（Emu3、Show-o、Janus-Pro、Transfusion）已成为研究的默认路径；生产级前沿模型似乎也使用某种变体。

## 概念

### Emu3 tokenizer

关键要素是视觉 tokenizer。Emu3 训练了一个自定义的 IBQ 类 tokenizer（Inverse Bottleneck Quantizer，SBER-MoVQGAN 家族），每个词元做 8x8 的分辨率缩减。一张 512x512 的图像在码本大小 32768 下变为 64x64 = 4096 个词元。

这比 Chameleon 在 K=8192 下每张 512x512 的 1024 个词元更多，但每个词元更便宜（更小的码本查找、更简单的编解码器）。关键指标：重建 PSNR 为 30.5 dB，与 Stable Diffusion 在 32 dB 的连续潜空间相当。

对于视频：一个 3D VQ tokenizer 将一个时空 patch（4x4x4 像素）编码为一个整数。一段 8 FPS 的 4 秒片段有 32 帧；在 256x256 分辨率下做 4 倍空间和 4 倍时间缩减，词元数为 (256/4) * (256/4) * (32/4) = 64 * 64 * 8 = 32,768 个词元。

tokenizer 质量就是上限。Emu3 的贡献部分在于"我们训练了一个非常好的 tokenizer"。

### 单损失训练

Emu3 使用一个目标：在文本词元、2D 图像词元和 3D 视频词元的共享词表上做下一词元预测。训练时权重会乘以特定模态的因子以平衡贡献，但损失函数完全相同。

在以下混合数据上训练：
- 图像生成：`<text caption> <image> image_tokens </image>`
- 图像感知：`<image> image_tokens </image> <question> text_tokens`
- 视频生成：`<text caption> <video> video_tokens </video>`
- 视频感知：类似。
- 纯文本：标准 NTP。

模型从数据分布中学会何时发出图像词元、何时发出文本词元。生成能力源自模型在 `<image>` 标签之后预测图像词元。

### 无分类器引导与温度

自回归图像生成在推理阶段使用无分类器引导（CFG）后会大幅改善。Emu3 采用了它：生成两次，一次用完整 caption，一次用空 caption，用引导权重（典型值 3.0-7.0）混合 logits。这与扩散使用的 CFG 技巧相同，被借用到了自回归场景。

温度很重要：太高会产生伪影；太低会模式坍缩。Emu3 推荐的温度为感知 1.0、图像生成 0.8。

### 三种角色，一个模型

Emu3 以三个功能上不同的 API 发布，但底层是同一套权重：

- Emu3-Gen。图像生成。输入文本，输出图像词元。
- Emu3-Chat。VQA 和图像描述。输入图像（词元），输出文本。
- Emu3-Stage2。视频生成和视频 VQA。输入文本或视频，输出文本或视频。

没有任务专属的头。只是不同的提示模板。同一个 checkpoint。

### 基准测试

来自 Emu3 论文（2024 年 9 月）：

- 图像生成：在 MJHQ-30K FID（5.4 对 5.6）、GenEval 总分（0.54 对 0.55——统计上打平）上击败 SDXL，Deep-Eval 综合分持平。
- 图像感知：在 VQAv2（75.1 对 72.4）上击败 LLaVA-1.6，在 MMMU 上大致打平。
- 视频生成：4 秒片段质量在 FVD 上与 Sora 时代公开基准测试过的模型相当。

这些数字并非总是领先——Emu3 在这里让一分、在那里赢一分——但"下一词元预测就是你所需要的一切"这一主张在各模态上都站得住脚。

### 算力成本

Emu3 用一个 7B 参数模型在约 3000 亿个多模态词元上训练。GPU 小时数大致与 Llama-2-7B 预训练相当（A100 级别硅片上的 2k-4k GPU-年）。像 Stable Diffusion 3 这样的扩散模型在相似预算内训练，但需要独立的文本编码器和更复杂的流水线。

在推理时，Emu3 每张图像比 SDXL 慢：4096 个图像词元在 30 tok/s 下约为每张 512x512 图像 2 分钟，而 SDXL 为 2-5 秒。投机解码和 KV-cache 优化能缩小差距但无法消除。自回归图像生成算力密集；这是长期存在的权衡。

### 为何重要

Emu3 的深层贡献是概念性的。如果下一词元预测能扩展到在图像生成上匹敌扩散，那么统一模型路径（一个损失、一个主干、任意模态）就是可行的。未来的模型不需要独立的文本编码器、独立的扩散调度器、独立的 VAE。一个 transformer，每个模态一个 tokenizer，加上规模。

Show-o、Janus-Pro 和 InternVL-U 都建立在或挑战这一论点之上。截至 2025 年，中国实验室（BAAI、DeepSeek）在这个方向上的发表比美国实验室更激进。

## 动手用

`code/main.py` 构建了两个玩具部件：

- 一个 2D 与 3D VQ tokenizer 计数计算器：给定（分辨率、patch、片段长度、FPS），计算图像与视频的词元数。
- 一个带无分类器引导和温度的自回归图像词元采样器。

CFG 实现匹配 Emu3 的配方——用引导权重混合条件 logits 和无条件 logits。

## 交付

本课产出 `outputs/skill-token-gen-cost-analyzer.md`。给定一份生成产品规格（图像或视频、目标分辨率、质量等级、延迟预算），它计算词元数、推理成本，并在 Emu3 家族与扩散之间做选择。

## 练习

1. Emu3 在 8x8 缩减下每张 512x512 图像产生 4096 个词元。计算 1024x1024 和 2048x2048 的等效值。推理延迟会发生什么变化？

2. 阅读 Emu3 第 3.3 节关于视频 tokenizer 的内容。描述 3D VQ patch 的形状，以及为何是 4x4x4 而非 8x8x1。

3. 无分类器引导权重 5.0 对 3.0：有什么视觉效果？在 `code/main.py` 中追踪这一数学过程。

4. 计算 Emu3-7B 在 3000 亿词元下的训练 FLOPs，并与 Stable Diffusion 3 对比。哪个训练起来更昂贵？

5. Emu3 在 FID 上击败 SDXL，但在 VQAv2 上不及专用 VLM。解释为何统一损失方法在不同基准上相对专才表现出不同的强项。

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|-----------------|------------------------|
| Next-token prediction | "NTP" | 标准自回归损失：给定 token[0..i] 预测 token[i+1]；只要做了词元化，对每个模态都有效 |
| IBQ tokenizer | "Inverse bottleneck quantizer" | 一类 VQ-VAE，码本更大（32768+），重建质量优于 Chameleon |
| 3D VQ | "Spatiotemporal quantizer" | 由（时间、行、列）索引的码本；一个词元覆盖一个 4x4x4 像素立方体 |
| Classifier-free guidance | "CFG" | 用权重 gamma 混合条件和无条件 logits；在推理时提升图像质量 |
| Unified vocabulary | "Shared tokens" | 文本 + 图像 + 视频都取自同一整数空间；模型预测接下来出现的任意模态 |
| MJHQ-30K | "图像生成基准" | 含 3 万条提示的 Midjourney 质量基准；Emu3 在此报告 FID |

## 延伸阅读

- [Wang et al. — Emu3: Next-Token Prediction is All You Need (arXiv:2409.18869)](https://arxiv.org/abs/2409.18869)
- [Sun et al. — Emu: Generative Pretraining in Multimodality (arXiv:2307.05222)](https://arxiv.org/abs/2307.05222)
- [Liu et al. — LWM (arXiv:2402.08268)](https://arxiv.org/abs/2402.08268)
- [Yu et al. — MAGVIT-v2 (arXiv:2310.05737)](https://arxiv.org/abs/2310.05737)
- [Tian et al. — VAR (arXiv:2404.02905)](https://arxiv.org/abs/2404.02905)
