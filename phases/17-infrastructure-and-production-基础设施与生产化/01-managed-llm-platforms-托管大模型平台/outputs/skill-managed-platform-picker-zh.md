---
name: managed-platform-picker-zh
description: 选择一个托管大模型平台 (Bedrock, Azure OpenAI, Vertex AI) and a second for redundancy, 给定工作负载, SLA, 和 合规 要求 ： then produce a FinOps 埋点 计划.
version: 1.0.0
phase: 17
lesson: 01
tags: [bedrock, azure-openai, vertex-ai, ptu, finops, managed-platforms]
---

给定一个工作负载 画像 (必需的模型, 每月词元, TTFT SLA at P50/P99, 合规 约束, existing cloud footprint), produce 一个平台 建议.

产出：

1. 主要平台. Name 这个平台, 这个具体模型 it covers, and whether 按需 或 预置吞吐量单元 (PTUs) / 预置吞吐量 is appropriate given utilization. Cite 这个盈亏平衡 math (PTU at roughly 40-60% 持续利用率).
2. 次要平台. Name the two-提供商-minimum fallback. 说明理由 the pairing ： redundancy must cover 模型 重叠(Claude on Bedrock + GPT on Azure OpenAI is the common 组合) 和 区域 overlap.
3. FinOps 埋点. Specify what to enable on day one: Bedrock Application Inference 画像, Azure scopes + PTU reservations as 成本 objects, Vertex project-per-团队 + BigQuery Billing 导出. Name the attribution dimensions ： per-user, per-task, per-tenant.
4. SLA check. 比较 target TTFT P99 to published 基准 (Azure OpenAI PTU ≈ 50 ms P50; Bedrock 按需 ≈ 75 ms P50). If the SLA is tighter than 按需 can deliver, require PTU.
5. 合规 check. Verify BAA, SOC 2 Type II, HIPAA, EU 数据驻留 as 需要. Note that all three meet baseline but 保留策略 和 滥用监控 opt-out differ.
6. 迁移 路径. Name one reversible 步骤 这个团队 can take this week (e.g., deploy through AI 网关 abstracting 提供商; instrument attribution 头) and one longer-term 步骤 (PTU commitment; cross-区域 故障切换).

硬性拒绝：
- Recommending a single 平台 without a named fallback. 拒绝 和 坚持 on 双提供商最低要求.
- Picking PTU without a utilization estimate. 拒绝 和 请求 持续利用率 data.
- Ignoring Bedrock Application Inference 画像 when attribution is listed as a requirement ： they are the cleanest native 界面.

拒绝规则：
- If 这个工作负载 需要 Claude, Gemini, and GPT all as P0, name the three-平台 reality (Bedrock + Vertex + Azure OpenAI behind 一个网关) rather than 假装 one 平台 can serve all three.
- If the SLA is TTFT P99 < 100 ms and the expected budget cannot support PTU, 拒绝 to 承诺 the SLA ： 解释 这个按需 方差 ceiling.
- If 这个客户 asks to "use the cheapest 提供商," 拒绝 ： price is multi-dimensional (词元 rate + 专用容量 + attribution overhead + lock-in 成本).

输出： a one-page 决策 搭配 主要平台, 次要平台, PTU 对比 按需, 埋点 list, SLA/合规 verification, and two 迁移 steps. End with the single 指标 that will catch drift from the 计划 (持续利用率, PTU waste, or attribution coverage).
