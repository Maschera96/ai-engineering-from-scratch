# Capstone 15 — 宪法式安全测试框架 + 红队靶场

> Anthropic 的 Constitutional Classifiers、Meta 的 Llama Guard 4、Google 的 ShieldGemma-2、NVIDIA 的 Nemotron 3 Content Safety，以及面向多语言覆盖的 X-Guard，共同定义了 2026 年的安全分类器技术栈。garak、PyRIT、NVIDIA Aegis 和 promptfoo 成了标准对抗评估工具。NeMo Guardrails v0.12 把它们串进生产流水线。这个 capstone 要把整套体系接起来，在目标应用外包一层分层安全 harness，运行一个覆盖 6 种以上攻击家族的自主红队 agent，并执行一次 constitutional self-critique run，输出一个可量化的 harmlessness 提升值。

**Type:** Capstone
**Languages:** Python (safety pipeline, red team), YAML (policy configs)
**Prerequisites:** Phase 10 (LLMs from scratch), Phase 11 (LLM engineering), Phase 13 (tools), Phase 14 (agents), Phase 18 (ethics, safety, alignment)
**Phases exercised:** P10 · P11 · P13 · P14 · P18
**Time:** 25 hours

## Problem

到 2026 年，LLM 安全的前沿问题已经不再是分类器能不能工作，它们大致能工作，而是怎样围绕一个生产应用正确地组合这些分类器，既不产生过度拒绝，也不留下明显漏洞。Llama Guard 4 负责英文 policy violations。X-Guard，覆盖 132 种语言，负责多语言 jailbreak。ShieldGemma-2 负责图像式 prompt injection。NVIDIA Nemotron 3 Content Safety 覆盖企业级类别。Anthropic 的 Constitutional Classifiers 则是另一条路径，主要用于训练阶段，而不是 serving 阶段。

攻击也在进化。PAIR 和 TAP 会自动化发现 jailbreak。GCG 会运行基于梯度的 suffix attacks。多轮攻击和 code-switch 攻击会利用 agent memory。任何已部署的 LLM 都需要一个 red-team range，也就是以 garak 和 PyRIT 为代表的标准驱动框架，再加上文档化的缓解措施和带 CVSS 评分的 findings。

你将加固一个目标应用，可以是一个 8B instruction-tuned model，也可以是其他 capstones 中的某个 RAG chatbot，对它运行 6 种以上攻击家族，并产出前后对比的 harmlessness 测量结果。

## Concept

安全流水线有五层。**Input sanitize**，去除零宽字符、解码 base64 / rot13、规范化 Unicode。**Policy layer**，用 NeMo Guardrails v0.12 rails 处理 off-domain、toxicity、PII extraction。**Classifier gate**，输入走 Llama Guard 4，非英文走 X-Guard，图像输入走 ShieldGemma-2。**Model**，即目标 LLM。**Output filter**，输出再走 Llama Guard 4、Presidio PII scrub，以及在适用时做 citation enforcement。**HITL tier**，高风险输出进入 Slack 队列，交由人工处理。

Red-team range 由调度器驱动。PAIR 和 TAP 自主发现 jailbreaks。GCG 运行梯度型 suffix attacks。还包括 ASCII / base64 / rot13 编码攻击、多轮攻击，角色扮演、记忆利用，以及 code-switch 攻击，把英语与斯瓦希里语或泰语混合。每次运行都要产出一个结构化 findings 文件，包含 CVSS 评分和 disclosure timeline。

Constitutional self-critique run 是训练时干预。取 1k 条 harmful-attempt prompts，让模型先起草 response，再依据一份书面 constitution，禁止伤害等规则，对草稿进行批判，然后在 critique loop 产出的改写数据上重新训练。最后在保留评测集上测量前后 harmlessness 的变化。

## Architecture

```
request (text / image / multilingual)
      |
      v
input sanitize (strip zero-width, decode, normalize)
      |
      v
NeMo Guardrails v0.12 rails (off-domain, policy)
      |
      v
classifier gate:
  Llama Guard 4 (English)
  X-Guard (multilingual, 132 langs)
  ShieldGemma-2 (image prompts)
  Nemotron 3 Content Safety (enterprise)
      |
      v (allowed)
target LLM
      |
      v
output filter: Llama Guard 4 + Presidio PII + citation check
      |
      v
HITL tier for flagged outputs

parallel:
  red-team scheduler
    -> garak (classic attacks)
    -> PyRIT (orchestrated red team)
    -> autonomous jailbreak agent (PAIR + TAP)
    -> GCG suffix attacks
    -> multilingual / code-switch
    -> multi-turn persona adoption

output: CVSS-scored findings + disclosure timeline + before/after harmlessness delta
```

## Stack

- Safety classifiers: Llama Guard 4、ShieldGemma-2、NVIDIA Nemotron 3 Content Safety、X-Guard
- Guardrail framework: NeMo Guardrails v0.12 + OPA
- Red-team drivers: garak，NVIDIA，PyRIT，Microsoft Azure，NVIDIA Aegis、promptfoo
- Jailbreak agents: PAIR，Chao et al., 2023，Tree-of-Attacks，TAP，GCG suffix
- Constitutional training: Anthropic 风格的 self-critique loop + critique 数据上的 SFT
- PII scrub: Presidio
- Target: 一个 8B instruction-tuned model，或其他 capstones 中的某个 RAG chatbot

## Build It

1. **Target setup.** 在 vLLM 上启动一个 8B instruction-tuned model，或复用其他 capstone 的 RAG chatbot。这就是被测试的应用。

2. **Safety pipeline wrap.** 在目标应用外面接上五层安全流水线。验证每一层都能独立观测，最好在 Langfuse 中为每层单独打 span。

3. **Classifier coverage.** 加载 Llama Guard 4、X-Guard，多语言，和 ShieldGemma-2，图像。先在一小组标注数据上跑出基线表现。

4. **Red-team scheduler.** 调度 garak、PyRIT、一个 PAIR agent、一个 TAP agent、一个 GCG runner、一个 multi-turn attacker 和一个 code-switch attacker。每个都在独立队列上运行。

5. **Attack suite.** 六类攻击家族，(1) PAIR 自动化 jailbreak，(2) TAP tree-of-attacks，(3) GCG 梯度 suffix，(4) ASCII / base64 / rot13 编码，(5) 多轮 persona 攻击，(6) 多语言 code-switch。按攻击家族分别报告成功率。

6. **Constitutional self-critique.** 整理 1k 条 harmful-attempt prompts。对每条 prompt，让 target 先起草 response。再由一个 critic LLM 按书面 constitution 打分，比如 “do no harm”、“cite evidence”、“refuse illegal requests”。若 critic 反对，就改写 prompt-response 对，再用这些 critique-improved pairs 对 target 做 fine-tune。最后在保留集上测量前后 harmlessness。

7. **Over-refusal measurement.** 在良性 prompt 集上跟踪 false-positive rate，例如 XSTest。目标系统在良性问题上仍然必须保持有帮助。

8. **CVSS scoring.** 对每一个成功 jailbreak，按 CVSS 4.0 评分，覆盖 attack vector、complexity 和 impact。输出 disclosure timeline 和 mitigation plan。

9. **Range automation.** 以上所有流程都挂在 cron 上运行。findings 写入队列。过度拒绝回归要向 Slack 发送告警。

## Use It

```
$ safety probe --model=target --family=PAIR --budget=50
[attacker]   PAIR agent running on target
[attack]     attempt 1/50: disguise query as academic research ... blocked
[attack]     attempt 2/50: appeal to roleplay ... blocked
[attack]     attempt 3/50: chain-of-thought coax ... SUCCEEDED
[finding]    CVSS 4.8 medium: roleplay bypass on target
[range]      7 successes out of 50 (14% success rate)
```

## Ship It

`outputs/skill-safety-harness.md` 是可交付成果。它是一条生产级分层安全流水线，加上一个可复现的红队靶场，并报告前后 harmlessness 的变化值。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | Attack-surface coverage | 演练 6 种以上攻击家族，覆盖 2 种以上语言 |
| 20 | True-positive / false-positive trade-off | 攻击拦截率，对比 XSTest 良性请求通过率 |
| 20 | Self-critique delta | 保留评测集上的前后 harmlessness 提升 |
| 20 | Documentation and disclosure | 带时间线的 CVSS 评分 findings |
| 15 | Automation and repeatability | 所有流程可由 cron 自动运行，并附带告警 |
| **100** | | |

## Exercises

1. 在一个 RAG chatbot 上运行 garak 的 prompt-injection plugin，并比较加上与不加 output-filter layer 时的攻击成功率。

2. 增加第七类攻击，通过被检索文档实施间接 prompt injection。测量额外所需的防御措施。

3. 实现 “refuse-with-help” 模式。当 guardrail 拦截时，target 不做生硬拒绝，而是提供一个更安全的相关回答。测量它在 XSTest 上的变化。

4. 找出一个 X-Guard 表现较差的语言。针对它提出一套 fine-tune dataset。

5. 在一个 30B 模型上运行 constitutional self-critique，并测量 harmlessness 提升是否随模型规模变化。

## Key Terms

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Layered safety | “Defense in depth” | 在输入、分类 gate、输出和 HITL 多层叠加 guardrails |
| Llama Guard 4 | “Meta’s safety classifier” | 2026 年输入输出内容分类器的参考方案 |
| PAIR | “Jailbreak agent” | 一篇关于用 LLM 驱动越狱发现的论文，Chao et al. |
| TAP | “Tree-of-Attacks” | PAIR 的树搜索变体 |
| GCG | “Greedy coordinate gradient” | 基于梯度的对抗 suffix attack |
| Constitutional self-critique | “Anthropic-style training” | target 起草，critic 评分，重写，再重训练 |
| XSTest | “Benign probe set” | 用于测试 over-refusal 回归的 benchmark |
| CVSS 4.0 | “Severity score” | 用于安全 findings 的标准漏洞严重性评分 |

## Further Reading

- [Anthropic Constitutional Classifiers](https://www.anthropic.com/research/constitutional-classifiers) — 训练阶段参考方案
- [Meta Llama Guard 4](https://ai.meta.com/research/publications/llama-guard-4/) — 2026 年输入输出分类器参考
- [Google ShieldGemma-2](https://huggingface.co/google/shieldgemma-2b) — 图像与多模态安全模型
- [NVIDIA Nemotron 3 Content Safety](https://developer.nvidia.com/blog/building-nvidia-nemotron-3-agents-for-reasoning-multimodal-rag-voice-and-safety/) — 企业场景参考
- [X-Guard (arXiv:2504.08848)](https://arxiv.org/abs/2504.08848) — 132 语言多语言安全模型
- [garak](https://github.com/NVIDIA/garak) — NVIDIA 红队工具链
- [PyRIT](https://github.com/Azure/PyRIT) — Microsoft 红队框架
- [NeMo Guardrails v0.12](https://docs.nvidia.com/nemo-guardrails/) — rails 框架
- [PAIR (arXiv:2310.08419)](https://arxiv.org/abs/2310.08419) — 越狱 agent 论文
