# Agno 与 Mastra：生产级运行时

> Agno（Python）与 Mastra（TypeScript）是 2026 年的生产运行时组合。Agno 的目标是微秒级的智能体实例化和无状态的 FastAPI 后端。Mastr在 Vercel AI SDK 基础上提供智能体、工具、工作流、统一的模型路由以及组合式存储。

**类型：** 学习
**语言：** Python、TypeScript
**前置知识：** 阶段 14 · 01（智能体循环）、阶段 14 · 13（LangGraph）
**时长：** 约 45 分钟

## 学习目标

- 识别 Agno 的性能目标，以及它们何时重要。
- 说出 Mastr的三个基础原语——Agents、Tools、Workflows——以及受支持的服务端适配器。
- 解释为什么无状态、按会话划分作用域的 FastAPI 后端是推荐的 Agno 生产路径。
- 针对给定的技术栈（Python 优先 vs TypeScript 优先）在 Agno 与 Mastr之间做出选择。

## 问题

LangGraph、AutoGen、CrewAI 都偏重框架。想要“只要智能体循环，要快，跑在我自己的运行时里”的团队会转向 Agno（Python）或 Mastra（TypeScript）。两者都放弃了一部分由框架掌控的原语，换取原始速度和与周边技术栈更紧密的契合。

## 概念

### Agno

- Python 运行时，前身为 Phi-data。
- “没有图、链或复杂的模式——只是纯粹的 python。”
- 来自其文档的性能目标：约 2μs 的智能体实例化、每个智能体约 3.75 KiB 内存、约 23 个模型提供商。
- 生产路径：无状态、按会话划分作用域的 FastAPI 后端。每个请求都启动一个全新的智能体；会话状态存储在数据库中。
- 原生多模态（文本、图像、音频、视频、文件）和智能体式 RAG。

当你每秒有成千上万个短生命周期的智能体时（聊天扇入、评估流水线），速度目标才重要。当一个智能体运行 10 分钟时，它们就没那么重要了。

### Mastra

- TypeScript，构建于 Vercel AI SDK 之上。
- 三个原语：**Agents**、**Tools**（Zod 类型化）、**Workflows**。
- 统一模型路由——94 个提供商中的 3,300+ 个模型（2026 年 3 月）。
- 组合式存储：记忆、工作流、可观测性可分别写入不同后端；大规模可观测性推荐使用 ClickHouse。
- Apache 2.0 协议，源码中的 `ee/` 目录采用源码可见的企业许可证。
- 面向 Express、Hono、Fastify、Ko的服务端适配器；对 下一个.js 和 Astro 提供一流集成。
- 提供 MastrStudio（localhost:4111）用于调试。
- 在 1.0 版本（2026 年 1 月）时拥有 22k+ GitHub 星标、每周 300k+ 次 npm 下载。

### 定位

两者都不打算成为 LangGraph。它们的竞争点在于：

- **语言契合度。** Agno 面向 Python 优先的团队；Mastr面向 TypeScript 优先的团队。
- **运行时人体工学。** Agno = 近乎零开销；Mastr= 与 Vercel 生态集成。
- **可观测性。** 两者都集成 Langfuse/Phoenix/Opik（第 24 课），但 MastrStudio 是第一方的。

### 何时选择哪一个

- **Agno**——Python 后端、大量短生命周期智能体、强性能要求、FastAPI 团队。
- **Mastra**——TypeScript 后端、下一个.js / Vercel 部署、统一的多提供商模型路由、Zod 类型化工具。
- **LangGraph**（第 13 课）——当持久化状态和显式的图推理比原始速度更重要时。
- **OpenAI / Claude Agent SDK**——当你想要提供商产品化的形态时（第 16–17 课）。

### 这种模式会在哪里出错

- **为性能而性能。** 当工作负载是每个请求一次缓慢的智能体调用时，仅仅因为“2μs”听起来不错就选择 Agno。开销并不是瓶颈。
- **生态锁定。** Mastr的 Vercel 风格集成在 Vercel 上是加分项，在别处则是减分项。
- **企业许可证混淆。** Mastr的 `ee/` 目录是源码可见的，并非 Apache 2.0。如果你打算 fork，请阅读这些许可证。

## 动手构建

本课主要是比较性的——没有任何单一的代码产物能让两个框架都得到充分展示。请参阅 `code/main.py` 中的并排玩具示例：一个最小化的“运行一个智能体、流式输出、持久化会话”流程被实现了两次（一次按 Agno 风格，一次按 Mastr风格）。

运行它：

```
python3 code/main.py
```

两条结构上不同但功能上等价的轨迹。

## 使用它

- **Agno**——需要速度和 FastAPI 形态的 Python 后端。
- **Mastra**——拥有众多提供商和工作流原语的 TypeScript 后端。
- 两者都提供第一方可观测性钩子。两者都集成 Langfuse。

## 交付它

`outputs/skill-runtime-picker.md` 会根据技术栈、延迟预算和运维形态，在 Agno、Mastra、LangGraph 或提供商 SDK 之间做出选择。

## 练习

1。阅读 Agno 的文档。把标准库的 ReAct 循环（第 01 课）移植到 Agno。什么消失了？什么保留了？
2。阅读 Mastr的文档。把同一个循环移植到 Mastra。工具类型化方面有什么变化（Zod vs 什么都没有）？
3。基准测试：在你的技术栈上测量智能体实例化延迟。Agno 的 2μs 对你的工作负载重要吗？
4。设计一次迁移：如果你一直在用 Python 跑 CrewAI，迁移到 Agno 会破坏什么？
5。阅读 Mastr的 `ee/` 许可证条款。哪些限制会影响一个开源 fork？

## 关键术语

| 术语 | 人们怎么说 | 它实际意味着什么 |
|------|----------------|------------------------|
| Agno | “快速的 Python 智能体” | 无状态、按会话划分作用域的智能体运行时 |
| Mastr| “基于 Vercel AI SDK 的 TypeScript 智能体” | Agents + Tools + Workflows + Model Router |
| Unified Model Router | “多提供商访问” | 跨 94 个提供商访问 3,300+ 个模型的单一客户端 |
| Composite storage | “多个后端” | 记忆/工作流/可观测性各自写入不同的存储 |
| MastrStudio | “本地调试器” | 用于内省智能体的 localhost:4111 界面 |
| Source-available | “非开源” | 许可证允许阅读源码但限制商业使用 |

## 延伸阅读

- [Agno Agent Framework docs](https://www.agno.com/agent-framework) — 性能目标、FastAPI 集成
- [Mastrdocs](https://mastra.ai/docs) — 原语、服务端适配器、Model Router
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) — 有状态图的替代方案
- [Comet Opik](https://www.comet.com/site/products/opik/) — Mastr集成所引用的可观测性比较
