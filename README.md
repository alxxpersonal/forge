# Forge

**Research-backed skills and configs for LLM agent tools.**

## Overview

Forge is a toolkit of **battle-tested skills, prompts, and agent configurations** built from reverse engineering top AI products and studying arxiv papers on LLM instruction following.

> Portable across projects. Works with Claude Code, Codex, Cursor, and any SKILL.md-compatible agent.


## Skills

### Install

```bash
# All skills (works with Claude Code, Cursor, Copilot, Codex, 40+ agents)
npx skills add https://github.com/alxxpersonal/forge

# Specific skill
npx skills add https://github.com/alxxpersonal/forge --skill claude-md-forge

# Or manual
git clone https://github.com/alxxpersonal/forge.git
ln -s $(pwd)/forge/skills/* ~/.claude/skills/
```


<details>
<summary><b>Available Skills</b></summary>
<br>

| Skill | Description |
|-------|-------------|
| [claude-md-forge](skills/claude-md-forge) | Optimized CLAUDE.md and .claude/ infrastructure |
| [skill-forge](skills/skill-forge) | Skills for any LLM agent tool |
| [readme-forge](skills/readme-forge) | READMEs in a consistent, dense style |

</details>

## Why

Forge skills are built on actual research - primacy effects, instruction decay curves, token budget math, and structural patterns extracted from arxiv papers and top open-source repos.

## License

[MIT](LICENSE)
