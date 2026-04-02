# Building Effective Agents

*By Erik Schluntz and Barry Zhang, Anthropic — 2024-09-19*
*Source: https://www.anthropic.com/engineering/building-effective-agents*

---

## Core thesis

Start simple. Only add complexity when it demonstrably improves outcomes. The most successful agent implementations use simple, composable patterns — not complex frameworks.

## Key distinction: workflows vs agents

- **Workflows**: LLMs + tools orchestrated through predefined code paths
- **Agents**: LLMs dynamically direct their own processes and tool usage

Use workflows when tasks are well-defined; agents when flexibility and model-driven decisions are needed at scale.

## Workflow patterns

| Pattern | When to use | Example |
|---------|------------|---------|
| **Prompt chaining** | Task cleanly decomposes into fixed sequential subtasks | Generate outline → validate → write document |
| **Routing** | Distinct input categories need separate handling | Route support queries to refund/technical/general handlers |
| **Parallelization** | Subtasks can run simultaneously, or multiple perspectives needed | Sectioning (guardrails + response in parallel), Voting (multiple vulnerability reviewers) |
| **Orchestrator-workers** | Can't predict subtasks needed; dynamic decomposition | Coding agent editing multiple files based on task description |
| **Evaluator-optimizer** | Clear evaluation criteria; iterative refinement adds value | Literary translation with critic loop, multi-round search |

## Agent design principles

1. **Maintain simplicity** in agent design
2. **Prioritize transparency** — show planning steps explicitly
3. **Craft the ACI (Agent-Computer Interface)** through thorough tool documentation and testing

## Tool prompt engineering (Appendix 2)

- **Format matters**: Give the model tokens to "think" before writing itself into a corner. Keep formats close to naturally occurring text. Avoid formatting overhead (line counting, escaping).
- **Put yourself in the model's shoes**: If a tool isn't obvious to use from its description, it won't be obvious to the model either. Include example usage, edge cases, input format requirements.
- **Poka-yoke tools**: Change arguments to make mistakes harder (e.g., require absolute filepaths instead of relative).
- **Iterate on tools, not just prompts**: "We spent more time optimizing our tools than the overall prompt" (SWE-bench agent).

## When NOT to use agents

- When optimizing single LLM calls with retrieval + in-context examples is sufficient
- When latency/cost tradeoff doesn't make sense
- When the task doesn't need flexibility or model-driven decisions
