# The "Think" Tool: Enabling Claude to Stop and Think

*By Anthropic — 2025-01-06*
*Source: https://www.anthropic.com/engineering/claude-think-tool*

---

**Update (Dec 2025)**: Extended thinking has improved enough that Anthropic recommends it over a dedicated think tool in most cases. The think tool remains valuable for specific scenarios below.

## Think tool vs extended thinking

| Feature | Extended thinking | Think tool |
|---------|------------------|------------|
| When | Before response generation | During response, between tool calls |
| Best for | Non-sequential tool calls, simple instruction following, coding/math | Long tool call chains, policy-heavy envs, sequential decisions |
| Nature | Deep pre-reasoning | Lightweight mid-chain scratchpad |

## Implementation

```json
{
  "name": "think",
  "description": "Use the tool to think about something. It will not obtain new information or change the database, but just append the thought to the log. Use it when complex reasoning or some cache memory is needed.",
  "input_schema": {
    "type": "object",
    "properties": {
      "thought": {
        "type": "string",
        "description": "A thought to think about."
      }
    },
    "required": ["thought"]
  }
}
```

## When to use

- **Tool output analysis**: Processing previous tool call results before acting; backtracking needed
- **Policy-heavy environments**: Following detailed guidelines, verifying compliance
- **Sequential decision making**: Each action builds on previous ones; mistakes are costly

## When NOT to use

- Non-sequential or parallel tool calls
- Simple instruction following where default behavior is good enough

## Key findings

- Think tool with **optimized prompting** >> think tool alone >> baseline (54% improvement on airline domain of tau-bench)
- Prompting matters enormously on hard domains. On easy domains, just having the tool available helps.
- Improvements maintained across pass^k up to k=5 (consistency, not just peak)

## Optimized prompt pattern

Include domain-specific examples in system prompt showing HOW to think:
```
Before taking any action after receiving tool results, use the think tool to:
- List specific rules that apply to current request
- Check if all required information is collected
- Verify planned action complies with all policies
- Iterate over tool results for correctness
```

Place complex think-tool guidance in the **system prompt**, not in the tool description itself.
