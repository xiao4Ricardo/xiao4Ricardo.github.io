---
layout:     post
title:      "Diffusion Models：从噪声中创造世界"
subtitle:   "Understanding the Math and Magic Behind Stable Diffusion"
date:       2026-02-22 10:00:00
author:     "Tony L."
header-img: "img/bg-little-universe.jpg"
tags:
    - Deep Learning
    - Diffusion Models
    - Computer Vision
    - AI
---

## 什么是 Diffusion Models？

Diffusion Models（扩散模型）是一类基于概率论的生成模型。它的核心思想出乎意料地简单：

1. **前向过程（Forward Process）**：逐步向数据中添加高斯噪声，直到数据变成纯噪声
2. **反向过程（Reverse Process）**：学习如何逐步去除噪声，从纯噪声恢复出数据

如果模型能学会"去噪"，那么从随机噪声开始去噪，就能生成全新的数据。

## 数学基础

### 前向过程

给定一个干净的数据点 x_0，前向过程在 T 个时间步内逐渐添加噪声：

```
q(x_t | x_{t-1}) = N(x_t; sqrt(1-β_t) * x_{t-1}, β_t * I)
```

其中 β_t 是噪声调度（noise schedule），控制每一步添加多少噪声。通常 β_t 从很小（如 0.0001）线性增加到较大（如 0.02）。

一个关键的数学性质：我们可以直接从 x_0 跳到任意时间步 t 的 x_t：

```
q(x_t | x_0) = N(x_t; sqrt(α̅_t) * x_0, (1-α̅_t) * I)

where α_t = 1 - β_t, α̅_t = ∏_{s=1}^{t} α_s
```

这意味着：
```python
x_t = sqrt(α̅_t) * x_0 + sqrt(1-α̅_t) * ε,  ε ~ N(0, I)
```

### 反向过程

反向过程也是一个高斯分布，但其均值需要通过神经网络来预测：

```
p_θ(x_{t-1} | x_t) = N(x_{t-1}; μ_θ(x_t, t), σ_t^2 * I)
```

### 训练目标

经过数学推导（变分下界），训练目标简化为：

```
L = E_{t, x_0, ε} [||ε - ε_θ(x_t, t)||^2]
```

即：给定带噪声的 x_t 和时间步 t，让模型预测被添加的噪声 ε。这个目标函数的简洁性是 Diffusion Models 成功的关键之一。

### 训练算法

```python
def train_step(model, x_0):
    # 1. 随机采样时间步
    t = torch.randint(0, T, (batch_size,))

    # 2. 随机采样噪声
    noise = torch.randn_like(x_0)

    # 3. 添加噪声得到 x_t
    x_t = sqrt_alpha_bar[t] * x_0 + sqrt_one_minus_alpha_bar[t] * noise

    # 4. 模型预测噪声
    predicted_noise = model(x_t, t)

    # 5. 计算损失
    loss = F.mse_loss(predicted_noise, noise)
    return loss
```

### 采样算法 (DDPM)

```python
def sample(model, shape):
    x = torch.randn(shape)  # 从纯噪声开始

    for t in reversed(range(T)):
        predicted_noise = model(x, t)

        # 计算均值
        alpha = alphas[t]
        alpha_bar = alpha_bars[t]
        mu = (1 / sqrt(alpha)) * (x - (1-alpha)/sqrt(1-alpha_bar) * predicted_noise)

        # 添加噪声（t > 0 时）
        if t > 0:
            sigma = sqrt(betas[t])
            x = mu + sigma * torch.randn_like(x)
        else:
            x = mu

    return x
```

## U-Net 架构

Diffusion Models 通常使用 U-Net 作为噪声预测网络：

```
Input (x_t, t)
    ↓
[Down Block] 64 → 128 → 256 → 512
    ↓
[Middle Block] 512 (with Self-Attention)
    ↓
[Up Block] 512 → 256 → 128 → 64    (+ skip connections)
    ↓
Output (predicted noise)
```

关键组件：
- **时间嵌入**：将时间步 t 编码为向量，注入到每个 ResNet 块中
- **Self-Attention**：在中间层引入注意力机制，捕捉全局关系
- **Cross-Attention**：用于条件生成（如文本条件）
- **Skip Connections**：连接对应的 down/up 块，保留细节信息

## DDPM → DDIM → Latent Diffusion

### DDIM (Denoising Diffusion Implicit Models)

DDPM 的一个问题是采样需要跑完所有 T 步（通常 T=1000），速度很慢。DDIM 发现可以使用非马尔可夫的采样过程，跳过中间步骤：

```python
# DDIM 采样：只需 50 步而不是 1000 步
timesteps = [1, 21, 41, ..., 981]  # 50 个间隔的时间步
```

DDIM 在只用 50 步的情况下就能生成质量接近 1000 步 DDPM 的图像。

### Latent Diffusion (Stable Diffusion)

Stable Diffusion 的关键创新是在**潜在空间**（latent space）而非像素空间中进行扩散：

```
Image (512x512x3) → VAE Encoder → Latent (64x64x4) → Diffusion → VAE Decoder → Image
```

在潜在空间中操作：
- 维度从 786,432 (512x512x3) 降到 16,384 (64x64x4)
- 计算量减少了约 48 倍
- 同时保留了足够的图像信息

### 文本条件生成

Stable Diffusion 使用 CLIP Text Encoder 将文本 prompt 编码为向量，然后通过 Cross-Attention 注入到 U-Net 中：

```
text_embedding = CLIP_encoder("a cat sitting on a rainbow")

# 在 U-Net 的每个 Attention 层
Attention(Q=image_features, K=text_embedding, V=text_embedding)
```

### Classifier-Free Guidance

提升生成质量的关键技术：

```python
# 同时预测有条件和无条件的噪声
noise_cond = model(x_t, t, text_embedding)
noise_uncond = model(x_t, t, null_embedding)

# 引导：放大条件方向
predicted_noise = noise_uncond + guidance_scale * (noise_cond - noise_uncond)
```

guidance_scale 越大，生成的图像越忠实于 prompt，但多样性降低。通常取 7-12。

## 最新进展

### DiT (Diffusion Transformers)

用 Transformer 替代 U-Net 作为骨干网络。DALL-E 3 和 Sora 都基于 DiT 架构。Transformer 在 scaling 方面的优势使得 DiT 成为趋势。

### Flow Matching

一种更简洁的生成模型框架，将前向过程定义为从数据分布到噪声分布的最优传输路径。Stable Diffusion 3 采用了 Flow Matching。

### Consistency Models

OpenAI 提出的新框架，可以在单步或少步内生成高质量图像，大幅加速推理。

## 总结

Diffusion Models 用一个优雅的数学框架——"加噪声然后学去噪"——实现了惊人的生成能力。从 DDPM 到 Stable Diffusion 再到 Sora，扩散模型正在重塑我们与视觉内容的交互方式。理解其背后的数学原理，是把握 AI 生成式内容发展方向的关键。

## References

1. Ho, J., et al. (2020). "Denoising Diffusion Probabilistic Models."
2. Song, J., et al. (2020). "Denoising Diffusion Implicit Models."
3. Rombach, R., et al. (2022). "High-Resolution Image Synthesis with Latent Diffusion Models."
4. Peebles, W. & Xie, S. (2023). "Scalable Diffusion Models with Transformers."
