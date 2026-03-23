---
name: charm-lipgloss
description: "CSS-like terminal styling for Go with lipgloss v2 - styles, colors, borders, layout, tables, lists, and trees. Use when styling Go terminal output, lipgloss, terminal layout composition, or building styled tables/lists/trees in Go."
---

# Lip Gloss v2

CSS-like terminal styling for Go. Import: `charm.land/lipgloss/v2`

v1 (`github.com/charmbracelet/lipgloss`) is deprecated. See `references/v1-to-v2-migration.md`.

## Quick Start

```go
package main

import "charm.land/lipgloss/v2"

func main() {
    style := lipgloss.NewStyle().
        Bold(true).
        Foreground(lipgloss.Color("#FAFAFA")).
        Background(lipgloss.Color("#7D56F4")).
        Padding(1, 2).
        Width(30)

    lipgloss.Println(style.Render("Hello, terminal!"))
}
```

Rules:
- Use `lipgloss.Println` not `fmt.Println` for automatic color downsampling
- `Style` is a value type. Assignment copies. No renderer, no pointers.
- `lipgloss.Color()` is a function returning `color.Color`, not a type

## Core API Reference

### Style

Create: `lipgloss.NewStyle()`. All methods return new `Style` (immutable).

```go
s := lipgloss.NewStyle().
    Bold(true).Italic(true).Faint(true).Strikethrough(true).Reverse(true).Blink(true).
    Underline(true).  // or UnderlineStyle(lipgloss.UnderlineCurly)
    UnderlineColor(lipgloss.Color("#FF0000")).
    Foreground(lipgloss.Color("#FF0000")).
    Background(lipgloss.Color("63")).
    Width(40).Height(10).MaxWidth(80).MaxHeight(20).
    Align(lipgloss.Center).                // horizontal; or Align(hPos, vPos)
    Padding(1, 2).PaddingChar('.').        // CSS shorthand: 1-4 args
    Margin(1, 2).MarginBackground(c).     // same shorthand as padding
    Border(lipgloss.RoundedBorder()).
    BorderForeground(lipgloss.Color("63")).
    Inline(true).                          // single line, no block formatting
    Transform(strings.ToUpper).
    Hyperlink("https://example.com")

output := s.Render("text", "more")  // renders with style applied
```

Position constants: `Left`/`Top` (0.0), `Center` (0.5), `Right`/`Bottom` (1.0).

Padding/Margin shorthand follows CSS: 1 arg = all, 2 = vert/horiz, 3 = top/horiz/bottom, 4 = clockwise.

Underline styles: `UnderlineNone`, `UnderlineSingle`, `UnderlineDouble`, `UnderlineCurly`, `UnderlineDotted`, `UnderlineDashed`.

Inheritance: `child.Inherit(parent)` copies unset rules. Margins/padding are NOT inherited.

Copying: `copy := style` (value type). `.Copy()` is deprecated.

Getters/Unsetters: every property has `Get*()` and `Unset*()` variants. Key sizing getters: `GetHorizontalFrameSize()`, `GetVerticalFrameSize()` (border + margin + padding). See `references/api-details.md`.

### Color

All implement `image/color.Color`.

```go
lipgloss.Color("#FF0000")     // hex TrueColor
lipgloss.Color("#F00")        // short hex
lipgloss.Color("21")          // ANSI256
lipgloss.Color("5")           // ANSI 16
lipgloss.Magenta              // named constant (ansi.BasicColor)
lipgloss.NoColor{}            // absence of color
lipgloss.RGBColor{R: 255}     // direct RGB
lipgloss.ANSIColor(134)       // ANSI256 by number
```

Named ANSI 16: `Black`, `Red`, `Green`, `Yellow`, `Blue`, `Magenta`, `Cyan`, `White`, `BrightBlack` through `BrightWhite`.

Utilities: `Darken(c, 0.5)`, `Lighten(c, 0.35)`, `Complementary(c)`, `Alpha(c, 0.5)`, `Blend1D(steps, colors...)`, `Blend2D(w, h, angle, colors...)`.

**Adaptive colors:**
```go
// Standalone
hasDark := lipgloss.HasDarkBackground(os.Stdin, os.Stdout)
ld := lipgloss.LightDark(hasDark)
fg := ld(lipgloss.Color("#333"), lipgloss.Color("#EEE"))

// Bubble Tea
case tea.BackgroundColorMsg:
    ld := lipgloss.LightDark(msg.IsDark())

// Per-profile exact colors
complete := lipgloss.Complete(colorprofile.Detect(os.Stdout, os.Environ()))
c := complete(lipgloss.Color("1"), lipgloss.Color("124"), lipgloss.Color("#ff34ac"))
```

### Border

```go
// Built-in: NormalBorder, RoundedBorder, ThickBorder, DoubleBorder,
// BlockBorder, OuterHalfBlockBorder, InnerHalfBlockBorder,
// HiddenBorder, MarkdownBorder, ASCIIBorder

s.Border(lipgloss.RoundedBorder())               // all sides
s.Border(lipgloss.NormalBorder(), true, false)     // top+bottom only
s.BorderStyle(lipgloss.RoundedBorder()).BorderTop(true).BorderLeft(false)
s.BorderForeground(lipgloss.Color("63"))           // all sides
s.BorderTopForeground(c).BorderBackground(c)       // per-side

// Gradient borders (2+ colors required)
s.BorderForegroundBlend(c1, c2, c1)  // wrap for seamless loop

// Custom: lipgloss.Border{Top, Bottom, Left, Right, TopLeft, TopRight, ...}
```

If `BorderStyle()` is set without any side booleans, all 4 sides render. Setting any side explicitly means only those render.

### Layout

```go
// Join blocks
lipgloss.JoinHorizontal(lipgloss.Top, a, b, c)    // vertical alignment
lipgloss.JoinVertical(lipgloss.Center, a, b)        // horizontal alignment

// Place in whitespace
lipgloss.Place(80, 30, lipgloss.Right, lipgloss.Bottom, content)
lipgloss.PlaceHorizontal(80, lipgloss.Center, content)
lipgloss.PlaceVertical(30, lipgloss.Bottom, content,
    lipgloss.WithWhitespaceStyle(lipgloss.NewStyle().Background(c)),
    lipgloss.WithWhitespaceChars("."),
)

// Measure
w, h := lipgloss.Size(rendered)  // also Width(), Height()

// Compositor
a := lipgloss.NewLayer(content).X(4).Y(2).Z(1)
lipgloss.Compose(a, b).Render()

// Wrap (preserves ANSI)
lipgloss.Wrap(text, 40, " ")
```

### Table

```go
import "charm.land/lipgloss/v2/table"

t := table.New().
    Headers("NAME", "AGE").
    Row("Alice", "30").
    Rows([][]string{{"Bob", "25"}}...).
    Border(lipgloss.RoundedBorder()).
    BorderStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("99"))).
    BorderHeader(true).BorderColumn(true).BorderRow(false).
    Width(60).Height(20).Wrap(true).
    StyleFunc(func(row, col int) lipgloss.Style {
        if row == table.HeaderRow { // HeaderRow == -1
            return lipgloss.NewStyle().Bold(true).Align(lipgloss.Center)
        }
        return lipgloss.NewStyle().Padding(0, 1)
    })
lipgloss.Println(t)

// Custom data: implement table.Data{At(row,cell)string, Rows()int, Columns()int}
// Filtering: table.NewFilter(data).Filter(func(row int) bool { ... })
// Markdown: table.New().Border(lipgloss.MarkdownBorder()).BorderTop(false).BorderBottom(false)
```

### List

```go
import "charm.land/lipgloss/v2/list"

l := list.New("A", "B", "C")
l := list.New("Fruits", list.New("Apple", "Banana"), "Veggies", list.New("Carrot"))

// Enumerators: Bullet (default), Arabic, Alphabet, Roman, Dash, Asterisk
l.Enumerator(list.Arabic)
l.EnumeratorStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("99")).MarginRight(1))
l.ItemStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("212")))
l.ItemStyleFunc(func(items list.Items, i int) lipgloss.Style { ... })

// Custom: func(items list.Items, i int) string
l.Item("incremental").Offset(1, -1)  // pagination
```

### Tree

```go
import "charm.land/lipgloss/v2/tree"

t := tree.Root("Project").
    Child("src", tree.Root("cmd").Child("main.go")).
    Child("README.md")

// Enumerators: DefaultEnumerator (square), RoundedEnumerator (rounded)
t.Enumerator(tree.RoundedEnumerator)
t.RootStyle(s).ItemStyle(s).EnumeratorStyle(s)
t.ItemStyleFunc(func(children tree.Children, i int) lipgloss.Style { ... })
t.Width(40).Hide(true)
```

## Common Patterns

### 1. Styled card

```go
func Card(title, body string, w int) string {
    head := lipgloss.NewStyle().Bold(true).
        Foreground(lipgloss.Color("#FAFAFA")).Background(lipgloss.Color("#7D56F4")).
        Padding(0, 1).Width(w)
    frame := lipgloss.NewStyle().Padding(1, 2).Width(w).
        Border(lipgloss.RoundedBorder()).BorderForeground(lipgloss.Color("#7D56F4"))
    return lipgloss.JoinVertical(lipgloss.Left, head.Render(title), frame.Render(body))
}
```

### 2. Two-column layout

```go
func Columns(left, right string, total int) string {
    col := lipgloss.NewStyle().Width(total / 2).Padding(1, 2)
    return lipgloss.JoinHorizontal(lipgloss.Top, col.Render(left), col.Render(right))
}
```

### 3. Adaptive theme

```go
func NewTheme(hasDark bool) Theme {
    ld := lipgloss.LightDark(hasDark)
    return Theme{
        Primary: ld(lipgloss.Color("#5A56E0"), lipgloss.Color("#7571F9")),
        Subtle:  ld(lipgloss.Color("#999"), lipgloss.Color("#666")),
    }
}
```

### 4. Alternating-row table

```go
t := table.New().Headers(headers...).Rows(rows...).
    Border(lipgloss.RoundedBorder()).
    BorderStyle(lipgloss.NewStyle().Foreground(lipgloss.Color("99"))).
    StyleFunc(func(row, col int) lipgloss.Style {
        base := lipgloss.NewStyle().Padding(0, 1)
        if row == table.HeaderRow { return base.Bold(true) }
        if row%2 == 0 { return base.Foreground(lipgloss.Color("245")) }
        return base.Foreground(lipgloss.Color("241"))
    })
```

## Integration Patterns

### Bubble Tea

```go
func (m model) Init() tea.Cmd { return tea.RequestBackgroundColor }
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.BackgroundColorMsg:
        m.styles = newStyles(msg.IsDark())
    }
    return m, nil
}
func (m model) View() string {
    return m.styles.title.Render("My App")
    // Bubble Tea handles downsampling - no lipgloss.Println needed
}
```

### Glamour (Markdown)

```go
md, _ := glamour.Render(content, "dark")
frame := lipgloss.NewStyle().Border(lipgloss.RoundedBorder()).Padding(1, 2).Width(80)
lipgloss.Println(frame.Render(md))
```

## Common Mistakes

1. **`fmt.Println` instead of `lipgloss.Println`** - no color downsampling. Always use lipgloss writers for standalone apps.
2. **Width includes padding and borders** - `Width(40)` is 40 total cells, not content width.
3. **`Color()` is a function in v2** - returns `color.Color`, not a type literal.
4. **`Inherit()` skips margins/padding** - only text formatting and colors are inherited.
5. **v1/v2 import mixing** - v2 is `charm.land/lipgloss/v2`. Sub-packages: `charm.land/lipgloss/v2/{table,list,tree}`.
6. **`table.HeaderRow` is -1** - not 0. Data rows start at 0.
7. **Border side defaults** - `BorderStyle()` alone renders all 4 sides. Setting any `BorderTop/Right/Bottom/Left` explicitly means only those render.
8. **Renderer usage** - `*Renderer` does not exist in v2. Remove all renderer references.

## Checklist

- [ ] Import `charm.land/lipgloss/v2` (not `github.com/charmbracelet/lipgloss`)
- [ ] Colors: `lipgloss.Color("...")` function call, returns `color.Color`
- [ ] Output: `lipgloss.Println` for standalone apps
- [ ] Width accounts for padding + border sizes
- [ ] Table `StyleFunc` handles `table.HeaderRow` (-1)
- [ ] Adaptive colors: `lipgloss.LightDark()`, not removed `lipgloss.AdaptiveColor`
- [ ] No `*Renderer`, no `.Copy()` - both removed in v2
- [ ] Sub-package imports: `charm.land/lipgloss/v2/{table,list,tree}`
