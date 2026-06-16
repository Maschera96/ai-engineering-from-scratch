---
name: ecosystem-blueprint-zh
description: 在给定产品需求的前提下，产出完整的 Phase 13 生态架构；命名各类原语、安全态势、遥测以及打包方式。
version: 1.0.0
phase: 13
lesson: 22
tags: [mcp, capstone, ecosystem, architecture, a2a, otel]
---

在给定产品需求（研究、摘要、自动化，或任何由智能体驱动的工作流）的前提下，产出完整架构。

产出内容：

1. MCP 原语。需要哪些工具、资源、提示和任务。是否有 `ui://` 应用？是否有异步任务？
2. 安全态势。OAuth 2.1 作用域集合、网关 RBAC 矩阵、固定哈希清单、Rule of Two 审计。
3. A2A 协作。识别任何子智能体调用。定义它们的 Agent Card。
4. 遥测。OTel GenAI span 层级结构。导出器与后端选择。
5. 打包。AGENTS.md、SKILL.md，以及部署面（Docker Compose、K8s）。
6. 映射到 Phase 13 课程。每个设计选择可追溯到哪一节课。

硬性拒绝：
- 任何在单轮交互中同时组合不可信输入、敏感数据和有后果操作的架构（Rule of Two）。
- 任何在 MCP 与 A2A 跳转之间不传播 trace 的架构。
- 任何在 LLM 层没有至少一个回退提供方的架构。

拒绝规则：
- 如果产品需求用一次直接的 LLM 调用就能更好地满足，则拒绝搭建完整生态。
- 如果团队没有 SRE 来维护网关，推荐使用托管网关（Cloudflare MCP Portals、Portkey）。
- 如果架构涉及支付，将 AP2 标记为存在漂移风险的 A2A 扩展，并建议单独签核。

输出：一页式蓝图，包含各类原语、安全态势、A2A 跳转、遥测方案、打包以及课程映射。最后用一句话指出该部署面临的单一最难运维风险。
