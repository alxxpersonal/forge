# Lip Gloss v1 to v2 Migration Quick Reference

## Import Path

```
github.com/charmbracelet/lipgloss     -> charm.land/lipgloss/v2
github.com/charmbracelet/lipgloss/table -> charm.land/lipgloss/v2/table
github.com/charmbracelet/lipgloss/tree  -> charm.land/lipgloss/v2/tree
github.com/charmbracelet/lipgloss/list  -> charm.land/lipgloss/v2/list
```

## Removed APIs

| v1 | v2 Replacement |
|---|---|
| `type Renderer` | Removed entirely |
| `DefaultRenderer()` | Not needed |
| `NewRenderer(w, opts...)` | Not needed |
| `ColorProfile()` | `colorprofile.Detect(w, env)` |
| `SetColorProfile(p)` | Set `lipgloss.Writer.Profile` |
| `HasDarkBackground()` (no args) | `lipgloss.HasDarkBackground(in, out)` |
| `type TerminalColor` | `image/color.Color` |
| `type Color string` | `func Color(string) color.Color` |
| `type AdaptiveColor` | `compat.AdaptiveColor` or `LightDark()` |
| `type CompleteColor` | `compat.CompleteColor` or `Complete()` |
| `WithWhitespaceForeground(c)` | `WithWhitespaceStyle(s)` |
| `WithWhitespaceBackground(c)` | `WithWhitespaceStyle(s)` |
| `renderer.NewStyle()` | `lipgloss.NewStyle()` |
| `style.Copy()` | plain assignment `copy := style` |

## Color System Changes

```go
// v1: Color is a string type
var c lipgloss.Color = "#ff00ff"

// v2: Color is a function returning color.Color
var c color.Color = lipgloss.Color("#ff00ff")

// v1: methods accept TerminalColor
func (s Style) Foreground(c TerminalColor) Style

// v2: methods accept color.Color (from image/color)
func (s Style) Foreground(c color.Color) Style
```

## Printing Changes

```go
// v1: fmt works, renderer handles downsampling
fmt.Println(style.Render("hello"))

// v2: use lipgloss writers for downsampling
lipgloss.Println(style.Render("hello"))
// Available: Print, Println, Printf, Fprint, Fprintln, Fprintf,
//            Sprint, Sprintln, Sprintf
```

## Adaptive Colors

```go
// v1
color := lipgloss.AdaptiveColor{Light: "#fff", Dark: "#000"}

// v2 - recommended
hasDark := lipgloss.HasDarkBackground(os.Stdin, os.Stdout)
ld := lipgloss.LightDark(hasDark)
color := ld(lipgloss.Color("#fff"), lipgloss.Color("#000"))

// v2 - compat (quick migration)
import "charm.land/lipgloss/v2/compat"
color := compat.AdaptiveColor{
    Light: lipgloss.Color("#fff"),
    Dark:  lipgloss.Color("#000"),
}
```

## New in v2

- `UnderlineStyle()` / `UnderlineColor()` - curly, dotted, dashed, double underlines
- `PaddingChar()` / `MarginChar()` - custom fill characters
- `Hyperlink()` - clickable links in supporting terminals
- `BorderForegroundBlend()` - gradient borders
- `Blend1D()` / `Blend2D()` - color gradient generation
- `NewLayer()` / `Compose()` - cell-based compositor
- `Wrap()` - ANSI-aware text wrapping
- Named ANSI color constants (`lipgloss.Red`, `lipgloss.BrightCyan`, etc.)
- `lipgloss.Alpha()`, `lipgloss.Darken()`, `lipgloss.Lighten()`, `lipgloss.Complementary()`
