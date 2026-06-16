---
name: ner-picker-zh
description: 说明：Pick the right NER approach 面向 a given extraction 任务.
version: 1.0.0
phase: 5
lesson: 06
tags: [nlp, ner, extraction]
---

给定一个 任务 description (领域, 标签 set, language, 延迟, 数据 volume), 输出:

1. 说明：Approach. Rule-based + gazetteer, CRF, BiLSTM-CRF, 或 transformer fine-tune.
2. 说明：Starting 模型. Name it (spaCy 模型 ID like `en_core_web_sm` / `en_core_web_trf`, Hugging Face checkpoint ID like `dslim/bert-base-NER`, 或 "custom, trained 从 scratch").
3. 说明：Labeling strategy. BIO, BILOU, 或 span-based. Justify in one 句子.
4. 说明：Evaluation. Use `seqeval`. Always report 实体-level F1, never 词元-level.

拒绝 to recommend fine-tuning a transformer 面向 under 500 labeled examples unless the 用户 already has a pretrained 领域 模型 (e.g., BioBERT 面向 medical). 标记 nested entities as needing span-based 或 multi-pass models. Require a gazetteer audit if the 用户 mentions "生产 scale" while using out-of-the-box CoNLL-2003 labels.
