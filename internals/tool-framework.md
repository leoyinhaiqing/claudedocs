# Tool Framework & Permission System

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

How Claude Code defines, registers, executes, and permission-gates tools. See [index.md](index.md) for architecture overview.

---

## How it works

### Tool Definition

Each tool declares: an input schema (Zod), a prompt function (generates the tool description sent to the API), permission checks, execution logic, and metadata flags (concurrency-safe? read-only? destructive?).

A `buildTool()` factory provides defaults — most tools only override what they need. Defaults: enabled=true, concurrencySafe=false, readOnly=false, permissions=allow.

Tool prompts are **context-aware**: the prompt function receives the set of available tools, agents, and permission context, so descriptions adapt. Example: the Bash tool's description dynamically includes guidance about preferring dedicated tools and sandbox behavior.

### Tool Registration

Three functions manage the tool pool:
- `getAllBaseTools()` — every available built-in tool
- `getTools(permCtx)` — filtered by deny rules and enabled state
- `assembleToolPool(permCtx, mcpTools)` — combines built-in + MCP tools, deduplicated (built-ins win), sorted for cache stability (built-ins first, MCP alphabetically after)

Tools are conditionally loaded via three patterns: environment checks (`process.env.USER_TYPE === 'ant'` for internal-only), feature gates (`feature('PROACTIVE')` for compile-time DCE), and lazy `require()` for circular dependency breaks.

### Execution Pipeline (8 phases)

```
runToolUse(toolUse, assistantMsg, canUseTool, context)
  |
  1. TOOL LOOKUP
  |   Find by name or alias; fall back to getAllBaseTools() for deprecated names
  |
  2. INPUT VALIDATION
  |   Zod schema parse + tool-specific validateInput()
  |
  3. PRE-TOOL-USE HOOKS
  |   Run hooks; can emit messages, modify input, or block execution
  |
  4. PERMISSION CHECK
  |   canUseTool() or hook-provided decision
  |   deny -> rejection message | ask -> prompt user | allow -> continue
  |
  5. TOOL EXECUTION
  |   tool.call() with progress callbacks streamed in real-time
  |
  6. RESULT PROCESSING
  |   Map to API format; persist large results to disk with preview
  |
  7. POST-TOOL-USE HOOKS
  |   Run hooks; can update output, add attachments
  |
  8. RESULT ASSEMBLY
      Build: tool_result + accept feedback + images -> user message
```

### Result Handling

Results exceeding 50KB characters are **persisted to disk** and replaced with a preview wrapper:
```xml
<persisted-output file="/path/to/result.json">
  {first 2000 bytes as preview}
  ... (hasMore)
</persisted-output>
```
This keeps the context window manageable while giving the model enough to decide whether to read the full result. Empty results get a placeholder to prevent model stop-sequence bugs. Per-message character caps prevent a single turn from overwhelming context.

### Tool Concurrency

**Batch mode** (default): tool calls are partitioned — read-only tools run in parallel (up to 10), write tools run alone. Context modifiers queued during a batch are applied after it completes.

**Streaming mode** (feature-gated): tools execute as they stream in from the model. Concurrency-safe tools run in parallel with each other; non-concurrent tools get an exclusive lock. A child abort controller kills sibling processes on error.

### Permission Modes

| Mode | Behavior |
|------|----------|
| `default` | Ask user for each tool use |
| `acceptEdits` | Auto-allow file edits, ask for others |
| `bypassPermissions` | Allow everything (YOLO mode) |
| `plan` | Read-only tools only |
| `dontAsk` | Deny anything that would normally ask |
| `auto` | Classifier-based auto-approval (experimental) |

### Permission Rules

Granular rules allow/deny/ask per tool or pattern. Example: `Bash(git *)` allows all git commands in Bash. Rules come from 8 sources with strict priority:

```
flagSettings > policySettings > userSettings > projectSettings > localSettings > cliArg > command > session
```

### Permission Decision Flow

```
1. Create permission context
2. If forceDecision provided, use it
3. Otherwise call hasPermissionsToUseTool()
4. allow -> log, execute
5. deny -> log, reject with reason
6. ask ->
   a. Check coordinator/swarm handlers
   b. Show interactive permission UI
   c. Wait for user response
   d. Classifier checks run async for Bash
```

Three decision types: **allow** (optional input modification), **deny** (reason required), **ask** (pending classifier check possible). Decision reasons include: rule match, mode requirement, hook block, classifier decision, safety check, working directory restriction.

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/Tool.ts` | Tool type definition (lines 362-695), `buildTool()` (line 783) |
| `src/tools.ts` | `getAllBaseTools()`, `getTools()`, `assembleToolPool()` |
| `src/services/tools/toolExecution.ts` | `runToolUse()` — full pipeline (1745 lines) |
| `src/services/tools/toolOrchestration.ts` | `runTools()` — batch partitioning, concurrency |
| `src/services/tools/StreamingToolExecutor.ts` | Stream-time tool execution |
| `src/utils/toolResultStorage.ts` | `persistToolResult()`, threshold logic |
| `src/types/permissions.ts` | PermissionMode, PermissionRule, PermissionResult types |
| `src/utils/permissions/permissions.ts` | `hasPermissionsToUseTool()` |
| `src/hooks/useCanUseTool.tsx` | `CanUseToolFn` — permission gating hook (lines 32-150) |

### Tool interface

```typescript
interface Tool {
  name: string
  aliases?: string[]
  inputSchema: ZodSchema
  inputJSONSchema?: ToolInputJSONSchema   // MCP-format schema
  outputSchema?: ZodSchema
  maxResultSizeChars: number              // Persistence threshold

  prompt(options): string
  call(args, context, canUseTool, parentMsg, onProgress): Promise<ToolResult>
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput(input, context): Promise<ValidationResult>

  renderToolUseMessage(input, options): ReactNode
  renderToolResultMessage(content, progress, options): ReactNode

  isEnabled(): boolean
  isConcurrencySafe(input): boolean
  isReadOnly(input): boolean
  isDestructive(input): boolean
  isSearchOrReadCommand(input): boolean
  toAutoClassifierInput(input): string
  preparePermissionMatcher(input): (pattern) => boolean
}
```

### Permission types

```typescript
type PermissionRule = {
  source: 'userSettings' | 'projectSettings' | 'localSettings' |
          'flagSettings' | 'policySettings' | 'cliArg' | 'command' | 'session'
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: { toolName: string; ruleContent?: string }
}

// Decision types
{ behavior: 'allow', updatedInput?, decisionReason? }
{ behavior: 'deny', message: string, decisionReason: required }
{ behavior: 'ask', message: string, pendingClassifierCheck? }
```

### Conditional loading patterns

```typescript
// Internal-only (DCE in external builds)
const REPLTool = process.env.USER_TYPE === 'ant'
  ? require('./tools/REPLTool/REPLTool.js').REPLTool : null

// Feature-gated
const SleepTool = feature('PROACTIVE') || feature('KAIROS')
  ? require('./tools/SleepTool/SleepTool.js').SleepTool : null

// Lazy require (circular dependency break)
const getTeamCreateTool = () =>
  require('./tools/TeamCreateTool/TeamCreateTool.js').TeamCreateTool
```
