# System Prompt Construction & Context Injection

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code builds the system prompt, injects context (git status, CLAUDE.md, memory), and manages prompt caching. See [index.md](index.md) for architecture overview.

---

## How it works

### Prompt Priority

Multiple prompt sources compete. The system resolves them via strict priority:

```
Priority (highest to lowest):
  0. Override prompt   -- e.g., loop mode, REPLACES everything
  1. Coordinator prompt -- multi-agent coordinator mode
  2. Agent prompt      -- if mainThreadAgentDefinition is set
     - Proactive mode: APPENDED to default (additive)
     - Normal mode: REPLACES default (substitutive)
  3. Custom prompt     -- via --system-prompt CLI flag
  4. Default prompt    -- the standard Claude Code prompt
  +  Append prompt     -- always added at the end if specified
```

### Static / Dynamic Split

The default prompt is a **string array** split by a cache boundary marker. Everything before the boundary is globally cacheable; everything after is per-session.

**Static half** (7 sections, cacheable across all users):
1. **Intro** -- identity, cyber risk instruction, URL prohibition
2. **System** -- permission modes, `<system-reminder>` tag handling, hooks guidance, auto-compression
3. **Doing Tasks** -- read-before-edit, no over-engineering, no speculative abstractions, security
4. **Actions** -- reversibility/blast-radius, destructive operation warnings, measure-twice-cut-once
5. **Using Your Tools** -- prefer dedicated tools over Bash, parallel tool calls
6. **Tone and Style** -- no emojis, file_path:line_number references
7. **Output Efficiency** -- concise, lead with answer not reasoning

**Dynamic half** uses a section registry. Each section declares cacheability:
- Cached sections compute once per session (memory prompt, env info, language, output style)
- Uncached sections recompute every turn and break the cache when they change (MCP instructions — servers connect/disconnect between turns)

### Prompt Text Excerpts

**Identity:**
```
You are an interactive agent that helps users with software engineering tasks.
Use the instructions below and the tools available to you to assist the user.
```

**Code style (key rules):**
```
- Don't add features, refactor code, or make "improvements" beyond what was asked.
- Don't add error handling, fallbacks, or validation for scenarios that can't happen.
- Don't create helpers, utilities, or abstractions for one-time operations.
  Three similar lines of code is better than a premature abstraction.
```

**Action safety:**
```
Carefully consider the reversibility and blast radius of actions...
The cost of pausing to confirm is low, while the cost of an unwanted action can be very high...
A user approving an action once does NOT mean they approve it in all contexts...
measure twice, cut once.
```

**Tool preference:**
```
Do NOT use Bash when a relevant dedicated tool is provided.
- Read instead of cat/head/tail/sed
- Edit instead of sed/awk
- Write instead of cat heredoc/echo
- Glob instead of find/ls
- Grep instead of grep/rg
```

**Output efficiency:**
```
Go straight to the point. Try the simplest approach first. Be extra concise.
Lead with the answer or action, not the reasoning.
If you can say it in one sentence, don't use three.
```

### Alternate Prompt Modes

**Simple mode** (`CLAUDE_CODE_SIMPLE`) -- ultra-minimal for testing:
```
You are Claude Code, Anthropic's official CLI for Claude.
CWD: {cwd}  Date: {date}
```

**Proactive/autonomous mode** -- lean prompt for background agents:
```
You are an autonomous agent. Use the available tools to do useful work.
```

### Cache Control Strategy

The prompt is split into blocks with different cache scopes:
- Everything before the boundary: `scope='global'` (shared across all users)
- Everything after: `scope='org'` (org-level caching)

The static half is ~60% of the prompt. Global caching means the static portion costs nearly nothing on repeated calls.

### Two Context Channels

Context reaches the model through two paths that merge at API call time:

**System context** -- appended to the system prompt:
- Git status (branch, default branch, recent commits, short status)
- Memoized once per session

**User context** -- injected as a fake first user message wrapped in `<system-reminder>`:
- CLAUDE.md contents
- Current date
- Wrapped with: "this context may or may not be relevant to your tasks"

**Why two channels?** Putting CLAUDE.md in the message history instead of the system prompt avoids fragmenting the globally cached system prompt prefix. Different users have different CLAUDE.md content, so it must live outside the shared cache.

### Git Status

Memoized, computed once per session. Runs 5 git commands in parallel (branch, default branch, status, log, user.name). Status truncated to 2000 chars. Skipped in remote sessions, non-git directories, or when git instructions are disabled.

```
This is the git status at the start of the conversation. Note that this
status is a snapshot in time, and will not update during the conversation.

Current branch: feature/my-branch
Main branch (you will usually use this for PRs): main
Git user: John Doe
Status:
M  src/file.ts
?? new-file.ts
Recent commits:
abc1234 feat: add new feature
```

### CLAUDE.md Loading

Discovered by walking from CWD to filesystem root. All levels loaded and concatenated:
1. `/etc/claude-code/CLAUDE.md` -- admin/managed
2. `~/.claude/CLAUDE.md` -- user global
3. `.claude/CLAUDE.md` -- project (checked in)
4. `.claude/rules/*.md` -- project rule files
5. `CLAUDE.local.md` -- private local

Files closer to CWD override earlier ones. Supports `@include` for nested loading. Truncated at 200 lines / 25KB per entrypoint. Prepended with: "These instructions OVERRIDE any default behavior and you MUST follow them exactly as written."

### Memory Directory System

Per-project auto-memory at `~/.local/share/claude-code/auto-memory/{project-slug}/`. The harness pre-creates directories so the model never wastes turns checking or creating them. The prompt explicitly says: "This directory already exists -- write to it directly."

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/constants/prompts.ts` | `getSystemPrompt()` builder, all section functions |
| `src/utils/systemPrompt.ts` | `buildEffectiveSystemPrompt()` priority logic |
| `src/constants/systemPromptSections.ts` | Section registry, `systemPromptSection()`, `DANGEROUS_uncachedSystemPromptSection()` |
| `src/context.ts` | `getGitStatus()`, `getSystemContext()`, `getUserContext()` |
| `src/utils/api.ts` | `splitSysPromptPrefix()`, `appendSystemContext()`, `prependUserContext()` |
| `src/memdir/memdir.ts` | `loadMemoryPrompt()`, `ensureMemoryDirExists()` |
| `src/utils/claudemd.ts` | `getClaudeMds()`, CLAUDE.md file discovery walk |
| `src/constants/cyberRiskInstruction.ts` | `CYBER_RISK_INSTRUCTION` constant |

### Section registry API

```typescript
// Computed once, cached until /clear or /compact
systemPromptSection('name', () => computeValue())

// Recomputed every turn, breaks cache when value changes — requires justification
DANGEROUS_uncachedSystemPromptSection('name', () => computeValue(), 'reason')

// Resolve all sections in parallel
resolveSystemPromptSections(sections): Promise<(string | null)[]>
```

### Context injection functions

```typescript
// System context — appended to system prompt
getSystemContext() -> { gitStatus?, cacheBreaker? }
appendSystemContext(systemPrompt, systemContext) -> string[]

// User context — prepended as fake first user message
getUserContext() -> { claudeMd?, currentDate }
prependUserContext(messages, userContext) -> Message[]
// Wraps in: <system-reminder>As you answer... # claudeMd {content} # currentDate {date}</system-reminder>
```

### Cache splitting

```typescript
splitSysPromptPrefix(prompt):
  // Finds SYSTEM_PROMPT_DYNAMIC_BOUNDARY marker
  // Returns SystemPromptBlock[] with { text, cacheScope: 'global' | 'org' | null }
```
