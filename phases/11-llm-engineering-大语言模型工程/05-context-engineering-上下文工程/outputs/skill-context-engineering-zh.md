---
name: skill-context-engineering-zh
description: Decision framework for designing 上下文 assembly pipelines based on 任务 type, window size, and 延迟 预算
version: 1.0.0
phase: 11
lesson: 05
tags: [context-engineering, context-window, rag, memory, tool-selection, lost-in-the-middle]
---

# 上下文工程

当building an LLM 应用, apply this framework to design the 上下文 assembly 流水线.

## Core principles

1. **上下文 is scarce.** A 128K window sounds large but fills fast. 预算 every component explicitly.
2. **注意力 is uneven.** 模型 attend more to the start and end. Put critical information there. The middle is the dead zone.
3. **Dynamic beats static.** Different 查询 need different 上下文. Assemble per 查询, not once at startup.
4. **Less is more.** A curated 10K 上下文 outperforms a dumped 100K 上下文. Signal-to-noise 比例 matters more than total information.
5. **Measure everything.** You cannot 优化 what you do not measure. Count 词元 per component on every request.

## 上下文 预算 guidelines

|Component|Typical Range|Priority|压缩 Strategy|
|-----------|-------------|----------|---------------------|
|系统 提示词|200-1,000 词元|修复ed, high|Write tight, remove redundancy|
|工具 definitions|500-3,000 词元|Dynamic, medium|Prune by 查询 intent|
|检索到的 上下文|1,000-5,000 词元|Dynamic, high|Rerank + 阈值 + deduplicate|
|Conversation history|500-5,000 词元|Dynamic, medium|Summarize old turns|
|少样本 examples|500-2,000 词元|Dynamic, high|Select by 任务 相似度|
|用户 查询|50-500 词元|修复ed, highest|N/A|
|生成 reserve|2,000-8,000 词元|修复ed|Adjust by expected 输出 length|

## When to use each 内存 type

**Short-term (conversation history):** The current session. Managed by summarization. Compress turns older than 5-10 exchanges. Keep the last 3-4 turns verbatim.

**Long-term (facts database):** Preferences and project facts that persist across sessions. Retrieve on session start. Examples: "用户 prefers Python", "project uses PostgreSQL", "team follows trunk-based development". Store in CLAUDE.md, a database, or a 结构化 内存 系统.

**Episodic (past interactions):** Specific past conversations relevant to the current 任务. Store as 嵌入s, retrieve by 相似度. "Last week we debugged a similar auth issue" is episodic 内存.

## 工具 selection strategy

Do not include all 工具 in every request. This wastes 词元 and confuses the 模型.

1. Classify the 查询 intent (code, email, calendar, research, 数据)
2. Map intents to 工具 categories
3. Include only 匹配 工具
4. 如果intent is ambiguous, include 工具 from the top 2 categories
5. Always include a "general" 工具 (like web search) as 备选方案

Expected savings: 60-80% of 工具 definition 词元 on 查询 with clear intent.

## 检索 best practices

- **Rerank after 检索.** Vector 相似度 is a rough filter. A reranker (cross-encoder or LLM-based) improves precision significantly.
- **Set a relevance 阈值.** Do not include chunks below 0.3 cosine 相似度. They add 噪声.
- **Deduplicate.** If two chunks share 80%+ content, keep only the higher-scored one.
- **Apply lost-in-the-middle ordering.** Place the most relevant chunks first and last.
- **限制 total 检索 词元.** 3-5 highly relevant chunks beat 15 mediocre ones.

## History management

- Keep the last 3-4 turns verbatim (the 模型 needs recent 上下文)
- Summarize older turns into a digest ("We discussed X, decided Y, and blocked on Z")
- Drop system-generated turns that add no information (工具 invocations with no user-facing content)
- Trigger 压缩 when history exceeds 30% of the available 预算

## Red flags

- 系统 提示词 exceeds 2,000 词元: probably includes information that should be dynamic
- All 工具 included on every request: implement intent-based selection
- No relevance filtering on 检索: you are dumping 噪声 into the window
- History grows unbounded: summarization is not implemented
- No 生成 reserve: the 模型 truncates its 响应
- Same information in 3 places (系统 提示词, 检索到的 doc, history): deduplicate
- 上下文 utilization over 60%: you are leaving too little room for the 模型 to "think"
