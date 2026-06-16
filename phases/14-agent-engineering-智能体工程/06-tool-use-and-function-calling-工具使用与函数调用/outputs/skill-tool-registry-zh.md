---
name: tool-registry-zh
description: 构建一个生产级工具目录与注册表，包含 JSON Schema 校验、并行分发和可观测性。
version: 1.0.0
phase: 14
lesson: 06
tags: [function-calling, tools, schema, validation, bfcl, parallel-tools]
---

给定一个任务领域，产出一个工具目录，使智能体能够在 BFCL V4 的各个维度（agentic、multi-turn、live、non-live、hallucination）上可靠地使用这些工具。

产出内容：

1. 工具定义。对每个工具给出：`name`（snake_case）、`description`（告诉模型何时该用它、何时不该用）、带类型属性的 JSON Schema 输入、必填字段、适用时的枚举值、数值的 minimum/maximum、每个工具的超时时间、每个工具的沙箱策略（fs 访问面、网络、内存上限）。
2. 描述质量检查。让每条描述都通过这一拷问："它是否告诉模型在何时该选这个工具而非其他工具？" 如果两个工具的描述有重叠，则拒绝并重写。
3. 并行分发计划。对每个真实任务，识别哪些工具调用是相互独立的（可以并行）、哪些必须顺序执行。输出一张预期的分发图。
4. 校验策略。枚举检查、类型强制转换规则（例如"接受 int-as-string，拒绝 float-as-string"）、必填字段强制。每一次失败都返回一个结构化的观察字符串，绝不向循环抛出异常。
5. 可观测性。每个工具发出一个 OpenTelemetry GenAI `tool_call` span，带有属性 `gen_ai.tool.name`、`gen_ai.tool.call.id`、`gen_ai.tool.call.arguments`、`gen_ai.tool.call.result`（当内容策略要求时，使用引用而非内联）。

硬性拒绝项：

- 通用的 shell/命令执行工具。拒绝并拆分为具体的动词（`git_status`、`fs_read`、`npm_test`）。
- 当参数取值是一个封闭集合时却缺少枚举。枚举校验是捕捉漂移最廉价的方式。
- 两个不同工具用相同的描述。模型无法可靠地在它们之间做选择。
- 只写了工具名的 `description`（"将两个数字相加"）。要包含何时选它而非其他备选项。
- 没有超时。每个工具调用都必须有上限。

拒绝规则：

- 如果单个智能体的工具列表超过 30 个工具，拒绝并建议采用子智能体委派（第 17 课）。
- 如果任何工具在没有确认门控的情况下执行破坏性操作，拒绝并指向第 09 课（权限、沙箱）。
- 如果任务是计算机操作（点击、输入、截图），拒绝并指向第 21 课——那是一种独立的工具形态，使用基于视觉的动作。

输出：一份可直接粘贴进 Anthropic / OpenAI / Gemini SDK 调用的 JSON 工具目录、一张分发图、一份校验策略文档，以及一个注册表应当通过的 BFCL 风格迷你评估。

以一个"接下来读什么"的指引结尾：第 09 课（沙箱）、第 23 课（OTel GenAI spans）或第 30 课（评估驱动）。
