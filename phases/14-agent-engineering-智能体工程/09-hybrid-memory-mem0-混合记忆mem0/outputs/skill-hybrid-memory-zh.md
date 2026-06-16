---
name: hybrid-memory-zh
description: 生成一个 Mem0 形态的三存储记忆系统（向量 + KV + 图），包含融合打分器、作用域分类法和时间失效机制。
version: 1.0.0
phase: 14
lesson: 09
tags: [memory, mem0, vector, graph, kv, fusion, scope]
---

给定一个目标运行时、一个向量后端（Qdrant、pgvector、Chroma、sqlite-vec）、一个 KV 后端（Postgres、Redis、dict）和一个图后端（Neo4j、内存边），产出一个融合记忆系统。

产出：

1. 三个存储类，隐藏在一个 `add(text, user_id, session_id, scope, importance, tags)` 门面之后。写入时，抽取器将 `text` 分解为记录、KV 三元组和图三元组。任何存储都不可省略。
2. 融合打分器 `score = w_rel * relevance + w_imp * importance + w_rec * recency`。将三个权重全部暴露为配置。按产品调优，而非按调用调优。
3. 作用域分类法：`user`、`session`、`agent`。检索必须尊重作用域。一个用户的查询绝不能泄露另一个用户的记录。
4. 时间失效。矛盾会将旧的边/记录标记为失效；绝不删除。暴露 `search(query, as_of=timestamp)` 用于历史查询。
5. 一个抽取器接口。默认可以由 LLM 驱动；为测试提供一个确定性的正则回退方案。对每次 `add()` 的图边数量设上限，以防止爆炸。

硬性拒绝项：

- 将单存储记忆描述为“Mem0 形态”。仅向量、仅 KV、仅图的产品没问题，但它们不是混合记忆。不要误称它们。
- 没有按作用域设置权重或没有显式 `scope=` 过滤的跨作用域检索。作用域泄露是一起合规与隐私事件。
- 在矛盾时删除。应失效并打上时间戳。删除会掩盖 bug 并破坏审计。

拒绝规则：

- 如果用户要求“不做重要性加权”，拒绝。在一百万条记录上做扁平的相关性排序，是一场注定要发生的检索失败。
- 如果图后端没有冲突检测器，拒绝将由此产生的系统称为“Mem0 形态”。降级其名称。
- 如果产品涉及 PII（医疗、法律、HR），拒绝交付一个未经产品负责人审计的抽取器。

输出：每个存储一个文件，外加 `memory.py`（门面）、`config.py`（权重）、`README.md`（解释融合权重、作用域策略、抽取器契约和失效语义）。以“接下来读什么”收尾：如果智能体需要学习新技能，指向 Lesson 10；如果记忆操作需要 OTel span，指向 Lesson 23；如果检索需要处理不可信输入，指向 Lesson 27。
