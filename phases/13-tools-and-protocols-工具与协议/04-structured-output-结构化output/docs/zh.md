# Structured Output — JSON Schema, Pydantic, Zod, Constrained Decoding

> “礼貌地请求模型返回 JSON” 即便在前沿模型上也有 5% 到 15% 的失败率。结构化输出用约束解码（constrained decoding）填平了这道鸿沟：模型在字面意义上被阻止产出任何会违反模式的 token。OpenAI 的 strict 模式、Anthropic 的模式定型工具使用、Gemini 的 `responseSchema`、Pydantic AI 的 `output_type`，以及 Zod 的 `.parse` 是同一个理念的五种表层形态。本课构建学习者将在每一条生产抽取流水线中用到的模式验证器和 strict 模式契约。

**Type:** Build
**Languages:** Python (stdlib, JSON Schema 2020-12 subset)
**Prerequisites:** Phase 13 · 02 (function calling deep dive)
**Time:** ~75 minutes

## Learning Objectives

- 为某个抽取目标编写一份 JSON Schema 2020-12，并使用恰当的约束（enum、min/max、required、pattern）。
- 解释为什么 strict 模式和约束解码提供的保证不同于“生成后再验证”。
- 区分三种失败模式：解析错误、模式违规、模型拒答。
- 交付一条带有定型修复与定型拒答处理的抽取流水线。

## The Problem

一个读取采购订单邮件的智能体需要把自由文本转化为 `{customer, line_items, total_usd}`。有三种做法。

**做法一：用提示词请求 JSON。**“以 JSON 回复，包含字段 customer、line_items、total_usd。”在前沿模型上有 85% 到 95% 的成功率。会以六种方式失败：缺少花括号、尾随逗号、类型错误、幻觉出的字段、在 token 上限处被截断、泄露出诸如“这是你的 JSON：”之类的散文。

**做法二：生成后再验证。**自由生成，解析，对照模式验证，失败则重试。可靠但昂贵——你要为每次重试付费，而截断 bug 每出现一次就多花一个回合。

**做法三：约束解码。**提供方在解码时强制执行模式。无效的 token 会从采样分布中被掩蔽。输出保证可解析，也保证通过验证。失败收敛为单一模式：拒答（模型判定输入不适配该模式）。

每一家 2026 年的前沿提供方都提供了某种形式的做法三。

- **OpenAI。** `response_format: {type: "json_schema", strict: true}`，若模型拒绝则在响应中附带 `refusal`。
- **Anthropic。** 对 `tool_use` 输入做模式强制；`stop_reason: "refusal"` 并不存在，但带有 `end_turn` 而无工具调用即是信号。
- **Gemini。** 请求级的 `responseSchema`；在 2026 年，Gemini 为部分类型提供 token 级语法约束。
- **Pydantic AI。** `output_type=InvoiceModel` 会产出一个定型为 `InvoiceModel` 的结构化 `RunResult`。
- **Zod (TypeScript)。** 运行时解析器，对照 Zod 模式验证提供方输出；与 OpenAI 的 `beta.chat.completions.parse` 配对使用。

共同的主线：模式声明一次，端到端强制执行。

## The Concept

### JSON Schema 2020-12 — 通用语言

每家提供方都接受 JSON Schema 2020-12。你最常用的构件：

- `type`：`object`、`array`、`string`、`number`、`integer`、`boolean`、`null` 之一。
- `properties`：字段名到子模式的映射。
- `required`：必须出现的字段名列表。
- `enum`：允许值的封闭集合。
- `minimum` / `maximum`（数字），`minLength` / `maxLength` / `pattern`（字符串）。
- `items`：应用于每个数组元素的子模式。
- `additionalProperties`：`false` 禁止额外字段（默认值因模式而异）。

OpenAI 的 strict 模式额外加了三条要求：每个属性都必须列入 `required`、处处都要 `additionalProperties: false`、不允许未解析的 `$ref`。如果你违反这些，API 会在请求时返回 400。

### Pydantic，Python 绑定

Pydantic v2 通过 `model_json_schema()` 从 dataclass 形状的模型生成 JSON Schema。Pydantic AI 对此做了封装，于是你只需写：

```python
class Invoice(BaseModel):
    customer: str
    line_items: list[LineItem]
    total_usd: Decimal
```

智能体框架便会在边界处把该模式翻译成 OpenAI strict 模式、Anthropic `input_schema` 或 Gemini `responseSchema`。模型的输出会以定型的 `Invoice` 实例形式返回。验证错误会抛出 `ValidationError`，并带有定型的错误路径。

### Zod，TypeScript 绑定

Zod（`z.object({customer: z.string(), ...})`）是其 TS 等价物。OpenAI 的 Node SDK 暴露了 `zodResponseFormat(Invoice)`，它会翻译成 API 的 JSON Schema 载荷。

### Refusals

strict 模式无法强迫模型作答。如果输入无法适配模式（“这封邮件是一首诗，不是一张发票”），模型会产出一个 `refusal` 字段，其中包含原因。你的代码必须把它当作一等公民的结果来处理，而非一种失败。拒答也可作为安全信号：当模型被要求从受保护内容的邮件中抽取信用卡号时，它会返回一个附带安全原因的拒答。

### 开放权重中的约束解码

开放权重的实现使用三种技术。

1. **基于语法的解码**（`outlines`、`guidance`、`lm-format-enforcer`）：从模式构建一个确定性有限自动机；在每一步掩蔽掉会违反该 FSM 的 token 的 logits。
2. **配合 JSON 解析器的 logit 掩蔽**：让一个流式 JSON 解析器与模型同步推进；在每一步计算合法的下一 token 集合。
3. **带验证器的推测解码**：廉价的草稿模型提议 token，验证器强制执行模式。

商业提供方在幕后选用其中之一。2026 年的最新水平对于短的结构化输出比纯生成更快，对于长的输出则速度大致相同。

### 三种失败模式

1. **解析错误。**输出不是有效的 JSON。在 strict 模式下不会发生。在非 strict 的提供方上仍可能发生。
2. **模式违规。**输出可解析但违反了模式。在 strict 模式下不会发生。在其之外则很常见。
3. **拒答。**模型拒绝作答。必须作为定型结果来处理。

### 重试策略

当你处于 strict 模式之外（Anthropic 工具使用、非 strict 的 OpenAI、较旧的 Gemini）时，恢复模式是：

```
generate -> parse -> validate -> if fail, inject error and retry, max 3x
```

一次重试通常就足够。三次重试能兜住弱模型的偶发抖动。超过三次则是模式糟糕的信号：模型对某些输入无法满足它，需要修正提示词或模式。

### 小模型支持

约束解码在小模型上也有效。一个带语法强制的 3B 参数开放模型，在结构化任务上胜过一个仅靠原始提示词的 70B 参数模型。这正是结构化输出对生产至关重要的主要原因：它把可靠性与模型规模解耦了。

## Use It

`code/main.py` 提供了一个仅用 stdlib 实现的最小化 JSON Schema 2020-12 验证器（types、required、enum、min/max、pattern、items、additionalProperties）。它封装了一份 `Invoice` 模式，并把一段假的 LLM 输出送过验证器，演示解析错误、模式违规和拒答路径。在生产中，把假输出替换为任意提供方的真实响应即可。

要看的要点：

- 验证器返回一个带路径和消息的定型 `[ValidationError]` 列表。这正是你想暴露给重试提示词的形状。
- 拒答分支**不**重试。它记录日志并返回一个定型拒答。Phase 14 · 09 把拒答作为安全信号使用。
- `additionalProperties: false` 检查在对抗性测试输入上触发，展示了为什么 strict 模式对幻觉字段关上了门。

## Ship It

本课产出 `outputs/skill-structured-output-designer.md`。给定一个自由文本抽取目标（发票、支持工单、简历等），该技能产出一份与 strict 模式兼容的 JSON Schema 2020-12，以及一个与之镜像的 Pydantic 模型，并预置了定型拒答与重试处理的桩代码。

## Exercises

1. 运行 `code/main.py`。新增第四个测试用例，其 `total_usd` 为负数。确认验证器以 `minimum` 约束路径将其拒绝。

2. 扩展验证器以支持带判别器（discriminator）的 `oneOf`。常见情形：`line_item` 要么是产品要么是服务，由 `kind` 标记。strict 模式在此处有一些微妙的规则；查阅 OpenAI 的结构化输出指南。

3. 把同一份 Invoice 模式写成一个 Pydantic BaseModel，并把 `model_json_schema()` 的输出与你手写的模式作对比。找出 Pydantic 默认设置而手写版本省略掉的那个字段。

4. 测量拒答率。构造十个不应可抽取的输入（一段歌词、一份数学证明、一封空白邮件），用一个真实的提供方在 strict 模式下跑一遍。统计拒答数与幻觉输出数。这就是你做拒答感知重试的真实基准。

5. 从头到尾通读 OpenAI 的结构化输出指南。找出它在 strict 模式中明确禁止、而纯 JSON Schema 却允许的那一个构件。然后设计一份非必要地使用该禁用构件的模式，并把它重构为 strict 兼容。

## Key Terms

| Term | What people say | What it actually means |
|------|----------------|------------------------|
| JSON Schema 2020-12 | “模式规范” | 每家现代提供方都通行的 IETF 草案模式方言 |
| Strict mode | “保证模式” | 通过约束解码强制执行模式的 OpenAI 标志 |
| Constrained decoding | “logit 掩蔽” | 解码时强制执行，掩蔽掉无效的下一 token |
| Refusal | “模型拒绝” | 输入无法适配模式时的定型结果 |
| Parse error | “无效 JSON” | 输出无法解析为 JSON；在 strict 下不可能 |
| Schema violation | “形状错误” | 可解析但违反了类型 / required / enum / 范围 |
| `additionalProperties: false` | “不许有额外字段” | 禁止未知字段；OpenAI strict 中必需 |
| Pydantic BaseModel | “定型输出” | 产出并验证 JSON Schema 的 Python 类 |
| Zod schema | “TypeScript 输出类型” | 用于验证提供方输出的 TS 运行时模式 |
| Grammar enforcement | “开放权重约束解码” | 基于 FSM 的 logit 掩蔽，如 outlines / guidance |

## Further Reading

- [OpenAI — Structured outputs](https://platform.openai.com/docs/guides/structured-outputs) — strict 模式、拒答与模式要求
- [OpenAI — Introducing structured outputs](https://openai.com/index/introducing-structured-outputs-in-the-api/) — 2024 年 8 月的发布博文，解释了解码保证
- [Pydantic AI — Output](https://ai.pydantic.dev/output/) — 序列化到各提供方的定型 output_type 绑定
- [JSON Schema — 2020-12 release notes](https://json-schema.org/draft/2020-12/release-notes) — 权威规范
- [Microsoft — Structured outputs in Azure OpenAI](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/structured-outputs) — 企业部署说明与 strict 模式的注意事项
