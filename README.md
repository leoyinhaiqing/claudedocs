# claudedocs

Local knowledge base for Claude Code — official documentation cache, Anthropic engineering articles, and reverse-engineered harness internals.

## Why local cache?

Claude Code already has a [Guide subagent](articles/seeing-like-an-agent.md) that fetches docs from the web at runtime. As the Anthropic team described it:

> We gave Claude a link to its docs which it could then load to search for more information. This worked but we found that Claude would load a lot of results into context to find the right answer when really all you needed was the answer.

The Guide agent solves this with progressive disclosure — a subagent that knows how to search docs well and return just the answer. That works for answering questions about Claude Code.

But this repo serves a different purpose: **building harness engineering for other projects**. When I'm designing my own agent harness, I need:

1. **The design patterns** — not "how do I configure hooks?" but "how does Claude Code's hook system work internally, and what can I learn from it?"
2. **The craft details** — the actual prompt text, anti-rationalization lists, error message engineering, layered reinforcement patterns. These are the things that make a harness work vs. just exist.
3. **Cross-referencing** — reading the verification agent's adversarial prompt alongside the "generator-evaluator" article, or the compaction prompt alongside the context engineering article.

The Guide agent can't do this — it searches official docs, not source internals. And web fetching loads too much noise when you need a specific design decision.

The `/claude-docs` skill routes questions to the right file via a keyword table, loads only what's needed, and the progressive disclosure within `internals/` means you get architecture first, implementation detail only when you drill in.

## Structure

```
claudedocs/
├── manifest.md                  # File index with descriptions
│
├── reference-docs/              # Claude Code configuration reference
│   ├── tools-reference.md       #   All tools, permission requirements
│   ├── permissions.md           #   Permission rules, modes, managed settings
│   ├── settings.md              #   Settings fields, file locations, precedence
│   ├── model-config.md          #   Model aliases, effort levels, 1M context
│   ├── memory.md                #   CLAUDE.md, rules/, auto memory
│   ├── mcp.md                   #   MCP server config, tool search
│   ├── cli-reference.md         #   CLI flags and options
│   ├── commands.md              #   Built-in slash commands
│   └── env-vars.md              #   Environment variables
│
├── agents-docs/                 # Agent system
│   ├── sub-agents.md            #   Subagent definitions, frontmatter
│   ├── agent-teams.md           #   Multi-session teams, mailbox
│   └── headless.md              #   SDK, -p flag, CI/CD
│
├── skills-docs/                 # Skills system
│   ├── skills.md                #   SKILL.md format, invocation
│   └── features-overview.md     #   When to use what (decision table)
│
├── hooks-docs/                  # Hooks system
│   ├── hooks.md                 #   Hook events, schema, reference
│   └── hooks-guide.md           #   Practical patterns, setup
│
├── guides-docs/                 # Usage guides
│   ├── best-practices.md        #   Context management, prompting
│   ├── checkpointing.md         #   Track/rewind session state
│   ├── output-styles.md         #   Custom output styles
│   ├── scheduled-tasks.md       #   /loop, cron scheduling
│   ├── channels.md              #   Push events, Telegram/Discord
│   └── terminal-config.md       #   Terminal setup, Vim mode
│
├── plugins-docs/                # Plugin system
│   ├── plugins.md               #   Plugin structure, distribution
│   └── plugins-reference.md     #   Manifest schema, component specs
│
├── articles/                    # Anthropic engineering blog posts
│   ├── building-effective-agents.md
│   ├── context-engineering.md
│   ├── effective-harnesses.md
│   ├── harness-design-long-running-apps.md
│   ├── prompt-caching-lessons.md
│   ├── seeing-like-an-agent.md
│   ├── writing-tools-for-agents.md
│   ├── claude-think-tool.md
│   ├── multi-agent-research-system.md
│   └── how-we-use-skills.md
│
└── internals/                   # Reverse-engineered harness analysis
    ├── index.md                 #   Architecture overview (start here)
    ├── system-prompt-and-context.md
    ├── tool-framework.md
    ├── turn-loop.md
    ├── hooks-and-extensibility.md
    ├── design-patterns.md
    ├── verification-agent.md    #   Deep dive: adversarial evaluation
    ├── internal-prompts.md      #   Compaction, classifier, memory
    ├── tool-prompts.md          #   Tool description engineering
    └── mid-conversation-injection.md  # <system-reminder> patterns, errors
```

## The internals/ directory

Reverse-engineered from the [`@anthropic-ai/claude-code` v2.1.88](https://www.npmjs.com/package/@anthropic-ai/claude-code/v/2.1.88) npm package source map. Not an official Anthropic release.

**Progressive disclosure — three tiers:**

| Tier | What you read | When |
|------|--------------|------|
| `index.md` | Architecture + key design decisions (~136 lines) | "How does Claude Code's harness work?" |
| Detail file top half | Conceptual explanation + flow diagrams | "How does the tool system work?" |
| Detail file bottom half | TypeScript interfaces, source paths, code patterns | "I'm implementing this, show me the interface" |

**What's covered:**

- **System prompt** — static/dynamic split, cache boundary, two context channels, CLAUDE.md hierarchy
- **Tool framework** — 8-phase execution pipeline, result persistence, concurrency, permissions
- **Turn loop** — `while(true)` with immutable state, 5 compaction layers, progressive error recovery
- **Hooks & extensibility** — 25 hook events, 4 types, agent orchestration, skills
- **Verification agent** — adversarial prompt engineering: anti-rationalizations, 11 type-specific strategies, evidence format
- **Internal prompts** — compaction (9-section summary), auto-mode classifier (two-stage), session memory (10-section template), built-in agent designs
- **Tool prompts** — layered reinforcement, Bash git/PR protocols, tool anti-patterns
- **Mid-conversation injection** — `<system-reminder>` patterns, error message steering, denial workaround guidance

## Usage

Used via the `/claude-docs` skill (defined in `~/.claude/skills/claude-docs/SKILL.md`). The skill matches user questions to files via a keyword routing table and loads only the relevant file(s).

You can also just say `read ~/claudedocs` to let Claude explore the knowledge base directly — start with `manifest.md` for the full file index, or `internals/index.md` for the harness architecture overview.

## Future: progressive disclosure for this repo itself

This repo currently has 45 files totaling ~12,000 lines. The `/claude-docs` skill routes by keyword and loads 1-2 files per question — manageable today.

As the repo grows (more articles, deeper internals analysis, new Claude Code versions), the routing table approach may hit limits:
- Too many routing entries make the skill definition itself a context burden
- Overlapping keywords cause wrong-file matches
- A single file growing past ~300 lines becomes too dense for a single load

When that happens, consider applying the same progressive disclosure pattern used in `internals/`:
- **Index files per directory** — each directory gets an `index.md` that summarizes its contents and points to detail files (like `internals/index.md` already does)
- **Tiered loading** — the skill loads the directory index first, then the specific file only if needed
- **Split large files** — any file past ~250 lines should be split into a conceptual top half and a reference bottom half, or into separate files

The `internals/` directory already demonstrates this pattern. The rest of the repo (`articles/`, `reference-docs/`, etc.) doesn't need it yet — each file is self-contained and under 200 lines. But the design is there to replicate when the time comes.

## Sources

- **Official docs:** [code.claude.com/docs](https://code.claude.com/docs/en/overview) (cached 2026-03-14)
- **Articles:** Anthropic engineering blog posts (dates noted in each file)
- **Internals:** Source map of `@anthropic-ai/claude-code@2.1.88` — all rights belong to Anthropic
