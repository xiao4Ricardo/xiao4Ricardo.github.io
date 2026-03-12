---
layout:     post
title:      "Transformer 架构深度解析"
subtitle:   "From Self-Attention to Multi-Head Attention"
date:       2025-11-10 14:00:00
author:     "Tony L."
header-img: "img/post-bg-universe.jpg"
tags:
    - Deep Learning
    - NLP
    - Transformer
---

## 前言

2017 年 Google 发表的论文 *"Attention Is All You Need"* 彻底改变了自然语言处理的格局。Transformer 架构摒弃了传统的 RNN 和 CNN 结构，完全依赖注意力机制（Attention Mechanism）来捕捉序列中的长距离依赖关系。如今，从 GPT 到 BERT，从 Claude 到 Gemini，几乎所有主流的大语言模型都建立在 Transformer 之上。

本文将从底层原理出发，逐步拆解 Transformer 的核心组件。

## 1. 为什么需要 Transformer？

在 Transformer 出现之前，序列建模主要依赖 RNN（循环神经网络）及其变体 LSTM 和 GRU。这些模型存在几个根本性的问题：

- **串行计算瓶颈**：RNN 必须按时间步顺序处理输入，无法并行化，训练速度慢。
- **长距离依赖衰减**：即使 LSTM 引入了门控机制，在处理超长序列时，早期信息仍然容易丢失。
- **梯度消失/爆炸**：反向传播通过时间（BPTT）在长序列上容易出现梯度不稳定。

Transformer 通过 Self-Attention 机制，让序列中的每个位置都能直接关注到其他所有位置，从根本上解决了这些问题。

## 2. Self-Attention 机制

Self-Attention 的核心思想是：对于输入序列中的每一个 token，计算它与所有其他 token 之间的关联程度，然后根据这些关联程度对其他 token 的表示进行加权求和。

### 2.1 Query, Key, Value

给定输入矩阵 X（形状为 [n, d_model]），通过三个不同的线性变换生成：

- **Query (Q)**：Q = X * W_Q，代表"我在寻找什么"
- **Key (K)**：K = X * W_K，代表"我能提供什么"
- **Value (V)**：V = X * W_V，代表"我的实际内容"

### 2.2 注意力计算

```
Attention(Q, K, V) = softmax(Q * K^T / sqrt(d_k)) * V
```

其中 `sqrt(d_k)` 是缩放因子，防止点积值过大导致 softmax 梯度消失。这个缩放看似简单，但在实践中至关重要——没有它，当 d_k 较大时，点积的方差会线性增长，使得 softmax 的输出趋近于 one-hot 分布，梯度接近于零。

### 2.3 直觉理解

想象你在读一句话："The cat sat on the mat because it was tired." 当模型处理 "it" 这个词时，Self-Attention 会让 "it" 的 Query 与句子中每个词的 Key 计算相似度。理想情况下，"it" 与 "cat" 之间的注意力权重应该最高，因为 "it" 指代的是 "cat"。

## 3. Multi-Head Attention

单一的注意力头只能捕捉一种类型的关系。Multi-Head Attention 通过并行运行多个注意力头，让模型同时关注不同类型的信息：

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) * W_O
where head_i = Attention(Q * W_Q_i, K * W_K_i, V * W_V_i)
```

例如，在一个 8 头的设置中：
- Head 1 可能关注语法关系（主谓宾）
- Head 2 可能关注指代关系（代词解析）
- Head 3 可能关注位置邻近性
- Head 4 可能关注语义相似性
- ...以此类推

这种设计让模型能够在同一层中同时学习多种不同的注意力模式。

## 4. Position Encoding

由于 Self-Attention 本身是置换不变的（permutation invariant），它无法区分 token 的位置。因此，Transformer 需要显式地注入位置信息。

原始论文使用正弦/余弦位置编码：

```
PE(pos, 2i) = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```

这种编码的巧妙之处在于：对于任意固定偏移量 k，PE(pos+k) 可以表示为 PE(pos) 的线性函数，这使得模型能够学习到相对位置关系。

后来的发展中，RoPE（Rotary Position Embedding）成为了更主流的选择，它将位置信息编码为旋转矩阵，在处理长序列时表现更优。

## 5. Encoder-Decoder 结构

完整的 Transformer 由 Encoder 和 Decoder 两部分组成：

**Encoder**（每层包含）：
1. Multi-Head Self-Attention
2. Layer Normalization + Residual Connection
3. Feed-Forward Network (FFN)
4. Layer Normalization + Residual Connection

**Decoder**（每层包含）：
1. Masked Multi-Head Self-Attention（防止看到未来 token）
2. Multi-Head Cross-Attention（关注 Encoder 输出）
3. Feed-Forward Network

值得注意的是，现代 LLM 大多采用 Decoder-only 架构（如 GPT 系列），而 BERT 采用 Encoder-only 架构。这种简化在实践中被证明对各自的任务类型更加高效。

## 6. Feed-Forward Network

FFN 是 Transformer 中常被忽视但极其重要的组件：

```
FFN(x) = max(0, x * W_1 + b_1) * W_2 + b_2
```

研究表明，FFN 层实际上充当了"知识存储器"的角色——它们存储了模型在训练过程中学到的事实性知识。每个 FFN 层可以被看作是一个巨大的 key-value 记忆体。

## 7. 训练技巧

### 7.1 学习率调度

Transformer 使用了著名的 Warmup + Decay 策略：

```
lr = d_model^(-0.5) * min(step^(-0.5), step * warmup_steps^(-1.5))
```

先线性增加学习率（warmup），再按照逆平方根衰减。这种策略对 Transformer 的稳定训练至关重要。

### 7.2 Layer Normalization

Pre-LN（在 attention/FFN 之前做 LayerNorm）vs Post-LN（在之后做）是一个重要的架构选择。现代实践普遍倾向于 Pre-LN，因为它使训练更加稳定，不容易出现梯度爆炸。

### 7.3 Dropout

在 Attention 权重和 FFN 的隐藏层中应用 Dropout 是标准做法，通常设置为 0.1。

## 总结

Transformer 的成功不是偶然的——它优雅地解决了序列建模中的核心难题：并行计算、长距离依赖和灵活的注意力模式。理解 Transformer 的每一个组件，是深入理解现代 AI 系统的基础。

下一篇我们将探讨 RAG（检索增强生成）技术，看看如何将外部知识融入到基于 Transformer 的 LLM 中。

## References

1. Vaswani, A., et al. (2017). "Attention Is All You Need." NeurIPS.
2. Su, J., et al. (2021). "RoFormer: Enhanced Transformer with Rotary Position Embedding."
3. Xiong, R., et al. (2020). "On Layer Normalization in the Transformer Architecture."
