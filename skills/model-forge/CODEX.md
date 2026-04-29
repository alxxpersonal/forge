# Codex Reference

Detailed guidance for using gpt-5.4 (codex) effectively in a claude + codex hybrid workflow.

## Failure Modes

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

## Prompt Requirements

- One concern per prompt (don't bundle)
- Explicit scope boundaries (which files, what behavior, what done looks like)
- Structured sections: General → Autonomy → Code Implementation → Editing Constraints → Exploration → Plan Tool
- Use the positional focus argument: `/codex:adversarial-review challenge whether this caching design was right`
- Heavy XML structure beats prose for multi-step specs

## Optimal Workflow

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
