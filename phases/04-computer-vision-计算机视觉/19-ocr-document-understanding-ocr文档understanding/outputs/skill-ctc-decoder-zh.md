---
name: skill-ctc-decoder-zh
description: 编写 greedy 和 beam-search CTC 解码器s 从零实现, including length n或malisation
version: 1.0.0
phase: 4
lesson: 19
tags: [ocr, ctc, decoding, sequence-models]
---

# CTC 解码器

Produce two decoding routines f或 CTC outputs: greedy (fast) 和 beam (better on noisy inputs).

## When 到 use

- 运行ning OCR 推理 on cus到m CRNN outputs.
- Benchmarking 一个pretrained OCR 模型 against different 解码器s.
- 实现ing 一个simple beam search 带有out pulling in ctcdecode.

## 输入

- `log_probs`: (T, N, C) log-s的tmax over vocab (index 0 = blank by convention).
- `vocab`: list 的 C characters.
- `beam_width` (beam only): typically 5-10.

## Greedy 解码器

```python
def greedy_ctc_decode(log_probs, vocab, blank=0):
    preds = log_probs.argmax(dim=-1).transpose(0, 1).cpu().tolist()
    out = []
    for seq in preds:
        decoded = []
        prev = None
        for idx in seq:
            if idx != prev and idx != blank:
                decoded.append(vocab[idx])
            prev = idx
        out.append("".join(decoded))
    return out
```

## Beam search 解码器

```python
import heapq
import math

def beam_ctc_decode(log_probs, vocab, beam_width=5, blank=0):
    T, N, C = log_probs.shape
    lp = log_probs.cpu()
    results = []
    for n in range(N):
        beams = {("",): (0.0, -math.inf)}  # (prefix_tuple) -> (p_blank, p_nonblank)
        for t in range(T):
            logits_t = lp[t, n]
            new_beams = {}
            for prefix, (p_b, p_nb) in beams.items():
                for c in range(C):
                    p = logits_t[c].item()
                    if c == blank:
                        nb = p_b + p
                        nnb = p_nb + p
                        upd = new_beams.get(prefix, (-math.inf, -math.inf))
                        new_beams[prefix] = (
                            _logsumexp(upd[0], _logsumexp(nb, nnb)),
                            upd[1],
                        )
                    else:
                        last = prefix[-1] if prefix else ""
                        char = vocab[c]
                        if char == last:
                            # Case 1: stay on same prefix (collapse from p_nb)
                            upd = new_beams.get(prefix, (-math.inf, -math.inf))
                            new_beams[prefix] = (upd[0], _logsumexp(upd[1], p_nb + p))
                            # Case 2: extend prefix via blank-separated repeat ("a_a" -> "aa")
                            new_prefix = prefix + (char,)
                            upd = new_beams.get(new_prefix, (-math.inf, -math.inf))
                            new_beams[new_prefix] = (upd[0], _logsumexp(upd[1], p_b + p))
                        else:
                            new_prefix = prefix + (char,)
                            upd = new_beams.get(new_prefix, (-math.inf, -math.inf))
                            nb = _logsumexp(p_b, p_nb) + p
                            new_beams[new_prefix] = (upd[0], _logsumexp(upd[1], nb))
            beams = dict(heapq.nlargest(
                beam_width,
                new_beams.items(),
                key=lambda kv: _logsumexp(kv[1][0], kv[1][1]),
            ))
        best = max(beams.items(), key=lambda kv: _logsumexp(kv[1][0], kv[1][1]))[0]
        results.append("".join(best))
    return results


def _logsumexp(a, b):
    if a == -math.inf: return b
    if b == -math.inf: return a
    m = max(a, b)
    return m + math.log(math.exp(a - m) + math.exp(b - m))
```

## 规则

- blank index in CTC 是0 by convention in PyT或ch's `nn.CTCLoss`.
- Beam search improves 准确率 on low-confidence inputs; on cle一个inputs improvement 是<1% CER.
- 不要 prune beam below 5; 准确率-延迟 trade flattens below that.
- When running beam search inside 一个tight 延迟 budget, drop 到 greedy; quality hit 是small on most 生产 OCR data.
- F或 large vocabularies (CJK 带有 3000+ characters), switch 到 `ctcdecode` (C++) instead 的 pure Python version above; Python beam quickly becomes bottleneck.
