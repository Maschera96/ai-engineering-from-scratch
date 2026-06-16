# 子词分词，BPE、WordPiece、Unigram、SentencePiece

> 说明：词 tokenizers choke on unseen 词. Character tokenizers blow up 序列 length. Subword tokenizers split the difference. Every 现代 LLM ships on one.

**类型:** 学习
**语言:** Python
**先修要求:** 说明：Phase 5 · 01 (文本 Processing), Phase 5 · 04 (GloVe / FastText / Subword)
**时间:** 约60分钟

## 问题

Your vocabulary has 50,000 词. A 用户 types "untokenizable". Your 分词器 returns `[UNK]`. The 模型 now has no signal about the 词. Worse: the 90th-percentile 文档 in your 语料库 has 40 rare 词, 这 means 40 bits of dropped information per 文档.

Subword 分词 solves this. 常见 词 stay single 词元. Rare 词 decompose 到 meaningful pieces: `untokenizable` → `un`, `token`, `izable`. 训练 数据 covers everything 因为 any string is ultimately a 序列 of bytes.

每个 frontier LLM in 2026 ships on one of three algorithms (BPE, Unigram, WordPiece), wrapped in one of three libraries (tiktoken, SentencePiece, HF Tokenizers). You cannot ship a 语言模型 不使用 picking one.

## 概念

说明：![说明：BPE vs Unigram vs WordPiece, character-by-character](../assets/subword-tokenization.svg)

说明：**BPE (Byte-Pair Encoding).** Start 使用 a character-level vocabulary. Count every adjacent pair. Merge the most frequent pair 到 a new 词元. Repeat until you hit the target vocabulary size. Dominant 算法: GPT-2/3/4, Llama, Gemma, Qwen2, Mistral.

**Byte-level BPE.** Same 算法 but over 原始 bytes (256 base 词元) instead of Unicode characters. Guarantees zero `[UNK]` 词元，any byte 序列 encodes. GPT-2 uses 50,257 词元 (256 bytes + 50,000 merges + 1 special).

**Unigram.** Start 使用 a huge vocabulary. Assign each 词元 a unigram 概率. Iteratively prune 词元 whose removal least increases the 语料库 log-likelihood. Probabilistic at 推理: can sample tokenizations (useful 面向 数据 augmentation via subword regularization). Used by T5, mBART, ALBERT, XLNet, Gemma.

**WordPiece.** Merge pairs 这 maximize likelihood of the 训练 语料库 rather than 原始 frequency. Used by BERT, DistilBERT, ELECTRA.

**SentencePiece vs tiktoken.** SentencePiece is the 库 这 *trains* vocabularies (BPE 或 Unigram) directly on 原始 Unicode 文本, encoding whitespace as `▁`. tiktoken is OpenAI's 快 *encoder* against pre-built vocabularies; it does not train.

Rule of thumb:

- **训练 a new vocabulary:** SentencePiece (多语言, no pre-分词) 或 HF Tokenizers.
- 说明：**快 推理 against GPT vocab:** tiktoken (cl100k_base, o200k_base).
- **Both:** HF Tokenizers，one 库, 训练 + serving.

```figure
bpe-merge
```

## 动手构建

### Step 1: BPE 从 scratch

See `code/main.py`. The loop:

```python
def train_bpe(corpus, num_merges):
    vocab = {tuple(word) + ("</w>",): count for word, count in corpus.items()}
    merges = []
    for _ in range(num_merges):
        pairs = Counter()
        for symbols, freq in vocab.items():
            for a, b in zip(symbols, symbols[1:]):
                pairs[(a, b)] += freq
        if not pairs:
            break
        best = pairs.most_common(1)[0][0]
        merges.append(best)
        vocab = apply_merge(vocab, best)
    return merges
```

Three facts the 算法 encodes. `</w>` marks 词 end so "low" (suffix) 与 "lower" (prefix) stay distinct. Frequency weighting makes high-frequency pairs win early. The merge list is ordered，推理 applies merges in 训练 order.

### Step 2: encode 使用 the learned merges

```python
def encode_bpe(word, merges):
    symbols = list(word) + ["</w>"]
    for a, b in merges:
        i = 0
        while i < len(symbols) - 1:
            if symbols[i] == a and symbols[i + 1] == b:
                symbols = symbols[:i] + [a + b] + symbols[i + 2:]
            else:
                i += 1
    return symbols
```

说明：Naive O(n·|merges|). Production implementations (tiktoken, HF Tokenizers) use merge-rank lookup 使用 priority queues 与 run in near-linear time.

### Step 3: SentencePiece in practice

```python
import sentencepiece as spm

spm.SentencePieceTrainer.train(
    input="corpus.txt",
    model_prefix="my_tokenizer",
    vocab_size=8000,
    model_type="bpe",          # or "unigram"
    character_coverage=0.9995, # lower for CJK (e.g. 0.9995 for English, 0.995 for Japanese)
    normalization_rule_name="nmt_nfkc",
)

sp = spm.SentencePieceProcessor(model_file="my_tokenizer.model")
print(sp.encode("untokenizable", out_type=str))
# ['▁un', 'token', 'izable']
```

说明：Notice: no pre-分词 required, space encoded as `▁`, `character_coverage` controls how aggressively rare characters are preserved vs mapped to `<unk>`.

### Step 4: tiktoken 面向 OpenAI-compatible vocabs

```python
import tiktoken
enc = tiktoken.get_encoding("o200k_base")
print(enc.encode("untokenizable"))        # [127340, 101028]
print(len(enc.encode("Hello, world!")))   # 4
```

Encoding-only. 快 (Rust backend). Exact match 使用 GPT-4/5 分词 面向 byte-counting, 成本 estimation, context-window budgeting.

## Pitfalls 这 still ship in 2026

- **分词器 drift.** 训练 on vocab A, deploying against vocab B. 词元 IDs differ; 模型 outputs garbage. Check `tokenizer.json` hash in CI.
- 说明：**Whitespace ambiguity.** BPE "hello" vs " hello" produce different 词元. Always specify `add_special_tokens` 与 `add_prefix_space` explicitly.
- 说明：**Multilingual undertraining.** English-heavy corpora produce vocabularies 这 split non-Latin scripts 到 5-10x more 词元. Same prompt costs 5-10x more in Japanese/Arabic on GPT-3.5. o200k_base partially 固定 this.
- 说明：**Emoji splits.** A single emoji can take 5 词元. Checkpoint emoji handling when budgeting context.

## 投入使用

这个 2026 stack:

|Situation|Pick|
|-----------|------|
|训练 a monolingual 模型 从 scratch|HF Tokenizers (BPE)|
|训练 a 多语言 模型|SentencePiece (Unigram, `character_coverage=0.9995`)|
|Serving an OpenAI-compatible API|tiktoken (`o200k_base` 面向 GPT-4+)|
|领域-specific vocab (code, math, protein)|Train custom BPE on 领域 语料库, merge 使用 base vocab|
|边缘 推理, 小 模型|说明：Unigram (smaller vocabularies work better)|

Vocabulary size is a scaling decision, not a constant. Rough heuristic: 32k 面向 <1B params, 50-100k 面向 1-10B, 200k+ 面向 多语言/frontier.

## 交付成果

保存为 `outputs/skill-bpe-vs-wordpiece.md`:

```markdown
---
name: tokenizer-picker
description: Pick tokenizer algorithm, vocab size, library for a given corpus and deployment target.
version: 1.0.0
phase: 5
lesson: 19
tags: [nlp, tokenization]
---

Given a corpus (size, languages, domain) and deployment target (training from scratch / fine-tuning / API-compatible inference), output:

1. Algorithm. BPE, Unigram, or WordPiece. One-sentence reason.
2. Library. SentencePiece, HF Tokenizers, or tiktoken. Reason.
3. Vocab size. Rounded to nearest 1k. Reason tied to model size and language coverage.
4. Coverage settings. `character_coverage`, `byte_fallback`, special-token list.
5. Validation plan. Average tokens-per-word on held-out set, OOV rate, compression ratio, round-trip decode equality.

Refuse to train a character-coverage <0.995 tokenizer on corpora with rare-script content. Refuse to ship a vocab without a frozen `tokenizer.json` hash check in CI. Flag any monolingual tokenizer under 16k vocab as likely under-spec.
```

## 练习

1. **Easy.** Train a 500-merge BPE on `code/main.py`'s tiny 语料库. Encode three held-out 词. How many produced exactly 1 词元 vs >1 词元?
2. 说明：**Medium.** Compare 词元 counts on 100 English Wikipedia sentences between `cl100k_base`, `o200k_base`, 与 a SentencePiece BPE you train 使用 vocab=32k. Report the compression ratio of each.
3. **Hard.** Train the same 语料库 使用 BPE, Unigram, 与 WordPiece. Measure downstream 准确率 when using each on a 小 sentiment classifier. Does the choice move the needle by more than 1 point F1?

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|BPE|Byte-Pair Encoding|说明：Greedy merge of most-frequent character pairs until target vocab size hit.|
|Byte-level BPE|No unknown 词元 ever|BPE over 原始 256 bytes; GPT-2 / Llama use this.|
|Unigram|Probabilistic 分词器|说明：Prunes 从 a 大 candidate set using log-likelihood; used by T5, Gemma.|
|SentencePiece|这个 whitespace one|说明：Library 这 trains BPE/Unigram on 原始 文本; space encoded as `▁`.|
|tiktoken|这个 快 one|说明：OpenAI's Rust-backed BPE encoder 面向 pre-built vocabs. No 训练.|
|Merge list|这个 magic numbers|Ordered list of `(a, b) → ab` merges; 推理 applies in order.|
|Character coverage|How rare is too rare?|Fraction of characters in 训练 语料库 the 分词器 must cover; ~0.9995 typical.|

## 延伸阅读

- 说明：[说明：Sennrich, Haddow, Birch (2015). Neural Machine Translation of Rare 词 使用 Subword Units](https://arxiv.org/abs/1508.07909)，the BPE paper.
- 说明：[Kudo (2018). Subword Regularization 使用 Unigram 语言模型](https://arxiv.org/abs/1804.10959)，the Unigram paper.
- 说明：[说明：Kudo, Richardson (2018). SentencePiece: A simple 与 language independent subword 分词器](https://arxiv.org/abs/1808.06226)，the 库.
- 说明：[Hugging Face，Summary of the tokenizers](https://huggingface.co/docs/transformers/tokenizer_summary)，concise reference.
- 说明：[OpenAI tiktoken repo](https://github.com/openai/tiktoken)，cookbook + encoding list.
