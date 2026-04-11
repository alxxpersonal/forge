---
name: model-forge
description: Pick the right model for a task and the right git strategy for the work. Use when about to dispatch a subagent, start a new feature, or unsure whether to use opus/sonnet/haiku/codex, or whether to branch/worktree/work direct. NOT for actually running the work - this only routes.
disable-model-invocation: false
---

# Model Forge

Routing reference for picking the right model and the right git strategy. Consult before any subagent dispatch or non-trivial change.

## Critical Rule

**ALWAYS run subagents in background (`run_in_background: true`) UNLESS the user explicitly asks for foreground.** The main thread stays unblocked, the user can keep working, and long-running dispatches don't freeze the conversation. Foreground subagents are the exception, not the default.

## Model Capability Matrix

### claude-opus-4-6
Heaviest reasoning, best at multi-file architecture and critical correctness. Slowest and most expensive claude.
**Speed/cost:** ~3-5x slower than sonnet, ~5x cost premium.
**Use for:**
- Deep architecture design and hard refactors
- Security audits requiring full mental model
- Complex math, proofs, algorithmic correctness
- Novel problems with no clear path
- Multi-system integration design
- Critical decisions where being wrong is expensive

### claude-sonnet-4-6
Balanced workhorse. Default for ~90% of work and most subagent dispatches.
**Speed/cost:** solid throughput, mid-tier cost.
**Use for:**
- Standard feature coding and debugging
- Research synthesis and writing
- Code review for logic and readability
- Scaffolding new modules from spec
- Most multi-step subagent tasks
- Anything you'd default to without thinking

### claude-haiku-4-5
Fastest and cheapest. Capable enough for mechanical work.
**Speed/cost:** near-instant, fraction of sonnet cost.
**Use for:**
- File reads, scans, classification
- Transcription (youtube, audio, OCR)
- Single tool calls with no reasoning
- Grep-style searches across large trees
- Generating structured metadata from raw input
- Bulk context gathering across many files

### gpt-5.4-codex (via codex plugin)
Adversarial engineer with different priors than claude. Mechanically excellent on focused tasks, brittle on vague ones. Extremely slow.
**Speed/cost:** ~30x slower than sonnet (10 min vs 18s in real tests). 3-4x fewer tokens than claude on comparable work. ChatGPT Plus = 30-150 msgs/5hr window.
**Benchmarks:** terminal-bench 77.3% vs claude 65.4%. Debate mode (claude + codex) bug detection jumps 53% → 80%.
**IMPORTANT:** Always pass `--model gpt-5.4-codex` (or `model: "gpt-5.4-codex"` in subagent config). Default model selection is unreliable - lock it explicitly every time.
**Use for:**
- Adversarial code review (highest-ROI use case - independent priors catch what claude misses)
- Focused bug fixes with a clear spec and acceptance criteria
- Mechanical refactors to a defined pattern (85-90% success on bounded scope)
- Test coverage for existing logic (stays in lane, no scope creep)
- Maintenance batch work (deps, doc fixes, webhook changes - queue 3-5 at session start)
- Terminal/CLI workflows
- Second-opinion on completed code

## Routing Decision Tree

1. Is the task trivial / single tool call / scan / transcription / classification? → **haiku**
2. Standard coding / research / writing / debugging / scaffolding? → **sonnet** (default)
3. Deep architecture / hard multi-file refactor / correctness-critical / novel? → **opus**
4. Adversarial bug hunt / security audit / can wait 10+ min? → **codex** (background only)
5. Time-sensitive (need result < 2 min)? → **never codex**, sonnet or haiku

## Parallelism Rules

**Dispatch in parallel when:**
- Subtasks are independent (different dirs, different topics)
- Multiple haiku scanning unrelated paths
- Multiple sonnet researching separate questions
- Send all dispatches in a single message, collect results

**Dispatch in background when:**
- Long-running work (codex, opus refactor)
- User can keep working while it runs
- Result doesn't block immediate next step

**Keep in main agent when:**
- Small fast iterative loops (write -> test -> fix)
- Round-trip overhead exceeds task duration
- You need the tool results in your own context

**Always:**
- Run subagents in background (`run_in_background: true`) so the main thread stays unblocked
- Lock codex model explicitly to `gpt-5.4-codex` on every invocation
- Send parallel dispatches in a single message, not sequential calls

**Never:**
- Run parallel agents on the same files without worktrees
- Dispatch codex for anything you're watching live
- Use opus for trivial tool calls
- Run subagents in foreground when background works

## Git Strategy

### Direct on current branch
- Change is < 50 lines, 1-2 files
- No parallel agents, pure linear flow
- Easily reversible
- **Example:** fix a typo, add a small util, edit a config

### Feature branch (`feat/<slug>`)
- Diff is large enough to bundle commits
- Single linear workstream, no overlapping agents
- Want to test/review before merging
- **Example:** `git checkout -b feat/auth-refresh` for new auth flow

### Git worktree
- Multiple subagents touching overlapping files
- Want experiment isolation while main work continues
- Long-running opus/codex shouldn't dirty working tree
- **Example:**
  ```bash
  git worktree add ../project-feat-api feat/api-layer
  # dispatch sonnet into ../project-feat-api
  # main agent stays unblocked in original tree
  git merge feat/api-layer
  git worktree remove ../project-feat-api
  ```

## Quick Reference

| task type | model | branch | parallel? | background? |
|---|---|---|---|---|
| transcription / extraction | haiku | current | yes | yes |
| file scan / grep | haiku | current | yes | no |
| bulk context gathering | haiku | current | yes | yes |
| research synthesis | sonnet | current | yes | optional |
| code review (logic/style) | sonnet | current | no | no |
| scaffolding new module | sonnet | feat | no | no |
| writing docs | sonnet | feat | no | no |
| debugging (interactive) | sonnet | current | no | no |
| large refactor | opus | worktree | no | yes |
| novel architecture | opus | feat | no | no |
| security design | opus | feat | no | no |
| adversarial code review | codex | feat | no | yes |
| bug hunt | codex | worktree | no | yes |
| security audit | codex | feat | no | yes |

## Codex-Specific Failure Modes

Codex is **too literal**. It will fulfill the exact request and break adjacent things. Real example from HN: asked to "fix compiler warnings", it made a bunch of values nullable to silence the warnings - technically correct, broke data integrity downstream.

**Codex will fail at (NEVER use it for):**
- **UI work of any kind** - layouts, components, styling, UX, design judgment. Codex has zero taste and no eye for visual hierarchy. Always claude.
- **Anything you need to discuss** - "yo i need to talk about this", brainstorming, exploration, "what do you think", trade-off conversations. Codex is a one-shot executor, not a collaborator. Always claude.
- **Reading user intent** - "make it better" → won't make leaps, will pick the most literal interpretation
- **Architecture/product judgment** - no sense of tradeoffs beyond what's written
- **Obscure libraries** - hallucinates rather than admitting unknown
- **Multi-file dependency chains** - gets stuck in circles
- **Continuous conversation context** - context degrades fast, not built for back-and-forth

**Codex will excel at:**
- Tasks where being literal is a feature (test writing, mechanical refactors)
- Clearly-bounded specs with acceptance criteria
- Adversarial passes with explicit focus arguments

**Prompt requirements for codex:**
- One concern per prompt (don't bundle)
- Explicit scope boundaries (which files, what behavior, what done looks like)
- Structured sections: General → Autonomy → Code Implementation → Editing Constraints → Exploration → Plan Tool
- Use the positional focus argument: `/codex:adversarial-review challenge whether this caching design was right`
- Heavy XML structure beats prose for multi-step specs

## Optimal Codex Workflow

The pattern that works (claude + codex hybrid):

1. **Claude builds** - architecture, exploration, back-and-forth iteration
2. **Codex reviews** - dispatch `/codex:adversarial-review` with a specific focus after a major change. 6-10 min wait, actionable findings
3. **Claude filters** - triage codex output: "which are real issues vs noise, which need design decisions"
4. **Codex fixes** - confirmed bugs with clear specs get delegated back to codex
5. **Batch maintenance** - queue codex with 3-5 P2 tasks at session start, work on real stuff while it churns

What doesn't work:
- Letting codex drive architecture decisions
- Vague task descriptions
- Auto-running codex on every claude response (drains usage fast)

## Anti-patterns

1. **Codex for time-sensitive work** - 10+ min latency kills interactive flow
2. **Codex with vague prompts** - it will interpret literally and break things
3. **Opus for trivial tool calls** - haiku does it at 5% cost
4. **Worktrees for non-overlapping single-agent work** - merge overhead with no upside
5. **Big refactors on main** - hard rollback, polluted history
6. **Parallel agents on same files without worktrees** - concurrent write conflicts
7. **Dispatching codex synchronously** - always background, never wait
8. **Forgetting `--model gpt-5.4-codex`** - default selection is unreliable
9. **Codex for UI work** - zero taste, no visual judgment, will produce bricks
10. **Codex when you need to discuss/brainstorm** - it's a one-shot executor, not a collaborator

## Examples

**Bulk context gathering:** 5 independent file reads → 5 haiku in parallel, single message dispatch.

**New feature scaffolding (small):** sonnet, current branch, no subagents.

**New feature scaffolding (big, multi-file):** sonnet, `feat/<slug>` branch, optional sonnet subagents for independent modules.

**Security audit before launch:** codex in background on `feat/audit` worktree, you keep building on main.

**Hard refactor across 20 files:** opus in worktree, you stay on main fixing other stuff, merge when done.

**"Find all references to X across the codebase":** haiku, current branch, no subagent (Grep tool is faster).
