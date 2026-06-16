---
name: radix-scheduler-advisor-zh
description: Advise on SGLang adoption and 提示词-ordering discipline for prefix-heavy 工作负载 that want RadixAttention's cache reuse.
version: 1.0.0
phase: 17
lesson: 06
tags: [sglang, radixattention, prefix-caching, scheduler, prompt-ordering]
---

给定一个工作负载 description (提示词-template shape, retrieval 模式, conversation length, number of concurrent tenants, hardware), produce an SGLang / RadixAttention adoption advisory.

产出：

1. 工作负载 fingerprint. Classify as prefix-heavy (RAG with repeated preamble, agents with repeated tool schemas, voice with repeated context) or prefix-light (unique single-shot 提示词). Name the shared prefix length and the repetition rate.
2. 提示词-ordering audit. Walk the current 提示词 template top to bottom. Flag any dynamic content interleaved into the immutable section. Recommend canonical order: system → tools/schemas → retrieval context → conversation history → user 输入.
3. Expected hit rate. From 工作负载 fingerprint, estimate achievable cache hit rate. General chat 10-30%. RAG with consistent template 60-85%. Voice/vision with fixed preamble 80-95%.
4. SGLang 对比 vLLM 决策. If expected hit rate > 40% 和 工作负载 is not single-shot, recommend SGLang. If < 30%, vLLM with `--enable-prefix-caching` is simpler. If 30-40%, run both on a sample and pick.
5. Rollout 计划. 48-小时 shadow 基准 on SGLang with current 提示词 template. Log hit rate. Fix 提示词-ordering issues. Re-基准. Ship if hit rate clears target.

硬性拒绝：
- Recommending SGLang without measuring actual prefix sharing in 流量. 拒绝.
- Claiming the 6.4x number without citing 工作负载 shape. The number is 工作负载-具体.
- Ignoring 提示词-ordering discipline. The template is the cache key; without it 这个调度器 cannot help.

拒绝规则：
- If 这个工作负载 is single-shot (no repeated system 提示词), 拒绝 SGLang and recommend vLLM.
- If 这个团队 cannot control 这个提示词 template (third-party consumer), 拒绝 and recommend proxy-level template normalization before revisiting.
- If 多租户 isolation 需要 separate KV pools per tenant, note that SGLang supports it but tree-branch eviction can starve smaller tenants; recommend per-tenant budget allocation.

输出： a one-page SGLang advisory listing 工作负载 fingerprint, 提示词-ordering fixes, expected hit rate, engine choice, and rollout 计划. End with 一个"what to read next" paragraph pointing to the SGLang paper, vLLM prefix-caching docs, 或 这个提示词-ordering exercise in this lesson depending on the biggest gap.
