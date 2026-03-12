---
layout:     post
title:      "AI Agents 与 Tool Use：LLM 的行动力革命"
subtitle:   "Building Autonomous AI Agents That Can Act in the Real World"
date:       2026-03-02 09:30:00
author:     "Tony L."
header-img: "img/post-bg-nextgen-web-pwa.jpg"
tags:
    - AI
    - Agent
    - LLM
    - Tool Use
---

## 从对话到行动

2025 年 AI 领域最重要的范式转变是：LLM 从"对话工具"进化为"行动者"（Agent）。一个 AI Agent 不仅能理解和生成文本，还能使用工具、执行代码、调用 API、操作计算机——甚至完成复杂的多步骤任务。

## 什么是 AI Agent？

一个 AI Agent 由以下核心组件构成：

```
┌─────────────────────────────────┐
│           AI Agent              │
│                                 │
│  ┌───────────┐  ┌───────────┐  │
│  │   LLM     │  │  Memory   │  │
│  │  (Brain)  │  │ (Context) │  │
│  └─────┬─────┘  └───────────┘  │
│        │                        │
│  ┌─────▼──────────────────┐    │
│  │   Planning & Reasoning  │    │
│  └─────┬──────────────────┘    │
│        │                        │
│  ┌─────▼──────────────────┐    │
│  │       Tool Use          │    │
│  │  [Search] [Code] [API]  │    │
│  └─────────────────────────┘    │
└─────────────────────────────────┘
```

- **LLM（大脑）**：负责理解任务、推理和决策
- **Memory（记忆）**：短期记忆（当前对话上下文）和长期记忆（持久化知识）
- **Planning（规划）**：将复杂任务分解为可执行的步骤
- **Tool Use（工具使用）**：调用外部工具来执行具体操作

## Tool Use 机制

Tool Use 是 Agent 能力的核心。现代 LLM（Claude, GPT-4, Gemini）都原生支持 function calling：

### 定义工具

```json
{
  "name": "search_web",
  "description": "Search the web for current information",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The search query"
      },
      "num_results": {
        "type": "integer",
        "default": 5
      }
    },
    "required": ["query"]
  }
}
```

### Agent 循环

```python
def agent_loop(user_message, tools, max_iterations=10):
    messages = [{"role": "user", "content": user_message}]

    for i in range(max_iterations):
        # 1. LLM 决策
        response = llm.chat(messages, tools=tools)

        # 2. 检查是否需要调用工具
        if response.tool_calls:
            for tool_call in response.tool_calls:
                # 3. 执行工具
                result = execute_tool(
                    tool_call.function.name,
                    tool_call.function.arguments
                )
                # 4. 将结果反馈给 LLM
                messages.append({
                    "role": "tool",
                    "content": result,
                    "tool_call_id": tool_call.id
                })
        else:
            # LLM 给出最终回答
            return response.content

    return "Max iterations reached"
```

## Agent 设计模式

### ReAct (Reasoning + Acting)

ReAct 让 Agent 在每一步都先进行推理（Thought），然后采取行动（Action），最后观察结果（Observation）：

```
User: What is the population of the capital of France?

Thought: I need to find the capital of France, then look up its population.
Action: search("capital of France")
Observation: The capital of France is Paris.

Thought: Now I know the capital is Paris. I need to find its population.
Action: search("population of Paris 2025")
Observation: The population of Paris is approximately 2.1 million.

Thought: I have the answer now.
Answer: The population of Paris, the capital of France, is approximately 2.1 million.
```

### Plan-and-Execute

先制定完整计划，再逐步执行：

```
User: Help me set up a new React project with TypeScript and testing.

Plan:
1. Create React project with Vite and TypeScript template
2. Install testing dependencies (Vitest, @testing-library/react)
3. Configure Vitest in vite.config.ts
4. Create a sample component with test
5. Verify tests pass

Executing Step 1...
Executing Step 2...
...
```

适合任务步骤明确、较少需要动态调整的场景。

### Multi-Agent

多个专业化 Agent 协作完成任务：

```
┌──────────┐     ┌───────────┐     ┌──────────┐
│ Planner  │────→│  Coder    │────→│ Reviewer │
│  Agent   │     │  Agent    │     │  Agent   │
└──────────┘     └───────────┘     └──────────┘
      ↑                                  │
      └──────────── feedback ────────────┘
```

## Coding Agents

Coding Agent 是目前最成熟的 AI Agent 类型：

### 能力范围
- 读取和理解代码库
- 编写和修改代码
- 运行测试和调试
- 执行 git 操作
- 管理依赖和配置

### 实际工作流

以 Claude Code 为例：

```
User: "Add a dark mode toggle to the settings page"

Agent:
1. [Read] Scan project structure and find settings page
2. [Read] Understand existing theme system
3. [Think] Plan the implementation approach
4. [Write] Create dark mode toggle component
5. [Write] Update theme store with dark mode state
6. [Write] Add CSS variables for dark theme
7. [Run] Execute tests to verify nothing broke
8. [Git] Commit changes with descriptive message
```

### 关键挑战

1. **上下文窗口限制**：大型代码库无法完全放入上下文
2. **代码正确性**：生成的代码可能有 bug
3. **理解意图**：用户描述可能模糊或不完整
4. **安全性**：Agent 有执行代码的能力，需要沙箱机制

## Computer Use Agents

更进一步，Agent 可以直接操作计算机界面：

- **屏幕理解**：通过视觉模型理解当前屏幕内容
- **鼠标/键盘操作**：点击按钮、输入文本、滚动页面
- **任务执行**：打开应用、填写表单、管理文件

这使得 Agent 可以操作任何没有 API 的软件系统。

## 构建可靠 Agent 的最佳实践

1. **工具设计**
   - 工具描述要清晰、具体
   - 参数类型要严格定义
   - 返回结果要结构化

2. **错误处理**
   - 工具调用可能失败，Agent 需要能处理错误
   - 设置最大重试次数和超时
   - 提供 fallback 策略

3. **安全约束**
   - 限制 Agent 能访问的资源
   - 敏感操作需要人类确认
   - 沙箱执行不信任的代码

4. **可观测性**
   - 记录每一步的决策和工具调用
   - 提供 Agent 思维过程的透明性
   - 支持人类在任意步骤介入

## 展望

AI Agent 正在从实验走向生产。未来的方向包括：

- **更长的自主运行时间**：从几分钟到几小时甚至几天
- **更广的工具生态**：MCP（Model Context Protocol）等标准化协议
- **更强的推理能力**：结合推理模型（o1 等）增强规划能力
- **多模态交互**：同时处理文本、图像、代码、终端输出

Agent 代表了 AI 应用的下一个 paradigm shift。我们正在从"AI 辅助人类"转向"AI 自主执行"，这将深刻改变软件开发和知识工作的本质。

## References

1. Yao, S., et al. (2023). "ReAct: Synergizing Reasoning and Acting in Language Models."
2. Anthropic. (2024). "Tool Use and Computer Use."
3. Wang, L., et al. (2024). "A Survey on Large Language Model based Autonomous Agents."
