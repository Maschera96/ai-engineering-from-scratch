---
name: onevision-budget-planner-zh
description: 为目标产品组合在单图、多图和视频场景间分配 LLaVA-OneVision 风格的统一视觉 token 预算。
version: 1.0.0
phase: 12
lesson: 08
tags: [llava-onevision, token-budget, curriculum, multi-image, video]
---

给定一个产品的预期任务分布——单图、多图和视频请求的占比——以及每样本的视觉 token 预算，输出一份按场景划分的分配方案和一套训练课程（curriculum）。

产出：

1. 按场景的配置。单图：AnyRes 瓦片数 + 缩略图 + 池化因子；多图：每样本图像数 + 每张图像的池化；视频：帧数 + 每帧池化。
2. token 预算平衡。每个场景的总 token 数应落在目标预算的 ±30% 以内；标记任何低于目标 70%（token 不足）或高于 130%（上下文风险）的场景。
3. 课程方案。三个阶段（SI → OV → TT），并附数据权重。对于 TT 阶段，使用用户的产品组合。
4. 预期涌现能力。根据用户的产品组合，预测哪些 LLaVA-OneVision 风格的涌现能力可能出现（多摄像头、set-of-mark、screenshot-agent，或产品特定的变体）。
5. 训练数据量级估算。在 7B 基座 LLM 的前提下，估算每个阶段所需的大致 token / 图像 / 帧数，并引用 OneVision-1.5 的数据规模。

硬性拒绝项：
- 提出把视频或多图排在单图之前的阶段顺序。OneVision 表明这会损失 2-4 个 MMMU 分数。
- 当产品中 80% 是单图时却把全部预算分给视频。这是浪费，不是平衡。
- 假设 AnyRes-16（4x4 网格）能在 4k token 预算内、且不做激进池化的情况下容纳。它做不到。

拒绝规则：
- 如果每样本的 token 预算低于 1024，对多图或视频用例予以拒绝——低于这个下限，场景会崩溃。
- 如果用户想要在完整 729-token 分辨率下使用 5 帧及以上的视频，予以拒绝；建议采用 3x 池化或减少帧数。
- 如果产品分布完全没有单图，予以拒绝，并改为推荐 Qwen2.5-VL 风格的 M-RoPE——OneVision 的课程假设单图作为感知基础。

输出：一份一页的方案，包含按场景的 token 配置、课程阶段权重、涌现能力预测以及数据规模估算。结尾给出指向 arXiv 2408.03326（OneVision）和 arXiv 2509.23661（OneVision-1.5 完全开放）的链接。
