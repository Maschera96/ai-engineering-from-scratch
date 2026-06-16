# 用于文本的 CNN 与 RNN

> 说明：Convolutions learn n-grams. Recurrences remember. Both are superseded by 注意力. Both still matter on constrained hardware.

**类型:** 构建
**语言:** Python
**先修要求:** 说明：Phase 3 · 11 (PyTorch Intro), Phase 5 · 03 (词 嵌入), Phase 4 · 02 (Convolutions 从 Scratch)
**时间:** 约75分钟

## 问题

说明：TF-IDF 与 Word2Vec produced flat 向量 这 ignored 词 order. A classifier built on them could not tell `dog bites man` 从 `man bites dog`. 词 order sometimes carries the signal.

说明：Two families of architectures filled 这 gap before transformers arrived.

**Convolutional nets 面向 文本 (TextCNN).** Apply 1D convolutions over sequences of 词 嵌入. A filter of width 3 is a learnable trigram detector: it spans three 词 与 outputs a score. Stack different widths (2, 3, 4, 5) to detect multi-scale patterns. Max-pool to a 固定-size representation. Flat, parallel, 快.

**Recurrent nets (RNN, LSTM, GRU).** Process 词元 one at a time, maintaining a hidden 状态 这 carries information forward. Sequential, memory-bearing, flexible input lengths. Dominated 序列 modeling 从 2014 to 2017, then 注意力 happened.

说明：本课 builds both, then names the failure 这 motivated 注意力.

## 概念

说明：**TextCNN** (Kim, 2014). 词元 get embedded. A width-`k` 1D convolution slides a filter over consecutive `k`-grams of 嵌入, producing a feature map. Global max-pooling over 这 map picks the strongest activation. Concatenate max-pooled outputs 从 several filter widths. Feed to a classifier head.

为什么 it works. A filter is a learnable n-gram. Max-pooling is position-invariant, so "not good" fires the same feature at the start 或 middle of a review. Three filter widths 使用 100 filters each gives you 300 learned n-gram detectors. 训练 is parallel; no sequential dependency.

**RNN.** At each time step `t`, the hidden 状态 `h_t = f(W * x_t + U * h_{t-1} + b)`. Share `W`, `U`, `b` across time. The hidden 状态 at time `T` is a 摘要 of the entire prefix. 面向 分类, pool across `h_1 ... h_T` (max, mean, 或 last).

说明：Plain RNNs suffer vanishing gradients. The **LSTM** adds gates 这 decide what to forget, what to store, 与 what to 输出, stabilizing gradients through 长 sequences. The **GRU** simplifies LSTM to two gates; performs similarly 使用 fewer parameters.

说明：**Bidirectional RNNs** run one RNN forward 与 another backward, concatenating hidden states. Every 词元's representation sees both left 与 right context. Essential 面向 tagging tasks.

```figure
rnn-unroll
```

## 动手构建

### Step 1: TextCNN in PyTorch

```python
import torch
import torch.nn as nn
import torch.nn.functional as F


class TextCNN(nn.Module):
    def __init__(self, vocab_size, embed_dim, n_classes, filter_widths=(2, 3, 4), n_filters=64, dropout=0.3):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.convs = nn.ModuleList([
            nn.Conv1d(embed_dim, n_filters, kernel_size=k)
            for k in filter_widths
        ])
        self.dropout = nn.Dropout(dropout)
        self.fc = nn.Linear(n_filters * len(filter_widths), n_classes)

    def forward(self, token_ids):
        x = self.embed(token_ids).transpose(1, 2)
        pooled = []
        for conv in self.convs:
            c = F.relu(conv(x))
            p = F.max_pool1d(c, c.size(2)).squeeze(2)
            pooled.append(p)
        h = torch.cat(pooled, dim=1)
        return self.fc(self.dropout(h))
```

这个 `transpose(1, 2)` reshapes `[batch, seq_len, embed_dim]` to `[batch, embed_dim, seq_len]` 因为 `nn.Conv1d` treats the middle axis as channels. The pooled 输出 is 固定-size regardless of input length.

### Step 2: LSTM classifier

```python
class LSTMClassifier(nn.Module):
    def __init__(self, vocab_size, embed_dim, hidden_dim, n_classes, bidirectional=True, dropout=0.3):
        super().__init__()
        self.embed = nn.Embedding(vocab_size, embed_dim, padding_idx=0)
        self.lstm = nn.LSTM(embed_dim, hidden_dim, batch_first=True, bidirectional=bidirectional)
        factor = 2 if bidirectional else 1
        self.dropout = nn.Dropout(dropout)
        self.fc = nn.Linear(hidden_dim * factor, n_classes)

    def forward(self, token_ids):
        x = self.embed(token_ids)
        out, _ = self.lstm(x)
        pooled = out.max(dim=1).values
        return self.fc(self.dropout(pooled))
```

Max-pool over the 序列, not last-状态 pool. 面向 分类, max-pooling usually beats taking the last hidden 状态 因为 information at the end of a 长 序列 tends to dominate the last 状态.

### 说明：Step 3: the vanishing gradient demo (intuition)

一个 plain RNN 不使用 gating cannot learn 长-range dependencies. Consider a toy 任务: predict whether 词元 `A` appeared anywhere in a 序列. If `A` is at position 1 与 the 序列 is 100 词元 长, the gradient 从 the loss has to flow back through 99 multiplications of the recurrent weight. If the weight is less than 1, the gradient vanishes. If more than 1, it explodes.

```python
def vanishing_gradient_sim(seq_len, recurrent_weight=0.9):
    import math
    return math.pow(recurrent_weight, seq_len)


# At weight=0.9 over 100 steps:
#   0.9 ^ 100 ≈ 2.7e-5
# The gradient from step 100 to step 1 is effectively zero.
```

LSTMs fix this 使用 a **cell 状态** 这 runs through the network 使用 only additive interactions (the forget gate scales it multiplicatively, but gradients still flow along the "highway"). GRUs do something similar 使用 fewer parameters. Both give you stable 训练 through 100+ step sequences.

### Step 4: why this still was not enough

Three problems persisted even 使用 LSTMs.

1. 说明：**Sequential bottleneck.** 训练 an RNN on a 序列 of length 1000 requires 1000 serial forward/backward steps. Cannot parallelize across time.
2. 说明：**固定-size context 向量 in encoder-decoder setups.** The decoder sees only the final hidden 状态 of the encoder, compressed over the entire input. 长 inputs lose detail. Lesson 09 covers this directly.
3. 说明：**Distant-dependency 准确率 ceiling.** LSTMs outperform plain RNNs but still struggle to propagate specific information across 200+ steps.

说明：注意力 solved all three. Transformers dropped recurrence entirely. Lesson 10 is the pivot.

## 投入使用

PyTorch's `nn.LSTM`, `nn.GRU`, 与 `nn.Conv1d` are 生产-ready. 训练 code is standard.

说明：Hugging Face ships pretrained 嵌入 you plug in as the input layer:

```python
from transformers import AutoModel

encoder = AutoModel.from_pretrained("bert-base-uncased")
for param in encoder.parameters():
    param.requires_grad = False


class BertCNN(nn.Module):
    def __init__(self, n_classes, filter_widths=(2, 3, 4), n_filters=64):
        super().__init__()
        self.encoder = encoder
        self.convs = nn.ModuleList([nn.Conv1d(768, n_filters, kernel_size=k) for k in filter_widths])
        self.fc = nn.Linear(n_filters * len(filter_widths), n_classes)

    def forward(self, input_ids, attention_mask):
        with torch.no_grad():
            out = self.encoder(input_ids=input_ids, attention_mask=attention_mask).last_hidden_state
        x = out.transpose(1, 2)
        pooled = [F.max_pool1d(F.relu(conv(x)), kernel_size=conv(x).size(2)).squeeze(2) for conv in self.convs]
        return self.fc(torch.cat(pooled, dim=1))
```

Use-when-it-fits-the-constraint checklist.

- **边缘 / on-device 推理.** TextCNN 使用 GloVe 嵌入 is 10-100x smaller than a transformer. If your deploy target is a phone, this is the stack.
- **Streaming / online 分类.** RNN processes one 词元 at a time; transformers need the full 序列. 面向 real-time incoming 文本, LSTMs still win.
- 说明：**Tiny models 面向 baselines.** 快 iteration on a new 任务. Train a TextCNN in 5 minutes on a CPU.
- **序列 labeling 使用 limited 数据.** BiLSTM-CRF (lesson 06) is still a 生产-grade NER architecture 面向 1k-10k labeled sentences.

Everything else goes to a transformer.

## 交付成果

保存为 `outputs/prompt-text-encoder-picker.md`:

```markdown
---
name: text-encoder-picker
description: Pick a text encoder architecture for a given constraint set.
phase: 5
lesson: 08
---

Given constraints (task, data volume, latency budget, deploy target, compute budget), output:

1. Encoder architecture: TextCNN, BiLSTM, BiLSTM-CRF, transformer fine-tune, or "use a pretrained transformer as a frozen encoder + small head".
2. Embedding input: random init, GloVe / fastText frozen, or contextualized transformer embeddings.
3. Training recipe in 5 lines: optimizer, learning rate, batch size, epochs, regularization.
4. One monitoring signal. For RNN/CNN models: attention mechanism absence means they miss long-range deps; check per-length accuracy. For transformers: fine-tuning collapse if LR too high; check train loss.

Refuse to recommend fine-tuning a transformer when data is under ~500 labeled examples without showing that a TextCNN / BiLSTM baseline has plateaued. Flag edge deployment as needing architecture-before-everything.
```

## 练习

1. 说明：**Easy.** Train a TextCNN on a 3-class toy dataset (you invent the 数据). Verify 这 filter widths (2, 3, 4) outperform a single width (3) on average F1.
2. **Medium.** Implement max-pool, mean-pool, 与 last-状态 pooling 面向 the LSTM classifier. Compare on a 小 dataset; 文档 这 pooling wins 与 hypothesize why.
3. **Hard.** Build a BiLSTM-CRF NER tagger (combine lesson 06 与 this one). Train on CoNLL-2003. Compare to the CRF-alone 基线 从 lesson 06 与 to a BERT fine-tune. Report 训练 time, memory, 与 F1.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|TextCNN|CNN 面向 文本|说明：Stack of 1D convolutions over 词 嵌入 使用 global max-pool. Kim (2014).|
|RNN|Recurrent net|Hidden 状态 updated at each time step: `h_t = f(W x_t + U h_{t-1})`.|
|LSTM|Gated RNN|说明：Adds input / forget / 输出 gates + a cell 状态. Trains stably through 长 sequences.|
|GRU|Simpler LSTM|说明：Two gates instead of three. Similar 准确率, fewer parameters.|
|Bidirectional|两者都 directions|说明：Forward + backward RNN concatenated. Every 词元 sees both sides of its context.|
|Vanishing gradient|训练 signal dies|说明：Repeated multiplication by <1 weights in plain RNNs makes early-step gradients effectively zero.|

## 延伸阅读

- 说明：[说明：Kim, Y. (2014). Convolutional Neural Networks 面向 句子 Classification](https://arxiv.org/abs/1408.5882)，the TextCNN paper. Eight pages. Readable.
- 说明：[Hochreiter, S. 与 Schmidhuber, J. (1997). 长 短-Term Memory](https://www.bioinf.jku.at/publications/older/2604.pdf)，the LSTM paper. Unexpectedly lucid.
- 说明：[Olah, C. (2015). Understanding LSTM Networks](https://colah.github.io/posts/2015-08-Understanding-LSTMs/)，the diagrams 这 made LSTMs accessible to everyone.
