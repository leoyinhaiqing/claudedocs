---
name: claude-docs
description: Loads official Claude Code documentation from a local cache (`~/claudedocs/`) and answers grounded in it — delegating the actual doc-reading to a subagent so large doc bodies stay out of the main context. Triggers when the user asks about Claude Code features, configuration, or capabilities — including subagents, agent teams, workflows, worktrees, routines, skills, hooks, permissions & permission modes, MCP, settings, model config, CLI flags, commands, env vars, plugins, channels, code review, artifacts, sandboxing, the Agent SDK, Claude in Chrome / browser automation, computer use, auto mode config, prompt caching, session management, the advisor tool, config debugging, or internal harness architecture. Also triggers on "check the docs", "how does Claude Code handle X", or any "can Claude Code..." / "does Claude Code..." / "how do I..." question about Claude Code itself.
---

# Claude Docs

Official Claude Code documentation lives at `~/claudedocs/` — raw Mintlify
markdown. Provenance (fetch date, CLI version) and the staleness-recheck /
refresh recipe are in `~/claudedocs/manifest.md`.

## Operating model — delegate the reading; never load doc bodies into the main context

The cache is large: 50 official pages plus `articles/` and `internals/`, with
individual pages running past 250KB. Reading them directly would flood the main
session's context. Instead, treat this as high-volume, low-conclusion work and
push it into a subagent so only a short digest returns:

1. **Route** — match the user's topic to file path(s) using the routing table
   below. This is a cheap string match; it does NOT require opening any doc.
2. **Delegate** — spawn a general-purpose subagent (a cheap tier such as
   `sonnet` is enough) and hand it the matched path(s) plus the user's
   question. Instruct it to read only those file(s) and return a focused, cited
   answer (a digest) — not the raw text.
3. **Relay** — the subagent's digest is the answer; relay it / act on it. The
   large doc body stays in the subagent's context, not yours.

**Do not open files under `~/claudedocs/` with the Read tool in the main
session** — that defeats the purpose. The sole exception is `manifest.md`
(small, index only), which you may read directly to pick files.

**Fallback:** if the caller cannot spawn a subagent, read only the single
most-relevant matched file directly and keep the scope minimal. Delegation is
the default; direct read is the degraded mode.

## Routing table

Match the user's topic to the right file(s), then hand those paths to the
subagent. Most questions need one file; cross-cutting questions may need two.

### agents-docs/ — subagents, teams, orchestration, sessions
| Topic keywords | File path |
|----------------|-----------|
| subagent, agent definition, frontmatter, resume, spawn nested subagent, fork subagent | `~/claudedocs/agents-docs/sub-agents.md` |
| agent team, teammate, parallel sessions, mailbox, SendMessage | `~/claudedocs/agents-docs/agent-teams.md` |
| headless, `-p` flag, print mode, CI/CD scripting, stream-json (CLI programmatic use) | `~/claudedocs/agents-docs/headless.md` |
| dynamic workflow, Workflow tool, orchestrate subagents at scale, ultracode, pipeline/parallel fan-out, phases | `~/claudedocs/agents-docs/workflows.md` |
| worktree, parallel worktree session, `isolation: worktree`, EnterWorktree | `~/claudedocs/agents-docs/worktrees.md` |
| agent view, `claude agents`, manage/attach/detach sessions, background agents, agent dashboard | `~/claudedocs/agents-docs/agent-view.md` |
| run agents in parallel, multiple parallel agent sessions | `~/claudedocs/agents-docs/agents.md` |
| routine, scheduled cloud agent, `/schedule`, cron cloud agent, `/fire` endpoint, GitHub-event trigger | `~/claudedocs/agents-docs/routines.md` |
| remote control, continue session from phone/another device, mobile push, permission relay | `~/claudedocs/agents-docs/remote-control.md` |

### skills-docs/ — skills system
| Topic keywords | File path |
|----------------|-----------|
| skill, SKILL.md, slash command, `context: fork`, `$ARGUMENTS`, allowed-tools, disable-model-invocation | `~/claudedocs/skills-docs/skills.md` |
| which feature, skill vs hook vs subagent vs MCP vs CLAUDE.md, decision table | `~/claudedocs/skills-docs/features-overview.md` |

### hooks-docs/ — hooks
| Topic keywords | File path |
|----------------|-----------|
| hook event, hook schema, exit code, hook reference, mcp_tool hook, PermissionDenied/FileChanged/Elicitation events | `~/claudedocs/hooks-docs/hooks.md` |
| hook setup, auto-format, file protection, matchers, `if` field, notify | `~/claudedocs/hooks-docs/hooks-guide.md` |

### reference-docs/ — tools, permissions, config
| Topic keywords | File path |
|----------------|-----------|
| what is Claude Code, overview, where it runs | `~/claudedocs/reference-docs/overview.md` |
| how Claude Code works, agentic loop (official), built-in tools overview | `~/claudedocs/reference-docs/how-claude-code-works.md` |
| `.claude` directory, project vs `~/.claude`, where config files live | `~/claudedocs/reference-docs/claude-directory.md` |
| auto mode trusted repos/domains, environment context, auto-mode CLI subcommands | `~/claudedocs/reference-docs/auto-mode-config.md` |
| prompt caching, cache hit rate, cache break, `/compact` cost, CLAUDE.md edit timing (official) | `~/claudedocs/reference-docs/prompt-caching.md` |
| session management, `--continue`, `--resume`, `--from-pr`, `/resume`, transcript export | `~/claudedocs/reference-docs/sessions.md` |
| glossary, terminology, what does X mean | `~/claudedocs/reference-docs/glossary.md` |
| error message, error reference, what does this error mean | `~/claudedocs/reference-docs/errors.md` |
| tool list, tool permissions, Bash persistence, per-tool behavior | `~/claudedocs/reference-docs/tools-reference.md` |
| permission rules, deny/allow syntax, `Tool(param:value)` matching, managed settings | `~/claudedocs/reference-docs/permissions.md` |
| permission mode, auto mode, manual mode, plan mode, acceptEdits, bypassPermissions | `~/claudedocs/reference-docs/permission-modes.md` |
| model alias, opus/sonnet/haiku/fable, effort (xhigh/max), 1M context, fallback model | `~/claudedocs/reference-docs/model-config.md` |
| CLAUDE.md, memory, rules/, auto memory, CLAUDE.local.md, AGENTS.md | `~/claudedocs/reference-docs/memory.md` |
| context window, context budget, compaction, microcompact, `/compact` behavior | `~/claudedocs/reference-docs/context-window.md` |
| MCP server, tool search, external service, WebSocket transport, OAuth | `~/claudedocs/reference-docs/mcp.md` |
| settings.json, settings fields, precedence, managed-settings.d/ | `~/claudedocs/reference-docs/settings.md` |
| CLI flag, `--model`, `--allowedTools`, `--agents`, `--bg`, subcommands | `~/claudedocs/reference-docs/cli-reference.md` |
| /compact, /clear, /model, /usage, built-in slash commands | `~/claudedocs/reference-docs/commands.md` |
| environment variable, feature flag, `CLAUDE_CODE_*` toggle | `~/claudedocs/reference-docs/env-vars.md` |
| fast mode, `/fast`, speed up responses | `~/claudedocs/reference-docs/fast-mode.md` |
| sandboxed Bash tool, sandbox rules, sandbox credentials | `~/claudedocs/reference-docs/sandboxing.md` |
| choose a sandbox environment, OS-level isolation | `~/claudedocs/reference-docs/sandbox-environments.md` |
| security guidance, catch security issues while coding, security-guidance plugin | `~/claudedocs/reference-docs/security-guidance.md` |

### guides-docs/ — how-to guides, terminal/UI, review, artifacts, channels
| Topic keywords | File path |
|----------------|-----------|
| output style, custom style, non-engineering, Proactive style | `~/claudedocs/guides-docs/output-styles.md` |
| scheduled task, `/loop`, cron expression, one-time reminder (local) | `~/claudedocs/guides-docs/scheduled-tasks.md` |
| terminal, line breaks, Vim mode, notifications, themes, tmux | `~/claudedocs/guides-docs/terminal-config.md` |
| interactive mode, input modes, session interaction | `~/claudedocs/guides-docs/interactive-mode.md` |
| keybindings, customize keyboard shortcuts, `keybindings.json`, chords | `~/claudedocs/guides-docs/keybindings.md` |
| status line, statusLine setting, refresh interval | `~/claudedocs/guides-docs/statusline.md` |
| checkpoint, rewind, undo, `/rewind`, summarize up-to/from-here | `~/claudedocs/guides-docs/checkpointing.md` |
| best practices, prompting, context management, adversarial review | `~/claudedocs/guides-docs/best-practices.md` |
| channels, push events into a session, Telegram, iMessage, `--channels` | `~/claudedocs/guides-docs/channels.md` |
| channels reference, channel plugin schema, channel event types | `~/claudedocs/guides-docs/channels-reference.md` |
| code review, `/code-review`, diff review, `--comment`/`--fix`, `/simplify` | `~/claudedocs/guides-docs/code-review.md` |
| ultrareview, cloud multi-agent review, adversarial review in CI | `~/claudedocs/guides-docs/ultrareview.md` |
| artifact, `Artifact` tool, publish interactive page to claude.ai | `~/claudedocs/guides-docs/artifacts.md` |
| goal, `/goal`, autonomous work toward a completion condition | `~/claudedocs/guides-docs/goal.md` |
| Claude in Chrome, `--chrome` flag, `/chrome`, browser automation, `mcp__claude-in-chrome__*`, browser tools in plan mode | `~/claudedocs/guides-docs/chrome.md` |
| computer use, control macOS apps, native app automation | `~/claudedocs/guides-docs/computer-use.md` |
| ultraplan, plan in the cloud, draft plan then execute remotely | `~/claudedocs/guides-docs/ultraplan.md` |
| debug config, `/context`, `/doctor`, why isn't my hook/MCP/skill loading | `~/claudedocs/guides-docs/debug-your-config.md` |
| advisor tool, escalate to a stronger model, second opinion mid-task | `~/claudedocs/guides-docs/advisor.md` |

### plugins-docs/ — plugins
| Topic keywords | File path |
|----------------|-----------|
| plugin, distribution, plugin manifest, skills-dir plugin, monitors | `~/claudedocs/plugins-docs/plugins.md` |
| plugin schema, component spec, plugin CLI, hook event table, agent frontmatter fields | `~/claudedocs/plugins-docs/plugins-reference.md` |

### agent-sdk-docs/ — Agent SDK library (distinct from headless CLI `-p`)
| Topic keywords | File path |
|----------------|-----------|
| Agent SDK overview, build on the SDK, SDK capabilities, language bindings | `~/claudedocs/agent-sdk-docs/overview.md` |
| SDK agent loop, how the SDK loop works, turn cycle | `~/claudedocs/agent-sdk-docs/agent-loop.md` |
| SDK subagents, programmatic subagents | `~/claudedocs/agent-sdk-docs/subagents.md` |
| SDK permissions, `canUseTool` callback, SDK managedSettings | `~/claudedocs/agent-sdk-docs/permissions.md` |
| SDK custom tools, in-process MCP, createSdkMcpServer | `~/claudedocs/agent-sdk-docs/custom-tools.md` |
| SDK structured outputs, schema-validated agent output | `~/claudedocs/agent-sdk-docs/structured-outputs.md` |

### articles/ — practitioner essays (external blog posts / tweets, not versioned API docs)
| Topic keywords | File path |
|----------------|-----------|
| prompt caching, cache hit, prefix match, cache break, compaction, forking | `~/claudedocs/articles/prompt-caching-lessons.md` |
| tool design, action space, elicitation, progressive disclosure, seeing like an agent | `~/claudedocs/articles/seeing-like-an-agent.md` |
| skill categories, skill tips, gotchas, distribute skills, on-demand hooks | `~/claudedocs/articles/how-we-use-skills.md` |
| agent patterns, workflow vs agent, orchestrator-workers, evaluator-optimizer, prompt chaining, routing | `~/claudedocs/articles/building-effective-agents.md` |
| think tool, extended thinking, scratchpad, policy compliance, tau-bench | `~/claudedocs/articles/claude-think-tool.md` |
| multi-agent, research system, subagent delegation, parallel search, orchestrator | `~/claudedocs/articles/multi-agent-research-system.md` |
| writing tools, MCP tool design, tool evaluation, namespacing, token efficiency | `~/claudedocs/articles/writing-tools-for-agents.md` |
| context engineering, context rot, attention budget, just-in-time retrieval, note-taking | `~/claudedocs/articles/context-engineering.md` |
| long-running agent, harness, initializer agent, feature list, session handoff | `~/claudedocs/articles/effective-harnesses.md` |
| GAN-inspired, generator-evaluator, sprint contract, context anxiety, Playwright QA | `~/claudedocs/articles/harness-design-long-running-apps.md` |

### internals/ — reverse-engineered harness source analysis
**Caveat: `internals/` were reverse-engineered from `@anthropic-ai/claude-code`
v2.1.88 (see each file's header). They describe design, not the current public
API, and may lag the installed CLI. When answering from these, flag that they
are background/architecture, not authoritative current behavior — confirm
current specifics against the official-docs files above.** Use for building
your own harness / understanding internals; combine with `articles/` for
design principles.

| Topic keywords | File path |
|----------------|-----------|
| build a harness, harness architecture/overview, how Claude Code works internally, reference implementation | `~/claudedocs/internals/index.md` |
| system prompt construction, prompt priority, cache boundary, static/dynamic split, context channels, CLAUDE.md loading | `~/claudedocs/internals/system-prompt-and-context.md` |
| tool interface, tool registration, execution pipeline, tool concurrency, permission modes/decision flow, auto-mode classifier | `~/claudedocs/internals/tool-framework.md` |
| turn loop, while(true) query loop, compaction/microcompact/autocompact, stop conditions, API streaming, error recovery | `~/claudedocs/internals/turn-loop.md` |
| hooks system internals, hook events/types, shell/prompt/agent/HTTP hooks, SSRF guard, AgentTool, fork subagent, SkillTool | `~/claudedocs/internals/hooks-and-extensibility.md` |
| model selection, adaptive thinking, fast mode, cost tracking, token budget, safety guardrails, prompt-injection mitigation, design patterns | `~/claudedocs/internals/design-patterns.md` |
| verification agent, adversarial evaluation, PASS/FAIL verdict, anti-rationalization, generator-evaluator separation | `~/claudedocs/internals/verification-agent.md` |
| compaction prompt, conversation summary, auto-mode/permission classifier, session memory, built-in agents (explore/plan/guide), cache-safe forking | `~/claudedocs/internals/internal-prompts.md` |
| tool description/prompt, layered reinforcement, bash git/PR workflow, git safety, read-before-edit, fork vs spawn, tool anti-patterns | `~/claudedocs/internals/tool-prompts.md` |
| system-reminder, mid-conversation injection, recurring attachments, error message design, permission-denial message, validation recovery, error steering | `~/claudedocs/internals/mid-conversation-injection.md` |

## Answering guidelines (delegated model)

1. **Route, don't read.** Match the topic to file path(s) from the routing
   table. Do not open the doc in the main session.
2. **Delegate the read.** Spawn a general-purpose subagent (cheap tier) with the
   matched path(s) + the user's question; it reads and returns a cited digest.
   Batch multiple matched files into one subagent.
3. **Relay + cite.** The digest is the answer; cite the source file(s).
4. **No clean match?** Read `manifest.md` yourself (it's small) or have the
   subagent consult it for a closer file. If the question is outside documented
   topics, say so honestly rather than guessing.
5. **internals/ caveat.** Those files are pinned to an older CLI (v2.1.88) —
   when answering from them, flag that they are architecture/background and may
   lag current behavior; confirm current specifics against the official-docs
   files.
6. **articles/** are broad practitioner essays, not API references — route to
   them only for pattern/design questions, not for current flag/field lookups.

## Keeping this cache current

This folder is a point-in-time snapshot. `manifest.md` records the fetch date +
CLI version and a "How to check for staleness later" recipe (compare upstream
`llms.txt` for coverage gaps; read the upstream `whats-new/` weekly changelog
dated after the fetch date for content drift; re-pull with
`curl -s https://code.claude.com/docs/en/<slug>.md`). `internals/` and
`articles/` are excluded from that official-docs refresh (external essays /
reverse-engineered from a pinned build).
