# Harness Design for Long-Running Application Development

*By Prithvi Rajasekaran, Anthropic Labs — 2026-03-06*
*Source: https://www.anthropic.com/engineering/harness-design-long-running-apps*

---

## Core insight: GAN-inspired multi-agent architecture

Separate **generation** from **evaluation**. Tuning a standalone evaluator to be skeptical is far more tractable than making a generator critical of its own work.

## Two failure modes of naive implementations

1. **Context coherence loss**: As context fills, model loses focus. Some models exhibit "context anxiety" — wrapping up prematurely near perceived context limit.
   - **Context resets** (clearing window entirely + structured handoff) >> **compaction** (summarize in place). Compaction doesn't give a clean slate, so context anxiety persists.
   - Opus 4.5 largely eliminated context anxiety, allowing continuous sessions with automatic compaction instead of resets.

2. **Self-evaluation bias**: Agents confidently praise their own mediocre work. Especially bad for subjective tasks (design), but even for verifiable tasks.
   - Separate evaluator that navigates/interacts with the output before scoring.

## Frontend design experiment

Four grading criteria given to both generator and evaluator:
1. **Design quality** — coherent whole vs collection of parts (heavily weighted)
2. **Originality** — custom decisions vs template defaults/AI slop (heavily weighted)
3. **Craft** — typography, spacing, color harmony (model already good at this)
4. **Functionality** — usability independent of aesthetics (model already good at this)

Key findings:
- Evaluator used **Playwright MCP** to navigate live page, screenshot, interact before scoring
- 5-15 iterations per generation, runs up to 4 hours
- Generator instructed to make strategic decision: refine current direction or pivot entirely
- Criteria wording directly shaped output character ("museum quality" pushed toward specific convergence)
- Even first iteration better than no-criteria baseline — criteria steer before any feedback

## Full-stack architecture (3 agents)

### Planner
- Takes simple 1-4 sentence prompt → expands to full product spec
- Focuses on product context + high-level technical design (NOT granular implementation details — errors cascade)
- Constrained on deliverables, agents figure out implementation path

### Generator
- Works in **sprints**, one feature at a time
- Self-evaluates at sprint end before handing off to QA
- Has git for version control
- Stack: React, Vite, FastAPI, SQLite/PostgreSQL

### Evaluator
- Uses **Playwright MCP** to click through running application like a user
- Tests UI features, API endpoints, database states
- Grades each sprint against bugs found + criteria (product depth, functionality, visual design, code quality)
- Hard threshold per criterion — below any threshold = sprint fails with detailed feedback

### Sprint contract
Before each sprint, generator and evaluator negotiate: agree on what "done" looks like before any code is written. Bridges gap between high-level spec and testable implementation.

### Communication
Via files: one agent writes, another reads and responds. Keeps work faithful to spec without over-specifying implementation.

## Results

| Harness | Duration | Cost |
|---------|----------|------|
| Solo agent | 20 min | $9 |
| Full 3-agent harness | 6 hr | $200 |

The 3-agent harness produced significantly richer, more polished applications with real functionality (vs solo agent's quick but shallow output).
