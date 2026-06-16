---
name: vla-action-format-picker-zh
description: 为机器人任务选择动作格式（discrete bin、FAST、flow-matching、dual-system）和 VLA 家族（RT-2、OpenVLA、π0、GR00T）。
version: 1.0.0
phase: 12
lesson: 21
tags: [vla, rt-2, openvla, pi0, groot, action-tokenization]
---

给定一个机器人任务（操作、导航、全身人形）、DOF 数量、控制频率要求和算力约束，选择一种动作格式和一个 VLA 家族。

产出：

1. 动作格式。简单单臂任务用 discrete-bin，对速度敏感的轨迹用 FAST，平滑连续控制用 flow-matching，人形用 dual-system。
2. VLA 家族选择。RT-2（闭源）、OpenVLA（开源 7B）、π0（开源 flow）、GR00T N1（开源 dual-system 人形）。
3. 控制频率可行性。让格式吞吐与所需控制 Hz 匹配。在 7B 模型上 discrete bin 无法做到 >10 Hz。
4. 训练数据配比。Co-fine-tune 比例（web VQA : robot）。从 0.5:1 起步，按任务调整。
5. 微调方案。在约 500-1000 条任务演示上做 LoRA；约 10k 条演示时做全量微调。
6. 安全闸门。VLA 之外所需的控制层检查。

硬性拒绝：
- 在没有安全层规格的情况下推荐 VLA。务必包含关节限位、速度裁剪。
- 声称 discrete-bin 分词足以胜任 30 Hz 控制。它做不到。
- 在没有充分平滑约束的情况下提出 flow-matching。分布外动作仍会出现。

拒绝规则：
- 如果在 <=7B 模型上使用 discrete-bin 格式而控制频率要求 >50 Hz，拒绝；推荐 π0 或专用的 head。
- 如果机器人 DOF >30（人形），拒绝单阶段架构；要求 dual-system（GR00T）。
- 如果预算无法负担 Open X-Embodiment 规模的预训练，拒绝从零训练 VLA；推荐微调 OpenVLA。

输出：一页式方案，包含动作格式、VLA 选择、控制频率检查、co-fine-tune 配比、安全闸门。结尾给出 arXiv 2307.15818（RT-2）、2406.09246（OpenVLA）、2410.24164（π0）、2503.14734（GR00T）。
