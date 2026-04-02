# Effective Context Engineering for AI Agents

*By Anthropic — 2025-09-17*
*Source: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents*

---

## Core principle

Context engineering = finding the **smallest possible set of high-signal tokens** that maximize likelihood of desired outcome. Context is a finite resource with diminishing marginal returns.

## Context engineering vs prompt engineering

- **Prompt engineering**: writing and organizing LLM instructions for one-shot tasks
- **Context engineering**: curating the optimal set of ALL tokens during multi-turn inference — system instructions, tools, MCP, external data, message history

As agents run in loops generating data, context must be cyclically refined. This is iterative curation, not a discrete writing task.

## Why it matters: context rot

As token count increases, model's ability to accurately recall information decreases (n² pairwise attention relationships). Performance degrades gradually, not as a cliff. Models have an "attention budget" — every token depletes it.

## Anatomy of effective context

### System prompts
Find the **Goldilocks altitude**: specific enough to guide behavior, flexible enough to provide strong heuristics. Avoid:
- Brittle if-else hardcoded logic (fragile, high maintenance)
- Vague high-level guidance that assumes shared context

Organize with XML tags or Markdown headers. Strive for **minimal set that fully outlines expected behavior** (minimal ≠ short). Start with a minimal prompt on the best model, then add instructions based on observed failure modes.

### Tools
- Self-contained, robust to error, extremely clear on intended use
- Minimal viable toolset — if a human can't tell which tool to use in a situation, neither can an agent
- See "Writing effective tools for agents" article for details

### Examples (few-shot)
- Curate diverse, canonical examples. Don't stuff a laundry list of edge cases.
- Examples are the "pictures" worth a thousand words for an LLM.

## Context retrieval: just-in-time vs pre-computed

**Just-in-time** (agentic search): agents maintain lightweight references (file paths, stored queries, links) and load data at runtime using tools. Mirrors human cognition — we don't memorize corpuses, we use indexing systems.

- Enables **progressive disclosure**: agents discover context incrementally. File sizes, naming conventions, timestamps provide navigation signals.
- Tradeoff: slower than pre-computed retrieval; requires good tool design

**Hybrid strategy** (recommended): retrieve some data up front for speed, let agents explore further autonomously. Claude Code does this: CLAUDE.md files loaded up front, glob/grep for just-in-time file retrieval.

## Long-horizon techniques

### Compaction
Summarize conversation nearing context limit, reinitiate with summary. Art lies in what to keep vs discard:
- Maximize recall first, then improve precision
- Low-hanging fruit: clear tool call results deep in history (why see raw results again?)
- Tool result clearing is the "safest lightest touch" form of compaction

### Structured note-taking (agentic memory)
Agent writes notes persisted outside context window, pulls them back later. Like maintaining a to-do list or NOTES.md file. Enables coherence across thousands of steps (e.g., Claude playing Pokemon tracks objectives across context resets).

### Sub-agent architectures
Specialized subagents handle focused tasks with clean context windows. Each explores extensively but returns condensed summary (1,000-2,000 tokens). Clear separation of concerns — detailed search context isolated within subagents, lead agent synthesizes.

### Choosing between them
- **Compaction**: conversational flow, extensive back-and-forth
- **Note-taking**: iterative development with clear milestones
- **Multi-agent**: complex research/analysis with parallel exploration
