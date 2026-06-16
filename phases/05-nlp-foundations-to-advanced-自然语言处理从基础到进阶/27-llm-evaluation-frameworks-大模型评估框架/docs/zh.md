# 大模型评估，RAGAS、DeepEval、G-Eval

> Exact-match 与 F1 miss 语义 equivalence. Human review does not scale. LLM-as-judge is the 生产 答案，使用 enough calibration to trust the number.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 5 · 13 (Question Answering), Phase 5 · 14 (Information Retrieval)
**时间:** 约75分钟

## 问题

Your RAG 系统 answers: "June 29th, 2007."
这个 gold reference is: "June 29, 2007."
说明：Exact Match scores 0. F1 scores ~75%. A human would score 100%.

说明：Now multiply by 10,000 test cases. Multiply again by every change to the retriever, 分块, prompt, 或 模型. You need an evaluator 这 understands meaning, runs cheaply at scale, does not lie about regressions, 与 surfaces the right failure modes.

2026 has three frameworks 这 own this problem.

- **RAGAS.** Retrieval-Augmented Generation ASsessment. Four RAG metrics (faithfulness, 答案-relevance, context-精确率, context-召回率) 使用 NLI + LLM-judge backends. Research-backed, lightweight.
- 说明：**DeepEval.** Pytest 面向 LLMs. G-Eval, 任务-completion, hallucination, bias metrics. CI/CD-native.
- 说明：**G-Eval.** A 方法 (与 a DeepEval 指标): LLM-as-judge 使用 chain-of-thought, custom criteria, 0-1 score.

说明：All three lean on LLM-as-judge. This lesson builds intuition 面向 the 方法 与 the trust layer around it.

## 概念

说明：![说明：Four 评估 dimensions, LLM-as-judge architecture](../assets/llm-evaluation.svg)

说明：**LLM-as-judge.** Replace a static 指标 使用 an LLM 这 scores outputs given a rubric. Given `(query, context, answer)`, prompt a judge LLM: "Score 0-1 on faithfulness." Return the score.

说明：为什么 it works: LLMs approximate human judgment at a tiny fraction of the 成本. GPT-4o-mini at ~$0.003 per scored case enables 1000-sample regression eval runs 面向 under $5.

为什么 it fails silently:

1. 说明：**Judge bias.** Judges prefer longer answers, answers 从 their own 模型 family, answers 这 match the prompt style.
2. 说明：**JSON parsing failures.** Bad JSON → NaN score → silently excluded 从 the aggregate. RAGAS users know this pain. Gate 使用 try/except + explicit 失败模式.
3. 说明：**Drift over 模型 versions.** Upgrading the judge changes every 指标. Freeze judge 模型 + version.

**The RAG four.**

|Metric|Question|Backend|
|--------|----------|---------|
|Faithfulness|说明：Does each claim in the 答案 come 从 the retrieved context?|NLI-based entailment|
|Answer relevance|Does the 答案 address the 问题?|说明：Generate hypothetical questions 从 答案; compare to real 问题|
|Context 精确率|说明：Of retrieved chunks, what fraction were relevant?|LLM-judge|
|Context 召回率|Did 检索 return everything needed?|LLM-judge against gold 答案|

**G-Eval.** Define a custom criterion: "Did the 答案 cite the 正确 source?" The framework auto-expands 到 chain-of-thought 评估 steps, then scores 0-1. Good 面向 领域-specific 质量 dimensions RAGAS does not cover.

说明：**Calibration.** Never trust the 原始 judge score until you have a correlation against human labels. Run 100 hand-labeled examples. Plot judge vs human. 计算 Spearman rho. If rho < 0.7, your judge rubric needs work.

## 动手构建

### Step 1: faithfulness 使用 NLI (RAGAS-style)

```python
from typing import Callable
from transformers import pipeline

nli = pipeline("text-classification",
               model="MoritzLaurer/DeBERTa-v3-large-mnli-fever-anli-ling-wanli",
               top_k=None)

# `llm` is any callable: prompt str -> generated str.
# Example: llm = lambda p: client.messages.create(model="claude-haiku-4-5", ...).content[0].text
LLM = Callable[[str], str]


def atomic_claims(answer: str, llm: LLM) -> list[str]:
    prompt = f"""Break this answer into simple factual claims (one per line):
{answer}
"""
    return llm(prompt).splitlines()


def faithfulness(answer: str, context: str, llm: LLM) -> float:
    claims = atomic_claims(answer, llm)
    if not claims:
        return 0.0
    supported = 0
    for claim in claims:
        result = nli({"text": context, "text_pair": claim})[0]
        entail = next((s for s in result if s["label"] == "entailment"), None)
        if entail and entail["score"] > 0.5:
            supported += 1
    return supported / len(claims)
```

说明：Decompose the 答案 到 atomic claims. NLI-check each claim against the retrieved context. Faithfulness = fraction supported.

### Step 2: 答案 relevance

```python
import numpy as np
from sentence_transformers import SentenceTransformer

# encoder: any model implementing .encode(texts, normalize_embeddings=True) -> ndarray
# e.g., encoder = SentenceTransformer("BAAI/bge-small-en-v1.5")

def answer_relevance(question: str, answer: str, encoder, llm: LLM, n: int = 3) -> float:
    prompt = f"Write {n} questions this answer could be the answer to:\n{answer}"
    generated = [line for line in llm(prompt).splitlines() if line.strip()][:n]
    if not generated:
        return 0.0
    q_emb = np.asarray(encoder.encode([question], normalize_embeddings=True)[0])
    g_embs = np.asarray(encoder.encode(generated, normalize_embeddings=True))
    sims = [float(q_emb @ g_emb) for g_emb in g_embs]
    return sum(sims) / len(sims)
```

说明：如果 the 答案 implies different questions than the one asked, relevance drops.

### Step 3: G-Eval custom 指标

```python
from deepeval.metrics import GEval
from deepeval.test_case import LLMTestCaseParams, LLMTestCase

metric = GEval(
    name="Correctness",
    criteria="The answer should be factually accurate and match the expected output.",
    evaluation_steps=[
        "Read the expected output.",
        "Read the actual output.",
        "List factual claims in the actual output.",
        "For each claim, mark supported or unsupported by the expected output.",
        "Return score = fraction supported.",
    ],
    evaluation_params=[LLMTestCaseParams.INPUT, LLMTestCaseParams.ACTUAL_OUTPUT, LLMTestCaseParams.EXPECTED_OUTPUT],
)

test = LLMTestCase(input="When was the first iPhone released?",
                   actual_output="June 29th, 2007.",
                   expected_output="June 29, 2007.")
metric.measure(test)
print(metric.score, metric.reason)
```

说明：这个 评估 steps are the rubric. Explicit steps are more stable than implicit "score 0-1" prompts.

### Step 4: CI gate

```python
import deepeval
from deepeval.metrics import FaithfulnessMetric, ContextualRelevancyMetric


def test_rag_system():
    cases = load_regression_cases()
    faith = FaithfulnessMetric(threshold=0.85)
    rel = ContextualRelevancyMetric(threshold=0.7)
    for case in cases:
        faith.measure(case)
        assert faith.score >= 0.85, f"faithfulness regression on {case.id}"
        rel.measure(case)
        assert rel.score >= 0.7, f"relevancy regression on {case.id}"
```

说明：Ship as a pytest file. Run on every PR. Block merges on regressions.

### Step 5: toy eval 从 scratch

See `code/main.py`. Stdlib-only approximations of faithfulness (overlap of 答案 claims 使用 context) 与 relevance (overlap of 答案 词元 使用 问题 词元). Not 生产. Shows the shape.

## 陷阱

- 说明：**No calibration.** A judge 使用 0.3 correlation to human labels is 噪声. Require a calibration run before shipping.
- 说明：**Self-评估.** Using the same LLM to generate 与 judge inflates scores by 10-20%. Use a different 模型 family 面向 the judge.
- 说明：**Positional bias in pairwise judging.** Judges prefer the first option presented. Always randomize order 与 run both.
- 说明：**原始 aggregate hides failures.** Mean score 0.85 often hides 5% catastrophic failures. Always inspect the bottom quantile.
- 说明：**Golden dataset rot.** Unversioned eval sets 这 drift over time break longitudinal comparison. Tag the dataset 使用 every change.
- **LLM 成本.** At scale, judge calls dominate 成本. Use the cheapest 模型 这 meets calibration threshold. GPT-4o-mini, Claude Haiku, Mistral-小.

## 投入使用

这个 2026 stack:

|Use case|Framework|
|---------|-----------|
|RAG 质量 monitoring|RAGAS (4 metrics)|
|CI/CD regression gates|DeepEval + pytest|
|Custom 领域 criteria|G-Eval within DeepEval|
|Online live-traffic monitoring|RAGAS 使用 reference-free mode|
|Human-in-the-loop spot checks|LangSmith 或 Phoenix 使用 annotation UI|
|Red-teaming / safety eval|Promptfoo + DeepEval|

说明：Typical stack: RAGAS 面向 monitoring, DeepEval 面向 CI, G-Eval 面向 novel dimensions. Run all three; they disagree usefully.

## 交付成果

保存为 `outputs/skill-eval-architect.md`:

```markdown
---
name: eval-architect
description: Design an LLM evaluation plan with calibrated judge and CI gates.
version: 1.0.0
phase: 5
lesson: 27
tags: [nlp, evaluation, rag]
---

Given a use case (RAG / agent / generative task), output:

1. Metrics. Faithfulness / relevance / context-precision / context-recall + any custom G-Eval metrics with criteria.
2. Judge model. Named model + version, rationale for cost vs accuracy.
3. Calibration. Hand-labeled set size, target Spearman rho vs human > 0.7.
4. Dataset versioning. Tag strategy, change log, stratification.
5. CI gate. Thresholds per metric, regression-window logic, bottom-quantile alert.

Refuse to rely on a judge untested against ≥50 human-labeled examples. Refuse self-evaluation (same model generates + judges). Refuse aggregate-only reporting without bottom-10% surfacing. Flag any pipeline where judge upgrade lands without parallel baseline eval.
```

## 练习

1. 说明：**Easy.** Use RAGAS on 10 RAG examples 使用 known hallucinations. Verify the faithfulness 指标 catches each one.
2. 说明：**Medium.** Hand-标签 50 QA answers 0-1 面向 correctness. Score 使用 G-Eval. Measure Spearman rho between judge 与 human.
3. 说明：**Hard.** Build a pytest CI gate 使用 DeepEval. Intentionally regress the retriever. Verify the gate fails. Add bottom-quantile alerting via threshold check on the lowest 10%.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|LLM-as-judge|Scoring 使用 an LLM|说明：Prompt a judge 模型 to score outputs 0-1 given a rubric.|
|RAGAS|这个 RAG 指标 库|说明：开放-source eval framework 使用 4 reference-free RAG metrics.|
|Faithfulness|Is the 答案 grounded?|说明：Fraction of 答案 claims entailed by retrieved context.|
|Context 精确率|Were retrieved chunks relevant?|说明：Fraction of top-K chunks 这 actually mattered.|
|Context 召回率|Did 检索 find everything?|说明：Fraction of gold-答案 claims supported by retrieved chunks.|
|G-Eval|Custom LLM judge|Rubric + chain-of-thought eval steps + 0-1 score.|
|Calibration|Trust but verify|说明：Spearman correlation between judge score 与 human score.|

## 延伸阅读

- 说明：[说明：Es et al. (2023). RAGAS: Automated Evaluation of Retrieval Augmented Generation](https://arxiv.org/abs/2309.15217)，the RAGAS paper.
- 说明：[说明：Liu et al. (2023). G-Eval: NLG Evaluation using GPT-4 使用 Better Human Alignment](https://arxiv.org/abs/2303.16634)，the G-Eval paper.
- [DeepEval docs](https://deepeval.com/docs/metrics-introduction)，开放 生产 stack.
- 说明：[说明：Zheng et al. (2023). Judging LLM-as-a-Judge 使用 MT-Bench 与 Chatbot Arena](https://arxiv.org/abs/2306.05685)，biases, calibration, limits.
- 说明：[MLflow GenAI Scorer](https://mlflow.org/blog/third-party-scorers)，unifying framework 这 integrates RAGAS, DeepEval, Phoenix.
