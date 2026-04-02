# Claude Docs Manifest

Local cache of Claude Code official documentation. Fetched: 2026-03-14.
Source: https://code.claude.com/docs/

## agents-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `sub-agents.md` | Subagent definition & config | Frontmatter fields, tool restrictions, permission modes, hooks in agents, memory, skill preloading, resume pattern |
| `agent-teams.md` | Multi-session agent teams | Team lead/teammate architecture, task list, mailbox, parallel work patterns, comparison with subagents |
| `headless.md` | Programmatic / Agent SDK | `-p` flag, `--output-format json/stream-json`, `--allowedTools`, `--continue`/`--resume`, system prompt flags |

## skills-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `skills.md` | Skills system | SKILL.md frontmatter, `disable-model-invocation`, `context: fork`, `$ARGUMENTS`, supporting files, `allowed-tools`, bundled skills |
| `features-overview.md` | When to use what | CLAUDE.md vs Skills vs Subagents vs Agent teams vs MCP vs Hooks — decision table, context cost per feature |

## hooks-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `hooks.md` | Hooks reference | All hook events schema, JSON input/output, exit codes, async/HTTP/prompt/agent hooks, MCP tool hooks |
| `hooks-guide.md` | Hooks practical guide | Setup walkthrough, common patterns (notify, auto-format, file protection), matchers, hook location config |

## reference-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `tools-reference.md` | All available tools | Full tool list with permission requirements, Bash persistence behavior |
| `permissions.md` | Permission system | Rule syntax, permission modes, Bash/Read/Edit/WebFetch/MCP/Agent rules, managed settings |
| `model-config.md` | Model selection | Aliases (sonnet/opus/haiku/opusplan), effort levels, 1M context, env vars, per-subagent model |
| `memory.md` | Persistent memory | CLAUDE.md scoping, `.claude/rules/` path-specific rules, auto memory, `/memory` command |
| `mcp.md` | Model Context Protocol | MCP server config, tool search, scoping to subagents, managed MCP, troubleshooting |
| `settings.md` | Settings system | All settings fields, settings files locations, precedence, permission settings table |
| `cli-reference.md` | CLI flags & commands | All `claude` CLI flags, `--agents`, `--model`, `--allowedTools`, `--output-format`, etc. |
| `commands.md` | Built-in slash commands | Full reference for `/compact`, `/clear`, `/model`, `/permissions`, `/hooks`, `/agents`, etc. |
| `env-vars.md` | Environment variables | All env vars controlling Claude Code behavior, model overrides, feature flags |

## guides-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `output-styles.md` | Output styles | Built-in and custom output styles, frontmatter, use beyond software engineering |
| `scheduled-tasks.md` | Scheduled tasks | `/loop`, `CronCreate`/`CronList`/`CronDelete`, one-time reminders, cron expression reference |
| `terminal-config.md` | Terminal setup | Line breaks, notifications, Vim mode, terminal optimization |
| `checkpointing.md` | Checkpointing | Track/rewind/summarize edits and conversation state |
| `channels.md` | Channels | Push events into running sessions via MCP; Telegram/Discord setup, fakechat quickstart, sender allowlists, pairing, enterprise controls, research preview |
| `best-practices.md` | Best practices | Context management, CLAUDE.md authoring, prompting patterns, parallel sessions, automation, common failure patterns |

## plugins-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `plugins.md` | Create plugins | Plugin structure, skills/hooks/MCP in plugins, local testing, distribution |
| `plugins-reference.md` | Plugins reference | Full manifest schema, all component schemas, CLI commands, scopes, caching, debugging |

## internals/

Reverse-engineered source-level analysis of Claude Code's harness implementation (from `@anthropic-ai/claude-code` v2.1.88 source map). Not official docs — use for understanding how the harness is actually built.

**Progressive disclosure:** Start with `index.md` for architecture and design decisions. Each detail file has a "How it works" top half (conceptual) and "Implementation reference" bottom half (TypeScript, source paths).

| File | Topic | Key content |
|------|-------|-------------|
| `index.md` | **Architecture overview** | Flow diagram, 5 subsystem summaries with key design decisions and numbers, pointers to detail files |
| `system-prompt-and-context.md` | System prompt & context injection | Prompt priority, static/dynamic split, cache boundary, prompt text excerpts, two context channels, git status, CLAUDE.md hierarchy, memory directory |
| `tool-framework.md` | Tool system & permissions | Tool definition, registration, execution pipeline (8 phases), result persistence, concurrency, permission modes/rules/decision flow |
| `turn-loop.md` | Core loop & compaction | while(true) loop, immutable State, 6 phases, stop/continue conditions, API streaming, 5-layer compaction, progressive error recovery |
| `hooks-and-extensibility.md` | Hooks, agents & skills | 4 hook types, 25 events, execution mechanisms, SSRF protection, AgentTool, fork subagents, verification agent, skill loading |
| `design-patterns.md` | Patterns, model config & safety | Model selection, extended thinking, fast mode, cost tracking, token budgets, 8-layer safety, enterprise policies, architectural patterns |
| `verification-agent.md` | Verification agent deep dive | Adversarial system prompt, anti-rationalization list, 11 type-specific strategies, evidence format, tool restrictions, trigger mechanisms, design lessons |
| `internal-prompts.md` | Internal harness prompts | Compaction (9-section summary, analysis stripping, cache-safe forking), auto-mode classifier (two-stage, PowerShell rules), session memory (10-section template), built-in agent prompts and design patterns |
| `tool-prompts.md` | Tool description engineering | Layered reinforcement pattern, Bash tool (git commit/PR protocols, sandbox, safety), FileEdit (read-first), Agent (fork vs spawn, prompt authoring), Grep/Glob (exclusivity), anti-patterns |
| `mid-conversation-injection.md` | Runtime injection & errors | `<system-reminder>` 5 patterns (user context, recurring attachments, hooks, warnings, side questions), permission denial steering, rejection memory hints, validation recovery instructions |

## articles/

Engineering blog articles and practitioner insights. Date indicates original publication — check for staleness on rapidly evolving topics.

| File | Date | Topic | Key content |
|------|------|-------|-------------|
| `building-effective-agents.md` | 2024-09-19 | Agent architecture patterns | Workflows vs agents, 5 workflow patterns (chaining/routing/parallelization/orchestrator-workers/evaluator-optimizer), tool prompt engineering, ACI design |
| `claude-think-tool.md` | 2025-01-06 | Think tool for mid-chain reasoning | Think tool vs extended thinking, tau-bench results, optimized prompt patterns, when to use vs not |
| `multi-agent-research-system.md` | 2025-04-18 | Anthropic's Research multi-agent | Orchestrator-worker architecture, 90% improvement over single-agent, prompt engineering for delegation/scaling/tool selection, eval design |
| `writing-tools-for-agents.md` | 2025-06-26 | Tool design for agents | Prototype→eval→collaborate loop, choosing/namespacing/designing tools, token-efficient responses, eval task design |
| `context-engineering.md` | 2025-09-17 | Context engineering for agents | Context rot, attention budget, just-in-time retrieval, compaction/note-taking/sub-agents for long-horizon, hybrid retrieval strategy |
| `effective-harnesses.md` | 2025-11-24 | Long-running agent harnesses | Initializer+coding agent pattern, feature list JSON, incremental progress, session startup sequence, failure modes table |
| `harness-design-long-running-apps.md` | 2026-03-06 | GAN-inspired multi-agent harness | Generator-evaluator separation, context anxiety vs compaction, sprint contracts, Playwright MCP for QA, 3-agent full-stack architecture |
| `prompt-caching-lessons.md` | 2026-02-19 | Prompt caching in agents | Cache hit mechanics, prefix matching, cache-breaking pitfalls, compaction interaction |
| `seeing-like-an-agent.md` | 2026-02-27 | Agent perception & tool design | Action space design, elicitation, progressive disclosure |
| `how-we-use-skills.md` | 2026-03-17 | Skills system in practice | Skill categories, tips, gotchas, distribution, on-demand hooks |
