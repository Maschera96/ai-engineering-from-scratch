---
name: distributed-fsdp-ddp-zh
description: 在 gloo 或 nccl backend 上启动 multi-rank training，包含从零实现的 DDP wrapper 和 FSDP parameter sharding sketch。
version: 1.0.0
phase: 19
lesson: 48
tags: [distributed, ddp, fsdp, collectives]
---

## 何时使用

Model 能放进单个 device，但你需要更高 throughput，这就是 DDP。Model 放不进单个 device，这就是 FSDP。两种情况都需要 multi-rank training setup，并共用同一条 code path。

## Bring up the process group

```python
os.environ["MASTER_ADDR"] = "127.0.0.1"
os.environ["MASTER_PORT"] = str(port)
dist.init_process_group(backend="gloo", rank=rank, world_size=world_size)
```

`gloo` 是 CPU backend，`nccl` 是 GPU backend。两者实现相同的 collective surface。

## Wrap the model

1. 在 rank 0 上，用你的 seed 构建 model。
2. 用 DDP shell 包裹它。
3. Shell 的 `__init__` 会对每个 parameter 和 buffer 调用 `dist.broadcast(p.data, src=0)`。
4. 每次 `loss.backward()` 之后，trainer 调用 `sync_grads()`。
5. `sync_grads()` 调用 `dist.all_reduce(p.grad, op=SUM)`，然后执行 `p.grad.div_(world_size)`。
6. 每个 rank 都用相同的 averaged gradient 执行 optimizer step。

## Shard parameters (FSDP sketch)

1. Flatten 每个 parameter，并 pad 到 `world_size` 的倍数。
2. 本地只保留自己的 shard，释放其余部分。
3. Forward 前，执行 `dist.all_gather(...)`，在每个 rank 上重建完整 tensor。
4. Forward 后，丢弃完整 tensor。

## Failure modes

- 跳过 broadcast：ranks 从不同 init 开始，会静默 diverge。
- Sum 后忘记除法：gradients 按 world_size 缩放，optimizer steps 过大。
- 对 checkpoints 使用跨设备 rename：不是原子操作，和 lesson 47 是同一个陷阱。
- 在同一个 collective 中混用 CPU 和 CUDA tensors：backend mismatch，run 会挂起。
