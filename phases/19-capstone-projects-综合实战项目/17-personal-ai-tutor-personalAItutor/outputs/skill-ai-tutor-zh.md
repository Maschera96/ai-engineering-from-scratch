---
name: ai-tutor-zh
description: 为特定科目交付一个 adaptive multimodal personal tutor，包含 Bayesian knowledge tracing、curriculum graph、safety filters 和两周 efficacy study。
version: 1.0.0
phase: 19
lesson: 17
tags: [capstone, tutor, adaptive, bkt, fsrs, livekit, multimodal, coppa]
---

给定一个科目（K-12 algebra 或 intro Python），构建个人 tutor，支持 text + voice + photo-math input、Bayesian knowledge tracing learner model、curriculum-graph-driven concept selection、COPPA-aware memory 和 safety filters。用 10 名学习者运行两周 efficacy study。

构建计划：

1. Neo4j 中的 curriculum graph：50-150 个 concept nodes，带 prerequisite edges 和附加 OER content（OpenStax、Open Textbook）。
2. Learner model：Bayesian knowledge tracing，每个 concept 有 guess/slip/learn-rate priors；持久化每个 learner 的 state。
3. Tutor policy（LangGraph over Claude Sonnet 4.7，使用 prompt caching）：read_signal -> select_concept（graph walk）-> scaffold（Socratic）-> update_mastery。
4. Memory：agentmemory 风格的持久 episodic + semantic store；COPPA-aware auto-delete after 1 year；parent-accessible deletion。
5. Voice：LiveKit Agents worker，使用 Whisper-v3-turbo ASR 和 Cartesia Sonic-2 TTS；复用 capstone 03 pipeline。
6. Photo math：dots.ocr 或 PaliGemma 2 做 equation recognition；将结构化输入送入 tutor。
7. Safety：Llama Guard 4 input/output；age-appropriate filter 阻止 self-harm/adult/violence；learner-scoped memory isolation。
8. 每个 learner 每周 PDF progress reports。
9. Efficacy study：10 learners，pre-test（标准化 30-question baseline），2 周 sessions（每周 3 次），post-test；与 non-adaptive linear cohort 对比。

评估标准：

| Weight | Criterion | Measurement |
|:-:|---|---|
| 25 | Learning gain delta | 10-learner 2-week study 中的 pre/post-test delta |
| 20 | Socratic fidelity | Transcript samples 上的 rubric score |
| 20 | Multimodal UX | Voice + photo + text coherence 端到端 |
| 20 | Safety + privacy posture | Llama Guard 4 pass rate + COPPA-aware retention + cross-learner isolation |
| 15 | Curriculum breadth and graph quality | Concept coverage + prerequisite graph consistency |

硬性拒收：

- 直接 answer-dump 而不是追问下一步的 tutor policies。Socratic 是硬要求。
- 不按每次 interaction 更新的 learner models。BKT 是底线。
- 没有 COPPA-aware retention 的 memory。对 K-12 audience 不可接受。
- 没有 non-adaptive baseline cohort 的 efficacy claims。

拒绝规则：

- 输入和输出没有 Llama Guard 4 时，拒绝部署。
- 没有 parent-accessible deletion surface 时，拒绝持久化 learner data。
- 没有并行运行 non-adaptive baseline 时，拒绝声称 “adaptive”。

输出：一个仓库，包含 curriculum graph、BKT learner model、LangGraph tutor policy、multimodal input handlers、LiveKit voice pipeline、safety pipeline、parental dashboard、efficacy-study runner、pre/post test harness，以及一份说明文档，用 confidence intervals 记录相对 linear baseline 的 learning gain delta。
