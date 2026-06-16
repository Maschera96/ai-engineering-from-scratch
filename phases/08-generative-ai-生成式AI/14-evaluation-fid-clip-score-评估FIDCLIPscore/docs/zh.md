# 评估，FID、CLIP 分数、人类偏好

> 每个生成模型排行榜都会引用 FID、CLIP 分数和人类偏好竞技场的胜率。每个数字都有一个故障模式，有决心的研究人员可以进行游戏。如果您不知道故障模式，您就无法从游戏运行中看出真正的改进。

**类型：** 建造
**语言：** Python
**先修：** 第 8 阶段·01（分类）、第 2 阶段·04（评估指标）
**时间：** 约45分钟

## 问题

生成模型的判断标准是“样本质量”和“条件依从性”。两者都没有封闭式措施。你的模型必须渲染 10,000 张图像；必须给它们分配编号；你必须相信跨型号系列、跨分辨率、跨架构的数字。三个指标在 2014 年至 2026 年的挑战中幸存下来：

- **FID（Fréchet Inception Distance）。** Inception 网络特征空间中两个分布（真实分布和生成分布）之间的距离。越低越好。
- **CLIP 分数。** 生成图像的 CLIP 图像嵌入和提示的 CLIP 文本嵌入之间的余弦相似度。越高越好。措施迅速遵守。
- **人类偏好。** 在同一提示下将两个模型进行正面对决，让人类（或 GPT-4 级模型）选择更好的一个，汇总为 Elo 分数。

您还将看到：IS（初始分数，大部分已退役）、KID、CMMD、ImageReward、PickScore、HPSv2、MJHQ-30k。每一个都纠正了前一个的一个失败。

## 概念

![FID、CLIP 和偏好：三轴，不同的故障模式](../assets/evaluation.svg)

### FID，样品质量

赫塞尔等人。 （2017）。步骤：

1.提取N张真实图像和N张生成图像的Inception-v3特征（2048-D）。
2. 对每个池拟合高斯：计算均值 `μ_r, μ_g` 和协方差 `Σ_r, Σ_g`。
3.FID=`||μ_r - μ_g||² + Tr(Σ_r + Σ_g - 2 · (Σ_r · Σ_g)^0.5)`。

解释：特征空间中两个多元高斯之间的 Fréchet 距离。越低=分布越相似。

失效模式：
- **偏向于小 N。** FID 是特征分布的均方 — 小 N 低估了协方差，给出了错误的低 FID。始终使用 N ≥ 10,000。
- **依赖于 Inception。** Inception-v3 在 ImageNet 上进行训练。远离 ImageNet 的领域（面孔、艺术、文本图像）会产生毫无意义的 FID。使用特定于域的特征提取器。
- **游戏。** 过拟合 Inception 先验会导致 FID 较低，而不会提高视觉质量。用 CMMD 击败它（如下）。

### CLIP 分数 — 及时遵守

雷德福等人。 （2021）。对于生成的图像+提示：

```
clip_score = cos_sim( CLIP_image(x_gen), CLIP_text(prompt) )
```

30k 生成图像的平均值 → 模型之间可比较的标量。

失效模式：
- **CLIP 自己的盲点。** CLIP 的构图推理很弱（“蓝色球体上的红色立方体”经常失败）。模型无需真正遵循复杂的提示即可在 CLIP 分数上排名靠前。
- **短提示偏差。** 短提示在野外有更多的 CLIP 图像匹配。较长的提示自然具有较低的 CLIP 分数。
- **提示游戏。** 在提示中包含“高品质、4k、杰作”会增加 CLIP 分数，但不会改善图像文本绑定。

CMMD（Jayasumana 等人，2024）修复了其中的一些问题：使用 CLIP 特征代替 Inception，使用最大均值差异代替 Fréchet。更擅长检测细微的质量差异。

### 人类偏好，基本事实

选择一组提示。使用模型 A 和模型 B 生成。向人类（或强大的 LLM 法官）展示配对。总胜利分为 Elo 或 Bradley-Terry 分数。基准：

- **PartiPrompts (Google)**：1,600 个不同的提示，12 个类别。
- **HPSv2**：107k 人工注释，广泛用作自动代理。
- **ImageReward**：137k 提示图像偏好对，MIT 许可。
- **PickScore**：根据 Pick-a-Pic 2.6M 偏好进行训练。
- **聊天机器人竞技场风格的图像竞技场**：https://imagearena.ai/ 等。

失效模式：
- **判断差异。** 非专家与专家有不同的偏好。两者都用。
- **及时分发。** 精心挑选的提示有利于一个家庭。始终记录。
- **LLM 法官奖励黑客攻击。** GPT-4 法官被漂亮但错误的输出所愚弄。与人类进行三角测量。

## 一起使用

生产评估报告应包括：

1. 根据保留的真实分布（样本质量）对 10-30k 样本进行 FID。
2. 相同样本的 CLIP 分数/CMMD 与其提示（依从性）。
3. 与之前模型相比，盲区的胜率（总体偏好）。
4. 故障模式分析：50 个随机采样的输出，标记已知问题（手部解剖、文本渲染、一致的对象计数）。

任何单一指标都是谎言。三个确证指标+定性审查是一个主张。

## 构建它

`code/main.py` 在合成“特征向量”上实现 FID、CLIP-score-like 和 Elo 聚合（我们使用 4-D 向量作为 Inception 特征的替代）。你看：

- 小 N 和大 N 上的 FID 计算 — 偏差。
- “CLIP 分数”作为特征池之间的余弦相似度。
- 来自综合偏好流的 Elo 更新规则。

### 第 1 步：四行 FID

```python
def fid(real_features, gen_features):
    mu_r, cov_r = mean_and_cov(real_features)
    mu_g, cov_g = mean_and_cov(gen_features)
    mean_diff = sum((a - b) ** 2 for a, b in zip(mu_r, mu_g))
    trace_term = trace(cov_r) + trace(cov_g) - 2 * sqrt_cov_product(cov_r, cov_g)
    return mean_diff + trace_term
```

### 第 2 步：CLIP 式余弦相似度

```python
def clip_like(image_feat, text_feat):
    dot = sum(a * b for a, b in zip(image_feat, text_feat))
    norm = math.sqrt(dot_self(image_feat) * dot_self(text_feat))
    return dot / max(norm, 1e-8)
```

### 第 3 步：Elo 聚合

```python
def elo_update(r_a, r_b, winner, k=32):
    expected_a = 1 / (1 + 10 ** ((r_b - r_a) / 400))
    actual_a = 1.0 if winner == "a" else 0.0
    r_a_new = r_a + k * (actual_a - expected_a)
    r_b_new = r_b - k * (actual_a - expected_a)
    return r_a_new, r_b_new
```

## 常见陷阱

- **N=1000 时的 FID。** 启发式在 N=10k 下不可靠。报道低氮 FID 的论文纯属游戏。
- **比较不同分辨率的 FID。** Inception 的 299×299 调整大小改变了特征分布。仅在匹配的分辨率下进行比较。
- **报告一粒种子。** 至少运行 3 颗种子。报告标准。
- **通过负面提示来提高 CLIP 分数。** 一些管道通过过拟合提示来提高 CLIP。检查视觉饱和度。
- **来自提示重叠的 Elo 偏差。** 如果两个模型在训练期间都看到基准提示，则 Elo 毫无意义。使用保留的提示集。
- **人类评估付费人群偏向。** 多产的 MTurk 注释者偏向年轻/技术友好。与招募的art/design专家混在一起。

## 使用它

2026 年生产评估协议：

| 支柱 | 最低限度 | 受到推崇的 |
|--------|---------|-------------|
| 样品质量 | 10k 的 FID 与保留的真实金额 | + 5k 上的 CMMD + 每个类别子集上的 FID |
| 立即遵守 | CLIP 得分为 30k | + HPSv2 + ImageReward + VQA 式问答 |
| 偏爱 | 200 对盲法与基线对比 | + 2000 配对人类 + LLM 法官 + 聊天机器人竞技场 |
| 故障分析 | 50 手旗 | 500 个手动标记 + 自动安全分类器 |

一份报告中的所有四个支柱 = 主张。任何一个单独=营销。

## 交付它

保存`outputs/skill-eval-report.md`。 Skill 采用新的模型检查点 + 基线并输出完整的评估计划：样本大小、指标、故障模式探测、签核标准。

## 练习

1. **简单。** 运行 `code/main.py`。比较相同合成分布上 N=100 与 N=1000 时的 FID。报告偏差程度。
2. **中。** 从合成的 CLIP 式特征实现 CMMD（公式参见 Jayasumana 等人，2024）。比较对质量差异的敏感度与 FID。
3. **困难。** 复制 HPSv2 设置：从 Pick-a-Pic 的子集中获取 1000 个图像提示对，根据首选项微调基于 CLIP 的小型记分器，并测量其与保留集的一致性。

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|-----------------|-----------------------|
| 火焰离子化检测器 | “弗雷谢起始距离” | 高斯的 Fréchet 距离适合真实与一代 Inception 特征。 |
| 剪辑评分 | “文本-图像相似度” | CLIP 图像和文本嵌入之间的余弦相似度。 |
| CMMD | “FID 的替代品” | CLIP函数MMD；偏差较小，没有高斯假设。 |
| 是 | 《起始分数》 | Exp KL(p(y|x) || p(y));与现代模型的相关性很差，已退役。 |
| HPSv2 / ImageReward / PickScore | “学习偏好代理” | 根据人类偏好训练的小型模型；用作自动法官。 |
| 埃洛 | 《国际象棋等级》 | 布拉德利-特里两两获胜的总和。 |
| 部分提示 | “基准提示设置” | 1,600 个 Google 策划的提示，涵盖 12 个类别。 |
| FD-恐龙 | 「自补更换」 | 使用 DINOv2 函数的 FD；更适合 ImageNet 域外。 |

## 生产笔记：评估也是一个推理工作量

在 10k 样本上运行 FID 意味着生成 10k 图像。对于单个 L4 上 1024² 的 50 步 SDXL 基础，这大约需要 11 小时的单请求推理。评估预算是真实的，框架正是离线推理场景（最大化吞吐量，忽略TTFT）：

- **硬批处理，忘记延迟。** 离线评估 = 以适合内存的最大大小进行静态批处理。 80GB H100 上的 `pipe(...).images` 和 `num_images_per_prompt=8` 的挂钟运行速度比单个请求快 4-6 倍。
- **缓存真实特征。** 对真实参考集的 Inception (FID) 或 CLIP (CLIP-score, CMMD) 特征提取运行*一次*，存储为 `.npz`。不要根据评估重新计算。

对于 CI/回归门：在每个 PR 的 500 个样本子集上运行 FID + CLIP 评分（约 30 分钟）；每晚运行完整的 10k FID + HPSv2 + Elo。

## 延伸阅读

- [休塞尔等人。 （2017）。通过两个时间尺度更新规则训练的 GAN 收敛到局部纳什均衡 (FID)](https://arxiv.org/abs/1706.08500) — FID 论文。
- [Jayasumana 等人。 （2024）。重新思考 FID：迈向更好的图像生成评估指标 (CMMD)](https://arxiv.org/abs/2401.09603) — CMMD。
- [雷德福等人。 （2021）。从自然语言监督中学习可迁移的视觉模型（CLIP）]（https://arxiv.org/abs/2103.00020）- CLIP。
- [吴等人。 （2023）。 HPSv2：综合人类偏好评分](https://arxiv.org/abs/2306.09341) — HPSv2。
- [徐等人。 （2023）。 ImageReward：学习和评估人类对文本到图像生成的偏好](https://arxiv.org/abs/2304.05977) — ImageReward。
- [于等人。 （2023）。缩放自回归模型以生成内容丰富的文本到图像 (Parti + PartiPrompts)](https://arxiv.org/abs/2206.10789) — PartiPrompts。
- [斯坦等人。 （2023）。暴露生成模型评估指标的缺陷](https://arxiv.org/abs/2306.04675)，故障模式调查。
