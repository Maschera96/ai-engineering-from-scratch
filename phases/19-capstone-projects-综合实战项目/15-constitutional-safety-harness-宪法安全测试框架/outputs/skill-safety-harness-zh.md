---
name: safety-harness-zh
description: 围绕目标 LLM app 接入 layered safety pipeline，运行六类 red-team range，并进行 constitutional self-critique 以获得可度量 harmlessness delta。
version: 1.0.0
phase: 19
lesson: 15
tags: [capstone, safety, red-team, llama-guard, x-guard, garak, pyrit, constitutional-ai]
---

给定一个目标 LLM application（8B instruction-tuned model 或 RAG chatbot），用 layered safety pipeline 加固它，并跨六类攻击运行 autonomous red-team range。产出 before/after harmlessness report。

构建计划：

1. 五层 pipeline：input sanitize（zero-width strip、encoding decode、Unicode normalize）-> NeMo Guardrails v0.12 rails -> classifier gate（Llama Guard 4 / X-Guard / ShieldGemma-2 / Nemotron 3）-> target LLM -> output filter（Llama Guard 4 + Presidio PII + citation check）。被标记输出进入 Slack HITL queue。
2. 每层发出一个 Langfuse span，使 attribution 端到端可观测。
3. Red-team scheduler 按 cron 运行 garak、PyRIT、PAIR、TAP、GCG、multi-turn persona 和 multilingual code-switch attacks。
4. 每个成功 jailbreak：CVSS 4.0 score、repro、mitigation plan、disclosure timeline。
5. 持续运行 XSTest benign-prompt probe，捕获 over-refusal regressions。
6. Constitutional self-critique run：1k harmful-attempt prompts -> target drafts -> critic 根据 written constitution 评分 -> rewritten pairs -> SFT。用 held-out harmlessness eval 度量 before/after。
7. Alerts：benign-regression 发 Slack warning，新 jailbreak family 发 PagerDuty critical。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | Attack-surface coverage | 演练 6+ attack families，2+ languages |
| 20 | True-positive / false-positive trade-off | Attack block rate vs XSTest benign pass rate |
| 20 | Self-critique delta | held-out eval 上的 before/after harmlessness |
| 20 | Documentation and disclosure | 带 timeline 的 CVSS-scored findings |
| 15 | Automation and repeatability | Cron-driven，alerts 端到端演练 |

硬性拒收：

- 单层 safety stacks。这个 capstone 的论点是 defense in depth。
- 只报告 red-team success rate，却没有 XSTest over-refusal numbers。
- 没有 held-out eval 的 constitutional self-critique（报告的是 training-set accuracy，不是 generalization）。
- Jailbreak findings 缺少 CVSS scoring。

拒绝规则：

- 没有 benign-probe counterpoint 时，拒绝报告 safety number。缺一不可，否则误导。
- 没有人类 curate critique pairs 时，拒绝自动用 red-team successes retrain。
- 没有至少两种非英语语言上运行 X-Guard 时，拒绝声称 multilingual coverage。

输出：一个仓库，包含五层 pipeline、red-team scheduler、PAIR/TAP/GCG runners、constitutional-self-critique training harness、XSTest over-refusal dashboard、CVSS findings tracker，以及一份说明文档，列出加固前成功率最高的三类 attack families 和缓解每类攻击的具体 pipeline layer。
