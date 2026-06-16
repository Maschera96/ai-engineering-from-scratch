# LLaVA 与视觉指令调优

> LLaVA（2023 年 4 月）是这个星球上被复制最多的多模态架构。它用一个 2 层 MLP 取代了 BLIP-2 的 Q-Former，用朴素的 token 拼接取代了 Flamingo 的门控交叉注意力，并在 158k 条由 GPT-4 从纯文本描述生成的视觉指令对话轮次上训练。任何在 2023 到 2026 年间构建 VLM 的从业者，都构建了某种 LLaVA 的变体。LLaVA-1.5 加入了 AnyRes。LLaVA-NeXT 提升了分辨率。LLaVA-OneVision 用一套配方统一了图像、多图像与视频。本课会研读这套配方、实现 projector，并解释为什么"更简单的方案赢了"。

**类型：** 构建
**语言：** Python（标准库，projector + 指令模板构建器）
**前置：** Phase 12 · 02（CLIP）、Phase 11（LLM 工程——指令调优）
**时长：** 约 180 分钟

## 学习目标

- 构建一个 2 层 MLP projector，把 ViT 的图块嵌入（维度 1024）映射到 LLM 的嵌入维度（维度 4096）。
- 走一遍 LLaVA 的两阶段配方：(1) 在 558k 条描述对上做 projector 对齐，(2) 在 158k 条 GPT-4 生成的对话轮次上做视觉指令调优。
- 构造一个 LLaVA 格式的提示，包含图像 token 占位符、系统提示和用户/助手轮次。
- 解释为什么社区从 Q-Former 转向 MLP，尽管 Q-Former 在 token 预算上占优。

## 问题

BLIP-2 的 Q-Former（第 12.03 课）把一张图像压缩到 32 个 token。干净、高效、跑基准测试表现好。但它有两个问题。

第一，Q-Former 是可训练的，但它的损失并非最终任务。阶段 1 训练 ITC+ITM+ITG。阶段 2 训练 LM 损失。这些查询学到的是某种中间表示，随后 LLM 还得去解码它。信息在瓶颈处丢失了。

第二，Q-Former 占用 188M 参数，而在 LLaVA 的 2023 年规模下，你必须把它和目标 LLM 协同设计。换 LLM，就要重训 Q-Former。换视觉编码器，就要重训。每一种组合都是一个独立的研发项目。

LLaVA 的答案简单到令人尴尬：取 ViT 的 576 个图块 token，让每一个都过一个 2 层 MLP（`1024 → 4096 → 4096`），然后把全部 576 个一股脑塞进 LLM 的输入序列。没有瓶颈。不在奇怪的目标上做阶段 1 预训练。只在直接的 LM 损失上训练这个 MLP。

数据从哪来？LLaVA 的第二个洞见：用 GPT-4（纯文本）来生成指令数据。把一张图像的 COCO 描述和边界框数据喂给 GPT-4，让它生成对话、描述和复杂推理问题。158k 条指令-回应轮次，免费得来。没有人工标注。

结果是：一个在 8 块 A100 上跑一天的 VLM，在 MMMU 上击败 Flamingo，并发布了一个社区可以扩展的开放检查点。到 2023 年底，它已经催生了 50 多个分支。

## 概念

### 架构

LLaVA-1.5 在 13B 规模下：
- 视觉编码器：CLIP ViT-L/14 @ 336（阶段 1 冻结，阶段 2 可选解冻）。
- Projector：带 GELU 激活的 2 层 MLP，`1024 → 4096 → 4096`。
- LLM：Vicuna-13B（后来是 Llama-3.1-8B）。

对图像 + 文本提示的前向传播：

```
img -> ViT -> 576 patches of dim 1024
patches -> MLP -> 576 tokens of dim 4096
prompt: system + "<image>" placeholder + user question
replace <image> token with the 576 projected tokens
feed the full sequence to the LLM
decode response
```

图像占据 LLM 上下文中的 576 个 token。在 2048 上下文下，这给文本留下 1472 个 token。在 32k 上下文下，这就是个舍入误差。

### 阶段 1：projector 对齐

冻结 ViT。冻结 LLM。只训练那个 2 层 MLP。数据集：558k 图像-描述对（LAION-CC-SBU）。损失：以投影后的图像 token 为条件，在描述上做语言建模。

在批大小 128、单个 epoch 下，几个小时就能完成。Projector 学会把 ViT 空间映射到 LLM 空间。没有任务专属的监督。

### 阶段 2：视觉指令调优

解冻 projector（仍可训练）。解冻 LLM（通常完全解冻，有时用 LoRA）。在 158k 条视觉指令轮次上训练。

指令数据才是关键所在。Liu 等人是这样生成的：
1. 取一张 COCO 图像。
2. 提取文本描述（5 条人工描述 + 边界框列表）。
3. 用三种提示模板发给 GPT-4：
   - 对话："生成一段用户与助手关于这张图像的来回对话。"
   - 详细描述："给出对这张图像丰富、详尽的描述。"
   - 复杂推理："提出一个需要对图像进行推理的问题，然后回答它。"
4. 把 GPT-4 的输出解析为（指令，回应）对。

这些都不直接接触图像——只接触文本描述。GPT-4 会幻想出看似合理的图像内容。会有些噪声，但它奏效了：158k 条轮次足以解锁对话能力。

### 为什么社区复制了这套方案

- 没有阶段 1 专属的损失要调。全程都是 LM 损失。
- Projector 几小时就能训完，不用几天。
- 只需重训 projector，就能换掉 LLM（LLaVA-Llama2、LLaVA-Mistral、LLaVA-Llama3）。
- 视觉指令数据流水线使用 GPT-4，为新领域重新生成成本低廉。

### LLaVA-1.5 与 LLaVA-NeXT

LLaVA-1.5（2023 年 10 月）加入了：
- 把学术任务数据（VQA、OKVQA、RefCOCO）混入指令调优。
- 更好的系统提示。
- 2048 → 32k 上下文。

LLaVA-NeXT（2024 年 1 月）加入了：
- AnyRes：把高分辨率图像切成 2x2 或 1x3 的 336x336 裁剪网格，再加一张全局低分辨率缩略图。每块裁剪变成 576 个 token；每张图像总共约 2880 个视觉 token。OCR 和图表任务大幅提升。
- 更好的指令数据混合，加入了 ShareGPT4V（高质量 GPT-4V 描述）。
- 更强的基座 LLM（Mistral-7B、Yi-34B）。

### LLaVA-OneVision

第 12.08 课会深入讲 OneVision。简短版本：同样的 projector，但用一套覆盖单图像、多图像和视频的课程来训练，在一个模型里共享视觉 token 预算。

### 与 Q-Former 的对比

| | Q-Former（BLIP-2） | MLP（LLaVA） |
|---|---|---|
| 每张图像的视觉 token 数 | 32 | 576（基础）或 2880（AnyRes） |
| 可训练参数 | 188M + LM | 40M + LM |
| 阶段 1 损失 | ITC+ITM+ITG | 仅 LM |
| LLM 即插即用 | 需要重训 | 以极少重训即可替换 |
| 多图像 | 别扭 | 自然（拼接） |
| 视频 | 别扭 | 自然（逐帧拼接） |
| token 预算 | 小 | 大 |

MLP 在简单性和 token 灵活性上胜出。Q-Former 在 token 预算上胜出。到 2023 年底，token 预算不再是约束性瓶颈（LLM 上下文增长到 32k-128k+），于是简单性占据了主导。

### 提示格式

```
A chat between a curious human and an artificial intelligence assistant. The assistant gives helpful, detailed, and polite answers to the human's questions. USER: <image> Describe this image in detail. ASSISTANT: The image shows ...
```

`<image>` 是一个占位 token。在分词之前，它会被替换为 576 个视觉 token（用 AnyRes 则是 2880 个）。分词器看到的序列比它训练时见过的略长，但 LLM 能处理这种新颖输入，因为阶段 1 已经教会了它。

### 参数经济性

LLaVA-1.5-7B 的构成：
- CLIP ViT-L/14 @ 336：303M（阶段 1 冻结，阶段 2 常解冻）。
- Projector（2 个线性层）：约 22M 可训练。
- Llama-7B：7B。
- 合计：7.3B 参数。阶段 2 可训练部分：完整的 7B + 22M projector。

阶段 2 的训练成本：在 8xA100 上约 20 小时。这就是关键数字——一天、一个节点、可复现。这正是 LLaVA 得以传播的原因。

## 上手用

`code/main.py` 实现了：

1. 用纯 Python 写的 2 层 MLP projector（玩具规模为 dim 16 → 32 → 32）。
2. 提示构建流水线：系统提示 + 被替换为 N 个投影 token 的 `<image>` + 用户轮次 + 助手生成占位符。
3. 一个可视化工具，展示 576 个 token 的视觉块在 LLM 上下文中占多大（消耗 2k / 32k / 128k 上下文的百分比）。

## 交付它

本课会产出 `outputs/skill-llava-vibes-eval.md`。给定一个 LLaVA 家族的检查点，它会运行一套 10 条提示的 vibes-eval 套件（3 条描述、3 条 VQA、2 条推理、2 条拒答），并报告一份人类可读的评分卡。这不是基准测试；而是一个冒烟测试，用来确认 projector 和 LLM 连接良好。

## 练习

1. 计算 `1024 → 4096 → 4096` 的 2 层 MLP projector 的可训练参数量。在带 GELU 和偏置的情况下，它占 LLaVA-13B 的多大比例？

2. 为一个"拒答"情形构造一个 LLaVA 提示——图像中包含一名私人个体。写出预期的助手回应。为什么 LLaVA 应当零样本地拒绝这一请求，又需要什么训练数据来强化这种拒答？

3. 阅读 LLaVA-NeXT 博客中关于 AnyRes 的部分。计算一张 1344x672 图像在 AnyRes 下的视觉 token 数。与 336x336 下基础的 576 个 token 作对比。

4. LLaVA 的阶段 1 projector 用描述上的 LM 损失训练。如果跳过阶段 1，直接进入阶段 2（视觉指令调优），会发生什么？引用 Prismatic VLMs 的消融实验（arXiv:2402.07865）来回答。

5. LLaVA-Instruct-150k 用 GPT-4 配合 COCO 描述来生成指令。对于一个新领域（医学 X 光片、卫星影像），描述生成领域指令的四步数据流水线。每一步可能出什么问题？

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|----------------|------------------------|
| Projector | "MLP 桥接" | 带 GELU 的 2 层 MLP，把 ViT 维度映射到 LLM 维度 |
| 图像 token | "<image> 占位符" | 提示中的标记，推理前被替换为 N 个投影后的视觉 token |
| 视觉指令调优 | "LLaVA 阶段 2" | 在 GPT-4 生成的（图像，指令，回应）三元组上训练 |
| 阶段 1 对齐 | "Projector 预训练" | 冻结 ViT 和 LLM，用描述上的 LM 损失训练 projector |
| AnyRes | "多裁剪平铺" | 把高分辨率图像切成裁剪网格，拼接每块裁剪的视觉 token |
| LLaVA-Instruct | "GPT-4 生成" | 从 COCO 描述 + GPT-4 合成的 158k 条指令-回应对 |
| 视觉编码器冻结 | "主干锁定" | CLIP 权重在阶段 1 不更新，有时阶段 2 也不更新 |
| ShareGPT4V | "更好的描述" | 由 GPT-4V 生成的 1M 条密集描述，用于更高质量的对齐 |
| VQA | "视觉问答" | 回答关于一张图像的自由形式问题的任务 |
| Prismatic VLMs | "设计空间论文" | Karamcheti 2024 的消融研究，系统性测试 projector 与数据选择 |

## 延伸阅读

- [Liu et al. — Visual Instruction Tuning (arXiv:2304.08485)](https://arxiv.org/abs/2304.08485) —— LLaVA 论文。
- [Liu et al. — Improved Baselines with Visual Instruction Tuning (arXiv:2310.03744)](https://arxiv.org/abs/2310.03744) —— LLaVA-1.5。
- [Chen et al. — ShareGPT4V (arXiv:2311.12793)](https://arxiv.org/abs/2311.12793) —— 密集描述数据集。
- [Karamcheti et al. — Prismatic VLMs (arXiv:2402.07865)](https://arxiv.org/abs/2402.07865) —— 设计空间消融。
- [Li et al. — LLaVA-OneVision (arXiv:2408.03326)](https://arxiv.org/abs/2408.03326) —— 统一单图像、多图像、视频。
