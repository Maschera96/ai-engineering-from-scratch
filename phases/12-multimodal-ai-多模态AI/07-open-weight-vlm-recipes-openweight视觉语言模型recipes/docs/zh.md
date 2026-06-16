# Open-Weight VLM Recipes: What Actually Matters

> 2024-2026 年的 open-weight VLM 文献是一片消融实验表格的森林。Apple 的 MM1 测试了图像编码器、连接器和数据混合的 13 种组合。Allen AI 的 Molmo 证明了详尽的人工标注图注胜过 GPT-4V 蒸馏。Cambrian-1 跑了 20 多个编码器对比。Idefics2 把五轴设计空间形式化。Prismatic VLMs 在一个受控基准上对比了 27 种训练 recipe。在这些噪声中，有一小组结论跨论文成立：图像编码器比连接器架构更重要，数据混合比这两者都更重要，详尽的人工标注图注胜过蒸馏出来的合成数据。这节课替你读完这些表格。

**类型：** 学习 + 实验
**语言：** Python（标准库，消融表解析器 + recipe 选择器）
**前置：** Phase 12 · 05（LLaVA 基线）
**时间：** ~180 分钟

## 学习目标

- 说出五轴 VLM 设计空间：图像编码器、连接器、LLM、数据混合、分辨率调度。
- 读懂一张 MM1 / Idefics2 / Cambrian-1 消融表，并预测哪个旋钮会推动某个给定基准。
- 在给定算力预算和任务组合的情况下，为一个新 VLM 挑选 recipe（编码器、连接器、数据、分辨率）。
- 解释为什么在相同 token 数下详尽的人工标注图注胜过 GPT-4V 蒸馏。

## 问题

存在着数百个 open-weight VLM。"好"与"最先进"之间的差距大部分不在于架构，而在于数据、分辨率调度和编码器选择。当你的模型表现不佳时，知道该先转动哪个旋钮，能为你省下一个耗费 500 万 GPU 小时的错误。

2023 年那一波（LLaVA-1.5、InstructBLIP、MiniGPT-4）跑的是图注对预训练 + LLaVA-Instruct-150k。是个不错的基线，MMMU 顶到大约 35%。

2024 年那一波（MM1、Idefics2、Molmo、Cambrian-1、Prismatic VLMs）跑了穷尽式的消融实验。结果出人意料且实用。

## 概念

### 五轴设计空间

Idefics2（Laurençon 等，2024）给这些轴命了名：

1. 图像编码器。CLIP ViT-L/14、SigLIP SO400m/14、DINOv2 ViT-g/14、InternViT-6B。编码器在 patch 大小、分辨率和预训练目标上各不相同。
2. 连接器。MLP（2-4 层）、Q-Former（32 个 query + 交叉注意力）、Perceiver Resampler（64 个 query）、C-Abstractor（卷积 + 双线性池化）。
3. 语言模型。Llama-3 8B / 70B、Mistral 7B、Phi-3、Gemma-2、Qwen2.5。LLM 大小是参数成本的主导项。
4. 训练数据。图注对（CC3M、LAION）、交错式（OBELICS、MMC4）、指令（LLaVA-Instruct、ShareGPT4V、PixMo、Cauldron）。
5. 分辨率调度。固定 224/336/448、AnyRes、原生动态。训练过程中递增或恒定。

每个生产级 VLM 都要在每个轴上做出选择。MMMU 分数的大部分方差由轴 1、4、5 解释——而不是你挑了哪个连接器。

### 轴 1：编码器 > 连接器

MM1 第 3.2 节表明：从 CLIP ViT-L/14 换成 SigLIP SO400m/14 增加了 3 分以上的 MMMU。把连接器从 MLP 换成 Perceiver Resampler 增加了不到 1 分。Idefics2 复现了这一点：SigLIP > CLIP，在相同 token 数下 Q-Former ≈ MLP ≈ Perceiver。

Cambrian-1 的"Cambrian Vision Encoders Match-Up"（Tong 等，2024）在一个以视觉为中心的基准（CV-Bench）上跑了 20 多个编码器。排行榜顶端是 DINOv2 和 SigLIP 的混合；CLIP 居中；ImageBind 和 ViT-MAE 更靠下。从 CLIP ViT-L 到 DINOv2 ViT-g/14 在 CV-Bench 上的差距约 5-7 分。

2026 年 open VLM 的默认编码器是 SigLIP 2 SO400m/14（用于语义 + 稠密特征），有时会与 DINOv2 ViT-g/14 特征拼接（Cambrian 的"Spatial Vision Aggregator"就是这么做的）。

### 轴 2：连接器设计无关紧要

MM1、Idefics2、Prismatic 和 MM-Interleaved 都得出了相同结论：在固定的视觉 token 数下，连接器架构几乎没有影响。在均值池化的 patch 上加一个 2 层 MLP，在相同 token 预算下，表现与 32-query Q-Former 相差不到 1 分。

真正重要的是 token 数。更多视觉 token = 更多 LLM 算力 = 更好的性能，到某个点之后收益递减。每张图 64 个 token 对 OCR 来说太少。576-1024 个 token 是大多数 open VLM 的甜点区。2048+ 只对文档和图表有帮助。

Q-Former 对比 MLP 是成本问题，不是质量问题：无论图像分辨率多高，Q-Former 都把 token 数封顶在 32-64；MLP 则输出所有 patch token。对于高分辨率输入，Q-Former 节省 LLM 上下文；对于低分辨率，两者差异只是噪声。

### 轴 3：LLM 大小设定上限

把 LLM 从 7B 翻倍到 13B，在每篇 VLM 论文中都可靠地为 MMMU 增加 2-4 分。到 70B 时大多数基准已饱和。VLM 的多模态推理上限就是 LLM 的文本推理上限——视觉编码器只能给它喂数据，不能替它推理。

这就是为什么 Qwen2.5-VL-72B 和 Claude Opus 4.7 在 MMMU-Pro 和 ScreenSpot-Pro 上碾压：语言大脑非常庞大。7B VLM 无法通过巧妙的连接器设计来替代 70B VLM。

### 轴 4：数据——详尽的人工标注图注胜过蒸馏

Molmo + PixMo（Deitke 等，2024）是每个人都应该读的 2024 年成果。Allen AI 让人工标注员用 1-3 分钟的稠密语音转文字方式描述图像，产出了 712K 张稠密标注图像。训练数据里没有任何 GPT-4V 蒸馏。

Molmo-72B 在 11 项基准中全部 11 项击败了 Llama-3.2-90B-Vision。差距不在架构——在于图注质量。详尽的人工标注图注每张图包含的信息量是短网页图注的 5-10 倍，而且在 GPT-4V 蒸馏会产生幻觉的地方仍然保持事实可靠。

ShareGPT4V（Chen 等，2023）和 Cauldron（Idefics2）以人工 + GPT-4V 混合图注沿用了同一套打法。趋势很清晰：对于 2026 年的前沿，图注密度 > 图注数量 > 蒸馏便利性。

### 轴 5：分辨率及其调度

Idefics2 的消融：384 -> 448 增加 1-2 分。448 -> 980 配合图像切分（AnyRes）在 OCR 基准上再增加 3-5 分。平坦的分辨率训练在中等准确率处停滞；分辨率递增（从 224 起，到 448 或原生结束）训练更快、收尾更高。

Cambrian-1 跑了一个分辨率对比 token 数的权衡：在固定算力下，你可以用更低分辨率换更多 token，或用更少 token 换更高分辨率。OCR 中更高分辨率胜出；通用场景理解中低分辨率配更多 token 胜出。

2026 年生产 recipe：第 1 阶段以固定 384 训练，第 2 阶段对 OCR 密集任务用动态分辨率，最高到 1280。

### Prismatic 受控对比

Prismatic VLMs（Karamcheti 等，2024）是那篇控制了所有轴的论文。相同的 13B LLM、相同的指令数据、相同的评估——每次只变化一个轴。结果：

- 每张图的视觉 token 数解释了约 60% 的方差。
- 编码器选择解释了约 20%。
- 连接器架构解释了约 5%。
- 其余一切（数据混合、调度器、LR）解释剩下的约 15%。

这是个粗略的分解，但它是文献中对"我应该先消融什么"最干净的回答。

### 一个面向 2026 的选择器

根据这些证据，2026 年新项目的默认 open-VLM recipe：

- 编码器：SigLIP 2 SO400m/14，用 NaFlex 跑原生分辨率，如果你需要分割/grounding，就与 DINOv2 ViT-g/14 拼接以获得稠密特征。
- 连接器：在 patch token 上加 2 层 MLP。除非你受 token 约束，否则跳过 Q-Former。
- LLM：Qwen2.5 / Llama-3.1 / Gemma 2，省成本选 7B，重质量选 70B，按目标延迟挑选。
- 数据：PixMo + ShareGPT4V + Cauldron，再补充任务特定的指令数据。
- 分辨率：动态（长边最小 256，最大 1280 像素）。
- 调度：第 1 阶段对齐（仅 projector），第 2 阶段全量微调，第 3 阶段任务特定微调。

这些默认值的每一项都可以追溯到本课末尾所引论文中的某个实测消融。

## 用起来

`code/main.py` 是一个消融表解析器和 recipe 选择器。它编码了 MM1 和 Idefics2 的消融表（精简版），让你可以查询：

- "给定预算 X 和任务 Y，哪个 recipe 胜出？"
- "如果我在 7B Llama 上把 SigLIP 换成 CLIP，预期的 MMMU 差值是多少？"
- "要得到 80% 置信度的答案，我应该先消融哪个轴？"

输出是一个带预期基准差值的排序 recipe 列表，以及一条"先消融"建议。

## 交付

本课产出 `outputs/skill-vlm-recipe-picker.md`。给定目标任务组合、算力预算和延迟目标，它会输出一份完整 recipe（编码器、连接器、LLM、数据混合、分辨率调度），并为每个选择附上证明其合理性的消融引用。让工程师不必每次启动新 VLM 项目时都重新发明 Idefics2 的消融表。

## 练习

1. 阅读 MM1 第 3.2 节。对于固定的 2B LLM、预算 5000 万张图，哪个编码器胜出？在 13B LLM 下答案会反转吗？为什么？

2. Cambrian-1 发现，在以视觉为中心的基准上，拼接 DINOv2 + SigLIP 优于单独使用任一者，但在 MMMU 上不增加任何信号。预测哪些基准会受益、哪些会保持不变。

3. 你的目标是在 2B LLM 上做一个移动端 UI 智能体。挑选编码器、连接器、分辨率和数据混合。用具体的消融表证明每个选择的合理性。

4. Molmo 发布了 4B 和 72B 模型。4B 与闭源 7B VLM 不相上下；72B 在 11/11 项基准上击败 Llama-3.2-90B-Vision。这告诉你关于 LLM 大小停滞假说的什么信息？

5. 设计一张消融表，以在 7B VLM 上将数据混合质量与编码器质量隔离开。最少需要多少次训练运行？提出这四个轴的设置。

## 关键术语

| 术语 | 人们怎么说 | 它实际指什么 |
|------|-----------------|------------------------|
| 消融（Ablation） | "转动一个旋钮" | 训练多次运行，每次恰好在一个设计空间轴上有所不同，其余一切保持不变 |
| 连接器（Connector） | "桥" / "projector" | 把视觉编码器输出映射到 LLM token 空间的可训练模块（MLP、Q-Former、Perceiver） |
| 详尽的人工标注图注 | "稠密图注" | 一段多句的人工撰写描述（通常 80-300 token），比网页 alt text 更丰富 |
| 蒸馏（Distillation） | "GPT-4V 图注" | 由更强的专有 VLM 生成的训练数据；方便，但容易继承幻觉 |
| AnyRes / 动态分辨率 | "高分辨率路径" | 通过切片或 M-RoPE 来喂入超出编码器原生分辨率图像的策略 |
| 分辨率递增 | "课程（curriculum）" | 从低分辨率开始并逐步提高的训练调度，加速对齐学习 |
| 以视觉为中心的基准 | "CV-Bench / BLINK" | 侧重细粒度视觉感知而非语言密集型推理的评估 |
| PixMo | "Molmo 的数据" | Allen AI 的 712K 稠密标注图像数据集；人工语音转写成稠密图注 |

## 延伸阅读

- [McKinzie et al. — MM1 (arXiv:2403.09611)](https://arxiv.org/abs/2403.09611)
- [Laurençon et al. — Idefics2 / What matters building VLMs (arXiv:2405.02246)](https://arxiv.org/abs/2405.02246)
- [Deitke et al. — Molmo and PixMo (arXiv:2409.17146)](https://arxiv.org/abs/2409.17146)
- [Tong et al. — Cambrian-1 (arXiv:2406.16860)](https://arxiv.org/abs/2406.16860)
- [Karamcheti et al. — Prismatic VLMs (arXiv:2402.07865)](https://arxiv.org/abs/2402.07865)
