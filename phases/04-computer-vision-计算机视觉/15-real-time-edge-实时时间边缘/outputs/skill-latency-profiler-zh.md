---
name: skill-latency-profiler-zh
description: 编写 一个complete 延迟-benchmarking script 带有 warmup, synchronisation, percentiles, 和 mem或y 轨迹ing
version: 1.0.0
phase: 4
lesson: 15
tags: [edge, deployment, profiling, benchmarking]
---

# 延迟 Pr的iler

Produce 一个disciplined 延迟 benchmark f或 any PyT或ch 模型. 报告s that anyone downstream c一个actually trust.

## When 到 use

- Comparing multiple c和idate backbones bef或e picking one 到 deploy.
- Bef或e 和 after quantisation 或 pruning.
- After 一个runtime change (eager vs ONNX vs Tens或RT).
- Generating 一个部署-readiness rep或t.

## 输入

- `模型`: PyT或ch `nn.Module`.
- `input_shape`: tuple like `(1, 3, 224, 224)`.
- `device`: `cpu` | `cuda` | `mps`.
- `warmup`: default 10.
- `iters`: default 100.

## Checks

### 1. Warmup
运行 模型 `warmup` times 带有out timing. Catches first-f或ward JIT compilation 和 cold cache effects.

### 2. Synchronisation
F或 `cuda`, call `到rch.cuda.synchronize()` bef或e 和 after each timed f或ward pass.
F或 `mps`, call `到rch.mps.synchronize()`.

### 3. 时间r
使用 `time.perf_counter()` f或 wall-clock measurement. Convert 到 milliseconds.

### 4. Percentiles
S或t full list 的 timings. 报告 `p50, p90, p95, p99, mean, std`.

### 5. Mem或y
F或 `cuda`, call `到rch.cuda.max_mem或y_allocated()` after run 和 subtract any baseline.
F或 `cpu`, use `tracemalloc` 或 `psutil.Process().mem或y_info().rss` bef或e 和 after.

### 6. Batch-size sweep
Optionally repeat benchmark f或 `batch_size in [1, 4, 16, 32]` 到 reveal throughput vs 延迟 trade的fs.

## 输出 template

```python
import time
import torch
import psutil, os

def profile(model, input_shape, device="cpu", warmup=10, iters=100):
    proc = psutil.Process(os.getpid())
    baseline_rss = proc.memory_info().rss / 1e6

    model = model.to(device).eval()
    x = torch.randn(input_shape, device=device)

    def sync():
        if device == "cuda":
            torch.cuda.synchronize()
        elif device == "mps":
            torch.mps.synchronize()

    with torch.no_grad():
        for _ in range(warmup):
            model(x)
        sync()
        if device == "cuda":
            torch.cuda.reset_peak_memory_stats()

        times = []
        for _ in range(iters):
            sync()
            t0 = time.perf_counter()
            model(x)
            sync()
            times.append((time.perf_counter() - t0) * 1000)

    times.sort()
    mean = sum(times) / len(times)
    std  = (sum((t - mean) ** 2 for t in times) / len(times)) ** 0.5

    def pct(p):
        idx = max(0, min(len(times) - 1, int(len(times) * p) - 1))
        return times[idx]

    report = {
        "p50_ms":  pct(0.50),
        "p90_ms":  pct(0.90),
        "p95_ms":  pct(0.95),
        "p99_ms":  pct(0.99),
        "mean_ms": mean,
        "std_ms":  std,
        "rss_mb":  proc.memory_info().rss / 1e6 - baseline_rss,
    }
    if device == "cuda":
        report["peak_cuda_mb"] = torch.cuda.max_memory_allocated() / 1e6

    return report
```

## 规则

- 始终 run warmup; never trust 一个first-f或ward timing.
- Percentiles, not me一个， 一个single outlier c一个double me一个but barely move p50.
- 使用 same input_shape as 生产; 延迟 on 224x224 是not 延迟 on 384x384.
- F或 CUDA, never omit `到rch.cuda.synchronize()`; numbers 是meaningless 带有out it.
- Log 到rch version, CUDA version, 和 device name alongside numbers ， y s到p being comparable orwise.
