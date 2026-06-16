# 记忆 block 与 Sleep-Time Compute（Letta）

> MemGPT 在 2024 年更名为 Letta。2026 年的演进新增了两个理念：模型可以直接编辑的离散功能性记忆 block，以及在主智能体空闲时异步整合记忆的 sleep-time 智能体。这正是把记忆扩展到单次对话之外的方式。

**类型：** 构建
**语言：** Python（标准库）
**前置条件：** Phase 14 · 07（MemGPT）
**时长：** 约 75 分钟

## 学习目标

- 说出 Lett使用的三个记忆层级（core、recall、archival）以及各自的作用。
- 解释记忆 block 模式：Human block、Personblock 和用户自定义 block 作为一等类型化对象。
- 描述什么是 sleep-time compute、它为何位于关键路径之外、以及它为何可以运行比主智能体更强的模型。
- 实现一个脚本化的双智能体循环，其中主智能体负责生成响应，sleep-time 智能体在各轮之间整合 block。

## 问题

MemGPT（第 07 课）解决了虚拟内存式的控制流。随之出现了三个生产环境问题：

1。**延迟。** 每一次记忆操作都位于关键路径上。如果智能体必须在用户等待时进行裁剪、摘要或调和，尾部延迟就会爆炸。
2。**记忆腐烂。** 写入不断累积。被推翻的事实仍然保留。检索淹没在过时内容里。
3。**结构丢失。** 扁平的 archival 存储无法表达“Human block 始终在 prompt 中；Personblock 始终在 prompt 中；Task block 每个会话各自切换”。

Letta（letta.com）是 2026 年的重写版本。记忆 block 让结构显式化；sleep-time compute 把整合搬到关键路径之外。

## 概念

### 三个层级

| 层级 | 范围 | 存放位置 | 写入者 |
|------|-------|----------------|------------|
| Core | 始终可见 | 在主 prompt 内 | 智能体工具调用 + sleep-time 重写 |
| Recall | 对话历史 | 可检索 | 自动轮次记录 |
| Archival | 任意事实 | 向量 + KV + 图 | 智能体工具调用 + sleep-time 摄入 |

Core 就是 MemGPT 的 core。Recall 是带有被驱逐尾部的对话缓冲区。Archival 是外部存储。这种拆分清理了 MemGPT 两层结构的职责过载问题。

### 记忆 block

block是core 层级中一个类型化、持久化、可编辑的段落。最初的 MemGPT 论文定义了两种：

- **Human block** —— 关于用户的事实（姓名、角色、偏好、目标）。
- **Personblock** —— 智能体的自我概念（身份、语气、约束）。

Lett将其泛化为任意用户自定义 block：用于当前目标的 `Task` block、用于代码库事实的 `Project` block、用于硬性约束的 `Safety` block。每个 block 都有 `id`、`label`、`value`、`limit`（字符上限）、`description`（让模型知道何时编辑它）。

block 通过工具接口可编辑：

- `block_append(label，text)`
- `block_replace(label，old，new)`
- `block_read(label)`
- `block_summarize(label)` —— 压缩一个接近上限的 block。

### Sleep-time compute

2025 年 Lett的新增功能：在后台、关键路径之外运行第二个智能体。sleep-time 智能体处理对话记录和代码库上下文，把 `learned_context` 写入共享 block，并整合或失效化 archival 记录。

由此带来的特性：

- **无延迟代价。** 主响应不必等待记忆操作。
- **允许更强的模型。** sleep-time 智能体可以是更昂贵、更慢的模型，因为它不受延迟约束。
- **天然的整合窗口。** 在用户不等待时去重、摘要、失效化被推翻的事实。

这种形态契合人类的工作方式：你做完任务，睡上一觉，长期记忆在夜里沉淀下来。

### LettV1 与原生推理

LettV1（`letta_v1_agent`，2026）弃用了 `send_message`/heartbeat 和内联的 `Thought:` token，转而采用原生推理。Responses API（OpenAI）和带扩展思考的 Messages API（Anthropic）在单独的通道上输出推理，并跨轮次传递（生产环境中跨提供商加密）。控制循环仍然是 ReAct。思考轨迹是结构性的，而非 prompt 形态的。

### 这种模式会在哪里出错

- **block 膨胀。** 无限的 `block_append` 很快就会触及上限。在写入超出上限之前接入一个 block 摘要器。
- **静默漂移。** sleep-time 智能体重写了某个 block，而主智能体从未察觉。给 block 加版本，并在轨迹中呈现 diff。
- **被污染的整合。** sleep-time 智能体把攻击者可触及的内容处理进 core。第 27 课同样适用于 sleep-time 接触面。

## 动手构建

`code/main.py` 实现了：

- `Block` —— id、label、value、limit、description。
- `BlockStore` —— CRUD + `near_limit(label)` 辅助方法。
- 两个脚本化智能体 —— `PrimaryAgent` 负责一轮服务，`SleepTimeAgent` 在各轮之间整合。
- 一段轨迹，展示一个带 block 写入的三轮对话，外加一次 sleep-time 处理，对某个 block 进行摘要并失效化一条过时的事实。

运行：

```
python3 code/main.py
```

记录展示了这种拆分：主轮次快速且产生原始写入；sleep 处理则压缩并清理。

## 实际使用

- **Letta**（letta.com）作为参考实现。可自托管或使用托管云。
- **Claude Agent SDK skills** 作为 block 形态的知识 —— skill 是一个具名、带版本、可检索的指令 block，智能体按需加载。
- **自建方案** 适用于希望掌控存储后端的团队。使用 LettAPI 契约，以便日后迁移。

## 交付

`outputs/skill-memory-blocks.md` 为任意运行时生成一个 Lett形态的 block 系统，带有 sleep-time 钩子，包括安全规则和引用接线。

## 练习

1。添加一个 `block_summarize` 工具，当 `near_limit` 返回 true 时，用模型生成的摘要替换 block 的值。哪个触发阈值能同时最小化摘要调用次数和 block 溢出？
2。在 archival 上实现 sleep-time 去重：两条文本 token 重叠率 >90% 的记录合并为一条。只在 sleep 处理中做，绝不在关键路径上做。
3。给 block 加版本。每次写入都记录旧值和一份 diff。暴露 `block_history(label)`，让运维人员能调试“为什么智能体忘了 X”。
4。把 sleep-time 智能体当作不可信的写入者。当它们触碰 Person或 Safety block 时，提交前要求第二个智能体审查。
5。把示例移植为使用 LettAPI（`letta_v1_agent`）。block 模式有哪些变化，原生推理又如何改变轨迹形态？

## 关键术语

| 术语 | 人们常说的 | 它实际的含义 |
|------|----------------|------------------------|
| 记忆 block | “可编辑的 prompt 段落” | core 记忆中类型化、持久化、可被 LLM 编辑的段落 |
| Human block | “用户记忆” | 关于用户的事实，固定在 core 中 |
| Personblock | “智能体身份” | 自我概念、语气、约束，固定在 core 中 |
| Sleep-time compute | “异步记忆工作” | 在关键路径之外做整合的第二个智能体 |
| Core / Recall / Archival | “层级” | 三层记忆拆分：始终可见 / 对话 / 外部 |
| block 上限 | “上限” | 每个 block 的字符限制；强制进行摘要 |
| 原生推理 | “思考通道” | 提供商层级的推理输出，而非 prompt 层级的 `Thought:` |
| 学习ed context | “sleep 输出” | sleep-time 智能体写入共享 block 的事实 |

## 延伸阅读

- [Letta，Memory Blocks blog](https://www.letta.com/blog/memory-blocks) —— block 模式
- [Letta，Sleep-time Compute blog](https://www.letta.com/blog/sleep-time-compute) —— 异步整合
- [Letta，RearchitectingAgent Loop](https://www.letta.com/blog/letta-v1-agent) —— 原生推理重写
- [Packer et al.，MemGPT (arXiv:2310.08560)](https://arxiv.org/abs/2310.08560) —— 起源
