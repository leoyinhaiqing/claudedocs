# Claude Docs Manifest

Local cache of Claude Code official documentation.
Source: https://code.claude.com/docs/ (raw markdown, per-page `<slug>.md`)
Fetched: 2026-07-13 (Claude Code CLI v2.1.203)

## How to check for staleness later

This cache is a point-in-time snapshot of the official pages listed below. To
re-check whether it has fallen behind upstream — without re-discovering the
method each time:

1. **Coverage gap** — fetch `https://code.claude.com/docs/llms.txt` (the
   authoritative upstream page index) and compare its page list against the
   files below. Upstream pages absent here are coverage gaps.
2. **Content drift** — read the upstream `whats-new/` weekly changelog pages
   (`https://code.claude.com/docs/en/whats-new/2026-wNN.md`) dated *after* the
   `Fetched:` date above. Each week enumerates feature / setting / flag /
   command changes; any entry touching a topic below means that cached page is
   behind.
3. **Refresh** — re-pull is reproducible: the cache is Mintlify raw markdown,
   so `curl -s https://code.claude.com/docs/en/<slug>.md > <local file>` for
   each page below, then bump the `Fetched:` date + CLI version on the line
   above. Always refresh the whole set together so that single date stays
   accurate for every file.

`internals/` and `articles/` are NOT part of this official-docs snapshot and
are excluded from the checks above: `articles/` are external engineering blog
posts / tweets; `internals/` are reverse-engineered from a pinned CLI version
and refresh only by re-deriving against a newer build.

## agents-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `sub-agents.md` | Subagent definition & config | Frontmatter fields, tool restrictions, permission modes, hooks in agents, memory, skill preloading, resume pattern |
| `agent-teams.md` | Multi-session agent teams | Team lead/teammate architecture, task list, mailbox, parallel work patterns, comparison with subagents |
| `headless.md` | Programmatic / Agent SDK | `-p` flag, `--output-format json/stream-json`, `--allowedTools`, `--continue`/`--resume`, system prompt flags |
| `workflows.md` | Dynamic workflows | Claude-authored orchestration scripts over many background subagents; `Workflow` tool, phases, `pipeline`/`parallel`, `ultracode` opt-in |
| `worktrees.md` | Parallel sessions with worktrees | Isolated sessions in git worktrees, `isolation: worktree`, `EnterWorktree`, auto-cleanup |
| `agent-view.md` | Agent view / manage sessions | `claude agents` dashboard, attach/detach, background execution, nested-agent tree |
| `agents.md` | Run agents in parallel | Launching and coordinating multiple parallel agent sessions |
| `routines.md` | Routines (scheduled cloud agents) | Cron / GitHub-event / API-triggered cloud agents, `/schedule`, tokened `/fire` endpoint |
| `remote-control.md` | Remote control | Continue local sessions from any device, mobile push, permission relay |

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
| `permission-modes.md` | Permission modes | `auto` / `manual` / `plan` / `acceptEdits` / `bypassPermissions`; classifier-based auto mode |
| `fast-mode.md` | Fast mode | High-speed Opus configuration, `/fast`, pricing tier |
| `context-window.md` | Context window | Context budget, compaction, microcompact, context management |
| `sandboxing.md` | Sandboxed Bash | Configuring the sandboxed Bash tool, sandbox rules, credentials |
| `sandbox-environments.md` | Sandbox environments | Choosing a sandbox environment, OS-level isolation |
| `security-guidance.md` | Security guidance | Catch security issues as Claude writes code, security-guidance plugin |

## guides-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `output-styles.md` | Output styles | Built-in and custom output styles, frontmatter, use beyond software engineering |
| `scheduled-tasks.md` | Scheduled tasks | `/loop`, `CronCreate`/`CronList`/`CronDelete`, one-time reminders, cron expression reference |
| `terminal-config.md` | Terminal setup | Line breaks, notifications, Vim mode, terminal optimization |
| `checkpointing.md` | Checkpointing | Track/rewind/summarize edits and conversation state |
| `best-practices.md` | Best practices | Context management, CLAUDE.md authoring, prompting patterns, parallel sessions, automation, common failure patterns |
| `channels.md` | Channels (push events into a running session) | Channel plugins, Telegram / iMessage setup, permission relay, enterprise controls (`channelsEnabled` / `allowedChannelPlugins`) |
| `channels-reference.md` | Channels reference | Channel plugin schema, event types, permission-relay capability, enterprise controls |
| `code-review.md` | Code review | `/code-review` skill, effort levels, `--comment` / `--fix`, subagent diff review |
| `ultrareview.md` | Ultrareview | Cloud multi-agent adversarial code review, CI subcommand |
| `artifacts.md` | Artifacts | `Artifact` tool, publish interactive pages to claude.ai, update-in-place |
| `goal.md` | Goal-directed work | `/goal`, autonomous multi-turn work toward a verifiable completion condition |
| `interactive-mode.md` | Interactive mode | Keyboard shortcuts, input modes, session interaction |
| `keybindings.md` | Keybindings | Customize keyboard shortcuts, `~/.claude/keybindings.json`, chords |
| `statusline.md` | Status line | Customize the status line, `statusLine` setting, refresh interval |

## plugins-docs/

| File | Topic | Key content |
|------|-------|-------------|
| `plugins.md` | Create plugins | Plugin structure, skills/hooks/MCP in plugins, local testing, distribution |
| `plugins-reference.md` | Plugins reference | Full manifest schema, all component schemas, CLI commands, scopes, caching, debugging |

## agent-sdk-docs/

Core subset of the Agent SDK section (not the full ~34-page SDK area — see the
staleness recipe above to pull more from `agent-sdk/` on demand).

| File | Topic | Key content |
|------|-------|-------------|
| `overview.md` | Agent SDK overview | What the SDK is, when to use it, capabilities, language bindings |
| `agent-loop.md` | Agent loop | How the SDK agent loop works, turn cycle, tool execution |
| `subagents.md` | Subagents in the SDK | Defining / spawning subagents programmatically |
| `permissions.md` | SDK permissions | `canUseTool` callback, permission modes, `managedSettings` |
| `custom-tools.md` | Custom tools | Give Claude custom tools via the SDK, in-process MCP |
| `structured-outputs.md` | Structured outputs | Force schema-validated output from agents |
