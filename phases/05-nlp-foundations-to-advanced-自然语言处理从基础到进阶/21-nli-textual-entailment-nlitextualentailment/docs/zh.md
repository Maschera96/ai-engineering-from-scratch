# 自然语言推理，文本蕴含

> 说明："t entails h" means a human reading t would conclude h is true. NLI is the 任务 of predicting entailment / contradiction / neutral. Boring on the surface, load-bearing in 生产.

**类型:** 学习
**语言:** Python
**先修要求:** 说明：Phase 5 · 05 (Sentiment Analysis), Phase 5 · 13 (Question Answering)
**时间:** 约60分钟

## 问题

说明：You built a summarizer. It produced a 摘要. How do you know the 摘要 does not contain a hallucination?

说明：You built a 聊天机器人. It answered "yes." How do you know the 答案 is supported by the retrieved passage?

你需要 to classify 10,000 news articles by 主题. You have no 训练 labels. Can you reuse a 模型?

说明：All three problems reduce to Natural Language 推理. NLI asks: given a premise `t` 与 a hypothesis `h`, is `h` entailed by `t`, contradicted, 或 neutral (unrelated)?

- 说明：**Hallucination check:** `t` = source 文档, `h` = 摘要 claim. Not entailment = hallucination.
- 说明：**Grounded QA:** `t` = retrieved passage, `h` = generated 答案. Not entailment = fabrication.
- **Zero-shot 分类:** `t` = 文档, `h` = verbalized 标签 ("This is about sports"). Entailment = predicted 标签.

One 任务, three 生产 uses. This is why every RAG 评估 framework ships an NLI 模型 under the hood.

## 概念

说明：![NLI: three-way 分类, premise vs hypothesis](../assets/nli.svg)

**The three labels.**

- 说明：**Entailment.** `t` → `h`. "The cat is on the mat" entails "There is a cat."
- 说明：**Contradiction.** `t` → ¬`h`. "The cat is on the mat" contradicts "There is no cat."
- 说明：**Neutral.** No 推理 either way. "The cat is on the mat" is neutral to "The cat is hungry."

说明：**Not logical entailment.** NLI is *natural* language 推理，what a typical human reader would infer, not strict logic. "John walked his dog" entails "John has a dog" in NLI, but strict first-order logic would only admit it if you axiomatize possession.

**Datasets.**

- 说明：**SNLI** (2015). 570k human-annotated pairs, image captions as premises. Narrow 领域.
- 说明：**MultiNLI** (2017). 433k pairs across 10 genres. The standard 训练 语料库 in 2026.
- 说明：**ANLI** (2019). Adversarial NLI. Humans wrote examples specifically designed to break existing models. Harder.
- 说明：**DocNLI, ConTRoL** (2020–21). 文档-length premises. Tests multi-hop 与 长-range 推理.

说明：**The architecture.** A transformer encoder (BERT, RoBERTa, DeBERTa) reads `[CLS] premise [SEP] hypothesis [SEP]`. The `[CLS]` representation feeds a 3-way softmax. Train on MNLI, evaluate on held-out benchmarks, get 90%+ 准确率 on in-分布 pairs.

**Zero-shot via NLI.** 给定一个 文档 与 candidate labels, turn each 标签 到 a hypothesis ("This 文本 is about sports"). 计算 entailment 概率 面向 each. Pick the max. This is the mechanism behind Hugging Face's `zero-shot-classification` 流水线.

## 动手构建

### Step 1: run a pretrained NLI 模型

```python
from transformers import pipeline

nli = pipeline("text-classification",
               model="facebook/bart-large-mnli",
               top_k=None)  # return all labels; replaces deprecated return_all_scores=True

premise = "The cat is sleeping on the couch."
hypothesis = "There is a cat in the room."

result = nli({"text": premise, "text_pair": hypothesis})[0]
print(result)
# [{'label': 'entailment', 'score': 0.97},
#  {'label': 'neutral', 'score': 0.02},
#  {'label': 'contradiction', 'score': 0.01}]
```

说明：对于 生产 NLI, `facebook/bart-large-mnli` 与 `microsoft/deberta-v3-large-mnli` are the 开放 defaults. DeBERTa-v3 tops leaderboards.

### Step 2: zero-shot 分类

```python
zs = pipeline("zero-shot-classification", model="facebook/bart-large-mnli")

text = "The stock market rallied after the central bank cut interest rates."
labels = ["finance", "sports", "politics", "technology"]

result = zs(text, candidate_labels=labels)
print(result)
# {'labels': ['finance', 'politics', 'technology', 'sports'],
#  'scores': [0.92, 0.05, 0.02, 0.01]}
```

这个 template is "This example is about {标签}." by 默认. Customize 使用 `hypothesis_template`. No 训练 数据 required. No fine-tuning. Works out of the box.

### Step 3: faithfulness check 面向 RAG

```python
def is_faithful(answer, context, threshold=0.5):
    result = nli({"text": context, "text_pair": answer})[0]
    entail = next(s for s in result if s["label"] == "entailment")
    return entail["score"] > threshold
```

说明：This is the core of RAGAS faithfulness. Split the generated 答案 到 atomic claims. Check each claim against the retrieved context. Report the fraction 这 entail.

### 说明：Step 4: hand-rolled NLI classifier (conceptual)

See `code/main.py` 面向 a stdlib-only toy: premise 与 hypothesis are compared via lexical overlap + negation detection. Not competitive 使用 transformer models，but it shows the shape of the 任务: two texts in, 3-way 标签 out, loss = cross-entropy over `{entail, contradict, neutral}`.

## 陷阱

- **Hypothesis-only shortcuts.** Models can predict the 标签 从 the hypothesis alone at ~60% on SNLI 因为 "not", "nobody", "never" correlate 使用 contradiction. Strong 基线 面向 detecting 标签 leakage.
- 说明：**Lexical overlap heuristic.** The subsequence heuristic ("every subsequence is entailed") passes SNLI but fails HANS/ANLI. Use adversarial benchmarks.
- **文档-length degradation.** Single-句子 NLI models drop 20+ F1 on 文档-length premises. Use DocNLI-trained models 面向 长 context.
- **Zero-shot template sensitivity.** "This example is about {标签}" vs "{标签}" vs "The 主题 is {标签}" can swing 准确率 by 10+ points. Tune the template.
- 说明：**领域 mismatch.** MNLI trains on general English. Legal, medical, 与 scientific 文本 need 领域-specific NLI models (e.g., SciNLI, MedNLI).

## 投入使用

这个 2026 stack:

|Use case|模型|
|---------|-------|
|General-purpose NLI|`microsoft/deberta-v3-large-mnli`|
|快 / 边缘|`cross-encoder/nli-deberta-v3-base`|
|Zero-shot 分类 (lightweight)|`facebook/bart-large-mnli`|
|文档-level NLI|`MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli`|
|Multilingual|`MoritzLaurer/multilingual-MiniLMv2-L6-mnli-xnli`|
|Hallucination detection in RAG|NLI layer inside RAGAS / DeepEval|

这个 2026 meta-pattern: NLI is the duct tape of 文本 understanding. Whenever you need "does A support B?" 或 "does A contradict B?"，reach 面向 NLI before you reach 面向 another LLM call.

## 交付成果

保存为 `outputs/skill-nli-picker.md`:

```markdown
---
name: nli-picker
description: Pick an NLI model, label template, and evaluation setup for a classification / faithfulness / zero-shot task.
version: 1.0.0
phase: 5
lesson: 21
tags: [nlp, nli, zero-shot]
---

Given a use case (faithfulness check, zero-shot classification, document-level inference), output:

1. Model. Named NLI checkpoint. Reason tied to domain, length, language.
2. Template (if zero-shot). Verbalization pattern. Example.
3. Threshold. Entailment cutoff for the decision rule. Reason based on calibration.
4. Evaluation. Accuracy on held-out labeled set, hypothesis-only baseline, adversarial subset.

Refuse to ship zero-shot classification without a 100-example labeled sanity check. Refuse to use a sentence-level NLI model on document-length premises. Flag any claim that NLI solves hallucination — it reduces it; it does not eliminate it.
```

## 练习

1. 说明：**Easy.** Run `facebook/bart-large-mnli` on 20 hand-crafted (premise, hypothesis, 标签) triples covering all three classes. Measure 准确率. Add adversarial "subsequence heuristic" traps ("I did not eat the cake" vs "I ate the cake") 与 see if it breaks.
2. 说明：**Medium.** Compare the zero-shot template `"This text is about {label}"` against `"The topic is {label}"` 与 `"{label}"` on 100 AG News headlines. Report 准确率 swing.
3. 说明：**Hard.** Build a RAG faithfulness checker: atomic-claim decomposition + NLI per claim. Evaluate on 50 RAG-generated answers 使用 gold context. Measure false-positive 与 false-negative rates vs hand labels.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|NLI|Natural Language 推理|3-way 分类 of premise-hypothesis relationship.|
|RTE|Recognizing Textual Entailment|Older name 面向 NLI; same 任务.|
|Entailment|"t implies h"|说明：一个 typical reader would conclude h is true given t.|
|Contradiction|"t rules out h"|说明：一个 typical reader would conclude h is false given t.|
|Neutral|"undecided"|No 推理 从 t to h either way.|
|Zero-shot 分类|NLI as classifier|说明：Verbalize labels as hypotheses, pick max entailment.|
|Faithfulness|Is the 答案 supported?|NLI over (retrieved context, generated 答案).|

## 延伸阅读

- [Bowman et al. (2015). A 大 annotated 语料库 面向 learning natural language 推理](https://arxiv.org/abs/1508.05326)，SNLI.
- [Williams, Nangia, Bowman (2017). A Broad-Coverage Challenge 语料库 面向 句子 Understanding through 推理](https://arxiv.org/abs/1704.05426)，MultiNLI.
- [Nie et al. (2019). Adversarial NLI](https://arxiv.org/abs/1910.14599)，the ANLI 基准.
- 说明：[说明：Yin, Hay, Roth (2019). Benchmarking Zero-shot 文本 Classification](https://arxiv.org/abs/1909.00161)，NLI-as-classifier.
- 说明：[说明：He et al. (2021). DeBERTa: Decoding-enhanced BERT 使用 Disentangled 注意力](https://arxiv.org/abs/2006.03654)，the 2026 NLI workhorse.
