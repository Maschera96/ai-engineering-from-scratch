# 长上下文评估，NIAH、RULER、LongBench、MRCR

> Gemini 3 Pro advertises 10M 词元 of context. At 1M 词元, 8-needle MRCR drops to 26.3%. Advertised ≠ usable. 长-context 评估 tells you the actual capacity of the 模型 you are shipping on.

**类型:** 学习
**语言:** Python
**先修要求:** 说明：Phase 5 · 13 (Question Answering), Phase 5 · 23 (Chunking Strategies)
**时间:** 约60分钟

## 问题

你有 a 200-page contract. The 模型 claims a 1M-词元 context. You paste the contract in 与 ask: "What is the termination clause?" The 模型 answers，but answers 从 the cover page 因为 the termination clause sits at 120k 词元 deep, past 其中 the 模型 actually attends.

说明：This is the 2026 context-capacity gap. Spec sheets say 1M 或 10M. Reality says 60-70% of 这 is usable, 与 "usable" depends on the 任务.

- 说明：**Retrieval (single needle in haystack):** near-perfect up to the advertised max on frontier models.
- 说明：**Multi-hop / aggregation:** degrades sharply past ~128k on most models.
- 说明：**Reasoning over dispersed facts:** the first 任务 to fail.

长-context 评估 measures these axes. This lesson names the benchmarks, what each actually measures, 与 how to build a custom needle test 面向 your 领域.

## 概念

说明：![NIAH 基线, RULER multi-任务, LongBench holistic](../assets/long-context-eval.svg)

**Needle-in-a-Haystack (NIAH, 2023).** Place a fact ("the magic 词 is pineapple") at a controlled depth in a 长 context. Ask the 模型 to retrieve it. Sweep depth × length. The original 长-context 基准. Frontier models now saturate this; it is a necessary but not sufficient 基线.

**RULER (Nvidia, 2024).** 13 任务 types across 4 categories: 检索 (single / multi-key / multi-值), multi-hop tracing (variable tracking), aggregation (常见 词 frequency), QA. Configurable context length (4k to 128k+). Reveals models 这 saturate NIAH but fail on multi-hop. In the 2024 release, only half of 17 models claiming 32k+ context maintained 质量 at 32k.

**LongBench v2 (2024).** 503 multiple-choice questions, 8k-2M 词 contexts, six 任务 categories: single-doc QA, multi-doc QA, 长 in-context learning, 长 对话, code repo, 长 structured 数据. The 生产 基准 面向 real-world 长-context behavior.

说明：**MRCR (Multi-Round Coreference Resolution).** Multi-turn coreference at scale. 8-needle, 24-needle, 100-needle variants. Exposes how many facts a 模型 can juggle before 注意力 degrades.

说明：**NoLiMa.** "Non-lexical needle." The needle 与 the 查询 share no literal overlap; 检索 requires one step of 语义 reasoning. Harder than NIAH.

**HELMET.** Concatenates many 文档, asks a 问题 从 any one. Tests selective 注意力.

说明：**BABILong.** Embeds bAbI reasoning chains inside irrelevant haystacks. Tests reasoning-in-a-haystack, not just 检索.

### What to actually report

- 说明：**Advertised context window.** The spec-sheet number.
- 说明：**Effective 检索 length.** NIAH pass at some threshold (e.g., 90%).
- 说明：**Effective reasoning length.** Multi-hop 或 aggregation pass at 这 threshold.
- 说明：**Degradation curve.** 准确率 vs context length, plotted per 任务 type.

说明：Two numbers 面向 your spec sheet: 检索-effective 与 reasoning-effective. Usually the reasoning-effective is 25-50% of the advertised window.

## 动手构建

### Step 1: a custom NIAH 面向 your 领域

See `code/main.py`. The skeleton:

```python
def build_haystack(filler_text, needle, depth_ratio, total_tokens):
    if not (0.0 <= depth_ratio <= 1.0):
        raise ValueError(f"depth_ratio must be in [0, 1], got {depth_ratio}")
    if total_tokens <= 0:
        raise ValueError(f"total_tokens must be positive, got {total_tokens}")

    filler_tokens = tokenize(filler_text)
    needle_tokens = tokenize(needle)
    if not filler_tokens:
        raise ValueError("filler_text produced no tokens")

    # Repeat filler until long enough to fill the haystack body.
    body_len = max(total_tokens - len(needle_tokens), 0)
    while len(filler_tokens) < body_len:
        filler_tokens = filler_tokens + filler_tokens
    filler_tokens = filler_tokens[:body_len]

    insert_at = min(int(body_len * depth_ratio), body_len)
    haystack = filler_tokens[:insert_at] + needle_tokens + filler_tokens[insert_at:]
    return " ".join(haystack)


def score_niah(model, haystack, question, expected):
    answer = model.complete(f"Context: {haystack}\nQ: {question}\nA:", max_tokens=50)
    return 1 if expected.lower() in answer.lower() else 0
```

说明：Sweep `depth_ratio` ∈ {0, 0.25, 0.5, 0.75, 1.0} × `total_tokens` ∈ {1k, 4k, 16k, 64k}. Plot the heatmap. 这 is the NIAH card 面向 your target 模型.

### Step 2: a multi-needle variant

```python
def build_multi_needle(filler, needles, total_tokens):
    depths = [0.1, 0.4, 0.7]
    chunks = [filler[:int(total_tokens * 0.1)]]
    for depth, needle in zip(depths, needles):
        chunks.append(needle)
        next_chunk = filler[int(total_tokens * depth): int(total_tokens * (depth + 0.3))]
        chunks.append(next_chunk)
    return " ".join(chunks)
```

说明：Questions like "What are the three magic 词?" require retrieving all three. Single-needle success does not predict multi-needle success.

### 说明：Step 3: multi-hop variable tracing (RULER-style)

```python
haystack = """X1 = 42. ... (filler) ... X2 = X1 + 10. ... (filler) ... X3 = X2 * 2."""
question = "What is X3?"
```

说明：这个 答案 requires chaining three assignments. Frontier models at 128k often drop to 50-70% 准确率 here.

### Step 4: LongBench v2 on your stack

```python
from datasets import load_dataset
longbench = load_dataset("THUDM/LongBench-v2")

def eval_model_on_longbench(model, subset="single-doc-qa"):
    tasks = [x for x in longbench["test"] if x["task"] == subset]
    correct = 0
    for x in tasks:
        answer = model.complete(x["context"] + "\n\nQ: " + x["question"], max_tokens=20)
        if normalize(answer) == normalize(x["answer"]):
            correct += 1
    return correct / len(tasks)
```

说明：Report per-category 准确率. Aggregate scores hide big 任务-level differences.

## 陷阱

- 说明：**NIAH-only 评估.** Passing NIAH at 1M 词元 says nothing about multi-hop. Always run RULER 或 a custom multi-hop test.
- 说明：**Uniform depth sampling.** Many implementations only test depth=0.5. Test depth=0, 0.25, 0.5, 0.75, 1.0，the "lost in the middle" effect is real.
- 说明：**Lexical overlap 使用 filler.** If the needle shares keywords 使用 the filler, 检索 becomes trivial. Use NoLiMa-style non-overlapping needles.
- **Ignoring 延迟.** 1M-词元 prompts take 30-120 seconds to prefill. Measure time-to-first-词元 alongside 准确率.
- 说明：**Vendor-self-reported numbers.** OpenAI, Google, Anthropic all publish their own scores. Always re-run independently on your use case.

## 投入使用

这个 2026 stack:

|Situation|基准|
|-----------|-----------|
|Quick sanity check|Custom NIAH at 3 depths × 3 lengths|
|模型 selection 面向 生产|RULER (13 tasks) at your target length|
|Real-world QA 质量|LongBench v2 single-doc-QA subset|
|Multi-hop reasoning|BABILong 或 custom variable-tracing|
|Conversational / 对话|MRCR 8-needle at your target length|
|模型 upgrade regression|说明：固定 in-house NIAH + RULER harness, run on every new 模型|

说明：Rule of thumb 面向 生产: never trust a context window until you have NIAH + 1 reasoning 任务 at your intended length.

## 交付成果

保存为 `outputs/skill-long-context-eval.md`:

```markdown
---
name: long-context-eval
description: Design a long-context evaluation battery for a given model and use case.
version: 1.0.0
phase: 5
lesson: 28
tags: [nlp, long-context, evaluation]
---

Given a target model, target context length, and use case, output:

1. Tests. NIAH depth × length grid; RULER multi-hop; custom domain task.
2. Sampling. Depths 0, 0.25, 0.5, 0.75, 1.0 at each length.
3. Metrics. Retrieval pass rate; reasoning pass rate; time-to-first-token; cost-per-query.
4. Cutoff. Effective retrieval length (90% pass) and effective reasoning length (70% pass). Report both.
5. Regression. Fixed harness, rerun on every model upgrade, surface deltas.

Refuse to trust a context window from the model card alone. Refuse NIAH-only evaluation for any multi-hop workload. Refuse vendor self-reported long-context scores as independent evidence.
```

## 练习

1. 说明：**Easy.** Build a NIAH 使用 3 depths (0.25, 0.5, 0.75) × 3 lengths (1k, 4k, 16k). Run on any 模型. Plot pass rate as a 3×3 heatmap.
2. 说明：**Medium.** Add a 3-needle variant. Measure 检索 of all 3 at each length. Compare to single-needle pass rate at the same length.
3. **Hard.** Construct a variable-tracing 任务 (X1 → X2 → X3, 使用 3 hops) embedded in 64k of filler. Measure 准确率 across 3 frontier models. Report effective reasoning length per 模型.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|NIAH|Needle in haystack|说明：Plant a fact in filler, ask the 模型 to retrieve it.|
|RULER|NIAH on steroids|13 任务 types across 检索 / multi-hop / aggregation / QA.|
|Effective context|这个 real capacity|Length at 这 准确率 still holds above threshold.|
|Lost in the middle|Depth bias|说明：Models under-attend to content in the middle of 长 inputs.|
|Multi-needle|Many facts at once|Multiple plants; tests 注意力 juggling, not 检索 alone.|
|MRCR|Multi-round coref|8, 24, 或 100-needle coreference; exposes 注意力 saturation.|
|NoLiMa|Non-lexical needle|说明：Needle 与 查询 share no literal 词元; requires reasoning.|

## 延伸阅读

- 说明：[Kamradt (2023). Needle in a Haystack analysis](https://github.com/gkamradt/LLMTest_NeedleInAHaystack)，the original NIAH repo.
- 说明：[说明：Hsieh et al. (2024). RULER: What's the Real Context Size of Your 长-Context LMs?](https://arxiv.org/abs/2404.06654)，the multi-任务 基准.
- 说明：[Bai et al. (2024). LongBench v2](https://arxiv.org/abs/2412.15204)，real-world 长-context eval.
- 说明：[说明：Modarressi et al. (2024). NoLiMa: Non-lexical needles](https://arxiv.org/abs/2404.06666)，harder needles.
- 说明：[Kuratov et al. (2024). BABILong](https://arxiv.org/abs/2406.10149)，reasoning-in-haystack.
- 说明：[说明：Liu et al. (2024). Lost in the Middle: How Language Models Use 长 Contexts](https://arxiv.org/abs/2307.03172)，the depth-bias paper.
