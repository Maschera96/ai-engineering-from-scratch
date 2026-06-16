---
name: skill-production-checklist-zh
description: Decision framework for shipping LLM applications to 生产 -- covers every component with specific thresholds and pass/fail criteria
version: 1.0.0
phase: 11
lesson: 13
tags: [production, deployment, llm, architecture, scaling, cost, observability, guardrails]
---

# 生产 LLM Checklist

当shipping an LLM 应用, work through this checklist in order. Each section has pass/fail criteria with specific thresholds.

## 1. Security (Ship Blockers)

每个item here must pass before any deployment.

|Check|Pass Criteria|How to Verify|
|-------|--------------|---------------|
|API keys in env vars|Zero hardcoded keys in codebase|`grep -r "sk-" --include="*.py"` returns nothing|
|输入 护栏 active|提示词 injection patterns blocked|Send "Ignore all previous instructions" -- returns blocked 响应|
|PII redaction|SSN, credit card, email patterns caught|Send "My SSN is 123-45-6789" -- PII redacted before LLM call|
|输出 filtering|Dangerous content blocked|模型 cannot return `DROP TABLE`, `rm -rf`, `exec()` patterns|
|速率 limiting|Per-user request cap enforced|100 requests from same 用户 in 10 seconds -- last 50+ rejected|
|Auth on all endpoints|No unauthenticated LLM access|`curl /v1/chat` without 词元 returns 401|
|CORS restricted|Only 生产 domains allowed|`Origin: evil.com` request rejected|
|Max 输入 词元|Requests over 限制 rejected|Send 50K 词元 输入 -- returns 413 or truncation|

## 2. Reliability (Week-One Survival)

These prevent your first on-call incident.

|Check|Pass Criteria|How to Verify|
|-------|--------------|---------------|
|Retry with backoff|3 retries on 5xx, exponential delay|Kill LLM mock mid-request -- retries visible in logs|
|备选方案 模型 链|2+ 模型 in 链|Primary 模型 unavailable -- 响应 still returns from 备选方案|
|Request timeout|30s max on all external calls|Slow LLM mock (60s) -- request times out at 30s|
|Graceful degradation|缓存/RAG failure does not crash service|Stop 缓存 -- requests still succeed (slower, more expensive)|
|Health check endpoint|Returns dependency status|`GET /health` returns `{"status": "healthy", "cache": ..., "llm": ...}`|
|Streaming works|First 词元 under 500ms|时间-to-first-token measured, consistently < 500ms|
|错误 消息 are safe|Internal 错误 never leak to users|Force 500 -- 用户 sees generic 错误, not stack trace|

## 3. 成本 Control (Month-One Economics)

These prevent the $50K surprise invoice.

|Check|Pass Criteria|How to Verify|
|-------|--------------|---------------|
|成本 per request tracked|Every request logs 词元 count + USD 成本|Request log has `input_tokens`, `output_tokens`, `cost_usd` fields|
|语义 缓存 active|> 20% hit 速率 on repeated patterns|缓存 stats show hit 速率 after 1000 test requests|
|缓存 TTL configured|Entries expire (default: 1 hour)|Entry inserted -- not returned after TTL|
|Per-user 成本 tracking|成本 aggregated by user_id|Dashboard/API shows top 10 users by 成本|
|成本 alerting|Alert at 80% of daily 预算|Set $10 daily 预算, send $8.50 in requests -- alert fires|
|模型 路由 by 成本|Low-complexity 查询 use cheaper 模型|Simple 问题 routes to gpt-4o-mini, complex to gpt-4o|
|Max 输出 词元 set|响应 capped per template|Template with max_output_tokens=512 -- 响应 never exceeds it|

**成本 estimation formula:**
```text
Monthly LLM cost = DAU x queries_per_user x 30 x (1 - cache_hit_rate) x (avg_input_tokens x input_price + avg_output_tokens x output_price) / 1,000,000
```

**基准 thresholds by 规模:**

|DAU|目标 成本/request|Monthly 预算|
|-----|-------------------|----------------|
|1K|< $0.005|< $750|
|10K|< $0.003|< $4,500|
|100K|< $0.001|< $15,000|

## 4. Observability (调试 in 生产)

你cannot fix what you cannot see.

|Check|Pass Criteria|How to Verify|
|-------|--------------|---------------|
|结构化 JSON logging|Every request produces a JSON log line|Log contains: request_id, user_id, 模型, 词元, latency_ms, 成本|
|Request tracing|End-to-end trace with component timing|Single request shows: guardrail (5ms) + 缓存 (2ms) + llm (3200ms) + 评估 (1ms)|
|延迟 tracking|P50, P95, P99 measured|After 1000 requests: P50 < 2s, P99 < 10s|
|错误 速率 monitoring|错误 counted and categorized|Dashboard shows: 0.5% API 错误, 0.1% guardrail 块, 0.01% timeouts|
|缓存 指标|Hit 速率, miss 速率, entry count visible|`GET /v1/cache/stats` returns current numbers|
|A/B test 指标|Per-variant 质量 指标 logged|Each request logs prompt_template + version for comparison|
|评估 logging|质量 signals recorded per request|响应 length, 延迟, 模型, template version stored for offline analysis|

## 5. 提示词 Management

Prompts are code. Treat them like code.

|Check|Pass Criteria|How to Verify|
|-------|--------------|---------------|
|Versioned templates|Every template has a name + version string|Template change creates new version, old version preserved|
|A/B 测试 support|Traffic split by deterministic 用户 hash|Same 用户 always sees same variant within experiment|
|Rollback capability|Revert to previous version in < 1 分钟|Change experiment 配置 -- traffic instantly shifts|
|Template 验证|Variables validated before rendering|Missing variable in template raises clear 错误, not KeyError|
|系统 提示词 separation|系统 and 用户 消息 in separate fields|系统 提示词 is not concatenated into 用户 消息|

## 6. 扩展 Readiness

Not needed at launch. Needed at 10x.

|Check|Pass Criteria|How to Verify|
|-------|--------------|---------------|
|异步 LLM calls|No thread blocking on API calls|50 concurrent requests -- 服务器 CPU stays < 30%|
|Connection pooling|HTTP connections reused|Network trace shows persistent connections to LLM provider|
|Horizontal 扩展|Stateless 服务器 design|2 instances behind load balancer -- all requests succeed|
|Queue support|Non-real-time tasks go to queue|Summarization request returns job_id, result available via polling|
|Load tested|100 concurrent users, < 5% 错误 速率|`wrk` or `locust` test passes at 目标 concurrency|

## Implementation order for new projects

1. **Day 1:** API 服务器 + 提示词 templates + single LLM call with retry
2. **Day 2:** 输入 护栏 + 输出 护栏 + 错误 handling
3. **Day 3:** 语义 缓存 + 成本 tracking per request
4. **Day 4:** Streaming (SSE) + health check endpoint
5. **Day 5:** 结构化 logging + request tracing + 评估 logging
6. **Week 2:** A/B 测试 + 提示词 versioning + rollback
7. **Week 3:** 备选方案 模型 链 + graceful degradation
8. **Week 4:** Load 测试 + 异步 优化 + horizontal 扩展

## Quick diagnostic

如果something is wrong in 生产, check in this order:

1. **Users complaining about 错误?** Check health endpoint, then 错误 速率 in logs, then LLM provider status page
2. **响应 are slow?** Check P99 延迟, then 缓存 hit 速率, then LLM 响应 times in traces
3. **成本 spiking?** Check cost-per-request trend, then 缓存 hit 速率, then top users by 成本, then look for 提示词 template changes that increased 词元 count
4. **质量 dropped?** Check if a new 提示词 version was deployed, check if RAG 检索 accuracy changed, check if 模型 provider changed default 模型 version
5. **Security incident?** Check guardrail 块 速率 (sudden drop = 护栏 disabled), check request logs for unusual patterns, rotate API keys immediately
