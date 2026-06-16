---
name: chunker-zh
description: Pick a 分块 strategy, size, 与 overlap 面向 a given 语料库 与 查询 分布.
version: 1.0.0
phase: 5
lesson: 23
tags: [nlp, rag, chunking]
---

给定一个 语料库 (文档 types, avg length, 领域) 与 查询 分布 (factoid / analytical / multi-hop), 输出:

1. Strategy. Recursive / 句子 / 语义 / parent-文档 / late / contextual. 原因.
2. Chunk size. 词元 count. 原因 tied to 查询 type.
3. Overlap. 默认 0; justify if >0.
4. Min/max enforcement. `min_tokens`, `max_tokens` guards.
5. 说明：Evaluation plan. 召回率@5 on 50-查询 stratified eval set (factoid, analytical, multi-hop).

拒绝 any 分块 strategy 不使用 min/max chunk size enforcement. 拒绝 overlap above 20% 不使用 an ablation showing it helps. 标记 语义 分块 recommendations 不使用 a min-词元 floor.
