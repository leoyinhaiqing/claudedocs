# Hooks, Agents & Skills

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code's extensibility mechanisms work: the hooks system, subagent orchestration, and skills framework. See [index.md](index.md) for architecture overview.

---

## How it works

### Hooks

Hooks are user-configurable actions that fire in response to harness events. They provide interception points across the full lifecycle without modifying the core harness.

**4 hook types:**
- **Shell command** -- run a shell command with configurable shell and timeout
- **Prompt** -- single-turn LLM evaluation using a small/fast model (Haiku). 30-second timeout. JSON schema validation for response.
- **Agent** -- multi-turn LLM execution via temporary hook agent. 50-turn safety limit. Uses fast model. Disallowed tools filtered. Structured output enforced.
- **HTTP** -- HTTP call with security controls: SSRF guard (blocks private/link-local IPs), URL allowlist, env var allowlist for interpolation, header injection prevention (strips CR/LF/NUL), 10-minute timeout.

**25 hook events across the lifecycle:**

| Category | Events |
|----------|--------|
| Tool lifecycle | `PreToolUse` (can block/modify), `PostToolUse` (can update output), `PostToolUseFailure` |
| Turn lifecycle | `UserPromptSubmit`, `Stop` (can block continuation), `StopFailure` |
| Session | `SessionStart`, `SessionEnd`, `Setup` |
| Permission | `PermissionDenied`, `PermissionRequest` |
| Compaction | `PreCompact`, `PostCompact` |
| Filesystem | `FileChanged`, `CwdChanged` |
| Worktree | `WorktreeCreate`, `WorktreeRemove` |
| Config | `ConfigChange`, `InstructionsLoaded` |
| Subagent | `SubagentStart`, `SubagentStop` |
| UI | `Notification`, `Elicitation`, `ElicitationResult` |
| Multi-agent | `TeamMateIdle`, `TaskCreated`, `TaskCompleted` |

**Hook sources and priority:**
```
flagSettings > policySettings > userSettings > projectSettings > localSettings > pluginHook > builtinHook
```
Enterprise `allowManagedHooksOnly` policy blocks user/project/local hooks entirely.

**Hook responses:** `{ ok: boolean, reason?: string }`. A `PreToolUse` returning `ok: false` blocks tool execution. A `Stop` returning `ok: false` prevents the turn from ending — the harness injects the error and retries.

**How the model sees hooks:** The system prompt tells the model hooks exist and to treat their feedback as coming from the user: "If you get blocked by a hook, determine if you can adjust your actions."

### Agents & Subagents

The **AgentTool** spawns isolated Claude instances with their own context windows. Subagents protect the main context from excessive tool output and enable parallel work.

**3 built-in agent types:**
- **General-purpose** -- full tool access, default
- **Explore** -- read-only tools only, fast codebase exploration
- **Plan** -- read-only, no Edit/Write, architecture planning

Custom agent definitions can be loaded from `~/.claude/agents/` or `.claude/agents/`.

**Subagent prompt:** Subagents get a distinct system prompt focused on task completion:
```
You are an agent for Claude Code. Given the user's message, use the tools
available to complete the task. Complete the task fully--don't gold-plate,
but don't leave it half-done.
```
Plus: always use absolute file paths, share relevant paths in final response, no emojis.

**Fork subagents:** When enabled, the AgentTool creates background "forks" that keep their tool output out of the main context. The system prompt prevents re-delegation: "If you ARE the fork -- execute directly; do not re-delegate."

**Verification agent** (feature-gated, not available in external builds): An adversarial verification agent for non-trivial changes (3+ file edits, backend/API changes). Read-only — can't edit project files, but can write ephemeral test scripts to `/tmp`. Independently checks implementation quality and assigns PASS/FAIL/PARTIAL verdicts backed by command output evidence. The main model cannot self-assign PARTIAL — only the verifier can. Auto-triggered after 3+ tasks completed without verification. For the full behavioral design — adversarial framing, anti-rationalization list, 11 type-specific strategies, evidence format — see [verification-agent.md](verification-agent.md).

### Skills

Skills are markdown files with frontmatter that define specialized capabilities -- a prompt template plus optional hooks, model preferences, and tool configurations.

**Discovered from 5 sources:** bundled (17 built-in), user (`~/.claude/skills/`), project (`.claude/skills/`), managed (`~/.claude/managed/.claude/skills/`), plugins. Deduplicated via `realpath()`.

**Invocation:** Users type `/skill-name`. The `SkillTool` expands the skill's prompt template and executes it. The system prompt instructs: "Use the Skill tool to execute them. Only use Skill for skills listed in its user-invocable skills section -- do not guess."

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/utils/hooks/hooksSettings.ts` | `getAllHooks()`, `getHooksForEvent()`, hook loading |
| `src/utils/hooks/hookEvents.ts` | `registerHookEventHandler()`, event emission |
| `src/utils/hooks/execCommandHook.ts` | Shell hook execution |
| `src/utils/hooks/execPromptHook.ts` | Prompt hook execution (Haiku, 30s) |
| `src/utils/hooks/execAgentHook.ts` | Agent hook execution (50-turn limit) |
| `src/utils/hooks/execHttpHook.ts` | HTTP hook execution (SSRF guard, URL allowlist) |
| `src/utils/hooks/AsyncHookRegistry.ts` | Async hook tracking and completion |
| `src/tools/AgentTool/AgentTool.tsx` | AgentTool -- subagent spawning |
| `src/tools/AgentTool/builtInAgents.ts` | Built-in agent type definitions |
| `src/skills/loadSkillsDir.ts` | Skill discovery, frontmatter parsing |
| `src/tools/SkillTool/SkillTool.ts` | Skill invocation |

### Hook type definition

```typescript
type HookCommand =
  | { type: 'command'; command: string; shell?: string; if?: string }
  | { type: 'prompt'; prompt: string; if?: string }
  | { type: 'agent'; prompt: string; if?: string }
  | { type: 'http'; url: string; if?: string }
  | { type: 'function' }  // Internal only
```

### AgentTool input

```typescript
input: {
  prompt: string           // Task description
  description: string      // Short summary (3-5 words)
  subagent_type?: string   // 'Explore', 'Plan', etc.
  model?: string           // Optional model override
  run_in_background?: boolean
  isolation?: 'worktree'   // Git worktree isolation
}
```
