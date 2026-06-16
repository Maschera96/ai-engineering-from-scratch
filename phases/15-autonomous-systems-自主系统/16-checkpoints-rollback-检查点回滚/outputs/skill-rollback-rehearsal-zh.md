---
name: rollback-rehearsal-zh
description: 为长周期自主工作流设计回滚演练测试，并审计检查点后端是否满足审计要求。
版本: 1.0.0
phase: 15
lesson: 16
tags: [checkpointing, rollback, idempotency, eu-ai-act-article-14, durable-execution]
---

给定a 拟议的 long-horizon 自主 工作流, 设计 a 回滚-rehearsal test that proves the 幂等性 + 前置条件 + 验证 + 回滚 stack actually works end-to-end, and 审计 the 检查点 后端 for regulator-就绪度.

产出:

1. **Rehearsal script.** 具体 test that (a) starts the 工作流, (b) crashes it mid-commit, (c) resumes, (d) asserts the action fires exactly once, (e) injects a 验证 失败, (f) asserts the 回滚 fires and 说明 is restored. No 生产环境 工作流 应该 运行 没有 this test having passed at least once.
2. **幂等性 审计.** 确认 the 幂等性 key is derived 来自 proposal content (Lesson 15) and commit logic uses 明确 execution states (`pending` -> `executing` -> `committed`/`failed`). Reserve/lock by 幂等性 key 之前 the side effect, and mark `committed` 只 之后 the side effect has been verified.
3. **前置条件 inventory.** 列出 每个 前置条件 the 工作流 必须 re-check at commit time. Time-of-check vs time-of-use 缺口 are the most common 生产环境 bug; the 前置条件 必须 be evaluated at commit, 不at propose.
4. **验证 inventory.** For 每个 consequential action, 命名 the 具体 阅读 that confirms the side effect happened. "Returned 200" is 不acceptable.
5. **回滚 inventory.** For 每个 consequential action, classify the 回滚 as in-band, compensating transaction, or out-of-band 告警. No-op 回滚s ("we 不能 undo this") 必须 be 具名 explicitly in the proposal (Lesson 15 metadata).

硬性拒绝:
- Workflows 带有 no rehearsed 回滚.
- 检查点 backends that lose data on deploy.
- Commit paths 在哪里 status is written 之后 execution, 不before.
- "Verified" states that 只 check the 返回 code of the 工具 call.
- 前置条件 checks that 运行 只 at propose time, 不commit time.

拒绝规则:
- If the 用户 has 不run the rehearsal script at least once in 预发布环境, 拒绝 生产环境 rollout.
- If the 用户 不能 生成 the 检查点 store 模式, 拒绝 and 要求 模式 documentation first. Regulators want 可查询 说明.
- If the 工作流 depends on an in-记忆 检查点 (no persistence), 拒绝.

输出格式:

返回a rehearsal plan 带有:
- **Test script outline** (steps 带有 assertions)
- **幂等性 表格** (key composition, status-编写 order)
- **前置条件 表格** (check, when evaluated, consequence)
- **验证 表格** (action, 阅读 that confirms)
- **回滚 表格** (action, type, target 说明)
- **后端 attestation** (store, 可跨部署保留 是/否, query-ready 是/否)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
