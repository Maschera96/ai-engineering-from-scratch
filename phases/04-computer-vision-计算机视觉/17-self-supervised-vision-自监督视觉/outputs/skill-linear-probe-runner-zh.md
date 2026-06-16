---
name: skill-linear-probe-runner-zh
description: 编写 complete linear-probe evaluation f或 any frozen 编码器 和 labelled 数据集
version: 1.0.0
phase: 4
lesson: 17
tags: [self-supervised, evaluation, linear-probe, pytorch]
---

# Linear Probe 运行ner

Evaluate 一个frozen 编码器's 特征 by 训练 一个single linear 分类器 on 到p. st和ard evaluation f或 every 自监督 paper.

## When 到 use

- Comparing 自监督 checkpoints.
- 轨迹ing 特征 quality over pre训练 epochs.
- Deciding wher 一个pretrained 编码器 是good enough f或 一个downstream task 带有out fine-tuning.

## 输入

- `编码器`: frozen `nn.Module` returning 一个fixed-dim 特征 per 图像.
- `特征_dim`: dimensionality 的 编码器 output.
- `train_数据集`: labelled 数据集 (图像, class_id).
- `val_数据集`: held-out set.
- `num_classes`: task classes.
- `epochs`: typically 100 f或 图像Net-scale, 50 f或 smaller 数据集s.

## Steps

1. Set 编码器 到 eval mode 和 `requires_grad=False` on every parameter.
2. 特征-extract both train 和 val sets once. S到re as numpy arrays 或 一个mem或y-mapped file.
3. Train 一个`nn.Linear(特征_dim, num_classes)` on cached 特征 带有 SGD + cosine schedule.
4. St和ard hyperparameters: `lr=0.1`, `momentum=0.9`, `weight_decay=0`, `batch_size=1024`. Linear probe 是surprisingly sensitive 到 `lr` ， sweep if 准确率 是po或.
5. 报告 到p-1 准确率 on val at end 的 训练.

## 输出 template

```python
import torch
import torch.nn as nn
import torch.nn.functional as F
from torch.utils.data import DataLoader
from torch.optim import SGD
from torch.optim.lr_scheduler import CosineAnnealingLR

def extract(encoder, loader, device="cpu"):
    encoder.eval()
    feats, labels = [], []
    with torch.no_grad():
        for x, y in loader:
            f = encoder(x.to(device)).cpu()
            feats.append(f)
            labels.append(y)
    return torch.cat(feats), torch.cat(labels)


def linear_probe(encoder, feature_dim, train_loader, val_loader,
                 num_classes, epochs=50, lr=0.1, device="cpu"):
    for p in encoder.parameters():
        p.requires_grad = False

    f_train, y_train = extract(encoder, train_loader, device)
    f_val, y_val = extract(encoder, val_loader, device)

    head = nn.Linear(feature_dim, num_classes).to(device)
    opt = SGD(head.parameters(), lr=lr, momentum=0.9, weight_decay=0)
    sched = CosineAnnealingLR(opt, T_max=epochs)

    ds = torch.utils.data.TensorDataset(f_train, y_train)
    train_iter = DataLoader(ds, batch_size=1024, shuffle=True)

    best_val = 0.0
    for ep in range(epochs):
        head.train()
        for x, y in train_iter:
            x, y = x.to(device), y.to(device)
            loss = F.cross_entropy(head(x), y)
            opt.zero_grad(); loss.backward(); opt.step()
        sched.step()

        head.eval()
        with torch.no_grad():
            acc = (head(f_val.to(device)).argmax(-1).cpu() == y_val).float().mean().item()
        best_val = max(best_val, acc)
    return best_val
```

## 报告

```
[linear probe]
  encoder:     <name + pretrain checkpoint>
  feature_dim: <int>
  epochs:      <int>
  best_val_top1: <float>
```

## 规则

- 不要 update 编码器 weights during linear probe; that would be 一个fine-tune, not 一个probe.
- Precompute 特征 once; re训练 编码器 on every epoch wastes 100x compute.
- 使用 SGD 带有 cosine schedule 和 no weight decay; Adam sometimes underperf或ms here.
- Sweep learning rates at least once per 编码器 family; optimum varies across SSL methods.
