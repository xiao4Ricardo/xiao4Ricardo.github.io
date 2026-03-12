---
layout:     post
title:      "Mamba 与 State Space Models：Transformer 的挑战者"
subtitle:   "Linear-Time Sequence Modeling Beyond Attention"
date:       2026-03-07 15:00:00
author:     "Tony L."
header-img: "img/post-bg-rwd.jpg"
tags:
    - Deep Learning
    - Mamba
    - SSM
    - AI
---

## Transformer 的局限性

Transformer 虽然强大，但 Self-Attention 的 O(n^2) 复杂度在处理长序列时成为瓶颈：

| 序列长度 | Attention 计算量 | 内存占用 |
|----------|-----------------|----------|
| 1K | 1x | 1x |
| 4K | 16x | 16x |
| 16K | 256x | 256x |
| 128K | 16,384x | 16,384x |

这意味着将上下文窗口从 4K 扩展到 128K，计算和内存需求增长了 1000 倍以上。虽然 Flash Attention 等技术优化了常数因子，但 O(n^2) 的根本瓶颈依然存在。

State Space Models（SSM）提供了一条 O(n) 复杂度的替代路径。

## State Space Models 基础

SSM 源自控制理论中的状态空间方程：

```
h'(t) = A * h(t) + B * x(t)    # 状态更新
y(t) = C * h(t) + D * x(t)     # 输出
```

其中：
- x(t)：输入信号
- h(t)：隐藏状态
- y(t)：输出信号
- A, B, C, D：可学习的参数矩阵

这个连续系统可以离散化为：

```
h_t = Ā * h_{t-1} + B̄ * x_t
y_t = C * h_t
```

看起来很像 RNN？确实如此。但 SSM 的关键区别在于它可以同时以两种模式运行：
- **递归模式**（像 RNN）：用于推理，O(1) per step
- **卷积模式**（像 CNN）：用于训练，可并行化

### 训练时的卷积视角

展开递归关系：

```
y_0 = C * B̄ * x_0
y_1 = C * Ā * B̄ * x_0 + C * B̄ * x_1
y_2 = C * Ā^2 * B̄ * x_0 + C * Ā * B̄ * x_1 + C * B̄ * x_2
```

这实际上是一个卷积操作，卷积核为：

```
K = [C*B̄, C*Ā*B̄, C*Ā^2*B̄, ...]
```

可以用 FFT 加速，实现 O(n log n) 的训练复杂度。

## S4: Structured State Space

S4 (Structured State Spaces for Sequence Modeling) 是 SSM 复兴的开端。它的关键创新是对矩阵 A 使用 HiPPO 初始化：

HiPPO（High-order Polynomial Projection Operators）矩阵让状态 h 能够高效地记忆历史输入的多项式近似。简单来说，它让模型天生擅长处理长距离依赖。

但 S4 有一个限制：A, B, C 参数是固定的（不依赖于输入），这意味着模型无法根据不同的输入内容动态调整其"关注"模式。

## Mamba：选择性状态空间

Mamba 的核心创新是**选择性机制**（Selective Mechanism）：让 B, C, Δ 参数依赖于输入，实现输入感知的动态状态更新。

```python
class SelectiveSSM(nn.Module):
    def __init__(self, d_model, d_state, d_conv=4):
        super().__init__()
        self.d_model = d_model
        self.d_state = d_state

        # 投影层
        self.in_proj = nn.Linear(d_model, d_model * 2)  # x 和 z
        self.conv1d = nn.Conv1d(d_model, d_model, d_conv,
                               padding=d_conv-1, groups=d_model)

        # 选择性参数（输入依赖）
        self.x_proj = nn.Linear(d_model, d_state * 2 + 1)  # B, C, dt

        # 固定参数
        self.A_log = nn.Parameter(torch.randn(d_model, d_state))
        self.D = nn.Parameter(torch.ones(d_model))

        self.out_proj = nn.Linear(d_model, d_model)

    def forward(self, x):
        # x: [batch, seq_len, d_model]
        xz = self.in_proj(x)  # [batch, seq_len, 2*d_model]
        x, z = xz.chunk(2, dim=-1)

        # 1D convolution
        x = self.conv1d(x.transpose(1, 2)).transpose(1, 2)
        x = F.silu(x)

        # 选择性参数
        x_proj = self.x_proj(x)
        dt, B, C = x_proj.split([1, self.d_state, self.d_state], dim=-1)
        dt = F.softplus(dt)

        # 离散化
        A = -torch.exp(self.A_log)
        dA = torch.exp(dt * A)
        dB = dt * B

        # 选择性扫描（高效实现）
        y = selective_scan(x, dA, dB, C)

        # 门控
        y = y * F.silu(z)
        return self.out_proj(y)
```

### 选择性机制的直觉

在 Transformer 的 Attention 中，模型可以动态地"选择"关注输入的哪些部分。SSM 的固定参数无法做到这一点。

Mamba 通过让参数依赖于输入来解决这个问题：
- 遇到重要信息时，dt（步长）变大 → 状态更新幅度大 → "记住"这个信息
- 遇到不重要信息时，dt 变小 → 状态基本不变 → "忽略"这个信息

这就是"选择性"的含义——模型学会选择性地记忆和遗忘信息。

### 硬件感知算法

Mamba 的另一个关键贡献是高效的硬件实现。Selective Scan 的朴素实现是 O(n) 的串行循环，无法利用 GPU 的并行能力。

Mamba 使用了类似 Flash Attention 的思路：
1. 避免将中间状态写入 GPU HBM
2. 在 SRAM 中完成计算
3. 使用并行扫描（parallel scan）算法

## Mamba 2

Mamba 2 将 SSM 的状态更新重新表述为半可分矩阵的结构化矩阵乘法，这使得：

1. **理论连接**：证明了 SSM 和 Attention 在特定条件下是等价的
2. **SSD 层**：State Space Duality 层可以利用 GPU 的 Tensor Core 加速
3. **速度提升**：比 Mamba 1 快 2-8 倍

## 混合架构：Mamba + Transformer

纯 Mamba 和纯 Transformer 各有优劣。最新的研究趋势是将两者结合：

```
Block 1: Mamba Layer        ← 高效处理长序列
Block 2: Mamba Layer
Block 3: Attention Layer    ← 精确的全局信息交互
Block 4: Mamba Layer
Block 5: Mamba Layer
Block 6: Attention Layer
...
```

Jamba（AI21 Labs）是第一个大规模混合架构模型，交替使用 Mamba 和 Transformer 层。

### 为什么混合有效？

- **Mamba 层**：O(n) 复杂度，高效处理局部模式和序列压缩
- **Attention 层**：精确的全局 token-to-token 交互（但只需要少量层）

实验表明，每 4-6 个 Mamba 层搭配 1 个 Attention 层，就能在保持线性复杂度的同时获得接近纯 Transformer 的效果。

## 性能对比

在长序列任务上的对比（approximate）：

| 模型 | 1K tokens | 8K tokens | 64K tokens | 内存 |
|------|-----------|-----------|------------|------|
| Transformer | 基准 | 基准 | OOM/极慢 | O(n^2) |
| Mamba | ~1.2x | ~3x | ~10x faster | O(n) |
| Hybrid | ~1.1x | ~2.5x | ~8x faster | O(n) |

## 总结

Mamba 和 State Space Models 为序列建模提供了一条 O(n) 复杂度的路径，在长序列任务上展现了显著的效率优势。虽然 Transformer 仍然是当前 LLM 的主流架构，但混合架构（Mamba + Attention）正在成为一个有前途的研究方向。理解 SSM 的原理，是把握 AI 架构演进方向的重要一环。

## References

1. Gu, A. & Dao, T. (2023). "Mamba: Linear-Time Sequence Modeling with Selective State Spaces."
2. Gu, A., et al. (2022). "Efficiently Modeling Long Sequences with Structured State Spaces (S4)."
3. Dao, T. & Gu, A. (2024). "Transformers are SSMs: Generalized Models and Efficient Algorithms Through Structured State Space Duality."
4. Lieber, O., et al. (2024). "Jamba: A Hybrid Transformer-Mamba Language Model."
