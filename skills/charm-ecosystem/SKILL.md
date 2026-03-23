---
name: charm-ecosystem
description: "Architect's guide to the charmbracelet Go TUI ecosystem - which libraries to combine, dependency hierarchy, integration patterns. Use when choosing charm libraries, planning TUI architecture, combining bubbletea/lipgloss/bubbles/huh, or asking 'which charm library for X'. NOT for specific library API details (use individual charm-* skills)."
argument-hint: "[what you're building or which libraries to combine]"
---

# Charm Ecosystem

The architect's guide to charmbracelet's terminal UI ecosystem. Individual charm-* skills teach HOW to use each library. This skill teaches WHICH libraries to combine and WHY.

**$ARGUMENTS context**: If a specific project type or library question is mentioned, focus on that decision path.

## Ecosystem Map

### Go Libraries (code you import)

| Library | Import Path | Role |
|---------|-------------|------|
| **bubbletea** | `charm.land/bubbletea/v2` | TUI framework - Elm architecture (Model/Update/View) |
| **lipgloss** | `charm.land/lipgloss/v2` | Terminal styling - colors, borders, layout, tables, lists, trees |
| **bubbles** | `charm.land/bubbles/v2/*` | Pre-built TUI components - spinner, textinput, list, table, viewport |
| **huh** | `charm.land/huh/v2` | Interactive forms - input, select, confirm, validation |
| **glamour** | `charm.land/glamour/v2` | Markdown-to-ANSI rendering |
| **harmonica** | `github.com/charmbracelet/harmonica` | Physics-based animation - spring oscillator, projectile |
| **ultraviolet** | `github.com/charmbracelet/ultraviolet` | Low-level terminal primitives - cell buffers, screen management |
| **fang** | `charm.land/fang/v2` | Cobra wrapper - styled help, auto versioning, manpages |

### CLI Tools (binaries you run)

| Tool | Role |
|------|------|
| **gum** | Shell script UI - prompts, filters, spinners, styled output |
| **glow** | Markdown viewer - CLI and TUI browser |
| **vhs** | Terminal demo recorder - .tape scripts to GIF/MP4 |
| **freeze** | Code/terminal screenshot - PNG/SVG/WebP |
| **pop** | Send email from terminal - TUI and CLI modes |

### Dependency Hierarchy

```
ultraviolet (cell-level primitives)
  |
  +-- lipgloss v2 (styling, layout, composition)
  |
  +-- bubbletea v2 (framework: Elm architecture, commands)
        |
        +-- bubbles v2 (components: spinner, list, table, viewport...)
        |     |
        |     +-- harmonica (physics animations, optional addition to any bubbletea app)
        |     +-- lipgloss v2 (component styling)
        |
        +-- huh v2 (forms, standalone or embedded in bubbletea)
              |
              +-- bubbles v2 (huh uses bubbles internally)
              +-- lipgloss v2 (theming)
```

Go libraries: `charm.land/*` vanity imports. CLI tools: `github.com/charmbracelet/*` or Homebrew.

## Decision Tree

### "I want to build a full TUI app"

**Use: bubbletea + lipgloss + bubbles**

The standard stack. Bubbletea provides the Elm architecture runtime, lipgloss handles all styling and layout, bubbles gives you pre-built components (list, table, viewport, spinner, textinput, progress, etc).

Add huh if you need form flows. Add harmonica if you need animated transitions.

### "I want shell script UI / interactive bash prompts"

**Use: gum**

No Go code needed. Gum provides choose, filter, input, confirm, spin, style, join, file, pager, table, log. All output to stdout for shell capture.

### "I want terminal forms"

**Go app: huh** (standalone or embedded in bubbletea)
**Shell script: gum** (input, choose, confirm commands)

Huh runs standalone with `.Run()` for simple cases, or embeds in bubbletea as a `tea.Model` for complex apps. Use gum when you're in bash/zsh and don't want to write Go.

### "I want to render markdown in the terminal"

**Programmatically (Go code): glamour**
**View files from CLI: glow**

Glamour is a library - import it, call `glamour.Render()`. Glow is a tool - run `glow README.md`.

### "I want animations in my TUI"

**Use: harmonica + bubbletea**

Harmonica provides spring (damped oscillator) and projectile physics. Drive with `tea.Tick` at 60fps. Create `NewSpring` once, call `Update` per frame.

### "I want an SSH-accessible TUI"

**Use: bubbletea + wish**

Wish (not covered by individual skills) wraps bubbletea for SSH serving. Wish provides SSH middleware that wraps your bubbletea handler per-connection. See the wish repo for setup.

### "I want to record terminal demos"

**Use: vhs** (optionally with gum for the demo content)

Write a `.tape` script with Type/Enter/Sleep/Wait commands. VHS renders to GIF/MP4/WebM. Use `gum` commands in your tape for pretty interactive demos.

### "I want terminal screenshots"

**Use: freeze**

Pipe code or use `--execute` to capture command output. Outputs PNG/SVG/WebP with syntax highlighting, window decorations, shadows.

### "I want a CLI framework with styled help"

**Use: fang + cobra**

Fang wraps Cobra - you still write `*cobra.Command` structs. Fang adds lipgloss-styled help, auto version from build info, manpage generation, signal handling.

### "I want to send emails from the terminal"

**Use: pop**

TUI and CLI modes. Supports SMTP and Resend API. Pipe body from stdin, attach files.

### "I need cell-level terminal control"

**Use: ultraviolet** (only if bubbletea is insufficient)

Low-level primitives: cell buffers, screen management, input decoding. Powers bubbletea and lipgloss internally. Unstable API. Only use for custom renderers or framework-building.

## Architecture Patterns

### The Elm Architecture for Go Developers

Bubbletea follows The Elm Architecture (TEA), a unidirectional data flow:

1. **Model** - a struct holding ALL application state (like a Redux store)
2. **Update(msg) -> (model, cmd)** - pure function: receives a message, returns new state + side effect
3. **View(model) -> string** - pure function: renders state to terminal output
4. **Cmd** - a function that performs I/O and returns a Msg back to Update

The runtime owns the loop. You never mutate state outside Update, never call View yourself, never do I/O in Update.

**Go-specific mental model**: think of it like an HTTP handler. The request is a `Msg`, the response is the new model + a `Cmd` to run next. The framework calls your handler, not the other way around.

### Parent-Child Component Composition

Children are just fields on the parent model. Each child has its own Update/View. Parent delegates:

```go
type parent struct {
    header textinput.Model
    body   viewport.Model
    footer help.Model
}

func (p parent) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    // Forward to all children, collect commands
    var cmd tea.Cmd
    p.header, cmd = p.header.Update(msg)
    cmds = append(cmds, cmd)

    p.body, cmd = p.body.Update(msg)
    cmds = append(cmds, cmd)

    return p, tea.Batch(cmds...)
}
```

Children self-filter messages (spinners ignore wrong IDs, unfocused inputs ignore keys). Forward everything, let children decide.

### Message Passing Between Components

Components communicate through the parent via typed messages:

```go
// Child emits a "done" message
type formDoneMsg struct{ name string }

// Parent catches it and routes to another child
case formDoneMsg:
    m.body.SetContent("Welcome, " + msg.name)
    m.state = stateMain
```

No direct child-to-child communication. Always go through the parent's Update.

### When to Use What

| Scenario | Use |
|----------|-----|
| Full interactive TUI with state | bubbletea + bubbles + lipgloss |
| Simple form/prompt then exit | huh standalone (`.Run()`) |
| Form inside a larger TUI | huh embedded in bubbletea |
| One-off styled output (no interaction) | lipgloss only |
| Shell script needs user input | gum |
| Quick markdown preview | glow CLI |
| Markdown in a Go app | glamour (+ viewport for scrolling) |

## v2 Migration Quick Reference

The ecosystem moved from `github.com/charmbracelet/*` to `charm.land/*` vanity imports.

### bubbletea v1 -> v2

| v1 | v2 |
|----|-----|
| `github.com/charmbracelet/bubbletea` | `charm.land/bubbletea/v2` |
| `View() string` | `View() tea.View` (use `tea.NewView(s)`) |
| `tea.KeyMsg` | `tea.KeyPressMsg` |
| `tea.WithAltScreen()` program option | `v.AltScreen = true` in View |
| `tea.WithMouseCellMotion()` | `v.MouseMode = tea.MouseModeCellMotion` in View |
| `tea.WindowSizeMsg` `Width`/`Height` | Same, unchanged |

### lipgloss v1 -> v2

| v1 | v2 |
|----|-----|
| `github.com/charmbracelet/lipgloss` | `charm.land/lipgloss/v2` |
| `lipgloss.Color("#F00")` is a type | `lipgloss.Color("#F00")` is a function returning `color.Color` |
| `lipgloss.AdaptiveColor{}` | `lipgloss.LightDark(hasDark)` |
| `style.Copy()` | Just assign (value type) |
| `lipgloss.NewRenderer()` | Removed. No renderers in v2. |
| Sub-packages at github path | `charm.land/lipgloss/v2/{table,list,tree}` |

### bubbles v1 -> v2

| v1 | v2 |
|----|-----|
| `github.com/charmbracelet/bubbles/*` | `charm.land/bubbles/v2/*` |
| `tea.KeyMsg` matching | `tea.KeyPressMsg` + `key.Matches()` |

### huh v1 -> v2

| v1 | v2 |
|----|-----|
| `github.com/charmbracelet/huh` | `charm.land/huh/v2` |
| `huh.NewTheme()` | `huh.ThemeFunc(fn)` |

### glamour v1 -> v2

| v1 | v2 |
|----|-----|
| `github.com/charmbracelet/glamour` | `charm.land/glamour/v2` |
| `WithAutoStyle()` | Removed. Use `WithStandardStyle("dark")` |
| Auto color detection | Pure output. Use lipgloss for downsampling. |

## Integration Cookbook

Five multi-library combinations with working code. See [references/cookbook.md](references/cookbook.md):

1. **Standard TUI app** - bubbletea + lipgloss + bubbles (list with styled header/footer)
2. **Forms in TUI** - bubbletea + huh (form then result display)
3. **Markdown viewer** - bubbletea + glamour + viewport (scrollable rendered markdown)
4. **Animated transitions** - bubbletea + harmonica (spring-animated position)
5. **Scripted demo** - gum + vhs (tape file demoing a gum script)

## Checklist

- [ ] Import paths use `charm.land/*/v2` (not `github.com/charmbracelet/*`)
- [ ] harmonica still uses `github.com/charmbracelet/harmonica` (no vanity import)
- [ ] ultraviolet still uses `github.com/charmbracelet/ultraviolet` (no vanity import)
- [ ] View returns `tea.NewView(s)` not raw string (bubbletea v2)
- [ ] AltScreen/MouseMode set in View, not as Program options (bubbletea v2)
- [ ] Colors use `lipgloss.Color()` function, not type literal (lipgloss v2)
- [ ] WindowSizeMsg handled for responsive layout
- [ ] huh forms embedded via Init/Update/View, not `.Run()`, when inside bubbletea
