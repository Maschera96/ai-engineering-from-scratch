---
name: checkpoint-save-resume-zh
description: 原子化、分片式 checkpoints，完整捕获 RNG，让被杀掉的 run 能在 epoch 中途恢复，并保持相同的 loss trajectory。
version: 1.0.0
phase: 19
lesson: 47
tags: [training, durability, resume, sharded-state]
---

## 何时使用

任何训练时间超过 cluster wallclock cap 的 run，任何必须承受 node reboot 的 run，任何大到无法放进单个 payload 的 model。

## Payload shape

```python
{
  "schema": "ckpt.v1",
  "model": model.state_dict(),
  "optimizer": opt.state_dict(),
  "scheduler": sched.state_dict(),
  "state": {"step": int, "epoch": int, "batch_in_epoch": int, "losses": [float, ...]},
  "rng": {"python": ..., "numpy": ..., "torch_cpu": ..., "torch_cuda": ...},
  "wall_saved_at": time.time(),
}
```

## Atomic save

1. 将 payload 写入与 target 位于同一目录的唯一 temp file。
2. 用 `os.replace(tmp, target)` 原子化交换。
3. 永远不要直接写入 target name。

## Sharded layout

- 每个 shard 一个 `model.shard-NNN.pt`，按 keys round robin，或按 parameter group 拆分。
- `meta.pt` 承载 optimizer、scheduler、train state、RNG 和 shard manifest。
- `index.json` 承载每个 shard 和 `meta.pt` 的 `sha256`。
- Loader 在合并前验证每个 hash。

## Mid-epoch resume

- 在 `step` 旁边保存 `(epoch, batch_in_epoch)`。
- 在恢复后 epoch 的第一个 batch 之前恢复 RNG state。
- 将 generator 快进，跳过已消费的 batches。

## Failure modes

- 跨设备 rename：不是原子操作，会丢掉旧文件。把 temp 放在同一目录。
- 忘记 RNG：恢复后的 loss 会偏离 baseline。运行 demo 的 assertion。
- 忘记 optimizer state：下一步会猛跳。同样的 diff 会爆炸。
- 删除了错误的 checkpoint：保留最近 K 个以及 best。
