# 编排模式：Supervisor、Swarm、Hierarchical

> 四种编排模式在 2026 年的各类框架中反复出现：supervisor-worker、swarm / peer-to-peer、hierarchical、debate。Anthropic 的建议是："关键在于为你的需求构建正确的系统。" 从简单开始；只有当单个 agent 加上五种工作流模式仍不够用时，才引入拓扑结构。

**类型：** Learn + Build
**语言：** Python (stdlib)
**前置知识：** Phase 14 · 12（工作流模式）、Phase 14 · 25（多智能体辩论）
**时间：** ~60 分钟

## 学习目标

- 说出四种反复出现的编排模式以及各自适用的场景。
- 描述 2026 年 LangChain 的建议：基于工具调用的监督 vs supervisor 库。
- 解释 Anthropic 的"构建正确系统"原则，以及它如何约束拓扑选择。
- 用 stdlib 针对一个共用的脚本化 LLM 实现全部四种模式。

## 问题

团队往往在真正需要之前就伸手去用"多智能体"。有四种模式在各框架中反复出现；一旦你能叫出它们的名字，就能挑出正确的那个——或者干脆跳过拓扑结构。

## 概念

### Supervisor-worker

- 一个中心路由 LLM 把任务分派给专家 agent。
- 它来决定：回到自身循环、移交给专家、还是终止。
- 专家之间互不交谈；所有路由都经过 supervisor。

框架：LangGraph `create_supervisor`、Anthropic 的 orchestrator-workers、CrewAI Hierarchical Process。

**2026 年 LangChain 的建议：** 通过直接的工具调用来做监督，而不是用 `create_supervisor`。这能提供更精细的上下文工程控制——你来精确决定每个专家看到什么。

### Swarm / peer-to-peer

- agent 之间通过共享的工具面直接移交。
- 没有中心路由器。
- 比 supervisor 延迟更低（跳转更少）。
- 更难以推理（没有单一控制点）。

框架：LangGraph swarm 拓扑、OpenAI Agents SDK handoffs（当所有 agent 都可以移交给其他所有 agent 时）。

### Hierarchical

- supervisor 管理子 supervisor，子 supervisor 再管理 worker。
- 在 LangGraph 中实现为嵌套子图；在 CrewAI 中实现为嵌套 crew。
- 能扩展到庞大的 agent 群体，代价是运维复杂度。

何时需要它：当单个 supervisor 的上下文预算无法容纳所有专家的描述时。

### Debate

- 并行提议者 + 迭代式交叉批判（第 25 课）。
- 它其实不算编排——更像是验证——但在各框架中它会作为一种拓扑选择出现。

### CrewAI Crew vs Flow

CrewAI 把两种部署模式规范化：

- **Flow** 用于确定性的事件驱动自动化（生产环境推荐的起点）。
- **Crew** 用于基于角色的自主协作。

这与上面四种模式正交，但可以映射到拓扑：Flow 通常是 supervisor 或 hierarchical；Crew 通常是带有 LLM 路由器的 supervisor。

### Anthropic 的建议

"在 LLM 领域的成功不在于构建最复杂的系统，而在于为你的需求构建正确的系统。"

决策顺序：

1. 单个 agent + 工作流模式（第 12 课）——从这里开始。
2. Supervisor-worker——当你有 2-4 个专家时。
3. Swarm——当延迟比推理清晰度更重要时。
4. Hierarchical——只在 supervisor 上下文预算失败时。
5. Debate——当准确性比成本更重要时。

### 这个模式会在哪里出错

- **拓扑优先思维。** 在弄清楚多智能体能解决什么问题之前就说"我们需要多智能体"。
- **swarm 中来回弹跳的移交。** A -> B -> A -> B。使用跳转计数器。
- **假层级。** 因为"企业级"就搞三层；实际只有两个团队。把它压扁。

## 动手构建

`code/main.py` 用 stdlib 针对一个脚本化 LLM 实现全部四种模式：

- `Supervisor` —— 中心路由器。
- `Swarm` —— 带直接移交的 peer-to-peer。
- `Hierarchical` —— supervisor 的 supervisor。
- `Debate` —— 并行提议者 + 批判。

每种模式都处理同一个三意图任务（退款 / 缺陷 / 销售）。trace 的形状各不相同。

运行它：

```
python3 code/main.py
```

输出：每种模式的 trace + 操作计数。Supervisor 最清晰；swarm 最短；hierarchical 最深；debate 最昂贵。

## 使用它

- **LangGraph** 用于 supervisor 和 hierarchical（嵌套子图）。
- **OpenAI Agents SDK** 用于 handoffs-as-tools（supervisor 形态）。
- **CrewAI Flow** 用于生产环境的确定性。
- **自定义** 用于 debate，或者当你想要精确控制时。

## 交付它

`outputs/skill-orchestration-picker.md` 挑选一种拓扑并实现它。

## 练习

1. 通过移除路由器把 supervisor-worker 转换为 swarm。什么坏了？什么变好了？
2. 给 swarm 加一个跳转计数器：超过 3 次移交后拒绝。它能抓住 A->B->A 的来回弹跳吗？
3. 为一个有 12 个专家的领域构建一个两级 hierarchical 系统。如果不嵌套，上下文预算会在哪里失败？
4. 在一个生产形态的工作负载上对四种模式做剖析。在每个指标上（延迟、成本、准确性、可调试性）谁胜出？
5. 阅读 Anthropic 的 "Building Effective Agents" 文章。把你的每个生产流程映射到四种模式之一。有没有哪个无法干净地映射？

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| Supervisor-worker | "路由器 + 专家" | 中心 LLM 分派给专家；它们之间互不交谈 |
| Swarm | "peer-to-peer" | 通过共享工具直接移交；没有中心路由器 |
| Hierarchical | "supervisor 的 supervisor" | 用于大群体的嵌套子图 |
| Debate | "提议者 + 批判" | 并行提议者，交叉批判（第 25 课） |
| 基于工具调用的监督 | "不用库的 supervisor" | 把 supervisor 实现为直接工具调用以控制上下文 |
| Crew | "自主团队" | CrewAI 基于角色的协作模式 |
| Flow | "确定性工作流" | CrewAI 的事件驱动生产模式 |

## 延伸阅读

- [Anthropic, Building Effective Agents](https://www.anthropic.com/research/building-effective-agents) —— 五种模式 + agent vs 工作流
- [LangGraph overview](https://docs.langchain.com/oss/python/langgraph/overview) —— supervisor、swarm、hierarchical
- [CrewAI docs](https://docs.crewai.com/en/introduction) —— Crew vs Flow
- [Du et al., Society of Minds (arXiv:2305.14325)](https://arxiv.org/abs/2305.14325) —— 辩论模式
