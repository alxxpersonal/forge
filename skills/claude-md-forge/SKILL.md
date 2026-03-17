---
name: claude-md-forge
description: Generate or optimize a CLAUDE.md and .claude/ infrastructure for a repository. Use when user says "create CLAUDE.md", "optimize CLAUDE.md", "set up .claude", "audit my CLAUDE.md", or is bootstrapping a new project.
argument-hint: "[repo path or leave blank for current]"
---

# CLAUDE.md Forge

Generate the best possible CLAUDE.md and .claude/ infrastructure for a repository. Research-backed principles, patterns from top open-source repos.

## When to Use

- Bootstrapping a new project
- Optimizing an existing CLAUDE.md
- Setting up .claude/ directory (rules, skills, settings, hooks)
- Auditing current CLAUDE.md for anti-patterns

## Process

Target repository: $ARGUMENTS (if blank, use current working directory).

### Step 1: Analyze the Repository

Read the codebase to understand:
- Languages and frameworks used
- Build system and package manager
- Test framework and commands
- Project structure (monorepo vs single package)
- Existing CLAUDE.md, .claude/, AGENTS.md, CONTRIBUTING.md
- CI/CD setup
- Linting and formatting tools

### Step 2: Apply Research-Backed Principles

The following principles are derived from 23+ academic papers and official Anthropic documentation.

#### Attention and Positioning

**Primacy effect dominates.** Evidence shows LLMs attend most strongly to the beginning and end of context, with 30%+ accuracy drop for middle content. The primacy effect is stronger than recency.

**Action:** Put the 3-5 most critical behavioral rules at the TOP of CLAUDE.md. Put the "Do NOT" prohibitions at the BOTTOM. Reference material goes in the middle where lower attention is acceptable.

#### Length and Density

**More context hurts even with perfect retrieval.** Evidence shows performance degrades 13-85% as input length increases, regardless of content quality.

**Instruction following degrades with count.** Studies across 20 frontier models show threshold decay at ~150 instructions for reasoning models, linear decay for Claude Sonnet. Errors shift from "wrong thing" to "forgetting entirely" above ~100 instructions.

**Action:** Target under 200 lines per CLAUDE.md (official Anthropic guidance). Target 30-70 distinct instructions (sweet spot). Apply the pruning test: "Would removing this line cause Claude to make mistakes?" If no, cut it.

#### Formatting

**Markdown formatting improves performance up to 40%** compared to unformatted text for Claude-class models.

**Action:** Use `##` headers for sections, `-` bullets for lists, backtick code blocks for commands. Max 2 levels of heading nesting.

#### Instruction Framing

**Negative instructions are disproportionately forgotten** under cognitive load. Evidence shows excessive constraints can cause advanced models to over-focus on avoidance instead of achieving goals.

**Positive instructions outperform negative ones.** "All SQL in .sql files via QueryLoader" beats "Do NOT write inline SQL."

**Action:** State rules positively in their relevant sections. Reserve "Do NOT" for 3-5 genuinely critical prohibitions that have no positive equivalent. Use ALWAYS/NEVER/PREFER/AVOID verb prefixes for unambiguous scanning (astral-sh/uv pattern).

#### Context Persistence

**CLAUDE.md survives compaction.** It is re-injected fresh on every API request. It is the single most durable piece of project context.

**At 70% context utilization, instruction precision drops.** At 90%+, behavior becomes erratic. Shorter CLAUDE.md = more room for actual work.

**Action:** Keep CLAUDE.md short. Add a "Compact Instructions" section telling the compaction process what to preserve.

#### Instruction Hierarchy

**System/user priority is unreliable.** Even simple formatting conflicts produce inconsistent behavior in choosing which instruction wins across all major LLMs (Control Illusion, 2025).

**Action:** Make CLAUDE.md instructions non-conflicting with likely user requests. Do not depend on CLAUDE.md always overriding user messages.

### Step 3: Apply Structural Patterns from Top Repos

Based on analysis of React (240k stars), Deno (103k stars), PyTorch (90k stars), uv (55k stars), Next.js (138k stars), Biome (17k stars), Crush (21k stars), LangChain (105k stars), and Anthropic's own repos.

#### CLAUDE.md Structure (target 80-150 lines)

```markdown
# Project Name (use proper capitalized display name, e.g. "Exo Teams" not "exo-teams")
One-line description.

## Critical Rules
3-5 ALWAYS/NEVER rules that matter most. Primacy effect = maximum attention here.

## Architecture
Stack decisions that affect daily coding. 5-10 bullets, NOT a directory tree.
Claude explores filesystems natively - do not waste lines on project structure.

## Stack Decisions (Locked)
Prevent the model from second-guessing locked choices.

## Commands
Build, test, lint, deploy. Only commands Claude cannot guess from the codebase.

## Implementation Pitfalls
What WILL break if you are not careful. Non-obvious gotchas.
(Pattern from Anthropic's claude-code-action repo.)

## Commit Style
Convention + any repo-specific rules. NEVER add co-author tags.

## Compact Instructions
Tell the compaction process what to preserve (e.g. "always keep: current task, file paths being edited, test results, architectural decisions made this session").

## Do NOT
3-5 hard prohibitions. Benefits from recency effect at bottom of file.
```

#### What to EXCLUDE from CLAUDE.md

These are explicitly called out by Anthropic's official documentation as anti-patterns:

- Project structure trees (Claude explores filesystems)
- Standard language conventions Claude already knows
- Detailed API documentation (link to docs instead)
- Information that changes frequently (entity types, enum values)
- Long explanations or tutorials
- File-by-file codebase descriptions
- Self-evident practices ("write clean code")
- Code conventions details (put in path-scoped rules instead)

#### .claude/ Directory Infrastructure

Before generating, **analyze the codebase** and determine which path-scoped rules are needed based on:
- What languages/frameworks exist in the repo
- What file patterns need specific conventions
- What testing patterns are used
- What database/query patterns exist

Only create rules for languages and patterns that actually exist in the codebase. Don't generate a `go.md` rule for a Python-only project.

```
.claude/
├── settings.json          # hooks, deny list, project metadata
├── rules/                 # path-scoped rules (auto-load by file pattern)
│   ├── python.md          # paths: ["**/*.py"] - only if Python in repo
│   ├── typescript.md      # paths: ["**/*.ts", "**/*.tsx"] - only if TS in repo
│   ├── go.md              # paths: ["**/*.go"] - only if Go in repo
│   ├── sql.md             # paths: ["**/queries/**/*.sql"] - only if SQL queries exist
│   └── testing.md         # paths: ["**/tests/**", "**/*_test.*"] - only if tests exist
└── skills/                # on-demand invokable skills
    └── code-conventions/
        └── SKILL.md       # full style guide with examples
```

Each rule file should contain the **specific conventions, linters, formatters, and patterns** for that language as used in THIS codebase. Read existing code to extract the actual patterns being followed, don't assume generic defaults.

**settings.json patterns to include:**

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|MultiEdit",
        "command": "auto-format.sh \"$CLAUDE_FILE_PATH\""
      }
    ]
  },
  "permissions": {
    "deny": ["wrong-package-manager", "git push --force", "git reset --hard"]
  }
}
```

The PostToolUse auto-format hook (from uv, React, Anthropic repos) eliminates "run the formatter" reminders entirely. The deny list prevents wrong tools. Detect the actual formatter from the repo (ruff for Python, gofmt for Go, prettier for TS/JS, biome, etc.) and wire it into the auto-format script.

**Path-scoped rules vs CLAUDE.md vs skills:**

| Put here | When |
|----------|------|
| CLAUDE.md | Rule applies to ALL work in the repo, every session |
| .claude/rules/ | Rule applies only when touching specific file types |
| .claude/skills/ | Detailed reference or workflow invoked on demand |

Rules are more token-efficient than CLAUDE.md because they only load when relevant. A python style rule does not burn context during Go work.

### Step 4: Generate the Files

1. Write CLAUDE.md following the structure above
2. Create settings.json with auto-format hook and deny list
3. Create path-scoped rules for each language/domain in the repo
4. Optionally create skills for complex workflows
5. Create the auto-format script if the repo has linters/formatters

### Step 5: Validate

- Count lines: CLAUDE.md should be under 200
- Count distinct instructions: should be 30-70
- Apply pruning test to every line
- Check for duplication across CLAUDE.md, rules, and skills
- Verify no project structure trees or enumerable data that will go stale
- Confirm critical rules are at the top, prohibitions at the bottom

