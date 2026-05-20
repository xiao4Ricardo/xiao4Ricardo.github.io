---
layout:     post
title:      "放弃 Single-Prompt：为什么 Agentic Workflow 才是大模型落地的未来"
subtitle:   "Moving Beyond Single-Prompt: Why Agentic Workflows are the Future of LLM Applications"
date:       2026-04-01 10:00:00
author:     "Tony L."
header-img: "img/post-bg-digital-native.jpg"
tags:
    - AI
    - AI Agents
    - Software Architecture
    - LLM
---

## 引言

当大语言模型（LLM）初露锋芒时，全世界都在研究 **Prompt Engineering（提示词工程）**。我们试图写出长达几千字、结构复杂的“神级提示词”，试图让模型通过一次交互（Single-Prompting）就吐出完美的、毫无瑕疵的最终结果。

但实际落地过的工程师都知道，对于复杂任务，**Single-Prompt 几乎不可能稳定产出工业级可用的产品**。

AI 领域的宗师级人物 Andrew Ng（吴恩达）曾指出：“通过引入协作式的智能体工作流（Agentic Workflow），即便使用较弱的基础模型（如 GPT-3.5 级别），其产出效果也能超过使用最高端模型单次直接生成的效果。”

今天，我们结合代码与设计模式，聊聊从“写 Prompt”过渡到“建 Workflow”的范式转变。

---

## 1. 为什么 Single-Prompt 不行？

单一提示词的交互，本质上是**一次性的、开环控制的**生成：

```
Prompt ─────────────────────────► [ 大语言模型 ] ─────────────────────────► 最终结果
```

这带来三个硬伤：
* **无法自我修正**：如果模型在生成的第 10 行犯了一个常识性错误，它没有机会返回修改，只能硬着头皮沿着错误的方向生成后面的所有内容。
* **思考广度受限**：人类在撰写一份万字报告时，需要收集资料、罗列大纲、分段起草、审阅修改。你不可能在脑子里瞬间“渲染”出最终成文，而 Single-Prompt 就是在逼模型做这件事。
* **缺乏系统整合**：复杂的商业逻辑通常包含各种不确定性判定、条件分支和外部工具调用，这不是纯粹的文本补全能稳健解决的。

---

## 2. 什么是 Agentic Workflow？

**Agentic Workflow（智能体工作流）** 将大模型的使用方式从“一次输出”升级为“多次迭代、协作、带工具链的闭环系统”。

在架构上，它倡导引入四种核心的“代理人交互设计模式”：

```
                              ┌───────────────┐
                              │ 1. 反思与修正  │
                              └───────┬───────┘
                                      │
  ┌───────────────┐           ┌───────▼───────┐           ┌───────────────┐
  │  2. 工具使用   ├──────────►│    LLM 引擎   │◄──────────┤  3. 规划与拆解  │
  └───────────────┘           └───────▲───────┘           └───────────────┘
                                      │
                              ┌───────┴───────┐
                              │ 4. 多智能体协作│
                              └───────────────┘
```

### 1. Reflection（反思与自我修正）
这是一种最基础但也最有效的工作流。模型在生成完初代结果后，由另一个（或同个）模型充当“审查官”，指出其代码/文本的不足之处，并将其作为负反馈再次输入模型，进行增量修改。

### 2. Tool Use（工具使用）
不再指望模型依靠大脑记忆来回答实时问题。模型被赋予可以检索数据库、发起 Web 搜索、执行 Python 代码或调用计算器等外部工具的能力，通过数据反馈纠正生成路径。

### 3. Planning（规划与任务拆解）
对于庞大的目标，模型先不写具体内容，而是首先规划出子任务链（e.g., Sub-goals），然后分批步进式地执行每个子目标。

### 4. Multi-Agent Collaboration（多智能体协作）
让不同的 Agent 扮演不同的角色（例如：程序员、测试员、产品经理）。不同的 Agent 拥有不同的提示词系统和能力边界，在工作流中串联或并联工作，通过竞争或互补关系产生极佳的结果。

---

## 3. 实战范式对比：代码审查工作流的设计

让我们用一段简单的 Python 伪代码，对比这两种模式的实现差异。

### 方案 A：传统的 Single-Prompt 直出
```python
def generate_code_single_prompt(user_requirement):
    prompt = f"你是一个顶级的资深架构师。请直接为以下需求写出完美的 Rust 代码：{user_requirement}。"
    return llm.call(prompt)
```
* **缺点**：生成的代码可能带有编译错误、内存泄漏隐患，甚至完全理解错了需求。

### 方案 B：使用 Agentic Workflow (Reflection + Planning)
```python
class DeveloperAgent:
    def write_code(self, plan, requirement):
        prompt = f"根据步骤 {plan}，实现以下需求的核心 Rust 代码：{requirement}。"
        return llm.call(prompt)

class QA_Agent:
    def review_code(self, code):
        prompt = f"请严格审查以下 Rust 代码并指出可能存在的 3 个潜在 Bug：\n{code}"
        return llm.call(prompt)

def agentic_workflow_execution(requirement):
    # 1. Planning Phase (规划阶段)
    plan = llm.call(f"请为需求 '{requirement}' 规划两步走的实现大纲。")
    
    # 2. Writing Phase (代码编写)
    dev = DeveloperAgent()
    draft_code = dev.write_code(plan, requirement)
    
    # 3. Reflection Phase (反思与审查)
    qa = QA_Agent()
    review_feedback = qa.review_code(draft_code)
    
    # 4. Refinement Phase (增量修正)
    final_prompt = f"请结合审查意见：\n{review_feedback}\n对以下代码进行完善：\n{draft_code}"
    final_code = llm.call(final_prompt)
    
    return final_code
```

在方案 B 中，即便我们使用更便宜、响应更敏捷的轻量级模型，经过**规划、第一轮起草、QA自我审查、最终润色**这四个步骤，最终产出的 Rust 代码在健壮性和可用性上也将远远超越方案 A 中单次强行生成的产物。

---

## 结论

大模型时代应用落地不仅拼的是模型本身的能力，更拼的是**应用层架构的系统设计**。

如果你的大模型项目正卡在“生成准确率不够稳定”的瓶颈上，别再试图去绞尽脑汁修改提示词里的形容词了。果断放弃 **Single-Prompt**，为你的业务流程设计一个逻辑闭环的 **Agentic Workflow**，这才是让 AI 走向生产力的正确道路。
