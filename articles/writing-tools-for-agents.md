# Writing Effective Tools for Agents — with Agents

*By Anthropic — 2025-06-26*
*Source: https://www.anthropic.com/engineering/writing-tools-for-agents*

---

## Development process

1. **Prototype**: Stand up quick tools, wrap in MCP server or DXT, test manually
2. **Evaluate**: Generate realistic tasks (not toy examples), run programmatic eval with agentic loops
3. **Collaborate with agents**: Feed eval transcripts to Claude Code → it analyzes failures and refactors tools

Use held-out test sets to avoid overfitting. Agent-optimized tools outperformed both manual expert and AI-generated tools.

## Principles for effective tools

### 1. Choose the right tools (and skip the wrong ones)

More tools ≠ better. Don't just wrap existing API endpoints. Design for agent affordances:

- Agents have limited context (expensive); computer memory is cheap
- `search_contacts` >> `list_contacts` (don't make agents brute-force search)
- Consolidate multi-step workflows: `schedule_event` (finds availability + reserves) >> separate `list_users` + `list_events` + `create_event`
- Each tool should have a clear, distinct purpose. Overlapping tools confuse agents.

### 2. Namespace your tools

Group related tools under common prefixes: `asana_projects_search`, `asana_users_search`. Prefix vs suffix namespacing has non-trivial effects — test with your own evals.

### 3. Return meaningful context

- Prioritize contextual relevance over flexibility
- Return high-signal information only
- Eschew low-level technical identifiers when human-readable alternatives exist
- Enrich responses with related metadata agents might need

### 4. Optimize for token efficiency

Tool responses consume context budget. Every token returned must earn its place.

### 5. Prompt-engineer tool descriptions

- Descriptions are as important as implementation
- Include example usage, edge cases, input format requirements
- When Claude was appending `2025` to web search queries (biasing results), fixing the tool description solved it

## Evaluation design

**Strong tasks** require multiple tool calls, are grounded in realistic data, and mirror real-world complexity:
- "Customer ID 9182 was charged three times. Find log entries and determine if others were affected."

**Weak tasks** are overly specific or pre-decomposed:
- "Search the payment logs for `purchase_complete` and `customer_id=9182`."

Pair each task with a verifiable outcome. Avoid overly strict verifiers that reject correct answers for formatting differences.

## Metrics to track

- Top-level accuracy
- Runtime per tool call and per task
- Total tool calls (redundant calls → needs pagination/limit tuning)
- Token consumption
- Tool errors (invalid params → needs clearer descriptions)
