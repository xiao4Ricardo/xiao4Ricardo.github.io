---
layout:     post
title:      "ML 系统设计：从实验到生产"
subtitle:   "Designing Machine Learning Systems for Production"
date:       2026-02-15 13:00:00
author:     "Tony L."
header-img: "img/post-bg-2015.jpg"
tags:
    - Machine Learning
    - System Design
    - MLOps
    - Software Engineering
---

## 引言

机器学习项目的失败率出奇地高——据估计，只有约 20% 的 ML 项目能成功部署到生产环境。问题往往不在模型本身，而在系统设计。本文探讨如何设计可靠、可扩展的 ML 系统。

## ML 系统 vs 传统软件系统

传统软件系统的行为由代码决定，是确定性的。ML 系统的行为由代码、数据和模型共同决定，是概率性的。这带来了独特的挑战：

| 维度 | 传统软件 | ML 系统 |
|------|----------|---------|
| 行为定义 | 代码逻辑 | 数据 + 模型 |
| 测试 | 单元测试、集成测试 | + 数据测试、模型测试 |
| 调试 | Stack trace | + 数据分析、模型分析 |
| 版本管理 | 代码版本 | + 数据版本、模型版本 |
| 退化 | Bug 引入 | + 数据漂移、概念漂移 |

## ML 系统架构

一个完整的 ML 系统包含以下组件：

```
┌─────────────────────────────────────────────┐
│                 ML Platform                  │
│                                             │
│  ┌─────────┐  ┌──────────┐  ┌───────────┐ │
│  │  Data    │  │  Model   │  │  Serving  │ │
│  │ Pipeline │→ │ Training │→ │  Layer    │ │
│  └─────────┘  └──────────┘  └───────────┘ │
│       ↑                           ↓        │
│  ┌─────────┐              ┌───────────┐   │
│  │ Feature │              │ Monitoring│   │
│  │  Store  │              │ & Alerts  │   │
│  └─────────┘              └───────────┘   │
│                                             │
│  ┌─────────────────────────────────────┐   │
│  │    Experiment Tracking & Registry    │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### 1. 数据管道 (Data Pipeline)

数据管道负责数据的收集、清洗、转换和存储。

**设计原则：**

- **幂等性**：相同输入始终产生相同输出，支持安全重跑
- **可回溯性**：能够追溯到任意时间点的数据状态
- **数据验证**：在管道的每个阶段检查数据质量

```python
# 使用 Great Expectations 进行数据验证
import great_expectations as gx

context = gx.get_context()

# 定义数据期望
expectation_suite = context.add_expectation_suite("training_data")
expectation_suite.add_expectation(
    gx.expectations.ExpectColumnValuesToNotBeNull(column="user_id")
)
expectation_suite.add_expectation(
    gx.expectations.ExpectColumnValuesToBeBetween(
        column="price", min_value=0, max_value=1000000
    )
)
```

### 2. 特征工程 (Feature Store)

Feature Store 是 ML 系统中常被忽视但至关重要的组件：

**离线特征**：用于模型训练，从历史数据中计算
```sql
-- 用户过去 30 天的平均订单金额
SELECT user_id,
       AVG(order_amount) as avg_order_30d,
       COUNT(*) as order_count_30d
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY user_id
```

**在线特征**：用于实时推理，需要低延迟
```python
# 从 Redis 中获取实时特征
features = feature_store.get_online_features(
    entity_rows=[{"user_id": "12345"}],
    feature_refs=["user_features:avg_order_30d",
                  "user_features:order_count_30d"]
)
```

### 3. 模型训练

训练管道应该是**可复现**的：

```python
# 使用 MLflow 追踪实验
import mlflow

with mlflow.start_run():
    mlflow.log_params({
        "model_type": "xgboost",
        "learning_rate": 0.1,
        "max_depth": 6,
        "n_estimators": 100,
    })

    model = train_model(X_train, y_train)
    metrics = evaluate_model(model, X_test, y_test)

    mlflow.log_metrics(metrics)
    mlflow.sklearn.log_model(model, "model")
```

### 4. 模型服务 (Serving)

模型部署有多种模式：

**批处理推理**：定期对大量数据进行预测
```python
# 每天凌晨运行
predictions = model.predict(all_users_features)
save_to_database(predictions)
```

**在线推理**：实时响应单个请求
```python
@app.post("/predict")
async def predict(request: PredictRequest):
    features = feature_store.get_online_features(request.user_id)
    prediction = model.predict(features)
    return {"prediction": prediction, "confidence": confidence}
```

**边缘推理**：在设备端运行模型（移动端、IoT）

### 5. 监控

ML 系统需要监控的不仅是系统指标（延迟、吞吐量），还有模型指标。

**数据漂移检测**：
```python
from evidently import ColumnDriftMetric
from evidently.report import Report

report = Report(metrics=[
    ColumnDriftMetric(column_name="feature_1"),
    ColumnDriftMetric(column_name="feature_2"),
])
report.run(reference_data=training_data,
           current_data=production_data)
```

**模型性能监控**：
- 预测分布是否偏移
- 在线指标（点击率、转化率）是否下降
- 推理延迟是否异常

## 常见设计模式

### 模型 A/B 测试

```
             ┌─── Model A (90% traffic) ──→ Response
User Request ─┤
             └─── Model B (10% traffic) ──→ Response
```

关键点：
- 使用一致性哈希确保同一用户始终看到同一模型
- 收集足够的数据（统计显著性）再做决策
- 监控业务指标而不仅仅是模型指标

### Shadow Mode（影子模式）

新模型在后台运行，接收所有请求但不返回给用户：

```
User Request → Model A (production) → Response to User
            → Model B (shadow)     → Log for comparison
```

这是最安全的新模型验证方式。

### Champion-Challenger

生产环境中始终有一个"冠军模型"和一个或多个"挑战者模型"。当挑战者在 A/B 测试中胜出时，自动升级为冠军。

## 可扩展性考虑

### 训练可扩展性

- **数据并行**：多个 GPU 各持有模型副本，处理不同数据
- **模型并行**：模型太大放不进单 GPU，分割到多个 GPU
- **梯度累积**：在小 GPU 上模拟大 batch size

### 推理可扩展性

- **水平扩展**：多个模型副本 + 负载均衡
- **模型优化**：量化、剪枝、知识蒸馏
- **批处理**：将多个请求合并为一个 batch
- **缓存**：对相同/相似输入缓存预测结果

## 总结

ML 系统设计的核心挑战不是选择什么模型，而是如何将模型可靠地集成到生产系统中。数据管道的健壮性、特征存储的一致性、模型部署的灵活性、监控告警的及时性——这些"无聊"的工程问题决定了 ML 项目的成败。

## References

1. Huyen, C. (2022). "Designing Machine Learning Systems." O'Reilly.
2. Sculley, D., et al. (2015). "Hidden Technical Debt in Machine Learning Systems." NeurIPS.
3. Google. "MLOps: Continuous delivery and automation pipelines in machine learning."
