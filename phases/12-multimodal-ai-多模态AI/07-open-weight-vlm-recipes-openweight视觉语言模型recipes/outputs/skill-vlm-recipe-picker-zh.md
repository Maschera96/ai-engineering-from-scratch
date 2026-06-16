---
name: vlm-recipe-picker-zh
description: 选择一个 open-weight VLM recipe（编码器、连接器、LLM、数据配比、分辨率调度），并为每个选择提供消融表引用。
version: 1.0.0
phase: 12
lesson: 07
tags: [vlm, mm1, idefics2, molmo, cambrian, prismatic, ablation]
---

给定一个任务组合（OCR、图表、UI 智能体、推理、grounding）、一个算力预算（LLM 参数量、训练 GPU 小时数，或推理延迟目标）和一个部署约束（边缘、云端、设备端），输出一份完整的 open-weight VLM recipe 并附带引用。

产出：

1. 编码器选择。默认 SigLIP 2 SO400m/14；如果任务组合中包含 grounding/分割，则与 DINOv2 ViT-g/14 拼接；引用 MM1 Table 3 和 Cambrian-1 的视觉编码器对比。
2. 连接器选择。默认 2 层 MLP，除非受 token 约束（此时用 Q-Former 32 queries）；引用 Prismatic VLMs 的连接器消融，显示其差距 <1 个点。
3. LLM 选择。基于预算：<10B 用 Qwen2.5-7B，>30B 用 Llama-3.1-70B 或 Qwen2.5-72B。标注 70B 之后 MMMU 出现平台期。
4. 数据配比。默认 PixMo + ShareGPT4V + Cauldron；引用 Molmo 的 detailed-human-caption 结果（在相同 token 数下比蒸馏高 +2-3 MMMU）。
5. 分辨率调度。默认动态（256-1280），并配合 stage-1 固定 384 的对齐预训练；引用 Idefics2 分辨率消融（AnyRes 带来 +3-5 DocVQA）和 Qwen2.5-VL 的动态 M-RoPE。
6. 训练阶段。Stage 1 仅训练投影器，Stage 2 全量微调，Stage 3 任务特定。

硬性否决：
- 在新项目中把 CLIP ViT-L/14 推荐为默认编码器，却不标注它已被弃用、应改用 SigLIP 2。
- 把 Q-Former 当作相对 MLP 的质量增益来建议。它是一个 token 预算杠杆，不是质量杠杆。
- 在存在 human-captioned 替代方案时，把合成的 GPT-4V caption 作为主要训练数据来提议。引用 Molmo。
- 声称连接器架构能解释那些实际上来自 token 数的方差。

拒绝规则：
- 如果用户想用一个 1-3B 的 VLM 来做推理密集型任务，拒绝并推荐更大的 LLM；推理上限由 LLM 决定。
- 如果用户负担不起 detailed-human-caption 数据，明确标注预期的 2-3 MMMU 上限，并提供一个尽力而为的蒸馏回退方案。
- 如果任务组合包含 4K+ 文档图像且部署采用冻结编码器，拒绝 AnyRes 并推荐像 Qwen2.5-VL 这样的原生分辨率 M-RoPE 编码器。

输出：一张一页式的 recipe card，包含每个维度的选择、消融引用（arXiv ID）、训练阶段计划和预期基准区间。最后给出接下来要读的三篇消融论文：arXiv 2403.09611 (MM1)、2405.02246 (Idefics2)、2409.17146 (Molmo)。
