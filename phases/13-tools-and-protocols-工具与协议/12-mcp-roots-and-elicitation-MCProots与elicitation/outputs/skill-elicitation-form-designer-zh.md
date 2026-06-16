---
name: elicitation-form-designer-zh
description: 为需要在调用过程中进行用户确认或消歧的工具设计 elicitation 表单模式与消息模板。
version: 1.0.0
phase: 13
lesson: 12
tags: [mcp, elicitation, user-input, forms]
---

给定一个行为可能需要在调用过程中获取用户输入的工具，设计其 elicitation 模式与消息。

产出：

1. 触发条件。明确指出哪种输入或歧义应该导致工具调用 `elicitation/create`。
2. 消息模板。展示给用户的一句话，由宿主呈现。朴实、具体、不含术语。
3. 模式。扁平的 JSON Schema，包含带类型的属性以及 `enum` 列表（用于消歧）或 `boolean`（用于确认）。不要嵌套。
4. 分支处理。将 `accept` / `decline` / `cancel` 映射到工具行为。
5. 限流规则。限制每次工具调用的 elicitation 次数；切勿在循环内进行 elicitation。

硬性拒绝：
- 任何嵌套对象的模式。Elicitation v1 是扁平的。
- 任何用于补足 LLM 本可以通过文字提问获取的缺失参数的 elicitation。
- 任何高频 elicitation（每次工具调用超过一次）。

拒绝规则：
- 如果工具是只读且低风险的，拒绝进行 elicitation，直接返回结果。
- 如果工具具有破坏性，且宿主支持 `destructiveHint` 注解，建议使用注解并让客户端原生处理确认。
- 如果需求是 OAuth 登录，推荐使用 URL 模式 elicitation，并标记 SEP-1036 的偏移风险。

输出：一页式设计，包含触发条件、消息模板、模式、分支处理、限流规则，以及一条关于表单模式还是 URL 模式更合适的说明。
