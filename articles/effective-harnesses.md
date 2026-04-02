# Effective Harnesses for Long-Running Agents

*By Justin Young, Anthropic — 2025-11-24*
*Source: https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents*

---

## Problem

Agents working across multiple context windows lose memory between sessions. Like engineers on shifts with no handoff notes — each new session starts blank.

## Two-part solution

### 1. Initializer agent (first session only)
Sets up environment with everything future sessions need:
- `init.sh` script to run dev server
- `claude-progress.txt` — log of what agents have done
- `feature_list.json` — comprehensive feature requirements (all initially `"passes": false`)
- Initial git commit

### 2. Coding agent (every subsequent session)
Makes incremental progress, leaves structured artifacts:
- Works on **one feature at a time** (critical — prevents one-shotting)
- Commits to git with descriptive messages
- Updates progress file
- Only marks features "passing" after careful testing

## Session startup sequence

Every coding agent session:
1. `pwd` — see working directory
2. Read git logs + progress files — understand recent work
3. Read feature list — choose highest-priority incomplete feature
4. Run `init.sh` — start dev server
5. Run basic end-to-end test — catch broken state before adding new features
6. Start work on chosen feature

## Failure modes and solutions

| Problem | Initializer solution | Coding agent solution |
|---------|---------------------|----------------------|
| Declares victory too early | Feature list file (structured JSON) | Read feature list, choose single feature |
| Leaves buggy/undocumented state | Initial git repo + progress file | Read progress + git log, run basic test, commit + update at end |
| Marks features done prematurely | Feature list with pass/fail tracking | Self-verify all features after implementation |
| Wastes time figuring out how to run app | Write `init.sh` script | Read `init.sh` at session start |

## Key design decisions

- **JSON over Markdown** for feature list: model less likely to inappropriately edit/overwrite JSON
- **Strong-worded instructions**: "It is unacceptable to remove or edit tests" — prevents model from cheating by removing failing tests
- **Git for recovery**: model can revert bad changes and recover working states
- **End-to-end testing tools** (Puppeteer MCP): dramatically improved bug detection vs unit tests alone. Vision limitations remain (can't see browser-native alert modals).

## Open questions

- Single general-purpose agent vs specialized multi-agent (testing agent, QA agent, cleanup agent)?
- Generalizing beyond web apps to scientific research, financial modeling, etc.
