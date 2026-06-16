---
name: resolution-budget-planner-zh
description: 为混合长宽比的 VLM 工作负载在 square-resize、AnyRes、M-RoPE 和 NaFlex 之间做选择，并输出每任务的 token 预算方案。
version: 1.0.0
phase: 12
lesson: 06
tags: [vlm, patch-n-pack, naflex, anyres, m-rope, token-budget]
---

给定一个工作负载——对 VLM 将看到的图像的描述（OCR 文档、图表、UI 截图、自然照片、视频帧）以及每个请求的总 token 预算——为每个图像类别选择一种分辨率策略，并产出可运行的配置。

产出：

1. 每个图像类别的策略。对每个声明的类别（OCR、图表、UI、照片、视频帧），从 {square-resize、AnyRes、M-RoPE、NaFlex} 中选择一个。用一句话说明理由，引用该任务对分辨率的敏感度。
2. 每张图像的 token 预算。包含 min_pixels、max_pixels（Qwen2.5-VL 风格），以及所选策略下预期的序列长度。如果任何单张图像超过 LLM 上下文的 40%，则标记出来。
3. 批量打包方案。如果请求是批量处理的，指定使用 `cu_seqlens`（FlashAttn varlen）、稠密的块对角掩码，还是非批量的单图像推理。注明当批次内长宽比相差超过 2 倍时，varlen 带来的 FLOP 节省。
4. 编码器推荐。混合工作负载用 SigLIP 2 NaFlex；agent UI 用 Qwen2.5-VL 原生方案；冻结编码器部署用 CLIP-336 + AnyRes；仅照片路径用 224 分辨率的原始 ViT。
5. 失败模式告警。所选配置下每张图像的 token 数；在 30 tok/s 预填充下的延迟成本；上下文填充百分比；在典型 OCR 基准上相对 square-resize 的预期精度差。

硬性拒绝：
- 在不引用用户会损失哪个基准分数的情况下，为 OCR 或图表任务推荐 square-resize。
- 提出一个产生的 token 数超过 LLM 上下文允许范围的策略。始终对照声明的上下文窗口来做预算。
- 把 AnyRes 当作万能答案——它的乘性分块开销可能在一张图像编码完成之前就超过 LLM 上下文。

拒绝规则：
- 如果用户声明的 token 预算低于每张图像 256 token，则除仅照片的语义任务外一律拒绝——在该预算下，再多的池化也无法挽回 OCR 精度。
- 如果用户想要稠密预测输出（分割、深度）但编码器中没有 ViT register token，则拒绝，并指向启用了 register 的 DINOv2 / SigLIP 2。
- 如果用户的 LLM 上下文小于 8k 而工作负载包含文档或截图，则拒绝，并推荐更大的上下文或采用 OCR 优先的流水线。

输出：一页式预算方案，包含每个类别的策略表、批量打包方案、编码器推荐和告警列表。在结尾给出相关的 arXiv 论文以便跟进——NaViT 为 2307.06304，SigLIP 2 / NaFlex 为 2502.14786，Qwen2.5-VL 为 2502.13923。
