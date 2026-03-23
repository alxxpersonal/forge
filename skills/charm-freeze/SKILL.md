---
name: charm-freeze
description: "Generate PNG, SVG, or WebP screenshots of code and terminal output with freeze. Use when screenshotting code, freeze, terminal-to-image, or capturing styled code snippets as images."
---

# charm-freeze

Generate images of code and terminal output using `freeze`.

## Basic Usage

```bash
# screenshot a file (auto-detects language)
freeze main.go -o main.png

# pipe code in
cat artichoke.hs | freeze -o out.svg

# capture live terminal command output
freeze --execute "eza -lah" -o eza.png
```

## Output Formats

Default output is `freeze.png`. Supports `.svg`, `.png`, `.webp`.

```bash
freeze main.go -o out.svg
freeze main.go -o out.png
freeze main.go -o out.webp

# all at once
freeze main.go -o out.{svg,png,webp}
```

If piped (e.g. `freeze main.go > out.svg`), outputs to stdout.

## Language Detection

Auto-detects from filename or content. Override with `--language`:

```bash
cat script.sh | freeze --language bash
freeze file.txt --language python
```

## Themes

```bash
freeze main.go --theme dracula
freeze main.go --theme charm       # default
freeze main.go --theme github
```

## Fonts

Defaults: JetBrains Mono, 14px, 1.2 line-height.

```bash
freeze main.go \
  --font.family "SF Mono" \
  --font.size 16 \
  --line-height 1.4

# embed a font file (TTF, WOFF, WOFF2)
freeze main.go --font.file ./FiraCode.ttf

# enable ligatures
freeze main.go --font.ligatures
```

## Decorations

```bash
# window controls (macOS style)
freeze main.go --window

# rounded corners
freeze main.go --border.radius 8

# border outline
freeze main.go --border.width 1 --border.color "#515151" --border.radius 8

# drop shadow
freeze main.go --shadow.blur 20 --shadow.x 0 --shadow.y 10

# background color
freeze main.go --background "#08163f"
```

## Layout

```bash
# padding (1, 2, or 4 values)
freeze main.go --padding 20
freeze main.go --padding 20,40
freeze main.go --padding 20,60,20,40   # top right bottom left

# margin (same syntax)
freeze main.go --margin 20,40

# fixed height
freeze main.go --height 400
```

## Line Numbers

```bash
freeze main.go --show-line-numbers

# capture specific lines only
freeze main.go --show-line-numbers --lines 10,25
```

## Configurations

Three built-in presets:

```bash
freeze -c base main.go    # minimal
freeze -c full main.go    # macOS-like, shadow + window controls
freeze -c user main.go    # your saved config (~/.config/freeze/user.json)

# custom JSON config
freeze -c ./custom.json main.go
```

Example `custom.json`:

```json
{
  "window": true,
  "border": { "radius": 8, "width": 1, "color": "#515151" },
  "shadow": { "blur": 20, "x": 0, "y": 10 },
  "padding": [20, 40, 20, 20],
  "font": { "family": "JetBrains Mono", "size": 14 },
  "line_height": 1.2
}
```

## Interactive Mode

```bash
freeze --interactive
```

Opens a TUI to tweak all settings visually. Saves result to `~/.config/freeze/user.json`.

## Screenshot TUIs

Capture a running TUI via tmux:

```bash
tmux capture-pane -pet 1 | freeze -c full -o helix.png
```

## Wrap Long Lines

```bash
freeze main.go --wrap 80
```
