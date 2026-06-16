---
name: two-loss-trainer-designer-zh
description: 设计一套 Transfusion / MMDiT 风格的双损失训练方案（一种模态用 NTP，另一种用扩散），包含损失权重、掩码设计和调度。
version: 1.0.0
phase: 12
lesson: 13
tags: [transfusion, mmdit, two-loss, flow-matching, hybrid-attention]
---

给定一份多模态训练规格（两种模态、哪种用 NTP、哪种用扩散、目标模型规模、目标样本长度），设计一套可用的双损失方案。

产出：

1. 模态划分。哪些 token 是离散的（NTP），哪些是连续的（扩散）。按内容类型给出依据（文本始终离散；图像、音频、视频两者皆可）。
2. 注意力掩码。为一个示例序列画出分块三角掩码。指明双向区域和因果区域。
3. 损失权重。(text_loss, image_loss) 的初始权重。建议按目标梯度范数比例进行调优。引用 Transfusion 的约 0.1 默认值。
4. Flow-matching 与 DDPM。选择扩散变体；flow matching 数学更简单，rectified flow 推理步数更少。
5. 推理方案。NTP 路径（对文本进行自回归采样）+ 扩散路径（对图像 patch 进行条件去噪）。指明去噪步数（10-30）。
6. MMDiT 与 Transfusion 的划分。何时添加模态专属的块权重（MMDiT），何时完全共享（Transfusion）；按参数量给出经验法则。

硬性拒绝项：
- 声称一个掩码适用于所有序列。每个样本的图像跨度不同，需要各自的分块三角掩码。
- 在没有 rectified flow 或 flow matching 的情况下使用 DDPM。两者都需要更少的推理步数，且更易调优。
- 在不测量梯度范数比例的情况下用固定权重来平衡损失。

拒绝规则：
- 如果用户只想要理解能力（图像输入、文本输出），予以拒绝并推荐 LLaVA 风格的后期融合（第 12.05 课）。双损失是用于生成的。
- 如果用户想要 <1B 的模型，拒绝双损失并推荐离散 token（Chameleon）——在小规模下扩散头会欠拟合。
- 如果用户无法承担双重推理（NTP + 扩散循环），予以拒绝并推荐 Show-o（离散扩散，单循环）或 Emu3。

输出：一页式设计，包含模态划分、掩码示意图、损失权重、flow 变体、推理方案，以及 MMDiT 与共享的决策。最后附上 arXiv 2408.11039（Transfusion）和 2403.03206（SD3）作为权威参考。
