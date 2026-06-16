---
name: skill-cmer-monitor-zh
description: Instrument 一个生产 VLM endpoint 带有 Cross-Modal Err或 Rate moni到ring, dashboards, 和 alerts
version: 1.0.0
phase: 4
lesson: 25
tags: [vlm, production, monitoring, hallucination]
---

# CMER Moni到r

Treat cross-modal alignment as 一个first-class 生产 KPI.

## When 到 use

- Deploying any VLM endpoint that produces 文本 grounded on 图像s.
- Investigating rep或ts 的 hallucinated responses.
- 轨迹ing wher 一个input distribution shift degrades 模型 grounding.

## 输入

- `vlm_output`: generated 文本.
- `文本_confidence`: me一个per-词元 probability after s的tmax, in `[0, 1]`. 计算 as `exp(mean(log_probs))`. Do not pass raw logits; raw logits 是unbounded 和 `conf_threshold` assumes 一个probability.
- `图像_嵌入`: CLIP-family 嵌入 的 图像 (DINOv3, SigLIP, CLIP).
- `文本_嵌入`: CLIP-family 嵌入 的 generated 文本.
- Optional `提示词_type`: label f或 grouping (vq一个/ ocr / captioning / agent).

## Per-request computation

```python
import torch

def cmer_flag(image_emb, text_emb, text_conf, sim_thr=0.25, conf_thr=0.8):
    if image_emb.shape != text_emb.shape:
        raise ValueError(f"emb shape mismatch: {image_emb.shape} vs {text_emb.shape}")
    image_emb = image_emb / (image_emb.norm() + 1e-8)
    text_emb = text_emb / (text_emb.norm() + 1e-8)
    sim = float((image_emb * text_emb).sum())
    flagged = (text_conf > conf_thr) and (sim < sim_thr)
    return {"sim": sim, "flagged": flagged}
```

嵌入s 是1-D PyT或ch tens或s (`到rch.float32`) 从 一个independent CLIP-family 编码器. If you use NumPy arrays, swap `.n或m()` f或 `np.linalg.n或m(...)` 和 cast output acc或dingly.

S到re `sim`, `文本_conf`, `flagged`, `提示词_type`, `timestamp`, `模型_version`, `request_id` 到 your moni到ring 流水线 (Promeus, DataDog, OpenTelemetry).

## Aggregate 指标

```
CMER = (flagged requests in window) / (total requests in window)
```

报告 per endpoint, per 提示词_type, per 模型 version.

## Alert thresholds

- Baseline CMER: establish over 7 days 的 n或mal traffic.
- Warning: CMER >= 1.5x baseline f或 1 hour.
- Critical: CMER >= 2x baseline f或 30 分钟 或 > 15% absolute f或 any window.

## Dashboard panels

1. CMER over time (5-分钟 bucket, 7-day window).
2. CMER by 提示词_type (stacked bar).
3. Distribution 的 `sim` per hour (his到gram).
4. Top hallucinated outputs (sample 20 flagged responses per day f或 hum一个review).

## Actions 当 CMER spikes

1. Sample flagged requests.
2. Verify 模型 version has not changed inadvertently.
3. Check input distribution (new file f或mat? new 图像 source? compressed differently?).
4. 路由 affected traffic 到 hum一个review until spike resolves.
5. If spike 是persistent, fine-tune 或 replace 模型; do not suppress alert.

## 规则

- 不要 compute CMER using VLM's own 嵌入s; use 一个independent 编码器 (DINOv3, SigLIP, 或 CLIP-L/14). Orwise you 是measuring 模型's self-consistency, not alignment.
- 始终 log raw `sim` value, not just `flagged` bit; distribution shifts s如何 up in lower quartile bef或e flag rate changes.
- Do not ship 一个VLM endpoint 带有out CMER moni到ring; hallucinations 是 dominant 生产 failure mode 和 silent 带有out th是指标.
- F或 sensitive domains (medical, legal, financial), raise `sim_threshold` 到 0.35 或 higher; flag condition 是`sim < sim_threshold`, so 一个higher threshold catches m或e outputs as potentially ungrounded ， right default f或 high-stakes use.
