# NOTES.md

*Personal notes — things I (the repo owner) figured out while using Claude Code, that aren't quite the same as the official docs or the reverse-engineered internals. These are my own working understanding, written for me. They may be opinionated, incomplete, or wrong.*

---

## Context construction: where instructions actually come from

I kept being surprised by what Claude already "knew" at session start. Turns out the harness has a precise loading model and it's worth understanding once.

### The two-channel design

Context reaches the model through **two separate channels** that merge at API call time:

1. **System prompt channel** — globally cached across all users. Static identity, tool guidance, safety, output style. Plus a dynamic half (git status, env info, etc.) that's per-session-cached.
2. **User context channel** — injected as a fake first user message wrapped in `<system-reminder>` tags. This is where `CLAUDE.md` and `.claude/rules/*.md` go.

**Why split?** Different users have different `CLAUDE.md`. If it lived in the system prompt, it would fragment the globally-cached prefix. Putting per-user content in message history keeps the shared system prompt cacheable.

When you start a session you can literally see this channel — the very first user-side block in the conversation looks like:

```xml
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
Codebase and user instructions are shown below. Be sure to adhere to these
instructions. IMPORTANT: These instructions OVERRIDE any default behavior
and you MUST follow them exactly as written.

Contents of <project>/CLAUDE.md (project instructions, checked into the codebase):
... CLAUDE.md body ...

Contents of <project>/.claude/rules/<rule>.md (project instructions, checked into the codebase):
... rule body ...

# currentDate
Today's date is YYYY-MM-DD.

      IMPORTANT: this context may or may not be relevant to your tasks.
      You should not respond to this context unless it is highly relevant.
</system-reminder>
```

The "OVERRIDE any default behavior" framing is constant — it's in the source as `MEMORY_INSTRUCTION_PROMPT`. The "may or may not be relevant" footer is intentional weakening so the model doesn't reflexively *respond* to the injection but does *obey* it when relevant. Two carefully calibrated instructions.

### The discovery walk (root → CWD)

At session start, `getMemoryFiles()` walks from filesystem root down to CWD and collects files at every level. Order:

```
1. /etc/claude-code/CLAUDE.md              ← Managed (admin policy)
2. /etc/claude-code/rules/*.md             ← Managed rules
3. ~/.claude/CLAUDE.md                     ← User global
4. ~/.claude/rules/*.md                    ← User global rules
5. For each dir from root → CWD:
   a. <dir>/CLAUDE.md                      ← Project
   b. <dir>/.claude/CLAUDE.md              ← Project
   c. <dir>/.claude/rules/*.md             ← Project rules (recursive)
   d. <dir>/CLAUDE.local.md                ← Local (gitignored)
```

Files are loaded in this order but the model pays more attention to whichever comes **last** (closer to CWD). So a deeply-nested `.claude/rules/foo.md` outweighs `~/.claude/CLAUDE.md` in priority.

`.claude/rules/` is walked **recursively** — subdirectories are followed, all `.md` files at any depth are eligible. The filename doesn't matter; only the `.md` extension and the location under a `rules/` directory.

Caps: ~40,000 chars per entrypoint (`MAX_MEMORY_CHARACTER_COUNT`), `@include` directive supported for nested loading.

### Conditional vs unconditional rules

This is the part most people miss. When `processMdRules()` reads a rule file, it checks the YAML frontmatter. Two outcomes:

- **No `paths:` field** → unconditional rule, loaded eagerly at session start, injected with everything else.
- **Has `paths:` field** → conditional rule, **NOT** loaded at session start. Lazy-loaded mid-conversation when the model Reads or Edits a file matching the glob pattern.

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "tests/**/*.test.ts"
---

# API Development Rules
...
```

**Important — easy mistake**: the frontmatter key is `paths:`, **not** `globs:`. The internal data structure stores the parsed result on a field called `file.globs`, which is what tripped me up when I first read the source. If you write `globs:` in your frontmatter, it will be silently ignored and the rule will be treated as unconditional (loaded eagerly every session).

The conditional-rule mechanism is powerful: you can have a rule file with `paths: ["**/test_*.py"]` that only appears in context when the model touches a test file. Lean startup context, targeted just-in-time guidance.

### Recurring re-injection

Beyond startup, the harness re-injects context periodically (`src/utils/attachments.ts`):

| Content | Cadence | Trigger |
|---|---|---|
| Todo reminders | Every 10 turns | Todo tool exists, hasn't been used recently |
| Task reminders | Every 10 turns | Task tools exist |
| Memory file refreshes | Up to 5 per turn | With "created N days ago" staleness annotations |
| Plan mode context | Every 5 turns | Plan mode active |
| Skill discovery | Per turn | Skill system enabled |
| Conditional rules | On matching file access | `processConditionedMdRules()` |

Hooks also emit `<system-reminder>` wrapped output — that's how `SubagentStop` hooks reach the parent session.

### Where to put what

Decision table for cross-cutting instructions:

| Use case | Right home | Why |
|---|---|---|
| Cross-cutting invariants (always in context) | `.claude/rules/*.md` (no frontmatter) | Auto-loaded with override framing |
| File-type-specific instructions | `.claude/rules/*.md` with `paths:` frontmatter | Lazy-loaded only when relevant |
| Reusable workflow content | `.claude/skills/<name>/SKILL.md` | Loaded on demand or via an agent's `skills:` frontmatter |
| Per-agent procedures | `.claude/agents/<agent>.md` | Loaded only when that specific agent runs |
| Human-only design docs | `docs/` or any non-`.claude/` folder | Honest about who reads it |
| Project-wide context for humans + Claude | `CLAUDE.md` or `.claude/CLAUDE.md` | Auto-loaded, project-wide |

The tradeoff: `.claude/rules/*.md` is the most aggressive — it's injected with "OVERRIDE any default behavior" framing into every session. Skills are opt-in per agent. CLAUDE.md is a middle ground (project-wide but not override-framed).

### Cross-references

Every fact above is in the official docs or the internals notes — this section is just my synthesis:

- `~/claudedocs/reference-docs/memory.md` — official user-facing reference (rules directory, `paths:` frontmatter, glob patterns, symlinks, `claudeMdExcludes`, `/memory` command)
- `~/claudedocs/internals/system-prompt-and-context.md` — reverse-engineered internals (two-channel design, cache scoping, section registry, `MEMORY_INSTRUCTION_PROMPT` constant)
- `~/claudedocs/internals/mid-conversation-injection.md` — `<system-reminder>` pattern deep-dive
- Source: `src/utils/claudemd.ts` — `getMemoryFiles()`, `processMdRules()`, `processConditionedMdRules()`

---

## Recognized `.claude/` subdirectories — the definitive list

Anything I see under `.claude/` that isn't on this list is invisible to the auto-loaders. Useful for spotting DIY conventions in the wild.

### Loaded by `loadMarkdownFilesForSubdir` (from `src/utils/markdownConfigLoader.ts:29-36`)

```typescript
export const CLAUDE_CONFIG_DIRECTORIES = [
  'commands',         // slash commands
  'agents',           // subagent definitions
  'output-styles',    // output style definitions
  'skills',           // Skill definitions (loaded via SKILL.md)
  'workflows',        // workflow definitions
  ...(feature('TEMPLATES') ? (['templates'] as const) : []),
] as const
```

### Loaded by `claudemd.ts` (memory loader, separate code path)

- `.claude/CLAUDE.md` — project memory
- `.claude/rules/*.md` — project rules (recursive, with optional `paths:` frontmatter for conditional loading)

### Other recognized paths (handled by other modules, not auto-injected as content)

- `.claude/settings.json` / `.claude/settings.local.json` — settings
- `.claude/worktrees/` — git worktree management (`src/utils/worktree.ts`)
- `.claude/ide/` — IDE integration
- `.claude/scheduled_tasks.json` / `.lock` — cron tasks
- `.claude/hooks/` — referenced from `settings.json`, NOT auto-discovered by name; the directory name is convention only

**That's the complete list.** Anything else under `.claude/` is invisible to auto-loaders.

### `.claude/references/` and friends — DIY directories

I've seen people use `.claude/references/`, `.claude/notes/`, `.claude/docs/`, `.claude/prompts/`, etc. None of these are recognized. Files in them will never be auto-loaded into context.

Three reasons people end up doing this, in decreasing order of carefulness:

1. **Intentional human-readable archive** — design notes, API specs, decision records that *humans* read. The `.claude/` parent is incidental, often for git ignore reasons or co-location. Harmless, but the name is misleading. `docs/` or `notes/` would communicate the same thing without implying auto-load.

2. **Used with explicit `@include` from CLAUDE.md** — a careful user splits a large `CLAUDE.md` by topic and pulls them in via `@./references/api.md`. This works (the loader follows `@include` chains), but `.claude/rules/` does the same job natively without the `@include` indirection.

3. **Cargo-cult / confusion with `.claude/rules/`** — someone saw `.claude/rules/*.md` is auto-loaded and assumed any folder under `.claude/` has the same magic. These files just sit on disk doing nothing. The conventions inside are not being applied. **This is the failure mode worth catching in code review.**

### How to tell which case you're looking at

Three quick checks when you see a `.claude/references/` (or similar) in the wild:

1. **`grep -rn "@.*references" .claude/CLAUDE.md .claude/rules/`** — if any hits, it's case 2 (intentional `@include`).
2. **Run `/memory` in a session** at that project's CWD — if the references files don't appear in the loaded list, they're not in context.
3. **Read the project's own README/docs** — case 1 usually has a convention note explaining why.

If none of those check out, it's case 3 and the files are dead weight.
