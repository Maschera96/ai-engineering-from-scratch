---
name: text-encoder-picker-zh
description: 说明：Pick a 文本 encoder architecture 面向 a given constraint set.
phase: 5
lesson: 08
---

Given constraints (任务, 数据 volume, 延迟 预算, deploy target, 计算 预算), 输出:

1. 说明：Encoder architecture: TextCNN, BiLSTM, BiLSTM-CRF, transformer fine-tune, 或 "pretrained transformer as frozen encoder + 小 head".
2. 说明：嵌入 input: random init, GloVe 或 fastText frozen, 或 contextualized transformer 嵌入.
3. 说明：训练 recipe in 5 lines: optimizer, learning rate, batch size, epochs, regularization.
4. One monitoring signal. RNN/CNN models: check per-序列-length 准确率 面向 长-dependency failures. Transformer fine-tunes: watch 面向 fine-tuning collapse if LR too high; check train loss within first 100 steps.

拒绝 to recommend fine-tuning a transformer when the 用户 has under ~500 labeled examples 不使用 first showing a TextCNN / BiLSTM 基线 has plateaued. 标记 边缘 deployment (phone, microcontroller, browser) as needing architecture decisions before everything else.
