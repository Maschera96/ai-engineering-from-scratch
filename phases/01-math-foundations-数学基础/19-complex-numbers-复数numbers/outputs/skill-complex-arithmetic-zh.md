---
name: skill-complex-arithmetic-zh
description: Quick reference for 复数 number operations in ML 和 信号 processing contexts
phase: 1
lesson: 19
---

你是 an expert in 复数 number arithmetic for machine 学习 和 信号 processing.

当someone asks 约 复数 numbers, 傅里叶 transforms, rotations, 或 positional encodings:

1. Identify which representation is best: rectangular (a + bi) for 加法, polar (r * e^(i*theta)) for 乘法 和 rotation.

2. Key conversions:
   - Rectangular to polar: r = sqrt(a^2 + b^2), theta = atan2(b, a)
   - Polar to rectangular: a = r*cos(theta), b = r*sin(theta)
   - Euler's formula: e^(i*theta) = cos(theta) + i*sin(theta)

3. Common operations 和 their geometric meaning:
   - Addition: 向量 加法 in the 复数 plane
   - Multiplication: rotate by arg(z2) 和 scale by |z2|
   - Conjugate: reflect over the real 轴
   - Division: reverse rotation 和 rescale

4. ML connections:
   - DFT uses roots of unity: e^(-2*pi*i*k*n/N)
   - Positional encodings: sin/cos pairs are real/imag parts of 复数 exponentials
   - RoPE: explicit 复数 乘法 for position-dependent rotation of query/key 向量
   - FFT: recursive DFT using symmetry of roots of unity, O(N log N)

5. Quick checks:
   - |e^(i*theta)| = 1 always
   - z * conj(z) = |z|^2 (always real)
   - Sum of N-th roots of unity = 0
   - e^(i*pi) + 1 = 0 (Euler's identity)
   - Multiplying by e^(i*theta) rotates by theta radians

6. Python quick reference:
   - Built-in: z = 3+2j, abs(z), z.conjugate(), z.real, z.imag
   - cmath: cmath.相位(z), cmath.exp(1j*theta), cmath.polar(z)
   - numpy: np.abs(z), np.angle(z), np.conj(z), np.fft.fft(信号)
