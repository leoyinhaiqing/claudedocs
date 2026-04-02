# Best Practices for Claude Code

> Tips and patterns for getting the most out of Claude Code, from configuring your environment to scaling across parallel sessions.

Claude Code is an agentic coding environment. Unlike a chatbot that answers questions and waits, Claude Code can read your files, run commands, make changes, and autonomously work through problems while you watch, redirect, or step away entirely.

This changes how you work. Instead of writing code yourself and asking Claude to review it, you describe what you want and Claude figures out how to build it. Claude explores, plans, and implements.

But this autonomy still comes with a learning curve. Claude works within certain constraints you need to understand.

This guide covers patterns that have proven effective across Anthropic's internal teams and for engineers using Claude Code across various codebases, languages, and environments. For how the agentic loop works under the hood, see How Claude Code works.

***

Most best practices are based on one constraint: Claude's context window fills up fast, and performance degrades as it fills.

Claude's context window holds your entire conversation, including every message, every file Claude reads, and every command output. However, this can fill up fast. A single debugging session or codebase exploration might generate and consume tens of thousands of tokens.

This matters since LLM performance degrades as context fills. When the context window is getting full, Claude may start "forgetting" earlier instructions or making more mistakes. The context window is the most important resource to manage.

***

## Give Claude a way to verify its work

Include tests, screenshots, or expected outputs so Claude can check itself. This is the single highest-leverage thing you can do.

Claude performs dramatically better when it can verify its own work, like run tests, compare screenshots, and validate outputs.

| Strategy | Before | After |
|---|---|---|
| **Provide verification criteria** | "implement a function that validates email addresses" | "write a validateEmail function. example test cases: user@example.com is true, invalid is false. run the tests after implementing" |
| **Verify UI changes visually** | "make the dashboard look better" | "[paste screenshot] implement this design. take a screenshot and compare it to the original. list differences and fix them" |
| **Address root causes** | "the build is failing" | "the build fails with this error: [paste error]. fix it and verify the build succeeds. address the root cause, don't suppress the error" |

***

## Explore first, then plan, then code

Separate research and planning from implementation to avoid solving the wrong problem. Use Plan Mode to separate exploration from execution.

**Four phases:**

1. **Explore** — Enter Plan Mode. Claude reads files without making changes.
2. **Plan** — Ask Claude to create a detailed implementation plan. Press `Ctrl+G` to open the plan in your text editor before Claude proceeds.
3. **Implement** — Switch back to Normal Mode and let Claude code, verifying against its plan.
4. **Commit** — Ask Claude to commit with a descriptive message and create a PR.

> Planning is most useful when you're uncertain about the approach, when the change modifies multiple files, or when you're unfamiliar with the code being modified. If you could describe the diff in one sentence, skip the plan.

***

## Provide specific context in your prompts

The more precise your instructions, the fewer corrections you'll need.

| Strategy | Before | After |
|---|---|---|
| **Scope the task** | "add tests for foo.py" | "write a test for foo.py covering the edge case where the user is logged out. avoid mocks." |
| **Point to sources** | "why does ExecutionFactory have such a weird api?" | "look through ExecutionFactory's git history and summarize how its api came to be" |
| **Reference existing patterns** | "add a calendar widget" | "look at how HotDogWidget.php is implemented, follow the pattern to implement a new calendar widget" |
| **Describe the symptom** | "fix the login bug" | "users report login fails after session timeout. check src/auth/, especially token refresh. write a failing test, then fix it" |

### Provide rich content

* **Reference files with `@`** instead of describing where code lives
* **Paste images directly** — copy/paste or drag and drop
* **Give URLs** for documentation and API references
* **Pipe in data**: `cat error.log | claude`
* **Let Claude fetch** context itself using Bash commands or MCP tools

***

## Configure your environment

### Write an effective CLAUDE.md

Run `/init` to generate a starter CLAUDE.md, then refine over time. Include Bash commands, code style, and workflow rules.

**Include:**
- Bash commands Claude can't guess
- Code style rules that differ from defaults
- Testing instructions and preferred test runners
- Repository etiquette (branch naming, PR conventions)
- Architectural decisions specific to your project
- Developer environment quirks (required env vars)

**Exclude:**
- Anything Claude can figure out by reading code
- Standard language conventions Claude already knows
- Long explanations or tutorials
- File-by-file descriptions of the codebase

Keep it concise. For each line ask: "Would removing this cause Claude to make mistakes?" If not, cut it. Bloated CLAUDE.md files cause Claude to ignore your actual instructions.

CLAUDE.md supports `@path/to/import` syntax for importing additional files.

### Configure permissions

Use `/permissions` to allowlist safe commands or `/sandbox` for OS-level isolation. Two approaches:
- **Permission allowlists**: permit specific tools you know are safe
- **Sandboxing**: OS-level isolation that restricts filesystem and network access

### Use CLI tools

Tell Claude Code to use CLI tools like `gh`, `aws`, `gcloud`, and `sentry-cli` when interacting with external services. Claude is effective at learning CLI tools it doesn't already know: "Use 'foo-cli-tool --help' to learn about foo tool, then use it to solve A, B, C."

### Connect MCP servers

Run `claude mcp add` to connect external tools like Notion, Figma, or your database.

### Set up hooks

Use hooks for actions that must happen every time with zero exceptions. Unlike CLAUDE.md instructions which are advisory, hooks are deterministic. Claude can write hooks for you: "Write a hook that runs eslint after every file edit."

### Create skills

Create `SKILL.md` files in `.claude/skills/` to give Claude domain knowledge and reusable workflows. Claude applies them automatically when relevant, or you invoke with `/skill-name`.

### Create custom subagents

Define specialized assistants in `.claude/agents/` that Claude can delegate to for isolated tasks. Subagents run in their own context with their own allowed tools.

### Install plugins

Run `/plugin` to browse the marketplace. Plugins bundle skills, hooks, subagents, and MCP servers into a single installable unit.

***

## Communicate effectively

### Ask codebase questions

Use Claude Code for learning and exploration on new codebases:
- How does logging work?
- How do I make a new API endpoint?
- Why does this code call `foo()` instead of `bar()` on line 333?

### Let Claude interview you

For larger features, have Claude interview you first:

```
I want to build [brief description]. Interview me in detail using the AskUserQuestion tool.

Ask about technical implementation, UI/UX, edge cases, concerns, and tradeoffs. Don't ask obvious questions, dig into the hard parts I might not have considered.

Keep interviewing until we've covered everything, then write a complete spec to SPEC.md.
```

Once the spec is complete, start a fresh session to execute it.

***

## Manage your session

### Course-correct early and often

- **`Esc`**: stop Claude mid-action, context preserved
- **`Esc + Esc` or `/rewind`**: restore previous conversation and code state
- **`"Undo that"`**: have Claude revert its changes
- **`/clear`**: reset context between unrelated tasks

If you've corrected Claude more than twice on the same issue, run `/clear` and start fresh with a more specific prompt.

### Manage context aggressively

- Use `/clear` frequently between tasks
- Run `/compact <instructions>` for targeted compaction: `/compact Focus on the API changes`
- Use `Esc + Esc` or `/rewind` to summarize from a checkpoint
- Add compaction instructions to CLAUDE.md: "When compacting, always preserve the full list of modified files"
- Use `/btw` for quick questions that don't need to stay in context

### Use subagents for investigation

Delegate research with "use subagents to investigate X." They explore in a separate context, keeping your main conversation clean:

```
Use subagents to investigate how our authentication system handles token
refresh, and whether we have any existing OAuth utilities I should reuse.
```

### Rewind with checkpoints

Every action Claude makes creates a checkpoint. Double-tap `Escape` or run `/rewind` to restore conversation, code, or both to any previous checkpoint. Checkpoints persist across sessions.

### Resume conversations

```bash
claude --continue    # Resume the most recent conversation
claude --resume      # Select from recent conversations
```

Use `/rename` to give sessions descriptive names like "oauth-migration" or "debugging-memory-leak".

***

## Automate and scale

### Run non-interactive mode

```bash
claude -p "your prompt"                                    # one-off
claude -p "List all API endpoints" --output-format json    # structured output
claude -p "Analyze this log file" --output-format stream-json  # streaming
```

### Run multiple Claude sessions

Three main ways:
- **Claude Code desktop app**: Manage multiple local sessions visually with isolated worktrees
- **Claude Code on the web**: Run on Anthropic's secure cloud infrastructure
- **Agent teams**: Automated coordination with shared tasks and messaging

**Writer/Reviewer pattern:**
- Session A writes the implementation
- Session B reviews for edge cases and consistency
- Session A addresses review feedback

### Fan out across files

```bash
for file in $(cat files.txt); do
  claude -p "Migrate $file from React to Vue. Return OK or FAIL." \
    --allowedTools "Edit,Bash(git commit *)"
done
```

***

## Avoid common failure patterns

- **The kitchen sink session**: Mixing unrelated tasks → `/clear` between tasks
- **Correcting over and over**: After two failed corrections, `/clear` and write a better prompt
- **Over-specified CLAUDE.md**: Too long → Claude ignores it. Ruthlessly prune.
- **The trust-then-verify gap**: Always provide verification (tests, scripts, screenshots)
- **The infinite exploration**: Scope investigations narrowly or use subagents

***

## Related resources

* How Claude Code works: the agentic loop, tools, and context management
* Extend Claude Code: skills, hooks, MCP, subagents, and plugins
* Common workflows: step-by-step recipes for debugging, testing, PRs, and more
* CLAUDE.md: store project conventions and persistent context
