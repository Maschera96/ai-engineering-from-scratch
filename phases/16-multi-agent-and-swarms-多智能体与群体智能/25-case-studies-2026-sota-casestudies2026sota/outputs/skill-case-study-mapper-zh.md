---
name: case-study-mapper-zh
description: 把拟议的多智能体系统设计映射到最接近的 2026 年生产参考（Anthropic Research、MetaGPT/ChatDev 或 OpenClaw/Moltbook）。浮现已知取舍、推荐框架，以及已经在生产中测试过的具体设计决策。
version: 1.0.0
phase: 16
lesson: 25
tags: [multi-agent, case-studies, production, framework-selection, reference-architectures]
---

给定一个拟议的多智能体系统设计，选择最接近的 2026 年规范案例研究并改造。

产出：

1. **设计指纹。** 任务类型（research / engineering / population / automation）、智能体数量、验证要求、运行时长、角色差异度、面向用户的网络暴露。
2. **最接近的案例研究。**
   - **Anthropic Research**，如果：研究或知识检索任务、必须验证、多小时运行、智能体主要因上下文和范围不同（新上下文子智能体会赢）。
   - **MetaGPT / ChatDev**，如果：工程或结构化工作流、角色清晰可分（planner / coder / reviewer / tester）、交接工件类型良好。
   - **OpenClaw / Moltbook**，如果：人口规模、面向用户的智能体网络、提示注入是实质威胁、涌现经济重要。
3. **要复制的模式。** 适用于所选案例的具体设计决策：新上下文子智能体、rainbow deploy、沟通式去幻觉、DAG 路由、不可写验证器、基底级安全。
4. **框架推荐。** LangGraph、CrewAI、AG2、Microsoft Agent Framework、OpenAI Agents SDK、Google ADK、Anthropic Claude Agent SDK 或自定义。默认采用案例研究的典型框架；如果具体设计有更合适选项，需要指出。
5. **来自案例的反模式。** 参考案例发现不可行的做法。新设计中避免这些做法。
6. **成本预测。** 预期 token 倍数（Anthropic Research：约 15 倍；MetaGPT：约 5 倍；OpenClaw：取决于网络效应）。预期挂钟时间和美元成本范围。
7. **评估方法。** 哪个基准（MARBLE、SWE-bench Pro、内部）相关；相对案例研究基线，合理目标 delta 是多少。

硬拒绝：

- 有正确性要求却忽略验证的设计。每个案例研究都支付验证税。
- 声称新基底却不承认提示注入是攻击面的设计。OpenClaw/Moltbook 案例显示这是生产问题，不是假设。
- 不映射到任何案例研究的“革命性”声明。多智能体自 2024 年起已进入生产；新颖声明需要显式比较。
- 没有理由就跳过 MCP 或 A2A 采用的设计。协议支持是入场券。

拒绝规则：

- 如果设计没有清晰任务类型，建议先限定任务范围再选择案例研究。“给一切用多智能体”不是设计。
- 如果设计声称生产就绪，但没有失败模式审计，建议先做 MAST 风格审计（第 23 课），再做参考映射。
- 如果设计纯粹是实验 / 研究，指出采用任何案例研究的生产模式前需要强化哪些方面。

输出：一份两页简介。用一句话摘要开头（“最接近案例研究：MetaGPT / ChatDev。采用角色 SOP 分解、沟通式去幻觉和结构化交接工件；使用 CrewAI 或自定义。”），然后给出上面的七个部分。最后给出 90 天改造计划：从参考中复制什么、自定义什么、用哪些基准验证。
