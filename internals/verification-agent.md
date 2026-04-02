# Verification Agent: Adversarial Quality Checking

*Source: Reverse-engineered from `@anthropic-ai/claude-code` v2.1.88 source map*

A case study in prompt engineering for adversarial evaluation. The verification agent independently checks implementation quality before the main agent reports completion. Feature-gated (`VERIFICATION_AGENT` + `tengu_hive_evidence` GrowthBook flag, defaults to `false` in external builds).

---

## How it works

### Core Design

The verification agent is a **read-only adversarial evaluator** that runs in the background. It cannot edit project files — only read, search, and execute commands. It produces a PASS/FAIL/PARTIAL verdict backed by command output evidence.

**Identity framing:**
> You are a verification specialist. Your job is not to confirm the implementation works — it's to try to break it.

### Two Named Failure Patterns

The prompt explicitly names the failure modes it's designed to prevent:

1. **Verification avoidance** — reading code, narrating what tests would do, then writing "PASS" without running anything. The prompt says: "Reading is not verification. Run it."

2. **Being seduced by the first 80%** — seeing polish and passing tests, then missing that half the features don't work. Surface quality is not a signal of complete correctness.

### Anti-Rationalization List

The prompt lists excuses the agent will feel urged to reach for, and preemptively blocks each one:

| Rationalization | Response |
|----------------|----------|
| "The code looks correct based on my reading" | Reading is not verification. Run it. |
| "The implementer's tests already pass" | The implementer is an LLM. Verify independently. |
| "This is probably fine" | Probably is not verified. Run it. |
| "Let me start the server and check the code" | No. Start the server and hit the endpoint. |
| "I don't have a browser" | Did you check for `mcp__claude-in-chrome__*` or `mcp__playwright__*`? |
| "This would take too long" | Not your call. |

### Type-Specific Verification Strategies

The prompt provides 11 different strategies based on what changed:

| Change type | Strategy |
|------------|----------|
| **Frontend** | Start dev server, use browser automation, screenshot, curl subresources, run frontend tests |
| **Backend/API** | Start server, curl endpoints, test response shapes, test error handling |
| **CLI/scripts** | Run with inputs, verify stdout/stderr/exit codes, test edge cases |
| **Infrastructure/config** | Validate syntax, dry-run, check env vars |
| **Library/package** | Build, full test suite, import and test public API |
| **Bug fixes** | Reproduce original bug, verify fix, run regression tests |
| **Mobile** | Clean build, install on simulator, dump accessibility tree, test persistence |
| **Data/ML pipeline** | Run with samples, verify output shape, test edge cases, check for silent data loss |
| **Database migrations** | Test up/down, reversibility, existing data |
| **Refactoring** | Existing tests must pass unchanged, check API surface, spot-check behavior |
| **Other** | Figure out how to exercise the change directly, check outputs, try to break it |

### Universal Required Steps

Regardless of change type, the verifier must always:
1. Read CLAUDE.md / README for build/test commands
2. Run the build (broken build = automatic FAIL)
3. Run the test suite (failing tests = automatic FAIL)
4. Run linters/type-checkers
5. Check for regressions in related code
6. Apply the type-specific strategy
7. Match rigor to stakes

### Adversarial Probes

Every PASS report must include at least ONE adversarial probe — concurrency, boundary values, idempotency, orphan operations, etc. — with actual results. Even if "handled correctly," the probe must be documented. This prevents perfunctory verification.

### Tool Restrictions

**Disallowed tools:** Agent, FileEdit, FileWrite, NotebookEdit, ExitPlanMode

**Key constraint:** Strictly prohibited from creating, modifying, or deleting files in the project directory. MAY write ephemeral test scripts to `/tmp` or `$TMPDIR` via Bash redirection — this allows writing temporary test harnesses without touching the project.

**Dynamic tool discovery:** The prompt instructs the verifier to check what tools are actually available (especially browser automation MCP tools) rather than assuming from the prompt text.

### Output Format

Every check must follow this structure:
```
### Check: [what you're verifying]
**Command run:**
  [exact command]
**Output observed:**
  [actual terminal output]
**Result: PASS** (or FAIL with Expected vs Actual)
```

Final line must be exactly one of (parsed by the harness):
```
VERDICT: PASS
VERDICT: FAIL
VERDICT: PARTIAL
```

### Critical System Reminder

Re-injected every turn via `criticalSystemReminder_EXPERIMENTAL`:
```
CRITICAL: This is a VERIFICATION-ONLY task. You CANNOT edit, write, or create files
IN THE PROJECT DIRECTORY (tmp is allowed for ephemeral test scripts). You MUST end
with VERDICT: PASS, VERDICT: FAIL, or VERDICT: PARTIAL.
```

### Trigger Mechanisms

**From the main agent's system prompt** (when feature enabled): The main agent is instructed that non-trivial implementation (3+ file edits, backend/API changes, infrastructure changes) requires independent verification before reporting completion.

**Auto-nudge from TaskUpdateTool:** When 3+ tasks are marked completed without a verification step, the harness injects: "Before writing your final summary, spawn the verification agent."

### The Main Agent's Contract

The main agent receives these instructions about working with the verifier:

- Pass the original user request, all files changed, the approach, and the plan file path
- Do NOT share test results or claim things work — only the verifier judges
- On FAIL: fix the issue, resume the verifier with findings + fix, repeat until PASS
- On PASS: spot-check — re-run 2-3 commands from the verifier's report, confirm every PASS has a Command run block with matching output
- On PARTIAL: report what passed and what could not be verified
- The main agent cannot self-assign PARTIAL — only the verifier can

### Agent Configuration

| Property | Value |
|----------|-------|
| Agent type | `verification` |
| Model | `inherit` (uses parent's model) |
| Color | `red` |
| Background | `true` (runs async) |
| Feature gate | `VERIFICATION_AGENT` compile-time + `tengu_hive_evidence` runtime |
| Default availability | `false` in external builds (ant-only A/B test) |

---

## Implementation reference

### Source files

| File | Role |
|------|------|
| `src/tools/AgentTool/built-in/verificationAgent.ts` | Agent definition, full system prompt (lines 10-152) |
| `src/tools/AgentTool/constants.ts` | `VERIFICATION_AGENT_TYPE = 'verification'` |
| `src/tools/AgentTool/builtInAgents.ts` | Feature-gated registration (lines 64-69) |
| `src/constants/prompts.ts` | Main agent's verification contract (lines 391-395) |
| `src/tools/TaskUpdateTool/TaskUpdateTool.ts` | Auto-nudge after 3+ tasks (line 397) |
| `src/tools/TodoWriteTool/TodoWriteTool.ts` | Auto-nudge after 3+ todos (line 107) |

### Design lessons for your own harness

1. **Name the failure modes.** Don't just say "verify carefully" — describe the specific ways verification goes wrong (avoidance, surface-level checking).
2. **List the rationalizations.** The model will reach for excuses. Pre-empt them explicitly.
3. **Separate generation from evaluation.** The verifier can't edit files — this removes the temptation to "just fix it" instead of reporting failures.
4. **Require evidence format.** Structured output (Command/Output/Result) ensures the verdict is backed by actual execution, not narration.
5. **Re-inject constraints every turn.** The `criticalSystemReminder_EXPERIMENTAL` prevents drift over long verification sessions.
6. **Type-specific strategies.** Generic "test everything" instructions produce shallow verification. Concrete strategies per change type produce thorough checks.
7. **Mandatory adversarial probes.** Requiring at least one probe prevents "everything looks fine" PASS reports.
