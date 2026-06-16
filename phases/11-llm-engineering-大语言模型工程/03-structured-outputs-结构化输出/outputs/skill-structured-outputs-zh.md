---
name: skill-structured-outputs-zh
description: Decision framework for choosing the right 结构化 输出 strategy based on provider, reliability, and complexity
version: 1.0.0
phase: 11
lesson: 03
tags: [structured-output, json, schema, constrained-decoding, pydantic, function-calling]
---

# 结构化 输出 Strategy

当building an LLM 应用 that requires 结构化 数据, apply this decision framework.

## When to use each approach

**Prompt-based ("Return JSON"):** Prototyping only. Acceptable for internal 工具 where occasional parse failures are tolerable. Add a try/except with retry. Never use in 生产 pipelines.

**JSON mode (API flag):** You need guaranteed 有效 JSON but the 模式 is simple or flexible. Works when you 验证 the shape on the 应用 side. Available: OpenAI, Anthropic (via 工具使用), Google.

**模式 mode (constrained decoding):** 生产 systems where every 输出 must match a specific 模式. Zero parse failures. Zero 模式 violations. Use this by default for any 生产 extraction or 分类 任务. Available: OpenAI 结构化输出, Outlines, Guidance.

**函数 calling / 工具使用:** The 模型 needs to choose which 函数 to call, not just fill 参数. You have multiple schemas and the 模型 selects the appropriate one. Also use when integrating with existing 工具/函数 infrastructure.

**Instructor library:** You want Pydantic 验证 with automatic retry across any provider. Best DX for Python projects. Wraps OpenAI, Anthropic, Google, and open-source 模型.

## Provider-specific guidance

**OpenAI:** Use `response_format` with `json_schema` type. Constrained decoding is built in. Pydantic 模型 work directly. Most reliable 结构化 输出 implementation.

**Anthropic:** Use 工具使用 for 结构化 输出. Define a single 工具 with the desired 模式. The 模型 returns 工具 call arguments 匹配 the 模式. Reliable but requires the 工具使用 API pattern.

**Open-source 模型 (vLLM, Ollama):** Use Outlines or Guidance for constrained decoding. These libraries compile JSON Schemas into finite 状态 machines that 掩码 无效 词元 during 生成. Requires running 推理 locally.

## 模式 design guidelines

1. Keep schemas flat when possible. Nested objects beyond 2 levels increase extraction 错误.
2. 使用enums for categorical fields. Do not rely on the 模型 inventing the right string.
3. Make ambiguous fields required with explicit null support rather than optional. Forces the 模型 to make a decision.
4. Add descriptions to 模式 properties. The 模型 reads these as instructions.
5. Avoid union types (oneOf/anyOf) unless necessary. They increase decoding complexity.
6. Set minimum/maximum on numbers. Catches hallucinated extreme values.
7. 使用minItems/maxItems on arrays to prevent empty or unbounded outputs.

## Common failure patterns and fixes

- **模型 wraps JSON in markdown fences**: switch from prompt-based to JSON mode or 模式 mode
- **Schema-valid but factually wrong**: add an LLM-as-judge 验证 步骤 after extraction
- **Inconsistent enum values**: switch to constrained decoding or add post-processing 归一化
- **Missing optional fields**: make them required or add default values in 应用 code
- **Very slow extraction**: constrained decoding adds 5-15% 延迟, reduce 模式 complexity if latency-sensitive
- **Large arrays with varied items**: 分块 the 输入 and extract per-chunk, then merge results

## Reliability ladder

|方法|Parse Success|模式 Match|Setup Effort|
|----------|-------------|-------------|-------------|
|Prompt-based|~90%|~80%|1 分钟|
|JSON mode|100%|~90%|5 分钟|
|模式 mode|100%|~99%|15 分钟|
|Constrained decoding|100%|100%|30 分钟|
|Instructor + retry|100%|~99.5%|10 分钟|
