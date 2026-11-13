---
title: "Agent 设计：Workflow 还是 Agent"
date: "2026-07-01"
tags: ["LLM", "Agents"]
---

做 LLM 功能时要先回答一个问题：这件事需要的是 agent，还是一个自动化的 workflow？

## 1. 先判断是不是真的需要 agent

- 很多人说要做 agent，其实要的是**自动化 workflow**：步骤固定，只是其中几步交给模型做。
- Workflow 的路径由代码决定，agent 的路径由模型决定。路径能提前写出来的，就写成 workflow。它更便宜、更快，也更容易测试和排查。
- 只有步骤数量和顺序事先不知道、要根据中间结果决定下一步时，才值得用 agent。
- 即使用 agent，也尽量把大任务拆成子任务，每个子任务的输入和输出要清楚。

> [!TIP]
> 路径能提前写出来，就写成 workflow。

## 2. 常见的 workflow 模式

| 模式 | 什么时候用 |
|---|---|
| Prompt chaining | 任务能拆成固定的几步，上一步的输出是下一步的输入 |
| Routing | 先分类，再交给专门的处理流程 |
| Parallelization | 多个独立子任务并行跑，或者同一任务跑多次再投票 |
| Orchestrator-workers | 由一个模型动态拆分任务，分给多个 worker |
| Evaluator-optimizer | 一个模型生成，另一个模型评审，循环改进 |

回头看现有代码，多半能找到这些模式的影子。先认出来，再决定要不要加更多自主性。

## 3. 衡量可靠性：pass@k 与 pass^k

- **pass@k**：跑 k 次，**至少成功一次**的概率。适合可以重试、有人挑选结果的场景，比如生成候选方案。
- **pass^k**：跑 k 次，**每次都成功**的概率。适合直接面向用户、每次都必须对的场景。

同一个模型，k 变大时 pass@k 上升，pass^k 下降。面向用户的 agent 应该看 pass^k。

> [!IMPORTANT]
> 单次成功率 90% 看起来不错，但 pass^5 只有约 59%（0.9⁵ ≈ 0.59）。

## 4. 稳定有效的 prompt 技巧

跨模型、跨任务都比较稳定的几个方法：

- **n-shot 示例 / in-context learning**：给几个输入输出的例子。
- **Chain-of-thought**：让模型先推理再给结论。如果要求结构化输出，推理字段要排在结论字段前面，否则模型会先给出结论，推理就失去了作用。
- **提供相关资料**：把需要的上下文直接放进 prompt，而不是指望模型记得。

## 5. 延伸阅读

- Anthropic, [*Building Effective Agents*](https://www.anthropic.com/engineering/building-effective-agents)
- Phil Schmid, [*Pass@k vs Pass^k: Understanding Agent Reliability*](https://www.philschmid.de/agents-pass-at-k-pass-power-k)
- [*What We've Learned From A Year of Building with LLMs*](https://applied-llms.org/)
- Claude Cookbook，[*Async multi-agent orchestration*](https://platform.claude.com/cookbook/patterns-agents-async-multi-agent-orchestration)
