# Turn Loop & Compaction

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code's core conversation loop works, how it streams from the API, recovers from errors, and manages the context window. See [index.md](index.md) for architecture overview.

---

## How it works

### QueryEngine

One `QueryEngine` instance per conversation. Its `submitMessage(prompt)` is an async generator that yields SDK messages. It fetches system prompt parts, processes user input (slash commands, paste expansion), records the transcript, then delegates to the core `query()` loop. Accumulates usage, permission denials, and discovered skill names across turns.

### The Core Loop

`query()` is an **infinite `while(true)` loop** with immutable state. Each `continue` creates a fresh State object — no mutation between iterations. Each iteration runs 6 phases:

```
1. COMPACTION PHASE
   +-- Apply snip (fast granular pruning)
   +-- Apply microcompact (cache-aware content clearing)
   +-- Apply context collapse (staged collapses)
   +-- Apply autocompact if context > threshold

2. API CALL PHASE
   +-- Stream from Claude API
   +-- Buffer assistant messages + tool_use blocks
   +-- Stream-time tool execution (if enabled)
   +-- Withhold recoverable errors (PTL, max_output_tokens)

3. RECOVERY PHASE
   +-- Context collapse drain (if PTL withheld)
   +-- Reactive compact (if still PTL after drain)
   +-- Max output tokens escalation (8k -> 64k)
   +-- Max output tokens retry (add nudge message)

4. STOP HOOKS PHASE
   +-- Execute user-defined stop hooks
   +-- Handle blocking errors
   +-- Check token budget

5. TOOL EXECUTION PHASE
   +-- Execute tool calls (concurrent + serial batching)
   +-- Generate tool summaries (async Haiku call)
   +-- Drain queued commands, consume memory prefetch
   +-- Inject skill discovery attachments

6. DECISION PHASE
   +-- No tool follow-up needed -> return 'completed'
   +-- Max turns reached -> return 'max_turns'
   +-- Otherwise -> continue with new State
```

### Stop Conditions

The loop ends when:
- `completed` -- no tool calls, no follow-up needed
- `max_turns` -- turn limit exceeded
- `blocking_limit` -- context window limit hit
- `prompt_too_long` -- reactive compact failed
- `image_error` -- media recovery failed
- `stop_hook_prevented` -- a hook blocked continuation
- `aborted_tools` -- user aborted mid-execution

### Continue Conditions

The loop continues when:
- `next_turn` -- tool results received, normal continuation
- `collapse_drain_retry` -- drained staged collapses, retry API call
- `reactive_compact_retry` -- compact succeeded, retry
- `max_output_tokens_escalate` -- 8k->64k, same request
- `max_output_tokens_recovery` -- inject nudge message, retry
- `stop_hook_blocking` -- inject error, retry
- `token_budget_continuation` -- inject nudge, retry

Each site stores a `transition` type for debugging — you can trace exactly why each iteration happened.

### API Streaming

The model is called via streaming. Key mechanics:
- MCP tools use `defer_loading` — sent as name-only stubs, full schemas fetched on demand via ToolSearch (preserves cache stability)
- System prompt sent as blocks with `cache_control` markers
- User context prepended as first message
- Tool inputs backfilled via `tool.backfillObservableInput()` during streaming (hooks/UI see derived fields before tool completes)
- Orphaned tool_use blocks (from interrupted streams) get synthetic tool_result

**Error recovery during streaming:**
- Fallback model triggering on specific errors
- Prompt too long (413): withheld, recovered via collapse drain or reactive compact
- Max output tokens: withheld, escalated (8k->64k) or retried with nudge
- Image size errors: withheld, recovered via compact

### Multi-Layer Compaction

5 layers applied in order of increasing aggressiveness:

**A. Snip** -- fast, granular pruning by message depth. Marks messages for removal before other compactions.

**B. Microcompact** -- clears old tool_result content while keeping the tool_use/tool_result structure intact (needed for API message pairing). Replaced with `[Old tool result content cleared]`. Only for safe tools: Bash, Grep, Glob, Read, WebSearch, Write, Edit. Cache edits tracked and re-pinned across requests.

**C. Context Collapse** -- staged collapses applied per-iteration. Replayed on session resume via a collapse commit log. Drains overflow queue on 413 errors.

**D. Autocompact** -- triggers proactively when token usage exceeds threshold. Calls Claude via forked subagent (using same system prompt for cache reuse) to produce a summary. Resets turn tracking and clears system prompt section cache. Circuit breaker tracks consecutive failures.

**E. Reactive Compact** -- emergency recovery after 413 PTL that wasn't fixed by collapse drain. Attempts full compact on the failed request. Single shot per turn (guard prevents infinite retries).

### Progressive Error Recovery

5 recovery paths, tried in order of cost:
1. Context collapse drain (cheapest -- just release staged collapses)
2. Reactive compact (full summary)
3. Max output token escalation (8k->64k, no message change)
4. Max output token retry with nudge message
5. Surface error to user (last resort)

Each path has a guard to prevent loops.

### Token Budget Carry-Over

When autocompact fires mid-task with a token budget active, the pre-compact context window cost is subtracted from the remaining budget. The server sees only the summary post-compact, so spend must be pre-computed to maintain accurate tracking.

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/QueryEngine.ts` | `QueryEngine` class, `submitMessage()` (lines 209-1156) |
| `src/query.ts` | `query()` loop (lines 241-1729) |
| `src/services/api/claude.ts` | `queryModelWithStreaming()`, streaming loop |
| `src/query/stopHooks.ts` | `handleStopHooks()`, stop hook execution |
| `src/query/tokenBudget.ts` | Token budget tracking and continuation |
| `src/query/config.ts` | Immutable per-query config snapshot |
| `src/query/deps.ts` | Testable I/O abstraction (callModel, compaction) |
| `src/services/compact/compact.ts` | Main compaction logic |
| `src/services/compact/microCompact.ts` | Microcompact, cache editing |
| `src/services/compact/autoCompact.ts` | Auto-trigger compaction |

### Loop State type

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking?: AutoCompactTrackingState
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride?: number
  pendingToolUseSummary?: Promise<ToolUseSummaryMessage>
  stopHookActive?: boolean
  turnCount: number
  transition?: Continue  // Why the loop continued
}
```

### Streaming events

Standard Anthropic streaming sequence: `message_start` -> `content_block_start` -> `content_block_delta` (repeated) -> `content_block_stop` -> `message_delta` -> `message_stop`. Usage accumulated on message_start/message_delta/message_stop events.
