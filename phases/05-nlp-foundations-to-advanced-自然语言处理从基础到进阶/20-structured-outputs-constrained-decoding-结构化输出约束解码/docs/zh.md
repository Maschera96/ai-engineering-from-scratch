# 结构化输出与约束解码

> 说明：Ask an LLM 面向 JSON. Get JSON most of the time. In 生产, "most" is the problem. Constrained decoding turns "most" 到 "always" by editing the logits before sampling.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 17 (Chatbots), Phase 5 · 19 (Subword 分词)
**时间:** 约60分钟

## 问题

说明：一个 classifier prompts an LLM: "Return one of {positive, negative, neutral}." The 模型 returns "The sentiment is positive，this review is overwhelmingly favorable 因为 the customer explicitly states 这 they ...". Your parser crashes. Your classifier's F1 is 0.0.

说明：Free-form 生成 is not a contract. It is a suggestion. A 生产 系统 needs a contract.

Three layers exist in 2026.

1. 说明：**Prompting.** Ask nicely. "Return only the JSON object." Works ~80% on frontier models, less on smaller ones.
2. 说明：**Native structured 输出 APIs.** OpenAI `response_format`, Anthropic tool use, Gemini JSON mode. Reliable on supported schemas. Vendor-locked.
3. **Constrained decoding.** Modify the logits at every 生成 step so the 模型 *cannot* emit invalid 词元. 100% 有效 by construction. Works on any local 模型.

本课 builds intuition 面向 all three 与 names when to reach 面向 这.

## 概念

说明：![说明：Constrained decoding masking invalid 词元 at each step](../assets/constrained-decoding.svg)

**How constrained decoding works.** At each 生成 step, the LLM produces a logit 向量 over the full vocabulary (~100k 词元). A *logit processor* sits between the 模型 与 the sampler. It computes 这 词元 are 有效 given the 当前 position in the target grammar，JSON Schema, regex, context-free grammar，与 sets the logits of all invalid 词元 to negative infinity. The softmax over the remaining logits puts 概率 mass only on 有效 continuations.

Implementations in 2026:

- **Outlines.** Compiles JSON Schema 或 regex 到 a finite-状态 machine. Every 词元 gets an O(1) 有效-next-词元 lookup. FSM-based, so recursive schemas need flattening.
- 说明：**XGrammar / llguidance.** Context-free grammar engines. Handle recursive JSON Schema. Near-zero decoding overhead. OpenAI credited llguidance in their 2025 structured 输出 implementation.
- 说明：**vLLM guided decoding.** Built-in `guided_json`, `guided_regex`, `guided_choice`, `guided_grammar` via Outlines, XGrammar, 或 lm-format-enforcer backends.
- 说明：**Instructor.** Pydantic-based wrapper over any LLM. Retries on validation failure. Cross-provider, but does not modify logits，it relies on retries + structured-输出-aware prompts.

### 这个 counterintuitive result

Constrained decoding is often *faster* than unconstrained 生成. Two reasons. First, it shrinks the next-词元 search space. Second, clever implementations skip 词元 生成 entirely 面向 forced 词元 (scaffolding like `{"name": "`，every byte is determined).

### 这个 pitfall 这 costs you

Field order matters. Put `answer` before `reasoning`, 与 the 模型 commits to an 答案 before it thinks. JSON is 有效. Answer is 错误. No validation catches it.

```json
// BAD
{"answer": "yes", "reasoning": "because ..."}

// GOOD
{"reasoning": "... therefore ...", "answer": "yes"}
```

说明：Schema field order is logic, not formatting.

## 动手构建

### Step 1: regex-constrained 生成 从 scratch

说明：See `code/main.py` 面向 a standalone FSM implementation. The core idea in 30 lines:

```python
def mask_logits(logits, valid_token_ids):
    mask = [float("-inf")] * len(logits)
    for tid in valid_token_ids:
        mask[tid] = logits[tid]
    return mask


def generate_constrained(model, tokenizer, prompt, fsm):
    ids = tokenizer.encode(prompt)
    state = fsm.initial_state
    while not fsm.is_accept(state):
        logits = model.next_token_logits(ids)
        valid = fsm.valid_tokens(state, tokenizer)
        logits = mask_logits(logits, valid)
        tok = sample(logits)
        ids.append(tok)
        state = fsm.transition(state, tok)
    return tokenizer.decode(ids)
```

这个 FSM tracks what parts of the grammar we have satisfied so far. `valid_tokens(state, tokenizer)` computes 这 vocabulary 词元 can advance the FSM 不使用 leaving an accepting path.

### Step 2: Outlines 面向 JSON Schema

```python
from pydantic import BaseModel
from typing import Literal
import outlines


class Review(BaseModel):
    sentiment: Literal["positive", "negative", "neutral"]
    confidence: float
    evidence_span: str


model = outlines.models.transformers("meta-llama/Llama-3.2-3B-Instruct")
generator = outlines.generate.json(model, Review)

result = generator("Classify: 'The wait staff was attentive and the food arrived hot.'")
print(result)
# Review(sentiment='positive', confidence=0.93, evidence_span='attentive ... hot')
```

说明：Zero validation errors. Ever. The FSM makes invalid 输出 unreachable.

### 说明：Step 3: Instructor 面向 provider-agnostic Pydantic

```python
import instructor
from anthropic import Anthropic
from pydantic import BaseModel, Field


class Invoice(BaseModel):
    vendor: str
    total_usd: float = Field(ge=0)
    line_items: list[str]


client = instructor.from_anthropic(Anthropic())
invoice = client.messages.create(
    model="claude-opus-4-7",
    max_tokens=1024,
    response_model=Invoice,
    messages=[{"role": "user", "content": "Extract from: 'Acme Corp $420. Widget, Gizmo.'"}],
)
```

Different mechanism. Instructor does not touch logits. It formats the 模式 到 the prompt, parses the 输出, 与 retries on validation failure (默认 3 times). Works 使用 any provider. Retries add 延迟 与 成本. Cross-provider portability is the selling point.

### Step 4: native vendor APIs

```python
from openai import OpenAI

client = OpenAI()
response = client.responses.create(
    model="gpt-5",
    input=[{"role": "user", "content": "Classify: 'The food was cold.'"}],
    text={"format": {"type": "json_schema", "name": "sentiment",
          "schema": {"type": "object", "required": ["sentiment"],
                     "properties": {"sentiment": {"type": "string",
                                                  "enum": ["positive", "negative", "neutral"]}}}}},
)
print(response.output_parsed)
```

说明：Server-side constrained decoding. Reliability parity 使用 Outlines 面向 supported schemas. No local 模型 management. Locks you to the vendor.

## 陷阱

- 说明：**Recursive schemas.** Outlines flattens recursion to a 固定 depth. Tree-structured outputs (nested comments, AST) need XGrammar 或 llguidance (CFG-based).
- 说明：**Huge enums.** 10,000-option enum compiles slowly 或 times out. Switch to a retriever: predict top-k candidates first, constrain to those.
- **Grammar too strict.** Force `date: "YYYY-MM-DD"` regex 与 the 模型 cannot 输出 `"unknown"` 面向 missing dates. 模型 compensates by inventing a date. Allow `null` 或 a sentinel.
- 说明：**Premature commitment.** See field-order pitfall above. Always put reasoning first.
- **Vendor JSON mode 不使用 模式.** Pure JSON mode only guarantees 有效 JSON, not 有效 *面向 your use case*. Always provide a full 模式.

## 投入使用

这个 2026 stack:

|Situation|Pick|
|-----------|------|
|OpenAI/Anthropic/Google 模型, simple 模式|Native vendor structured 输出|
|说明：Any provider, Pydantic workflow, can tolerate retries|Instructor|
|Local 模型, need 100% validity, flat 模式|Outlines (FSM)|
|Local 模型, recursive 模式|XGrammar 或 llguidance|
|Self-hosted 推理 server|vLLM guided decoding|
|Batch processing 使用 retries acceptable|Instructor + cheapest 模型|

## 交付成果

保存为 `outputs/skill-structured-output-picker.md`:

```markdown
---
name: structured-output-picker
description: Choose a structured output approach, schema design, and validation plan.
version: 1.0.0
phase: 5
lesson: 20
tags: [nlp, llm, structured-output]
---

Given a use case (provider, latency budget, schema complexity, failure tolerance), output:

1. Mechanism. Native vendor structured output, Instructor retries, Outlines FSM, or XGrammar CFG. One-sentence reason.
2. Schema design. Field order (reasoning first, answer last), nullable fields for "unknown", enum vs regex, required fields.
3. Failure strategy. Max retries, fallback model, graceful `null` handling, out-of-distribution refusal.
4. Validation plan. Schema compliance rate (target 100%), semantic validity (LLM-judge), field-coverage rate, latency p50/p99.

Refuse any design that puts `answer` or `decision` before reasoning fields. Refuse to use bare JSON mode without a schema. Flag recursive schemas behind an FSM-only library.
```

## 练习

1. **Easy.** Prompt a 小 开放-weights 模型 (e.g., Llama-3.2-3B) 不使用 constrained decoding 面向 `Review(sentiment, confidence, evidence_span)`. Measure the fraction 这 parse as 有效 JSON on 100 reviews.
2. **Medium.** Same 语料库 使用 Outlines JSON mode. Compare compliance rate, 延迟, 与 语义 准确率.
3. 说明：**Hard.** Implement a regex-constrained decoder 从 scratch 面向 phone numbers (`\d{3}-\d{3}-\d{4}`). Verify 0 invalid outputs on 1000 samples.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Constrained decoding|Force 有效 输出|Mask invalid-词元 logits at every 生成 step.|
|Logit processor|这个 thing 这 constrains|Function: `(logits, state) -> masked_logits`.|
|FSM|Finite-状态 machine|说明：Compiled grammar representation; O(1) 有效-next-词元 lookup.|
|CFG|Context-free grammar|说明：Grammar 这 handles recursion; slower but more expressive than FSM.|
|Schema field order|Does it matter?|说明：Yes，first field commits; always put reasoning before 答案.|
|Guided decoding|vLLM's name 面向 it|Same concept, integrated 到 the 推理 server.|
|JSON mode|OpenAI's early version|说明：Guarantees JSON syntax; does NOT guarantee 模式 match.|

## 延伸阅读

- 说明：[说明：Willard, Louf (2023). Efficient Guided Generation 面向 LLMs](https://arxiv.org/abs/2307.09702)，the Outlines paper.
- 说明：[XGrammar paper (2024)](https://arxiv.org/abs/2411.15100)，快 CFG-based constrained decoding.
- 说明：[vLLM，Structured Outputs](https://docs.vllm.ai/en/latest/features/structured_outputs.html)，推理 server integration.
- 说明：[OpenAI，Structured Outputs guide](https://platform.openai.com/docs/guides/structured-outputs)，API reference + gotchas.
- 说明：[Instructor 库](https://python.useinstructor.com/)，Pydantic + retries across providers.
- 说明：[JSONSchemaBench (2025)](https://arxiv.org/abs/2501.10868)，benchmarking 6 constrained decoding frameworks.
