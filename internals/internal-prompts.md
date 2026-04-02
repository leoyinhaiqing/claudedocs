# Internal Prompts: Compaction, Classifier, Memory & Agents

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

The prompts Claude Code uses for internal harness operations — summarizing conversations, auto-approving tool calls, extracting session memory, and configuring built-in agents. These are prompts sent to the model for harness work, not shown to the user.

---

## How it works

### Compaction Prompts

When the context window fills up, Claude Code asks Claude to summarize the conversation. The compaction prompt is carefully engineered to preserve the right information.

**No-tools preamble** (prepended to every compaction call):
```
CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.
- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.
```

**9-section summary structure** (what the model must produce):
1. **Primary Request and Intent** — what the user wants
2. **Key Technical Concepts** — domain-specific knowledge
3. **Files and Code Sections** — with full code snippets (not summaries)
4. **Errors and Fixes** — what went wrong and how it was resolved
5. **Problem Solving** — approaches tried, what worked
6. **All User Messages** — non-tool-result messages preserved
7. **Pending Tasks** — what's still incomplete
8. **Current Work** — precise description with recent context
9. **Optional Next Step** — with direct quotes from conversation

**Two-block output format:** The model first writes `<analysis>` (thinking about what to keep), then `<summary>` (the actual summary). The analysis block is **stripped before storage** — it exists only to improve summary quality through chain-of-thought.

**Three compaction variants:**
- **Full compact** — summarize entire conversation
- **Partial compact (recent only)** — summarize only recent messages, earlier ones kept intact
- **Partial compact (up to boundary)** — summary will precede newer messages the model won't see; replaces "Current Work" with "Work Completed"

**Cache-safe forking:** Compaction uses the exact same system prompt, tools, and conversation prefix as the parent conversation. This means the cached prefix is reused — the only new tokens are the compaction prompt itself.

### Auto-Mode Classifier

The auto-mode permission system uses a **two-stage classifier** to decide whether to allow or block tool calls without user confirmation.

**Stage 1 — Fast decision** (64 max tokens):
- Suffix: "Err on the side of blocking. `<block>` immediately."
- Output: `<block>yes</block>` or `<block>no</block>`
- If allowed → tool executes immediately
- If blocked → escalates to Stage 2

**Stage 2 — Reasoning** (4096 max tokens):
- Suffix: "Review the classification process carefully... Use `<thinking>` before responding with `<block>`."
- Full chain-of-thought reasoning allowed
- Output: `<block>yes</block><reason>one short sentence</reason>` or `<block>no</block>`

**Classifier context includes:**
- Conversation transcript (JSONL or compact format)
- CLAUDE.md content as user configuration context
- User-provided allow/deny rules interpolated into the template
- PowerShell-specific deny guidance (when applicable)

**PowerShell safety rules** (appended when PowerShell auto-mode is enabled):
- Download-and-execute patterns (`iex (iwr ...)`) classified same as `curl | bash`
- Destructive operations (`Remove-Item -Recurse -Force`) classified same as `rm -rf`
- Persistence modifications (`$PROFILE`, `Register-ScheduledTask`, registry Run keys) classified same as `.bashrc` edits
- Elevation patterns (`Start-Process -Verb RunAs`, `-ExecutionPolicy Bypass`) classified as security weakening

**Configuration:** Temperature 0 (deterministic), prompt caching with 1-hour TTL.

### Session Memory Extraction

When enabled, the system periodically extracts structured notes from the session to preserve key information across compactions and session boundaries.

**10-section template:**

| Section | Purpose |
|---------|---------|
| Session Title | 5-10 word distinctive title, info-dense |
| Current State | What's being worked on now, pending tasks, next steps |
| Task Specification | What the user asked to build, design decisions |
| Files and Functions | Important files, what they contain, why relevant |
| Workflow | Bash commands, execution order, output interpretation |
| Errors & Corrections | Errors and fixes, failed approaches to avoid |
| Codebase Documentation | System components, how they fit together |
| Learnings | What worked, what didn't, what to avoid |
| Key Results | Exact output if user asked for specific results |
| Worklog | Step-by-step terse summary of what was done |

**Update rules:**
- Preserve section headers and italic descriptions
- Only update content below descriptions
- Per-section limit: ~2000 tokens
- Total limit: 12000 tokens
- "Current State" must reflect most recent work (critical for continuity)
- Write detailed, info-dense content with file paths, function names, error messages, exact commands

### Built-in Agent Prompts

Each built-in agent has a tailored system prompt. Here are the key designs beyond the general-purpose and verification agents (covered in their own files):

**Explore Agent** (`Explore` type, model: `haiku` for external, `inherit` for internal):
- Identity: "file search specialist"
- Read-only — cannot create, modify, or delete files
- Emphasizes speed: parallel tool calls, efficient searching
- Adapts to available tools (embedded search vs Glob/Grep)
- Omits CLAUDE.md (read-only agent doesn't need commit/lint guidelines)

**Plan Agent** (`Plan` type, model: `inherit`):
- Identity: "software architect and planning specialist"
- Read-only with same restrictions as Explore
- 4-step process: understand requirements → explore thoroughly → design solution → detail the plan
- Must end output with "Critical Files for Implementation" listing 3-5 files
- Applies an "assigned perspective" when exploring (allows callers to frame the analysis)

**Claude Code Guide Agent** (`claude-code-guide` type, model: `haiku`, permission mode: `dontAsk`):
- Answers questions about Claude Code features, Agent SDK, and Claude API
- Fetches documentation from official URLs:
  - Claude Code docs: `https://code.claude.com/docs/en/claude_code_docs_map.md`
  - Agent SDK/API docs: `https://platform.claude.com/llms.txt`
- **Dynamic prompt augmentation:** Appends the user's current configuration at runtime — custom skills, agents, MCP servers, and settings.json content. This means the guide knows what the user has installed.
- Only enabled in non-SDK entrypoints (disabled when Claude Code is used as a library)

**Statusline Setup Agent** (`statusline-setup` type, model: `sonnet`):
- Configures terminal status line by extracting PS1 from shell config files
- Reads: ~/.zshrc, ~/.bashrc, ~/.bash_profile, ~/.profile
- Converts PS1 escape sequences to shell commands
- Tools limited to Read and Edit only

### Agent Design Patterns

| Agent | Model | Tools | Read-only? | CLAUDE.md? | Background? |
|-------|-------|-------|-----------|------------|-------------|
| General-purpose | default | all | No | Yes | No |
| Explore | haiku/inherit | search+read | Yes | No | No |
| Plan | inherit | search+read | Yes | No | No |
| Verification | inherit | search+read+bash | Yes (project) | No | Yes |
| Claude Code Guide | haiku | search+read+web | N/A | Yes | No |
| Statusline Setup | sonnet | Read, Edit | No | No | No |

Key patterns:
- **Read-only agents omit CLAUDE.md** — they don't need commit/lint/PR guidelines, and omitting it saves context
- **Specialized agents use cheaper models** — Explore and Guide use Haiku because they don't need reasoning depth
- **Tool restrictions enforce role boundaries** — removing FileEdit/FileWrite from Explore is more reliable than prompt instructions alone
- **Dynamic prompt augmentation** — Guide agent's prompt changes based on user's installed extensions

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/services/compact/prompt.ts` | Compaction prompts (BASE_COMPACT_PROMPT, PARTIAL variants, NO_TOOLS preamble) |
| `src/utils/permissions/yoloClassifier.ts` | Auto-mode classifier (two-stage XML, PowerShell rules) |
| `src/services/SessionMemory/prompts.ts` | Session memory template and update rules |
| `src/tools/AgentTool/built-in/exploreAgent.ts` | Explore agent definition |
| `src/tools/AgentTool/built-in/planAgent.ts` | Plan agent definition |
| `src/tools/AgentTool/built-in/generalPurposeAgent.ts` | General-purpose agent definition |
| `src/tools/AgentTool/built-in/claudeCodeGuideAgent.ts` | Guide agent with dynamic prompt |
| `src/tools/AgentTool/built-in/statuslineSetup.ts` | Statusline setup agent |
| `src/tools/AgentTool/builtInAgents.ts` | Agent registration and feature gates |

### Design lessons for your own harness

1. **Strip reasoning from stored output.** Compaction uses `<analysis>` for chain-of-thought quality but strips it before storage — paying for better thinking without inflating the stored summary.
2. **Two-stage classification.** Fast reject (64 tokens) handles the easy cases cheaply; expensive reasoning (4K tokens) only fires when needed.
3. **Session memory as structured notes.** A fixed template with section limits prevents the model from writing sprawling narratives. Per-section caps (2000 tokens) force prioritization.
4. **Match agent model to task.** Haiku for search/guide (fast, cheap), inherit for planning/verification (needs reasoning depth), Sonnet for configuration (balanced).
5. **Omit irrelevant context from specialized agents.** Read-only agents don't need CLAUDE.md commit guidelines. This saves context and prevents confusion.
6. **Cache-safe compaction.** Reuse the parent conversation's system prompt and tools so the cached prefix hits. Don't create a separate "compaction system prompt."
7. **Enforce tool restrictions at the framework level.** Listing tools in `disallowedTools` is more reliable than telling the agent "don't use FileEdit" in the prompt. Use both, but trust the framework enforcement.
