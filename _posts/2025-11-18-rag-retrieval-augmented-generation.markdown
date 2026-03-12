---
layout:     post
title:      "RAG：检索增强生成全面指南"
subtitle:   "Retrieval-Augmented Generation — Bridging Knowledge Gaps in LLMs"
date:       2025-11-18 10:30:00
author:     "Tony L."
header-img: "img/post-bg-digital-native.jpg"
tags:
    - LLM
    - RAG
    - NLP
    - AI
---

## 什么是 RAG？

检索增强生成（Retrieval-Augmented Generation, RAG）是一种将信息检索与文本生成相结合的技术框架。它的核心思想很直观：与其让 LLM 完全依赖训练时记忆的知识来回答问题，不如在生成回答之前，先从外部知识库中检索相关文档，然后将检索到的信息作为上下文提供给 LLM。

这种方法有效解决了 LLM 的几个关键痛点：

1. **知识时效性**：LLM 的训练数据有截止日期，而 RAG 可以接入实时更新的知识库。
2. **幻觉问题**：LLM 容易生成看似合理但事实错误的内容，RAG 通过提供真实文档作为依据来降低幻觉率。
3. **领域专业性**：通用 LLM 在特定领域（法律、医疗、金融）的知识有限，RAG 可以接入专业知识库。
4. **可溯源性**：RAG 生成的回答可以追溯到具体的源文档，增强了可信度。

## RAG 的基本流程

一个标准的 RAG 系统包含以下步骤：

### 1. 文档预处理（Indexing）

```
原始文档 → 分块(Chunking) → 向量化(Embedding) → 存入向量数据库
```

**分块策略**是影响 RAG 效果的关键因素：
- **固定大小分块**：按字符数或 token 数分割，简单但可能切断语义。
- **语义分块**：基于句子或段落边界分割，保持语义完整性。
- **递归分块**：先按大分隔符（如标题）分，再对过长的块进行二次分割。
- **重叠分块**：相邻块之间保留一定重叠，避免边界信息丢失。

实践中，chunk size 通常在 256-1024 tokens 之间，overlap 为 chunk size 的 10-20%。

### 2. 查询与检索（Retrieval）

用户查询 → 向量化 → 在向量数据库中进行相似度搜索 → 返回 Top-K 相关文档

常用的相似度度量：
- **余弦相似度**：最常用，对向量长度不敏感
- **点积**：适合归一化后的向量
- **欧氏距离**：适合低维空间

### 3. 生成（Generation）

将检索到的文档与用户查询拼接成 prompt，送入 LLM 生成回答：

```
System: You are a helpful assistant. Use the following context to answer the question.
Context: [retrieved documents]
User: [original question]
```

## 进阶 RAG 技术

### Hybrid Search（混合搜索）

单纯的向量搜索有时不够精确，特别是对于包含专有名词或数字的查询。混合搜索结合了：
- **稀疏检索**（BM25）：基于关键词匹配，擅长精确匹配
- **稠密检索**（向量搜索）：基于语义相似度，擅长模糊匹配

通过加权融合两种检索结果，通常能获得更好的效果。

### Query Transformation（查询改写）

用户的原始查询可能不适合直接用于检索。常见的改写策略：

- **HyDE (Hypothetical Document Embedding)**：先让 LLM 生成一个假想的答案文档，再用这个假想文档去检索。
- **Multi-Query**：将一个查询拆分成多个子查询，分别检索后合并结果。
- **Step-Back Prompting**：将具体问题抽象为更通用的问题，以获取更广泛的背景知识。

### Re-ranking（重排序）

初次检索返回的 Top-K 结果可能包含噪声。使用 Cross-Encoder 对检索结果进行重排序，可以显著提升精度：

```python
from sentence_transformers import CrossEncoder
reranker = CrossEncoder('cross-encoder/ms-marco-MiniLM-L-6-v2')
scores = reranker.predict([(query, doc) for doc in retrieved_docs])
```

### Agentic RAG

最新趋势是将 RAG 与 AI Agent 结合：
- Agent 根据查询判断是否需要检索
- 动态选择检索源（不同知识库、Web 搜索等）
- 对检索结果进行质量评估，必要时重新检索
- 多步推理：先检索背景知识，再检索具体细节

## 向量数据库选型

| 数据库 | 特点 | 适用场景 |
|--------|------|----------|
| Pinecone | 全托管，易用 | 快速原型，中小规模 |
| Weaviate | 开源，混合搜索 | 需要 BM25+向量的场景 |
| Milvus | 高性能，分布式 | 大规模生产环境 |
| Chroma | 轻量，嵌入式 | 本地开发，小项目 |
| pgvector | PostgreSQL 扩展 | 已有 PG 基础设施的团队 |

## 评估 RAG 系统

评估 RAG 系统需要从多个维度考量：

1. **检索质量**：Precision@K, Recall@K, MRR (Mean Reciprocal Rank)
2. **生成质量**：RAGAS 框架提供了专门的 RAG 评估指标
   - Faithfulness：生成的回答是否忠于检索到的文档
   - Answer Relevancy：回答是否切题
   - Context Precision：检索到的文档中有多少是真正相关的
   - Context Recall：相关的文档是否都被检索到了

## 实践建议

1. **先做好分块**：分块质量直接决定了 RAG 系统的上限。花时间在分块策略上，比调 LLM prompt 更有效。
2. **选择合适的 Embedding 模型**：不要默认使用 OpenAI 的 embedding，根据你的语言和领域选择最合适的模型。中文场景下 BGE 系列表现优秀。
3. **建立评估流水线**：没有评估就没有优化。从项目初期就建立自动化评估体系。
4. **考虑成本**：向量数据库的存储和查询成本，以及 LLM 的 token 消耗，都需要纳入成本计算。

## 总结

RAG 不是银弹，但它是目前最实用的 LLM 知识增强方案。理解 RAG 的每个环节，并根据具体场景做出合适的技术选型，是构建可靠 AI 应用的关键。

## References

1. Lewis, P., et al. (2020). "Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks."
2. Gao, Y., et al. (2024). "Retrieval-Augmented Generation for Large Language Models: A Survey."
3. Es, S., et al. (2023). "RAGAS: Automated Evaluation of Retrieval Augmented Generation."
