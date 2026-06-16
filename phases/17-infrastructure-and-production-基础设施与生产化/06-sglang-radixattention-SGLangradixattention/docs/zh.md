# 面向前缀密集工作负载的 SGLang 与 RadixAttention

> SGLang treats the KV 缓存 as a first-class, reusable resource stored in a radix tree. Where vLLM schedules 请求 FCFS (first-come, first-served), SGLang's cache-aware 调度器 prioritizes 请求 with longer shared prefixes ： effectively a depth-first radix traversal so hot branches stay resident in HBM. On Llama 3.1 8B with ShareGPT-like 1K 提示词, SGLang hits ~16,200 tok/s to vLLM's ~12,500, 一个~29% edge. On prefix-heavy RAG 工作负载 the advantage reaches 6.4x. On voice-cloning-shaped 工作负载 cache hit rate cleared 86%. Deployed on 400,000+ GPUs in 2026 across xAI, LinkedIn, Cursor, Oracle, GCP, Azure, AWS. The gotcha is that the 6.4x number evaporates when prefix ordering is inconsistent ： ordering is the engineer's lever.

**Type:** Learn
**Languages:** Python（标准库， 玩具 radix-tree cache + cache-aware scheduler)
**Prerequisites:** Phase 17 · 04 (vLLM Serving Internals), Phase 14 (Agentic RAG)
**Time:** ~75 分钟

## 学习目标

- Diagram RadixAttention: how prefixes are stored in a radix tree and how KV blocks are shared across sequences rooted at the same branch.
- 解释 cache-aware scheduling and why FCFS is wrong for prefix-heavy 流量.
- Compute expected speedup for 一个工作负载 given prefix-cache hit rate and 提示词 length distribution.
- Name 这个提示词-ordering discipline that makes the 6.4x number real 对比 a lost upside.

## 问题

Classic serving treats each 请求's 提示词 as 不透明. Even when 5,000 RAG 请求 all start with the same 2,000-词元 system 提示词 plus same retrieval preamble, vLLM prefills that 2,000-词元 prefix 5,000 times. The GPU does the same work over and over.

The observation: 提示词 in agentic and RAG 工作负载 share long prefixes almost always. System 提示词, tool schemas, few-shot examples, retrieval 头, conversation history ： all repeat across 请求. If you stored the KV 缓存 for that prefix once and reused it, you would not prefill it again.

RadixAttention does exactly this. 词元 are indexed in a radix tree; each node owns KV blocks for 这个词元 sequence on its path from root. A new 请求 walks the tree: any node whose 词元 matches re-uses that node's KV blocks. Prefill 成本 becomes proportional to 这个"new" suffix, not the full 提示词.

The challenge is scheduling. If two 请求 share a 2,000-词元 prefix and a third shares only 200 词元 of the same prefix, you want to serve the two long-shared 请求 together so the long prefix stays in HBM. FCFS does the opposite ： it serves whoever arrived first, potentially evicting the hot branch before the next long-prefix 请求 hits.

## 概念

### 作为 KV 索引的基数树

A radix tree (compact trie) stores 词元 sequences. Each node owns 一个词元 range and the KV blocks computed for that range. Children extend the sequence one or more 词元.

```
root
 |- "You are a helpful assistant..."  (2,000 tokens, 124 KV blocks)
      |- "Context: <doc A>..."        (500 tokens, 31 blocks)
           |- "Question: Alice..."    (80 tokens, 5 blocks)
           |- "Question: Bob..."      (95 tokens, 6 blocks)
      |- "Context: <doc B>..."        (520 tokens, 33 blocks)
```

A new 请求 comes in with system 提示词 + "Context: <doc A>" + "Question: Carol". 这个调度器 walks: system prefix matches (124 blocks reused), doc-A branch matches (31 blocks reused), then allocates fresh blocks only for "Question: Carol" (4 blocks). Prefill 成本: 4 blocks of new 词元. Without the tree: 160 blocks. ~40x 节省 on prefill.

### 缓存感知调度

Radix-tree-backed reuse is pointless if the cache churns. Two key 策略:

1. **Depth-first dispatch**. When picking the next 请求 from 这个队列, prefer 请求 rooted at the same branch as the current running set. This keeps the hot branch pinned.
2. **LRU at branch level, not block level**. Evict whole branches (starting from shortest-used leaves) rather than individual blocks, so cache shape matches radix shape.

FCFS violates both. 一个请求 sharing 2,000 词元 sits behind 一个请求 sharing 50, then the 2,000-词元 branch gets evicted to admit the 50-词元 one.

### 你应该记住的基准数字

- Llama 3.1 8B, H100, ShareGPT 1K 提示词: SGLang ~16,200 tok/s 对比 vLLM ~12,500 (~29% edge).
- Prefix-heavy RAG (same system + same doc, varying question): up to 6.4x on SGLang.
- Voice cloning 工作负载: 86.4% prefix-cache hit rate.
- 生产化 hit rates across SGLang customers: 50-99% depending on 提示词 discipline.
- Deployed on 400,000+ GPUs in 2026.

### 排序陷阱

The 6.4x number relies on consistent 提示词-template ordering. If your client constructs 提示词 as `[system, tools, context, history, question]` in some 请求 和 `[system, context, tools, history, question]` in others, the tree cannot find the shared prefix. What looks like a shared prefix to a human is two distinct sequences to the radix tree.

Engineer's lever: your 提示词 template is a cache key. Fix the order. Put everything immutable (system, tools, schemas) first. Put retrieval context next. Put user question last. Do not interleave dynamic content into the prefix.

Real case from the research: moving dynamic content out of the cacheable prefix took one 部署 from 7% to 74% cache hit rate in one change.

### RadixAttention 的胜负场景

Wins:
- RAG (same retrieval preamble, varying question).
- Agents (same tool schemas, varying query).
- Chat with long system 提示词.
- Voice / vision 工作负载 with repeated preambles.

Loses (returns to vLLM-level 吞吐量):
- Single-shot generation with unique 提示词 (code completion, open-ended chat without system 提示词).
- Dynamic 提示词 where every 请求 interleaves unique content into the prefix.

### 为什么这是调度器问题，不只是内核问题

You can implement KV reuse as a kernel trick. SGLang's insight is that reuse only pays if 这个调度器 keeps the hot branch resident. A naive "reuse if 可用" 策略 will churn the cache under mixed load. The radix-tree-indexed 调度器 is what turns the kernel trick into a 29% 生产化 edge.

### 与 vLLM 的相互作用

The two systems are not strict competitors. In 2026 vLLM 新增 prefix caching (`--enable-prefix-caching`) and a cache-aware 路由器 (vLLM 路由器 in Rust). The gap closed but did not fully disappear ： SGLang's whole stack is radix-first; vLLM grafted it on. For 工作负载 dominated by prefix reuse, SGLang remains 这个默认. For general-purpose serving without strong prefix patterns, vLLM remains equal or better.

```figure
roofline
```

## 使用它

`code/main.py` implements a toy radix-tree KV 缓存 plus 一个调度器 with two 策略: FCFS and cache-aware. Runs the same 工作负载 through both, reports prefix-cache hit rate and 吞吐量 delta. Then runs 一个"scrambled ordering" 工作负载 to show the 6.4x collapse.

## 交付它

This lesson 产出 `outputs/skill-radix-scheduler-advisor.md`. Given 一个工作负载 description (提示词-template shape, retrieval 模式, number of concurrent tenants), it 产出 一个提示词-ordering prescription and a go/no-go for SGLang adoption.

## 练习

1. Run `code/main.py`. 比较 FCFS and cache-aware on the same 工作负载. Where does the delta come from ： prefill 节省, decode 节省, 或 队列 delay?
2. Modify 这个工作负载 so 提示词 randomly permute `[system, tools, context]`. Re-run. What happens to hit rate? Why?
3. Compute the HBM 成本 of keeping a 2,000-词元 system 提示词 resident as one radix branch on Llama 3.1 8B. 比较 to 这个成本 of a 16-sequence 批处理 without prefix reuse.
4. Read the SGLang RadixAttention paper. 解释 in three sentences why tree-shaped LRU eviction beats block-shaped LRU under prefix-heavy load.
5. 一个客户 reports only 8% cache hit rate. Name three likely causes and the diagnostic you would run for each.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| RadixAttention | "the SGLang thing" | KV 缓存 indexed as a radix tree so shared prefixes reuse blocks |
| Radix tree | "compact trie" | Tree where each node owns 一个词元 range and its KV blocks |
| Cache-aware 调度器 | "hot-branch-first" | 调度器 that prefers 请求 sharing the resident branch |
| Prefix-cache hit rate | "how much of your 提示词 was free" | Fraction of 提示词 词元 served from reused KV blocks |
| FCFS | "first-come first-served" | 默认 scheduling that breaks prefix locality |
| Branch-level LRU | "evict the leaf" | Eviction 策略 matched to radix shape |
| 提示词 template ordering | "the cache key" | 这个提示词's component order determines what the tree can share |
| System 提示词 pinning | "resident prefix" | Keep the immutable system portion pinned to avoid eviction thrash |

## 延伸阅读

- [SGLang GitHub](https://github.com/sgl-project/sglang) ： source and docs.
- [SGLang documentation](https://sgl-project.github.io/) ： RadixAttention and scheduling details.
- [SGLang paper — Efficiently Programming Large Language Models (arXiv:2312.07104)](https://arxiv.org/abs/2312.07104) ： the design reference.
- [LMSYS blog — SGLang with RadixAttention](https://www.lmsys.org/blog/2024-01-17-sglang/) ： 基准 numbers and 调度器 rationale.
- [vLLM — Prefix Caching](https://docs.vllm.ai/en/latest/features/prefix_caching.html) ： vLLM's own radix-like implementation, for comparison.
