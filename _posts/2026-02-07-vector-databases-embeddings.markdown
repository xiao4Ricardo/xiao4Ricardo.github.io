---
layout:     post
title:      "向量数据库与 Embeddings 技术全景"
subtitle:   "The Infrastructure Behind Semantic Search and RAG"
date:       2026-02-07 11:00:00
author:     "Tony L."
header-img: "img/post-bg-infinity.jpg"
tags:
    - AI
    - Vector Database
    - Embeddings
    - RAG
---

## 为什么向量数据库成为了 AI 基础设施？

在 LLM 时代，向量数据库（Vector Database）从一个小众技术跃升为 AI 应用栈的核心组件。无论是 RAG 系统、语义搜索、推荐引擎还是异常检测，向量数据库都扮演着关键角色。

核心原因：传统数据库基于精确匹配（SQL WHERE 子句），而 AI 应用需要**语义相似度匹配**——找到"意思相近"的内容，而不仅仅是"字面相同"的内容。

## Embeddings 基础

### 什么是 Embedding？

Embedding 是将高维离散数据（文本、图像、音频）映射为低维连续向量的技术。一个好的 embedding 应该保持语义关系：意义相近的内容在向量空间中距离相近。

```python
from sentence_transformers import SentenceTransformer

model = SentenceTransformer('all-MiniLM-L6-v2')

sentences = [
    "How to train a machine learning model",
    "Steps for building an ML pipeline",
    "Best Italian restaurants in New York",
]

embeddings = model.encode(sentences)
# 输出形状: (3, 384) — 每个句子被编码为 384 维向量
```

前两个句子的向量会非常接近（余弦相似度 > 0.8），而它们与第三个句子的向量距离较远。

### Embedding 模型选型

| 模型 | 维度 | 特点 |
|------|------|------|
| text-embedding-3-small (OpenAI) | 1536 | 高质量，收费 |
| text-embedding-3-large (OpenAI) | 3072 | 最高质量，成本高 |
| all-MiniLM-L6-v2 | 384 | 开源，轻量，速度快 |
| BGE-large-en-v1.5 | 1024 | 开源 SOTA |
| BGE-M3 | 1024 | 多语言，支持中英文 |
| jina-embeddings-v3 | 1024 | 多语言，支持 late interaction |
| Cohere embed-v3 | 1024 | 商用，质量高 |

选择建议：
- 英文场景：BGE-large 或 OpenAI text-embedding-3
- 中文场景：BGE-M3 或 Jina
- 低延迟场景：all-MiniLM-L6-v2
- 多模态场景：CLIP（文本 + 图像）

## 向量检索算法

### 暴力搜索 (Brute Force)

最简单的方法——计算查询向量与数据库中每个向量的相似度，返回 Top-K。

- 时间复杂度：O(n * d)
- 优点：100% 准确
- 缺点：数据量大时太慢

### IVF (Inverted File Index)

将向量空间划分为多个 Voronoi 区域（聚类），查询时只搜索最近的几个区域：

1. 训练阶段：对所有向量进行 K-Means 聚类
2. 查询阶段：找到查询向量最近的 nprobe 个聚类中心，只在这些聚类内搜索

```python
import faiss

# 创建 IVF 索引
nlist = 100  # 聚类数
quantizer = faiss.IndexFlatL2(dimension)
index = faiss.IndexIVFFlat(quantizer, dimension, nlist)
index.train(vectors)
index.add(vectors)

# 查询
index.nprobe = 10  # 搜索 10 个最近的聚类
distances, indices = index.search(query_vector, k=10)
```

### HNSW (Hierarchical Navigable Small World)

基于图的近似最近邻搜索，构建多层小世界图：

- 上层：稀疏的长距离连接（快速粗略定位）
- 下层：密集的短距离连接（精确搜索）

HNSW 的查询从最顶层开始，逐层下降，每层都贪心地移向最近的节点。

优势：
- 查询速度快（O(log n)）
- 召回率高（通常 > 95%）
- 支持增量插入

劣势：
- 内存占用大（需要存储图结构）
- 构建索引慢

### 量化 (Product Quantization)

将高维向量压缩为更紧凑的表示：

```
原始: 128 维 float32 → 512 bytes
PQ:   128 维 → 32 个子空间 × 8 bit = 32 bytes
压缩比: 16x
```

可以与 IVF 结合：IVF-PQ 是生产环境中最常用的索引类型之一。

## 向量数据库深度对比

### Pinecone

```python
import pinecone

pc = pinecone.Pinecone(api_key="xxx")
index = pc.Index("my-index")

# 插入
index.upsert(vectors=[
    {"id": "doc1", "values": embedding, "metadata": {"source": "wiki"}},
])

# 查询
results = index.query(vector=query_embedding, top_k=5,
                      include_metadata=True,
                      filter={"source": {"$eq": "wiki"}})
```

优点：全托管，零运维，metadata filtering
缺点：闭源，成本较高，供应商锁定

### Weaviate

```python
import weaviate

client = weaviate.connect_to_local()

# 混合搜索（向量 + 关键词）
response = collection.query.hybrid(
    query="machine learning optimization",
    alpha=0.75,  # 0=纯关键词, 1=纯向量
    limit=5
)
```

优点：开源，内置混合搜索，GraphQL API
缺点：资源消耗较大

### pgvector (PostgreSQL)

```sql
-- 创建扩展
CREATE EXTENSION vector;

-- 创建表
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    content TEXT,
    embedding vector(384)
);

-- 创建索引
CREATE INDEX ON documents
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);

-- 查询
SELECT content, embedding <=> '[0.1, 0.2, ...]' AS distance
FROM documents
ORDER BY distance
LIMIT 5;
```

优点：无需额外基础设施，SQL 生态兼容
缺点：大规模数据性能不如专用向量数据库

## 生产环境最佳实践

1. **选择合适的索引类型**
   - < 100K 向量：暴力搜索就够了
   - 100K - 10M：HNSW
   - > 10M：IVF-PQ + HNSW

2. **Embedding 缓存**
   - 相同文本不要重复计算 embedding
   - 使用 Redis 或本地缓存

3. **Metadata Filtering**
   - 先过滤再搜索，显著减少搜索空间
   - 合理设计 metadata schema

4. **监控指标**
   - 查询延迟 (P50, P95, P99)
   - 召回率
   - 索引大小和内存使用

## 总结

向量数据库是 AI 应用的"内存"。选择合适的 embedding 模型和向量数据库，设计高效的索引策略，是构建高质量 AI 应用的基础。对于大多数团队，我的建议是：如果你已经在用 PostgreSQL，从 pgvector 开始；如果需要更高性能或更丰富的功能，再考虑 Weaviate 或 Pinecone。

## References

1. Johnson, J., et al. (2019). "Billion-scale similarity search with GPUs." IEEE TBD.
2. Malkov, Y. & Yashunin, D. (2018). "Efficient and Robust Approximate Nearest Neighbor using Hierarchical Navigable Small World Graphs."
3. MTEB Leaderboard. https://huggingface.co/spaces/mteb/leaderboard
