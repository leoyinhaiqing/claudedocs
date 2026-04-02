# Claude Code Harness Architecture

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code orchestrates an LLM into an agentic coding assistant. This document covers the architecture and key design decisions — read the linked detail files for implementation specifics.

---

## Architecture

```
User Input
  |
  v
processUserInput()          -- Slash commands, paste expansion, hooks
  |
  v
QueryEngine.submitMessage() -- Session lifecycle, transcript recording
  |
  v
buildEffectiveSystemPrompt() -- Priority-based prompt selection
  |
  v
query() loop [while(true)]  -- The core turn loop
  |
  +---> Compaction phase    -- Snip / microcompact / autocompact
  +---> API call phase      -- Stream from Claude API
  +---> Recovery phase      -- Handle PTL, max output, errors
  +---> Stop hooks phase    -- Post-turn hooks, budget checks
  +---> Tool execution      -- Dispatch tool calls, collect results
  +---> Decision phase      -- Continue or return
  |
  v
Render via Ink              -- Terminal UI (React-compatible)
```

---

## 1. System Prompt & Context

The system prompt is a **string array** split into two halves by a cache boundary marker. The static half (identity, tool guidance, coding style, action safety, output style) is globally cacheable across all users. The dynamic half uses a section registry where each section declares whether it can be cached or must recompute every turn — preventing unnecessary cache busting.

Context reaches the model through **two separate channels**: system context (git status) is appended to the system prompt; user context (CLAUDE.md, date) is injected as a `<system-reminder>` in a fake first user message. This split exists because user-specific content in the message history doesn't fragment the system prompt cache.

CLAUDE.md files are discovered by walking from CWD to the filesystem root, loading a 5-level hierarchy (admin, user, project, rules, local). A memory directory system pre-creates per-project directories so the model never wastes turns on `mkdir`.

**Key numbers:** 7 static sections, 10+ dynamic sections, 5-level CLAUDE.md hierarchy, 2000-char git status cap, 200-line/25KB memory file truncation.

> Detail: [system-prompt-and-context.md](system-prompt-and-context.md) — prompt text excerpts, section registry API, context channel mechanics, CLAUDE.md loading, memory system.

---

## 2. Tool Framework & Permissions

Each tool declares its own input schema (Zod), permission check, execution logic, and prompt description. A `buildTool()` factory provides sensible defaults so most tools only override what they need. Tools self-describe: the prompt function receives context about available tools and agents, so descriptions adapt to what's active.

The tool execution pipeline has **8 phases**: lookup → input validation → pre-tool hooks → permission check → execution → result processing → post-tool hooks → result assembly. Large results (>50KB) are persisted to disk and replaced with a preview + file reference, keeping the context window manageable.

Concurrency is handled by partitioning tool calls: **read-only tools run in parallel** (up to 10), write tools run alone. A streaming executor can start tools before the model finishes generating, executing concurrency-safe tools as they stream in.

Permissions use a **mode + rules** system. Six modes (default/acceptEdits/bypass/plan/dontAsk/auto) set the baseline; granular rules allow/deny/ask per tool or pattern (e.g., `Bash(git *)`). Rules come from 8 sources with a strict priority order, so enterprise policies always override user settings.

**Key numbers:** 44+ built-in tools, 8 execution phases, 50KB result persistence threshold, 10 max concurrent tools, 6 permission modes, 8 rule sources.

> Detail: [tool-framework.md](tool-framework.md) — Tool interface, execution pipeline diagram, result persistence mechanics, concurrency modes, permission decision flow and types.

---

## 3. Turn Loop & Compaction

The core loop is an infinite `while(true)` with **immutable state between iterations** — each `continue` creates a fresh State object and records *why* it continued (transition tracking), making the loop debuggable despite 7 different continue paths. Each iteration runs 6 phases: compaction, API call, recovery, stop hooks, tool execution, decision.

The loop handles errors through **progressive recovery**: context collapse drain (cheapest) → reactive compact → max output token escalation (8k→64k) → retry with nudge → surface error (last resort). Each path has a guard to prevent infinite retries.

Context is managed through **5 compaction layers** applied in order of increasing aggressiveness: snip (fast pruning) → microcompact (clear old tool results while preserving structure for cache) → context collapse (staged removal) → autocompact (full LLM summary via forked subagent) → reactive compact (emergency recovery on 413 errors).

**Key numbers:** 6 phases per iteration, 7 stop conditions, 7 continue conditions, 5 compaction layers, 5 recovery paths.

> Detail: [turn-loop.md](turn-loop.md) — Loop State type, phase-by-phase diagram, stop/continue condition lists, API streaming mechanics, compaction layer details, token budget carry-over.

---

## 4. Hooks & Extensibility

Hooks are user-configurable actions that fire in response to **25 harness events** spanning the full lifecycle (pre/post tool use, session start/end, compaction, file changes, subagent lifecycle). Four execution mechanisms: shell commands, single-turn LLM prompts (Haiku, 30s timeout), multi-turn LLM agents (50-turn limit), and HTTP calls (with SSRF protection, URL allowlists, header injection prevention).

The **AgentTool** spawns subagents — isolated Claude instances with their own context windows. Three built-in types: general-purpose, Explore (read-only, fast), Plan (read-only, no edits). Fork subagents run in the background to keep tool output out of the main context. A feature-gated verification agent provides adversarial quality checks on non-trivial changes.

**Skills** are markdown files with frontmatter that define specialized capabilities. They combine a prompt template with optional hooks, model preferences, and tool configurations. Discovered from 5 sources (bundled, user, project, managed, plugin) and deduplicated. Users invoke them via `/skill-name` syntax.

**Key numbers:** 25 hook events, 4 hook types, 50-turn agent hook limit, 10-min HTTP timeout, 3 built-in agent types, 17 bundled skills, 5 skill sources.

> Detail: [hooks-and-extensibility.md](hooks-and-extensibility.md) — Hook event list, execution mechanisms, hook response format, AgentTool input, subagent prompts, skill loading.
> Deep dive: [verification-agent.md](verification-agent.md) — Full adversarial system prompt, anti-rationalization list, type-specific strategies, evidence format, trigger mechanisms.

---

## 5. Design Patterns, Model Config & Safety

Model selection follows a **5-level priority** (session override → CLI flag → env var → settings → tier-based default). Extended thinking supports three modes: adaptive (model decides), enabled with budget, disabled. Thinking blocks are protected through tool trajectories — they can't be the last content block.

Safety uses an **8-layer architecture**: system prompt behavioral guidance → tool-level permission rules → pre/post-tool hooks → enterprise policy settings → Zod input validation → result truncation → SSRF protection → rate limiting. Enterprise policies have highest priority and can lock down hooks, skills, agents, MCP, and permission bypass.

Key architectural patterns: immutable loop state (prevents mutation bugs), cache boundary splitting (static prompt globally cached, ~60% cost savings), memoized context (computed once per session), branded types for SystemPrompt (type-safe at compile time), dead code elimination via `feature()` gates, user context as fake first message (preserves cache), transition tracking on every `continue` (debuggability).

**Key numbers:** 5 model priority levels, 3 thinking modes, 8 safety layers.

> Detail: [design-patterns.md](design-patterns.md) — Model selection priority, ThinkingConfig, fast mode, cost tracking metrics, enterprise policy controls, pattern rationale.

---

## 6. Internal Prompts

Claude Code uses carefully engineered prompts for its own internal operations. The **compaction prompt** uses a 9-section summary structure with a stripped `<analysis>` block for chain-of-thought quality. The **auto-mode classifier** uses a two-stage approach: a fast 64-token reject, then a 4K-token reasoning stage only when needed. **Session memory** uses a 10-section template with per-section token limits to force concise, structured notes. Each **built-in agent** has a tailored prompt — read-only agents omit CLAUDE.md, specialized agents use cheaper models, and tool restrictions enforce role boundaries at the framework level.

> Detail: [internal-prompts.md](internal-prompts.md) — Compaction prompt structure, classifier two-stage design, session memory template, built-in agent prompt patterns.

---

## 7. Tool Prompt Engineering

Tool descriptions are as important as the system prompt for controlling model behavior. Claude Code uses **layered reinforcement** — the same guidance at three levels (system prompt, Bash tool description, dedicated tool description). The Bash tool prompt is a mini-harness: it embeds the full git commit protocol (4-step with parallel commands), PR creation protocol (3-step), git safety rules (NEVER directives), sandbox restrictions, and background task guidance. Each tool's `prompt()` function is **context-aware** — descriptions adapt to available tools, fork mode, sandbox state.

**Key numbers:** Bash prompt ~300 lines, full git commit protocol embedded, full PR creation protocol embedded.

> Detail: [tool-prompts.md](tool-prompts.md) — Layered reinforcement pattern, Bash tool workflow protocols, FileEdit read-first enforcement, Agent fork/spawn guidance, tool anti-patterns.

---

## 8. Mid-Conversation Injection & Error Engineering

Claude Code uses `<system-reminder>` tags to inject context mid-conversation without breaking the prompt cache. Five injection patterns: user context as fake first message, recurring attachments (every N turns), hook output, tool result warnings, and side questions. Error messages are **steering, not just diagnostics** — permission denials allow legitimate workarounds while blocking malicious ones, rejections prime the model to watch for user preferences, and validation errors include exact recovery instructions.

**Key numbers:** 5 injection patterns, 10-turn reminder intervals, 5-turn plan mode intervals.

> Detail: [mid-conversation-injection.md](mid-conversation-injection.md) — `<system-reminder>` patterns, error/denial message text, recovery instructions, recurring attachment intervals.
