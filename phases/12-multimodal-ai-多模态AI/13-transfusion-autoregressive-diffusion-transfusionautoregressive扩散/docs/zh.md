# Transfusion：在单一 Transformer 中实现自回归文本 + 扩散图像

> Chameleon 与 Emu3 把全部赌注押在离散 token 上。它们能用，但量化瓶颈清晰可见——图像质量停留在连续空间扩散模型之下。Transfusion（Meta，Zhou 等人，2024 年 8 月）押了相反的注：让图像保持连续，彻底丢掉 VQ-VAE，用两种损失训练同一个 transformer。文本 token 走 next-token-prediction，图像 patch 走 flow-matching / 扩散损失。两个目标优化同一组权重。Stable Diffusion 3 背后的架构（MMDiT）是它的近亲。本课研读 Transfusion 的核心论点，构建一个玩具级双损失训练器，并梳理那个让单个 transformer 同时干两件活的注意力掩码。

**类型：** 构建
**语言：** Python（标准库，MNIST 规模玩具上的双损失训练器）
**前置：** 阶段 12 · 11（Chameleon），阶段 8（生成式 AI）
**时间：** 约 180 分钟

## 学习目标

- 搭建一个在单一骨干上运行两种损失的 transformer（文本 token 上的 NTP，图像 patch 上的扩散 MSE）。
- 解释为什么图像 patch 之间的双向注意力加上文本 token 之间的因果注意力是正确的掩码选择。
- 在计算量、质量和代码复杂度上，对比 Transfusion 风格（连续图像，扩散损失）与 Chameleon 风格（离散图像，NTP）。
- 说出 MMDiT 的贡献：每个 block 上的模态专属权重，残差流上的联合注意力。

## 问题

离散 vs 连续图像 token 的争论比 LLM 还要古老。连续表示（原始像素、VAE 潜变量）保留细节。离散 token（VQ 索引）契合 transformer 原生的词表，但在量化这一步损失细节。

Chameleon / Emu3 选择离散：一种损失，一种架构，但图像保真度受限于分词器质量。

扩散模型选择连续：图像质量极佳，但它是一个与 LLM 分离的模型，需要复杂的噪声调度工程，且无法与文本生成干净地整合。

Transfusion 提出疑问：我们能不能两者兼得？让图像保持连续，仍然训练一个模型，用缝合进同一个梯度步的两种损失。

## 概念

### 双损失架构

单个纯解码器 transformer 处理一个包含以下内容的序列：

- 文本 token（离散，来自 BPE 词表）。
- 图像 patch（连续，16x16 像素块，通过线性嵌入投影到隐藏维度——与 ViT 编码器的输入相同）。
- 标记连续 patch 所在位置的 `<image>` 和 `</image>` 标签。

前向传播运行一次。损失为每个 token 选取两个 head 中的一个：

- 对文本 token：在词表 logits head 上做标准交叉熵。
- 对图像 patch：在连续 patch 上做扩散损失——预测加到每个 patch 上的噪声。

梯度流经共享的 transformer 主体。两种损失同时改进共享权重。

### 注意力掩码：因果文本 + 双向图像

文本 token 必须是因果的——你不能让一个文本 token 关注未来的文本，否则 teacher forcing 就会失效。然而图像 patch 表示的是一个快照；它们应当在同一图像块内彼此双向关注。

掩码：

```
M[i, j] = 1 if:
  (i is text and j is text and j <= i)   # causal for text
  OR (i is image and j is image and same_image_block(i, j))   # bidirectional within image
  OR (i is text and j is image and j < i_image_end)   # text attends to previous images
  OR (i is image and j is text and j < i_image_start)   # image attends to preceding text
```

在训练和推理时实现为块三角掩码。

### transformer 内部的扩散损失

扩散损失是标准的：给图像 patch 加噪声，让模型预测噪声（或等价地预测干净的 patch）。Transfusion 的版本使用 flow matching——预测从带噪到干净的速度场。

训练时：
1. 对每个图像 patch x0，采样一个随机时间步 t。
2. 采样噪声 ε，计算 xt = (1-t) * x0 + t * ε（flow matching 的线性插值）。
3. transformer 预测 v_theta(xt, t)；loss = MSE(v_theta(xt, t), ε - x0)。
4. 与来自同一序列的文本 NTP 损失一起反向传播。

推理时，生成过程为：
- 文本 token：标准自回归采样。
- 图像 patch：以先前文本 token 为条件的扩散采样循环（典型为 10-30 步）。

### MMDiT：Stable Diffusion 3 的变体

Stable Diffusion 3（Esser 等人，2024 年 3 月）推出了 MMDiT（Multimodal Diffusion Transformer），时间与 Transfusion 大致相同。这两种架构是同胞。

MMDiT 的关键差异：

- 每个 block 的模态专属权重。每个 transformer block 对文本 token 和图像 patch 分别拥有独立的 Q、K、V 和 MLP 权重。注意力是联合的（跨模态）；其余一切都是模态专属的。
- 整流流（rectified flow）训练。一种特定的 flow-matching 变体，采样方式已知，数学比 DDPM 更简单。
- 规模。MMDiT 是 SD3 的骨干（2B 和 8B 参数变体）。Transfusion 的论文扩展到 7B。

两者都收敛到同一个核心思路：一个 transformer 在文本上运行 NTP，在连续图像表示上运行扩散。

### 为什么它胜过 Chameleon 风格

连续扩散与离散 NTP 在图像生成上的质量差距是可测量的。Transfusion 论文报告：

- 在 7B 参数下，在 FID 上比同等规模的 Chameleon 风格模型高出 3-5 个点。
- 无需训练分词器——图像编码器更简单（线性投影到隐藏维度，与 ViT 的输入层相同）。
- 推理可以并行化图像 patch 的去噪，这是自回归图像 token 做不到的。

缺点：Transfusion 是一个双损失模型，使训练动态更棘手。损失权重需要调优。NTP 与扩散之间的调度失配会导致某一个 head 主导。

### 下游是什么

Janus-Pro（第 12.15 课）通过为理解和生成解耦视觉编码器——理解用 SigLIP，生成用 VQ——同时共享 transformer 主体，对 Transfusion 的思路加以精炼。Show-o（第 12.14 课）把扩散换成离散扩散（掩码预测）。统一生成家族在 Transfusion 之后迅速分支。

2026 年那些能输出图像的生产级 VLM——Gemini 3 Pro、GPT-5、Claude Opus 4.7 的图像生成路径——几乎肯定使用了该家族的某个后代。细节是专有的。

## 用起来

`code/main.py` 在一个极小的类 MNIST 问题上构建一个玩具级 Transfusion：

- 文本标题是描述某个数字（0-9）的短整数序列。
- 图像是 4x4 的字节网格。
- 一对共享权重的线性投影充当 transformer 的替身；文本上做 NTP 损失，带噪 patch 上做 MSE 损失。
- 训练循环交替运行两种损失，注意力掩码是显式的。
- 生成在一次前向传播中产出一个文本标题和一张 4x4 图像。

这个 transformer 是个玩具。双损失的管道布线、注意力掩码的构造和推理循环才是真正的产物。

## 交付

本课产出 `outputs/skill-two-loss-trainer-designer.md`。给定一个新的多模态训练任务（文本 + 图像、文本 + 音频、文本 + 视频），它设计双损失调度（损失权重、掩码形状、共享 vs 模态专属 block）并标记实现风险。

## 练习

1. 一个 Transfusion 风格的模型训练 70% 的文本 token 和 30% 的图像 patch。图像扩散损失的量级约为文本 NTP 损失的 10 倍。什么样的损失权重能平衡它们？

2. 为序列 `[T, T, <image>, P, P, P, P, </image>, T]` 实现块三角掩码。把每个条目标记为 0 或 1。

3. MMDiT 拥有模态专属的 QKV 权重。相比 Transfusion 完全共享的 transformer，这会增加多少参数量开销？在 7B 参数下，这值得吗？

4. 生成：给定一个文本提示，模型先运行 50 个 token 的 NTP，然后碰到 `<image>`，接着在 256 个 patch 上运行 20 个去噪步的扩散。总共需要多少次前向传播？

5. 阅读 SD3 论文第 3 节。描述整流流（rectified flow），以及为什么它比 DDPM 在更少的推理步数内收敛。

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|-----------------|------------------------|
| 双损失训练 | "NTP + 扩散" | 单个 transformer 在同一个梯度步中同时优化文本 token 上的交叉熵和连续图像 patch 上的 MSE |
| Flow matching | "整流流" | 一种扩散变体，预测从噪声到干净数据的速度场；数学比 DDPM 更简单 |
| MMDiT | "多模态 DiT" | Stable Diffusion 3 的架构：联合注意力，模态专属的 MLP 和 norm |
| 块三角掩码 | "因果文本 + 双向图像" | 一种注意力掩码，在文本上是因果的，但在图像区域内是双向的 |
| 连续图像表示 | "无 VQ" | 图像 patch 作为实值向量，而非整数码本索引 |
| 速度预测 | "v-参数化" | 网络输出的是噪声与数据之间的速度场，而非噪声本身 |

## 延伸阅读

- [Zhou et al. — Transfusion (arXiv:2408.11039)](https://arxiv.org/abs/2408.11039)
- [Esser et al. — Stable Diffusion 3 / MMDiT (arXiv:2403.03206)](https://arxiv.org/abs/2403.03206)
- [Peebles & Xie — DiT (arXiv:2212.09748)](https://arxiv.org/abs/2212.09748)
- [Zhao et al. — MonoFormer (arXiv:2409.16280)](https://arxiv.org/abs/2409.16280)
- [Xie et al. — Show-o (arXiv:2408.12528)](https://arxiv.org/abs/2408.12528)
