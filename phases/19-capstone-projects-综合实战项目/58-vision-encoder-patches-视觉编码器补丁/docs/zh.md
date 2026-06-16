# 视觉编码器补丁

> 读取像素的视觉模型需要像素 tokenizer。Patch embedding 就是这个 tokenizer。把图像切成方块网格，展平每个方块，通过一个线性层投影，然后加入 2D 位置信号，让 transformer 知道每个方块在原图中的位置。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 19 lessons 30-37 (Track B foundations)
**Time:** ~90 minutes

## 学习目标

- 将图像 tokenize 成固定长度的 patch embedding 序列。
- 实现一个基于 `Conv2d` 的 patch projection，并让它匹配 unfold 再 linear 的数学形式。
- 构建确定性的 2D sinusoidal position embedding，让 token 顺序编码空间位置。
- 在合成 fixture 上验证 patch 数量、embedding 形状，以及 `Conv2d`/unfold 等价性。

## 问题

Transformer 吃的是向量序列。图像是 3 通道网格。把每个像素作为 token 会让序列长度爆炸：一张 224x224 RGB 图像有 150,528 个 token，12 层 transformer 无法承受这样的 attention。把整张图当作一个巨大扁平向量又会丢掉局部性，而 attention 层无法把它恢复回来。编码器前端的任务，是把像素网格压缩成几百个 token，每个 token 总结一个方形区域。

Patch embedding 用一个线性投影解决这个问题。一张 224x224 图像切成 16x16 patch，会产生 14x14 网格，也就是 196 个 patch。每个 patch 从 `(3, 16, 16) = 768` 个像素值展平成一个向量，然后线性层把它映射到模型 hidden dimension。Transformer 看到的是 196 个维度为 `hidden`，通常 768，的 token，外加一个 CLS token。这是网络其余部分能处理的序列。

## 概念

```mermaid
flowchart LR
  Image[224x224x3 image] --> Cut[cut into 16x16 patches]
  Cut --> Grid[14x14 grid of patches]
  Grid --> Flatten[flatten each patch]
  Flatten --> Proj[linear projection]
  Proj --> Tokens[196 tokens of dim hidden]
  Tokens --> Pos[add 2D sinusoidal position]
  Pos --> Out[final token sequence]
```

### 为什么用 patch，而不是像素

Attention 的计算量随序列长度平方增长。196-token 序列每个 head、每层需要 `196 * 196 = 38,416` 个 attention 分数；150,528-token 序列需要 `150,528 * 150,528 = 22.6 billion` 个。Patch 带来 590,000 倍的 attention 计算缩减，而且单个 16x16 区域已经携带足够信号用于高层视觉任务。代价是一个 patch 内部的细粒度空间细节会丢失，这也是为什么下游多模态栈在需要精细定位时，常常会运行第二条高分辨率分支。

### 为什么线性投影就足够

每个 patch 都被当作独立向量。投影会学习一个基：边缘检测器、颜色滤波器、简单纹理。单个线性层很小，ViT-Base 中是 `768 * 768 = 589,824` 个参数，而且训练很快。更深的卷积 stem 也存在，即“hybrid” ViT，但平铺线性投影是标准做法，多数现代 open-weight 编码器都采用这个形状。

### `Conv2d` 技巧

一个无 padding 的 `Conv2d(in_channels=3, out_channels=hidden, kernel_size=patch_size, stride=patch_size)`，在数值上等价于 unfold 再 linear，因为每个输出位置都是 patch 像素和一个滤波器的点积。这个卷积就是 patch projection，而且多数生产代码库会这样实现，因为它在 GPU 上更快，也少一次 reshape。

### 位置嵌入

投影输出的 token 本身不携带顺序。2D sinusoidal embedding 给每个 token 一个固定信号，编码它的 `(row, col)` 位置。embedding 维度的一半用多个频率的 sin/cos 编码行位置，另一半编码列位置。这个编码是确定性的，因此你可以不重新训练就切换分辨率，而且它能干净地插值到模型训练时没见过的网格。

| Component | Shape | Parameters |
|-----------|-------|------------|
| Patch projection (`Conv2d`) | `(hidden, 3, patch, patch)` | `3 * P * P * hidden + hidden` |
| Position embedding (fixed) | `(num_patches, hidden)` | 0 (computed, not learned) |
| CLS token (learned) | `(1, hidden)` | `hidden` |

在 224 分辨率的 ViT-Base/16 中：projection 有 590,592 个参数，CLS token 有 768 个参数，sinusoidal position 为零参数。下一课，第 59 课，会在这个前端之上堆叠 12 层 transformer。

### 用等价性做 sanity check

Patch 步骤有两种写法：`Conv2d` projection 和显式 unfold 再 linear。它们使用相同权重时必须产生相同输出。如果不能，说明 unfold 数学错了，编码器其余部分就是建在沙子上。本课测试会检查这种等价性。

## Build It

`code/main.py` 实现：

- `PatchEmbed`，一个包装 `Conv2d` 做 patch projection 的 `nn.Module`。
- `sinusoidal_2d(grid_h, grid_w, dim)`，无状态函数，构建 2D 位置表。
- `VisionFrontEnd`，把 patch embedding、CLS 前置和位置相加组合成一次 forward pass。
- `synthesize_image(seed)` 辅助函数，用 `numpy.random` 构建确定性的 224x224x3 fixture。
- 一个演示，把一张 fixture 图像送过前端，并打印输出形状、CLS token 范数，以及 position embedding 的一行。

运行：

```bash
python3 code/main.py
```

输出：224x224 fixture 会被 tokenize 为形状 `(1, 197, 768)` 的序列。第一个 token 是 CLS；后面 196 个是 patch tokens。位置嵌入范数在同一行内一致，这是 sinusoidal 的特征。

## Use It

同样的 patch 前端出现在每个现代视觉语言模型中：CLIP ViT-L/14、SigLIP、DINOv2、Qwen-VL 家族和 InternVL 栈，都会从 `Conv2d` patch projection 加位置信号开始。不同家族之间的差异位于下游，CLS 与无 CLS pooling、register tokens、14 与 16 的不同 patch size、通过插值位置处理动态分辨率。本课的 frontend 是这些模型共同站立的基底。

## Tests

`code/test_main.py` 覆盖：

- patch count matches `(image_size / patch_size) ** 2`
- output shape matches `(batch, num_patches + 1, hidden)`
- the `Conv2d` projection equals manual unfold-then-linear on a small fixture
- sinusoidal position table is deterministic across calls
- CLS token broadcasts across batch dim without leakage

运行测试：

```bash
python3 -m unittest code/test_main.py
```

## 练习

1. 把 sinusoidal position 换成学习到的 `nn.Parameter`，并比较一个小型合成分类任务上的第一轮 loss。固定分辨率时 learned positions 胜出；训练后改变分辨率时 sinusoidal 胜出。

2. 把 `Conv2d` 换成显式 `nn.Unfold` 加 `nn.Linear`，并断言输出在浮点容差内匹配。同样的数学，两种写法。

3. 增加对非方形 patch size 的支持，例如用于宽高比输入的 32x16，并验证位置表能处理非方形网格。

4. 在 batch size 1、8、64 下 profile patch 步骤。Patch projection 很少是瓶颈；下游 attention 层占主导。

5. 把前端作为冻结特征提取器，在 4 类合成形状数据集上训练，圆形、正方形、三角形、星形。CLS token 输出应该可以线性分离。

## 关键术语

| Term | What it means |
|------|---------------|
| Patch | 图像中的方形子区域，通常为 14x14 或 16x16 |
| Patch embedding | 把一个展平 patch 线性投影到 hidden dim |
| Sequence length | Patch tokenization 后的 token 数，通常还会加上 CLS |
| Sinusoidal position | 编码 2D 网格坐标的固定 sin/cos 信号 |
| CLS token | 前置到序列中作为 pooling head 的学习向量 |

## 延伸阅读

- An Image is Worth 16x16 Words (ViT, 2021)，了解原始 patch-embed framing。
- Attention Is All You Need (2017)，了解这里改写为 2D 的 sinusoidal position 公式。
- DINOv2 paper，了解 register tokens，这是你可以作为练习 6 添加的扩展。
