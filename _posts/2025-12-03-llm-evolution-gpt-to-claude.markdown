---
layout:     post
title:      "大语言模型进化史：从 GPT-1 到 Claude"
subtitle:   "The Evolution of Large Language Models"
date:       2025-12-03 16:00:00
author:     "Tony L."
header-img: "img/post-bg-halting.jpg"
tags:
    - LLM
    - AI
    - GPT
    - Claude
---

## 引言

大语言模型（Large Language Models, LLMs）在短短几年内经历了爆发式的发展。从 2018 年 GPT-1 的 1.17 亿参数到如今千亿甚至万亿级参数的模型，LLM 的能力边界不断被刷新。本文梳理这段进化史中的关键节点。

## 第一阶段：预训练语言模型的崛起 (2018-2019)

### GPT-1 (2018)

OpenAI 发布的 GPT-1 使用了 12 层 Transformer Decoder，1.17 亿参数，在 BookCorpus 上进行无监督预训练。GPT-1 的突破性在于证明了"预训练 + 微调"范式的可行性——一个通用的语言模型可以通过少量的任务特定微调来适应各种下游任务。

### BERT (2018)

Google 的 BERT 采用了 Transformer Encoder 架构，通过双向上下文建模（Masked Language Model + Next Sentence Prediction），在 11 个 NLP 基准上取得了 SOTA。BERT 证明了双向注意力对于自然语言理解任务的重要性。

### GPT-2 (2019)

GPT-2 将参数量提升到 15 亿，训练数据扩展到 40GB 的 WebText。它展示了一个令人惊讶的现象：足够大的语言模型可以在 zero-shot 设置下完成多种任务，无需微调。OpenAI 最初因担心滥用风险而推迟了完整模型的发布，这也开启了 AI 安全讨论的先河。

## 第二阶段：Scaling Law 验证 (2020-2022)

### GPT-3 (2020)

GPT-3 是一个标志性的里程碑。1750 亿参数，在 570GB 的文本数据上训练。它展示了 in-context learning 的能力：仅通过在 prompt 中给出几个示例（few-shot），模型就能完成翻译、问答、代码生成等任务，无需更新任何参数。

GPT-3 验证了 Scaling Law——模型性能随参数量、数据量和计算量的增加呈幂律提升。

### Chinchilla (2022)

DeepMind 的 Chinchilla 论文对 Scaling Law 进行了修正：之前的模型大多是"过大而欠训练"的（undertrained）。Chinchilla 表明，700 亿参数的模型如果用更多数据训练，可以超越 2800 亿参数的 Gopher。这一发现深刻影响了后续模型的训练策略。

### InstructGPT (2022)

InstructGPT 引入了 RLHF（Reinforcement Learning from Human Feedback），通过人类反馈来对齐模型行为。这标志着从"预测下一个 token"到"遵循人类指令"的范式转变。

## 第三阶段：ChatGPT 时刻与军备竞赛 (2022-2024)

### ChatGPT (2022.11)

ChatGPT 基于 GPT-3.5（GPT-3 + 代码训练 + RLHF）构建，它的发布彻底改变了公众对 AI 的认知。两个月内用户突破一亿，创下历史记录。

### GPT-4 (2023.3)

GPT-4 是首个大规模多模态 LLM，同时处理文本和图像输入。在几乎所有专业考试中达到了人类 top 10% 的水平。它的推理能力、长上下文处理和指令遵循能力都有了质的飞跃。

### Claude 系列 (2023-2025)

Anthropic 的 Claude 系列走了一条独特的路线，将安全性（Constitutional AI）作为核心设计原则：

- **Claude 1** (2023)：引入 Constitutional AI，通过一套"宪法"原则来指导模型行为
- **Claude 2** (2023)：支持 100K context window，大幅提升长文本处理能力
- **Claude 3 系列** (2024)：推出 Haiku/Sonnet/Opus 三个档位，Opus 在多项基准上超越 GPT-4
- **Claude 3.5 Sonnet** (2024)：在保持高质量的同时大幅降低延迟和成本
- **Claude 4 系列** (2025)：进一步提升推理和编程能力

Claude 的特点是在推理能力和安全性之间取得了良好的平衡，特别是在长文本理解、代码生成和复杂指令遵循方面表现出色。

### 开源阵营

开源社区同样百花齐放：
- **LLaMA / LLaMA 2 / LLaMA 3** (Meta)：推动了开源 LLM 生态的繁荣
- **Mistral / Mixtral** (Mistral AI)：以小模型实现大模型效果，Mixtral 引入了 MoE 架构
- **Qwen 2.5** (Alibaba)：在中文和多语言任务中表现优异
- **DeepSeek** (DeepSeek)：在数学推理和代码能力上取得突破

## 第四阶段：推理与 Agent (2024-2025)

### o1 和推理模型

2024 年底，OpenAI 发布了 o1 系列，这些模型被训练来在回答之前进行深入的"思考"。通过 Chain-of-Thought 推理，o1 在数学、编程和科学推理任务上取得了惊人的提升。

这开启了"推理模型"（Reasoning Models）的新赛道：
- DeepSeek-R1：开源推理模型的代表
- Claude 的 extended thinking：在保持对话流畅性的同时增强推理能力

### AI Agent

2025 年的主流趋势是 AI Agent——能够自主使用工具、执行多步骤任务的 AI 系统。从 Claude 的 Computer Use 到各种 Coding Agent，LLM 正在从"对话工具"进化为"行动者"。

## 技术趋势总结

| 趋势 | 描述 |
|------|------|
| Scaling → Efficiency | 从单纯堆参数转向更高效的训练和推理 |
| 单任务 → 通用能力 | 从 fine-tuning 到 in-context learning |
| 文本 → 多模态 | 支持文本、图像、音频、视频 |
| 对话 → Agent | 从被动回答到主动执行任务 |
| 闭源 → 开源 | 开源模型快速追赶闭源模型 |

## 展望

LLM 的发展远未到达终点。更高效的架构、更好的对齐方法、更强的推理能力、更安全的部署方式——这些方向都蕴含着巨大的研究和工程机会。作为工程师，我们正处在一个前所未有的激动人心的时代。

## References

1. Radford, A., et al. (2018). "Improving Language Understanding by Generative Pre-Training."
2. Brown, T., et al. (2020). "Language Models are Few-Shot Learners."
3. Ouyang, L., et al. (2022). "Training language models to follow instructions with human feedback."
4. Anthropic. (2024). "The Claude Model Family."
