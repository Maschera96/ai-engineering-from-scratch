# vLLM 服务内部机制：PagedAttention、连续批处理、分块预填充

> vLLM's dominance in 2026 rests on three compounding defaults, not a single trick. PagedAttention is always on. 连续 batching injects new 请求 into the active 批处理 between decode iterations. Chunked prefill slices long 提示词 so decode 词元 never starve. Turn all three on and a Llama 3.3 70B FP8 on one H100 SXM5 pushes 2,200-2,400 tok/s at 128 concurrent ： roughly 25% above vLLM's own 默认 and 3-4x a naive PyTorch loop. This lesson reads 这个调度器 and attention kernel at a level you can diagram, and ends with a toy 连续 batcher in `code/main.py` that schedules prefill and decode the way vLLM does.

**Type:** Learn
**Languages:** Python（标准库， 玩具 continuous batching scheduler)
**Prerequisites:** Phase 17 · 01 (Model Serving), Phase 11 (大模型工程)
**Time:** ~75 分钟

## 学习目标

- 解释 PagedAttention as a KV 缓存 allocator: blocks, block tables, and why fragmentation stays under 4% at 生产化 load.
- Diagram 连续 batching at the iteration level: how finished sequences leave 这个批处理 and new ones join without draining.
- Describe chunked prefill in one sentence and name which 延迟 指标 it protects (hint: it is TTFT 尾部, not 均值 吞吐量).
- Name the 2026 vLLM v0.18.0 gotcha that bites 团队 enabling every optimization at once.

## 问题

A naive PyTorch serve loop runs one 请求 at a time: tokenize, prefill, decode until EOS, return. At one user this works. At one hundred, it is 一个队列 of patient people. The obvious fix ： static batching ： pads every 请求 to the longest 提示词 in the window, pads every decode to the longest expected 输出, and stalls the whole 批处理 on the slowest sequence. You pay for padding you never use, and fast 请求 wait for slow ones.

vLLM solves three problems at once. PagedAttention stops KV 缓存 fragmentation from eating 60-80% of GPU 内存 the way classic contiguous allocation does. 连续 batching lets 请求 join and leave 这个批处理 between each decode iteration, so 这个批处理 is always full of real work. Chunked prefill breaks a 32k-词元 提示词 into ~512-词元 slices that interleave with decode, so a long 提示词 does not freeze every decode 词元 on the GPU.

The 2026 生产化 默认 is all three on. You need to understand what each one does because the failure modes are all on 这个调度器, not 这个模型.

## 概念

### 作为虚拟内存系统的 PagedAttention

A KV 缓存 is `num_layers × 2 × num_heads × head_dim × seq_len × bytes_per_element` per sequence. For Llama 3.3 70B at 8192 词元, that is roughly 1.25 GB per sequence in BF16. If you pre-reserve 8192 slots for every 请求 but the average 请求 only uses 1500 词元, you waste roughly 82% of the HBM you 预留. Classic batching pays this waste.

PagedAttention borrows the idea from OS virtual 内存. KV 缓存 is not contiguous per sequence. It is allocated in fixed-size blocks (默认 16 词元). Each sequence has a block table that maps its logical 词元 positions to physical block IDs. When a sequence grows past its allocated blocks, one more block is 新增. When it finishes, its blocks return to the pool.

Fragmentation drops from 60-80% (classic) to under 4% (PagedAttention). You do not enable PagedAttention with a flag ： it is the only allocator vLLM ships. The knob is `--gpu-memory-utilization` (默认 0.9), which tells vLLM how much HBM to reserve for KV blocks after loading weights and activations.

### 迭代级连续批处理

The old "dynamic batching" waited for a window (say 10 ms) to fill 一个批处理, then ran prefill + decode + decode + decode until every sequence finished. Fast sequences left early and sat 空闲 while the GPU finished the slow ones.

连续 batching operates between each decode 步骤. Call the set of running sequences 这个`RUNNING` list. At each iteration:

1. Any sequence in `RUNNING` that just hit EOS or max_tokens is removed.
2. 这个调度器 looks at the waiting 队列. If there are free KV blocks, it admits new sequences (prefill or resumed).
3. The forward pass runs on whatever is now in `RUNNING`, emitting one new 词元 per sequence.

这个批处理 size is never padded to a fixed number. Sequences at different positions in their 输出 share one fused forward. In 2026 vLLM this is called 这个`V1 scheduler`. The key invariant: 这个调度器 runs once per decode iteration, not once per 请求.

### 分块预填充保护 TTFT 尾部

Prefill is compute-bound. A 32k-词元 提示词 on Llama 3.3 70B takes ~800 ms of pure prefill on one H100. While prefill runs, decode 词元 for every other sequence in 这个批处理 wait. In a serving loop, 这个首词元延迟 (TTFT) of one long 提示词 becomes the inter-词元 延迟 (ITL) blip for dozens of other users.

Chunked prefill splits prefill into fixed-size chunks (默认 512 词元) and schedules each chunk as a unit. Between chunks 这个调度器 can advance decode sequences by one 词元. You trade a small absolute prefill 延迟 hit (a few ms per chunk) for much lower decode-time jitter. P99 ITL under mixed load drops from ~50 ms to ~15 ms in published 基准.

### 三个默认项会互相影响

All three 功能 assume each other. PagedAttention gives 这个调度器 a fine-grained KV resource to trade against. 连续 batching needs that fine-grained resource so admitting a new sequence does not force a global reshuffle. Chunked prefill is 一个决策 这个调度器 makes on the same `RUNNING` list ： it is one more 调度器 策略, not a separate system.

You do not need to know every flag. You need to know what 这个调度器 optimizes: goodput under KV-block budget, subject to chunked prefill slicing.

### The 2026 v0.18.0 gotcha

In vLLM v0.18.0 you cannot combine `--enable-chunked-prefill` with draft-模型 speculative decoding (`--speculative-model`). The documented exception is N-gram GPU speculative decoding in the V1 调度器. 团队 that flip every flag on without reading the release notes get a run-time error at startup, not a soft regression. If your speculative gain was worth enabling chunked prefill for, revisit the choice ： the right answer in 2026 is often EAGLE-3 without chunked prefill, not a draft 模型 plus chunked prefill that does not compile.

### 你应该记住的数字

- Llama 3.3 70B FP8, H100 SXM5, 128 concurrent, all three on: 2,200-2,400 tok/s.
- Same 模型, 默认 vLLM (no chunked prefill): ~1,800 tok/s.
- Same 模型, naive PyTorch forward loop: ~600 tok/s.
- KV fragmentation waste under PagedAttention at 生产化 load: <4%.
- P99 ITL under mixed load: ~15 ms with chunked prefill, ~50 ms without.

### 调度器长什么样

```
while True:
    finished = [s for s in RUNNING if s.is_done()]
    for s in finished: release_blocks(s); RUNNING.remove(s)

    while WAITING and have_free_blocks_for(WAITING[0]):
        s = WAITING.pop(0)
        allocate_initial_blocks(s)
        RUNNING.append(s)

    # schedule prefill chunks + decode in one batch
    batch = []
    for s in RUNNING:
        if s.in_prefill:
            batch.append(next_prefill_chunk(s))   # e.g. 512 tokens
        else:
            batch.append(decode_one_token(s))     # 1 token

    run_forward(batch)                            # one fused GPU call
```

`code/main.py` is exactly this loop in stdlib Python with fake 词元 counts and fake forward 延迟. Running it shows how chunked prefill keeps decode sequences alive during a long prefill.

```figure
tensor-parallel
```

## 使用它

`code/main.py` simulates a vLLM-style 调度器 with toggleable 功能. Run it to see:

- `NAIVE` mode: one 请求 at a time, no batching.
- `STATIC` mode: pad and wait, classic batching.
- `CONTINUOUS` mode: iteration-level admission and release.
- `CONTINUOUS + CHUNKED` mode: prefill slices interleaved with decode.

这个输出 shows total 吞吐量 (词元 per virtual second), TTFT 均值, and P99 ITL. 这个`CONTINUOUS + CHUNKED` row should dominate on mixed 流量.

## 交付它

This lesson 产出 `outputs/skill-vllm-scheduler-reader.md`. Given a serving config (批处理 size, KV 内存 utilization, chunked prefill size, speculative config), it 产出 一个调度器 diagnosis that names which of the three defaults is bottlenecking and what to tune.

## 练习

1. Run `code/main.py`. 比较 `STATIC` to `CONTINUOUS` on 一个工作负载 with mixed short and long 请求. Where does 这个吞吐量 gap come from ： prefill efficiency, decode efficiency, 或 尾部 延迟?
2. Modify the toy 调度器 to add `--max-num-batched-tokens`. What is the right value for an H100 running Llama 3.3 70B FP8? (Hint: it is a function of KV block size and number of free blocks, not raw HBM.)
3. Re-read the vLLM v0.18.0 release notes. Which combinations of flags are mutually exclusive? List them.
4. Compute the KV 缓存 fragmentation waste for 一个追踪 of 1,000 请求 搭配 均值 1,500 输出 词元, std 600 词元, under (a) contiguous per-请求 allocation at 8192 max, (b) PagedAttention with 16-词元 blocks.
5. 解释 in one paragraph why chunked prefill helps P99 ITL but not 吞吐量 in isolation. Where does 这个吞吐量 win come from in practice?

## 关键术语

| 术语 | 人们常说 | 实际含义 |
|------|----------------|------------------------|
| PagedAttention | "the KV trick" | Fixed-size block allocator for KV 缓存; fragmentation <4% |
| Block table | "the page table" | Per-sequence map from logical 词元 position to physical KV block |
| 连续 batching | "dynamic batching, but right" | Admit/release decisions made every decode iteration |
| Chunked prefill | "prefill splitting" | Break long prefill into 512-词元 slices interleaved with decode |
| TTFT | "first 词元 time" | Prefill + 队列 + network; dominated by prefill at long 提示词 |
| ITL | "inter-词元 延迟" | Time between consecutive decode 词元; dominated by 批处理 size |
| Goodput | "吞吐量 that meets SLO" | 词元/sec where every 请求 still hit TTFT and ITL targets |
| V1 调度器 | "the new 调度器" | vLLM's 2026 调度器; N-gram spec decode is the chunked-prefill-compatible path |
| `--gpu-memory-utilization` | "这个内存 knob" | Fraction of HBM 预留 for KV blocks after weights and activations |

## 延伸阅读

- [vLLM documentation — Speculative Decoding](https://docs.vllm.ai/en/latest/features/spec_decode/) ： official source on chunked-prefill and speculative-decoding compatibility.
- [vLLM Release Notes (NVIDIA)](https://docs.nvidia.com/deeplearning/frameworks/vllm-release-notes/index.html) ： 2026 release cadence and version-具体 behavior.
- [vLLM Blog — PagedAttention](https://blog.vllm.ai/2023/06/20/vllm.html) ： the original write-up that still defines how to think about the allocator.
- [PagedAttention paper (arXiv:2309.06180)](https://arxiv.org/abs/2309.06180) ： fragmentation analysis and 调度器 design.
- [Aleksa Gordic — Inside vLLM](https://www.aleksagordic.com/blog/vllm) ： detailed V1 调度器 walkthrough with flame graphs.
