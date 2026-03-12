---
layout:     post
title:      "Prompt Engineering 工程师实用指南"
subtitle:   "Systematic Approaches to Effective Prompting"
date:       2026-01-06 10:00:00
author:     "Tony L."
header-img: "img/post-bg-web.jpg"
tags:
    - LLM
    - Prompt Engineering
    - AI
---

## 前言

Prompt Engineering 不是玄学，而是一门可以系统化学习的技术。随着 LLM 能力的增强，精心设计的 prompt 与随意编写的 prompt 之间的效果差距越来越大。本文整理了我在实际工程中验证有效的 prompting 技术。

## 基础原则

### 1. 清晰具体

差的 prompt：
```
帮我写个总结
```

好的 prompt：
```
请用 3-5 个要点总结以下技术文档的核心观点。
每个要点不超过 30 字。使用中文。
重点关注架构决策和性能优化部分。
```

### 2. 提供上下文

LLM 不了解你的项目背景。提供必要的上下文信息：

```
我正在开发一个 Vue 3 + Spring Boot 的 CRM 系统。
前端使用 Composition API + TypeScript，后端使用 JPA + PostgreSQL。
现在需要为"交易管道"模块添加拖拽排序功能。
请帮我设计 API 接口和前端交互方案。
```

### 3. 指定输出格式

```
请以 JSON 格式返回结果，包含以下字段：
- summary: 一句话总结
- key_points: 要点数组
- action_items: 待办事项数组
```

## 进阶技术

### Chain-of-Thought (CoT)

让模型"先思考，再回答"，在复杂推理任务中效果显著。

```
Q: 一个商店有 23 个苹果。如果他们用 20 个做苹果派，
又买了 6 个，他们现在有多少个苹果？

让我们一步一步思考：
1. 开始有 23 个苹果
2. 用了 20 个做苹果派：23 - 20 = 3
3. 又买了 6 个：3 + 6 = 9
答案：9 个苹果
```

Zero-Shot CoT 只需要在 prompt 末尾加上 "Let's think step by step"，就能触发模型的推理链。

### Few-Shot Learning

通过示例教会模型你期望的行为模式：

```
将以下句子分类为"积极"或"消极"：

句子：这部电影太精彩了！
分类：积极

句子：等了一个小时还没上菜。
分类：消极

句子：今天的天气很适合散步。
分类：
```

Few-Shot 的关键技巧：
- 示例数量：3-5 个通常足够
- 示例多样性：覆盖不同的边界情况
- 示例顺序：最相关的示例放在最后（recency bias）

### Role Prompting

给模型设定一个角色，引导它的行为模式：

```
你是一位有 15 年经验的后端架构师，专长是高并发系统设计。
你回答问题时会：
1. 先分析需求和约束条件
2. 提出 2-3 种可选方案并比较优劣
3. 给出明确的推荐和理由
4. 指出潜在的风险点
```

### Self-Consistency

对于复杂推理问题，让模型生成多个独立的推理路径，然后取多数答案：

```
请用 3 种不同的方法解决这个问题，然后比较它们的结果：
[问题描述]
```

### Structured Output

对于需要结构化输出的场景，明确定义 schema：

```
分析以下代码的问题，按以下格式输出：

## 问题列表
| 序号 | 严重程度 | 文件:行号 | 问题描述 | 修复建议 |
|------|----------|-----------|----------|----------|
| 1    | HIGH     | ...       | ...      | ...      |
```

## 系统提示词设计

System Prompt 是影响 LLM 行为最重要的杠杆：

```
## Role
你是 SwiftCrew CRM 的 AI 编程助手。

## Context
- 技术栈：Vue 3 + TypeScript + Spring Boot 3 + PostgreSQL
- 架构：RESTful API，JWT 认证，Pinia 状态管理
- 代码规范：参考项目 CLAUDE.md

## Rules
1. 所有 Vue 组件使用 <script setup lang="ts">
2. API 调用通过 src/api/ 模块，不在组件中直接使用 axios
3. 使用 Tailwind CSS，不使用内联样式
4. 后端严格遵循 Controller → Service → Repository 分层
5. 所有实体继承 BaseEntity

## Output Format
- 代码使用 markdown 代码块，标注语言
- 先解释思路，再给出代码
- 如果修改现有文件，标注文件路径和修改位置
```

## 常见陷阱

### 1. Prompt 过长
Token 越多，成本越高，关键信息也可能被"淹没"。保持 prompt 精简。

### 2. 指令冲突
避免在同一个 prompt 中给出矛盾的要求。

### 3. 过度约束
给模型留出合理的发挥空间，过多的约束反而会降低输出质量。

### 4. 忽略温度参数
- 创意任务：temperature 0.7-1.0
- 精确任务（代码、数据提取）：temperature 0-0.3
- 分类任务：temperature 0

## 评估 Prompt 效果

建立系统化的评估流程：

1. **构建测试集**：收集 50-100 个代表性输入
2. **定义评估指标**：准确率、格式正确率、相关性评分
3. **A/B 测试**：对比不同 prompt 版本的效果
4. **版本管理**：将 prompt 纳入版本控制（Git），记录变更历史

## 总结

Prompt Engineering 的核心是**清晰表达意图**和**提供足够的上下文**。没有"万能 prompt"，每个场景都需要根据具体需求来设计和迭代。最好的 prompt 是你通过系统化测试验证过效果的那一个。

## References

1. Wei, J., et al. (2022). "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models."
2. Wang, X., et al. (2022). "Self-Consistency Improves Chain of Thought Reasoning in Language Models."
3. Anthropic. (2024). "Prompt Engineering Guide."
