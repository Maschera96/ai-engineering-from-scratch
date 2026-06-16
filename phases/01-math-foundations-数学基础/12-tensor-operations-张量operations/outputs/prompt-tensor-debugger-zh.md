---
name: prompt-tensor-debugger-zh
description: Step-by-step debugging prompt for 张量 形状 误差 in deep 学习 code
phase: 1
lesson: 12
---

I have a 张量 形状 误差 in my deep 学习 code. Help me fix it.

**Error message:** [paste the 误差 here]

**My 张量 形状:**
- [name]: [形状]
- [name]: [形状]

**The operation I'm trying to do:** [describe it]

---

当debugging, follow this exact 过程:

**第 1 步: Identify the operation type.**
What operation produced the 误差? Map it to one of these:
- 矩阵 相乘 / Linear layer (inner 维度 must match)
- Broadcasting (align from right, each dim must be equal 或 1)
- Concatenation (all dims match except the cat 维度)
- Convolution (expects specific rank 和 channel position)
- Reshape (total elements must be preserved)

**第 2 步: Write out the 形状 contract.**
For the identified operation, write the expected 形状 explicitly:
```
matmul(A, B): A is (..., m, k), B is (..., k, n) -> (..., m, n)
broadcast(A, B): align right, each pair must be (equal) or (one is 1)
cat([A, B], dim=d): all dims match except dim d
Linear(in_f, out_f): input last dim must equal in_f
Conv2d(in_c, out_c, k): input must be (B, in_c, H, W)
```

**第 3 步: Find the mismatch.**
Compare actual 形状 against the contract. Identify the exact 维度 that violates the rule.

**第 4 步: Choose the minimal fix.**
Pick from this table:

| Symptom | Fix |
|---|---|
| Missing batch 维度 | `.unsqueeze(0)` |
| Missing channel 维度 | `.unsqueeze(1)` |
| Extra size-1 维度 | `.squeeze(dim)` |
| Inner dims wrong for matmul | `.transpose(-1, -2)` 或 check weight 形状 |
| Need NCHW from NHWC | `.permute(0, 3, 1, 2)` |
| Need NHWC from NCHW | `.permute(0, 2, 3, 1)` |
| Flatten spatial dims for linear | `.flatten(1)` 或 `.reshape(B, -1)` |
| Split heads: (B,T,D) to (B,H,T,D/H) | `.reshape(B, T, H, D//H).transpose(1, 2)` |
| Merge heads: (B,H,T,D/H) to (B,T,D) | `.transpose(1, 2).reshape(B, T, H*(D//H))` |
| Non-contiguous 张量 与 .view() | `.contiguous().view(...)` 或 use `.reshape(...)` |

**第 5 步: Verify the fix.**
S如何the resulting 形状 at each step. Confirm total elements are preserved across any reshape. Confirm the operation's 形状 contract is now satisfied.

**第 6 步: Check for silent bugs.**
Even if 形状 match, verify:
- Broadcasting is happening along the intended 轴 (not accidentally)
- Reduction is summing over the right 维度
- The batch 维度 (dim 0) survives through the entire forward pass
- Transpose + reshape is used (not just reshape) when 维度 ordering matters

Format your response as:
```
OPERATION: [what operation failed]
EXPECTED: [shape contract]
ACTUAL: [what shapes were provided]
MISMATCH: [which dimension, why]
FIX: [exact code]
RESULT: [shapes after fix]
```
