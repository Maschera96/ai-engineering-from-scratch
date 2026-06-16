---
name: skill-function-calling-patterns-zh
description: Decision framework for implementing 函数调用 in 生产 -- 工具 design, 错误 handling, security, and provider patterns
version: 1.0.0
phase: 11
lesson: 09
tags: [function-calling, tool-use, agents, mcp, security, openai, anthropic]
---

# 函数调用 Patterns

当building an LLM 应用 that uses 工具, apply this decision framework.

## When to use 函数调用

**Use 函数调用 when:**
- 这个模型 needs 实时 数据 (weather, stock prices, database 查询)
- 这个任务 requires side effects (sending emails, creating records, deploying code)
- 这个模型 must choose between multiple actions based on 用户 intent
- 你are building an 智能体 that interacts with external systems

**Use 结构化输出 instead when:**
- 你need 数据 extraction from 文本 (no external calls needed)
- 这个输出 is the final product, not an intermediate 步骤
- 你have a single 模式, not multiple 工具 to choose from

**Use both when:**
- 这个模型 calls a 工具, then structures the 工具 result into a specific 输出 format

## 工具 design guidelines

1. **One 工具, one 动作.** A 工具 named `manage_database` that handles 查询, inserts, updates, and deletes is too broad. Split into `query_records`, `insert_record`, `update_record`. The 模型 selects better with specific 工具.

2. **Descriptions are prompts.** The 模型 reads 工具 descriptions to decide selection. Write them like you would write instructions for a junior developer. Include what the 工具 returns, not just what it does.

3. **Constrain with enums.** If a 参数 has 3-10 有效 values, use an enum. The 模型 will invent strings -- "celsius", "Celsius", "C", "指标" -- unless you constrain it.

4. **Fewer 工具 is better.** GPT-4o handles 5-10 工具 well. At 20+ 工具, selection accuracy drops. At 50+ 工具, expect 10-15% wrong 工具 selection. Group related functionality or use a 路由 层.

5. **Required means required.** Only mark a 参数 as required if the 工具 literally cannot 函数 without it. Optional 参数 with good defaults reduce 工具 call failures.

## Provider-specific patterns

### OpenAI (GPT-4o, o3, GPT-4o-mini)

```python
tools=[{"type": "function", "function": {"name": ..., "parameters": ...}}]
tool_choice="auto"       # model decides
tool_choice="required"   # must call at least one tool
tool_choice={"type": "function", "function": {"name": "specific_tool"}}
```

- Supports 并行 工具 calls (multiple `tool_calls` in one 响应)
- 工具 call IDs must be passed back with results
- `gpt-4o-mini` is 10x cheaper and handles simple 工具 路由 well
- 结构化 outputs mode works with 工具 参数 for guaranteed 模式 compliance

### Anthropic (Claude 3.5 Sonnet, Claude 4 Opus)

```python
tools=[{"name": ..., "description": ..., "input_schema": ...}]
tool_choice={"type": "auto"}     # model decides
tool_choice={"type": "any"}      # must call at least one tool
tool_choice={"type": "tool", "name": "specific_tool"}
```

- 工具 calls appear as content 块 with `type: "tool_use"`
- Results go in 用户 消息 with `type: "tool_result"`
- Field name is `input_schema`, not `parameters` (common migration bug)
- Supports multiple 工具 calls per 响应

### Google (Gemini 2.0 Flash, Gemini 2.0 Pro)

```python
function_declarations=[{"name": ..., "description": ..., "parameters": ...}]
function_calling_config={"mode": "AUTO"}   # or "ANY" or "NONE"
```

- Uses `function_declarations` at the top level
- Results returned via `function_response` parts
- Supports 并行 函数调用

### Open-source 模型 (Llama 3, Hermes, Qwen)

- No standardized format -- varies by 模型 and serving framework
- Hermes format (NousResearch) is the most common fine-tuned convention
- vLLM supports OpenAI-compatible 工具 calling for supported 模型
- Ollama supports basic 工具 calling with compatible 模型
- Test 工具 selection accuracy before 生产 -- 开放 模型 are 15-30% less accurate than GPT-4o on the Berkeley 函数调用 Leaderboard

## 错误 handling patterns

### Return 结构化 错误

```json
{"error": true, "message": "City 'Toky' not found. Did you mean 'Tokyo'?", "code": "NOT_FOUND", "suggestions": ["Tokyo"]}
```

Include actionable information. "Not found" is bad. "Not found, did you mean X?" is good. The 模型 uses 错误 消息 to self-correct.

### Retry strategy

1. 工具 call fails with a correctable 错误 (typo, wrong enum value)
2. Send the 错误 back to the 模型 as a 工具 result
3. 这个模型 adjusts and retries
4. Maximum 3 retries per 工具 call
5. After 3 failures, return the 错误 to the 用户

### 时间out handling

Set timeouts on all 工具 executions. 30 seconds is a reasonable default. If a 工具 times out, return a 结构化 timeout 错误 so the 模型 can inform the 用户 rather than hanging.

## Security checklist

|Check|Why|How|
|-------|-----|-----|
|Allowlist 函数|Prevent arbitrary code execution|Only register 工具 the 用户 needs|
|验证 argument types|Prevent type confusion attacks|Check types before execution|
|Sanitize string arguments|Prevent injection|Reject or escape special characters|
|Parameterize database 查询|Prevent SQL injection|Never pass model-generated SQL directly|
|Filter 工具 results|Prevent 数据 leakage|Remove API keys, PII, internal 错误|
|速率 限制 工具 calls|Prevent runaway loops|Max 10-20 calls per conversation|
|Log all 工具 calls|Audit trail|Store 工具 name, arguments, result, timestamp|
|块 path traversal|Prevent file 系统 access|Reject `..` and absolute paths in file 工具|
|Sandbox code execution|Prevent 系统 access|Use containers or restricted builtins|
|验证 return size|Prevent 上下文 stuffing|Truncate results over 10KB|

## Performance 优化

- **并行 calls:** When the 模型 requests multiple independent 工具, execute them concurrently with `asyncio.gather()` or `concurrent.futures`
- **缓存:** 缓存 工具 results for identical arguments within the same session (weather does not change in 60 seconds)
- **Streaming:** Stream the 模型's final 响应 while 工具 results are being fetched
- **工具 pruning:** If 上下文 is tight, only include 工具 definitions relevant to the current 查询 (use a 分类器 to filter)
- **Smaller 模型 for 路由:** Use `gpt-4o-mini` or `claude-3-5-haiku` for 工具 selection, then pass results to a stronger 模型 for synthesis

## Common failure patterns

|Failure|Cause|修复|
|---------|-------|-----|
|Wrong 工具 selected|Ambiguous descriptions|Rewrite descriptions with specific trigger words|
|Missing required args|模型 forgot a 参数|Add clear examples in 参数 descriptions|
|Infinite 工具 循环|模型 keeps calling same 工具|Set max iterations (5-10) and detect repeated calls|
|Hallucinated arguments|模型 invents plausible but wrong values|Use enums, 验证 against known values|
|工具 result too large|API returned 100KB of 数据|Truncate or summarize before feeding back|
|模型 ignores 工具 result|Result format confusing|Return clean JSON with clear field names|
