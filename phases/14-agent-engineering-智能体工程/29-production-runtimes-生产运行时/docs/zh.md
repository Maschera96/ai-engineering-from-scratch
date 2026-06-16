# 生产运行时：队列、事件、定时任务

> 生产环境中的智能体运行在六种运行时形态上：请求-响应、流式、持久化执行、基于队列的后台、事件驱动和定时调度。先选定形态，再选框架。可观测性在每一种形态中都是承重结构。

**Type:** Learn
**Languages:** Python (stdlib)
**Prerequisites:** Phase 14 · 13 (LangGraph), Phase 14 · 22 (Voice)
**Time:** ~60 minutes

## Learning Objectives

- 说出六种生产运行时形态，并将每一种与对应的框架／产品模式匹配。
- 解释为什么持久化执行（LangGraph）对长周期任务至关重要。
- 描述事件驱动运行时，以及 Claude Managed Agents 在何时适用。
- 解释多步骤智能体中"可观测性是承重结构"这一论断。

## The Problem

生产环境中的智能体会以 Jupyter notebook 暴露不出来的方式失败：第 37 步发生网络超时、用户在语音通话中途挂断、定时任务在机器重启时挂掉、后台 worker 内存耗尽。运行时形态决定了哪些失败是可以幸存的。

## The Concept

### 请求-响应

- 同步 HTTP。用户等待任务完成。
- 仅适用于短任务（<30s）。
- 技术栈：Agno（Python + FastAPI）、Mastra（TypeScript + Express/Hono/Fastify/Koa）。
- 可观测性：标准 HTTP 访问日志 + OTel spans。

### 流式

- 通过 SSE 或 WebSocket 实现渐进式输出。
- LiveKit 将其扩展到 WebRTC，用于语音/视频（Lesson 22）。
- 技术栈：任何支持流式的框架 + 能处理 SSE/WS 的前端。
- 可观测性：逐 chunk 计时、首 token 延迟、尾延迟。

### 持久化执行

- 每一步之后都对状态做检查点；失败时自动恢复。
- AutoGen v0.4 的 actor 模型将故障隔离到单个智能体（Lesson 14）。
- LangGraph 的核心差异化能力（Lesson 13）。
- 在步骤数量未知且恢复成本高时不可或缺。

### 基于队列／后台

- 任务进入队列，worker 取出，结果通过 webhook 或 pub/sub 回流。
- 对长周期智能体（按 Anthropic 的 computer use 公告，每个任务数十到数百步）不可或缺。
- 技术栈：Celery（Python）、BullMQ（Node）、SQS + Lambda（AWS）、自建。
- 可观测性：队列深度、单任务延迟分布、DLQ 大小。

### 事件驱动

- 智能体订阅触发器：新邮件、PR 打开、cron 触发。
- Claude Managed Agents 开箱即覆盖这一点（Lesson 17）。
- CrewAI Flows（Lesson 15）将事件驱动的确定性工作流结构化。
- 可观测性：触发来源、事件到启动的延迟、智能体延迟。

### 定时调度

- 周期性运行的 cron 形态智能体。
- 与持久化执行结合，让失败的夜间运行在下一次触发时恢复。
- 技术栈：Kubernetes CronJob + 持久化框架；托管方案（Render cron、Vercel cron）。

### 2026 部署模式

- **CrewAI Flows** 用于事件驱动的生产环境。
- **Agno** 无状态 FastAPI，用于 Python 微服务。
- **Mastra** 服务端适配器（Express、Hono、Fastify、Koa），用于嵌入。
- **Pipecat Cloud / LiveKit Cloud** 用于托管语音（Lesson 22）。
- **Claude Managed Agents** 用于托管的长时间运行异步任务。

### 可观测性是承重结构

如果没有 OpenTelemetry GenAI spans（Lesson 23）外加 Langfuse/Phoenix/Opik 后端（Lesson 24），你无法调试一个在第 40 步失败的多步骤智能体。对生产环境而言这不是可选项。它决定了你是"快速调试"还是"加更多日志后从头重放"。

### 生产运行时会在哪里失败

- **形态选错。** 为一个 5 分钟的任务选了请求-响应。用户挂断；worker 堆积；重试叠加。
- **没有 DLQ。** 队列 worker 没有死信队列。失败的任务凭空消失。
- **不透明的后台工作。** 后台智能体运行时没有 trace 导出。在用户报告之前，失败都是不可见的。
- **跳过持久化状态。** 任何超过 30 秒、且无法承受重启的运行都需要持久化执行。

## Build It

`code/main.py` 是一个仅用标准库的多形态演示：

- 请求-响应端点（普通函数）。
- 流式处理器（生成器）。
- 带 DLQ 的基于队列的 worker。
- 事件触发器注册表。
- cron 形态的调度器。

运行它：

```bash
python3 code/main.py
```

输出：五条 trace，展示同一个任务在各形态下的行为。相同的智能体逻辑，不同的外层外壳。持久化执行（第六种形态）有意放在 Lesson 13 中通过 LangGraph 检查点来讲解。

## Use It

- **请求-响应** 用于聊天式 UX。
- **流式** 用于渐进式响应。
- **持久化** 用于长周期任务。
- **队列** 用于批处理／异步／长时间运行。
- **事件** 用于智能体的反应性。
- **Cron** 用于日常维护（记忆整合、评估、成本报告）。

## Ship It

`outputs/skill-runtime-shape.md` 为某个任务挑选运行时形态，并接好可观测性要求。

## Exercises

1. 把你 Lesson 01 的 ReAct 循环移植到你技术栈的全部六种形态上。哪种形态适合哪个产品界面？
2. 给基于队列的演示加上 DLQ。模拟 10% 的任务失败；暴露出 DLQ 大小。
3. 写一个 cron 触发的评估智能体，每晚针对当天排名前 20 的 trace 运行。
4. 实现带背压（backpressure）的流式：如果客户端很慢，就暂停智能体。这与轮次预算（turn budget）如何相互作用？
5. 阅读 Claude Managed Agents 文档。什么时候你会把一个自托管的长周期智能体迁移到托管方案？

## Key Terms

| Term | What people say | What it actually means |
|------|----------------|------------------------|
| 请求-响应 | "同步" | 用户等待；仅限短任务 |
| 流式 | "SSE / WS" | 渐进式输出；更好的 UX；逐 chunk 可观测延迟 |
| 持久化执行 | "从失败处恢复" | 检查点状态；在最后一步重启 |
| 基于队列 | "后台任务" | 生产者／worker 池／DLQ |
| 事件驱动 | "基于触发器" | 智能体对外部事件做出反应 |
| DLQ | "死信队列" | 失败任务的停放区 |
| Claude Managed Agents | "托管 harness" | Anthropic 托管的长时间运行异步，带缓存 + 压缩 |

## Further Reading

- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) — 持久化执行细节
- [Claude Managed Agents overview](https://platform.claude.com/docs/en/managed-agents/overview) — 托管的长时间运行异步
- [Anthropic, Introducing computer use](https://www.anthropic.com/news/3-5-models-and-computer-use) — "每个任务数十到数百步"
- [AutoGen v0.4 (Microsoft Research)](https://www.microsoft.com/en-us/research/articles/autogen-v0-4-reimagining-the-foundation-of-agentic-ai-for-scale-extensibility-and-robustness/) — actor 模型的故障隔离
