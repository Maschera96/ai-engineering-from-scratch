# Pipeline 并行与气泡分析

> Tensor parallelism 把矩阵乘法拆到多个 ranks 上。Pipeline parallelism 把模型拆到多个 ranks 上，每个 rank 一个 stage。Microbatches 在 pipeline 中流动。开头和结尾的空闲时间就是气泡，最小化它是全部手艺。

**Type:** Build
**Languages:** Python
**Prerequisites:** Phase 19 Track C lessons 42-49
**Time:** ~90 min

## Learning Objectives

- 把一个顺序模型拆成 N 个 stage，并在 N 个 ranks 上模拟 forward pipeline。
- 使用 GPipe 调度，把 M 个 microbatches 送入 pipeline，先 forward-only fill，再 backward，并计算气泡比例。
- 比较气泡与 Megatron-LM 和 PipeDream 使用的 interleaved 1F1B 调度。
- 解释 stage 分配：每个 stage 的 compute 均衡，比参数数量均衡更重要。

## 问题

一个 fp16 的 70B 参数模型，仅参数就需要 140 GB。单张消费级 GPU 放不下。ZeRO-3 能把参数分片到多个 ranks 上，但每层 forward 仍要每个 rank allgather 完整层，付出每层 log(N) 跳。Pipeline parallel 走了另一条路：把模型切成 N 个 stage，每个 rank 放一个 stage。第 0 层的 forward 在 rank 0 完成，把 activation tensor 交给 rank 1；rank 1 运行第 2 层并交给 rank 2；依此类推。Backward 反向流动。内存线性下降，因为每个 rank 只持有一个 stage；compute 是串行的，这就产生了气泡问题。

气泡是 pipeline 开头的空闲时间，等待第一个 microbatch 到达最后一个 stage，以及结尾的空闲时间，等待最后一个 microbatch 从尾部排空。对于 M 个 microbatches 和 N 个 stages，每个 stage 的气泡比例是 `(N-1)/(M+N-1)`。M=8、N=4 时是 27%。M=64、N=4 时是 4.5%。当每步有很多 microbatches 时，气泡会缩小，这意味着每个 microbatch 的 batch size 很小，这正是 microbatch 设计的约束。

## 概念

```mermaid
flowchart LR
  R0[rank 0: stage 0 / layer 0] --> R1[rank 1: stage 1 / layer 1]
  R1 --> R2[rank 2: stage 2 / layer 2]
  R2 --> R3[rank 3: stage 3 / loss]
  R3 -.backward.-> R2
  R2 -.backward.-> R1
  R1 -.backward.-> R0
```

### GPipe 调度

先把所有 M 个 microbatches 前向填满 pipeline，然后再反向排空。每个 microbatch 的 activation 都必须保留到它的 backward，因此 memory 随 M 线性增长。Forward 需要 M+N-1 个周期，backward 再需要 M+N-1 个周期。每个 stage 的有用工作是 2M 个周期；每个 stage 的气泡是 2(N-1) 个周期。若每次 forward 和 backward 视为一个单位时间，气泡比例是 `(N-1)/(M+N-1)`。让 M 远大于 N 能隐藏气泡。

### 1F1B 调度

交错执行：一旦一个 microbatch 的 forward 到达最后一个 stage，就启动它的 backward，并让它流回去。调度在每个 stage 上交替一个 forward 和一个 backward。气泡仍然是 N-1，但 activation memory 受 pipeline 深度而不是 microbatch 数限制。生产 pipeline 使用 1F1B，Megatron 和 PipeDream 都这样。先实现 GPipe 是因为更简单，1F1B 作为练习。

### 为什么每个 stage 的 compute 要均衡

如果 stage 0 需要 50 ms，stage 1 需要 100 ms，那么每个 cycle 都被 stage 1 卡住。其他 stage 每个 cycle 都会空等 50 ms。参数数量均衡是错的轴：Transformer 的 compute 主要由每层 attention 和 MLP 决定，而 embedding 层参数很多，但 compute 很少。stage 分配应该均衡 FLOPs，而不是 weights。

### Microbatch 与 batch

一个 pipeline 运行 M 个大小为 B 的 microbatch。有效 batch size 是 M*B。Pipeline step  শেষে的梯度就是这 M*B 个样本合在一起的梯度。气泡比例依赖 M；优化器看到的是 M*B。调 M 就是在气泡更低，意味着更高 M，与每 microbatch 的内存更高，GPipe 下 activation memory 也更高，之间做权衡。

## 构建

`code/main.py` 实现：

- `PipelineStage`，一个小型 `nn.Module`，持有一个 stage 的参数，并暴露 `forward(activation)`。
- `Pipeline(stages, num_microbatches)`，在模拟 stage 上使用模拟 wall-clock 调度 GPipe。
- `bubble_fraction(num_stages, num_microbatches)`，闭式 `(N-1)/(M+N-1)`。
- 一个 4-stage 演示，打印逐 microbatch trace 和测得的气泡比例。

运行：

```bash
python3 code/main.py
```

输出：stage-by-microbatch 甘特图，以及与闭式预测对比的气泡百分比。

## 野外生产模式

三种模式让 pipeline parallel 足够可靠，可以发布。

**Activation checkpointing pairs with pipeline.** 在 GPipe 上有 M 个 microbatch 飞行时，activation memory 是 M 倍单个 microbatch。Activation checkpointing 在 backward 时重算 forward，用 compute 换 memory；正是这两个组合让长序列 pipeline 变得可行。

**Stage balance is measured, not assumed.** 生产团队会做 profiling，测量目标硬件上每层的实际 compute，FLOPs 和 wall-clock，然后按这些测量切分。Megatron-LM 的 `--num-layers-per-stage` flag 接受一个列表，从而允许 stage layer 数不均等，只要每层成本不同。

**Send-recv schedule must avoid deadlock.** 一个 pipeline 如果每个 stage 都先 send 再 receive，会在网络上死锁。标准修复是交错：偶数 rank 的 stage 先 send 后 recv，奇数 rank 的 stage 先 recv 后 send。本课显式安排 ranks，模式会清楚可见。

## 使用

生产模式：

- **Megatron-LM.** 大规模 pipeline parallel 的参考实现。使用 1F1B，并支持 tensor + pipeline + data parallel 组合。
- **DeepSpeed Pipeline.** 与 ZeRO 集成；ZeRO-1 + pipeline 是最大开源模型的常见组合。
- **PyTorch Pipe.** PyTorch 原生 pipeline wrapper，基于 `torch.distributed.pipeline.sync.Pipe`。

## 交付

第 80 课会把每个 stage 的参数 shard 存入 sharded checkpoint。第 81 课把 DDP + ZeRO + pipeline 组合进端到端演示，虽然 demo 仍把 pipeline 保持为模拟态，以便控制运行时间。

## 练习

1. 实现 1F1B，并验证气泡比例与 GPipe 相同，但 activation memory 有界。
2. 在更深模型上测量真实每 stage 时间，并按测得的 wall-clock 重新平衡 stage。
3. 给 pipeline microbatches 添加 gradient accumulation，并检查梯度等于等价大 batch forward 的梯度。
4. 给 pipeline 配上 activation checkpointing，并测量相对 compute 成本的 memory 下降。
5. 把 pipeline 与 DDP 结合起来，每个 pipeline rank 在一个 data-parallel group 中复制，并推导二维调度。

## 关键术语

| Term | What people say | What it actually means |
|------|----------------|------------------------|
| Pipeline | “Model parallel along depth” | 每个 rank 一个 stage，activation 在 stage 间流动 |
| Bubble | “Pipeline idle time” | 开头和结尾 N-1 个步骤中部分 stage 没有工作 |
| Microbatch | “Slice of the batch” | 一个 forward/backward 单位；M 越大气泡越小 |
| GPipe | “Fill then drain” | 所有 M 个 forward 先跑完，再跑 backward；activation memory 高 |
| 1F1B | “Interleaved schedule” | 每个 stage 一个 forward 一个 backward；activation memory 有界 |

## 延伸阅读

- [Huang et al, GPipe: Efficient Training of Giant Neural Networks](https://arxiv.org/abs/1811.06965)
- [Narayanan et al, PipeDream: Generalized Pipeline Parallelism for DNN Training](https://arxiv.org/abs/1806.03377)
- [Megatron-LM pipeline parallel docs](https://github.com/NVIDIA/Megatron-LM)
- Phase 19 Lesson 76，调度使用的 send/recv 原语
- Phase 19 Lesson 78，ZeRO 与 pipeline 正交且常被组合使用
