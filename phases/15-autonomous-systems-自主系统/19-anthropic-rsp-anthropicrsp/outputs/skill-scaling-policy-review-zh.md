---
name: scaling-policy-review-zh
description: 按 RSP v3.0 参考形态审查前沿实验室扩展政策。
版本: 1.0.0
phase: 15
lesson: 19
tags: [rsp, scaling-policy, ai-rd-4, pause-commitment, saferai, governance]
---

给定a published or 拟议的 scaling 政策, 生成 a structured 审查 comparing it to the RSP v3.0 reference shape (AI R&D-4, affirmative case, two-tier 缓解措施, Frontier 安全 路线图, 风险 报告, 独立 审查).

产出:

1. **Two-tier inventory.** Separate commitments 进入 "lab-单边" and "行业范围 建议." Commitments in the 建议 tier are advocacy, 不promises. Count the 比例; a 政策 在哪里 most commitments live in the 建议 tier is a weak 政策.
2. **阈值.** 命名 每个 能力 阈值 and the 缓解措施 that triggers. 标出 阈值 that are 定性 在哪里 v2 had 定量. 标出 缺失 阈值 for 能力 the 政策 声明 to cover.
3. **暂停 承诺.** 确认 the 政策 names a 暂停 条款 (training stops, 部署 halts, or similar) at 具体 阈值. v3.0 removed this; 政策 that follow suit inherit the 回归.
4. **Standing artifacts.** 确认 the 政策 mandates standing Frontier 安全 路线图 and 风险 报告 documents 带有 declared 节奏. One-off artifacts published post-hoc do 不qualify.
5. **独立 审查.** 命名 the 外部 审查 机制. 内部-只 审查 (a "安全 Advisory Group" made of lab employees) does 不qualify as 独立 oversight.

硬性拒绝:
- 政策 带有 no 具名 能力 阈值.
- 政策 whose 缓解措施 所有 live in the industry-建议 tier.
- 政策 带有 no standing 路线图 / 风险 报告 artifacts.
- 政策 带有 no 独立 审查 机制.
- 政策 that 声明 to "learn 来自 real-world experience" 没有 stating 如何 the 政策 text updates and on 什么 节奏.

拒绝规则:
- If the 政策 document is marketing rather than governance (no 具体 commitments, no 阈值, no 节奏), 拒绝 to rate it as a scaling 政策.
- If the 用户 treats a 政策's existence as equivalent to compliance, 拒绝. A 政策 is a 承诺 device; compliance 要求 证据.
- If the 用户 cites an older 政策 版本 (e.g., 2023 Anthropic RSP) as current, 拒绝 and 要求 the current 版本.

输出格式:

返回a 政策 审查 带有:
- **Two-tier 比例** (单边 / 建议 / total count)
- **阈值 表格** (命名, type: 定量 / 定性, 触发器, 缓解措施)
- **暂停 承诺** (存在 是/否, 具体 条款)
- **Standing artifacts** (路线图 节奏, 风险 报告 节奏)
- **独立 审查** (机制, 审查者 identity, frequency)
- **摘要 评级** (strong / moderate / weak, justified)
