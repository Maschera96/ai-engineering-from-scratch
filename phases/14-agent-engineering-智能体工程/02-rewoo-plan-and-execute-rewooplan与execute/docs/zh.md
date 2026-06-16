# ReWOO 与 Plan-and-Execute：解耦的规划

> ReAct 在一条流中交替进行思考与行动。ReWOO 将二者分离：先做一个大规划，然后执行。在 HotpotQA 上 token 数减少 5 倍、准确率提升 +4%，而且你可以把规划器蒸馏到一个 7B 模型里。Plan-and-Execute 将其一般化；Plan-and-Act 把它扩展到了网页导航。

**类型：** 构建
**语言：** Python（标准库）
**前置要求：** Phase 14 · 01（Agent Loop）
**时长：** 约 60 分钟

## 学习目标

- 解释为什么 ReWOO 的 Planner / Worker / Solver 拆分相比 ReAct 的交替循环能节省 token 并提升鲁棒性。
- 实现一个 plan DAG、一个按依赖排序的执行器，以及一个组合 worker 输出的 solver——全部用 stdlib。
- 用 2026 年「五种工作流模式」框架（Anthropic）来判断一个任务应当采用 plan-然后-execute 还是交替式 ReAct。
- 认识到何时需要 Plan-and-Act 的合成规划数据来应对长跨度的网页或移动端任务。

## 问题

ReAct 交替的「思考-行动-观察」循环简单且灵活，但每次工具调用都必须携带完整的先前上下文——包括之前的每一次思考。token 用量随深度呈二次增长。更糟的是：当某个工具在循环中途失败时，模型必须从错误观察中重新推导出整个规划。

ReWOO（Xu 等人，arXiv:2305.18323，2023 年 5 月）注意到了这一点，并下了一个赌注：先把整件事规划好，并行地获取证据，最后再组合出答案。一次 LLM 调用做规划，N 次工具调用获取证据（可以并行），一次 LLM 调用做求解。代价是灵活性降低（规划是静态的），换来的是大幅提升的 token 效率和更清晰的失败模式。

## 概念

### 三个角色

```
Planner： user_question -> [plan_dag]
Workers： [plan_dag]     -> [evidence]        (tool calls，possibly parallel)
Solver：  user_question，plan_dag，evidence -> final_answer
```

Planner 产生一个 DAG。每个节点指明一个工具、它的参数，以及它依赖哪些更早的节点（用类似 `#E1`、`#E2` 的引用）。Workers 按拓扑顺序执行节点。Solver 把所有内容拼接在一起。

### 为什么 token 减少 5 倍

ReAct 的提示长度随步数线性增长。在第 10 步，提示中包含思考 1 加行动 1 加观察 1 加思考 2 加行动 2 加观察 2，依此类推。每个中间步骤还会冗余地包含原始提示。

ReWOO 只付出一次 planner 提示（很大）、N 次小的 worker 提示（每次只有工具调用，没有链条）和一次 solver 提示。在 HotpotQA 上，论文测得 token 数约减少 5 倍，同时准确率绝对值提升 +4。

### 为什么它更鲁棒

如果在 ReAct 中 worker 3 失败了，循环必须在流的中途推理摆脱错误。在 ReWOO 中，worker 3 返回一个错误字符串；solver 在上下文中连同原始规划一起看到它，并可以优雅降级。失败定位是按节点而非按步骤进行的。

### Planner 蒸馏

论文的第二个结果：因为 planner 不会看到观察，你可以在一个 175B 教师模型产出的 planner 输出上微调一个 7B 模型。小模型负责规划；推理时不再需要大模型。这如今已成标准做法——许多 2026 年的生产级智能体使用小规划器加大执行器，或反之。

### Plan-and-Execute（LangChain，2023）

LangChain 团队 2023 年 8 月的帖子把 ReWOO 一般化为一个模式名称：Plan-and-Execute。前置的 planner 发出一个步骤列表，executor 运行每个步骤，可选的 replanner 可以在观察结果后进行修订。这比 ReWOO 更接近 ReAct（replanner 把观察带回到规划中），但保留了 token 节省。

### Plan-and-Act（Erdogan 等人，arXiv:2503.09572，ICML 2025）

Plan-and-Act 把该模式扩展到长跨度的网页和移动端智能体。其关键贡献是合成规划数据：一个带标注的轨迹生成器产出训练数据，其中规划是显式的。用于微调能在 WebAren类任务上持续工作超过 30–50 步的 planner 模型——在这类任务中，单条 ReAct 轨迹会丧失连贯性。

### 何时选用哪一种

| 模式 | 何时使用 |
|---------|------|
| ReAct | 短任务、未知环境、需要响应式异常处理 |
| ReWOO | 已知工具的结构化任务、对 token 敏感、证据可并行化 |
| Plan-and-Execute | 类似 ReWOO，但在部分执行后进行重新规划 |
| Plan-and-Act | 长跨度（>30 步）、网页/移动端/computer-use |
| Tree of Thoughts | 值得为搜索付出代价（第 04 课） |

Anthropic 2024 年 12 月的建议：从最简单的开始。如果任务只是一次工具调用加一段摘要，就别构建 ReWOO。如果任务是一个 40 步的研究任务，就别单用 ReAct。

## 动手构建

`code/main.py` 实现了一个玩具版 ReWOO：

- `Planner` — 一个脚本化策略，从提示产出一个 plan DAG。
- `Worker` — 通过注册表分派每个节点的工具调用。
- `Solver` — 脚本化的组合逻辑，读取证据并产生最终答案。
- 依赖解析 — 类似 `#E1` 的引用会被替换为更早的 worker 输出。

该演示使用一个两步规划回答「法国首都的人口是多少，四舍五入到百万？」：(1) 查找首都，(2) 查找人口，然后求解。

运行它：

```
python3 code/main.py
```

追踪信息先展示完整规划，然后是 worker 结果，再是 solver 组合。把 token 计数（我们打印一个粗略的字符计数）与 ReAct 风格的交替运行做对比——在这类结构化任务上 ReWOO 胜出。

## 使用它

LangGraph 把 Plan-and-Execute 作为一个 recipe 提供（ReAct 用 `create_react_agent`，plan-execute 用自定义图）。CrewAI 的 Flows 直接编码了该模式：你预先定义任务，Flow DAG 执行它们。Plan-and-Act 的合成数据方法目前仍主要停留在研究阶段；其运行时模式（显式 plan DAG）已通过 LangGraph 和 CrewAI Flows 在生产中落地。

## 交付它

`outputs/skill-rewoo-planner.md` 在给定工具目录的情况下，从用户请求生成一个 ReWOO plan DAG。它在交给执行器之前会校验规划（无环、每个引用都已解析、每个工具都存在）。

## 练习

1。为相互独立的规划节点并行化 worker 执行。在一个有 2 个并行组的 6 节点 DAG 上，这能带来什么收益？
2。添加一个 replanner 节点，当任何 worker 返回错误时触发。把 ReWOO 变成 Plan-and-Execute 的最小改动是什么？
3。把 `Planner` 替换成一个小模型（7B 级别），并把 `Solver` 保留在前沿模型上。比较端到端质量——这种拆分在哪里会失败？
4。阅读 ReWOO 论文第 4 节关于 planner 蒸馏的内容。在概念上复现 175B -> 7B 的结果：你需要什么训练数据，以及如何为规划质量打分？
5。把这个玩具版移植到 Plan-and-Act 的轨迹形态：规划是一个序列，而非 DAG。哪些权衡发生了变化？

## 关键术语

| 术语 | 人们怎么说 | 它实际的含义 |
|------|----------------|------------------------|
| ReWOO | 「无观察推理」 | 先规划，然后并行获取证据，再求解——规划提示中没有观察 |
| Plan-and-Execute | 「LangChain 的 plan-execute 模式」 | 在执行后带一个可选 replanner 节点的 ReWOO |
| Plan-and-Act | 「扩展版的 plan-execute」 | 显式的 planner/executor 拆分，配合用于长跨度任务的合成规划训练数据 |
| 证据引用 | 「#E1、#E2、……」 | 规划节点占位符，在分派时替换为先前的 worker 输出 |
| Planner 蒸馏 | 「小规划器，大执行器」 | 在大教师模型的 planner 轨迹上微调一个小模型 |
| Token 效率 | 「更少的往返」 | 论文中相比 ReAct，在 HotpotQA 上 token 数减少 5 倍 |
| DAG 执行器 | 「拓扑分派器」 | 按依赖顺序运行规划节点；每一层内并行 |

## 延伸阅读

- [Xu 等人，ReWOO：Decoupling Reasoning from Observations (arXiv:2305.18323)](https://arxiv.org/abs/2305.18323) — 经典论文
- [Erdogan 等人，Plan-and-Act (arXiv:2503.09572)](https://arxiv.org/abs/2503.09572) — 配合合成规划的扩展版 planner-executor
- [LangGraph Plan-and-Execute 教程](https://docs.langchain.com/oss/python/langgraph/overview) — 框架 recipe
- [Anthropic，构建ing Effective Agents](https://www.anthropic.com/research/building-effective-agents) — 选择能奏效的最简单模式
