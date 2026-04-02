# Run Claude Code programmatically

> Use the Agent SDK to run Claude Code programmatically from the CLI, Python, or TypeScript.

The [Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) gives you the same tools, agent loop, and context management that power Claude Code. It's available as a CLI for scripts and CI/CD, or as Python and TypeScript packages for full programmatic control.

> The CLI was previously called "headless mode." The `-p` flag and all CLI options work the same way.

To run Claude Code programmatically from the CLI, pass `-p` with your prompt and any CLI options:

```bash
claude -p "Find and fix the bug in auth.py" --allowedTools "Read,Edit,Bash"
```

## Basic usage

Add the `-p` (or `--print`) flag to any `claude` command to run it non-interactively. All CLI options work with `-p`, including:

* `--continue` for continuing conversations
* `--allowedTools` for auto-approving tools
* `--output-format` for structured output

```bash
claude -p "What does the auth module do?"
```

## Examples

### Get structured output

Use `--output-format` to control how responses are returned:

* `text` (default): plain text output
* `json`: structured JSON with result, session ID, and metadata
* `stream-json`: newline-delimited JSON for real-time streaming

```bash
claude -p "Summarize this project" --output-format json
```

With JSON schema:

```bash
claude -p "Extract the main function names from auth.py" \
  --output-format json \
  --json-schema '{"type":"object","properties":{"functions":{"type":"array","items":{"type":"string"}}},"required":["functions"]}'
```

### Stream responses

```bash
claude -p "Explain recursion" --output-format stream-json --verbose --include-partial-messages
```

Filter for text deltas:

```bash
claude -p "Write a poem" --output-format stream-json --verbose --include-partial-messages | \
  jq -rj 'select(.type == "stream_event" and .event.delta.type? == "text_delta") | .event.delta.text'
```

### Auto-approve tools

```bash
claude -p "Run the test suite and fix any failures" \
  --allowedTools "Bash,Read,Edit"
```

### Create a commit

```bash
claude -p "Look at my staged changes and create an appropriate commit" \
  --allowedTools "Bash(git diff *),Bash(git log *),Bash(git status *),Bash(git commit *)"
```

The trailing ` *` enables prefix matching. The space before `*` matters: `Bash(git diff *)` allows commands starting with `git diff `.

### Customize the system prompt

```bash
gh pr diff "$1" | claude -p \
  --append-system-prompt "You are a security engineer. Review for vulnerabilities." \
  --output-format json
```

### Continue conversations

```bash
# First request
claude -p "Review this codebase for performance issues"

# Continue the most recent conversation
claude -p "Now focus on the database queries" --continue
claude -p "Generate a summary of all issues found" --continue
```

Resume a specific session:

```bash
session_id=$(claude -p "Start a review" --output-format json | jq -r '.session_id')
claude -p "Continue that review" --resume "$session_id"
```

## Next steps

* [Agent SDK quickstart](https://platform.claude.com/docs/en/agent-sdk/quickstart): build your first agent with Python or TypeScript
* [CLI reference](/en/cli-reference): all CLI flags and options
* [GitHub Actions](/en/github-actions): use the Agent SDK in GitHub workflows
