# Capstone 07 — 端到端微调流水线（Data to SFT to DPO to Serve）

> 一个 8B 模型，用你自己的数据训练，用你自己的偏好做 DPO 对齐，量化、推测解码，并以可度量的 $/1M tokens 服务。2026 年开放栈是 Axolotl v0.8、TRL 0.15、Unsloth 用于迭代、GPTQ/AWQ/GGUF 用于量化、vLLM 0.7 搭配 EAGLE-3 用于 serving。本综合实战是可复现地跑完整条流水线：YAML 进，served endpoint 出，并按 2026 Model Openness Framework 发布 model card。

**Type:** Capstone
**Languages:** Python（pipeline）、YAML（configs）、Bash（scripts）
**Prerequisites:** Phase 2（ML）、Phase 3（DL）、Phase 7（transformers）、Phase 10（LLMs from scratch）、Phase 11（LLM engineering）、Phase 17（infrastructure）、Phase 18（safety）
**Phases exercised:** P2 · P3 · P7 · P10 · P11 · P17 · P18
**Time:** 35 小时

## 问题

2026 年每个严肃 AI 团队都会随时保有一条 fine-tuning pipeline。不是因为他们发布 frontier base model，而是因为下游适配才有可度量收益：domain SFT、针对标注偏好的 DPO、用于 speculative decoding 的 distilled drafts、用 EAGLE-3 serving。Axolotl v0.8 处理 multi-GPU SFT configs。TRL 0.15 处理 DPO 和 GRPO。Unsloth 让你快速 single-GPU iteration。vLLM 0.7 搭配 EAGLE-3 在不损失质量的情况下把 decode throughput 推高 2-3x。工具已经能用；工艺在 YAML、data hygiene 和 eval discipline 里。

你会把一个 8B base（Llama 3.3、Qwen3 或 Gemma 3）在 task-specific data 上先跑 SFT 再跑 DPO，量化用于 serving，并用 lm-evaluation-harness、RewardBench-2、MT-Bench-v2 和 MMLU-Pro 度量提升。你会按 2026 Model Openness Framework 产出 model card。重点是可复现性：一条命令端到端重跑完整流水线。

## 概念

流水线有五个阶段。**Data**：dedup（MinHash / Datatrove）、quality filter（Nemotron-CC 风格 classifier）、PII scrub、对 public benchmark contamination 做 split-hygiene check。**SFT**：Axolotl YAML、8xH100 上 ZeRO-3、cosine schedule、packed sequences、2-3 epochs。**DPO or GRPO**：TRL config、1 epoch、preference pairs 可以由人类标注或模型评审，beta tuning。**Quantize**：GPTQ + AWQ + GGUF，保证部署灵活性。**Serve**：vLLM 0.7 搭配 EAGLE-3 speculative heads（或 SGLang 搭配 SpecForge）、K8s deployment、HPA on queue-wait。

Ablations 是交付物：在三个 task-specific benchmarks 上对比 SFT-only vs SFT+DPO vs SFT+GRPO。Serving metrics：batch 1 / 8 / 32 的 tokens/s、EAGLE-3 acceptance rate、$/1M tokens。Safety eval：Llama Guard 4 pass rate。Model card：bias evaluations、reproducibility seeds、data licensing。

## 架构

```
raw data (HF datasets + internal)
    |
    v
Datatrove dedup + Nemotron-CC quality filter + PII scrub
    |
    v
split hygiene (MMLU-Pro contamination check)
    |
    v
Axolotl SFT config (YAML)  ---> 8xH100, ZeRO-3
    |
    v
TRL DPO / GRPO config       ---> 4xH100, 1 epoch
    |
    v
GPTQ + AWQ + GGUF quantize
    |
    v
vLLM 0.7 + EAGLE-3 speculative decoding
    |
    v
K8s deployment, HPA on queue-wait
    |
    v
lm-eval-harness + RewardBench-2 + MT-Bench-v2 + MMLU-Pro
    |
    v
model card (2026 MOF) + safety eval (Llama Guard 4)
```

## 技术栈

- Data：Datatrove 用于 dedup，Nemotron-CC classifier 用于 quality，Presidio 用于 PII
- Base：Llama 3.3 8B、Qwen3 14B 或 Gemma 3 12B
- SFT：Axolotl v0.8，配 ZeRO-3、Flash Attention 3、packed sequences
- Preference tuning：TRL 0.15 用于 DPO 或 GRPO；Unsloth 用于 single-GPU iteration
- Quantization：GPTQ（Marlin）、AWQ、通过 llama.cpp 做 GGUF
- Serving：vLLM 0.7 搭配 EAGLE-3 speculative decoding（或 SGLang 0.4 + SpecForge）
- Eval：lm-evaluation-harness、RewardBench-2、MT-Bench-v2、MMLU-Pro
- Safety eval：Llama Guard 4、ShieldGemma-2
- Infrastructure：Kubernetes + NVIDIA device plugin，HPA on queue-wait metric
- Observability：W&B 用于 training，Langfuse 用于 inference

## 构建它

1. **Data pipeline。** 在 raw corpus 上运行 Datatrove dedup。应用 Nemotron-CC 风格 quality classifier。Presidio 清理 PII。用显式 seed 写出 train/val splits。

2. **Contamination check。** 对每个 validation split，计算它与 MMLU-Pro、MT-Bench-v2、RewardBench-2 test sets 的 MinHash。拒绝任何 overlap。

3. **Axolotl SFT。** YAML 包含 ZeRO-3、FA3、sequence packing。在 8xH100 上训练 2-3 epochs。记录到 W&B。

4. **TRL DPO / GRPO。** 取 SFT checkpoint，在 preference pairs 上跑一个 epoch 的 DPO（或在 math/code 上用 verifiable reward 跑 GRPO）。Sweep beta。

5. **Quantize。** 生成三种 quants：GPTQ-INT4-Marlin、AWQ-INT4、llama.cpp 的 GGUF-Q4_K_M。记录 size 和 nominal throughput。

6. **用 speculative decoding serving。** vLLM 0.7 config，EAGLE-3 draft heads 通过 Red Hat Speculators 训练。度量 batch 1 / 8 / 32 下的 acceptance rate 和 tail latency。报告在同一 eval 上相对 Anthropic / OpenAI 的 $/1M tokens。

7. **Eval matrix。** 在 base、SFT-only、SFT+DPO、SFT+GRPO 上运行 lm-eval-harness、RewardBench-2、MT-Bench-v2、MMLU-Pro。生成表格。

8. **Safety eval。** Dev set 上的 Llama Guard 4 pass rate。ShieldGemma-2 output filter。

9. **Model card。** MOF 2026 template：data、training、eval、safety、license、reproducibility section，包含 YAMLs 和 commit SHAs。

## 使用它

```
$ ./pipeline.sh config/llama3.3-8b-domainX.yaml
[data]    300k deduped, 12k filtered, 280k accepted (seed=7)
[SFT]     3 epochs, 8xH100, 6h12m, val loss 1.42 -> 1.03
[DPO]     1 epoch, beta=0.08, 4xH100, 1h40m
[quant]   GPTQ-INT4 4.6 GB, AWQ-INT4 4.8 GB, GGUF-Q4_K_M 5.1 GB
[serve]   vLLM 0.7, EAGLE-3 acceptance 0.74, p99 126ms @ bs=8
[eval]    MMLU-Pro +3.2, MT-Bench-v2 +0.41, RewardBench-2 +0.08
[card]    model-card.md generated under 2026 MOF
```

## 交付它

`outputs/skill-finetuning-pipeline.md` 描述交付物。一条命令把数据跑过 SFT、DPO、quant、serve、eval，并输出 model card + served endpoint。

| Weight | Criterion | How it is measured |
|:-:|---|---|
| 25 | 相比 base 的 eval delta | 在目标任务上的实测提升（MMLU-Pro、MT-Bench-v2、task-specific） |
| 20 | 流水线可复现性 | 一条命令用相同 seeds 端到端 rerun |
| 20 | 数据卫生 | Dedup rate、PII scrub coverage、contamination check green |
| 20 | Serving 效率 | bs=1/8/32 的 tokens/s、EAGLE-3 acceptance rate、$/1M tokens |
| 15 | Model card + safety eval | 2026 MOF 完整度 + Llama Guard 4 pass rate |
| **100** | | |

## 练习

1. 在同一个 task-specific benchmark 上运行 SFT-only vs SFT+DPO vs SFT+GRPO。报告哪种 preference method 胜出以及幅度。

2. 把 Llama 3.3 8B 换成 Qwen3 14B。度量匹配质量下的 $/1M tokens。

3. 度量 domain data vs generic ShareGPT 上的 EAGLE-3 acceptance rate。报告 delta 及其对 latency budgets 的意义。

4. 注入 1% contamination（把 MMLU-Pro 答案泄漏进训练数据）并重新 eval。观察 MMLU-Pro accuracy 不真实地跳升。构建能捕捉这一点的 contamination-check CI gate。

5. 增加 LoRA SFT 作为 full fine-tune 的替代。度量 10x 更低内存下的质量差距。

## 关键术语

| Term | What people say | What it actually means |
|------|-----------------|------------------------|
| Axolotl | “SFT trainer” | 面向 SFT、DPO 和 distillation 的统一 YAML-driven trainer |
| TRL | “Preference tuner” | Hugging Face 的 LLM DPO、GRPO、PPO 库 |
| GRPO | “Group-relative policy optimization” | DeepSeek R1 带 verifiable rewards 的 RL recipe |
| EAGLE-3 | “Speculative decoding draft” | 预测未来 N 个 tokens 的 draft heads；vLLM 用 target model 验证 |
| MOF | “Model Openness Framework” | 2026 年按 data、code、license 评级 model releases 的标准 |
| Contamination check | “Split hygiene” | 基于 MinHash 检测 test-set leakage into training |
| Acceptance rate | “EAGLE / MTP metric” | Target model 接受 drafted tokens 的比例 |

## 延伸阅读

- [Axolotl documentation](https://axolotl-ai-cloud.github.io/axolotl/) — SFT / DPO trainer 参考
- [TRL documentation](https://huggingface.co/docs/trl) — DPO 和 GRPO 参考实现
- [Unsloth](https://github.com/unslothai/unsloth) — single-GPU iteration 参考
- [DeepSeek R1 paper (arXiv:2501.12948)](https://arxiv.org/abs/2501.12948) — GRPO 方法
- [vLLM + EAGLE-3 documentation](https://docs.vllm.ai) — serving stack 参考
- [SGLang SpecForge](https://github.com/sgl-project/SpecForge) — 备用 speculative-decoding trainer
- [Model Openness Framework 2026](https://isocpp.org/) — open-release 评级标准
- [lm-evaluation-harness](https://github.com/EleutherAI/lm-evaluation-harness) — 标准 eval runner
