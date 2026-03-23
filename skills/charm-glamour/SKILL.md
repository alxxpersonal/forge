---
name: charm-glamour
description: "Render markdown to styled ANSI terminal output in Go with glamour v2. Use when rendering markdown programmatically in Go, glamour, terminal markdown rendering, or styled markdown output. NOT for viewing markdown files in terminal (use glow)."
---

# glamour - Terminal Markdown Rendering

`charm.land/glamour/v2` renders markdown to styled ANSI output. Built on goldmark, supports GFM (tables, task lists, strikethrough), syntax highlighting via Chroma, emoji, and fully customizable stylesheets.

## Quick Start

```bash
go get charm.land/glamour/v2@latest
```

### One-liner

```go
import "charm.land/glamour/v2"

out, err := glamour.Render("# Hello\n\nSome **bold** text.", "dark")
fmt.Print(out)
```

### With renderer (reusable)

```go
r, err := glamour.NewTermRenderer(
    glamour.WithStandardStyle("dark"),
    glamour.WithWordWrap(80),
)
if err != nil {
    log.Fatal(err)
}

out, err := r.Render(markdown)
fmt.Print(out)
```

## Core API

### Package-level functions

| Function | Description |
|---|---|
| `Render(in, stylePath string) (string, error)` | One-shot render with a style name or file path |
| `RenderBytes(in []byte, stylePath string) ([]byte, error)` | Same but bytes in/out |
| `RenderWithEnvironmentConfig(in string) (string, error)` | Uses `GLAMOUR_STYLE` env var, defaults to `"dark"` |

### TermRenderer

Created via `NewTermRenderer(options ...TermRendererOption)`. Reusable for multiple renders.

**Methods:**

| Method | Description |
|---|---|
| `Render(in string) (string, error)` | Render markdown string |
| `RenderBytes(in []byte) ([]byte, error)` | Render markdown bytes |
| `Write(b []byte) (int, error)` | Implements `io.Writer`, buffer markdown input |
| `Close() error` | Flush buffered input, call before `Read` |
| `Read(b []byte) (int, error)` | Implements `io.Reader`, read rendered output |

**io.ReadWriter pattern** (streaming):

```go
r, _ := glamour.NewTermRenderer(glamour.WithWordWrap(80))
r.Write([]byte("# Streamed\n\nContent here."))
r.Close()

rendered, _ := io.ReadAll(r)
fmt.Print(string(rendered))
```

### Options

| Option | Description |
|---|---|
| `WithStandardStyle(name string)` | Use a built-in style by name |
| `WithStylePath(path string)` | Style name OR path to JSON file |
| `WithStyles(cfg ansi.StyleConfig)` | Programmatic style struct |
| `WithStylesFromJSONBytes(b []byte)` | Parse style from JSON bytes |
| `WithStylesFromJSONFile(path string)` | Load style from JSON file |
| `WithEnvironmentConfig()` | Use `GLAMOUR_STYLE` env var |
| `WithWordWrap(width int)` | Word wrap width (default: 80) |
| `WithTableWrap(wrap bool)` | Wrap table content (default: true). False truncates with ellipsis |
| `WithInlineTableLinks(inline bool)` | Render links inline in tables instead of footer list |
| `WithPreservedNewLines()` | Keep newlines instead of reflowing |
| `WithEmoji()` | Enable `:emoji_code:` rendering |
| `WithBaseURL(url string)` | Resolve relative URLs against this base |
| `WithChromaFormatter(fmt string)` | Set Chroma formatter for code blocks |
| `WithOptions(opts ...TermRendererOption)` | Combine multiple options |

### Built-in styles

| Constant | String | Use case |
|---|---|---|
| `styles.DarkStyle` | `"dark"` | Dark terminal backgrounds (default) |
| `styles.LightStyle` | `"light"` | Light terminal backgrounds |
| `styles.DraculaStyle` | `"dracula"` | Dracula color scheme |
| `styles.TokyoNightStyle` | `"tokyo-night"` | Tokyo Night color scheme |
| `styles.PinkStyle` | `"pink"` | Pink accent theme |
| `styles.AsciiStyle` | `"ascii"` | ASCII-only, no unicode box chars |
| `styles.NoTTYStyle` | `"notty"` | No ANSI codes at all, plain text |

Each has a corresponding `StyleConfig` variable: `styles.DarkStyleConfig`, `styles.LightStyleConfig`, etc.

## Common Patterns

### Custom style (programmatic)

Start from a built-in config and modify fields. Style fields use pointers for optional values.

```go
import (
    "charm.land/glamour/v2"
    "charm.land/glamour/v2/ansi"
    "charm.land/glamour/v2/styles"
)

func boolPtr(b bool) *bool    { return &b }
func strPtr(s string) *string { return &s }
func uintPtr(u uint) *uint    { return &u }

func customRenderer() (*glamour.TermRenderer, error) {
    style := styles.DarkStyleConfig

    // Custom H1: green text, no background
    style.H1 = ansi.StyleBlock{
        StylePrimitive: ansi.StylePrimitive{
            Color:  strPtr("34"),
            Bold:   boolPtr(true),
            Prefix: "# ",
        },
    }

    // Wider margins
    style.Document.Margin = uintPtr(4)

    // Custom code block theme
    style.CodeBlock.Theme = "monokai"

    return glamour.NewTermRenderer(
        glamour.WithStyles(style),
        glamour.WithWordWrap(100),
    )
}
```

### Custom style (JSON file)

```json
{
    "document": {
        "color": "252",
        "margin": 2,
        "block_prefix": "\n",
        "block_suffix": "\n"
    },
    "heading": {
        "color": "39",
        "bold": true,
        "block_suffix": "\n"
    },
    "h1": {
        "color": "228",
        "background_color": "63",
        "bold": true,
        "prefix": " ",
        "suffix": " "
    },
    "h2": {
        "prefix": "## "
    },
    "code_block": {
        "theme": "dracula",
        "margin": 2
    },
    "link": {
        "color": "123",
        "underline": true
    },
    "strong": {
        "bold": true
    },
    "emph": {
        "italic": true
    }
}
```

```go
r, err := glamour.NewTermRenderer(
    glamour.WithStylesFromJSONFile("./my-style.json"),
    glamour.WithWordWrap(80),
)
```

### StyleConfig structure reference

```
StyleConfig
  Document, BlockQuote, Paragraph     -> StyleBlock
  List                                -> StyleList (StyleBlock + LevelIndent)
  Heading, H1-H6                     -> StyleBlock
  Text, Emph, Strong, Strikethrough  -> StylePrimitive
  HorizontalRule                     -> StylePrimitive (use Format for custom rule)
  Item, Enumeration                  -> StylePrimitive (BlockPrefix for bullet char)
  Task                               -> StyleTask (Ticked/Unticked strings)
  Link, LinkText                     -> StylePrimitive
  Image, ImageText                   -> StylePrimitive
  Code                               -> StyleBlock (inline code)
  CodeBlock                          -> StyleCodeBlock (Theme + Chroma)
  Table                              -> StyleTable (separators)
  DefinitionList/Term/Description    -> StyleBlock/StylePrimitive
  HTMLBlock, HTMLSpan                 -> StyleBlock

StyleBlock
  Indent *uint, IndentToken *string, Margin *uint
  + StylePrimitive (all fields below)

StylePrimitive
  Color, BackgroundColor  *string    // ANSI color number or hex "#RRGGBB"
  Bold, Italic, Underline *bool
  CrossedOut, Faint       *bool
  Inverse, Conceal, Blink *bool
  Upper, Lower, Title     *bool      // text transform
  Prefix, Suffix          string     // per-line prefix/suffix
  BlockPrefix, BlockSuffix string    // before/after entire block
  Format                  string     // Go template, e.g. link format
```

### Color downsampling with lipgloss (v2)

Glamour v2 is "pure" - same input always gives same output. It does NOT auto-detect terminal color capabilities. Use lipgloss to downsample colors for the actual terminal.

```go
import (
    "charm.land/glamour/v2"
    "charm.land/lipgloss/v2"
)

r, _ := glamour.NewTermRenderer(glamour.WithWordWrap(80))
out, _ := r.Render(markdown)

// lipgloss detects terminal capabilities and downsamples
lipgloss.Print(out)
```

Alternative with `colorprofile` for explicit control:

```go
import "github.com/charmbracelet/colorprofile"

w := colorprofile.NewWriter(os.Stdout, os.Environ())
fmt.Fprintf(w, "%s", out)
```

### Detect terminal background for style selection

```go
import "charm.land/lipgloss/v2"

style := "dark"
if !lipgloss.HasDarkBackground() {
    style = "light"
}
r, _ := glamour.NewTermRenderer(glamour.WithStandardStyle(style))
```

### Environment-based style

```bash
export GLAMOUR_STYLE=dracula
# or a file path:
export GLAMOUR_STYLE=/path/to/custom.json
```

```go
// Picks up GLAMOUR_STYLE, falls back to "dark"
out, err := glamour.RenderWithEnvironmentConfig(markdown)

// Or with a renderer:
r, err := glamour.NewTermRenderer(glamour.WithEnvironmentConfig())
```

## Integration

### Bubbletea viewport (scrollable markdown)

```go
import (
    "charm.land/glamour/v2"
    tea "charm.land/bubbletea/v2"
    "charm.land/bubbles/v2/viewport"
)

type model struct {
    viewport viewport.Model
    content  string
}

func initialModel(markdown string) model {
    r, _ := glamour.NewTermRenderer(
        glamour.WithStandardStyle("dark"),
        glamour.WithWordWrap(78), // viewport width minus padding
    )
    rendered, _ := r.Render(markdown)

    vp := viewport.New(viewport.WithWidth(80), viewport.WithHeight(24))
    vp.SetContent(rendered)

    return model{viewport: vp, content: rendered}
}

func (m model) Init() tea.Cmd {
    return nil
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmd tea.Cmd
    m.viewport, cmd = m.viewport.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return m.viewport.View()
}
```

Key point: set `WithWordWrap` to viewport width minus any horizontal padding/margin. Re-render when terminal resizes.

### Re-render on resize

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.viewport.SetWidth(msg.Width)
        m.viewport.SetHeight(msg.Height)

        r, _ := glamour.NewTermRenderer(
            glamour.WithStandardStyle("dark"),
            glamour.WithWordWrap(msg.Width - 2),
        )
        rendered, _ := r.Render(m.rawMarkdown)
        m.viewport.SetContent(rendered)
    }

    var cmd tea.Cmd
    m.viewport, cmd = m.viewport.Update(msg)
    return m, cmd
}
```

### Lipgloss styled container around rendered markdown

```go
import "charm.land/lipgloss/v2"

border := lipgloss.NewStyle().
    Border(lipgloss.RoundedBorder()).
    Padding(1, 2)

r, _ := glamour.NewTermRenderer(
    glamour.WithStandardStyle("dark"),
    glamour.WithWordWrap(76), // account for border + padding (2 border + 4 padding = 6)
)
rendered, _ := r.Render(markdown)

fmt.Println(border.Render(rendered))
```

## Common Mistakes

**Using `WithAutoStyle()` or `WithColorProfile()`** - Removed in v2. Use `WithStandardStyle("dark")` and `lipgloss.Print()` for color handling.

**Not accounting for margin/padding in word wrap** - If you wrap a glamour-rendered block in a lipgloss container with padding/border, subtract that width from the word wrap value or text will overflow.

**Creating a new renderer per render when unnecessary** - `TermRenderer` is reusable. Create once, call `Render()` many times.

**Using `fmt.Print` instead of `lipgloss.Print`** - If colors look wrong on some terminals, you need color downsampling. Use `lipgloss.Print(out)` instead of `fmt.Print(out)`.

**Forgetting `Close()` when using Write/Read pattern** - After writing markdown via `r.Write()`, you must call `r.Close()` before reading with `r.Read()` or `io.ReadAll(r)`.

**Import path still on v1** - v2 uses `charm.land/glamour/v2`, not `github.com/charmbracelet/glamour`.

**Setting style fields directly instead of via pointer** - `Color`, `Bold`, `Italic`, etc. are pointer types. Use helper functions like `func boolPtr(b bool) *bool { return &b }`.

**Using `WithStandardStyle` with a file path** - `WithStandardStyle` only accepts built-in style names. For file paths, use `WithStylePath` or `WithStylesFromJSONFile`.

## Checklist

- [ ] Import `charm.land/glamour/v2` (not the old github path)
- [ ] Pick a style: built-in name, JSON file, or programmatic `StyleConfig`
- [ ] Set `WithWordWrap` to match your output width minus borders/padding
- [ ] Use `lipgloss.Print()` for proper color downsampling on real terminals
- [ ] For bubbletea: re-render on `WindowSizeMsg` with updated wrap width
- [ ] Handle errors from `NewTermRenderer` and `Render` (malformed styles, etc.)
- [ ] For env-based config: use `WithEnvironmentConfig()` or `RenderWithEnvironmentConfig()`
- [ ] Test with `"notty"` style for CI/non-terminal environments
