---
name: multimodal-agent-designer-zh
description: 设计一个多模态智能体（计算机使用、GUI 接地、网页或移动端），包含动作模式、记忆策略和基准评估计划。
version: 1.0.0
phase: 12
lesson: 25
tags: [multimodal-agents, computer-use, gui-grounding, visualwebarena, agentvista]
---

给定一份计算机使用产品规格（领域、动作集、评估目标），设计智能体循环、记忆策略、接地模式和评估。

产出：

1. 动作模式。受支持动作的 JSON 定义（click、type、scroll、drag、select、navigate、done，以及任何视觉工具）。
2. 输入模式。仅截图、accessibility-tree 或混合模式。浏览器默认混合模式；没有可访问性钩子的桌面应用使用仅截图模式。
3. 模型选型。Qwen2.5-VL-72B（开源）、Claude Opus 4.7 computer-use（闭源，强）、GPT-5（闭源，更强）。依据基准和成本进行论证。
4. 记忆策略。每 5 步进行一次摘要链 + 实时保留最近 2 张截图；超长工作流仅记录日志。
5. 错误恢复。动作失败时，通过 element_desc 语义提示重新接地；最多重试 2 次；回退到重新规划。
6. 评估计划。用 ScreenSpot-Pro 评估接地，用 VisualWebArena 评估端到端，用 AgentVista 评估困难的多步工作流。预期分数层级。

硬性拒绝：
- 使用自由文本动作输出。始终采用具有显式模式的 JSON 结构化输出。
- 宣称开源 7B 模型在 AgentVista 上能匹敌前沿模型。差距为 10-20 分。
- 跨截图依赖坐标记忆。坐标在不同捕获之间会漂移。

拒绝规则：
- 如果产品需要 >50 步的工作流，拒绝单智能体循环，建议采用分层的规划器 + 执行器拆分。
- 如果产品运行在没有可访问性钩子的受监管平台上，标记仅截图模式的可靠性局限，并提议进行重度验证。
- 如果任务类别超出训练分布（专业工业软件），拒绝开箱即用方案，并提议在领域截图上进行微调。

输出：一页智能体设计，包含动作模式、输入模式、模型选型、记忆、恢复、评估。以 arXiv 2401.10935（SeeClick）、2401.13649（VisualWebArena）、2602.23166（AgentVista）结尾。
