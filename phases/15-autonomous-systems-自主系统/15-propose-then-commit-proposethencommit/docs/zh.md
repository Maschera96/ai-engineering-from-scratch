# 人在回路: Propose-Then-Commit

> The 2026 consensus on 人在回路 is 具体. It is 不"the 智能体 asks, the 用户 clicks Approve." It is propose-then-commit: the 拟议的 action is persisted to a 持久 store 带有 an 幂等性 key; surfaced to a 审查者 带有 intent, data lineage, permissions touched, blast radius, and a 回滚 plan; committed 只 之后 positive acknowledgement; verified 之后 execution to 确认 the side effect actually happened. LangGraph's `interrupt()` plus PostgreSQL 检查点ing, Microsoft Agent Framework's `RequestInfoEvent`, and Cloudflare's `waitForApproval()` 所有 implement the same shape. The canonical 失败 mode is the rubber-stamp approval: "Approve?" is clicked 没有 审查. The documented 缓解措施 is challenge-and-response 带有 an 明确 checklist.

**Type:** Learn
**Languages:** Python (stdlib, propose-then-commit 说明 machine 带有 幂等性)
**Prerequisites:** Phase 15 · 12 (持久 execution), Phase 15 · 14 (Tripwires)
**Time:** ~60 minutes

## 问题

一个agent takes an action. The 用户 has to decide: approve or not. If the decision is instant, it is probably 不a 审查. If the decision is structured, it is slow but trustworthy. The engineering question is 如何 to make a structured 审查 the path of least resistance.

这个2023-era 人在回路 pattern was a synchronous 提示词: "Agent wants to send email to X 带有 body Y — approve?" The 用户 clicks Approve. Everyone feels the 系统 is safe. In practice this 表面 is heavily rubber-stamped: users approve fast, approvals predict little, and when the 智能体 goes wrong, the 审计轨迹 shows a long history of approvals the 用户 不能 recall.

这个2026 pattern — propose-then-commit — moves 人在回路 onto a 持久 substrate, attaches structured metadata, and 要求 positive commit. 每个 managed 智能体 SDK ships a 版本: LangGraph `interrupt()`, Microsoft Agent Framework `RequestInfoEvent`, Cloudflare `waitForApproval()`. The API names differ; the shape does not.

## 概念

### 这个propose-then-commit 说明 machine

1. **Propose.** Agent produces a 拟议的 action. Persisted to a 持久 store (PostgreSQL, Redis, 持久 Object). Includes:
   - intent (为什么 is the 智能体 doing this)
   - data lineage (什么 来源 led to this proposal)
   - permissions touched (哪个 scopes / 文件 / endpoints)
   - blast radius (什么 is the worst case)
   - 回滚 plan (if committed, 如何 do we undo it)
   - 幂等性 key (unique per proposal; resubmission returns the same record)
2. **Surface.** Reviewer sees the proposal 带有 所有 metadata. The 审查者 is a person (不the 智能体 reviewing itself).
3. **Commit.** Positive acknowledgement. The action executes.
4. **验证.** 之后 execution, the side effect is 阅读 back and 已确认. If the 验证 step fails, the 系统 is in a known bad 说明 and alerting engages.

### 这个idempotency key

没有an 幂等性 key, a retry 之后 a transient 失败 can double-execute an approved action. 具体 example: 用户 approves "transfer $100 来自 A to B." 网络 blips. 工作流 retries. The 用户 has approved once but the transfer executes twice. The 幂等性 key ties the approval to a 单一, unique side effect; the second execution is a no-op.

这is the same 幂等性 pattern Stripe and AWS APIs use. Reusing it for 智能体 approvals is 明确 in the Microsoft Agent Framework docs.

### 持久性: 为什么 approvals outlast 进程

这个approval waiting room is a piece of 说明 the 智能体 does 不own. The 工作流 is paused (Lesson 12). When the approval arrives, the 工作流 resumes 来自 exactly that point. This is 为什么 LangGraph pairs `interrupt()` 带有 PostgreSQL 检查点ing and 不just in-记忆 说明 — an approval two days later still finds the 工作流 intact.

### Rubber-stamp approvals and the challenge-and-response 缓解措施

这个default UI for 人在回路 ("Approve" / "Reject" buttons) produces fast approvals 带有 no genuine 审查. Documented 缓解措施: a challenge-and-response checklist that 要求 positive answers to 具体 questions 之前 the Approve button is enabled. 具体 shape:

- "Do you understand 什么 resource this touches? [ ]"
- "Have you verified the blast radius is acceptable? [ ]"
- "Do you have a 回滚 plan if this fails? [ ]"

Not bureaucracy for its own sake — a forcing function. The 审查者 人员 不能 tick the boxes either asks for clarification (escalation) or declines (safe 默认值). The Anthropic 智能体-安全 研究 explicitly cites checklist-driven 人在回路 as a 缓解措施 for rubber-stamp approval patterns.

### 内容 counts as consequential

Not 每个 action needs propose-then-commit. The 2026 guidance:

- **Consequential actions** (always 人在回路): irreversible writes, financial transactions, outbound communication, 生产环境 database changes, destructive 文件-系统 operations.
- **Reversible actions** (sometimes 人在回路): edits to local 文件, 预发布环境-env changes, reversible writes 带有 clear 回滚.
- **Reads and inspections** (never 人在回路): reading a 文件, listing resources, calling a 阅读-只 API.

### Post-action verification

"The commit ran" is 不the same as "the side effect happened." 网络-partition and race conditions can 生成 a 工作流 that thinks it succeeded while the 后端 did 不persist. The 验证 step re-reads the target resource 之后 commit to 确认. This is the same pattern as database transactions 带有 `RETURNING` clauses or AWS `GetObject` 之后 `PutObject`.

### EU AI Act Article 14

Article 14 mandates effective 人类 oversight for high-风险 AI systems in the EU. "Effective" is 不decorative. Regulatory language specifically excludes rubber-stamp patterns. Propose-then-commit 带有 challenge-and-response is the shape that survives Article 14 scrutiny in the Microsoft Agent Governance Toolkit compliance docs.

## 使用它

`code/main.py` implements a propose-then-commit 说明 machine in stdlib Python. 持久 store is a JSON 文件. 幂等性 key is a hash of (thread_id, action_signature). The driver simulates three cases: a clean approval flow, a retry 之后 transient 失败 (哪个 必须 不double-execute), and a rubber-stamp 默认值 versus a challenge-and-response flow.

## 交付它

`outputs/skill-hitl-design.md` reviews a 拟议的 人在回路 工作流 for propose-then-commit shape and flags 缺失 metadata, 幂等性, verification, or challenge-and-response 层.

## 练习

1. 运行 `code/main.py`. 确认 that a retry of an approved proposal uses the 持久 record and does 不re-execute. Now change the 幂等性 key to include a timestamp and show the retry double-executes.

2. Extend the proposal record 带有 a `rollback` field. Simulate an execution whose 验证 step fails. Show the 回滚 firing automatically.

3. 阅读 Microsoft Agent Framework's `RequestInfoEvent` docs. 识别 one metadata field the API includes that the toy engine is 缺失. Add it and 解释 什么 it protects 针对.

4. 设计 a challenge-and-response checklist for a 具体 action (e.g., "post to a public Twitter account"). 内容 three questions 必须 the 审查者 answer? 原因 those three?

5. Pick one case 在哪里 a synchronous "Approve?" 提示词 would be sufficient (no 持久 store needed). 解释 为什么, and 命名 the 风险 类别 you are accepting.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| Propose-then-commit | "Two-phase approval" | Persisted proposal + positive commit + 验证 |
| 幂等性 key | "Retry-safe 词元" | Unique per proposal; second execution no-ops |
| Data lineage | "位置 it came 来自" | 这个specific 来源 content that led to the proposal |
| Blast radius | "Worst case" | Scope of effect if the action goes wrong |
| Rubber-stamp | "Fast approval" | "Approve" clicked 没有 genuine 审查 |
| Challenge-and-response | "Forcing checklist" | Reviewer 必须 positively acknowledge 具体 questions |
| RequestInfoEvent | "MS Agent Framework primitive" | 持久 人在回路 request 带有 structured metadata |
| `interrupt()` / `waitForApproval()` | "Framework primitives" | LangGraph / Cloudflare equivalents of the same shape |

## 延伸阅读

- [Microsoft Agent Framework — Human in the loop](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop) — `RequestInfoEvent`, 持久 approvals.
- [Cloudflare Agents — Human in the loop](https://developers.cloudflare.com/agents/concepts/human-in-the-loop/) — `waitForApproval()` and 持久 Objects.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 人在回路 as a 缓解措施 for long-horizon 风险.
- [EU AI Act — Article 14: Human oversight](https://artificialintelligenceact.eu/article/14/) — regulatory baseline for high-风险 systems.
- [Anthropic — Claude's Constitution (January 2026)](https://www.anthropic.com/news/claudes-constitution) — 宪法式 framing around oversight.
