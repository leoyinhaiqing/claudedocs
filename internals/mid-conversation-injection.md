# Mid-Conversation Injection & Error Message Engineering

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code injects context mid-conversation without breaking the prompt cache, and how error messages are crafted to guide the model's recovery. See [index.md](index.md) for architecture overview.

---

## How it works

### The `<system-reminder>` Pattern

Claude Code uses `<system-reminder>` tags as a **mid-conversation injection mechanism**. Content wrapped in these tags is invisible to the user (stripped from UI) but present in the message history sent to the API.

The model's system prompt explains these tags:
```
Tool results and user messages may include <system-reminder> tags. Tags contain
information from the system. They bear no direct relation to the specific tool
results or user messages in which they appear.
```

**Why not just change the system prompt?** From Claude Code's prompt caching lessons: any change to the system prompt invalidates the cached prefix for the entire conversation. `<system-reminder>` tags in messages avoid this — the system prompt stays stable, and dynamic context arrives through messages that only affect subsequent API calls.

### 5 Injection Patterns

#### 1. User Context (session start)

CLAUDE.md content and current date are injected as a **fake first user message**, not in the system prompt:

```xml
<system-reminder>
As you answer the user's questions, you can use the following context:
# claudeMd
{CLAUDE.md contents}
# currentDate
Today's date is 2025-01-15.

      IMPORTANT: this context may or may not be relevant to your tasks.
      You should not respond to this context unless it is highly relevant.
</system-reminder>
```

This preserves the globally-cached system prompt prefix. Different users have different CLAUDE.md — putting it in messages avoids fragmenting the shared cache.

#### 2. Recurring Attachments (every N turns)

The harness periodically injects reminders and context updates:

| Content | Interval | Purpose |
|---------|----------|---------|
| Todo reminders | Every 10 turns (if todo tool exists, not used recently) | Nudge: "consider using TaskCreate to track progress" |
| Task reminders | Every 10 turns (if task tools exist) | Nudge: "consider cleaning up the task list" |
| Memory files | Up to 5 files per turn | Surface relevant memories with staleness warnings |
| Plan mode context | Every 5 turns | Reinforce plan mode constraints |
| Skill discovery | Per turn (when enabled) | Surface skills relevant to current task |

Memory files include **freshness annotations**: "This memory was created 3 days ago" — helping the model assess whether information is still current.

#### 3. Hook Output

When hooks execute (especially on errors), their output is wrapped in `<system-reminder>`:
```
Stop hook blocking error from command "lint": eslint found 3 errors
```

The model's system prompt tells it to treat hook feedback as coming from the user, so it adjusts behavior accordingly.

#### 4. Tool Result Warnings

Injected into tool results when special conditions are detected:

- **Empty file:** `Warning: the file exists but the contents are empty.`
- **Offset exceeds file length:** `Warning: the file exists but is shorter than the offset.`
- **Malware detection:** Wraps analysis directives about refusing to improve malicious code.

#### 5. Side Questions

When the user asks a quick question mid-task:
```xml
<system-reminder>
This is a side question from the user. You must answer this question
directly in a single response.
Simply answer the question with the information you have.
</system-reminder>
```

Forces the model to give an immediate focused answer instead of continuing the ongoing task.

### Implementation: `wrapInSystemReminder()`

A simple utility wraps any string in the tags:
```typescript
wrapInSystemReminder(content: string): string
// Returns: <system-reminder>\n{content}\n</system-reminder>
```

Batch variant wraps all user messages in an array. Stripping is done before UI display and transcript search.

---

## Error & Denial Message Engineering

When tools fail or permission is denied, the error text isn't just diagnostic — it's **steering**. Each error type nudges the model toward the right next action.

### Permission Denied

The model receives this in the `tool_result` (with `is_error: true`):

```
Permission to use {toolName} has been denied. IMPORTANT: You *may* attempt to
accomplish this action using other tools that might naturally be used to accomplish
this goal, e.g. using head instead of cat. But you *should not* attempt to work
around this denial in malicious ways, e.g. do not use your ability to run tests
to execute non-test actions. You should only try to work around this restriction
in reasonable ways that do not attempt to bypass the intent behind this denial.
If you believe this capability is essential to complete the user's request, STOP
and explain to the user what you were trying to do and why you need this permission.
Let the user decide how to proceed.
```

This message does three things:
1. Allows legitimate workarounds (flexibility)
2. Explicitly blocks malicious workarounds (safety)
3. Escalates to the user when the tool is truly needed (transparency)

**Don't-ask mode variant:** Adds "because Claude Code is running in don't ask mode" — the model knows there's no one to ask.

**Subagent variant:** "Try a different approach or report the limitation" — subagents can't ask the user, so they must self-resolve or report.

### Tool Use Rejected by User

```
The user doesn't want to proceed with this tool use. The tool use was rejected
(eg. if it was a file edit, the new_string was NOT written to the file). STOP what
you are doing and wait for the user to tell you how to proceed.
```

With user reason:
```
...the user said:
{userReason}
```

**Memory correction hint** (when auto-memory enabled, appended to cancellations):
```
Note: The user's next message may contain a correction or preference. Pay close
attention — if they explain what went wrong or how they'd prefer you to work,
consider saving that to memory for future sessions.
```

This turns rejections into learning opportunities — the model is primed to watch for preferences it should remember.

### Tool Use Cancelled

```
The user doesn't want to take this action right now. STOP what you are doing and
wait for the user to tell you how to proceed.
```

Softer than rejection — "right now" implies it might be OK later.

### Request Interrupted

```
[Request interrupted by user]
[Request interrupted by user for tool use]
```

Minimal — the user hit Ctrl+C, there's nothing to explain.

### Hook Blocked Execution

```
Execution stopped by PreToolUse hook: {stopReason}
```

### Validation Errors

**Zod schema failure:**
```
{toolName} failed due to the following issues:
The required parameter `file_path` is missing
The parameter `timeout` type is expected as `number` but provided as `string`
```

**Deferred tool schema hint** (when tool wasn't in the discovered-tool set):
```
This tool's schema was not sent to the API — it was not in the discovered-tool
set derived from message history. Without the schema in your prompt, typed
parameters (arrays, numbers, booleans) get emitted as strings and the client-side
parser rejects them. Load the tool first: call ToolSearch with query
"select:{tool_name}", then retry this call.
```

This is a recovery instruction — telling the model exactly how to fix the problem.

### Unknown Tool

```
Error: No such tool available: {toolName}
```

Wrapped in `<tool_use_error>` tags.

### File Size Limits

```
File is too large to edit ({size}). Maximum editable file size is 1 GiB.
```

```
File content ({tokenCount} tokens) exceeds maximum allowed tokens ({maxTokens}).
Use offset and limit parameters to read specific portions of the file, or search
for specific content instead of reading the whole file.
```

Both include the recovery action in the message itself.

### Large Tool Result Truncation

When results exceed ~50KB:
```xml
<persisted-output>
Output too large ({size}). Full output saved to: {filepath}

Preview (first 2KB):
{preview content}
... (hasMore)
</persisted-output>
```

---

## Design Lessons

1. **Use message injection instead of system prompt changes.** The `<system-reminder>` pattern preserves the prompt cache while delivering dynamic context. This is Claude Code's most important cost optimization.

2. **Error messages are steering, not just diagnostics.** Every error should guide the model's next action — whether that's trying a workaround, asking the user, or loading a tool schema.

3. **Distinguish error severity in the language.** "STOP and wait" (hard stop) vs "try a different approach" (soft redirect) vs "consider saving to memory" (learning opportunity). The model responds to these tonal cues.

4. **Include recovery instructions in the error itself.** "Load the tool first: call ToolSearch with query `select:{tool_name}`, then retry" — don't make the model figure out the fix.

5. **Permission denials should be nuanced.** Allow legitimate workarounds, block malicious ones, and escalate when the tool is truly needed. A blanket "denied, stop" is too rigid.

6. **Turn rejections into learning.** The memory correction hint after cancellations primes the model to watch for user preferences — making future sessions better.

7. **Use recurring injection at tuned intervals.** Too frequent = context rot. Too infrequent = model forgets. Claude Code uses 5-turn intervals for plan mode and 10-turn intervals for task reminders.

8. **Announce the injection mechanism in the system prompt.** The model knows `<system-reminder>` tags exist and that they're system-generated. Without this, the model might treat injected content as user messages.

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/utils/messages.ts` | `wrapInSystemReminder()`, denial/rejection/cancellation constants |
| `src/utils/api.ts` | `prependUserContext()` — fake first message injection |
| `src/utils/attachments.ts` | Recurring attachment scheduling (todo/task/memory intervals) |
| `src/utils/toolErrors.ts` | `formatZodValidationError()`, schema hint |
| `src/utils/toolResultStorage.ts` | `<persisted-output>` wrapper for large results |
| `src/services/tools/toolExecution.ts` | Error formatting in tool pipeline |
| `src/memdir/memoryAge.ts` | Memory freshness annotations |
| `src/utils/sideQuestion.ts` | Side question injection |
| `src/tools/FileReadTool/FileReadTool.ts` | Empty file and offset warnings |
| `src/tools/FileEditTool/FileEditTool.ts` | File size limit messages |
