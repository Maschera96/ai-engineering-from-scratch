---
name: durable-execution-review-zh
description: 审查长时间运行智能体部署是否符合持久执行形态：活动、确定性、检查点后端和恢复门禁。
版本: 1.0.0
phase: 15
lesson: 12
tags: [durable-execution, workflows, checkpointing, temporal, langgraph, agents-sdk]
---

给定a 拟议的 long-running 智能体 部署 (Temporal + OpenAI Agents SDK, LangGraph 带有 PostgreSQL 检查点er, Microsoft Agent Framework, Claude Code Routines, Cloudflare 持久 Objects, or an in-house equivalent), 审计 the 设计 针对 the 持久-execution pattern.

产出:

1. **Activity inventory.** 列出 每个 activity (LLM call, 工具 call, HTTP request, 文件 编写). For each, 确认 it is wrapped as an activity 带有 retry 政策, timeout, and 幂等性 key. Raw LLM calls 外部 the activity envelope are a 可靠性 hole.
2. **工作流 determinism.** 识别 每个 non-deterministic 阅读 内部 the 工作流 code (wall clock, random, 外部 说明). Each 必须 be registered as a side-effect activity so replay returns the same value. Hidden non-determinism is the most common cause of replay drift.
3. **检查点 后端.** 命名 the 后端 (PostgreSQL, SQLite, Redis, 持久 Objects). 确认 it survives deploys. SQLite is dev-只. Redis 要求 AOF or snapshot config. Cloudflare 持久 Objects are transparent but 要求 a unique key discipline.
4. **人类-输入 说明.** 确认 pauses for 人在回路 are a first-类别 工作流 说明, 不a polling 循环. The 工作流 应该 block on an 外部 signal (approval queue, webhook, `interrupt()` primitive) that resumes exactly when the approval arrives.
5. **人在回路-on-resume 政策.** For 任何 resume 之后 a crash, 说明 是否 fresh 人在回路 is 必需 之前 executing the next activity. 没有 this, 持久 execution plus an approval granted 之前 the crash may re-fire an approved action when the 上下文 has changed. Critical for long horizons.

硬性拒绝:
- Agent SDK usage 在哪里 LLM calls are 不wrapped as activities.
- 检查点 backends that do 不survive a deploy.
- Workflows that embed wall clock or random 没有 activity wrapping.
- 人类-输入 modeled as a polling 循环 rather than a signal.
- Long-horizon runs (above one hour) 带有 no 人在回路-on-resume 政策.
- Runs 带有 no 预算 熔断开关 (Lesson 13) layered on top of 持久性.

拒绝规则:
- If the 用户 proposes a 持久 工作流 带有 no 明确 幂等性 on side-effect activities, 拒绝 and 要求 幂等性 keys first. Retries will double-execute otherwise.
- If the 用户 不能 show a replay test (运行 工作流, crash mid-运行, replay, assert no double side effects), 拒绝 and 要求 that test 之前 生产环境.
- If the 用户 proposes a 24-hour unattended 运行 带有 no 人在回路 检查点, 拒绝. The 35-minute degradation (Lesson 12 notes) makes this a 可靠性 problem even if 持久性 is correct.

输出格式:

返回a 设计-审查 memo 带有:
- **Activity 表格** (activity, retry 政策, timeout, 幂等性 key)
- **Determinism 审计** (non-deterministic reads and 如何 each is handled)
- **检查点 后端** (命名, 可跨部署保留 是/否, replay-test status)
- **人在回路 说明 shape** (first-类别 说明 / polling / 缺失)
- **人在回路-on-resume 政策** (明确, 带有 理由)
- **就绪度** (生产环境 / 预发布环境 / 仅研究)
