# DualPipe Parallelism

> DeepSeek-V3 was 训练后的 on 2,048 H800 GPUs with MoE experts scattered across nodes. Cross-node expert all-to-all communication 成本 1 GPU-hour of comm for every 1 GPU-hour of 计算. GPUs were idle half the time. DualPipe (DeepSeek, Dec 2024) is a bidirectional 流水线 that overlaps forward and backward computation with the all-to-all comms they trigger. Bubbles drop, throughput climbs, and the keeping of two model-parameter copies (the "dual" that gives the name) is cheap once Expert Parallelism is already spreading experts across ranks anyway. This lesson is a Learn-type walkthrough of what DualPipe actually does and why Sea AI Lab's DualPipeV refinement drops the 2x 参数 成本 at the expense of a marginally tighter bubble.

**类型：** Learn
**语言：** Python (stdlib, schedule simulator)
**先修：** Phase 10 · 05 (分布式 训练, FSDP, DeepSpeed), Phase 10 · 14 (open-model architectures and MoE)
**时间：** 约 60 分钟

## 学习目标

- Name the four components of a DualPipe forward-backward 分块 and why each one gets its own overlap window.
- 解释the 流水线 bubble problem at 规模, and what "bubble-free" means in practice versus in marketing.
- Trace a DualPipe 调度 by hand for 8 PP ranks and 16 micro-batches and confirm the forward and reverse streams fill each other's idle slots.
- 状态 the tradeoff DualPipeV (Sea AI Lab, 2025) makes: drops the 2x 参数 replication at the 成本 of a slightly larger bubble when Expert Parallelism is inactive.

## 问题

训练 a 671B MoE 模型 on 2k H800 GPUs runs into three compounding bottlenecks:

1. **内存 pressure.** Each GPU holds a slice of the 模型. 激活 内存 at 序列 8k across 61 层 on 128 头 is enormous.
2. **流水线 bubbles.** Traditional 流水线 parallelism (GPipe, 1F1B) leaves GPUs idle while they wait for their stage's 输入 or 梯度. At 8 stages, roughly 12% of GPU time can be bubble even with 1F1B scheduling.
3. **Cross-node all-to-all.** MoE with expert parallelism scatters experts across nodes. Every forward pass triggers an all-to-all to dispatch 词元 to their experts, and another to combine. At 2k GPUs this easily becomes a 1:1 compute-to-comm 比例.

Each of these has separate solutions: 梯度 checkpointing for 内存, Zero Bubble (Sea AI Lab, 2023) for 流水线 bubbles, expert-parallel comm kernels for all-to-all. What DualPipe does is make them play together. The 调度 overlaps 计算 and comm within a single forward-backward 分块, injects micro-batches from both ends of the 流水线 simultaneously, and uses the resulting 调度 to hide all-to-all inside the 计算 windows.

Reported result: near-elimination of 流水线 bubbles, over 95% GPU utilization in DeepSeek-V3's 14.8T-词元 训练 run.

## 概念

### 流水线 parallelism refresher

Split an N-layer 模型 across P devices. Device `i` holds 层 `i * N/P .. (i+1) * N/P - 1`. A micro-batch flows forward through devices 0 to P-1, then backward from P-1 to 0. Each device can only start its forward stage when the 先验 device sends its 输出 and can only start backward when the downstream device sends the upstream 梯度.

GPipe (Huang et al., 2019) schedules one micro-batch at a time, which wastes most GPU time. 1F1B (Narayanan et al., 2021) interleaves forward and backward passes for multiple micro-batches. Zero Bubble (Qi et al., 2023) splits the backward pass into two parts — backward-for-input (B) and backward-for-weights (W) — and schedules them to fill the bubble. After Zero Bubble, the 流水线 is almost tight.

DualPipe is the next 步骤. It adds two ideas on top:

### Idea 1: 分块 decomposition

Each forward 分块 is split into four components:

- **注意力.** Q/K/V projections, 注意力, 输出 projection.
- **All-to-all dispatch.** Cross-node communication that sends 词元 to their experts.
- **MLP.** The MoE expert computation.
- **All-to-all combine.** Cross-node communication that brings expert outputs back.

一个backward 分块 adds 梯度 versions of each of these. DualPipe schedules them so that all-to-all dispatch happens in 并行 with the 注意力 计算 of the next 分块, and all-to-all combine happens in 并行 with the MLP 计算 of the following 分块.

### Idea 2: bidirectional scheduling

Most 流水线 schedules inject micro-batches from stage 0 and 流 toward stage P-1. DualPipe injects micro-batches from BOTH ends. Stage 0 sees forward micro-batches originating there; stage P-1 sees forward micro-batches originating there too. The two streams meet in the middle.

For this to work, device `i` must hold BOTH the early-pipeline 层 `i` AND the late-pipeline 层 `P - 1 - i`. That is the "dual" part of DualPipe: each device keeps two copies of the 模型 层 it needs to serve (one for each direction). At DeepSeek-V3's 规模, this is a 2x 参数 replication 成本. It is affordable because Expert Parallelism already spreads the MoE experts so thin that replicating the non-expert 层 twice is small potatoes.

Crucially, the forward stream in one direction and the backward stream in the other direction overlap exactly where the bubbles would be in a single-direction 调度. The bubbles vanish.

### A hand-traced 调度

Consider P = 4 ranks, 8 micro-batches, divided 4 forward / 4 reverse. 时间 moves left to right; rows are device ranks.

```text
           Time →
rank 0:  F1 F2 F3 F4  F5R F6R F7R F8R  B1 B2 B3 B4  ...
rank 1:     F1 F2 F3  F4/F5R F6R F7R   B1 B2 ...
rank 2:        F1 F2  F3/F5R F4/F6R    B1 ...
rank 3:           F1  F2/F5R F3/F6R    ...
```

Reading the "F4/F5R" notation: 排序 1 is running forward of micro-batch 4 (going left-to-right in the 流水线) AND forward of micro-batch 5 (going right-to-left) in the same time slot. That is what "bidirectional" means operationally.

At 排序 2 the cross streams overlap sooner, at 排序 0 and P-1 they overlap latest. In the stable middle phase of the 调度, every 排序 runs forward-of-X-direction overlapped with backward-of-Y-direction. 计算 is busy. All-to-all dispatches for the forward pass hide inside backward 计算. All-to-all combines hide inside forward 计算. The bubbles are squeezed out.

### Bubble accounting

Standard 1F1B 流水线 bubble (time wasted per 排序):

```text
bubble_1F1B = (P - 1) * forward_chunk_time
```

Zero Bubble refinement brings it down but not to zero. DualPipe, in the stable phase, has zero bubble if the micro-batch count is divisible by 2 times the 流水线 深度. Outside the stable phase (预热 and cooldown), there is some bubble but it does not grow with the number of micro-batches — a key property the paper highlights.

In marketing terms: "bubble-free". In technical terms: bubbles do not grow with micro-batch count. Sea AI Lab's follow-up analysis (DualPipeV / Cut-in-half) shows the full zero-bubble only when Expert Parallelism is not the bottleneck; with EP-driven all-to-all, some scheduling compromise is always present.

### DualPipeV — the refinement

Sea AI Lab (2025) observed that the 2x 参数 replication is wasteful when EP comm overlap is not the point. Their DualPipeV 调度 folds the bidirectional injection into a "V-shape" 调度 that runs on a single 参数 copy. The bubble is slightly larger than DualPipe's, but the 内存 savings are substantial. DeepSeek adopted DualPipeV in their open-source DualPipe implementation as an EP-off mode.

这个tradeoff:

|特征|DualPipe|DualPipeV|1F1B|Zero Bubble|
|---------|---------|-----------|------|------------|
|Param copies per device|2|1|1|1|
|Bubble vs micro-batches|constant|small growth|grows|grows|
|Compute-comm overlap|full|partial|minimal|partial|
|Use when|EP-heavy MoE|稠密 or EP-light|基线|any 流水线|

### What it means for a 14.8T-词元 run

DeepSeek-V3's 预训练 consumed 14.8T 词元 on 2,048 H800 GPUs in roughly 2.8M GPU-hours. With naive 1F1B, they would have lost 12-15% of that to 流水线 bubbles — 340-420K GPU-hours, enough to 训练 a full 70B 模型. DualPipe recovered most of that. Directly quantifying the contribution is difficult without the internal logs, but the claim in the paper is over 95% GPU utilization averaged across 训练.

For smaller runs (under 1k GPUs), DualPipe is overkill — 流水线 bubbles are smaller relative to total 成本, and dense-model 训练 rarely hits the all-to-all bottleneck. For frontier MoE 训练 at multi-thousand GPU 规模, it is effectively required.

### Where it sits in the stack

- Complementary to **FSDP** (Phase 10 · 05). FSDP shards the 模型 参数 across ranks; DualPipe schedules the 计算 across ranks. They combine.
- Compatible with **ZeRO-3** 梯度 sharding. The bookkeeping for the two-copy replication needs to cooperate with ZeRO's sharded gradients.
- Requires **custom all-to-all kernels** tuned for the specific cluster 拓扑. DeepSeek's open-source kernels are the 参考 implementation.

```figure
expert-capacity
```

## 实际使用

`code/main.py` is a 流水线 调度 simulator. It takes `(P, n_micro_batches, schedule)` and prints the stable-phase utilization for each of 1F1B, Zero Bubble, DualPipe, and DualPipeV. It is a teaching 工具 — the numbers match the qualitative claims in the papers, they are not a claim about 生产 measured speedup.

这个simulator's value: run it with different P and micro-batch counts and watch how the bubble fraction grows for 1F1B but not DualPipe.

Integration considerations for a 真实 训练 run:

- 选择a pipeline-parallel 深度 that divides cleanly into your micro-batch count.
- Ensure your expert-parallel mesh supports bidirectional all-to-all. DeepSeek's kernels are the 参考.
- Expect to burn a week of 调试 time on the 调度 itself the first time. The bookkeeping is fiddly.
- Monitor GPU utilization per 排序, not just aggregate. DualPipe's benefit comes from tightening the stragglers.

## 交付成果

这lesson produces `outputs/skill-dualpipe-planner.md`. Given a 训练 cluster specification (GPU count, 拓扑, interconnect, 模型 shape), it recommends a 流水线 parallelism strategy, the scheduling algorithm to use, and the expected bubble fraction at the 目标 规模.

## 练习

1. 运行`code/main.py` on `(P=8, micro_batches=16, schedule=dualpipe)` and `(P=8, micro_batches=16, schedule=1f1b)`. 计算 the GPU utilization difference and express it as recovered GPU-hours per million 词元 of 训练.

2. Sketch the 调度 table for `(P=4, micro_batches=8, schedule=dualpipe)` by hand. Mark each time slot with the micro-batch ID and direction. Identify the first time slot where bubbles are absent.

3. Read Figure 5 of the DeepSeek-V3 technical report (arXiv:2412.19437). Identify the overlap window for all-to-all dispatch inside a DualPipe forward 分块. Explain how the 计算 调度 hides it.

4. 计算 the 2x 参数 overhead of DualPipe for a 70B 稠密 模型 with P=8 流水线 stages and a 671B MoE 模型 with P=16 流水线 stages. Show why the MoE case's overhead is proportionally smaller (most 参数 are experts, sharded across a large EP group).

5. 比较DualPipe to Chimera (a competing bidirectional scheduler from 2021). Identify the two specific properties DualPipe added that Chimera did not have, using the paper's Section 3.4 as the 参考.

## Key Terms

|Term|What people say|What it actually means|
|------|----------------|------------------------|
|流水线 bubble|"Idle time per 排序"|GPU cycles wasted because a 流水线 stage is waiting for its 输入 or 梯度|
|1F1B|"Default 流水线 调度"|One forward / one backward interleaved scheduling; the 基线 DualPipe beats|
|Zero Bubble|"Sea AI Lab 2023"|Splits backward into B (输入 梯度) and W (权重 梯度); almost fully tightens the 流水线|
|DualPipe|"DeepSeek-V3 调度"|Bidirectional 流水线 + compute-comm overlap; bubbles do not grow with micro-batch count|
|DualPipeV|"Cut-in-half"|V-shape refinement that drops the 2x 参数 replication at the 成本 of slightly larger bubbles|
|分块|"Unit of 流水线 work"|A forward or backward pass of one micro-batch through one 流水线 stage|
|All-to-all dispatch|"Send 词元 to experts"|Cross-node comm that routes 词元 to their assigned MoE experts|
|All-to-all combine|"Bring expert outputs back"|Cross-node comm that gathers expert outputs after the MLP|
|Expert Parallelism (EP)|"Experts across GPUs"|Shards MoE experts across ranks so different GPUs hold different experts|
|流水线 Parallelism (PP)|"层 across GPUs"|Shards 模型 层 across ranks; the 维度 DualPipe schedules|
|Bubble fraction|"Wasted GPU time"|(bubble_time / total_time); the fraction DualPipe drives toward zero|

## 延伸阅读

- [DeepSeek-AI — DeepSeek-V3 Technical Report (arXiv:2412.19437), Section 3.3.2 and Figure 5](https://arxiv.org/abs/2412.19437) — the primary DualPipe 参考
- [DeepSeek — DualPipe GitHub repository](https://github.com/deepseek-ai/DualPipe) — the open-source 参考 implementation, including DualPipeV (Cut-in-half) mode
- [Qi et al. — Zero Bubble Pipeline Parallelism (arXiv:2401.10241, Sea AI Lab 2023)](https://arxiv.org/abs/2401.10241) — the Zero Bubble predecessor
- [Sea AI Lab — DualPipe could be better without the Dual](https://sail.sea.com/blog/articles/63) — the DualPipeV analysis that informed DeepSeek's EP-off mode
- [Narayanan et al. — PipeDream / 1F1B (arXiv:1806.03377, 2018-2021)](https://arxiv.org/abs/1806.03377) — the 1F1B 调度 DualPipe compares against
- [Huang et al. — GPipe (arXiv:1811.06965, 2018)](https://arxiv.org/abs/1811.06965) — the original 流水线 parallelism paper and bubble problem
