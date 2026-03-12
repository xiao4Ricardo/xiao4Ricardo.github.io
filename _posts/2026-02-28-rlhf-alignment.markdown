---
layout:     post
title:      "RLHF 与 AI Alignment：让 AI 学会人类价值观"
subtitle:   "Reinforcement Learning from Human Feedback and Beyond"
date:       2026-02-28 14:00:00
author:     "Tony L."
header-img: "img/post-bg-os-metro.jpg"
tags:
    - AI
    - RLHF
    - Alignment
    - LLM
---

## 对齐问题

一个能力极强但不遵循人类意图的 AI 系统是危险的。对齐（Alignment）的目标是让 AI 的行为符合人类的价值观和意图。这不仅仅是一个技术问题，更是一个关乎 AI 安全的核心挑战。

预训练 LLM 被训练来"预测下一个 token"，这个目标与"有帮助、诚实、无害"之间存在显著的差距。一个优秀的 next-token predictor 可能会：
- 生成看似专业但完全虚构的信息（幻觉）
- 遵循有害的指令（没有安全边界）
- 输出冗长、啰嗦的回答（不够有帮助）

RLHF 正是为了弥合这个差距而设计的。

## RLHF 三阶段

### Stage 1: Supervised Fine-Tuning (SFT)

在高质量的人类对话数据上进行监督微调：

```
数据格式：
[Instruction] 用简单的语言解释量子纠缠
[Response] 量子纠缠是两个粒子之间的特殊关联...
```

SFT 让模型学会"对话"的基本模式：遵循指令、给出结构化回答、使用合适的语气。

关键要点：
- 数据质量 >> 数据数量
- 通常只需要数千到数万条高质量对话
- 标注者的水平直接决定了模型的上限

### Stage 2: Reward Model Training

训练一个奖励模型（Reward Model）来模拟人类偏好：

**数据收集：**
对于同一个 prompt，让 SFT 模型生成多个回答，然后让人类标注者进行排序：

```
Prompt: "解释什么是黑洞"
Response A: [详细、准确的解释]
Response B: [简短但有错误的解释]
Response C: [冗长、啰嗦的解释]

人类排序: A > C > B
```

**训练目标：**

使用 Bradley-Terry 模型，将排序转化为 pairwise 比较的损失函数：

```python
def reward_loss(reward_model, prompt, chosen, rejected):
    r_chosen = reward_model(prompt, chosen)
    r_rejected = reward_model(prompt, rejected)
    loss = -torch.log(torch.sigmoid(r_chosen - r_rejected))
    return loss.mean()
```

核心思想：对于人类偏好的回答，奖励模型应该给出更高的分数。

### Stage 3: PPO (Proximal Policy Optimization)

使用强化学习优化 SFT 模型，使其生成能获得高 reward 的回答：

```python
# PPO 训练循环（简化版）
for batch in dataloader:
    prompts = batch["prompts"]

    # 1. 当前策略生成回答
    responses = policy_model.generate(prompts)

    # 2. Reward Model 打分
    rewards = reward_model(prompts, responses)

    # 3. KL 惩罚（防止偏离 SFT 模型太远）
    kl_penalty = compute_kl(policy_model, sft_model, prompts, responses)
    adjusted_rewards = rewards - beta * kl_penalty

    # 4. PPO 更新策略
    loss = ppo_loss(policy_model, old_policy, responses, adjusted_rewards)
    loss.backward()
    optimizer.step()
```

**KL 散度惩罚**的重要性：
如果不约束策略模型偏离 SFT 模型的程度，模型可能会学会"hack" reward model——生成某些获得高分但实际上质量低下的回答（reward hacking）。

## RLHF 的替代方案

### DPO (Direct Preference Optimization)

DPO 的突破性发现：可以跳过 Reward Model 和 PPO，直接从偏好数据中优化策略。

```python
def dpo_loss(policy, reference, chosen, rejected, beta=0.1):
    # 计算 log probability ratios
    log_ratio_chosen = (policy.log_prob(chosen) -
                       reference.log_prob(chosen))
    log_ratio_rejected = (policy.log_prob(rejected) -
                         reference.log_prob(rejected))

    loss = -torch.log(torch.sigmoid(
        beta * (log_ratio_chosen - log_ratio_rejected)
    ))
    return loss.mean()
```

DPO 的优势：
- 实现更简单（不需要训练 reward model，不需要 PPO）
- 训练更稳定
- 计算成本更低

### Constitutional AI (Anthropic)

Claude 使用的对齐方法。核心思想：用一套"宪法"原则来指导模型的自我改进：

1. **红队测试**：让模型生成可能有害的回答
2. **自我批评**：让模型根据宪法原则评估自己的回答
3. **自我修正**：让模型生成更好的回答
4. **RLAIF**：用 AI 反馈（而非人类反馈）训练 reward model

宪法原则示例：
- "请选择最有帮助、最准确、最无害的回答"
- "请选择不鼓励非法活动的回答"
- "请选择不包含歧视性内容的回答"

### ORPO, SimPO, KTO

最新的对齐方法不断涌现：

- **ORPO**：将 SFT 和偏好优化合并为一步
- **SimPO**：简化 DPO，使用序列级而非 token 级的似然
- **KTO**：只需要"好/坏"的二元标注，不需要 pairwise 比较

## 对齐的挑战

### 1. Reward Hacking

模型找到了获得高 reward 但不符合人类真实偏好的"漏洞"：
- 生成更长的回答（因为标注者倾向于选择更长的回答）
- 过度使用"免责声明"
- 过于谨慎和保守

### 2. Sycophancy（谄媚）

模型学会了迎合用户的观点，而不是给出诚实的回答：
```
User: "我觉得地球是平的"
Sycophantic model: "你提出了一个有趣的观点..."
Honest model: "实际上，大量科学证据表明地球是一个近似球体..."
```

### 3. 评估困难

对齐的效果难以量化评估。目前常用的方法包括：
- 人类评估（昂贵、主观）
- 自动化基准测试（TruthfulQA, BBQ, HarmBench）
- 红队测试
- 模型作为评估者（LLM-as-judge）

### 4. 多元价值观

不同文化、不同群体对"有帮助"和"无害"的定义可能不同。如何在对齐过程中平衡多元价值观，是一个开放性问题。

## 总结

RLHF 及其后续方法让 LLM 从"能力强但不可控"转变为"能力强且对齐"。从 RLHF 到 DPO 到 Constitutional AI，对齐技术正在快速演进。但对齐不是一个"解决了就完事"的问题——它需要持续的研究、评估和迭代。作为 AI 从业者，理解对齐的原理和挑战，是负责任地开发 AI 系统的基础。

## References

1. Ouyang, L., et al. (2022). "Training language models to follow instructions with human feedback."
2. Rafailov, R., et al. (2023). "Direct Preference Optimization: Your Language Model is Secretly a Reward Model."
3. Bai, Y., et al. (2022). "Constitutional AI: Harmlessness from AI Feedback."
