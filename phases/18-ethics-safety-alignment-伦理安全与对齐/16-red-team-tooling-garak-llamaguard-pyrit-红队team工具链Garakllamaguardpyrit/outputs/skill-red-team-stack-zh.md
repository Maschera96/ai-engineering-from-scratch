---
name: red-team-stack-zh
description: Recommend a 红队 tool stack and configuration for a given 部署.
version: 1.0.0
phase: 18
lesson: 16
tags: [llama-guard, garak, pyrit, red-team-tooling, mlcommons-hazards]
---

给定 a 部署 description, recommend a 红队 tool stack and regression cadence.

产出：

1. 分类器 placement. Recommend Llama Guard (3-8B, 3-1B-INT4, or 4-12B) at 输入, 输出, or both. For edge deployments, prefer 3-1B-INT4. For multimodal, Llama Guard 4.
2. Probe scanner configuration. Recommend Garak probes relevant to the 部署: hallucination (for RAG systems), data leakage (for PII-adjacent), 提示注入 (always), jailbreaks (always). Specify the Prompt-Guard-86M + Llama-Guard-3-8B shield pairing for end-to-end 评估.
3. Campaign orchestrator. Recommend PyRIT for pre-release campaigns on models with novel capabilities. Specify converter chains to run (paraphrase, encode, translate, roleplay) and orchestrator (Crescendo for escalation, TAP for branching).
4. Cadence. Garak nightly for regression. PyRIT per-release for deep 红队测试. Llama Guard deployed continuously.
5. Judge calibration. Specify the judge LLM (GPT-4-turbo, StrongREJECT, internal) for every tool that uses one. Judge calibration drives reported ASRs.

硬性拒绝：
- Any 部署 without at least one Llama Guard-class 输入 or 输出 分类器.
- Any release without Garak or equivalent single-turn regression.
- Any high-stakes 部署 without a PyRIT-equivalent campaign before release.

拒绝规则：
- 如果用户询问 for a single "best" tool, refuse ， the three cover different layers and are layered, not substituted.
- 如果用户询问 for an all-in-one commercial alternative, refuse the recommendation and point to the 2026 state: the three open tools are the current best-practice stack.

输出：a one-page recommendation that names the 分类器 placement, probe configuration, campaign orchestrator, regression cadence, and judge identity. 引用 Meta (arXiv:2407.21783), NVIDIA Garak, and Microsoft PyRIT once each.
