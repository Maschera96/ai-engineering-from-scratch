# 多语言自然语言处理

> One 模型, 100+ languages, zero 训练 数据 面向 most of them. Cross-lingual transfer is the practical miracle of the 2020s.

**类型:** 学习
**语言:** Python
**先修要求:** 说明：Phase 5 · 04 (GloVe, FastText, Subword), Phase 5 · 11 (Machine Translation)
**时间:** 约45分钟

## 问题

English has billions of labeled examples. Urdu has thousands. Maithili has almost none. Any practical NLP 系统 这 serves a global audience has to work on the 长 tail of languages 其中 任务-specific 训练 数据 does not exist.

Multilingual models solve this by 训练 one 模型 on many languages simultaneously. The shared representation lets the 模型 transfer skills learned in high-resource languages to low-resource ones. Fine-tune the 模型 on English sentiment analysis, 与 it produces surprisingly good sentiment predictions on Urdu out of the box. 这 is zero-shot cross-lingual transfer, 与 it has reshaped how NLP ships to the world.

本课 names the tradeoffs, the canonical models, 与 the one decision 这 trips up teams new to 多语言 work: picking a source language 面向 transfer.

## 概念

![Cross-lingual transfer via shared 多语言 嵌入 space](../assets/multilingual.svg)

**Shared vocabulary.** Multilingual models use a SentencePiece 或 WordPiece 分词器 trained on 文本 从 all target languages. The vocabulary is shared: the same subword unit represents the same morpheme across related languages. `anti-` in English 与 Italian gets the same 词元.

**Shared representation.** A transformer pretrained on masked language modeling across many languages learns 这 semantically similar sentences in different languages produce similar hidden states. mBERT, XLM-R, 与 NLLB all exhibit this. 嵌入 面向 "cat" in English 聚类 near "chat" in French 与 "gato" in Spanish, 与 so do full-句子 嵌入.

**Zero-shot transfer.** Fine-tune the 模型 on labeled 数据 in one language (usually English). At 推理, run it on any other language the 模型 supports. No target-language labels needed. Results are strong 面向 typologically related languages 与 weaker 面向 distant ones.

**Few-shot fine-tuning.** Add 100-500 labeled examples in the target language. 准确率 jumps to 95-98% of the English 基线 on 分类 tasks. This is the single most 成本-effective lever in 多语言 NLP.

## 这个 models

|模型|Year|Coverage|Notes|
|-------|------|----------|-------|
|mBERT|2018|104 languages|说明：Trained on Wikipedia. First practical 多语言 LM. Weak on low-resource.|
|XLM-R|2019|100 languages|说明：Trained on CommonCrawl (much larger than Wikipedia). Sets the cross-lingual 基线. Base 270M, 大 550M.|
|XLM-V|2023|100 languages|说明：XLM-R 使用 1M-词元 vocabulary (vs 250k). Better on low-resource.|
|mT5|2020|101 languages|T5 architecture 面向 多语言 生成.|
|NLLB-200|2022|200 languages|Meta's 翻译 模型; includes 55 low-resource languages.|
|BLOOM|2022|46 languages + 13 programming|开放 176B LLM trained multilingually.|
|Aya-23|2024|23 languages|说明：Cohere's 多语言 LLM. Strong on Arabic, Hindi, Swahili.|

Pick by use case. Classification works well 使用 XLM-R-base as the sane 默认. Generation tasks call 面向 mT5 或 NLLB depending on 翻译 vs 开放 生成. LLM-style work pairs 使用 Aya-23 或 Claude using explicit 多语言 prompting.

## 这个 source-language decision (2026 research)

说明：大多数 teams 默认 to English as the fine-tuning source. Recent research (2026) shows this is often 错误.

Language 相似度 predicts transfer 质量 better than 原始 语料库 size. 面向 Slavic targets, German 或 Russian often beat English. 面向 Indic targets, Hindi often beats English. The **qWALS** 相似度 指标 (2026, based on World Atlas of Language Structures features) quantifies this. **LANGRANK** (Lin et al., ACL 2019) is a separate, earlier 方法 这 ranks candidate source languages 从 a combination of linguistic 相似度, 语料库 size, 与 genetic relatedness.

说明：Practical rule: if your target language has a typologically close high-resource relative, try fine-tuning on 这 one first, then compare to English fine-tune.

## 动手构建

### Step 1: zero-shot cross-lingual 分类

```python
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

tok = AutoTokenizer.from_pretrained("joeddav/xlm-roberta-large-xnli")
model = AutoModelForSequenceClassification.from_pretrained("joeddav/xlm-roberta-large-xnli")


def classify(text, candidate_labels, hypothesis_template="This text is about {}."):
    scores = {}
    for label in candidate_labels:
        hypothesis = hypothesis_template.format(label)
        inputs = tok(text, hypothesis, return_tensors="pt", truncation=True)
        with torch.no_grad():
            logits = model(**inputs).logits[0]
        entail_score = torch.softmax(logits, dim=-1)[2].item()
        scores[label] = entail_score
    return dict(sorted(scores.items(), key=lambda x: -x[1]))


print(classify("I love this product!", ["positive", "negative", "neutral"]))
print(classify("मुझे यह उत्पाद पसंद है!", ["positive", "negative", "neutral"]))
print(classify("J'adore ce produit !", ["positive", "negative", "neutral"]))
```

说明：One 模型, three languages, same API. XLM-R trained on NLI 数据 transfers well to 分类 via the entailment trick.

### Step 2: 多语言 嵌入 space

```python
from sentence_transformers import SentenceTransformer
import numpy as np

model = SentenceTransformer("sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2")

pairs = [
    ("The cat is sleeping.", "Le chat dort."),
    ("The cat is sleeping.", "El gato está durmiendo."),
    ("The cat is sleeping.", "Die Katze schläft."),
    ("The cat is sleeping.", "The dog is barking."),
]

for eng, other in pairs:
    emb_eng = model.encode([eng], normalize_embeddings=True)[0]
    emb_other = model.encode([other], normalize_embeddings=True)[0]
    sim = float(np.dot(emb_eng, emb_other))
    print(f"  {eng!r} <-> {other!r}: cos={sim:.3f}")
```

Translations land close in 嵌入 space. A different English 句子 lands further. This is what makes cross-lingual 检索, clustering, 与 相似度 work.

### Step 3: few-shot fine-tuning strategy

```python
from transformers import TrainingArguments, Trainer
from datasets import Dataset


def few_shot_finetune(base_model, base_tokenizer, examples):
    ds = Dataset.from_list(examples)

    def tokenize_fn(ex):
        out = base_tokenizer(ex["text"], truncation=True, max_length=128)
        out["labels"] = ex["label"]
        return out

    ds = ds.map(tokenize_fn)
    args = TrainingArguments(
        output_dir="out",
        per_device_train_batch_size=8,
        num_train_epochs=5,
        learning_rate=2e-5,
        save_strategy="no",
    )
    trainer = Trainer(model=base_model, args=args, train_dataset=ds)
    trainer.train()
    return base_model
```

对于 100-500 target-language examples, `num_train_epochs=5` 与 `learning_rate=2e-5` are the safe defaults. Higher learning rates cause the 多语言 alignment to collapse 与 you get an English-only 模型.

## Evaluation 这 actually works

- 说明：**Per-language 准确率 on held-out sets.** Not aggregated. The aggregate hides the 长 tail.
- **基准 against monolingual 基线.** 面向 languages 使用 enough 数据, a monolingual 模型 trained 从 scratch sometimes beats the 多语言 one. Test.
- 说明：**Entity-level tests.** Named entities in the target language. Multilingual models often have weak 分词 面向 scripts far 从 Latin.
- 说明：**Cross-lingual consistency.** Same meaning in two languages should produce the same prediction. Measure the gap.

## 投入使用

这个 2026 stack:

|任务|推荐|
|-----|-------------|
|Classification, 100 languages|XLM-R-base (~270M) fine-tuned|
|Zero-shot 文本 分类|`joeddav/xlm-roberta-large-xnli`|
|Multilingual 句子 嵌入|`sentence-transformers/paraphrase-multilingual-MiniLM-L12-v2`|
|Translation, 200 languages|`facebook/nllb-200-distilled-600M` (see lesson 11)|
|Generative 多语言|Claude, GPT-4, Aya-23, mT5-XXL|
|Low-resource language NLP|说明：XLM-V 或 a 领域-specific fine-tune on related high-resource language|

说明：Always 预算 面向 fine-tuning in the target language if performance matters. Zero-shot is a starting point, not a final 答案.

### 这个 分词 tax (what goes 错误 面向 low-resource languages)

Multilingual models share one 分词器 across all their languages. 这 vocabulary is trained on a 语料库 dominated by English, French, Spanish, Chinese, German. 面向 any language outside the dominant set, three taxes compound silently:

- **Fertility tax.** Low-resource language 文本 tokenizes 到 far more 词元 per 词 than English. A Hindi 句子 can need 3-5x the 词元 of an equivalent English 句子. 这 3-5x eats your context window, 训练 efficiency, 与 延迟.
- **Variant recovery tax.** Every typo, diacritic variant, Unicode normalization mismatch, 或 case variation becomes a cold-start unrelated 序列 in 嵌入 space. The 模型 cannot learn orthographic correspondences 这 a native speaker takes as obvious.
- **Capacity spillover tax.** Taxes 1 与 2 consume context positions, layer depth, 与 嵌入 dimensions. What remains 面向 actual reasoning is systematically smaller than what a high-resource language gets 从 the same 模型.

这个 practical symptom: your 模型 trains normally on Hindi, the loss curve looks right, eval perplexity looks reasonable, 与 生产 outputs are subtly 错误. Morphology collapses mid-句子. Rare inflections stay unrecoverable. **You cannot 数据-scale your way out of a broken 分词器.**

Mitigations: pick a 分词器 使用 good coverage 面向 your target language (XLM-V's 1M-词元 vocabulary is a direct fix); verify 分词 fertility on held-out target 文本 before 训练; use byte-level fallback (SentencePiece `byte_fallback=True`, GPT-2-style byte-level BPE) 面向 truly 长-tail scripts so nothing is ever OOV.

## 交付成果

保存为 `outputs/skill-multilingual-picker.md`:

```markdown
---
name: multilingual-picker
description: Pick source language, target model, and evaluation plan for a multilingual NLP task.
version: 1.0.0
phase: 5
lesson: 18
tags: [nlp, multilingual, cross-lingual]
---

Given requirements (target languages, task type, available labeled data per language), output:

1. Source language for fine-tuning. Default English; check LANGRANK or qWALS if target language has a typologically close high-resource language.
2. Base model. XLM-R (classification), mT5 (generation), NLLB (translation), Aya-23 (generative LLM).
3. Few-shot budget. Start with 100-500 target-language examples if available. Zero-shot only if labeling is infeasible.
4. Evaluation plan. Per-language accuracy (not aggregate), cross-lingual consistency, entity-level F1 on non-Latin scripts.

Refuse to ship a multilingual model without per-language evaluation — aggregate metrics hide long-tail failures. Flag scripts with low tokenization coverage (Amharic, Tigrinya, many African languages) as needing a model with byte-fallback (SentencePiece with byte_fallback=True, or byte-level tokenizer like GPT-2).
```

## 练习

1. **Easy.** Run the zero-shot 分类 流水线 on 10 sentences per language across English, French, Hindi, 与 Arabic. Report 准确率 on each. You should see strong French, decent Hindi, variable Arabic.
2. **Medium.** Use `paraphrase-multilingual-MiniLM-L12-v2` to build a cross-lingual retriever over a 小 mixed-language 语料库. Query in English, retrieve 文档 in any language. Measure 召回率@5.
3. **Hard.** Compare English-source 与 Hindi-source fine-tuning 面向 a Hindi 分类 任务. Use 500 target-language examples 面向 few-shot fine-tuning under both regimes. Report 这 source produces better Hindi 准确率 与 by how much. This is the LANGRANK thesis in miniature.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Multilingual 模型|One 模型, many languages|说明：Shared vocabulary 与 parameters across languages.|
|Cross-lingual transfer|Train on one language, run on another|说明：Fine-tune on source, evaluate on target 不使用 target-language labels.|
|Zero-shot|No target-language labels|说明：Transfer 不使用 fine-tuning on the target language.|
|Few-shot|小 target labels|说明：100-500 target-language examples used 面向 fine-tuning.|
|mBERT|First 多语言 LM|104-language BERT pretrained on Wikipedia.|
|XLM-R|Standard cross-lingual 基线|说明：100-language RoBERTa pretrained on CommonCrawl.|
|NLLB|Meta's 200-language MT|说明：No Language Left Behind. Includes 55 low-resource languages.|

## 延伸阅读

- 说明：[说明：Conneau et al. (2019). Unsupervised Cross-lingual Representation Learning at Scale](https://arxiv.org/abs/1911.02116)，the XLM-R paper.
- 说明：[说明：Pires, Schlinger, Garrette (2019). How Multilingual is Multilingual BERT?](https://arxiv.org/abs/1906.01502)，the analysis paper 这 started the cross-lingual transfer research line.
- 说明：[Costa-jussà et al. (2022). No Language Left Behind](https://arxiv.org/abs/2207.04672)，NLLB-200 paper.
- [Üstün et al. (2024). Aya 模型: An Instruction Finetuned 开放-Access Multilingual 语言模型](https://arxiv.org/abs/2402.07827)，Aya, Cohere's 多语言 LLM.
- 说明：[说明：Language 相似度 Predicts Cross-Lingual Transfer Learning Performance (2026)](https://www.mdpi.com/2504-4990/8/3/65)，the qWALS / LANGRANK source-language paper.
