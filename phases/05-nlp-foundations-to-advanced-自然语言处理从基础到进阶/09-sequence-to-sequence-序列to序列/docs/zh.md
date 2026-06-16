# 序列到序列模型

> 说明：Two RNNs pretending to be a translator. The bottleneck they hit is the 原因 注意力 exists.

**类型:** 构建
**语言:** Python
**先修要求:** Phase 5 · 08 (CNNs + RNNs 面向 文本), Phase 3 · 11 (PyTorch Intro)
**时间:** 约75分钟

## 问题

Classification maps a variable-length 序列 to a single 标签. Translation maps a variable-length 序列 to another variable-length 序列. The input 与 输出 live in different vocabularies, possibly different languages, 使用 no guarantee of length parity.

这个 seq2seq architecture (Sutskever, Vinyals, Le, 2014) cracked this 使用 a deliberately simple recipe. Two RNNs. One reads the source 句子 与 produces a 固定-size context 向量. The other reads 这 向量 与 generates the target 句子 词元 by 词元. Same code you wrote 面向 lesson 08, glued together differently.

This is worth studying 面向 two reasons. First, the context-向量 bottleneck is the most pedagogically useful failure in NLP. It motivates everything 注意力 与 transformers are good at. Second, the 训练 recipe (teacher forcing, scheduled sampling, beam search at 推理) still applies to every 现代 生成 系统 including LLMs.

## 概念

**Encoder.** An RNN 这 reads the source 句子. Its final hidden 状态 is the **context 向量**，a 固定-size 摘要 of the entire input. Lose nothing but the source, supposedly.

**Decoder.** Another RNN initialized 从 the context 向量. At each step it takes the previously generated 词元 as input 与 produces a 分布 over the target vocabulary. Sample 或 argmax to pick the next 词元. Feed it back in. Repeat until an `<EOS>` 词元 is produced 或 max length is hit.

**训练:** 说明：Cross-entropy loss at each decoder step, summed over the 序列. Standard backprop through time through both networks.

**Teacher forcing.** During 训练, the decoder's input at step `t` is the *ground-truth* 词元 at position `t-1`, not the decoder's own previous prediction. This stabilizes 训练; 不使用 it, early mistakes cascade 与 the 模型 never learns. At 推理, you have to use the 模型's own predictions, so there is always a train/推理 分布 gap. 这 gap is called **exposure bias**.

说明：**The bottleneck.** Everything the encoder learned about the source must be squeezed 到 这 one context 向量. 长 sentences lose detail. Rare 词 get blurred. Reordering (chat noir vs. black cat) has to be memorized, not computed.

说明：注意力 (lesson 10) fixes this by letting the decoder look at *every* encoder hidden 状态, not just the last one. 这 is the whole pitch.

```figure
lstm-gates
```

## 动手构建

### Step 1: an encoder

```python
import torch
import torch.nn as nn


class Encoder(nn.Module):
    def __init__(self, src_vocab_size, embed_dim, hidden_dim):
        super().__init__()
        self.embed = nn.Embedding(src_vocab_size, embed_dim, padding_idx=0)
        self.gru = nn.GRU(embed_dim, hidden_dim, batch_first=True)

    def forward(self, src):
        e = self.embed(src)
        outputs, hidden = self.gru(e)
        return outputs, hidden
```

`outputs` has shape `[batch, seq_len, hidden_dim]`，one hidden 状态 per input position. `hidden` has shape `[1, batch, hidden_dim]`，the final step. Lesson 08 said "pool over outputs 面向 分类." Here we keep the last hidden 状态 as the context 向量, 与 ignore the per-step outputs.

### Step 2: a decoder

```python
class Decoder(nn.Module):
    def __init__(self, tgt_vocab_size, embed_dim, hidden_dim):
        super().__init__()
        self.embed = nn.Embedding(tgt_vocab_size, embed_dim, padding_idx=0)
        self.gru = nn.GRU(embed_dim, hidden_dim, batch_first=True)
        self.fc = nn.Linear(hidden_dim, tgt_vocab_size)

    def forward(self, token, hidden):
        e = self.embed(token)
        out, hidden = self.gru(e, hidden)
        logits = self.fc(out)
        return logits, hidden
```

Decoder is called one step at a time. Input: a batch of single 词元 与 the 当前 hidden 状态. 输出: vocabulary logits 面向 the next 词元 与 the updated hidden 状态.

### Step 3: 训练 loop 使用 teacher forcing

```python
def train_batch(encoder, decoder, src, tgt, bos_id, optimizer, teacher_forcing_ratio=0.9):
    optimizer.zero_grad()
    _, hidden = encoder(src)
    batch_size, tgt_len = tgt.shape
    input_token = torch.full((batch_size, 1), bos_id, dtype=torch.long)
    loss = 0.0
    loss_fn = nn.CrossEntropyLoss(ignore_index=0)

    for t in range(tgt_len):
        logits, hidden = decoder(input_token, hidden)
        step_loss = loss_fn(logits.squeeze(1), tgt[:, t])
        loss += step_loss
        use_teacher = torch.rand(1).item() < teacher_forcing_ratio
        if use_teacher:
            input_token = tgt[:, t].unsqueeze(1)
        else:
            input_token = logits.argmax(dim=-1)

    loss.backward()
    optimizer.step()
    return loss.item() / tgt_len
```

Two knobs worth naming. `ignore_index=0` skips loss on padding 词元. `teacher_forcing_ratio` is the 概率 of using the true 词元 vs. the 模型's prediction at each step. Start at 1.0 (full teacher forcing) 与 anneal down to ~0.5 over 训练 to close the exposure-bias gap.

### Step 4: 推理 loop (greedy)

```python
@torch.no_grad()
def greedy_decode(encoder, decoder, src, bos_id, eos_id, max_len=50):
    _, hidden = encoder(src)
    batch_size = src.shape[0]
    input_token = torch.full((batch_size, 1), bos_id, dtype=torch.long)
    output_ids = []
    for _ in range(max_len):
        logits, hidden = decoder(input_token, hidden)
        next_token = logits.argmax(dim=-1)
        output_ids.append(next_token)
        input_token = next_token
        if (next_token == eos_id).all():
            break
    return torch.cat(output_ids, dim=1)
```

说明：Greedy decoding picks the highest-概率 词元 at every step. It can wander off: once you commit to a 词元, you cannot unsay it. **Beam search** keeps the top-`k` partial sequences alive 与 picks the highest-scoring complete one at the end. Beam width 3-5 is standard.

### Step 5: the bottleneck, demonstrated

Train the 模型 on a toy copy 任务: source `[a, b, c, d, e]`, target `[a, b, c, d, e]`. Increase 序列 length. Observe 准确率.

```
seq_len=5   copy accuracy: 98%
seq_len=10  copy accuracy: 91%
seq_len=20  copy accuracy: 62%
seq_len=40  copy accuracy: 23%
```

一个 single GRU hidden 状态 cannot losslessly memorize a 40-词元 input. The information is there at every encoder step, but the decoder only sees the last 状态. 注意力 fixes this directly.

## 投入使用

说明：PyTorch has `nn.Transformer` 与 `nn.LSTM`-based seq2seq templates. Hugging Face's `transformers` 库 ships full encoder-decoder models (BART, T5, mBART, NLLB) trained on billions of 词元.

```python
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM

tok = AutoTokenizer.from_pretrained("facebook/bart-base")
model = AutoModelForSeq2SeqLM.from_pretrained("facebook/bart-base")

src = tok("Translate this to French: Hello, how are you?", return_tensors="pt")
out = model.generate(**src, max_new_tokens=50, num_beams=4)
print(tok.decode(out[0], skip_special_tokens=True))
```

现代 encoder-decoders dropped RNNs 面向 transformers. The high-level shape (encoder, decoder, generate-词元-by-词元) is identical to the 2014 seq2seq paper. The mechanism inside each block is different.

### 当 to still reach 面向 RNN-based seq2seq

说明：Almost never, 面向 new projects. Specific exceptions:

- Streaming 翻译 其中 you consume input one 词元 at a time 使用 bounded memory.
- On-device 文本 生成 其中 transformer memory 成本 is prohibitive.
- 说明：Pedagogy. Understanding the encoder-decoder bottleneck is the fastest path to understanding why transformers won.

### Exposure bias 与 its mitigations

- 说明：**Scheduled sampling.** Anneal teacher forcing ratio during 训练 so the 模型 learns to recover 从 its own mistakes.
- 说明：**Minimum risk 训练.** Train on 句子-level BLEU score instead of 词元-level cross-entropy. Closer to what you actually want.
- **Reinforcement learning fine-tuning.** Reward the 序列 generator 使用 a 指标. Used in 现代 LLM RLHF.

说明：All three still apply to transformer-based 生成.

## 交付成果

保存为 `outputs/prompt-seq2seq-design.md`:

```markdown
---
name: seq2seq-design
description: Design a sequence-to-sequence pipeline for a given task.
phase: 5
lesson: 09
---

Given a task (translation, summarization, paraphrase, question rewrite), output:

1. Architecture. Pretrained transformer encoder-decoder (BART, T5, mBART, NLLB) is the default. RNN-based seq2seq only for specific constraints.
2. Starting checkpoint. Name it (`facebook/bart-base`, `google/flan-t5-base`, `facebook/nllb-200-distilled-600M`). Match the checkpoint to task and language coverage.
3. Decoding strategy. Greedy for deterministic output, beam search (width 4-5) for quality, sampling with temperature for diversity. One sentence justification.
4. One failure mode to verify before shipping. Exposure bias manifests as generation drift on longer outputs; sample 20 outputs at the 90th-percentile length and eyeball.

Refuse to recommend training a seq2seq from scratch for under a million parallel examples. Flag any pipeline that uses greedy decoding for user-facing content as fragile (greedy repeats and loops).
```

## 练习

1. **Easy.** Implement the toy copy 任务. Train a GRU seq2seq on input-输出 pairs 其中 the target equals the source. Measure 准确率 at lengths 5, 10, 20. Reproduce the bottleneck.
2. **Medium.** Add beam search decoding 使用 beam width 3. Measure BLEU on a 小 parallel 语料库 against greedy. 文档 其中 beam search wins (usually last 词元) 与 其中 it makes no difference.
3. 说明：**Hard.** Fine-tune `facebook/bart-base` on a 10k-pair paraphrase dataset. Compare the fine-tuned 模型's beam-4 输出 to the base 模型's on held-out inputs. Report BLEU 与 pick 10 qualitative examples.

## 关键术语

|Term|What people say|What it actually means|
|------|-----------------|-----------------------|
|Encoder|Input RNN|说明：Reads source. Produces per-step hidden states 与 a final context 向量.|
|Decoder|输出 RNN|说明：Initialized 从 context 向量. Generates target 词元 one at a time.|
|Context 向量|这个 摘要|说明：Final encoder hidden 状态. 固定 size. The bottleneck 注意力 solves.|
|Teacher forcing|Use true 词元|说明：Feed the ground-truth previous 词元 at 训练 time. Stabilizes learning.|
|Exposure bias|Train/test gap|说明：模型 trained on true 词元 never practiced recovering 从 its own mistakes.|
|Beam search|Better decoding|说明：Keep top-k partial sequences alive at each step instead of committing greedily.|

## 延伸阅读

- [说明：Sutskever, Vinyals, Le (2014). 序列 to 序列 Learning 使用 Neural Networks](https://arxiv.org/abs/1409.3215)，the original seq2seq paper. Four pages.
- 说明：[说明：Cho et al. (2014). Learning Phrase Representations using RNN Encoder-Decoder 面向 Statistical Machine Translation](https://arxiv.org/abs/1406.1078)，introduced the GRU 与 the encoder-decoder framing.
- 说明：[说明：Bahdanau, Cho, Bengio (2014). Neural Machine Translation by Jointly Learning to Align 与 Translate](https://arxiv.org/abs/1409.0473)，the 注意力 paper. Read immediately after this lesson.
- 说明：[PyTorch NLP 从 Scratch tutorial](https://pytorch.org/tutorials/intermediate/seq2seq_translation_tutorial.html)，buildable seq2seq + 注意力 code.
