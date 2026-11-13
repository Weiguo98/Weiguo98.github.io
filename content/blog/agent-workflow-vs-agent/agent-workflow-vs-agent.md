---
title: "Agent Design: Workflow or Agent?"
date: "2026-07-01"
tags: ["LLM", "Agents"]
---

Before building an LLM feature, answer one question first: does this need an agent, or an automated workflow?

## 1. Check whether you really need an agent

- When people say they want an agent, they often want an **automated workflow**: the steps are fixed, and only some of them are handed to a model.
- In a workflow, code decides the path. In an agent, the model decides the path. If you can write the path down in advance, build a workflow. It is cheaper, faster, and easier to test and debug.
- An agent is only worth it when you don't know the number or order of steps up front, and the next step depends on intermediate results.
- Even with an agent, split big tasks into subtasks, each with clear inputs and outputs.

> [!TIP]
> If you can write the path down in advance, make it a workflow.

## 2. Common workflow patterns

| Pattern | When to use |
|---|---|
| Prompt chaining | The task splits into fixed steps; each step's output is the next step's input |
| Routing | Classify first, then hand off to a specialized flow |
| Parallelization | Run independent subtasks in parallel, or run the same task several times and vote |
| Orchestrator-workers | One model splits the task at runtime and hands pieces to several workers |
| Evaluator-optimizer | One model generates, another reviews, and they loop to improve |

Look back at your existing code and you will likely find these patterns already. Recognize them first, then decide whether to add more autonomy.

## 3. Measuring reliability: pass@k vs pass^k

- **pass@k**: the probability that **at least one** of k runs succeeds. Fits cases where you can retry or a human picks the result, such as generating candidate solutions.
- **pass^k**: the probability that **all** k runs succeed. Fits user-facing cases where every run must be right.

For the same model, pass@k goes up as k grows, and pass^k goes down. A user-facing agent should be judged on pass^k.

> [!IMPORTANT]
> A 90% single-run success rate looks good, but pass^5 is only about 59% (0.9⁵ ≈ 0.59).

## 4. Prompt techniques that hold up

These work reliably across models and tasks:

- **n-shot examples / in-context learning**: give a few input-output examples.
- **Chain-of-thought**: have the model reason before it answers. With structured output, put the reasoning field before the answer field; otherwise the model commits to an answer first and the reasoning no longer helps.
- **Provide relevant material**: put the context the model needs directly in the prompt instead of hoping it remembers.

## 5. Further reading

- Anthropic, [*Building Effective Agents*](https://www.anthropic.com/engineering/building-effective-agents)
- Phil Schmid, [*Pass@k vs Pass^k: Understanding Agent Reliability*](https://www.philschmid.de/agents-pass-at-k-pass-power-k)
- [*What We've Learned From A Year of Building with LLMs*](https://applied-llms.org/)
- Claude Cookbook, [*Async multi-agent orchestration*](https://platform.claude.com/cookbook/patterns-agents-async-multi-agent-orchestration)
