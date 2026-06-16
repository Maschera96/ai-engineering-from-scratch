---
name: vllm-scheduler-reader-zh
description: Diagnose a vLLM serving config by reading 这个调度器-level knobs and identifying which of PagedAttention, 连续 batching, and chunked prefill is the bottleneck.
version: 1.0.0
phase: 17
lesson: 04
tags: [vllm, paged-attention, continuous-batching, chunked-prefill, serving, scheduler]
---

给定a vLLM serving config (模型, dtype, hardware, `--gpu-memory-utilization`, `--max-num-batched-tokens`, `--enable-chunked-prefill`, `--speculative-model` 或 `--speculative-config`, max concurrency, and an observed 指标 set of TTFT 均值/P99, ITL 均值/P99, 吞吐量 tok/s), produce 一个调度器-level diagnosis.

产出：

1. Config read. For each flag, name 这个调度器 behavior it 控制 and the 2026 默认. Flag any flag set to a non-默认 value and call out why.
2. Bottleneck identification. Classify the bottleneck as one of: PagedAttention under-provisioned (KV block starvation), 连续-batching stall (WAITING 队列 growth), chunked-prefill mis-sized (TTFT 尾部 spike), decode compute-bound (ITL floor), or HBM-bound (cannot fit 批处理). 说明理由 with the reported 指标.
3. Knob recommendations. 具体, ordered actions ： which flag to flip, which value to try, and which 指标 to watch. Do not suggest "try more GPUs" without first exhausting 调度器-level tuning.
4. Compatibility check. For vLLM v0.18.0 specifically: flag 这个`--enable-chunked-prefill` + `--speculative-model` combination as a hard incompatibility. Recommend N-gram GPU speculative decoding in V1 as the documented exception if both are desired.
5. What to read next. Point to one of the vLLM v0.18.0 release notes, the PagedAttention paper, or the Aleksa Gordic V1 调度器 walkthrough depending on what the diagnosis surfaced.

硬性拒绝：
- Diagnosing without the four core 指标 (TTFT, ITL, 吞吐量, concurrency). 拒绝 and ask for 这个指标 set.
- Recommending `--enable-chunked-prefill` without checking the speculative-decoding config.
- Treating `DCGM_FI_DEV_GPU_UTIL` as a scaling signal. vLLM pre-allocates KV; duty-cycle numbers are misleading.

拒绝规则：
- If the reported 吞吐量 is under 100 tok/s on an H100, the bottleneck is likely not vLLM ： check for tokenizer on client side, Python GIL, 或 请求-level serialization.
- If `--gpu-memory-utilization` is set below 0.7, 拒绝 to tune further ： the operator chose to leave HBM on the table and the fix is to raise the ceiling before flipping 调度器 flags.
- If the operator asks for a speculative-decoding + chunked-prefill recipe on draft-模型 speculation, 拒绝 and name the v0.18.0 incompatibility. Point to EAGLE-3 in Phase 17 · 05 instead.

输出： a one-page 调度器 diagnosis listing flags, bottleneck, ordered recommendations, compatibility notes, and a next-read pointer. End with 一个"what to measure next" paragraph naming one of P99 ITL, block allocation rate, or WAITING 队列 depth, depending on the bottleneck identified.
