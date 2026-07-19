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

The `/claude-docs` skill routes questions to the right file via a keyword table, then delegates the reading to a subagent that returns a digest (keeping large doc bodies out of the main context), and the progressive disclosure within `internals/` means you get architecture first, implementation detail only when you drill in.

## Structure

```
claudedocs/
├── SKILL.md                     # The /claude-docs skill (routing table + subagent-digest model)
├── manifest.md                  # File index + staleness-recheck / refresh recipe
├── NOTES.md                     # Personal notes on Claude Code context construction
│
├── reference-docs/              # Claude Code configuration reference
│   ├── overview.md              #   What Claude Code is, where it runs
│   ├── how-claude-code-works.md #   Agentic loop, built-in tools (official)
│   ├── claude-directory.md      #   .claude/ layout: CLAUDE.md, settings, hooks, skills...
│   ├── auto-mode-config.md      #   Trusted repos/domains, auto-mode CLI
│   ├── prompt-caching.md        #   Cache hit rate, /compact cost, cache breaks
│   ├── sessions.md              #   --continue/--resume/--from-pr, /resume
│   ├── glossary.md              #   Claude Code terminology
│   ├── errors.md                #   Runtime error reference
│   ├── tools-reference.md       #   All tools, permission requirements
│   ├── permissions.md           #   Permission rules, modes, managed settings
│   ├── settings.md              #   Settings fields, file locations, precedence
│   ├── model-config.md          #   Model aliases, effort levels, 1M context
│   ├── memory.md                #   CLAUDE.md, rules/, auto memory
│   ├── mcp.md                   #   MCP server config, tool search
│   ├── cli-reference.md         #   CLI flags and options
│   ├── commands.md              #   Built-in slash commands
│   ├── env-vars.md              #   Environment variables
│   ├── permission-modes.md      #   auto/manual/plan/acceptEdits/bypass
│   ├── fast-mode.md             #   /fast, high-speed Opus
│   ├── context-window.md        #   Context budget, compaction
│   ├── sandboxing.md            #   Sandboxed Bash tool
│   ├── sandbox-environments.md  #   Choose a sandbox environment
│   └── security-guidance.md     #   Catch security issues while coding
│
├── agents-docs/                 # Agent system, orchestration, sessions
│   ├── sub-agents.md            #   Subagent definitions, frontmatter
│   ├── agent-teams.md           #   Multi-session teams, mailbox
│   ├── headless.md              #   CLI -p / programmatic use
│   ├── workflows.md             #   Dynamic multi-subagent orchestration
│   ├── worktrees.md             #   Parallel sessions in git worktrees
│   ├── agent-view.md            #   claude agents dashboard
│   ├── agents.md                #   Run agents in parallel
│   ├── routines.md              #   Scheduled cloud agents
│   └── remote-control.md        #   Continue sessions from any device
│
├── skills-docs/                 # Skills system
│   ├── skills.md                #   SKILL.md format, invocation
│   └── features-overview.md     #   When to use what (decision table)
│
├── hooks-docs/                  # Hooks system
│   ├── hooks.md                 #   Hook events, schema, reference
│   └── hooks-guide.md           #   Practical patterns, setup
│
├── guides-docs/                 # Usage guides, terminal/UI, review, artifacts
│   ├── best-practices.md        #   Context management, prompting
│   ├── checkpointing.md         #   Track/rewind session state
│   ├── output-styles.md         #   Custom output styles
│   ├── scheduled-tasks.md       #   /loop, cron scheduling
│   ├── channels.md              #   Push events, Telegram/iMessage
│   ├── channels-reference.md    #   Channel plugin schema
│   ├── terminal-config.md       #   Terminal setup, Vim mode, themes
│   ├── interactive-mode.md      #   Input modes, session interaction
│   ├── keybindings.md           #   Customize keyboard shortcuts
│   ├── statusline.md            #   Customize the status line
│   ├── code-review.md           #   /code-review, diff review
│   ├── ultrareview.md           #   Cloud multi-agent review
│   ├── artifacts.md             #   Artifact tool, publish pages
│   ├── goal.md                  #   /goal, autonomous goal-directed work
│   ├── chrome.md                #   Claude in Chrome, --chrome, browser automation
│   ├── computer-use.md          #   macOS native app control (pairs with chrome.md)
│   ├── ultraplan.md             #   Cloud planning (pairs with ultrareview.md)
│   ├── debug-your-config.md     #   /context, /doctor, /hooks, /mcp diagnostics
│   └── advisor.md               #   Advisor tool: consult a stronger model mid-task
│
├── plugins-docs/                # Plugin system
│   ├── plugins.md               #   Plugin structure, distribution
│   └── plugins-reference.md     #   Manifest schema, component specs
│
├── agent-sdk-docs/              # Agent SDK library (core subset)
│   ├── overview.md              #   SDK overview, capabilities
│   ├── agent-loop.md            #   How the SDK agent loop works
│   ├── subagents.md             #   Programmatic subagents
│   ├── permissions.md           #   canUseTool, permission modes
│   ├── custom-tools.md          #   In-process MCP custom tools
│   └── structured-outputs.md    #   Schema-validated output
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

Used via the `/claude-docs` skill — its definition (`SKILL.md`) ships in this repo; install it by placing it at `~/.claude/skills/claude-docs/SKILL.md`, and clone this repo to `~/claudedocs` (the routing table uses `~/claudedocs/` paths). The skill matches questions to files via a keyword routing table, then **delegates the reading to a subagent that returns a digest** — keeping large doc bodies out of the main context.

You can also just say `read ~/claudedocs` to let Claude explore the knowledge base directly — start with `manifest.md` for the full file index (and its staleness-recheck / refresh recipe), or `internals/index.md` for the harness architecture overview.

## Future: progressive disclosure for this repo itself

This repo currently has 87 markdown files. The `/claude-docs` skill routes by keyword and now delegates the actual reading to a subagent (digest), so the main context stays light even as the file count grows — manageable today.

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

- **Official docs:** [code.claude.com/docs](https://code.claude.com/docs/en/overview) (cached 2026-07-13, CLI v2.1.203)
- **Articles:** Anthropic engineering blog posts (dates noted in each file)
- **Internals:** Source map of `@anthropic-ai/claude-code@2.1.88` — all rights belong to Anthropic
