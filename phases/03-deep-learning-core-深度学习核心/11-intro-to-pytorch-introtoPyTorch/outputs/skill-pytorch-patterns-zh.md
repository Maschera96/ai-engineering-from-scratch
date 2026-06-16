---
name: skill-pytorch-patterns-zh
description: Reference patterns 用于 PyTorch 训练, evaluation, 和 deployment
version: 1.0.0
phase: 03
lesson: 11
tags: [pytorch, training, deep-learning, gpu, patterns]
---

## Canonical 训练循环

```python
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
model = Model().to(device)
criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.AdamW(model.parameters(), lr=1e-3, weight_decay=0.01)

for epoch in range(num_epochs):
    model.train()
    for inputs, targets in train_loader:
        inputs, targets = inputs.to(device), targets.to(device)
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, targets)
        loss.backward()
        torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)
        optimizer.step()

    model.eval()
    with torch.no_grad():
        for inputs, targets in val_loader:
            inputs, targets = inputs.to(device), targets.to(device)
            outputs = model(inputs)
```

## Mixed Precision 训练

```python
from torch.amp import autocast, GradScaler

scaler = GradScaler()
for inputs, targets in train_loader:
    inputs, targets = inputs.to(device), targets.to(device)
    optimizer.zero_grad()
    with autocast(device_type="cuda"):
        outputs = model(inputs)
        loss = criterion(outputs, targets)
    scaler.scale(loss).backward()
    scaler.step(optimizer)
    scaler.update()
```

使用 当: 训练 在 GPU 用 float16-capable hardware (V100, A100, H100, RTX 3090+). Expect ~1.5-2x speedup 和 ~50% 内存 reduction.

## 梯度 Accumulation

```python
accumulation_steps = 4
optimizer.zero_grad()
for i, (inputs, targets) in enumerate(train_loader):
    inputs, targets = inputs.to(device), targets.to(device)
    outputs = model(inputs)
    loss = criterion(outputs, targets) / accumulation_steps
    loss.backward()
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

使用 当: effective 批次 size needs 到 be larger than GPU 内存 allows. Dividing 损失 by accumulation_steps keeps 梯度 尺度 consistent.

## Save 和 Load

```python
torch.save({
    "epoch": epoch,
    "model_state_dict": model.state_dict(),
    "optimizer_state_dict": optimizer.state_dict(),
    "loss": loss.item(),
}, "checkpoint.pt")

checkpoint = torch.load("checkpoint.pt", weights_only=True)
model.load_state_dict(checkpoint["model_state_dict"])
optimizer.load_state_dict(checkpoint["optimizer_state_dict"])
```

始终 save 优化器 state 用于 resuming 训练. For 推理-only, save just`model.state_dict()`.

## Custom 数据set

```python
class CustomDataset(torch.utils.data.Dataset):
    def __init__(self, data_dir, transform=None):
        self.samples = self._load_samples(data_dir)
        self.transform = transform

    def __len__(self):
        return len(self.samples)

    def __getitem__(self, idx):
        x, y = self.samples[idx]
        if self.transform:
            x = self.transform(x)
        return x, y

    def _load_samples(self, data_dir):
        ...
```

## 数据Loader Configuration

```python
train_loader = torch.utils.data.DataLoader(
    dataset,
    batch_size=64,
    shuffle=True,
    num_workers=4,
    pin_memory=True,
    drop_last=True,
    persistent_workers=True,
)
```

|Parameter|What it does|When 到 使用|
|-----------|-------------|-------------|
|num_workers=4|Parallel 数据 loading|始终 在 multi-core machines|
|pin_memory=True|Page-locked CPU 内存|When 训练 在 GPU|
|drop_last=True|Drop incomplete final 批次|When using BatchNorm|
|persistent_workers=True|Keep workers alive across 轮次|When num_workers > 0|

## 学习率 Schedules

```python
scheduler = torch.optim.lr_scheduler.OneCycleLR(
    optimizer,
    max_lr=1e-3,
    total_steps=num_epochs * len(train_loader),
    pct_start=0.1,
)

for epoch in range(num_epochs):
    for inputs, targets in train_loader:
        ...
        optimizer.step()
        scheduler.step()
```

OneCycleLR: best 默认 用于 most 任务. Warms up 到 max_lr, 然后 cosine decays. Call`scheduler.step()`之后 every 批次, 不 every 轮次.

## Weight Initialization

```python
def init_weights(module):
    if isinstance(module, nn.Linear):
        nn.init.kaiming_normal_(module.weight, nonlinearity="relu")
        if module.bias is not None:
            nn.init.zeros_(module.bias)
    elif isinstance(module, nn.Conv2d):
        nn.init.kaiming_normal_(module.weight, mode="fan_out", nonlinearity="relu")

model.apply(init_weights)
```

## Inference Mode

```python
model.eval()

with torch.inference_mode():
    outputs = model(inputs)
```

`torch.inference_mode()`是 faster than`torch.no_grad()`因为 it disables autograd entirely rather than just suppressing 梯度 computation.

## Common Mistakes Checklist

1. Applying softmax 之前 CrossEntropy损失 (it includes log_softmax internally)
2. Forgetting 到 call 模型.eval() during 验证
3. Forgetting 到 move 张量 到 same 设备 as 模型
4. Not calling 优化器.zero_grad() (梯度s accumulate by 默认)
5. Using torch.no_grad() during 训练 (disables 梯度 computation)
6. Setting num_workers too high (spawns too many processes, thrashes 内存)
7. Not using pin_memory=True 当 训练 在 GPU
8. Saving entire 模型 object instead of state_dict (breaks 在 refactor)
