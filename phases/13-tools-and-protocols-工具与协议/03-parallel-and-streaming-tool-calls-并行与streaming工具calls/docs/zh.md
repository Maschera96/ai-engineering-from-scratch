# 并行工具调用与工具 streaming

> 三次独立的天气查询串行执行就是三次往返。并行运行它们，总时间就收缩到最慢的那一次调用。如今每个前沿厂商都能在单轮里发出多个工具调用。收益是实实在在的；但管线很微妙。本课会走完两个部分：并行的扇出（fan-out），以及流式参数的重组，重点关注 id 关联陷阱。

**类型：** 构建
**语言：** Python（标准库，线程池 + streaming 测试框架）
**前置：** Phase 13 · 02（function calling 深入）
**时长：** 约 75 分钟

## 学习目标

- 解释 `parallel_tool_calls: true` 为何存在，以及何时应该禁用它。
- 在并行扇出过程中，把流式参数块关联到正确的工具调用 id。
- 在不提前解析的前提下，把部分 `arguments` 字符串重组成完整的 JSON。
- 运行一个三城市天气基准测试，演示串行与并行的延迟差异。

## 问题

没有并行调用时，一个回答“班加罗尔、东京、苏黎世的天气如何”的智能体会这样做：

```
user -> LLM
LLM -> call get_weather(Bengaluru)
host -> run executor, reply with result
LLM -> call get_weather(Tokyo)
host -> run executor, reply with result
LLM -> call get_weather(Zurich)
host -> run executor, reply with result
LLM -> final text answer
```

三次 LLM 往返，每次还要承担执行器的延迟。大约是理想墙钟时间的 4 倍。

有了并行调用：

```
user -> LLM
LLM -> call get_weather(Bengaluru); call get_weather(Tokyo); call get_weather(Zurich)
host -> run all three executors concurrently, reply with three results
LLM -> final text answer
```

一次 LLM 往返。执行器时间是三者的最大值，而不是总和。OpenAI、Anthropic 和 Gemini 上的生产基准显示，扇出工作负载的墙钟时间减少了 60% 到 70%。

代价是关联复杂度。当三次调用乱序完成时，你的结果必须携带匹配的 `tool_call_id`，模型才能把它们对上。当结果以流式返回时，你必须先把部分参数片段拼成完整的 JSON 再执行。Gemini 3 引入唯一 id，部分原因正是要解决一个现实问题：对同一个工具的两次并行调用此前无法区分。

## 概念

### 启用并行

- **OpenAI。** `parallel_tool_calls: true` 默认开启。设为 `false` 可强制串行。
- **Anthropic。** 通过 `disable_parallel_tool_use: false` 启用并行（Claude 3.5 及以上默认开启）。设为 `true` 可串行。
- **Gemini。** 始终具备并行能力；`tool_config.function_calling_config.mode = "AUTO"` 让模型自行决定。

当工具存在顺序依赖时（先 `create_file` 再 `write_file`）、当一次调用的输出会影响另一次调用的输入时，或当限流器无法承受扇出时，应禁用并行。

### id 关联

模型发出的每一次调用都带有 `id`。host 返回的每一个结果都必须包含相同的 id。否则结果就会有歧义。

- **OpenAI。** 每条 tool 角色消息上的 `tool_call_id`。
- **Anthropic。** 每个 `tool_result` 块上的 `tool_use_id`。
- **Gemini。** 每个 `functionResponse` 上的 `id`（Gemini 3 及以上；Gemini 2 按名称匹配，这对同名并行调用会失效）。

### 并发运行调用

host 在各自的线程、协程或远程 worker 上运行每次调用的执行器。最简单的测试框架使用线程池；生产环境使用 asyncio 配合 `asyncio.gather` 或结构化并发。完成顺序是不可预测的——id 才是标识符。

一个常见的 bug：按调用列表顺序而非完成顺序返回结果。这通常能正常工作，因为模型只关心 `tool_call_id`，但如果某个结果被丢弃或重复，乱序提交会让调试更困难。最好按完成顺序、带显式 id 返回。

### 流式工具调用

当模型以流式输出时，`arguments` 会分片到达。三次并行调用的三股独立分片流会在传输线路上交错。你需要为每个 id 准备一个累加器。

各厂商的结构：

- **OpenAI。** 每个 chunk 是 `choices[0].delta.tool_calls[i].function.arguments`（部分字符串）。chunk 携带 `index`（在调用列表中的位置）。你按 index 累加，在 `id` 首次出现时读取它，并在 `finish_reason = "tool_calls"` 时解析 JSON。
- **Anthropic。** 流事件依次为 `message_start`，然后每个块一个 `content_block_start`，类型为 `tool_use`（含 id、name、空 input）。`content_block_delta` 事件携带 `input_json_delta` 分片。`content_block_stop` 关闭每个块。
- **Gemini。** `streamFunctionCallArguments`（Gemini 3 及以上）发出的分片带有 `functionCallId`，因此各调用能干净地交错。在 Gemini 3 之前，流式一次只返回一个完整调用。

### 部分 JSON 与提前解析陷阱

在 `arguments` 完整之前你不能解析它。诸如 `{"city": "Beng` 这样的部分 JSON 是无效的，会抛出异常。正确的判定关口是厂商的调用结束信号：OpenAI 的 `finish_reason = "tool_calls"`、Anthropic 的 `content_block_stop`，或 Gemini 的流结束事件。只有到那时才尝试 `json.loads`。更稳健的做法是使用增量式 JSON 解析器，在结构完成时逐步产出事件；OpenAI 的 streaming 指南推荐用它来实现显示实时“思考中”指示器的 UX。括号计数作为完整性测试并不可靠（引号字符串内或转义内容中的括号会导致误判），只应作为非正式的调试启发式。

### 乱序完成

```
call_A: fast API, returns first
call_B: slow API, returns second
call_C: median API, returns third
```

host 的回复仍然必须引用 id：

```
[{role: "tool", tool_call_id: "call_A", content: ...},
 {role: "tool", tool_call_id: "call_B", content: ...},
 {role: "tool", tool_call_id: "call_C", content: ...}]
```

在 OpenAI 或 Anthropic 上，回复中的顺序对正确性没有影响。只要 id 匹配，Gemini 接受任意顺序。

### 基准测试：串行与并行

`code/main.py` 中的测试框架模拟了三个延迟分别为 400、600、800 毫秒的执行器。串行运行总共耗时 1800 毫秒。并行运行耗时 max(400, 600, 800) = 800 毫秒。差距是恒定的，而非成比例的，所以节省的时间会随工具数量增长。

现实告诫：并行调用会给下游 API 带来压力。对一个有限流的服务做 10 路扇出会失败。Phase 13 · 17 讲解网关级别的背压；重试语义计划放在后续阶段。

### 流式扇出的墙钟时间

如果模型本身以流式输出，你可以在某次调用的参数一完整就开始执行，而不必等所有调用都最终确定。这是 OpenAI 记录的一项优化，但并非所有 SDK 都暴露它。本课的测试框架就这样做：模拟流一产出完整的参数对象，host 就启动那次调用。

## 动手用

`code/main.py` 有两个部分。第一部分用 `concurrent.futures.ThreadPoolExecutor` 串行和并行地运行三次模拟天气调用，并打印墙钟时间。第二部分回放一个伪造的流式响应——三次并行调用的 `arguments` 分片在同一条流上交错——并用 `StreamAccumulator` 按 id 重组它们。没有 LLM，没有网络，只有重组逻辑。

要关注的内容：

- 串行计时器达到 1.8 秒。在相同的伪造延迟下，并行计时器达到 0.8 秒。
- 累加器通过按 id 缓冲来处理乱序到达的分片，并仅在每次调用的 JSON 完整时才解析。
- 执行器在某个 id 的参数确定后立即启动，而不是等所有流都结束。

## 交付

本课产出 `outputs/skill-parallel-call-safety-check.md`。给定一个工具注册表，该 skill 审计哪些工具可以安全地并行化、哪些有顺序依赖、哪些会压垮下游限流——返回一份带有每个工具 `parallel_safe` 标志的修订后注册表。

## 练习

1. 运行 `code/main.py` 并改变模拟延迟。确认并行与串行的比值约为 `max/sum`（由于线程调度、序列化和测试框架开销，实际运行会略微偏离理想值）。在什么样的延迟分布下，并行就不再重要了？

2. 扩展累加器以处理“调用在流中途被取消”的情况，做法是丢弃它的缓冲并发出一个 `cancelled` 事件。哪个厂商明确记录了这种情况？查看 Anthropic 的 `content_block_stop` 语义和 OpenAI 的 `finish_reason: "length"` 行为。

3. 用 `asyncio.gather` 替换线程池。对两者做基准测试。由于上下文切换成本更低，你应该会在异步上看到小幅优势，但前提是执行器进行真实的 I/O。

4. 选两个不应并行化的工具（例如先 `create_file` 再 `write_file`）。给注册表添加一个 `ordering_dependency` 图，并基于该图对并行扇出加以限制。这是依赖感知调度的最小机制，后续的某个智能体工程阶段会将其形式化。

5. 阅读 OpenAI 的 parallel-function-calling 章节和 Anthropic 的 `disable_parallel_tool_use` 文档。找出 Anthropic 建议禁用并行的那一种现实工具类型。（提示：对同一资源的有后果的变更。）

## 关键术语

| 术语 | 人们怎么说 | 实际含义 |
|------|----------------|------------------------|
| 并行工具调用 | “一轮内扇出” | 模型在单条 assistant 消息中发出多个工具调用 |
| `parallel_tool_calls` | “OpenAI 的开关” | 启用或禁用多调用发出 |
| `disable_parallel_tool_use` | “Anthropic 的反向开关” | 退出标志；默认启用并行 |
| 工具调用 id | “关联句柄” | 每次调用的标识符，结果消息必须回传它 |
| 累加器 | “流缓冲” | 为每个 id 缓冲部分 `arguments` 分片的字符串缓冲区 |
| 乱序完成 | “最快的先来” | 并行调用以不可预测的顺序完成；id 是黏合剂 |
| 依赖图 | “顺序约束” | 其输出会喂入其他工具输入的工具；不能并行化 |
| 提前解析陷阱 | “JSON.parse 炸了” | 试图解析一个不完整的 `arguments` 字符串 |
| `streamFunctionCallArguments` | “Gemini 3 特性” | 每次调用带唯一 id 的流式参数分片 |
| 按完成顺序回复 | “别等所有的” | 结果到达即回复，以 id 为键 |

## 延伸阅读

- [OpenAI — Parallel function calling](https://platform.openai.com/docs/guides/function-calling#parallel-function-calling) — 默认行为与退出标志
- [Anthropic — Tool use: implementing tool use](https://docs.anthropic.com/en/docs/agents-and-tools/tool-use/implementing-tool-use) — `disable_parallel_tool_use` 与结果批处理
- [Google — Gemini function calling parallel section](https://ai.google.dev/gemini-api/docs/function-calling) — 来自 Gemini 3 的 id 关联并行调用
- [OpenAI — Streaming responses with tools](https://platform.openai.com/docs/api-reference/responses-streaming) — OpenAI 流的分片参数重组
- [Anthropic — Streaming messages](https://docs.anthropic.com/en/api/messages-streaming) — 带 `input_json_delta` 的 `content_block_delta`
