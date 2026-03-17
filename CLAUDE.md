# Forge

Research-backed skills and configs for LLM agent tools.

## Critical Rules

- ALWAYS add new skills to the Available Skills table in README.md when creating them
- ALWAYS follow the SKILL.md frontmatter spec (name, description, optional: disable-model-invocation, argument-hint)
- NEVER add co-author tags to commits
- NEVER commit without explicit ask

## Architecture

- `skills/` - each skill is a directory with `SKILL.md` following the Agent Skills open standard
- Skills are portable across Claude Code, Cursor, Copilot, Codex, Crush, and any SKILL.md-compatible agent
- Each skill is self-contained - no cross-skill dependencies

## Stack Decisions (Locked)

- Skills use SKILL.md format (YAML frontmatter + markdown body)
- No build step, no compilation, pure markdown
- MIT licensed

## Commands

No build system. Skills are markdown files.

```
# Test a skill locally
cp -r skills/<name> ~/.claude/skills/

# Symlink for dev
ln -s $(pwd)/skills/<name> ~/.claude/skills/<name>
```

## Implementation Pitfalls

- Skill descriptions over 60 words waste token budget
- `disable-model-invocation: true` removes description from context entirely - don't over-optimize descriptions for these
- Skill body over 500 lines should be split into supporting files
- Don't duplicate content between description and body

## Commit Style

Conventional commits: `type(scope): description`

NEVER add co-author tags.

## Compact Instructions

Always keep: current skill being edited, research findings being applied, README.md update status.

## Do NOT

- Create skills without adding them to README.md Available Skills table
- Write descriptions longer than 60 words
- Add Claude-specific language when the skill works across agents
- Use long dashes
- Add AI slop phrases
