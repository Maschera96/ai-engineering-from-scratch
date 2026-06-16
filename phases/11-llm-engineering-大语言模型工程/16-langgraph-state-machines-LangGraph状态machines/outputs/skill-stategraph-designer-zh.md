---
name: stategraph-designer-zh
description: Turn an 智能体 任务 into a LangGraph StateGraph with named nodes, typed 状态, reducers, checkpointer, and human interrupts.
version: 1.0.0
phase: 11
lesson: 16
tags: [langgraph, stategraph, checkpointer, interrupt, time-travel, react-agent, human-in-the-loop]
---

给定the 智能体 任务 (user-facing goal, available 工具, expected turn count, side effects with 安全 blast radius, durability requirements, 目标 延迟 预算), 输出：

1. 节点 list. Name every discrete 步骤: the LLM thinker, each 工具 runner, every human review 步骤, any summarizer or critic, any retriever. Reject the design if any 节点 touches more than one concern; split it.
2. 状态 模式. 类型dDict (or Pydantic) fields with a reducer for every list. Always Annotated[list, add_messages] on the 消息 log. Hoist any task-specific list out of 消息 (a plan, a 预算 counter, a retrieved-docs list) so reducers stay correct under 并行 updates.
3. 边 map. Static edges where the next 步骤 is deterministic. 条件式 edges with a named 路由器 函数 only where the 模型 picks the next 步骤. Reject any 图 whose 路由器 函数 depends on a fresh LLM call you have not already made in a 先验 节点.
4. Interrupt placement. interrupt_before on every 节点 with an irreversible side effect (writes, deletes, payments, external API calls with 成本). interrupt_after on the 模型 节点 when 输出 验证 runs in a separate process. Reject interrupt_after on any side-effecting 节点; by then the side effect has happened.
5. Checkpointer. MemorySaver for tests only. Pick from PostgresSaver, SQLiteSaver, RedisSaver for any 环境 that must survive a restart. Confirm thread_id strategy (per-user, per-session, per-conversation) and the checkpoint TTL.

拒绝 ship a LangGraph without a checkpointer. No checkpointer means no resume, no time-travel, no human-in-the-loop replay. 拒绝 ship a 消息 field without add_messages; the second write overwrites the first silently and half the conversation disappears. Refuse a 图 whose every transition is a 条件式 边 routed by a planner LLM; that is AutoGen with extra 步骤 and burns 词元 per turn.

Example 输入: "Refund-handling 智能体 over Anthropic Claude with three 工具 (lookup_order, issue_refund, send_email), must pause for a human before any refund over 100 dollars, must resume after 服务器 restart, p95 延迟 预算 8 seconds."

Example 输出：
- Nodes: 智能体 (LLM call), lookup_tool, refund_tool, email_tool, human_review.
- 状态: 消息 with add_messages, order_context (overwrite), refund_amount (overwrite), reviewer_decision (overwrite).
- Edges: 智能体 to should_continue 路由器 with branches lookup_tool, refund_tool, email_tool, human_review, END. 工具 nodes go back to 智能体.
- Interrupts: interrupt_before on refund_tool when refund_amount > 100. No interrupt on lookup_tool or email_tool.
- Checkpointer: PostgresSaver with thread_id "用户:{user_id}:case:{case_id}" and 30-day TTL.
