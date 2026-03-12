---
layout:     post
title:      "注意力机制深度剖析：从 Bahdanau 到 Flash Attention"
subtitle:   "Understanding Attention Mechanisms in Modern Deep Learning"
date:       2025-12-12 11:00:00
author:     "Tony L."
header-img: "img/post-bg-infinity.jpg"
tags:
    - Deep Learning
    - Attention
    - NLP
---

## 注意力机制的起源

注意力机制的灵感来自人类视觉系统——我们不会同时关注视野中的所有信息，而是选择性地聚焦于最相关的部分。这个简单的直觉催生了深度学习中最重要的创新之一。

## 1. Bahdanau Attention (2014)

在 Transformer 之前，Bahdanau 等人首次将注意力机制引入神经机器翻译（NMT）。传统的 Seq2Seq 模型将整个源句子压缩为一个固定长度的向量，这在长句子翻译中损失了大量信息。

Bahdanau Attention 的核心创新是让 Decoder 在生成每个词时，动态地关注 Encoder 输出的不同部分：

```
e_ij = a(s_{i-1}, h_j)          # 对齐分数
alpha_ij = softmax(e_ij)        # 注意力权重
c_i = sum(alpha_ij * h_j)       # 上下文向量
```

其中 `a()` 是一个前馈神经网络。这种注意力也被称为 **Additive Attention**。

## 2. Luong Attention (2015)

Luong 提出了三种更简洁的注意力变体：

- **Dot Product**：`score(s_t, h_s) = s_t^T * h_s`
- **General**：`score(s_t, h_s) = s_t^T * W * h_s`
- **Concat**：`score(s_t, h_s) = v^T * tanh(W * [s_t; h_s])`

Dot Product Attention 计算效率最高，后来成为 Transformer 的基础。

## 3. Scaled Dot-Product Attention

Transformer 使用的注意力公式：

```
Attention(Q, K, V) = softmax(Q * K^T / sqrt(d_k)) * V
```

缩放因子 `sqrt(d_k)` 至关重要。假设 Q 和 K 的每个元素独立且服从均值为 0、方差为 1 的分布，那么它们的点积的方差为 d_k。当 d_k 很大时（如 64 或 128），点积值会变得很大，softmax 会产生接近 one-hot 的分布，梯度几乎为零。除以 `sqrt(d_k)` 将方差重新归一化为 1。

## 4. Multi-Head Attention

Multi-Head Attention 让模型能够同时关注不同表示子空间中的信息：

```
MultiHead(Q, K, V) = Concat(head_1, ..., head_h) * W_O
head_i = Attention(Q * W_Q_i, K * W_K_i, V * W_V_i)
```

如果 d_model = 512, h = 8，那么每个 head 的 d_k = d_v = 64。

研究发现，不同的 attention head 确实学到了不同的模式。例如在 BERT 中，某些 head 专注于语法依赖，某些关注指代关系，某些则负责位置编码。

## 5. 高效注意力变体

标准 Self-Attention 的时间和空间复杂度都是 O(n^2)，这在处理长序列时成为瓶颈。研究者提出了多种高效变体：

### 5.1 Sparse Attention

不计算所有 token 对之间的注意力，而是只关注部分 token：

- **Local Attention**：只关注局部窗口内的 token
- **Strided Attention**：间隔固定步长关注
- **Longformer**：结合 local 和 global attention
- **BigBird**：随机 + 局部 + 全局注意力的组合

### 5.2 Linear Attention

将 softmax 注意力近似为线性运算，复杂度降至 O(n)：

```
# 标准注意力
Attention = softmax(Q * K^T) * V

# 线性注意力 (通过核函数近似)
Attention = phi(Q) * (phi(K)^T * V)
```

通过改变计算顺序（先算 K^T * V），避免了 O(n^2) 的注意力矩阵。

### 5.3 Flash Attention (2022)

Flash Attention 是一个里程碑式的工程优化。它并没有改变注意力的数学计算，而是通过**IO 感知的算法设计**来加速计算：

核心思想：
1. **分块计算（Tiling）**：将 Q, K, V 分成块，每次只将一小块加载到 GPU SRAM 中
2. **减少 HBM 访问**：避免将巨大的 N×N 注意力矩阵写入 GPU 的 HBM（高带宽内存）
3. **重计算（Recomputation）**：在反向传播时重新计算注意力矩阵，而不是存储它

Flash Attention 2 进一步优化了并行性和工作分配，在 A100 上达到了理论最大吞吐量的 72%。

Flash Attention 3 针对 Hopper 架构（H100）进行了优化，利用了异步执行和 FP8 支持。

### 5.4 Grouped Query Attention (GQA)

GQA 是 MHA 和 MQA 之间的折中方案：

- **MHA**：每个注意力头都有独立的 Q, K, V
- **MQA**：所有头共享 K 和 V（大幅减少 KV cache，但质量下降）
- **GQA**：将头分组，每组共享 K 和 V

LLaMA 2 70B 和 Mistral 都采用了 GQA，在推理效率和模型质量之间取得了很好的平衡。

## 6. KV Cache

在自回归生成中，每生成一个新 token，都需要对之前所有 token 重新计算 K 和 V。KV Cache 通过缓存之前计算过的 K 和 V 值来避免重复计算。

KV Cache 的内存占用公式：
```
Memory = 2 * n_layers * n_heads * d_head * seq_len * batch_size * dtype_size
```

对于长序列，KV Cache 可能占用数十 GB 的内存。因此，GQA 和各种 KV Cache 压缩技术（如 PagedAttention in vLLM）变得至关重要。

## 总结

注意力机制从最初的 Seq2Seq 改进发展为现代 AI 的核心构件。理解这些注意力变体及其工程优化，对于高效部署和训练大语言模型至关重要。Flash Attention 和 GQA 的成功也提醒我们：有时候，最大的突破不是来自新的数学公式，而是来自对硬件特性的深刻理解。

## References

1. Bahdanau, D., et al. (2014). "Neural Machine Translation by Jointly Learning to Align and Translate."
2. Dao, T., et al. (2022). "FlashAttention: Fast and Memory-Efficient Exact Attention with IO-Awareness."
3. Ainslie, J., et al. (2023). "GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints."
