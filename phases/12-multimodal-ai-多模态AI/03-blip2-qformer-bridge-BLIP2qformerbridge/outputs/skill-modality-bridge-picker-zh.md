---
name: modality-bridge-picker-zh
description: 在给定 token 预算、质量目标和训练算力的情况下，为某个 VLM 配置推荐使用 Q-Former、MLP projector 还是 Perceiver resampler。
version: 1.0.0
phase: 12
lesson: 03
tags: [blip2, qformer, vlm, modality-bridge, architecture]
---

给定视觉编码器每张图像的 token 数量、LLM 的上下文预算、每个 prompt 的目标图像数量以及训练算力预算，推荐应使用哪种模态桥接（modality bridge），并用参数量和 token 经济性加以论证。

产出：

1. Token 预算审计。报告视觉编码器输出的每张图像原始 token 数、经过每种桥接方案后每张图像的 token 数，以及在声明的每 prompt 图像数下所消耗的 LLM 上下文比例。
2. 桥接方案对比。针对 Q-Former（32 个 token，约 188M 参数）、MLP projector（全部 patch，约 20M 参数）和 Perceiver resampler（通过 N 层交叉注意力的 K 个可学习查询，可变）这三种方案，分别给出参数量、质量代理指标以及训练成本的大致估算。
3. 推荐方案。针对所述约束给出唯一最佳选择，并附一行论证。当约束相互矛盾时（高质量 + 紧张的 token 预算 + 低训练算力）要明确标记出来。
4. 两阶段训练追踪。如果选择 Q-Former，请概述第一阶段的 ITC + ITM + ITG 损失，以及第二阶段的 LM 损失。为每个阶段指定一个代表性数据集（COCO、LAION、Visual Genome）。
5. 消融实验清单。在锁定桥接方案之前，调用者应运行的五项实验（查询数量、两阶段 vs 单阶段、projector 深度、冻结策略、微调子集）。

硬性拒绝项：
- 任何忽略 token 预算的推荐。在 4k 上下文中、每张图像 576 个 token 的情况下，"用 MLP" 在 10 张图像时就会失败。
- 声称 Q-Former 严格优于 MLP。在上下文无限的单图像高质量任务中，MLP 胜出。
- 把 Perceiver resampler 当作等同于 Q-Former。Flamingo 在每个 LLM 层都应用它；BLIP-2 只应用一次。

拒绝规则：
- 如果调用者要求一个能处理视频的桥接方案，却没有指定多少帧以及以什么帧率，则拒绝——视频桥接与单图像桥接在规格上就不同，而不仅仅是规模差异。
- 如果范围内的 LLM 是与视觉塔从头一起训练的（early-fusion，Chameleon 风格），则拒绝——课程 12.11 单独覆盖该情形。
- 如果未说明训练算力，则拒绝并询问调用者是否负担得起 BLIP-2 的第二阶段（约数百 A100 小时）还是只能进行仅 projector 的训练。

输出：一页纸的桥接推荐，包含 token 计算、参数量、推荐架构、训练概要和消融实验清单。以一段"接下来读什么"作为结尾，指向课程 12.04（Flamingo）了解"处处交叉注意力"、课程 12.05（LLaVA）了解仅 MLP 方案，或课程 12.07（消融实验）了解数据与架构的权衡。
