# How We Built Our Multi-Agent Research System

*By Anthropic — 2025-04-18*
*Source: https://www.anthropic.com/engineering/multi-agent-research-system*

---

## Architecture

Orchestrator-worker pattern: lead agent coordinates, specialized subagents operate in parallel.

- **Lead agent** (Opus): analyzes query, develops strategy, spawns subagents, synthesizes results
- **Subagents** (Sonnet): search independently with own context windows, return condensed findings
- **Citation agent**: post-processes to attribute claims to sources

Each subagent explores extensively (tens of thousands of tokens) but returns only ~1,000-2,000 tokens of distilled summary.

## Why multi-agent

- Multi-agent Opus+Sonnet outperformed single-agent Opus by **90.2%** on internal research eval
- Token usage explains **80% of performance variance** (BrowseComp). Number of tool calls and model choice explain remaining 15%.
- Multi-agent architectures effectively scale token usage for tasks exceeding single-agent limits
- Tradeoff: ~15x more tokens than chat, ~4x more than single agent

## Best fit for multi-agent

- Heavy parallelization opportunities
- Information exceeding single context window
- Interfacing with numerous complex tools
- High-value tasks (token cost justified)

## Prompt engineering lessons

1. **Think like your agents**: Build simulations, watch step-by-step. Failure modes become obvious.
2. **Teach orchestrator to delegate**: Each subagent needs objective, output format, tool guidance, clear boundaries. Vague instructions → duplicated work and gaps.
3. **Scale effort to query complexity**: Embed explicit scaling rules — simple fact-finding: 1 agent, 3-10 tool calls; complex research: 10+ subagents with divided responsibilities.
4. **Tool selection is critical**: Give agents explicit heuristics — examine available tools first, match to intent, prefer specialized over generic. Bad tool descriptions derail entire chains.
5. **Let agents improve themselves**: Claude 4 models are excellent prompt engineers. Tool-testing agent: use tool dozens of times, rewrite description to avoid failures → 40% decrease in task completion time.
6. **Start wide, then narrow**: Prompt agents to begin with short, broad queries, evaluate landscape, then progressively narrow. Agents default to overly specific queries.
7. **Guide the thinking process**: Extended thinking improves instruction-following, reasoning, efficiency. Subagents use interleaved thinking after tool results to evaluate quality and plan next query.
8. **Parallel tool calling**: Lead spawns 3-5 subagents in parallel; each subagent uses 3+ tools in parallel → up to 90% time reduction.

## Evaluation insights

- Start evaluating immediately with ~20 queries. Effect sizes in early development are large enough to see clearly.
- Judge **outcomes**, not paths — agents may take different valid routes to the same answer.
- Instill good **heuristics** rather than rigid rules. Encode expert human research strategies in prompts.
