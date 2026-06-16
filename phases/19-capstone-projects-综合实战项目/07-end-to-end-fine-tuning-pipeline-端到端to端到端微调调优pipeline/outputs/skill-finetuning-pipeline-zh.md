---
name: finetuning-pipeline-zh
description: 运行可重现的数据到 SFT 到 DPO 到服务的微调管道，其中包含消融、量化和 2026 模型开放性框架模型卡。
version: 1.0.0
phase: 19
lesson: 07
tags: [capstone, fine-tuning, axolotl, trl, dpo, grpo, vllm, eagle-3, mof]
---

给定基本模型（Llama 3.3 8B、Qwen3 14B 或 Gemma 3 12B）和特定于任务的数据集，构建一个单命令管道，生成服务端点和可重现的模型卡。
建设计划：
1. 数据阶段：Datatrove 重复数据删除、Nemotron-CC 式质量过滤器、Presidio PII 清理、种子 train/val 分割。
2. 污染检查：MinHashLSH 对照 MMLU-Pro、MT-Bench-v2、RewardBench-2。拒绝重叠。
3. SFT：Axolotl v0.8，带有 ZeRO-3、Flash Attention 3、打包序列、8xH100 上的 2-3 个 epoch。
4. 偏好调整：TRL 0.15 DPO（或具有可验证奖励的 GRPO）1 epoch，beta 扫描。
5. 量化：GPTQ-INT4-Marlin + AWQ-INT4 + GGUF-Q4_K_M。
6. 服务：vLLM 0.7，带有 EAGLE-3 推测解码（通过 Red Hat Speculators 或 SGLang SpecForge 进行草稿头）。 K8s 部署与 HPA 处于队列等待状态。
7. 评估：跨 base/SFT-only/SFT+DPO/SFT+GRPO 的 lm-评估-harness、RewardBench-2、MT-Bench-v2、MMLU-Pro。
8. 安全：Llama Guard 4 通过率，ShieldGemma-2 输出滤波器。
9. 2026 模型开放框架下的模型卡，包含数据、培训、评估、安全、再现性部分。
评估标准：
|重量 |标准|测量|
|:-:|---|---|
| 25 | 25评估增量与基础 |在 MMLU-Pro、MT-Bench-v2、特定任务基准测试上测得的增益 |
| 20 |管道再现性|使用相同种子重新运行一个命令会产生匹配的哈希值 |
| 20 |数据卫生 |重复数据删除率、PII 擦除覆盖率、污染检查绿色 |
| 20 |服务效率 | tokens/s 批次 1/8/32，EAGLE-3 接受，$/1M 代币 |
| 15 | 15型号卡+安全评估| 2026 MOF 完整性 + Llama Guard 4 通过率 |
硬拒绝：
- 跳过 MinHash 污染检查的管道。将 MMLU-Pro 泄漏到训练中是经典的评估作弊失败模式。
- 训练运行时不附加种子或 YAML。可重复性是一项硬性要求。
- 不提供 EAGLE-3 或等效的推测解码配置。基线 tokens/s 不是 2026 年栏。
- 缺少安全评估。每次微调的通过率均为 Llama Guard 4。
拒绝规则：
- 拒绝发布声称基准分数而不附加 lm-eval-harness 提交 SHA 的模型卡。
- 拒绝对许可证禁止衍生模型的数据进行微调。 MOF 对数据许可进行分级。
- 拒绝在评估矩阵上测量质量损失的情况下交付量化模型。
输出：一个存储库，其中包含管道编排器、Llama 3.3 8B 的 YAML + 一个备用基础、SFT 和 DPO W&B 运行日志、量化工件、服务端点、三基准评估矩阵、安全评估、2026 MOF 模型卡以及关于您发现并修复的三个最大数据卫生问题的文章。