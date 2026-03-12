---
layout:     post
title:      "微服务 vs 单体架构：工程决策指南"
subtitle:   "When to Choose Microservices and When to Stay Monolithic"
date:       2025-12-20 15:00:00
author:     "Tony L."
header-img: "img/post-bg-e2e-ux.jpg"
tags:
    - Software Engineering
    - Microservices
    - Architecture
---

## 前言

"微服务"一度成为技术圈的万能药。仿佛只要把单体拆成微服务，所有架构问题就迎刃而解。但现实远比这复杂。Amazon 的 Prime Video 团队在 2023 年将微服务合并回单体，节省了 90% 的成本。这个案例给了我们一个清醒的信号：架构选择必须基于实际需求，而非技术潮流。

## 单体架构的优势

单体架构（Monolith）将所有功能打包在一个部署单元中。它的优势常常被低估：

**1. 开发效率高**
- 一个代码库，一个 IDE 项目，一套调试工具
- 函数调用替代网络通信，没有序列化/反序列化开销
- 重构方便——IDE 的"重命名"功能就能搞定跨模块修改

**2. 运维简单**
- 一个应用部署、一个日志文件、一个监控面板
- 不需要服务发现、负载均衡、分布式追踪等基础设施
- 本地开发环境与生产环境差距小

**3. 数据一致性容易保证**
- 单一数据库，ACID 事务开箱即用
- 不需要 Saga 模式或分布式事务
- JOIN 查询简单直接

**4. 性能好**
- 进程内函数调用延迟 < 1ms
- 网络调用延迟 > 1ms（通常 5-50ms）
- 没有序列化开销

## 微服务的真正价值

微服务不是关于技术，而是关于**组织**。Conway's Law 告诉我们：系统架构反映组织沟通结构。微服务的核心价值在于：

**1. 独立部署**
- 团队可以独立发布，不需要协调其他团队
- 一个服务的故障不影响其他服务（理想情况下）
- 可以针对单个服务进行扩缩容

**2. 技术多样性**
- 不同服务可以使用不同语言、框架、数据库
- CPU 密集型服务用 Go/Rust，ML 服务用 Python，业务服务用 Java

**3. 团队自治**
- 每个团队拥有自己的服务，全权负责开发、测试、部署、运维
- 减少跨团队协调成本
- 团队规模保持在"两个披萨"的范围

## 微服务的隐藏成本

很多团队在拥抱微服务后才意识到这些成本：

### 分布式系统复杂性
- **网络不可靠**：超时、重试、幂等性、断路器
- **数据一致性**：分布式事务、Saga 模式、最终一致性
- **调试困难**：一个请求经过 10 个服务，哪里出了问题？

### 基础设施成本
- 服务发现（Consul, Eureka）
- API 网关（Kong, Nginx）
- 分布式追踪（Jaeger, Zipkin）
- 集中日志（ELK Stack）
- 容器编排（Kubernetes）
- CI/CD 流水线（每个服务一套）

### 开发体验下降
- 本地开发需要启动多个服务（或使用 Docker Compose）
- 集成测试变得复杂
- 跨服务 debug 困难
- API 版本管理

## 决策框架

| 因素 | 选单体 | 选微服务 |
|------|--------|----------|
| 团队规模 | < 20 人 | > 50 人，多个独立团队 |
| 部署频率 | 每周/每两周 | 每天多次，不同模块节奏不同 |
| 扩展需求 | 整体扩展即可 | 不同模块扩展需求差异大 |
| 技术栈 | 统一技术栈 | 需要多种语言/框架 |
| 项目阶段 | 早期/MVP | 成熟产品，边界清晰 |
| 运维能力 | 基础 | 有 DevOps/SRE 团队 |

## 最佳实践：Modular Monolith

一种折中方案是**模块化单体**（Modular Monolith）：

```
monolith/
├── modules/
│   ├── user/
│   │   ├── UserController.java
│   │   ├── UserService.java
│   │   └── UserRepository.java
│   ├── order/
│   │   ├── OrderController.java
│   │   ├── OrderService.java
│   │   └── OrderRepository.java
│   └── payment/
│       ├── PaymentController.java
│       ├── PaymentService.java
│       └── PaymentRepository.java
└── shared/
    ├── BaseEntity.java
    └── ApiResponse.java
```

关键原则：
- 模块之间通过明确定义的接口通信（不直接访问对方的数据库表）
- 每个模块有自己的领域模型和数据库表
- 模块边界严格执行（可以通过 ArchUnit 等工具自动化检查）
- 当某个模块真正需要独立部署时，可以相对容易地拆分出去

## 我的建议

作为一个行动精简主义者，我的建议是：

1. **从单体开始**。除非你有充分的理由，否则不要从微服务开始。
2. **保持模块化**。在单体内部维持清晰的模块边界，为未来可能的拆分做好准备。
3. **按需拆分**。当某个模块确实需要独立扩展、独立部署或使用不同技术栈时，再将它拆分为微服务。
4. **投资可观测性**。无论选择哪种架构，日志、监控和追踪都是必需品。

**架构选择不是信仰，而是工程权衡。**

## References

1. Newman, S. (2021). "Building Microservices", 2nd Edition. O'Reilly.
2. Amazon Prime Video Tech Blog. (2023). "Scaling up the Prime Video audio/video monitoring service and reducing costs by 90%."
3. Fowler, M. (2015). "MonolithFirst." martinfowler.com.
