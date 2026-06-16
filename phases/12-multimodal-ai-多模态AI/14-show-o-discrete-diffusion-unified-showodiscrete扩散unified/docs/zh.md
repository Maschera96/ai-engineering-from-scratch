# Show-o 与离散扩散统一模型

> Transfusion 混合了连续和离散表示。Show-o（Xie 等人，2024 年 8 月）走的是另一条路：文本 token 使用因果式 next-token 预测，图像 token 则沿用 MaskGIT 的思路使用掩码离散扩散。两者都置于一个带混合注意力掩码的 transformer 中。其结果是在一个主干网络上统一了 VQA、文生图、图像修复以及混合模态生成——每种模态一个分词器，一种损失表述（将 next-token 扩展到掩码预测）。本课讲解 Show-o 的设计——为什么掩码离散扩散是一个并行的、少步数的图像生成器——并与 Transfusion 和 Emu3 进行对比。

**类型：** 学习
**语言：** Python（标准库，掩码离散扩散采样器）
**先修：** Phase 12 · 13（Transfusion）
**时长：** 约 120 分钟

## 学习目标

- 解释掩码离散扩散：先均匀地掩码 token，再让 transformer 恢复它们的调度方案。
- 在速度和质量上对比并行图像解码（Show-o、MaskGIT）与自回归图像解码（Chameleon、Emu3）。
- 说出 Show-o 在一个检查点中处理的三种任务：T2I、VQA、图像修复。
- 选择一种掩码调度（cosine、linear、truncated）并推断其对样本质量的影响。

## 问题

Transfusion 的双损失训练有效，但动态特性更棘手——连续扩散损失与离散 NTP 损失处于不同的数值尺度上。平衡损失权重需要做一次超参数搜索。这套架构有效但复杂。

Show-o 的答案：让两种模态都保持离散（与 Chameleon 一样），但通过掩码离散扩散并行生成图像，而非顺序生成。训练目标因此变成单一的掩码 token 预测，它自然地泛化了 next-token 预测。

## 概念

### 掩码离散扩散（MaskGIT）

最初由 Chang 等人（2022）提出的 MaskGIT 技巧十分优雅。从一张完全被掩码的图像开始（每个 token 都是特殊的 `<MASK>` id）。在每一步，并行预测所有被掩码的 token，然后保留置信度最高的前 K 个预测，并对其余部分重新掩码。经过约 8-16 次迭代后，所有 token 都被填充。每步解掩多少 token 的调度方案是经过调优的——cosine 调度效果很好。

训练很简单：从 [0, 1] 均匀采样一个掩码比例，将其应用到图像的 VQ token 上，训练 transformer 恢复被掩码的那些。这正是 BERT 对文本所做的事，被扩展到了图像生成上。

### Show-o：一个 transformer，混合掩码

Show-o 把 MaskGIT 放进了一个因果语言模型 transformer 中。注意力掩码是这样的：

- 文本 token：因果（标准 LLM）。
- 图像 token：在图像块内部完全双向（这样被掩码的 token 在预测时能看到其他每一个图像 token）。
- 文生图：文本关注先前的图像，图像关注先前的文本。

训练在以下几种之间交替进行：
1. 在文本序列上的标准 NTP。
2. T2I 样本：文本 → 带掩码图像 token 的图像，使用掩码 token 预测损失。
3. VQA 样本：图像 → 带掩码文本 token 的文本（其实就是 NTP）。

统一的损失是对 `<MASK>` token 的交叉熵，它既覆盖文本 NTP（只有最后一个 token 被"掩码"），也覆盖图像掩码扩散（随机子集被掩码）。

### 并行采样

Show-o 用约 16 步生成一张图像，而不是约 1000 步（逐 token 自回归）或约 20 步（扩散）。在每一步，并行预测所有被掩码的 token；提交置信度最高的前 K 个；重复。

对比：
- Chameleon / Emu3（在 token 上自回归）：N_tokens 次前向传递，通常每张图像 1024-4096 次。
- Transfusion（连续扩散）：约 20 步，每步一次完整的 transformer 传递。
- Show-o（掩码离散扩散）：约 16 步，每步一次完整的 transformer 传递。

在相近规模的模型上，Show-o 比 Chameleon 更快，步数大致与 Transfusion 持平，但每步成本更低（离散词表 logits 对比连续 MSE 损失）。

### 一个检查点中的多种任务

Show-o 在推理时支持四种任务，由提示词格式选择：

- 文本生成：标准的自回归文本输出。
- VQA：输入图像，输出文本。
- T2I：输入文本，通过掩码离散扩散输出图像。
- 图像修复：对部分 token 被掩码的图像进行填充。

图像修复能力是掩码预测训练免费带来的。掩码掉 VQ token 网格的某个区域，喂入其余部分加一个文本提示，预测被掩码的 token。

### 掩码调度

每步解掩多少 token 的调度方案决定了质量。Show-o 推荐 cosine：

```
mask_ratio(t) = cos(pi * t / (2 * T))   # t = 0..T
```

在第 0 步，所有 token 都被掩码（比例 1.0）。在第 T 步，没有 token 被掩码。Cosine 把质量集中在预测信息量最大的中段比例上。Linear 调度也可行，但更快进入平台期。

### Show-o2

Show-o2（2025 年的后续工作，arXiv 2506.15564）扩展了 Show-o：更大的 LLM 基座、更好的分词器、改进的掩码调度。架构模式相同。

### Show-o 的定位

在 2026 年的分类法中：

- 离散 token + NTP：Chameleon、Emu3。简单但推理慢。
- 离散 token + 掩码扩散：Show-o、MaskGIT、LlamaGen、Muse。并行采样，但仍受分词器有损影响。
- 连续 + 扩散：Transfusion、MMDiT、DiT。质量最高，训练更复杂。
- VLM 中的连续 + 流匹配：JanusFlow、InternVL-U。最新方案。

按任务选择：当你想要在一个开放模型中以合理速度获得 T2I + 图像修复 + VQA 时，选 Show-o；当质量至上且你能承担双损失的复杂工程时，选 Transfusion。

## 动手用

`code/main.py` 模拟 Show-o 采样：

- 一个 16 个 VQ token 的玩具网格。
- 一个模拟"transformer"，它基于提示词和当前已解掩的 token 预测 logits。
- 使用 cosine 调度进行 8 步并行掩码采样。
- 打印中间状态（掩码模式的演变）和最终 token。

运行它，看着掩码一步步消散。

## 交付

本课产出 `outputs/skill-unified-gen-model-picker.md`。给定一个既需要理解（VQA、图像描述）又需要生成（T2I、图像修复）、且有开放权重约束的产品，它会在 Show-o 系列、Transfusion/MMDiT 系列与 Emu3 / Chameleon 系列之间做出选择，并给出具体的权衡。

## 练习

1. 掩码离散扩散用约 16 步采样。为什么不是 1 步？如果你在第 0 步就解掩所有内容，会出什么问题？

2. 图像修复在掩码扩散下是免费的。提出一个产品用例（真实或假设），在其中 Show-o 的图像修复胜过专用模型。

3. Cosine 调度对比 linear 调度：对 T=8，追踪每步已解掩的 token 数量。哪个更均衡？

4. 一张 512x512 的 Show-o 图像是 1024 个 token。在词表 K=16384 时，模型发出 1024 * log2(16384) = 14,336 比特（约 1.75 KiB）数据。Stable Diffusion 输出 512*512*24 比特 = 6,291,456 比特（约 768 KiB）的原始像素。压缩比是多少，它换来了什么质量？

5. 阅读 LlamaGen（arXiv:2406.06525）。LlamaGen 的类条件自回归图像模型与 Show-o 的掩码方法有何不同？

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|-----------------|------------------------|
| 掩码离散扩散 | "MaskGIT 风格" | 训练去预测被掩码的 token；推理时，迭代地解掩置信度最高的预测 |
| Cosine 调度 | "解掩调度" | 掩码比例随推理步数衰减；将置信度增长集中在中段 |
| 并行解码 | "所有 token 一次到位" | 每一步在一次前向传递中预测整个被掩码 token 序列，然后提交前 K 个 |
| 混合注意力 | "因果 + 双向" | 在文本 token 上是因果、在图像块内部是双向的掩码 |
| 图像修复 | "填充式生成" | 以部分 token 被掩码的图像为条件，预测缺失的那些；由训练目标免费带来 |
| 提交率 | "每步前 K 个" | 每次迭代声明为"完成"的 token 数量；控制推理速度与质量的权衡 |

## 延伸阅读

- [Xie et al. — Show-o (arXiv:2408.12528)](https://arxiv.org/abs/2408.12528)
- [Show-o2 (arXiv:2506.15564)](https://arxiv.org/abs/2506.15564)
- [Chang et al. — MaskGIT (arXiv:2202.04200)](https://arxiv.org/abs/2202.04200)
- [Sun et al. — LlamaGen (arXiv:2406.06525)](https://arxiv.org/abs/2406.06525)
- [Chang et al. — Muse (arXiv:2301.00704)](https://arxiv.org/abs/2301.00704)
