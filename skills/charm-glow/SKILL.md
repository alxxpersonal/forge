---
name: charm-glow
description: "View and browse markdown files in the terminal with glow - CLI and TUI modes, pager, word wrapping, styles. Use when viewing markdown in the terminal, glow, or browsing markdown files from the command line. NOT for rendering markdown programmatically in Go (use glamour)."
---

# charm-glow

`glow` renders markdown in the terminal with styled output. Has two modes: TUI (browse/stash) and CLI (direct render).

## CLI Usage

```bash
# Read a local file
glow README.md

# Read from stdin
echo "# Hello" | glow -

# Fetch from GitHub/GitLab
glow github.com/charmbracelet/glow

# Fetch from URL
glow https://host.tld/file.md
```

## Pager Mode

```bash
# Enable pager (defaults to less -r if $PAGER not set)
glow -p README.md
```

## Word Wrap

```bash
# Wrap at N columns
glow -w 80 README.md
```

## Styles

```bash
# Auto-detect from terminal background (default)
glow README.md

# Force dark or light
glow -s dark README.md
glow -s light README.md

# Custom JSON stylesheet (glamour format)
glow -s mystyle.json README.md
```

Styles come from [glamour](https://github.com/charmbracelet/glamour/blob/master/styles/gallery/README.md). Custom styles are JSON files following glamour's schema.

## TUI Mode

Run `glow` with no args to open the TUI. It scans the current directory (or git repo root) for `.md` files.

- Navigate with arrow keys / vim keys
- Open file to enter pager, press `?` for hotkeys
- Pager uses `less`-compatible keybindings

## Config File

```bash
glow config   # opens $EDITOR
```

Config lives at platform default path (check `glow --help`). Example `glow.yml`:

```yaml
style: "auto"       # "dark" | "light" | "auto" | path to JSON
pager: true         # always use pager
width: 80           # word wrap column
mouse: true         # mouse support in TUI
all: false          # show hidden/gitignored files
showLineNumbers: false
preserveNewLines: false
```

## Key Flags Summary

| Flag | Description |
|------|-------------|
| `-s` | Style: dark, light, auto, or path to JSON |
| `-w` | Word wrap width |
| `-p` | Enable pager |

## Notes

- Stash feature (syncing to Charm Cloud) was removed in v2
- `glow --help` lists config file location per platform
