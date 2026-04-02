# Tool Prompt Engineering

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code crafts tool descriptions to shape model behavior. Tool prompts are as important as the system prompt — they're the model's primary reference for how to use each tool. See [index.md](index.md) for architecture overview.

---

## How it works

### Layered Reinforcement

Claude Code uses a three-layer pattern to steer tool usage. The same guidance appears at multiple levels:

| Layer | Example: "Use Read instead of cat" |
|-------|-------------------------------------|
| System prompt | "Do NOT use Bash when a relevant dedicated tool is provided. To read files use Read instead of cat, head, tail, or sed" |
| Bash tool description | "IMPORTANT: Avoid using this tool to run `cat`, `head`, `tail`... Instead, use the appropriate dedicated tool" |
| Read tool description | "Reads a file from the local filesystem. You can access any file directly by using this tool." |

This isn't redundant — each layer catches a different failure mode. The system prompt sets the general principle. The Bash tool intercepts when the model reaches for Bash anyway. The Read tool makes itself the obvious choice by emphasizing capability.

### Context-Aware Descriptions

Tool prompts aren't static. The `prompt()` function receives context about available tools, agents, and permissions. Examples:

- **Agent tool** adapts to fork mode vs subagent mode — entirely different "when to use" and "when NOT to use" sections
- **Bash tool** includes sandbox restrictions only when sandboxing is enabled
- **Grep/Glob** guidance in the system prompt adapts to whether embedded search tools are available (ant-native builds use `find`/`grep` via Bash instead)

### Self-Describing Tools

Each tool's description contains its own usage manual. This is more reliable than putting all tool guidance in the system prompt because:
1. Tool descriptions are always adjacent to the tool call in the API request
2. The model attends to tool descriptions when deciding which tool to call
3. Tool-specific guidance doesn't compete with other system prompt instructions for attention

---

## Key Tool Descriptions

### Bash Tool (the most complex)

The Bash tool prompt is essentially a mini-harness. It includes:

**Tool preference hierarchy:**
```
IMPORTANT: Avoid using this tool to run `find`, `grep`, `cat`, `head`, `tail`,
`sed`, `awk`, or `echo` commands, unless explicitly instructed. Instead, use the
appropriate dedicated tool:
 - File search: Use Glob (NOT find or ls)
 - Content search: Use Grep (NOT grep or rg)
 - Read files: Use Read (NOT cat/head/tail)
 - Edit files: Use Edit (NOT sed/awk)
 - Write files: Use Write (NOT echo >/cat <<EOF)
 - Communication: Output text directly (NOT echo/printf)
```

**Command execution guidance:**
- Working directory persists between commands, but shell state does not
- Always quote file paths with spaces
- Use absolute paths to maintain CWD
- Timeout parameter (up to 10 minutes)
- `run_in_background` for long-running operations — don't sleep, don't poll, you'll be notified

**Multi-command batching rules:**
- Independent commands: multiple parallel Bash calls in a single message
- Sequential dependencies: chain with `&&`
- Don't care about failure: use `;`
- Do NOT use newlines to separate commands

**Full git commit protocol** (4-step, embedded in tool description):
1. Run `git status`, `git diff`, `git log` in parallel
2. Analyze changes, draft commit message (summarize the "why" not "what")
3. Stage specific files (never `git add -A`), create commit with HEREDOC, verify with `git status`
4. On pre-commit hook failure: fix and create NEW commit (never amend — the previous commit didn't happen)

Key safety rules embedded:
- NEVER update git config
- NEVER run destructive commands unless explicitly requested
- NEVER skip hooks (`--no-verify`) unless explicitly asked
- NEVER force push to main/master
- Always create NEW commits rather than amending
- NEVER commit unless explicitly asked

**Full PR creation protocol** (3-step):
1. Run `git status`, `git diff`, `git log`, check remote tracking
2. Draft title (<70 chars) and body with Summary + Test plan sections
3. Create branch if needed, push with `-u`, create PR via `gh pr create` with HEREDOC body

**Sandbox section** (when enabled):
- Filesystem read/write allowlists and denylists
- Network restrictions
- When to use `dangerouslyDisableSandbox`
- Use `$TMPDIR` instead of `/tmp`

### FileEdit Tool

**The read-first enforcement:**
```
You must use your `Read` tool at least once in the conversation before editing.
This tool will error if you attempt an edit without reading the file.
```

This is enforced at both the prompt level AND the execution level — the tool checks whether the file has been read and returns an error if not. The prompt also explains the indentation preservation requirement when editing from Read output.

**Uniqueness requirement:**
```
The edit will FAIL if `old_string` is not unique in the file. Either provide a
larger string with more surrounding context to make it unique or use `replace_all`.
```

### FileWrite Tool

**Prefer Edit over Write:**
```
Prefer the Edit tool for modifying existing files — it only sends the diff.
Only use this tool to create new files or for complete rewrites.
```

**Anti-documentation-creation:**
```
NEVER create documentation files (*.md) or README files unless explicitly requested.
```

### Agent Tool

**When NOT to use** (prevents over-delegation):
```
- If you want to read a specific file path, use the Read tool instead
- If you are searching for a specific class definition, use Glob instead
- If you are searching for code within 2-3 files, use Read instead
```

**Foreground vs background decision:**
```
Use foreground (default) when you need the agent's results before you can proceed.
Use background when you have genuinely independent work to do in parallel.
```

**Result visibility warning:**
```
The result returned by the agent is not visible to the user. To show the user
the result, you should send a text message with a concise summary.
```

**Prompt authoring guidance** (how to brief the subagent):
```
Brief the agent like a smart colleague who just walked in — it hasn't seen this
conversation, doesn't know what you've tried, doesn't understand why this task matters.
- Explain what you're trying to accomplish and why
- Describe what you've already learned or ruled out
- Give enough context that the agent can make judgment calls
```

**Fork mode** (when enabled, replaces subagent guidance):
```
Fork yourself (omit subagent_type) when intermediate output isn't worth keeping
in context. Criterion: "will I need this output again?"
If you ARE the fork — execute directly; do not re-delegate.
```

### Grep Tool

**Exclusivity enforcement:**
```
ALWAYS use Grep for search tasks. NEVER invoke `grep` or `rg` as a Bash command.
The Grep tool has been optimized for correct permissions and access.
```

### Glob Tool

**Agent escalation hint:**
```
When you are doing an open ended search that may require multiple rounds of
globbing and grepping, use the Agent tool instead.
```

### AskUserQuestion Tool

**Plan mode anti-pattern:**
```
Do NOT use this tool to ask "Is my plan ready?" or "Should I proceed?" — use
ExitPlanMode for plan approval. IMPORTANT: Do not reference "the plan" in your
questions because the user cannot see the plan in the UI until you call ExitPlanMode.
```

### Skill Tool

**Blocking requirement:**
```
When a skill matches the user's request, this is a BLOCKING REQUIREMENT: invoke
the relevant Skill tool BEFORE generating any other response about the task.
NEVER mention a skill without actually calling this tool.
```

---

## Design Lessons

1. **Layer reinforcement across system prompt and tool descriptions.** The model will sometimes ignore the system prompt. Having the same guidance in the tool description catches those cases.

2. **Embed complete workflows in tool descriptions.** The Bash tool contains the full git commit and PR protocols — not just "use git carefully" but step-by-step with exact commands. This eliminates ambiguity.

3. **Enforce constraints at both prompt and execution levels.** "Must read before editing" is in the FileEdit prompt AND checked by the tool. Belt and suspenders.

4. **Include anti-patterns in tool descriptions.** "Do NOT use this tool to ask 'Is my plan ready?'" is more effective than only saying what to do. Name the specific mistake.

5. **Guide delegation decisions.** The Agent tool explains when NOT to use it (simple file reads) and the Glob tool hints when TO escalate to Agent (open-ended searches). Tools help the model choose between them.

6. **Make tool descriptions context-aware.** The Agent tool adapts its entire guidance based on fork mode. The Bash tool includes sandbox rules only when sandboxing is on. Static descriptions miss important context.

7. **Include result visibility warnings.** "The agent's result is not visible to the user" prevents the model from assuming the user saw what the subagent found.

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/tools/BashTool/prompt.ts` | Bash description (~300 lines with git/PR/sandbox protocols) |
| `src/tools/FileEditTool/prompt.ts` | Edit description with read-first and uniqueness rules |
| `src/tools/FileReadTool/prompt.ts` | Read description with format types (PDF, images, notebooks) |
| `src/tools/FileWriteTool/prompt.ts` | Write description with prefer-Edit and anti-docs rules |
| `src/tools/AgentTool/prompt.ts` | Agent description with fork/spawn/delegation guidance |
| `src/tools/GrepTool/prompt.ts` | Grep description with exclusivity enforcement |
| `src/tools/GlobTool/prompt.ts` | Glob description with agent escalation hint |
| `src/tools/AskUserQuestionTool/prompt.ts` | Question tool with plan mode anti-patterns |
| `src/tools/SkillTool/prompt.ts` | Skill tool with blocking requirement |
| `src/tools/WebFetchTool/prompt.ts` | WebFetch with MCP preference and copyright handling |
