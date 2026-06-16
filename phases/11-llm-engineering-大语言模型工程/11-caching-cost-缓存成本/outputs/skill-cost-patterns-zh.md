---
name: skill-cost-patterns-zh
description: Decision framework for LLM 成本 优化 -- 缓存 strategies, 速率 limiting, 模型 路由, and 预算 controls
version: 1.0.0
phase: 11
lesson: 11
tags: [caching, cost-optimization, rate-limiting, model-routing, budget, llm-ops]
---

# LLM 成本 优化 Patterns

当building an LLM 应用 that needs to control 成本, apply this decision framework.

## When to 优化

**优化 immediately when:**
- Monthly LLM spend exceeds $500 or 10% of infrastructure 预算
- 成本 per 查询 is above $0.01 for a consumer product
- 你的系统 提示词 is over 1,000 词元 and sent with every request
- More than 30% of 查询 are duplicates or near-duplicates
- 你are 扩展 from 100 to 10,000+ daily users

**Do not 优化 yet when:**
- 你have fewer than 100 DAU and are still validating product-market fit
- Monthly spend is under $100 and growing slowly
- 你are still iterating on 提示词 design (缓存 locks you into a 提示词)

## 缓存 strategy selection

### Exact 缓存

**Use when:** temperature=0, identical prompts repeat, deterministic outputs needed.

```python
key = sha256(json.dumps({"model": m, "messages": msgs, "temp": 0}))
```

- Implementation: 30 分钟
- Hit 速率: 10-25% for most apps, 40-60% for FAQ bots
- 延迟: <1ms (dict lookup)
- 风险: stale 响应 if underlying 数据 changes

**Skip when:** temperature > 0, every 查询 is unique, 实时 数据 needed.

### 语义 缓存

**Use when:** users ask the same 问题 in different words, FAQ-heavy products, customer support.

- Implementation: 2-4 小时 (嵌入 + 相似度 + storage)
- Hit 速率: 15-35% on top of exact 缓存
- 延迟: 10-50ms (嵌入 + ANN search)
- 风险: false positives (returning wrong cached 答案 for a similar but different 问题)

**阈值 guidelines:**
- 0.98+: very conservative, almost no false positives, lower hit 速率
- 0.95: good balance for factual Q&A
- 0.90: aggressive, higher hit 速率 but 风险 of wrong answers
- 0.85: only for low-stakes applications (suggestions, autocomplete)

**Skip when:** every 查询 has unique 上下文 (code 生成), 响应 must reflect latest 数据, 查询 space is unbounded.

### Provider 提示词 缓存

**Use when:** 系统 提示词 > 1,024 词元 (OpenAI) or model-specific minimum, same prefix sent repeatedly.

|Provider|动作|Savings|
|----------|--------|---------|
|Anthropic|Add `cache_control: {"type": "ephemeral"}` to 系统 消息|90% on cached prefix (after 25% write premium)|
|OpenAI|Nothing (automatic)|50% on cached prefix|
|Google|Use 上下文 缓存 API with explicit TTL|~75% on cached 上下文|

**Skip when:** 系统 提示词 changes per request, 提示词 is under minimum length.

## 模型 路由 rules

### Keyword-based (simple, fast)

```text
simple:  <= 5 words OR matches FAQ keywords -> gpt-4o-mini ($0.15/$0.60)
medium:  general queries, summaries        -> claude-sonnet ($3/$15)
complex: "analyze", "compare", "debug"     -> gpt-4o ($2.50/$10)
```

- Implementation: 1 hour
- Accuracy: 70-80%
- Savings: 40-60% of 模型 成本

### 嵌入-based (more accurate)

Embed 50-100 标签ed 查询 per category. Classify new 查询 by nearest neighbor.

- Implementation: 4-8 小时
- Accuracy: 85-92%
- Savings: 50-70% of 模型 成本
- Additional 成本: ~$0.02/1M 词元 for 分类 嵌入s (negligible)

### ML-based (生产 grade)

训练 a small 分类器 (logistic 回归 or small BERT) on historical 查询/模型 pairs.

- Implementation: 1-2 weeks
- Accuracy: 90-95%
- Savings: 60-75% of 模型 成本
- Requires: 标签ed 训练 数据 from 生产 traffic

## 速率 limiting configuration

### 词元 bucket 参数 by tier

|Tier|Bucket Size|Refill 速率|Max RPM|Daily Cap|
|------|-------------|-------------|---------|-----------|
|Free|50K 词元|500/sec|10|50K|
|Pro|500K 词元|5K/sec|60|500K|
|Enterprise|5M 词元|50K/sec|300|5M|

### Implementation checklist

1. Store buckets in Redis (not in-memory) for multi-instance apps
2. 使用atomic operations (MULTI/EXEC) to prevent race conditions
3. 返回`Retry-After` header with rejection 响应
4. Track rejected requests as a 指标 (>5% rejection = tier limits too tight)
5. Implement graceful degradation: reject expensive 模型 requests first, keep cheap 模型 access

## 预算 controls

### Three-threshold circuit breaker

|阈值|动作|Reversible|
|-----------|--------|------------|
|70% of monthly 预算|Log warning, alert team via Slack/PagerDuty|Yes (auto)|
|85% of monthly 预算|Route all traffic to cheapest 模型|Yes (auto, next billing cycle)|
|95% of monthly 预算|Serve cached 响应 only, reject new LLM calls|Yes (manual reset or next cycle)|

### Per-user 成本 tracking

Track cumulative 成本 per 用户. Flag users exceeding 10x the median. Common causes:
- Legitimate power 用户 (upgrade their tier)
- 提示词 injection 循环 (bot sending automated requests)
- Inefficient integration (客户端 retrying on every 错误)

## 成本 tracking fields

Log every API call with these fields:

```json
{
  "timestamp": "2026-04-02T10:30:00Z",
  "model": "gpt-4o",
  "input_tokens": 1523,
  "output_tokens": 487,
  "cached_input_tokens": 1024,
  "latency_ms": 1847,
  "cost_usd": 0.006142,
  "user_id": "user_abc123",
  "cache_status": "partial_hit",
  "request_category": "customer_support",
  "complexity_class": "medium",
  "routed_from": "gpt-4o"
}
```

### Key 指标 to dashboard

- **成本 per 查询** (P50, P95, P99) -- by 模型, by 特征, by 用户 tier
- **缓存 hit 速率** -- exact vs 语义, trend over time
- **模型 分布** -- % of traffic per 模型, 成本 per 模型
- **预算 burn 速率** -- current spend vs projected monthly at current 速率
- **Rejection 速率** -- % of requests rate-limited, by tier

## Common mistakes

|Mistake|Why it hurts|修复|
|---------|-------------|-----|
|缓存 with temperature > 0|Non-deterministic outputs, stale 缓存 gives wrong variety|Only 缓存 temp=0 calls, or accept that cached 响应 lose randomness|
|语义 缓存 阈值 too low|Returns wrong answers for superficially similar 查询|Start at 0.95, lower only after measuring false positive 速率|
|No 缓存 invalidation|响应 go stale when underlying 数据 changes|Set TTL (1 hour for dynamic 数据, 24 小时 for static), invalidate on 数据 updates|
|路由 all traffic to cheapest 模型|质量 drops, users notice|Route by complexity, measure 质量 per tier, set minimum 质量 thresholds|
|No per-user limits|One abusive 用户 burns entire 预算|Always implement per-user quotas, even if generous|
|Ignoring 输出 词元|输出 成本 2-5x more than 输入 per 词元|Set max_tokens appropriately, use stop sequences, compress outputs|
|缓存 before 提示词 is stable|缓存 fills with 响应 from old prompts|Only enable 缓存 after 提示词 is finalized, flush 缓存 on 提示词 changes|

## Pricing 参考 (as of April 2026)

|模型|输入 ($/1M)|输出 ($/1M)|Cached 输入 ($/1M)|Best For|
|-------|-------------|--------------|--------------------|---------| 
|gpt-4.1-nano|$0.10|$0.40|$0.025|High-volume simple tasks|
|gpt-4o-mini|$0.15|$0.60|$0.075|Simple 路由, 分类|
|gemini-2.5-flash|$0.15|$0.60|$0.0375|预算 multimodal|
|claude-haiku-3.5|$0.80|$4.00|$0.08|Fast mid-tier tasks|
|o4-mini|$1.10|$4.40|$0.275|推理 on a 预算|
|gemini-2.5-pro|$1.25|$10.00|$0.3125|Long 上下文, multimodal|
|gpt-4o|$2.50|$10.00|$1.25|General purpose, 函数调用|
|claude-sonnet-4|$3.00|$15.00|$0.30|Balanced 质量/成本|
|claude-opus-4|$15.00|$75.00|$1.50|Maximum 质量, complex 推理|
