---
name: societal-risk-review-zh
description: 用 CAIS 四风险框架和 CAISI / SB-53 监管语境审查部署的社会规模风险姿态。
版本: 1.0.0
phase: 15
lesson: 22
tags: [cais, caisi, four-risk-framework, organizational-risk, sb-53, societal-risk]
---

给定a 拟议的 or operating AI 部署, 生成 a societal-scale-风险 审查 that tags the 部署 针对 the CAIS four-风险 framework, inventories organizational-风险 sub-levers, and names the regulatory 表面.

产出:

1. **Four-风险 tagging.** For each of the four 类别 (malicious use, AI races, organizational 风险, rogue AIs), 说明 是否 the 部署 touches it and 如何. A 部署 can touch multiple 类别; "does 不apply" 必须 be justified in 一句话.
2. **Organizational-风险 inventory.** 分数 the 部署 针对 the four sub-levers: 安全 culture, 审计 rigor, multi-layered 防御, information security. 任何 lever scored "缺失" is a flagged 缺口.
3. **Regulatory 表面.** 命名 the applicable regulatory frameworks: EU AI Act (if in EU or serving EU users), California SB-53 (if signed and applicable), CAISI voluntary agreements (if the lab has signed one). Compliance is a 部署 gate, 不a 部署 nice-to-have.
4. **外部-评估 posture.** 命名 the 外部 评估s the 部署 or its base 模型 has undergone (METR, CAISI, Apollo, Gray Swan, etc.). No 外部 评估 is a flagged 缺口 for long-horizon 自主 部署s.
5. **Structural-force exposure.** 估计 如何 much competitive-部署 pressure the organization is under and 如何 that trades 针对 the organizational-风险 levers. Teams under heavy race pressure de-prioritize 审计 first; this is the CAIS finding.

硬性拒绝:
- Deployments touching harmful-能力 类别 没有 a 硬编码-禁令 层 (Lesson 17).
- Deployments in competitive-race conditions 带有 no 独立 审计.
- Long-horizon 自主 部署s 带有 no 外部 能力 评估.
- EU 部署s 带有 no Article 14 人在回路 (Lesson 15).
- California 部署s 带有 no incident-reporting 进程 if SB-53 is signed.

拒绝规则:
- If the 用户 不能 命名 the 外部 评估器 for the base 模型, 拒绝 and 要求 identification first. Self-评估 alone is insufficient.
- If the 用户 treats "we have a scaling 政策" as compliance 带有 catastrophic-风险 regulation, 拒绝 and 要求 具体 regulatory-表面 mapping.
- If the 用户 proposes deploying under race pressure 没有 审计, 拒绝 and 命名 the CAIS finding on organizational 风险.

输出格式:

返回a societal-风险 审查 带有:
- **Four-风险 row 表格** (类别, touched 是/否, nature)
- **Organizational-风险 scorecard** (安全 culture / 审计 / 防御 / infosec)
- **Regulatory 表面** (applicable frameworks 带有 compliance status)
- **外部-评估 posture** (评估器, scope, 节奏)
- **Structural-force exposure** (low / medium / high 带有 理由)
- **部署 就绪度** (生产环境 / 预发布环境 / 仅研究)
