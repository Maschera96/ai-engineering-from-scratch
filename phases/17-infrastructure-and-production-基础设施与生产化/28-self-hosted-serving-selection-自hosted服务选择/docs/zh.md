# 自托管服务选择：llama.cpp、Ollama、TGI、vLLM、SGLang

> Four engines dominate 自托管 inference in 2026. Pick based on hardware, scale, and ecosystem. **llama.cpp** is fastest on CPU ： widest 模型 support, full control over 量化 and threading. **Ollama** is the dev-laptop one-command install, ~15-30% slower than llama.cpp (Go + CGo + HTTP serialization), 3x 吞吐量 gap under prod-like load. **TGI entered maintenance mode December 11, 2025** ： only bug fixes, ~10% slower raw 吞吐量 than vLLM but historically top 可观测性 and HF-ecosystem integration. That maintenance status makes it a risky long-term bet ： SGLang or vLLM are safer defaults for new projects. **vLLM** is the general-purpose 生产化 默认 ： v0.15.1 (February 2026) adds PyTorch 2.10, RTX Blackwell SM120, H200 optimization. **SGLang** is the agentic multi-turn / prefix-heavy specialist ： 400,000+ GPUs in 生产化 (xAI, LinkedIn, Cursor, Oracle, GCP, Azure, AWS). Hardware 约束: CPU-only → llama.cpp only. AMD / non-NVIDIA → vLLM only (TRT-LLM is NVIDIA-locked). 2026 pipeline 模式: dev = Ollama, staging = llama.cpp, prod = vLLM or SGLang. Same GGUF/HF weights throughout.

**Type:** Learn
**Languages:** Python（标准库， engine-decision tree遍历器)
**Prerequisites:** 所有 Phase 17 引擎相关课程 (04, 06, 07, 09, 18)
**Time:** ~45 分钟

## 学习目标

- Pick an engine given hardware (CPU / AMD / NVIDIA Hopper / Blackwell), scale (1 user / 100 / 10,000), 和 工作负载 (general chat / agent / 长上下文).
- Name the 2026 TGI maintenance-mode status (December 11, 2025) and why it biases new projects toward vLLM or SGLang.
- Describe the dev/staging/prod pipeline using the same GGUF or HF weights throughout.
- 解释 为什么 "CPU only" forces llama.cpp and "AMD" excludes TRT-LLM.

## 问题

Your 团队 starts a new 自托管 LLM project. One engineer says Ollama, another says vLLM, a third says "doesn't TGI just work out of the box?" All three are right for different contexts. None is right for all.

In 2026 the choice tree matters: hardware first, scale second, 工作负载 third. And one 具体 2025 event ： TGI entering maintenance mode December 11 ： changes 这个默认 for new projects.

## 概念

### 五个引擎

| Engine | Best for | Notes |
|--------|----------|-------|
| **llama.cpp** | CPU / edge / minimal deps / widest 模型 support | Fastest on CPU, full control |
| **Ollama** | Dev laptops, single user, one-command install | 15-30% slower than llama.cpp; 3x prod 吞吐量 gap |
| **TGI** | HF ecosystem, 受监管行业 | **Maintenance mode Dec 11, 2025** |
| **vLLM** | General-purpose 生产化, 100+ users | Broad 生产化 默认; v0.15.1 Feb 2026 |
| **SGLang** | Agentic multi-turn, prefix-heavy 工作负载 | 400,000+ GPUs in 生产化 |

### 硬件优先决策

**CPU only** → llama.cpp. Ollama works too but is slower. No other engine is competitive on CPU.

**AMD GPU** → vLLM (AMD ROCm support). SGLang also works. TRT-LLM is NVIDIA-locked, so it's out.

**NVIDIA Hopper (H100 / H200)** → vLLM or SGLang or TRT-LLM. All three top-tier.

**NVIDIA Blackwell (B200 / GB200)** → TRT-LLM is 这个吞吐量 leader (Phase 17 · 07). vLLM and SGLang follow close.

**Apple Silicon (M-series)** → llama.cpp (Metal). Ollama wraps this.

### 规模其次决策

**1 user / local dev** → Ollama. One command, first-词元 in seconds.

**10-100 users / small 团队** → vLLM single-GPU.

**100-10k users / 生产化** → vLLM 生产化-stack (Phase 17 · 18) or SGLang.

**10k+ users / 企业级** → vLLM 生产化-stack + disaggregated (Phase 17 · 17) + LMCache (Phase 17 · 18).

### 工作负载第三决策

**General chat / Q&A** → vLLM wins on broad 默认.

**Agentic multi-turn (tools, planning, 内存)** → SGLang's RadixAttention (Phase 17 · 06) dominates.

**RAG with heavy prefix reuse** → SGLang.

**Code generation** → vLLM fine; SGLang slightly better on cache.

**Long context (128K+)** → vLLM + chunked prefill; SGLang + tiered KV.

### The TGI maintenance trap

Hugging Face TGI entered maintenance mode December 11, 2025 ： only bug fixes going forward. Historically: top-tier 可观测性, best-in-class HF-ecosystem integration (模型 cards, safety tools), slightly behind vLLM on raw 吞吐量.

For new projects in 2026: 默认 away from TGI. Existing TGI deployments can continue but should migrate eventually. SGLang and vLLM are the safer defaults.

### 流水线模式

Dev (Ollama) → staging (llama.cpp) → prod (vLLM). Same GGUF or HF weights throughout. Engineers iterate quickly on laptops; staging mirrors 生产化 量化; prod is the serving target.

### Ollama caveat

Ollama is great for dev. It is not great for shared 生产化: Go HTTP serialization adds overhead, concurrency management is simpler than vLLM, OpenTelemetry support lags. Use Ollama where it shines ： one user, one command ： and switch to vLLM for shared.

### 自托管 对比 managed is a separate 决策

Phase 17 · 01 (managed hyperscalers), · 02 (inference 平台) cover managed. This lesson assumes you've already decided to self-host. Reasons to self-host: 数据驻留, custom fine-tune, total 成本 ownership at scale, domain 模型 not 可用 on hosted.

### 你应该记住的数字

- TGI maintenance mode: December 11, 2025.
- vLLM v0.15.1: February 2026; PyTorch 2.10; Blackwell SM120 support.
- SGLang 生产化 footprint: 400,000+ GPUs.
- Ollama 吞吐量 gap 对比 llama.cpp: 15-30% slower; 3x under prod load.

```figure
data-parallel
```

## 使用它

`code/main.py` is 一个决策-tree walker: given hardware + scale + 工作负载, picks an engine and explains why.

## 交付它

This lesson 产出 `outputs/skill-engine-picker.md`. 给定约束, picks an engine and writes 这个迁移 计划.

## 练习

1. Run `code/main.py` with your hardware / scale / 工作负载. Does 这个输出 match your intuition?
2. Your infra is 12 H100s and 8 MI300X AMD. What engine? Why is TRT-LLM off the table?
3. 一个团队 wants to use TGI in 2026 because "it's what we know." Argue 这个迁移 case.
4. Ollama dev to vLLM prod: what changes in 量化, configuration, 和 可观测性?
5. RAG 产品 with P99 prefix length 8K and high reuse across tenants. Pick an engine and stack it with Phase 17 · 11 + 18.

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| llama.cpp | "the CPU one" | Widest 模型 support, fastest on CPU |
| Ollama | "the laptop one" | One-command install, dev-grade 吞吐量 |
| TGI | "HF's serving" | Maintenance mode since Dec 2025 |
| vLLM | "这个默认" | Broad 生产化 baseline 2026 |
| SGLang | "the agentic one" | Prefix-heavy, RadixAttention |
| TRT-LLM | "NVIDIA-locked" | Blackwell 吞吐量 leader, NVIDIA only |
| GGUF | "llama.cpp format" | Bundled K-quant variants |
| 生产化-stack | "vLLM K8s" | Phase 17 · 18 reference 部署 |
| Pipeline 模式 | "dev→stage→prod" | Ollama → llama.cpp → vLLM on same weights |

## 延伸阅读

- [AI Made Tools — vLLM vs Ollama vs llama.cpp vs TGI 2026](https://www.aimadetools.com/blog/vllm-vs-ollama-vs-llamacpp-vs-tgi/)
- [Morph — llama.cpp vs Ollama 2026](https://www.morphllm.com/comparisons/llama-cpp-vs-ollama)
- [n1n.ai — Comprehensive LLM Inference Engine Comparison](https://explore.n1n.ai/blog/llm-inference-engine-comparison-vllm-tgi-tensorrt-sglang-2026-03-13)
- [PremAI — 10 Best vLLM Alternatives 2026](https://blog.premai.io/10-best-vllm-alternatives-for-llm-inference-in-production-2026/)
- [TGI maintenance announcement](https://github.com/huggingface/text-generation-inference) ： release notes.
- [vLLM v0.15.1 release notes](https://github.com/vllm-project/vllm/releases)
