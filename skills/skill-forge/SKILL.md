---
name: skill-forge
description: Write or improve an LLM agent skill. Use when user says "create skill", "write a skill", "improve this skill", "new skill", "skill for X", or wants to build a SKILL.md file.
disable-model-invocation: true
argument-hint: "[skill name or path to existing SKILL.md]"
---

# Skill Forge

Generate or improve skills for LLM agent tools (Claude Code, Cursor, Crush, and any SKILL.md-compatible system). Research-backed, based on official docs, plugin source, and real-world skill audits.

## When to Use

- Creating a new skill from scratch
- Improving an existing skill's description, structure, or settings
- Auditing a skill for common mistakes
- Deciding between rule vs skill vs CLAUDE.md for a piece of context

## Step 1: Understand the Request

If $ARGUMENTS points to an existing SKILL.md, read it first. Otherwise, ask:

1. **What does it do?** - the core capability or workflow
2. **Who triggers it?** - user only (`disable-model-invocation: true`), model auto (`default`), or background (`user-invocable: false`)
3. **Side effects?** - does it commit, deploy, send messages, modify state?
4. **Scope?** - personal (`~/.<agent>/skills/`), project (`.<agent>/skills/`), or plugin?

## Step 2: Choose the Right Mechanism

These paths use `.claude/` as example but apply to any agent that supports SKILL.md format (Crush, Cursor, etc.):

| Put here | When |
|----------|------|
| CLAUDE.md / agent config | Rule applies to ALL work, every session, short |
| `.<agent>/rules/` | Applies only when touching specific file types (use `paths` frontmatter) |
| `.<agent>/skills/` | On-demand workflow, detailed reference, or behavioral mode |

Don't make a skill when a rule or CLAUDE.md line would suffice.

## Step 3: Write the Frontmatter

### Required fields

```yaml
---
name: skill-name          # lowercase, hyphens, max 64 chars
description: ...          # THE most important field. See below.
---
```

### Optional fields

| Field | Use when |
|-------|----------|
| `disable-model-invocation: true` | Side effects (commits, deploys), behavioral modes, timing-sensitive workflows |
| `user-invocable: false` | Background knowledge Claude should absorb passively |
| `argument-hint: "[hint]"` | Skill takes input - shows in autocomplete |
| `allowed-tools: Read, Grep` | Restrict or pre-approve tools |
| `context: fork` | Run in isolated subagent (research, batch ops, analysis) |
| `agent: Explore` | Subagent type when using `context: fork` |

### Invocation matrix

| Setting | User invokes | Claude invokes | Description in context |
|---------|-------------|---------------|----------------------|
| (default) | yes | yes | yes |
| `disable-model-invocation: true` | yes | no | no |
| `user-invocable: false` | no | yes | yes |

**Key insight:** with `disable-model-invocation: true`, the description is NOT in context at all. Don't over-optimize the description - write it for your own docs reference only.

## Step 4: Write the Description

The description is what Claude pattern-matches against to decide whether to load the skill. There is NO algorithmic routing - Claude's language model reads all descriptions and semantically matches.

### Formula

1. **What it does** - one sentence, outcome-first
2. **Trigger phrases** - exact words: "Use when user says X, Y, Z"
3. **Anti-triggers** - "NOT for..." if false positives are a risk
4. **Prerequisites** - required setup if any

### Rules

- Under 60 words
- Every word costs token budget (2% of context window shared across ALL skill descriptions)
- Description handles matching only - body handles instructions
- Don't repeat body content in description
- Be slightly "pushy" - Claude undertriggers by default

### Good example

> "Translates Figma designs into production-ready code with 1:1 visual fidelity. Use when implementing UI from Figma files, when user mentions 'implement design', 'generate code', 'build Figma design', or provides Figma URLs. Requires Figma MCP server connection."

### Bad example

> "Use when user wants to plan something" - too vague, false positives everywhere.

## Step 5: Write the Body

### Structure

```markdown
# Skill Title

Brief context sentence.

## When to Use
(if not covered by description)

## Process / Steps
1. Step one
2. Step two

## Rules / Constraints
- Rule one
- Rule two

## Examples (if applicable)
Show patterns Claude should follow.

## Validation / Checklist (if applicable)
How to verify output quality.
```

### Rules

- **Under 500 lines.** Full SKILL.md loads into context when invoked. Move reference material to supporting files.
- **Use $ARGUMENTS** if the skill takes input. Without it, Claude Code appends `ARGUMENTS: <value>` at the end which is messier.
- **Numbered lists** for sequential steps, bullets for options/rules.
- **Bold critical constraints** with `**IMPORTANT:**`.
- **Say "do not skip steps"** explicitly when order matters.
- **H2 for major sections, H3 for sub-sections.** Max 2 heading levels.

### Supporting files

Put adjacent to SKILL.md in the skill directory:
- `examples.md` - example inputs/outputs
- `reference.md` - detailed reference docs
- `scripts/helper.sh` - executable scripts

Reference them from SKILL.md so Claude knows to load them. Files not accessed consume zero tokens.

### Dynamic content injection

Skills support shell command output injection with `!` prefix:
- `!`\`git branch --show-current\` - inject current branch
- `!`\`ls src/\` - inject file listing

Use for context that changes between invocations.

## Step 6: Validate

Run this checklist on the generated skill:

- [ ] Description under 60 words
- [ ] Description is trigger-focused (what/when), not instruction-focused (how)
- [ ] Body under 500 lines
- [ ] No duplication between description and body
- [ ] Correct invocation setting for the use case (side effects = `disable-model-invocation: true`)
- [ ] `$ARGUMENTS` used if skill takes input
- [ ] No vague trigger phrases that cause false positives
- [ ] No `context: fork` on reference-only content (subagent gets instructions with no task)
- [ ] Critical constraints are bolded
- [ ] Steps are numbered when order matters
- [ ] Supporting files referenced if body would exceed 500 lines

## Common Mistakes

1. **Vague descriptions** - "Use when user wants to plan" triggers on everything
2. **Missing `disable-model-invocation: true` on side-effect skills** - Claude will auto-commit, auto-deploy
3. **Body too long** - 500+ lines burns context. Split into supporting files.
4. **Redundant description + body** - description matches, body instructs. Don't duplicate.
5. **`context: fork` on reference content** - subagent gets instructions with no actionable task
6. **Over-explaining in description** - the description isn't the instructions
7. **Missing NOT conditions** - for behavioral modes, add explicit "NEVER activate unless..."
8. **`user-invocable: false` on things users should manually trigger** - confusing
