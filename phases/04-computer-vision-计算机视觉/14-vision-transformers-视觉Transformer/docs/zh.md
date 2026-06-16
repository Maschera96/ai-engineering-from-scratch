# 愿景Transformers (ViT)

> 将图像切割成补丁，将每个补丁视为一个单词，运行标准转换器。别回头。

**类型：** Build
**语言：** Python
**先修：** 第 7 阶段第 02 课（自我注意力），第 4 阶段第 04 课（图像分类）
**时间：** 约 45 分钟

## 学习目标

- 从头开始实现补丁嵌入、学习位置嵌入、类标记和转换器编码器块，以构建最小的 ViT
- 解释为什么 ViT 被认为需要大量预训练数据，直到 DeiT 和 MAE 证明并非如此
- 比较 ViT、Swin 和 ConvNeXt 的架构先验（无、本地窗口注意力、卷积主干）
- 使用 `timm` 和标准线性探针/微调配方在小数据集上微调预训练的ViT

## 问题

十年来，卷积一直是计算机视觉的代名词。 CNNs 具有很强的归纳偏差——局部性、翻译等变性——没有人认为你可以取代它们。然后是多索维茨基等人。 （2020）表明，应用于扁平化图像块的普通转换器（根本没有卷积机制）可以在规模上匹配或击败最好的CNNs。

捕获量是“大规模的”。 ViT 在ImageNet-1k 上输给了ResNet。 ViT 在 ImageNet-21k 或 JFT-300M 上预训练，然后在 ImageNet-1k 上进行微调，击败了它。结论是，Transformer缺乏有用的先验知识，但可以从足够的数据中学习它们。随后的工作（DeiT、MAE、DINO）表明，通过正确的训练方法——强增强、自监督预训练、蒸馏——ViTs 在小数据上也能很好地训练。

到 2026 年，纯 CNN 在边缘设备上仍然具有竞争力（ConvNeXt 最强），但 Transformer 主导其他一切：分割（Mask2Former、SegFormer）、检测（DETR、RT-DETR）、多模态（CLIP、SigLIP）、视频（VideoMAE、VJEPA）。 ViT 块结构是需要了解的。

## 概念

### 管道

```mermaid
flowchart LR
    IMG["图片<br/>(3, 224, 224)"] --> PATCH["补丁嵌入<br/>转换 16x16 s=16<br/>-> (768, 14, 14)"]
    PATCH --> FLAT["Flatten to<br/>(196, 768) tokens"]
    FLAT --> CAT["前置<br/>[CLS] 令牌"]
    CAT --> POS["添加学习的<br/>位置嵌入"]
    POS --> ENC["N 个转换器<br/>编码器块"]
    ENC --> CLS["获取[CLS]<br/>令牌输出"]
    CLS --> HEAD["MLP分类器"]

    style PATCH fill:#dbeafe,stroke:#2563eb
    style ENC fill:#fef3c7,stroke:#d97706
    style HEAD fill:#dcfce7,stroke:#16a34a
```

七步。补丁 -> 标记 -> 注意力 -> 分类器。每个变体（DeiT、Swin、ConvNeXt、MAE 预训练）都会更改七个变体中的一两个，并保留其余部分。

### 补丁嵌入

第一次会议是秘密。内核大小为 16，步长为 16，因此 224x224 图像变成由 16x16 块组成的 14x14 网格，每个块都投影到 768 维的嵌入。该单个转换既可以修补又可以线性投影。

```
Input:  (3, 224, 224)
Conv (3 -> 768, k=16, s=16, no padding):
Output: (768, 14, 14)
Flatten spatial: (196, 768)
```

196 patches = 196 个令牌。每个令牌的特征维度为 768 (ViT-B)、1024 (ViT-L) 或 1280 (ViT-H)。

### 类令牌

序列前面添加一个学习向量：

```
tokens = [CLS; patch_1; patch_2; ...; patch_196]   shape (197, 768)
```

在 N 个转换器块之后，`[CLS]` 输出是全局图像表示。分类头只读取这一个向量。

### 位置嵌入

Transformers没有内置的空间位置概念。将学习向量添加到每个标记：

```
tokens = tokens + learned_pos_embedding   (also shape (197, 768))
```

embedding是模型的一个参数；基于梯度的训练使其适应二维图像结构。正弦二维替代方案存在，但在实践中很少使用。

### Transformer编码器块

Standard. Multi-head self-attention, MLP, residual connections, pre-LayerNorm.

```
x = x + MSA(LN(x))
x = x + MLP(LN(x))

MLP is two-layer with GELU: Linear(d -> 4d) -> GELU -> Linear(4d -> d)
```

ViT-B/16 stacks 12 of these blocks, each with 12 attention heads, totalling 86M parameters.

### 为什么预 LN

早期的 Transformer 使用 LN 后 (`x = LN(x + sublayer(x))`)，并且在没有预热的情况下很难训练超过 6-8 层。 Pre-LN (`x = x + sublayer(LN(x))`) 无需预热即可稳定训练更深的网络。每个 ViT 和每个现代法学硕士都使用 LN 预科课程。

### 补丁大小权衡

- 16x16 补丁 -> 196 个标记，标准。
- 32x32 补丁 -> 49 个标记，速度更快但分辨率较低。
- 8x8 补丁 -> 784 个令牌，更精细，但 O(n^2) 注意力成本扩展得很严重。

更大的patches = 更少的tokens = 更快但空间细节更少。 SwinV2 在分层窗口中使用 4x4 补丁。

### DeiT 在ImageNet-1k 上训练ViT 的秘诀

原来的ViT需要JFT-300M才能击败CNNs。 DeiT（Touvron 等人，2020）仅在 ImageNet-1k 上将ViT-B 训练到 81.8% top-1，并进行了四项更改：

1. 重度增强：RandAugment、Mixup、CutMix、Random Erasing。
2. 随机深度（在训练期间随机丢弃整个块）。
3. 重复增强（每批对同一图像采样 3 次）。
4. 来自 CNN 老师的提炼（可选，进一步提高准确性）。

每一个现代的 ViT 训练方法都源自 DeiT。

### Swin 与 ConvNeXt

- **Swin**（Liu et al., 2021）——基于窗口的注意力。每个块都在本地窗口内参与；交替的块移动窗口以混合窗口之间的信息。在保留注意力运算符的同时，先带回类似 CNN 的局部性。
- **ConvNeXt** (Liu et al., 2022) - 重新设计的 CNN 匹配 Swin 的架构选择（深度卷积、LayerNorm、GELU、反向瓶颈）。表明差距不是“注意力 vs 卷积”而是“现代训练配方+架构”。

2026年，ConvNeXt-V2和Swin-V2均达到生产级；正确的选择取决于您的推理堆栈（ConvNeXt 可以更好地编译边缘）和预训练语料库。

### MAE预训练

屏蔽自动编码器（He et al., 2022）：随机屏蔽 75% 的补丁，训练编码器仅处理可见的 25%，训练小型解码器根据编码器的输出重建屏蔽补丁。预训练后，丢弃解码器并微调编码器。

MAE 使 ViT 可以单独在 ImageNet-1k 上训练，达到 SOTA，并且是当前默认的自我监督配方。

## Build It

### 第 1 步：补丁嵌入

```python
import torch
import torch.nn as nn

class PatchEmbedding(nn.Module):
    def __init__(self, in_channels=3, patch_size=16, dim=192, image_size=64):
        super().__init__()
        assert image_size % patch_size == 0
        self.proj = nn.Conv2d(in_channels, dim, kernel_size=patch_size, stride=patch_size)
        num_patches = (image_size // patch_size) ** 2
        self.num_patches = num_patches

    def forward(self, x):
        x = self.proj(x)
        return x.flatten(2).transpose(1, 2)
```

一种转换，一种压平，一种转置。这就是整个图像到令牌的步骤。

### 步骤2：Transformer块

Pre-LN、多头自注意力、带有 GELU 的 MLP、残差连接。

```python
class Block(nn.Module):
    def __init__(self, dim, num_heads, mlp_ratio=4, dropout=0.0):
        super().__init__()
        self.ln1 = nn.LayerNorm(dim)
        self.attn = nn.MultiheadAttention(dim, num_heads, dropout=dropout, batch_first=True)
        self.ln2 = nn.LayerNorm(dim)
        self.mlp = nn.Sequential(
            nn.Linear(dim, dim * mlp_ratio),
            nn.GELU(),
            nn.Dropout(dropout),
            nn.Linear(dim * mlp_ratio, dim),
            nn.Dropout(dropout),
        )

    def forward(self, x):
        a, _ = self.attn(self.ln1(x), self.ln1(x), self.ln1(x), need_weights=False)
        x = x + a
        x = x + self.mlp(self.ln2(x))
        return x
```

`nn.MultiheadAttention` 处理头部分割、缩放点积和输出投影。 `batch_first=True` 所以形状是`(N, seq, dim)`。

### 第 3 步：ViT

```python
class ViT(nn.Module):
    def __init__(self, image_size=64, patch_size=16, in_channels=3,
                 num_classes=10, dim=192, depth=6, num_heads=3, mlp_ratio=4):
        super().__init__()
        self.patch = PatchEmbedding(in_channels, patch_size, dim, image_size)
        num_patches = self.patch.num_patches
        self.cls_token = nn.Parameter(torch.zeros(1, 1, dim))
        self.pos_embed = nn.Parameter(torch.zeros(1, num_patches + 1, dim))
        self.blocks = nn.ModuleList([
            Block(dim, num_heads, mlp_ratio) for _ in range(depth)
        ])
        self.ln = nn.LayerNorm(dim)
        self.head = nn.Linear(dim, num_classes)
        nn.init.trunc_normal_(self.pos_embed, std=0.02)
        nn.init.trunc_normal_(self.cls_token, std=0.02)

    def forward(self, x):
        x = self.patch(x)
        cls = self.cls_token.expand(x.size(0), -1, -1)
        x = torch.cat([cls, x], dim=1)
        x = x + self.pos_embed
        for blk in self.blocks:
            x = blk(x)
        x = self.ln(x[:, 0])
        return self.head(x)

vit = ViT(image_size=64, patch_size=16, num_classes=10, dim=192, depth=6, num_heads=3)
x = torch.randn(2, 3, 64, 64)
print(f"output: {vit(x).shape}")
print(f"params: {sum(p.numel() for p in vit.parameters()):,}")
```

大约 280 万个参数——CPU 上一个很小的ViT。真实ViT-B为86M；与`dim=768, depth=12, num_heads=12`相同的类定义。

### 第 4 步：健全性检查 — 单图像推理

```python
logits = vit(torch.randn(1, 3, 64, 64))
print(f"logits: {logits}")
print(f"probs:  {logits.softmax(-1)}")
```

运行应该没有错误。概率总和为 1。

## Use It

`timm` 为每个 ViT 变体提供 ImageNet 预训练权重。一行：

```python
import timm

model = timm.create_model("vit_base_patch16_224", pretrained=True, num_classes=10)
```

`timm` 是 2026 年视觉Transformer的生产默认设置。在同一 API 下支持 ViT、DeiT、Swin、Swin-V2、ConvNeXt、ConvNeXt-V2、MaxViT、MViT、EfficientFormer 以及其他数十种。

对于多模式工作（图像 + 文本），`transformers` 提供 CLIP、SigLIP、BLIP-2、LLaVA。所有这些中的图像编码器都是 ViT 变体。

## Ship It

This lesson produces:

- `outputs/prompt-vit-vs-cnn-picker.md` — 根据数据集大小、计算和推理堆栈在 ViT、ConvNeXt 或 Swin 之间进行选择的提示。
- `outputs/skill-vit-patch-and-pos-embed-inspector.md` — 一种验证 ViT 的补丁嵌入和位置嵌入形状是否与模型的预期序列长度匹配的技能，捕获最常见的移植错误。

## 练习

1. **（简单）** 打印每个中间张量的形状，以便向前传递通过上面的微小ViT。确认：输入`(N, 3, 64, 64)` -> 修补`(N, 16, 192)` -> 使用CLS `(N, 17, 192)` -> 分类器输入`(N, 192)` -> 输出`(N, num_classes)`。
2. **（中）** 在第 4 课的合成 CIFAR 数据集上微调预训练的 `timm` ViT-S/16。与对相同数据进行的 ResNet-18 微调进行比较。报告训练时间和最终准确性。
3. **（难）** 对微小的ViT实施MAE预训练：屏蔽75％的补丁，训练编码器+一个小型解码器来重建屏蔽的补丁。评估预训练前后合成数据的线性探针准确性。

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|----------------|----------------------|
| Patch embedding | “第一次会议” | 具有内核 size = stride = 补丁大小的卷积；将图像变成令牌嵌入网格 |
| 类令牌 | “[CLS]” | 学习到的向量添加到令牌序列之前；它的最终输出是全局图像表示 |
| 位置嵌入 | 《学到了POS》 | 添加到每个标记的学习向量，以便Transformer知道每个补丁来自哪里 |
| 预LN | “子层之前的 LayerNorm” | 稳定Transformer变体：`x + sublayer(LN(x))`而不是`LN(x + sublayer(x))` |
| 多头注意力 | “平行关注” | 标准转换器注意力分为 num_heads 个独立子空间，然后连接 |
| ViT-B/16 | "Base, patch 16" | 规范大小：dim=768、depth=12、heads=12、patch_size=16、image=224； 约 86M 参数 |
| 德伊特 | “数据高效ViT” | ViT 单独在 ImageNet-1k 上进行了强增强训练；事实证明，并不严格需要大型预训练数据集 |
| MAE | "Masked autoencoder" | 自监督预训练：屏蔽 75% 的 patch，重建；占主导地位的 ViT 预训练方法 |

## 延伸阅读

- [一张图像胜过 16x16 个单词（Dosovitskiy 等人，2020）](https://arxiv.org/abs/2010.11929) — ViT 论文
- [DeiT：数据高效图像 Transformers (Touvron et al., 2020)](https://arxiv.org/abs/2012.12877) — 如何单独在 ImageNet-1k 上训练 ViT
- [Masked Autoencoders 是可扩展的视觉学习器（He et al., 2022）](https://arxiv.org/abs/2111.06377) — MAE 预训练
- [timm 文档](https://huggingface.co/docs/timm) — 您将在生产中使用的每个视觉转换器的参考
