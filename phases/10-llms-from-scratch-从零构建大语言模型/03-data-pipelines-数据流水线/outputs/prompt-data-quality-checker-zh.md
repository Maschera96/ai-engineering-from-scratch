---
name: prompt-data-quality-checker-zh
description: 验证 and 调试 数据 质量 in LLM 预训练 pipelines
version: 1.0.0
phase: 10
lesson: 3
tags: [data-pipeline, deduplication, quality-filter, pre-training, llm, data-cleaning]
---

# 数据 质量 Checker for LLM 预训练

当building or auditing a 数据 流水线 for LLM 预训练, use this framework to catch problems before they reach the 模型.

## Red Flags in 流水线 输出

**Deduplication removed less than 20% of web 数据.** Common Crawl typically contains 30-40% duplicates. If your dedup 步骤 removes less than 20%, your MinHash 参数 are too conservative or your 阈值 is too high. Check: shingle size k, number of hash 函数, number of LSH bands, Jaccard 阈值.

**压缩 比例 below 2.0 chars/词元.** This means your 分词器 is splitting too aggressively. Either retrain with more merges, increase 词表 size, or check that pre-tokenization is not fragmenting 文本 unnecessarily.

**压缩 比例 above 6.0 chars/词元.** Your 分词器 has learned very domain-specific merges that may not generalize. This is fine for a domain-specific 模型 but a warning sign for general-purpose 模型.

**序列 utilization below 90%.** Too much padding. Either your 文档 are very short (filter them or increase minimum 文档 length) or your 序列 packing is inefficient (switch from naive padding to multi-document packing).

**Vocab utilization below 50%.** More than half your 词表 is unused on this 语料库. Either the 词表 is too large for your 领域 or the 分词器 was 训练后的 on very different 数据.

## 质量 Filter Calibration

运行these checks on a random 样本 of 1,000 文档 at each 流水线 stage:

1. **Read 20 random 文档 after cleaning.** Do they contain residual HTML, JavaScript, navigation 文本, or boilerplate? If yes, your HTML stripping is incomplete.

2. **Read 20 random 文档 that PASSED the 质量 filter.** Are any of them spam, keyword lists, or machine-generated? If yes, tighten the filter thresholds.

3. **Read 20 random 文档 that FAILED the 质量 filter.** Are any of them genuinely good content? If yes, your filter is too aggressive. Relax thresholds or add exceptions for specific patterns.

4. **Read 20 random near-duplicate pairs from dedup.** Are they actually similar? If not, lower the Jaccard 阈值 or increase the number of hash 函数.

## 数据 Mixing Ratios

There is no universal formula. Start with these baselines and adjust based on 评估:

|Category|Llama 3 比例|Starting Point|
|----------|--------------|----------------|
|Web 文本|50%|50%|
|Code|25%|15-25%|
|Books/academic|13%|10-15%|
|Math|8%|5-10%|
|Multilingual web|4%|5-10%|

Increase code 比例 if the 模型 should be strong at programming. Increase math 比例 if 推理 matters. Decrease web 比例 if you need less 噪声. Always evaluate on benchmarks after changing ratios.

## 扩展 Estimates

For a given 目标 词元 count:

- 1T 词元 from web: expect ~3-5TB raw 文本, ~1.5-2TB after cleaning and dedup
- Tokenization speed (Rust): ~100M 词元/second per core
- Tokenization speed (Python): ~1-10M 词元/second per core
- MinHash dedup at 128 hashes, 16 bands: ~10K 文档/second per core
- 序列 packing: I/O bound, use memory-mapped files for corpora above 10GB

For 15T 词元 (Llama 3 规模), plan for ~30-50TB of raw 输入 数据, 1-2 weeks of preprocessing on a 64-core machine, and 100TB+ of disk for intermediate files.

## Checklist Before 训练

1. Total 词元 count matches your 计算 预算 (use Chinchilla 扩展 or the Llama 3 overtrain 比例 as a guide)
2. Dedup removed 30-40% of web 数据
3. 质量 filter removed 10-20% of remaining 数据
4. 压缩 比例 is 3-5 chars/词元 for English
5. 序列 utilization is above 95%
6. Random spot-checks show clean, coherent 文本 at every 流水线 stage
7. 数据 mix ratios have been validated on a small-scale 训练 run
8. PII removal has been verified on a 样本
9. All binary formats (packed sequences, 词元 ID arrays) pass round-trip encoding/decoding tests
10. 流水线 is reproducible: same 输入 produces identical 输出 with fixed random 种子
