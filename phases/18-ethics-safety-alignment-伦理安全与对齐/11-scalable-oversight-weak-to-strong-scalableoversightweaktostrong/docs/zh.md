# 可扩展监督 and 弱到强 Generalization

> Burns et al. (OpenAI Superalignment, "弱到强 Generalization", 2023) proposed a proxy for the superalignment problem: 微调 a strong 模型 using labels produced by a weaker 模型. If the strong 模型 generalizes correctly from imperfect weak supervision, current human-scale 对齐 methods may extend to superhuman systems. 可扩展监督 and W2SG are complementary. 可扩展监督 (debate, recursive reward modeling, task decomposition) increases the overseer's effective 能力 so it can keep up with the 模型 under oversight. W2SG ensures the strong 模型 generalizes correctly from whatever imperfect supervision the overseer provides. Debate Helps W2SG (arXiv:2501.13124, January 2025) combines them.

**类型:** 学习
**语言:** Python (stdlib, W2SG gap simulator)
**先修要求:** Phase 18 · 01 (指令遵循), Phase 18 · 10 (AI 控制), Phase 09 (RL foundations)
**时间:** ~60 分钟

## 学习目标

- Define 可扩展监督 and 弱到强 generalization and explain how they are complementary.
- 描述 the Burns et al. 2023 experimental setup: 微调 GPT-4 using labels from GPT-2.
- 解释 the performance gap recovered (PGR) metric and what it measures.
- 说明 the three major scalable-oversight mechanisms (debate, recursive reward modeling, task decomposition) and one strength of each.

## 问题

Every 对齐 technique so far in Phase 18 assumes the overseer can evaluate the 模型's behaviour. When the 模型 is superhuman, the overseer is the weak link. The superalignment question: can a weaker supervisor reliably produce a stronger, aligned 模型?

Burns et al. reduce this to an operationalized empirical setup: supervise strong with weak, measure how much of the strong 模型's 能力 survives the weak supervision. This is not a solution to superalignment ， it is a proxy that lets us measure progress.

## 概念

### W2SG: the Burns et al. setup

- Weak 模型: GPT-2 class.
- Strong 模型: GPT-4 class.
- Target: strong GPT-4 ceiling on the task.

Procedure:
1. Get weak 模型's zero-shot predictions on a task.
2. 微调 the strong 模型 on weak-labeled data.
3. Measure the strong fine-tuned 模型's accuracy.

Baselines: weak 模型's own accuracy, strong 模型's supervised ceiling on gold labels.

Gap metric: Performance Gap Recovered (PGR) = (fine-tuned - weak) / (ceiling - weak). PGR of 1.0 means weak supervision fully closes the gap; PGR of 0 means weak supervision did not help.

### Burns et al. empirical findings

Across NLP, chess-puzzle, and reward-modeling tasks, PGR was consistently positive: the strong 模型 generalized beyond the weak supervisor's mistakes. Task-specific PGR varied from ~20% to ~80%. The strong 模型 "understands" the intended task from imperfect labels, filling in with its 预训练 priors.

Limitations Burns et al. note:
- The 弱到强 gap here is 能力, not 对齐. The weak 模型 is aligned; a truly superhuman 模型 might be aligned by a human but with a much larger 能力 gap.
- The strong 模型's generalization may be more about task priors than about recovering ground-truth intent.
- Eliciting the strong 模型's latent knowledge is the real problem; PGR measures a specific operationalization.

### 可扩展监督: three mechanisms

- **Debate (Irving et al. 2018).** Two instances of U argue opposite sides; the weak judge decides. The assumption: finding a convincing true answer is easier than finding a convincing false answer. Recent 2024-2025 work (Khan et al., Michael et al.) shows debate sometimes helps and sometimes does not, and whether it helps depends on task structure.
- **Recursive Reward Modeling (Leike et al. 2018).** U helps the human train the 奖励模型 for U+1. The overseer's effective 能力 grows with the 模型's.
- **Task Decomposition (Christiano, Shlegeris, Amodei 2018).** Decompose a hard task into sub-tasks the human can check, recursively. Assumes decomposability.

Each mechanism assumes something about the structure of the task or the 对齐 of the intermediate components.

### Why 可扩展监督 and W2SG are complementary

可扩展监督 increases the overseer's effective signal quality.
W2SG closes the gap from whatever imperfect signal the overseer can provide.

Lang et al. ， Debate Helps 弱到强 Generalization (arXiv:2501.13124) combines them: a debate protocol provides better weak labels, and the strong 模型 is trained on those labels. Reported PGR gains on NLP tasks.

### The organizational drama

OpenAI's Superalignment team dissolved in May 2024 after Jan Leike's departure to Anthropic. The agenda (可扩展监督, W2SG, automated 对齐 research) continued at Anthropic and at academic labs ， MATS (Lesson 28), Redwood (Lesson 10), Apollo (Lesson 8), METR (Lesson 28). The organizational structure changed; the research questions did not.

### Where this fits in Phase 18

Lessons 6-10 describe the threat and the defensive paradigm under the assumption U is untrustworthy. Lesson 11 is the offensive paradigm: make the overseer strong enough to verify U's 对齐. Lessons 12-16 then turn to the practical tooling of adversarial 评估.

## 使用它

`code/main.py` simulates a W2SG 微调 on a synthetic task. Weak 标注员 has 70% accuracy with structured errors; strong 模型 has 95% ceiling on gold labels. You 微调 the strong 模型 on weak labels, measure PGR, and compare to strong-on-gold and weak-alone.

## 交付它

本课产出 `outputs/skill-w2sg-pgr.md`. 给定 an oversight setup description, it identifies the weak supervisor, the strong 模型, the supervision quality, and computes (or requests) PGR. It flags whether the claim is "weak can supervise strong" or "weak + oversight mechanism can supervise strong."

## 练习

1. 运行 `code/main.py`. Report PGR for weak_accuracy = 0.60, 0.70, 0.80. 解释 the shape of the PGR curve.

2. 修改 the weak 标注员 to have structured error (e.g., always wrong on a specific 输入 class). Does PGR increase, decrease, or stay the same? 解释.

3. 阅读 Burns et al. 2023 Section 4.3 (NLP tasks). Reproduce the "confidence auxiliary loss" intuition: when the strong 模型 is more confident than the weak labels, who wins?

4. Design a scalable-oversight protocol that combines debate and task decomposition for a software-engineering task. 说出 one failure mode of each component and explain how the combination addresses or fails to address each.

5. Articulate what would falsify the "弱到强 generalization is a viable path to superalignment" claim. Be specific about the empirical signature you would need to see.

## 关键术语

| 术语 | 常见说法 | 实际含义 |
|------|-----------------|------------------------|
| 可扩展监督 | "making the overseer stronger" | Mechanisms that increase an overseer's ability to evaluate a more-capable 模型 |
| W2SG | "weak supervises strong" | 微调 a strong 模型 on weak labels and measuring the 能力 recovered |
| PGR | "performance gap recovered" | (fine-tuned - weak) / (ceiling - weak); 1.0 = fully closed, 0 = no help |
| Debate | "two U instances argue" | 可扩展监督 mechanism where a weak judge picks between two U defenders |
| RRM | "recursive reward modeling" | U helps train the 奖励模型 for U+1; overseer 能力 tracks U |
| Task decomposition | "sub-tasks the human checks" | Break a hard task into sub-tasks the human can verify, recursively |
| Superalignment | "aligning superhuman AI" | The research agenda concerned with aligning models the human cannot directly evaluate |

## 延伸阅读

- [Burns et al. ， Weak-to-Strong Generalization (OpenAI 2023)](https://openai.com/index/weak-to-strong-generalization/) ， the W2SG paper
- [Irving, Christiano, Amodei ， AI safety via debate (arXiv:1805.00899)](https://arxiv.org/abs/1805.00899) ， the debate mechanism
- [Leike et al. ， Scalable agent alignment via reward modeling (arXiv:1811.07871)](https://arxiv.org/abs/1811.07871) ， recursive reward modeling
- [Khan et al. ， Debating with More Persuasive LLMs Leads to More Truthful Answers (arXiv:2402.06782)](https://arxiv.org/abs/2402.06782) ， 2024 empirical study of debate with stronger debaters
- [Lang et al. ， Debate Helps Weak-to-Strong Generalization (arXiv:2501.13124)](https://arxiv.org/abs/2501.13124) ， 2025 combination of debate + W2SG
