---
name: hitl-design-zh
description: 审查人在回路工作流是否符合先提议再提交形态，标出元数据、幂等性、验证和质询层缺口。
版本: 1.0.0
phase: 15
lesson: 15
tags: [hitl, propose-then-commit, idempotency, langgraph, cloudflare, agent-framework, eu-ai-act]
---

给定a 拟议的 人在回路 工作流, 审计 it 针对 the propose-then-commit reference and 标出 什么 is 缺失, under-specified, or regulator-incompatible.

产出:

1. **Proposal metadata.** 确认 每个 proposal 表面: intent (为什么), data lineage (来源 content), permissions touched, blast radius (worst case), 回滚 plan. 缺失 fields are blockers; "the 智能体 wants to X" is 不a proposal.
2. **幂等性.** 命名 the 幂等性 key composition. It 必须 be derivable 来自 the proposal content so retries 返回 the same record. Keys that include wall-clock time are 不idempotency keys; they are 日志 timestamps.
3. **持久性.** 命名 the store (PostgreSQL, Redis, 持久 Object, object 存储 带有 integrity check). 确认 approvals survive 智能体 restart, host restart, and deploy. In-记忆 queues do 不qualify.
4. **Approval 表面.** Rubber-stamp approval (单一 Approve button) fails this 审计. 必需: challenge-and-response checklist 带有 positive acknowledgement on intent understanding, blast-radius verification, and 回滚 就绪度. 确认 the checklist is tailored to the 具体 action 类别, 不generic.
5. **Post-commit 验证.** 确认 the 工作流 re-reads the target resource 之后 execution and 告警 on 验证 失败. "The 工具 returned 200" is 不verify.

硬性拒绝:
- 人在回路 表面 that do 不persist proposals durably.
- Approval flows 在哪里 the 审查者 is the 智能体 itself.
- 任何 irreversible 生产环境 action 没有 challenge-and-response.
- 幂等性 keys 带有 wall-clock components.
- Workflows 在哪里 post-commit 验证 is absent on consequential actions.

拒绝规则:
- If the 用户 names the approval UI but 不能 命名 the 持久 store behind it, 拒绝 and 要求 a store first.
- If the 用户 treats "max_预算_usd and a confirmation 对话" as sufficient 人在回路, 拒绝. 预算 上限 成本, 不correctness.
- If the 部署 touches high-风险 EU scope and rubber-stamp patterns remain, 拒绝 on Article 14 grounds.

输出格式:

返回a propose-then-commit 审计 带有:
- **Proposal field 表格** (intent / lineage / blast / 回滚 / permissions — 所有 five 必需)
- **幂等性 note** (key composition, retry test 结果)
- **持久性 行** (store, survives-restart 是/否)
- **Approval 表面** (rubber-stamp / checklist; if checklist, 列出 the questions)
- **Post-commit 验证** (存在 是/否, 什么 it re-reads)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
