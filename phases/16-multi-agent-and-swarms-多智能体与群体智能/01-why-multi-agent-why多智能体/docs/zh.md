# 为什么要使用多代理？

> 一名特工碰壁了。明智之举不是扩大代理规模，而是增加代理数量。

**类型：** 学习
**语言：** TypeScript
**先决条件：** 第 14 阶段（代理工程）
**时间：** ~60 分钟

## 学习目标

- 确定单代理的上限（上下文溢出、混合专业知识、顺序瓶颈）并解释何时拆分为多个代理是正确的举措
- 比较编排模式（管道、并行扇出、管理器、分层）并为给定的任务结构选择正确的模式
- 设计一个具有清晰角色边界、共享状态和通信契约的多代理系统
- 分析多代理复杂性（延迟、成本、调试难度）与单代理简单性的权衡

＃＃ 问题

您在第 14 阶段构建了一个代理。它有效。它可以读取文件、运行命令、调用 API 以及推理结果。然后你将其指向一个真实的代码库：200 个文件、三种语言、依赖于基础设施的测试以及在编写代码之前研究外部 API 的要求。

经纪人噎住了​​。不是因为 LLM 很愚蠢，而是因为任务超出了一个代理循环可以处理的范围。上下文窗口充满文件内容。代理忘记了 40 个工具调用前读取的内容。它试图同时成为一名研究员、一名编码员和一名审阅者，但三者都做得很差。

这就是单代理上限。每次任务需要时你都会点击它：

- **一个窗口无法容纳更多上下文** - 读取 50 个文件超过 200k 个标记
- **不同阶段的不同专业知识** - 研究需要与代码生成不同的提示
- **可以并行发生的工作** - 当您可以同时读取三个文件时，为什么要按顺序读取它们？

## 概念

### 单代理上限

单个代理是一个循环、一个上下文窗口、一个系统提示符。想象一下：

```
┌─────────────────────────────────────────┐
│            SINGLE AGENT                 │
│                                         │
│  ┌───────────────────────────────────┐  │
│  │         Context Window            │  │
│  │                                   │  │
│  │  research notes                   │  │
│  │  + code files                     │  │
│  │  + test output                    │  │
│  │  + review feedback                │  │
│  │  + API docs                       │  │
│  │  + ...                            │  │
│  │                                   │  │
│  │  ██████████████████████ FULL ███  │  │
│  └───────────────────────────────────┘  │
│                                         │
│  One system prompt tries to cover       │
│  research + coding + review + testing   │
│                                         │
│  Result: mediocre at everything         │
└─────────────────────────────────────────┘
```

三件事打破了：

1. **上下文饱和** - 工具结果堆积。到第 30 周，代理已消耗了 150k 个文件内容、命令输出和先前推理的令牌。第 5 回合的关键细节丢失了。

2. **角色混淆** - 系统提示“您是研究员、编码员、审阅者和测试员”，这会产生一个只进行一半研究、一半编码并且永远不会完成审阅的代理。

3. **顺序瓶颈** - 代理读取文件 A，然后读取文件 B，然后读取文件 C。三个串行 LLM 调用。三个串行工具执行。没有并行性。

### 多代理解决方案

分担工作。为每个代理提供一项作业、一个上下文窗口和一个针对该作业调整的系统提示：

```
┌──────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR                          │
│                                                          │
│  "Build a REST API for user management"                  │
│                                                          │
│         ┌──────────┬──────────┬──────────┐               │
│         │          │          │          │               │
│         ▼          ▼          ▼          ▼               │
│   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│   │RESEARCHER│ │  CODER   │ │ REVIEWER │ │  TESTER  │  │
│   │          │ │          │ │          │ │          │  │
│   │ Reads    │ │ Writes   │ │ Checks   │ │ Runs     │  │
│   │ docs,    │ │ code     │ │ code     │ │ tests,   │  │
│   │ finds    │ │ based on │ │ quality, │ │ reports  │  │
│   │ patterns │ │ research │ │ finds    │ │ results  │  │
│   │          │ │ + spec   │ │ bugs     │ │          │  │
│   └─────┬────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │
│         │           │            │             │         │
│         └───────────┴────────────┴─────────────┘         │
│                          │                               │
│                     Merge results                        │
└──────────────────────────────────────────────────────────┘
```

每个代理有：
- 集中的系统提示（“您是代码审查员。您唯一的工作就是发现错误。”）
- 它自己的上下文窗口（不受其他代理的工作污染）
- 清晰的 input/output 合约（接收研究笔记，输出代码）

### 执行此操作的真实系统

**Claude 代码子代理** - 当 Claude 代码使用 `Task` 生成子代理时，它会创建一个具有范围任务的子代理。父级保持其上下文干净。孩子做重点工作并返回总结。

**Devin** - 运行规划器代理、编码器代理和浏览器代理。计划者将工作分解为步骤。编码员编写代码。浏览器研究文档。每个都有单独的上下文。

**多代理编码团队 (SWE-bench)** - SWE-bench 上性能最佳的系统使用读取代码库的研究人员、设计修复程序的规划人员以及实现修复程序的编码人员。单代理系统得分较低。

**ChatGPT 深度研究** - 并行生成多个搜索代理，每个代理探索不同的角度，然后综合结果。

### 频谱

多代理不是二元的。它是一个谱：

```
SIMPLE ──────────────────────────────────────────── COMPLEX

 Single        Sub-         Pipeline      Team         Swarm
 Agent         agents

 ┌───┐       ┌───┐        ┌───┐───┐    ┌───┐───┐    ┌─┐┌─┐┌─┐
 │ A │       │ A │        │ A │ B │    │ A │ B │    │ ││ ││ │
 └───┘       └─┬─┘        └───┘─┬─┘    └─┬─┘─┬─┘    └┬┘└┬┘└┬┘
               │                │        │   │       ┌┴──┴──┴┐
             ┌─┴─┐          ┌───┘───┐    │   │       │shared │
             │ a │          │ C │ D │  ┌─┴───┴─┐    │ state │
             └───┘          └───┘───┘  │  msg   │    └───────┘
                                       │  bus   │
 1 loop      Parent +      Stage by    │       │    N peers,
 1 context   child tasks   stage       └───────┘    emergent
                                       Explicit      behavior
                                       roles
```

**单一代理** - 一个循环，一个提示。适合简单的任务。

**子代理** - 父代理为重点子任务生成子代理。家长维持该计划。孩子们回来报告。这就是 Claude 代码的作用。

**管道** - 代理按顺序运行。代理 A 的输出成为代理 B 的输入。适合分阶段工作流程：研究 -> 代码 -> 审查 -> 测试。

**团队** - 代理与共享消息总线并行运行。每个人都有自己的角色。协调者负责协调。当同时需要不同的技能时很好。

**群体** - 许多具有共享状态的相同或接近相同的代理。没有固定的协调器。代理从队列中获取工作。适合高吞吐量并行任务。

### 四种多代理模式

#### 模式 1：管道

```
Input ──▶ Agent A ──▶ Agent B ──▶ Agent C ──▶ Output
          (research)  (code)      (review)
```

每个代理都会转换数据并将其转发。道理很简单。一个阶段的失败会阻碍其余阶段的发展。

#### 模式 2：扇出/扇入

```
                ┌──▶ Agent A ──┐
                │              │
Input ──▶ Split ├──▶ Agent B ──├──▶ Merge ──▶ Output
                │              │
                └──▶ Agent C ──┘
```

将工作分配给并行代理，然后合并结果。适合分解为独立子任务的任务。

#### 模式 3：协调者-工作者

```
                    ┌──────────┐
                    │  Orch.   │
                    └──┬───┬───┘
                  task │   │ task
                 ┌─────┘   └─────┐
                 ▼               ▼
           ┌──────────┐   ┌──────────┐
           │ Worker A │   │ Worker B │
           └──────────┘   └──────────┘
```

聪明的协调者决定要做什么、委托给工作人员并综合结果。协调器本身就是一个代理，具有用于生成工作人员的工具。

#### 模式 4：同伴群

```
         ┌───┐ ◄──── msg ────▶ ┌───┐
         │ A │                  │ B │
         └─┬─┘                  └─┬─┘
           │                      │
      msg  │    ┌───────────┐     │ msg
           └───▶│  Shared   │◄────┘
                │  State    │
           ┌───▶│  / Queue  │◄────┐
           │    └───────────┘     │
      msg  │                      │ msg
         ┌─┴─┐                  ┌─┴─┐
         │ C │ ◄──── msg ────▶ │ D │
         └───┘                  └───┘
```

没有中央协调器。代理进行点对点通信。决策来自互动。调试起来比较困难，但可以扩展到许多代理。

### 何时不使用多代理

多代理增加了复杂性。代理之间的每条消息都是潜在的故障点。调试从“读取一个对话”到“跨五个代理跟踪消息”。

**在以下情况下保持单一代理：**
- 该任务适合一个上下文窗口（大约 100k 个工作数据标记）
- 不同阶段不需要不同的系统提示
- 顺序执行足够快
- 任务非常简单，拆分它会增加更多的开销而不是价值

**复杂性成本：**
- 每个代理边界都是一个有损压缩步骤：代理 A 的完整上下文被汇总为代理 B 的消息
- 协调逻辑（谁做什么、何时、以什么顺序）是其自身的错误来源
- 延迟增加：N 个代理意味着最少 N 个连续的 LLM 呼叫，如果需要来回通话则更多
- 成本成倍增加：每个代理独立燃烧代币

经验法则：如果一项任务调用的工具次数少于 20 次且适合 100k 个令牌，则将其保留为单代理。

```figure
swarm-messages
```

## 构建它

### 第 1 步：超载的单个代理

这是一个试图做所有事情的单一代理人。它有一个庞大的系统提示符和一个包含研究、代码和评论的上下文窗口：

```typescript
type AgentResult = {
  content: string;
  tokensUsed: number;
  toolCalls: number;
};

async function singleAgentApproach(task: string): Promise<AgentResult> {
  const systemPrompt = `You are a full-stack developer. You must:
1. Research the requirements
2. Write the code
3. Review the code for bugs
4. Write tests
Do ALL of these in a single conversation.`;

  const contextWindow: string[] = [];
  let totalTokens = 0;
  let totalToolCalls = 0;

  const research = await fakeLLMCall(systemPrompt, `Research: ${task}`);
  contextWindow.push(research.output);
  totalTokens += research.tokens;
  totalToolCalls += research.calls;

  const code = await fakeLLMCall(
    systemPrompt,
    `Given this research:\n${contextWindow.join("\n")}\n\nNow write code for: ${task}`
  );
  contextWindow.push(code.output);
  totalTokens += code.tokens;
  totalToolCalls += code.calls;

  const review = await fakeLLMCall(
    systemPrompt,
    `Given all previous context:\n${contextWindow.join("\n")}\n\nReview the code.`
  );
  contextWindow.push(review.output);
  totalTokens += review.tokens;
  totalToolCalls += review.calls;

  return {
    content: contextWindow.join("\n---\n"),
    tokensUsed: totalTokens,
    toolCalls: totalToolCalls,
  };
}
```

这种方法的问题：
- 上下文窗口随着每个阶段而增长。通过审查步骤，它包含研究笔记和代码以及先前的推理。
- 系统提示是通用的。无法针对每个阶段进行调整。
- 没有什么是并行运行的。

### 第 2 步：专业代理

现在把它分开。每个代理都有一份工作：

```typescript
type SpecialistAgent = {
  name: string;
  systemPrompt: string;
  run: (input: string) => Promise<AgentResult>;
};

function createSpecialist(name: string, systemPrompt: string): SpecialistAgent {
  return {
    name,
    systemPrompt,
    run: async (input: string) => {
      const result = await fakeLLMCall(systemPrompt, input);
      return {
        content: result.output,
        tokensUsed: result.tokens,
        toolCalls: result.calls,
      };
    },
  };
}

const researcher = createSpecialist(
  "researcher",
  "You are a technical researcher. Read documentation, find patterns, and summarize findings. Output only the facts needed for implementation."
);

const coder = createSpecialist(
  "coder",
  "You are a senior TypeScript developer. Given requirements and research notes, write clean, tested code. Nothing else."
);

const reviewer = createSpecialist(
  "reviewer",
  "You are a code reviewer. Find bugs, security issues, and logic errors. Be specific. Cite line numbers."
);
```

每个专家都有针对性的提示。每个都获得一个干净的上下文窗口，其中仅包含它需要的输入。

### 第 3 步：通过消息进行协调

通过显式消息传递将专家连接在一起：

```typescript
type AgentMessage = {
  from: string;
  to: string;
  content: string;
  timestamp: number;
};

async function multiAgentApproach(task: string): Promise<AgentResult> {
  const messages: AgentMessage[] = [];
  let totalTokens = 0;
  let totalToolCalls = 0;

  const researchResult = await researcher.run(task);
  messages.push({
    from: "researcher",
    to: "coder",
    content: researchResult.content,
    timestamp: Date.now(),
  });
  totalTokens += researchResult.tokensUsed;
  totalToolCalls += researchResult.toolCalls;

  const coderInput = messages
    .filter((m) => m.to === "coder")
    .map((m) => `[From ${m.from}]: ${m.content}`)
    .join("\n");

  const codeResult = await coder.run(coderInput);
  messages.push({
    from: "coder",
    to: "reviewer",
    content: codeResult.content,
    timestamp: Date.now(),
  });
  totalTokens += codeResult.tokensUsed;
  totalToolCalls += codeResult.toolCalls;

  const reviewerInput = messages
    .filter((m) => m.to === "reviewer")
    .map((m) => `[From ${m.from}]: ${m.content}`)
    .join("\n");

  const reviewResult = await reviewer.run(reviewerInput);
  messages.push({
    from: "reviewer",
    to: "orchestrator",
    content: reviewResult.content,
    timestamp: Date.now(),
  });
  totalTokens += reviewResult.tokensUsed;
  totalToolCalls += reviewResult.toolCalls;

  return {
    content: messages.map((m) => `[${m.from} -> ${m.to}]: ${m.content}`).join("\n\n"),
    tokensUsed: totalTokens,
    toolCalls: totalToolCalls,
  };
}
```

每个代理仅接收发送给它的消息。没有上下文污染。研究人员阅读的 50k 文档永远不会进入审阅者的上下文。

### 第 4 步：比较

```typescript
async function compare() {
  const task = "Build a rate limiter middleware for an Express.js API";

  console.log("=== Single Agent ===");
  const single = await singleAgentApproach(task);
  console.log(`Tokens: ${single.tokensUsed}`);
  console.log(`Tool calls: ${single.toolCalls}`);

  console.log("\n=== Multi-Agent ===");
  const multi = await multiAgentApproach(task);
  console.log(`Tokens: ${multi.tokensUsed}`);
  console.log(`Tool calls: ${multi.toolCalls}`);
}
```

多代理版本使用更多的总令牌（三个代理，三个单独的 LLM 调用），但每个代理的上下文保持干净。由于系统提示的专业化，每个阶段的质量都得到了提高。

## 使用它

本课程提供了一个可重复使用的提示，用于决定何时使用多代理。请参阅 `outputs/prompt-多智能体-decision.md`。

## 练习

1. 添加第四个专家：一个“测试人员”代理，负责接收编码人员的代码并审查审阅者的反馈，然后编写测试
2. 修改管道，以便审阅者可以将反馈发送回编码器以进行修订循环（最多 2 轮）
3. 将顺序管道转换为扇出：并行运行研究人员和“需求分析器”代理，然后在传递给编码器之前合并它们的输出

## 关键术语

| 学期 | 人们怎么说 | 它实际上意味着什么 |
|------|----------------|----------------------|
| 蜂群 | “人工智能代理的蜂巢思维” | 一组具有共享状态且没有固定领导者的对等代理。行为是从局部互动中产生的。 |
| 协调者 | 《老板经纪人》 | 其工具包括生成和管理其他代理的代理。它计划并授权，但可能不做实际工作。 |
| 协调员 | 《交通警察》 | 非代理组件（通常只是代码，不是 LLM），根据规则在代理之间路由消息。 |
| 共识 | “代理人同意” | 多个代理必须在继续之前达成一致的协议。当冲突的输出需要解决时使用。 |
| 突发行为 | “经纪人自己解决了这个问题” | 由代理交互产生但未明确编程的系统级模式。可能有用，也可能有害。 |
| 扇出/扇入 | “代理的地图缩减” | 将任务拆分为多个并行代理（扇出），然后组合它们的结果（扇入）。 |
| 消息传递 | “代理人互相交谈” | 代理之间的通信机制：从一个代理发送到另一个代理的结构化数据，取代共享上下文窗口。 |

## 进一步阅读

- [新兴人工智能代理架构的前景](https://arxiv.org/abs/2409.02977) - 多代理模式调查
- [AutoGen：启用下一代 LLM 应用程序](https://arxiv.org/abs/2308.08155) - 微软的多代理对话框架
- [Claude 代码子代理文档](https://docs.anthropic.com/en/docs/claude-code) - Claude 代码如何使用任务进行委托
- [CrewAI 文档](https://docs.crewai.com/) - 基于角色的多代理框架
