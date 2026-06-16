# Capstone 17 — 个人 AI 导师（自适应、多模态、带记忆）

> Khanmigo，Khan Academy，Duolingo Max、Google LearnLM / Gemini for Education、Quizlet Q-Chat 和 Synthesis Tutor，都在 2026 年大规模推出了自适应多模态辅导。它们的共同形态是，Socratic policy，不直接把答案倒给学生，能在每次互动后更新的 learner model，类似 Bayesian knowledge tracing，支持 voice + text + photo-math input，依赖 curriculum graph retrieval、spaced-repetition scheduling，以及用于年龄适配内容的硬性安全过滤。这个 capstone 的目标是，交付一个面向特定学科的 tutor，比如 K-12 algebra 或 intro Python，与 10 名学习者开展两周 efficacy study，并通过 content-safety audit。

**Type:** Capstone
**Languages:** Python (backend, learner model), TypeScript (web app), SQL (curriculum graph via Postgres + Neo4j)
**Prerequisites:** Phase 5 (NLP), Phase 6 (speech), Phase 11 (LLM engineering), Phase 12 (multimodal), Phase 14 (agents), Phase 17 (infrastructure), Phase 18 (safety)
**Phases exercised:** P5 · P6 · P11 · P12 · P14 · P17 · P18
**Time:** 30 hours

## Problem

Adaptive tutoring 过去是教育科技研究中的一个小众方向。到 2026 年，它已经成了消费级产品。Khanmigo 覆盖了美国大多数学区。Duolingo Max 的 MAU 达到数千万。Google 的 LearnLM / Gemini for Education 驱动 Google Classroom 中的辅导体验。Quizlet Q-Chat 和 flashcards 并排存在。Synthesis Tutor 则靠 “给好奇孩子的导师” 这一定位火了起来。它们的共同要素是，多模态输入，输入可以是打字、说话、拍摄公式，Socratic pedagogy，先追问，再解释，learner model 在每次互动后更新，以及严格的年龄适配安全机制。

你将为一个特定群体构建其中一种系统。衡量标准是真实的 efficacy study，两周时间里，对 10 名学习者比较 pre-test 和 post-test 分数。语音环路必须足够自然，复用 capstone 03 的子技术栈。Memory 必须尊重隐私。Safety filter 必须通过面向 K-12 的、符合 COPPA 意识的红队审查。

## Concept

系统包含四个核心组件。**Tutor policy** 是一个 Socratic loop，当学习者直接索要答案时，policy 会抛出一个引导性问题。当他们回答正确时，系统会前进到下一个概念。当他们卡住时，系统会提供带层次的 scaffolded hint。**Learner model** 是 Bayesian knowledge tracing，或一个简化变体，在每次互动后更新 curriculum node 对应的 mastery probability。**Curriculum graph** 是一张 Neo4j 图，节点是概念，边是 prerequisite 关系，policy 会沿着这张图选择下一个概念。**Memory** 是一个 episodic + semantic store，类似 agentmemory，保存过往互动、常见错误和偏好。

UX 是多模态的。文字输入用来提交 typed answers。语音输入通过 LiveKit + Whisper，复用 capstone 03。照片输入用于数学题，依赖 dots.ocr 或 PaliGemma 2。语音输出通过 Cartesia Sonic-2。安全部分使用 Llama Guard 4，再叠加年龄适配过滤器，阻止成人内容、暴力和自伤内容，并配套一套符合 COPPA 的 memory retention policy。

Efficacy study 本身就是可交付成果。10 名学习者，pre-test 与 post-test，两周周期。你要报告 learning gain delta 和 confidence interval，并与一个非自适应基线对比，也就是相同内容但线性呈现、没有 tutor policy 的版本。

## Architecture

```
learner device
  |
  +-- text         -> web app
  +-- voice        -> LiveKit Agents (ASR + TTS)
  +-- photo math   -> dots.ocr / PaliGemma 2
       |
       v
  tutor policy (LangGraph)
       - Socratic decision head
       - next-concept chooser (curriculum graph walk)
       - hint scaffolder
       - mastery update
       |
       v
  learner model (BKT / item-response theory)
       - per-concept mastery probability
       - spaced-repetition scheduler (SM-2 or FSRS)
       |
       v
  memory (agentmemory-style)
       - episodic: every interaction
       - semantic: learned mistakes, preferences
       - retention policy: COPPA / GDPR aware
       |
       v
  curriculum graph (Neo4j)
       - prerequisite edges
       - OER content attached
       |
       v
  safety:
    Llama Guard 4 + age-appropriate filter
    memory access guarded by learner ID scope
```

## Stack

- Subject choice: K-12 algebra 或 intro Python，二选一，专注做深
- Tutor policy: LangGraph + Claude Sonnet 4.7，启用 prompt caching
- Learner model: Bayesian knowledge tracing，经典方案，或用 FSRS 做 spacing
- Curriculum graph: Neo4j，存 concepts、prerequisite edges 和 OER content
- Memory: agentmemory 风格的 persistent vector + episodic + semantic store
- Voice: LiveKit Agents 1.0 + Cartesia Sonic-2，复用 capstone 03 子技术栈
- Photo math: dots.ocr 或 PaliGemma 2 做 equation recognition
- Safety: Llama Guard 4 + 自定义 age-appropriate filter
- Eval: Bloom-level question generation、pre/post test harness、efficacy study tooling

## Build It

1. **Curriculum graph.** 构建一张包含 50 到 150 个 concept nodes 的 Neo4j curriculum graph，例如从 “number line” 到 “quadratic formula” 的 K-12 algebra 路径，并通过 prerequisite edges 连接。为每个节点附上 OER content，比如 Open Textbook、OpenStax。

2. **Learner model.** 初始化 Bayesian knowledge tracing，设置 priors，包括 guess、slip、learn-rate。每次互动后更新每个 concept 的 mastery。按学习者分别持久化。

3. **Tutor policy.** 用 LangGraph 搭建节点，包括 `read_signal`，判断学习者回答是正确、部分正确还是卡住，`select_concept`，沿 curriculum graph 选出优先级最高的概念，`scaffold`，生成 Socratic prompt，`update_mastery`。

4. **Memory.** 每次互动都写入 episodic store。常见错误和偏好晋升到 semantic memory。Retention policy 要符合 COPPA，1 年后自动删除，并允许家长访问与删除。

5. **Voice path.** 为 tutor policy 接一个 LiveKit Agents worker。ASR 用 Whisper-v3-turbo。TTS 用 Cartesia Sonic-2。支持 barge-in，复用 capstone 03 的机制。

6. **Photo-math path.** 允许上传或拍照。用 dots.ocr 或 PaliGemma 2 识别公式，再把结构化输入交给 tutor。

7. **Safety.** 所有模型输出都必须经过 Llama Guard 4 和年龄适配过滤器，阻止自伤、成人内容和暴力内容。Memory access 必须按 learner ID 隔离，并提供家长删除入口。

8. **Efficacy study.** 组织 10 名学习者，先做 pre-test，标准化 30 题基线，再进行两周 tutor 互动，每周 3 次 session，最后做 post-test。与另一组 10 名使用非自适应线性内容的学习者对比。

9. **Weekly progress reports.** 为每位学习者自动生成 PDF 周报，总结已探索主题、mastery 轨迹和推荐的下一步学习内容。

## Use It

```
learner: "I don't understand why 3x + 6 = 12 means x = 2"
[signal]   stuck
[concept]  'isolating variables' (prerequisite: addition-subtraction-equality)
[scaffold] "what number would you subtract from both sides to start?"
learner: "6"
[signal]   correct
[mastery]  addition-subtraction-equality: 0.62 -> 0.77
[concept]  continue 'isolating variables'
[scaffold] "great. now what is 3x / 3 equal to?"
```

## Ship It

`outputs/skill-ai-tutor.md` 是可交付成果。它是一个面向特定学科的自适应导师，支持多模态输入，具备 learner model、memory、安全机制，并给出实测 efficacy。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Learning gain delta | 10 名学习者两周 study 中的 pre/post-test delta |
| 20 | Socratic fidelity | Transcript samples 上的 rubric score |
| 20 | Multimodal UX | Voice + photo + text 的端到端一致性 |
| 20 | Safety + privacy posture | Llama Guard 4 通过率 + 符合 COPPA 的 retention |
| 15 | Curriculum breadth and graph quality | Concept coverage + prerequisite graph consistency |
| **100** | | |

## Exercises

1. 在有和没有 adaptive learner model 的情况下各跑一次 efficacy study，后者使用随机概念顺序。报告两者的差值。自适应版本理应更好，但更有价值的是这个提升幅度究竟有多大。

2. 增加一个多模态 probe。同一个概念问题分别通过 text、voice 和 photo 形式交付。测量学习者在偏好模态下是否收敛更快。

3. 构建家长 dashboard。展示练习过的主题、mastery trajectories、接下来将学习的概念，以及安全事件，任何 guardrail hits。保持与 COPPA 对齐。

4. 增加语言切换模式。Tutor 接受西班牙语输入，并用西班牙语授课。测量 X-Guard 覆盖效果。

5. 对 memory privacy 做压力测试。验证 learner A 无法通过语音片段重摄取攻击看到 learner B 的数据。记录这次访问尝试并发出告警。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Socratic policy | “Ask, do not dump” | 导师不直接给答案，而是先提出引导性问题 |
| Bayesian knowledge tracing | “BKT” | 针对每个概念维护 mastery probability 的经典 learner-model 方程 |
| FSRS | “Free Spaced Repetition Scheduler” | 2024 年的 spaced-repetition scheduler，通常优于 SM-2 |
| Curriculum graph | “Concept DAG” | 存在 Neo4j 中的概念图，带 prerequisite edges |
| Episodic memory | “Per-interaction log” | 保存每次互动，便于后续检索 |
| Semantic memory | “Learned pattern store” | 从 episodic 中提炼出的常见错误与偏好模式 |
| COPPA | “Kids privacy law” | 美国针对 13 岁以下儿童数据收集的隐私法律 |

## Further Reading

- [Khanmigo (Khan Academy)](https://www.khanmigo.ai) — 面向 K-12 的消费级导师参考
- [Duolingo Max](https://blog.duolingo.com/duolingo-max/) — 语言学习导师参考
- [Google LearnLM / Gemini for Education](https://blog.google/technology/google-deepmind/learnlm) — 托管参考模型
- [Quizlet Q-Chat](https://quizlet.com) — 另一种参考产品
- [Synthesis Tutor](https://www.synthesis.com) — 创业公司参考方案
- [FSRS algorithm](https://github.com/open-spaced-repetition/fsrs4anki) — spaced-repetition scheduler
- [Bayesian Knowledge Tracing](https://en.wikipedia.org/wiki/Bayesian_knowledge_tracing) — 经典 learner model
- [LiveKit Agents](https://github.com/livekit/agents) — 语音栈参考
