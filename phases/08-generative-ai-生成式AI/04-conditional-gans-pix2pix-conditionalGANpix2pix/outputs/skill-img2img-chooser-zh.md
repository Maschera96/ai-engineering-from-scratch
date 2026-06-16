---
name: img2img-chooser-zh
description: 考虑到配对数据与未配对数据、领域特异性和延迟预算，选择一种图像到图像的方法。
version: 1.0.0
phase: 8
lesson: 04
tags: [pix2pix, img2img, conditional]
---

给定任务描述（源域、目标域、数据可用性 - paired/unpaired/N 样本、延迟预算、质量条），输出：

1. 方法。 Pix2Pix（配对，窄）、Pix2PixHD（配对、高分辨率）、CycleGAN（未配对）、SPADE（分段到图像）或 SD3 / Flux.1 上的 ControlNet 变体（通用、开放域）。
2. 训练数据规范。最小对数、分辨率、增强、许可注意事项。
3. 架构。 G（U-Net 深度、通道宽度）、D（PatchGAN 感受野、谱范数）、损失权重（adv、L1、VGG-感知）。
4. 推理延迟。在单个消费级 GPU（RTX 4090、M3 Max）上以 ms/image 为目标，进行分辨率权衡。
5. 评估。针对保留配对数据的 LPIPS、5k 样本的 FID、特定于任务的指标（seg 任务的 mIoU、超分辨率的 PSNR）、人类偏好。

当数据未配对时，拒绝推荐 Pix2Pix - 改为使用 CycleGAN 或 ControlNet。在没有增强/预训练建议的情况下，拒绝训练少于 500 对的配对模型。标记任何显示“任意文本提示”的请求 - 这些请求需要扩散 + ControlNet，而不是配对的 GAN。
