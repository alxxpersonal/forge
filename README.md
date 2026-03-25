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
| [commit-forge](skills/commit-forge) | Clean, atomic git commits with conventional format |
| [prompt-forge](skills/prompt-forge) | Universal prompt engineering guide for Claude and GPT |
| [claude-headless](skills/claude-headless) | Build custom UIs on Claude Code's headless NDJSON protocol |
| [exo-teams](skills/exo-teams) | Microsoft Teams CLI automation - messages, files, assignments, no admin consent |

</details>

<details>
<summary><b>Charmbracelet Skills</b></summary>
<br>

Production-grade skills for the entire [charmbracelet](https://github.com/charmbracelet) Go TUI ecosystem. Built by reading actual v2 source code - real API, real patterns, no hallucinated functions.

| Skill | Description |
|-------|-------------|
| [charm-ecosystem](skills/charm-ecosystem) | Architect's guide - which libraries to combine, decision tree, integration cookbook |

**Libraries:**

| Skill | Description |
|-------|-------------|
| [charm-bubbletea](skills/charm-bubbletea) | Elm Architecture TUI framework - Model/Update/View, commands, subscriptions |
| [charm-lipgloss](skills/charm-lipgloss) | CSS-like terminal styling - colors, borders, layout, tables, lists, trees |
| [charm-bubbles](skills/charm-bubbles) | Pre-built TUI components - spinner, textinput, list, table, viewport, progress |
| [charm-huh](skills/charm-huh) | Terminal forms and prompts - input, select, confirm, validation, theming |
| [charm-glamour](skills/charm-glamour) | Stylesheet-based markdown rendering for terminal |
| [charm-harmonica](skills/charm-harmonica) | Physics-based spring animations for TUI |
| [charm-ultraviolet](skills/charm-ultraviolet) | Low-level terminal primitives powering bubbletea v2 |

**CLI Tools:**

| Skill | Description |
|-------|-------------|
| [charm-gum](skills/charm-gum) | Shell scripting UI - prompts, spinners, filters, styled output |
| [charm-glow](skills/charm-glow) | Terminal markdown viewer |
| [charm-vhs](skills/charm-vhs) | Record terminal sessions to GIF/MP4/WebM via .tape files |
| [charm-freeze](skills/charm-freeze) | Screenshot terminal output to PNG/SVG |
| [charm-pop](skills/charm-pop) | Send emails from terminal |
| [charm-fang](skills/charm-fang) | CLI starter kit wrapping Cobra with styled help and auto-versioning |

</details>

## Why

Forge skills are built on actual research - primacy effects, instruction decay curves, token budget math, and structural patterns extracted from arxiv papers and top open-source repos.

## License

[MIT](LICENSE)
