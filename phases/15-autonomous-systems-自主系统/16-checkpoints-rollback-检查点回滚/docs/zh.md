# Checkpoints and 回滚

> 每个 graph-说明 transition persists. When a worker crashes, its lease expires and another worker picks up at the latest 检查点. Cloudflare 持久 Objects 暂停 说明 across hours or weeks. Propose-then-commit (Lesson 15) defines a 回滚 plan per action. Post-action verification closes the 循环. EU AI Act Article 14 makes effective 人类 oversight mandatory for high-风险 systems — in practice this means 检查点s 必须 be 可查询, 回滚s 必须 be rehearsed, and the 审计轨迹 必须 survive a deploy. The sharp 失败 mode: 没有 幂等性 keys and 前置条件 checks, a retry 之后 a transient 失败 can double-execute an already-approved action. Post-action verification is 什么 catches it.

**Type:** Learn
**Languages:** Python (stdlib, 检查点 and 回滚 说明 machine)
**Prerequisites:** Phase 15 · 12 (持久 execution), Phase 15 · 15 (Propose-then-commit)
**Time:** ~60 minutes

## 问题

持久 execution (Lesson 12) makes a crashed 智能体 resumable. Propose-then-commit (Lesson 15) makes an approved action auditable. This lesson joins them: 什么 happens when an approved action executes partially, crashes, and resumes? When does the 回滚 运行, and 针对 什么 说明?

Real systems wire this up differently:

- **LangGraph** 检查点s 每个 graph-说明 transition to PostgreSQL. On worker crash, the lease releases and another worker resumes at the latest 检查点. Workflows 暂停 on `interrupt()`, 哪个 itself persists.
- **Cloudflare 持久 Objects** 暂停 per-key 说明 across hours or weeks. Co-locate the computation 带有 the 存储 for the approved action.
- **Microsoft Agent Framework** exposes `Checkpoint` primitives in the 工作流 API; replay plus 幂等性 covers retries.

在every case, the combination that actually works is: 幂等性 key (prevents double-execute) + 前置条件 check (说明 is still 什么 we approved 针对) + post-action 验证 (the side effect actually happened) + 回滚 on 验证-fail.

## 概念

### 每个transition persists

一个graph-说明 transition is 任何 step that moves the 工作流 来自 one 具名 说明 to another. Naive implementations persist 只 at 具体 commit points; 生产环境 implementations persist 每个 transition. The 成本 (a few extra writes) is small relative to the 可靠性 gain (replay lands anywhere, lease recovery is precise).

### Lease recovery

当a worker crashes, the 工作流 is 不lost; the lease (a short-lived 声明 that this worker is executing this 运行) simply expires. Another worker picks up the latest 检查点 and resumes. The lease 机制 is 什么 lets 生产环境 systems survive rolling deploys 没有 losing in-flight work.

### 幂等性 plus 前置条件

幂等性 alone is 不enough. Consider: a 工作流 is approved to "transfer $100 来自 A to B when balance > $1000." The 工作流 is committed, crashes mid-execution, and resumes. If 只 the 幂等性 key is checked, and the execution resumes, the transfer runs once (correct). But consider that between crash and resume, A's balance drops to $500 via a different 工作流. The 幂等性 check still passes; the 前置条件 does not. 没有 a 前置条件 check, we ship an overdraft.

每个consequential action needs 两者:

- **幂等性 key**: prevents double-execute.
- **前置条件 check**: confirms the 说明 is still consistent 带有 什么 was approved.

### Post-action verification

"The 工具 returned 200" is 不verification. Real verification re-reads the target 说明 and confirms the side effect actually happened. Patterns:

- Database update: `UPDATE ... RETURNING *` then assert the returned row matches intended 说明.
- Email send: check sent-folder for the message ID 之后 submission.
- File 编写: 阅读 the 文件 back and hash it.
- API call: follow-up `GET` on the target resource.

如果verify fails, the 工作流 is in a known-bad 说明. 回滚 engages.

### 回滚 plans

每个consequential action in propose-then-commit (Lesson 15) carries a 回滚 plan. Types:

- **In-band 回滚**: reverse the side effect 直接 (`DELETE` 之后 `INSERT`, `Send-correction-email` 之后 send).
- **Compensating transaction**: a new action that neutralizes the original (standard SAGA pattern).
- **Out-of-band 回滚**: 告警 a 人类, 暂停 the 工作流, leave the bad 说明 for investigation.

No-op 回滚 ("we 不能 undo this") 必须 be 具名 in the proposal. Actions 带有 no 回滚 要求 stronger 人在回路 at commit time (Lesson 15 challenge-and-response).

### EU AI Act Article 14 operational reading

Article 14 要求 "effective 人类 oversight" for high-风险 systems. In operational terms, implementers 阅读 it as:

- Checkpoints are 可查询 by an auditor.
- Rollbacks are rehearsed (tested end-to-end at least once).
- The 审计轨迹 survives a deploy (检查点 后端 is 不ephemeral).
- Failed verifications are alerted on, 不silently logged.

一个工作流 that crashes mid-commit, resumes, and completes the side effect 没有 a 验证 + 回滚 pathway does 不survive the Article 14 test.

### 这个sharp 失败 mode: the double-execute

这个most common 生产环境 incident in this space:

1. Action approved, 幂等性 key k.
2. Commit starts, executes, returns 200.
3. 工作流 crashes 之前 persisting the "committed" status.
4. 工作流 resumes; sees "approved but 不committed"; re-executes.
5. Side effect fires twice.

缓解措施: persist an "in-flight" intent 之前 execution, execute 带有 an 幂等性 key, then mark "committed" only 之后 post-action verification succeeds. If the action fires and the status 编写 fails, you know to 验证 and (if necessary) re-fire. If the status 编写 succeeds and the action fails, you 验证 and fire exactly once via the recovery path.

## 使用它

`code/main.py` implements a 检查点ed 工作流 带有 幂等性, 前置条件, 验证, and 回滚. The driver simulates four scenarios: clean 运行, retry 之后 crash (幂等性 catches), 前置条件 fail (工作流 aborts 没有 firing), 验证 fail (回滚 fires).

## 交付它

`outputs/skill-rollback-rehearsal.md` designs a 回滚-rehearsal test for a 拟议的 工作流 and audits the 检查点 后端 for 审计-trail persistence.

## 练习

1. 运行 `code/main.py`. 验证 the four scenarios. For the crash-期间-commit case, 确认 the action fires exactly once across retries.

2. Modify the "mark as done first, then do it" pattern so the status 编写 fires 之后 the action. Rerun the crash scenario. Measure 如何 many duplicate actions fire.

3. 设计 a 回滚 plan for a 具体 生产环境 action (e.g., "post to a Slack 渠道"). Classify as in-band, compensating, or out-of-band. Justify the choice.

4. Take one 工作流 you know. 识别 每个 说明 transition. Mark each 带有 a 持久性 requirement (persist / do 不persist). Count the ones you are currently 不persisting.

5. Rehearsed-回滚 test: 设计 an end-to-end test that runs a real 工作流, crashes it, and confirms the 回滚 path fires. 内容 does the test assert?

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|---|---|---|
| 检查点 | "Save point" | 每个graph-说明 transition persists to a 持久 store |
| Lease | "Worker 声明" | Short-lived 声明 that a worker is executing a 运行; expires on crash |
| 前置条件 | "说明 gate" | Assertion that the 说明 is still consistent 带有 the approved action |
| Post-action 验证 | "Re-阅读 check" | 确认 the side effect actually happened in the target 系统 |
| In-band 回滚 | "Direct undo" | Reverse the side effect 带有 the inverse operation |
| Compensating transaction | "SAGA undo" | 一个new action that neutralizes the original |
| Mark-as-done-first | "Status 编写 order" | Persist the committed status 之前 returning 来自 commit |
| Article 14 | "EU AI Act 人类 oversight" | Operational: 可查询 检查点s, rehearsed 回滚s, auditable trail |

## 延伸阅读

- [Microsoft Agent Framework — Checkpointing and HITL](https://learn.microsoft.com/en-us/agent-framework/workflows/human-in-the-loop) — 检查点 primitives and lease recovery.
- [Cloudflare Agents — Human in the loop](https://developers.cloudflare.com/agents/concepts/human-in-the-loop/) — 持久 Objects as a 说明 substrate.
- [EU AI Act — Article 14: Human oversight](https://artificialintelligenceact.eu/article/14/) — regulatory baseline.
- [Anthropic — Measuring agent autonomy in practice](https://www.anthropic.com/research/measuring-agent-autonomy) — 可靠性 framing for long-horizon 工作流s.
- [Anthropic — Claude Code Agent SDK: agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop) — 工作流 shape for Claude Code Routines.
