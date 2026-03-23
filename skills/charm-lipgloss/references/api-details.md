# Lip Gloss v2 API Details

## Style Getters

```go
s.GetBold()               // bool
s.GetItalic()             // bool
s.GetUnderline()          // bool
s.GetUnderlineStyle()     // Underline
s.GetUnderlineColor()     // color.Color
s.GetStrikethrough()      // bool
s.GetReverse()            // bool
s.GetBlink()              // bool
s.GetFaint()              // bool
s.GetForeground()         // color.Color
s.GetBackground()         // color.Color
s.GetWidth()              // int
s.GetHeight()             // int
s.GetAlign()              // Position (horizontal)
s.GetAlignHorizontal()    // Position
s.GetAlignVertical()      // Position
s.GetPadding()            // top, right, bottom, left int
s.GetPaddingTop()         // int
s.GetPaddingRight()       // int
s.GetPaddingBottom()      // int
s.GetPaddingLeft()        // int
s.GetHorizontalPadding()  // int (left + right)
s.GetVerticalPadding()    // int (top + bottom)
s.GetMargin()             // top, right, bottom, left int
s.GetMarginTop()          // int
s.GetMarginRight()        // int
s.GetMarginBottom()       // int
s.GetMarginLeft()         // int
s.GetHorizontalMargins()  // int (left + right)
s.GetVerticalMargins()    // int (top + bottom)
s.GetBorderStyle()        // Border
s.GetBorderTop()          // bool
s.GetBorderRight()        // bool
s.GetBorderBottom()       // bool
s.GetBorderLeft()         // bool
s.GetHorizontalBorderSize() // int
s.GetVerticalBorderSize()   // int
s.GetHorizontalFrameSize()  // int (border + margin + padding)
s.GetVerticalFrameSize()    // int (border + margin + padding)
s.GetMaxWidth()           // int
s.GetMaxHeight()          // int
s.GetTabWidth()           // int
s.GetHyperlink()          // link, params string
```

## All Unset Methods

Every setter has a corresponding `Unset*` method:
`UnsetBold`, `UnsetItalic`, `UnsetUnderline`, `UnsetUnderlineStyle`, `UnsetUnderlineColor`,
`UnsetStrikethrough`, `UnsetReverse`, `UnsetBlink`, `UnsetFaint`,
`UnsetForeground`, `UnsetBackground`, `UnsetWidth`, `UnsetHeight`,
`UnsetAlign`, `UnsetAlignHorizontal`, `UnsetAlignVertical`,
`UnsetPadding`, `UnsetPaddingTop`, `UnsetPaddingRight`, `UnsetPaddingBottom`, `UnsetPaddingLeft`,
`UnsetMargin`, `UnsetMarginTop`, `UnsetMarginRight`, `UnsetMarginBottom`, `UnsetMarginLeft`,
`UnsetMarginBackground`, `UnsetBorderStyle`, `UnsetBorderTop`, `UnsetBorderRight`,
`UnsetBorderBottom`, `UnsetBorderLeft`, `UnsetBorderForeground`, `UnsetBorderBackground`,
`UnsetMaxWidth`, `UnsetMaxHeight`, `UnsetTabWidth`, `UnsetInline`

## Border Struct Fields

```go
type Border struct {
    Top, Bottom, Left, Right          string
    TopLeft, TopRight                 string
    BottomLeft, BottomRight           string
    MiddleLeft, MiddleRight           string  // used by tables for row separators
    Middle                            string  // used by tables for intersections
    MiddleTop, MiddleBottom           string  // used by tables for column header joins
}
```

## Table Data Interface

```go
type Data interface {
    At(row, cell int) string
    Rows() int
    Columns() int
}
```

Built-in implementations:
- `table.NewStringData(rows ...[]string)` - basic string data
- `table.NewFilter(data).Filter(func(row int) bool)` - filtered view

## Tree Node Interface

```go
type Node interface {
    fmt.Stringer
    Value() string
    Children() Children
    Hidden() bool
    SetHidden(bool)
    SetValue(any)
}
```

Types: `*tree.Tree` (has children), `*tree.Leaf` (no children)

## Color Types

```go
// Function - parse string to color
func Color(s string) color.Color

// Struct types
type NoColor struct{}                      // absence of color
type RGBColor struct{ R, G, B uint8 }      // RGB values
type ANSIColor = ansi.IndexedColor         // ANSI 256 by number

// Named ANSI 16 constants (type ansi.BasicColor)
Black, Red, Green, Yellow, Blue, Magenta, Cyan, White
BrightBlack, BrightRed, BrightGreen, BrightYellow,
BrightBlue, BrightMagenta, BrightCyan, BrightWhite

// Utility functions
func Darken(c color.Color, percent float64) color.Color
func Lighten(c color.Color, percent float64) color.Color
func Complementary(c color.Color) color.Color
func Alpha(c color.Color, alpha float64) color.Color
func Blend1D(steps int, stops ...color.Color) []color.Color
func Blend2D(width, height int, angle float64, stops ...color.Color) []color.Color

// Adaptive color helpers
func HasDarkBackground(in *os.File, out *os.File) bool
func LightDark(isDark bool) LightDarkFunc  // returns func(light, dark color.Color) color.Color
func Complete(p colorprofile.Profile) CompleteFunc  // returns func(ansi, ansi256, truecolor color.Color) color.Color
```

## Writer Functions

All auto-downsample colors for the terminal's capability:

```go
var Writer = colorprofile.NewWriter(os.Stdout, os.Environ())

func Print(v ...any) (int, error)
func Println(v ...any) (int, error)
func Printf(format string, v ...any) (int, error)
func Fprint(w io.Writer, v ...any) (int, error)
func Fprintln(w io.Writer, v ...any) (int, error)
func Fprintf(w io.Writer, format string, v ...any) (int, error)
func Sprint(v ...any) string
func Sprintln(v ...any) string
func Sprintf(format string, v ...any) string
```

## Whitespace Options

Used with `Place`, `PlaceHorizontal`, `PlaceVertical`:

```go
lipgloss.WithWhitespaceStyle(lipgloss.NewStyle().Background(c))
lipgloss.WithWhitespaceChars(".")
```

## List Enumerators

```go
list.Bullet    // "." (default)
list.Arabic    // "1.", "2.", "3."
list.Alphabet  // "A.", "B.", "C."
list.Roman     // "I.", "II.", "III."
list.Dash      // "-"
list.Asterisk  // "*"

// Custom: func(items list.Items, index int) string
```

## Tree Enumerators

```go
tree.DefaultEnumerator  // ├── and └──
tree.RoundedEnumerator  // ├── and ╰──

// Custom: func(children tree.Children, index int) string
```
