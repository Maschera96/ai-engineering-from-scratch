# CLIP 与对比式视觉-语言预训练

> OpenAI 的 CLIP（2021）证明了一个足以驱动未来五年发展的简单想法：仅用带噪声的网络图文对和一个对比损失，就能把一个图像编码器和一个文本编码器对齐到同一个向量空间里。零监督标签。4 亿对。最终得到的嵌入空间能做零样本分类、图文检索，并作为视觉塔接入每一个 2026 年的 VLM。SigLIP 2（2025）用 sigmoid 替换了 softmax，以更低的成本超越了 CLIP 的规模扩展。本课将从 InfoNCE 一路推导到 sigmoid 成对损失的数学，并用标准库 Python 构建训练步骤。

**类型：** 构建
**语言：** Python（标准库，InfoNCE + sigmoid 损失实现）
**前置：** Phase 12 · 01（ViT patch）、Phase 7（Transformer）
**时长：** 约 180 分钟

## 学习目标

- 从互信息推导出 InfoNCE 损失，并实现一个数值稳定的向量化版本。
- 解释为什么 sigmoid 成对损失（SigLIP）能扩展到批大小 32768+，而无需 softmax 所要求的 all-gather 开销。
- 通过构建文本模板（`a photo of a {class}`）并对余弦相似度取 argmax，运行零样本 ImageNet 分类。
- 说出 CLIP / SigLIP 预训练给你的四个调节杆：批大小、温度、提示模板、数据质量。

## 问题

CLIP 之前的视觉是监督式的。收集带标签的数据集（ImageNet：120 万张图像，1000 个类别），训练一个 CNN，然后交付。标签很昂贵，标签会偏向标注者能达成一致的内容，而且标签在不微调的情况下无法迁移到新任务。

图文网络免费提供了十亿以上的松散标注图文对。一张金毛寻回犬的图片，配上 alt 文本 "my dog Max in the park"，就携带了一个监督信号——文本描述了图像。问题是：你能把它转化为有用的训练吗？

CLIP 的答案：把图文对当作一个匹配任务。给定一批 N 张图像和 N 条标题，学会在 N-1 个干扰项中把每张图像匹配到它自己的标题。监督信号是“这两样东西属于一起；这 N-1 样不属于”。没有类别标签。没有人工标注。只有一个对比损失。

最终得到的嵌入空间能做的事情超出了 CLIP 训练的范围。ImageNet 零样本之所以有效，是因为 "a photo of a cat" 嵌入的位置靠近那些从未被明确标注为猫的猫的图片。正是这个押注催生了每一个 2026 年的 VLM。

## 概念

### 双编码器

CLIP 有两座塔：

- 图像编码器 `f`：ViT 或 ResNet，每张图像输出一个 D 维向量。
- 文本编码器 `g`：小型 transformer，每条标题输出一个 D 维向量。

两座塔都将其输出归一化为单位长度。由于两者都是单位范数，相似度即为 `cos(f(x), g(y)) = f(x)^T g(y)`。

对于一批 N 个（图像，标题）对，构建形状为 `(N, N)` 的相似度矩阵 `S`：

```
S[i, j] = cos(f(x_i), g(y_j)) / tau
```

其中 `tau` 是一个可学习的温度（CLIP 初始化为 0.07；在对数空间中学习）。

### InfoNCE 损失

CLIP 在行和列上使用对称的交叉熵：

```
loss_i2t = CE(S, labels=identity)     # each image's positive is its own caption
loss_t2i = CE(S^T, labels=identity)   # each caption's positive is its own image
loss = (loss_i2t + loss_t2i) / 2
```

这就是 InfoNCE。CE 中的 softmax 迫使每张图像与它自己的标题的匹配度高于批内其他每条标题。“负样本”就是所有其他批内项。批越大 = 负样本越多 = 信号越强。CLIP 在批大小 32k 下训练；规模很重要。

### 温度

`tau` 控制 softmax 的锐度。低 tau → 分布尖锐，产生难负样本挖掘的效果。高 tau → 分布平滑，所有样本都有贡献。CLIP 学习 log(1/tau)，并进行截断以防止坍缩。SigLIP 2 固定了初始 tau，转而使用一个可学习的偏置。

### 为什么 sigmoid 扩展性更好（SigLIP）

Softmax 需要整个相似度矩阵保持同步。在分布式训练中，你必须把每个嵌入 all-gather 到每个副本上，然后再做 softmax。其通信开销随 world size 呈二次增长。

SigLIP 用逐元素的 sigmoid 替换 softmax：对每一对 `(i, j)`，损失是一个“它们是否为匹配对？”的二元分类，正类标签是对角线，其余都是负类。损失为：

```
L = -1/N sum over (i, j) [ y_ij log sigmoid(S[i,j]) + (1-y_ij) log sigmoid(-S[i,j]) ]
```

当 `i == j` 时 `y_ij = 1`，否则为 0。每对的损失都是独立的。无需 all-gather。每块 GPU 计算它的本地块并求和。SigLIP 2 能廉价地扩展到批大小 32k-512k，而 CLIP 在此处需要按比例更多的通信。

### 零样本分类

给定 N 个类别名，为每个类别构建一个文本模板：

```
"a photo of a {class}"
```

用文本编码器嵌入每个模板。用图像编码器嵌入你的图像。余弦相似度的 argmax = 预测类别。无需在目标类别上训练。

提示模板很重要。CLIP 的原始论文每个类别使用 80 个模板（普通、艺术、照片、绘画等）并对嵌入求平均。ImageNet 上提升 +3 个点。现代用法通常只选一两个模板。

### 线性探针与微调

零样本是一个基线。线性探针（在冻结的 CLIP 特征之上为你的目标类别训练一个线性层）在域内任务上优于零样本。完整微调在域内优于线性探针，但可能损害零样本迁移能力。三种方案，三种权衡。

### SigLIP 2：NaFlex 与稠密特征

SigLIP 2（2025）新增了：
- NaFlex：单一模型处理可变的宽高比和分辨率。
- 更好的稠密特征，用于分割和深度估计，目标是作为 VLM 中的冻结骨干网络。
- 多语言：在 100+ 种语言上训练，而 CLIP 仅支持英语。
- 10 亿参数规模，而 CLIP 上限为 4 亿。

在 2026 年的开源 VLM 中，SigLIP 2 SO400m/14 是默认的视觉塔。在纯图文检索中，当特定的 LAION-2B 训练分布与你的查询模式匹配时，CLIP 仍是默认选择。

### ALIGN、BASIC、OpenCLIP、EVA-CLIP

ALIGN（Google，2021）：与 CLIP 思路相同，18 亿对规模，90% 带噪声。证明了带噪声数据可以扩展。OpenCLIP（LAION）：在 LAION-400M / 2B 上对 CLIP 的开源复现，多种规模，是首选的开源检查点。EVA-CLIP：从掩码图像建模初始化；是 VLM 的强力骨干。BASIC：Google 的 CLIP+ALIGN 混合方案。同一家族，数据和调参不同。

### 零样本天花板

CLIP 类模型的 ImageNet 零样本上限约为 76%（CLIP-G、OpenCLIP-G）。要突破则需要大得多的数据（SigLIP 2 达到 80%+）或架构改动（监督式头、更多参数）。该基准正在饱和；真正的价值在于下游 VLM 所消费的那个嵌入空间。

```figure
multimodal-fusion
```

## 使用它

`code/main.py` 实现了：

1. 一个玩具双编码器（基于哈希的图像特征、文本字符特征），让你无需 numpy 就能看到 InfoNCE 的形态。
2. 纯 Python 实现的 InfoNCE 损失（通过 log-sum-exp 保证数值稳定）。
3. 用于对比的 sigmoid 成对损失。
4. 一个零样本分类流程：针对一组文本提示计算余弦相似度，取 argmax 作为预测。

运行它并观察损失曲线。绝对数值是玩具级的；其形态与真实 CLIP 训练器输出的一致。

## 交付它

本课产出 `outputs/skill-clip-zero-shot.md`。给定一组图像（通过路径）和一个目标类别列表，它用 CLIP 模板构建文本提示，用指定的检查点（如 `openai/clip-vit-large-patch14`）嵌入两侧，并返回带相似度分数的 top-1 / top-5 预测。该技能拒绝对不在提示列表中的类别做出断言。

## 练习

1. 手工为一批 4 对实现 InfoNCE。构建 4x4 相似度矩阵，运行 softmax，取出对角线，计算交叉熵。用这个手算结果验证你的 Python 实现。

2. SigLIP 除温度外还使用一个偏置参数 `b`：`S'[i,j] = S[i,j]/tau + b`。当批内存在严重的类别不平衡（每行的负样本远多于正样本）时，`b` 起什么作用？阅读 SigLIP 第 3 节（arXiv:2303.15343）。

3. 构建一个猫 vs 狗的零样本分类器。尝试两个提示模板：`a photo of a {class}` 和 `a picture of a {class}`。在 100 张测试图像上测量准确率。模板集成是否优于单个模板？

4. 为一次 512-GPU、批大小 32k 的运行，计算 softmax InfoNCE 与 sigmoid 成对损失的通信成本。哪个按 O(N) 扩展，哪个按 O(N^2)？引用 SigLIP 第 4 节。

5. 阅读 OpenCLIP 缩放定律论文（arXiv:2212.07143，Cherti 等）。从图表中复现他们关于数据扩展的结论：在固定模型规模下，ImageNet 零样本准确率与训练数据规模之间的对数线性关系是怎样的？

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|----------------|------------------------|
| InfoNCE | “对比损失” | 在一批的相似度矩阵上做交叉熵；每一项的正样本是其配对项，负样本是其他一切 |
| Sigmoid 损失 | “SigLIP 损失” | 逐对的二元交叉熵；无 softmax，无 all-gather，在分布式训练中廉价扩展 |
| 温度 | “tau” | 在 softmax/sigmoid 之前缩放 logits 的标量；控制分布的锐度 |
| 零样本 | “免微调分类” | 用文本提示构建类别嵌入，并按余弦相似度分类；不在目标类别上训练 |
| 提示模板 | “a photo of a ...” | 围绕类别名的文本脚手架；可使零样本准确率波动 1-5 个点 |
| 双编码器 | “双塔” | 一个图像编码器 + 一个文本编码器，输出在共享的 D 维空间中 |
| 难负样本 | “棘手的干扰项” | 一个与正样本足够相似、模型必须费力才能区分的负样本 |
| 线性探针 | “冻结 + 一层” | 仅在冻结特征之上训练一个线性分类器；衡量特征质量 |
| NaFlex | “原生灵活分辨率” | SigLIP 2 的能力，可在不缩放的情况下摄入任意宽高比和分辨率的图像 |
| 温度缩放 | “对数参数化的 tau” | CLIP 参数化 `log(1/tau)` 以使梯度表现良好；并截断以防止坍缩到接近零的 tau |

## 延伸阅读

- [Radford 等 — Learning Transferable Visual Models From Natural Language Supervision (arXiv:2103.00020)](https://arxiv.org/abs/2103.00020) — CLIP 论文。
- [Zhai 等 — Sigmoid Loss for Language Image Pre-Training (arXiv:2303.15343)](https://arxiv.org/abs/2303.15343) — SigLIP。
- [Tschannen 等 — SigLIP 2 (arXiv:2502.14786)](https://arxiv.org/abs/2502.14786) — 多语言 + NaFlex。
- [Jia 等 — ALIGN (arXiv:2102.05918)](https://arxiv.org/abs/2102.05918) — 用带噪声的网络数据扩展规模。
- [Cherti 等 — Reproducible scaling laws for contrastive language-image learning (arXiv:2212.07143)](https://arxiv.org/abs/2212.07143) — OpenCLIP 缩放定律。
