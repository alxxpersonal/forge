---
name: charm-ultraviolet
description: "Low-level Go terminal primitives - cell-based rendering, input handling, screen management. Use when building custom Go terminal renderers, ultraviolet, cell buffers, or performance-critical TUI work below Bubble Tea's abstraction level."
---

# Ultraviolet (charmbracelet/ultraviolet)

## What Is Ultraviolet (and When NOT to Use It)

Ultraviolet is a set of low-level primitives for terminal manipulation in Go. It provides cell-based rendering, unified input handling, and screen management. It powers Bubble Tea v2 and Lip Gloss v2 internally.

**This is NOT a framework.** It is infrastructure. Think of it like syscalls vs stdlib - you can use it directly, but most of the time you should not.

### When NOT to use it

- Building a standard TUI app - use Bubble Tea v2 instead
- Styling terminal output - use Lip Gloss v2 instead
- You want an architecture/framework with state management - use Bubble Tea v2
- Prototyping - too low-level, too much boilerplate

### When to use it directly

- Building your own TUI framework on top of these primitives
- Writing a custom renderer that needs cell-level control
- Performance-critical rendering where you need direct buffer manipulation
- Embedding terminal rendering into a non-Bubble Tea application
- Working on Bubble Tea / Lip Gloss internals

## API Stability Warning

The project README explicitly states: "This project currently exists to serve internal use cases. API stability is a goal, but expect no stability guarantees as of now." Plan accordingly.

## Core Abstractions

The library lives in a single flat Go package `uv` (import path: `github.com/charmbracelet/ultraviolet`), with helper sub-packages `screen/` and `layout/`.

### Cell

The fundamental unit. One terminal cell = one grapheme cluster.

```go
type Cell struct {
    Content string    // single grapheme cluster
    Style   Style     // fg, bg, attrs (bold, italic, etc.)
    Link    Link      // OSC 8 hyperlink
    Width   int       // columns occupied (1 for normal, 2 for wide chars like CJK)
}
```

Key constants/values:
- `EmptyCell` - a cell with `" "`, width 1, no style
- Zero-width cells (`Width == 0`) are placeholders for wide characters

### Buffer

A 2D grid of cells, organized as `Lines []Line` where `Line []Cell`.

```go
buf := uv.NewBuffer(80, 24)        // width, height
buf.SetCell(x, y, &cell)           // write a cell
cell := buf.CellAt(x, y)          // read a cell (nil if out of bounds)
buf.Resize(newW, newH)             // resize, preserving content
buf.Clear()                        // fill with EmptyCell
buf.Fill(&cell)                    // fill with custom cell
buf.FillArea(&cell, area)          // fill rectangular region
clone := buf.Clone()               // deep copy
```

Buffer implements `Drawable`, so you can `buf.Draw(screen, area)` to composite buffers onto screens.

### RenderBuffer

Wraps Buffer with change tracking. Only touched lines/cells get re-rendered.

```go
rbuf := uv.NewRenderBuffer(80, 24)
rbuf.SetCell(x, y, &cell)         // auto-marks line as touched
rbuf.TouchLine(x, y, n)           // manually mark region dirty
rbuf.TouchedLines()               // count of dirty lines
```

### Screen (Interface)

The core abstraction that anything drawable targets.

```go
type Screen interface {
    Bounds() Rectangle
    CellAt(x, y int) *Cell
    SetCell(x, y int, c *Cell)
    WidthMethod() WidthMethod
}
```

Implemented by: `Buffer`, `ScreenBuffer`, `Window`, `TerminalScreen`.

### Drawable (Interface)

Anything that can render itself onto a Screen.

```go
type Drawable interface {
    Draw(scr Screen, area Rectangle)
}
```

Implemented by: `Buffer`, `Window`, `StyledString`, and your own components.

### Window

A rectangular area that can own its own buffer or share a parent's buffer (view).

```go
// Root window (owns its buffer)
root := uv.NewScreen(80, 24)

// Child window with own buffer
child := root.NewWindow(x, y, width, height)

// View into parent buffer (shared memory)
view := root.NewView(x, y, width, height)
```

Windows support `MoveTo`, `MoveBy`, `Resize`, `Clone`.

### Terminal

The main entry point for standalone UV apps. Manages console I/O, raw mode, event loop.

```go
t := uv.DefaultTerminal()
// or: t := uv.NewTerminal(console, opts)

t.Start()          // enter raw mode, start event loop
defer t.Stop()     // restore terminal, clean up

scr := t.Screen()  // returns *TerminalScreen

for ev := range t.Events() {
    switch ev := ev.(type) {
    case uv.WindowSizeEvent:
        scr.Resize(ev.Width, ev.Height)
    case uv.KeyPressEvent:
        if ev.MatchString("ctrl+c") { return }
    }
}
```

### TerminalScreen

The concrete screen for terminal output. Manages alt screen, cursor, colors, mouse mode, keyboard enhancements, synchronized updates.

```go
scr := t.Screen()

// Screen modes
scr.EnterAltScreen()    // alternate screen buffer
scr.ExitAltScreen()

// Rendering cycle
scr.SetCell(x, y, &cell)
scr.Render()            // diff current vs previous state
scr.Flush()             // write changes to terminal

// Or use Display for Drawable components
scr.Display(myDrawable) // clear + draw + render + flush

// Terminal features
scr.ShowCursor()
scr.SetCursorPosition(x, y)
scr.SetMouseMode(uv.MouseModeClick)
scr.SetBackgroundColor(color)
scr.SetWindowTitle("My App")
scr.SetSynchronizedUpdates(true)  // mode 2026
scr.SetKeyboardEnhancements(enh)  // kitty protocol

// Inline mode helper
scr.InsertAbove(content)  // insert text above without disrupting screen
```

### StyledString

Converts ANSI-styled strings into cell-based representation. Implements Drawable.

```go
ss := uv.NewStyledString("Hello \x1b[1mWorld\x1b[0m")
ss.Draw(screen, area)
```

## Sub-Packages

### screen/ - Screen Helpers

Utility functions that work with any `Screen` implementation.

```go
import "github.com/charmbracelet/ultraviolet/screen"

screen.Clear(scr)                    // clear entire screen
screen.ClearArea(scr, area)          // clear region
screen.Fill(scr, &cell)              // fill screen
screen.FillArea(scr, &cell, area)    // fill region
screen.Clone(scr)                    // deep copy to Buffer
screen.CloneArea(scr, area)          // deep copy region

// Drawing context with stateful style
ctx := screen.NewContext(scr)
ctx.SetForeground(ansi.Red)
ctx.SetBold(true)
ctx.DrawString("hello", x, y)
ctx.Printf("count: %d", n)          // implements io.Writer
```

### layout/ - Constraint-Based Layout

Cassowary-based layout solver (ported from Ratatui). Splits areas into non-overlapping rectangles.

```go
import "github.com/charmbracelet/ultraviolet/layout"

// Split area vertically into 3 parts
chunks := layout.New().
    Direction(layout.Vertical).
    Constraints(
        layout.Len(3),       // fixed 3 rows
        layout.Fill(1),      // fill remaining
        layout.Len(1),       // fixed 1 row
    ).
    Split(area)
```

Constraint types: `Len`, `Ratio`, `Percent`, `Fill`, `Min`, `Max`.

## Events

Events come from `t.Events()` channel. Key types:

| Event | Description |
|---|---|
| `WindowSizeEvent` | Terminal resized (width, height in cells) |
| `PixelSizeEvent` | Terminal resized (width, height in pixels) |
| `KeyPressEvent` | Key pressed. Use `ev.MatchString("ctrl+c", "q")` |
| `KeyReleaseEvent` | Key released (requires kitty keyboard protocol) |
| `MouseClickEvent` | Mouse click with position and button |
| `MouseMotionEvent` | Mouse moved (requires mouse mode enabled) |
| `PasteEvent` | Bracketed paste content |

Key matching uses human-readable strings: `"ctrl+a"`, `"shift+enter"`, `"alt+tab"`, `"f1"`, `"space"`.

## Geometry

Uses `image.Point` and `image.Rectangle` from stdlib:

```go
pos := uv.Pos(x, y)                    // == image.Point{X: x, Y: y}
rect := uv.Rect(x, y, width, height)   // origin + size (NOT min/max)
```

Note: `uv.Rect(x, y, w, h)` takes width/height, not max coordinates. This differs from `image.Rect(x0, y0, x1, y1)`.

## Style System

Styles are value types with bitfield attributes:

```go
style := uv.Style{
    Fg:             ansi.Red,
    Bg:             ansi.Black,
    UnderlineColor: ansi.Blue,
    Underline:      uv.UnderlineCurly,
    Attrs:          uv.AttrBold | uv.AttrItalic,
}
```

Attributes: `AttrBold`, `AttrFaint`, `AttrItalic`, `AttrBlink`, `AttrReverse`, `AttrConceal`, `AttrStrikethrough`.

Underline styles: `UnderlineNone`, `UnderlineSingle`, `UnderlineDouble`, `UnderlineCurly`, `UnderlineDotted`, `UnderlineDashed`.

Style diffing is built in - the renderer computes minimal ANSI sequences to transition between styles.

## Rendering Pipeline

The "Cursed Renderer" is a cell-based diffing engine inspired by ncurses:

1. You write cells to the screen buffer via `SetCell`
2. `Render()` diffs current buffer against previous state
3. Renderer emits minimal ANSI escape sequences (cursor movement, style changes, text)
4. `Flush()` writes the accumulated output to the terminal

Optimizations include:
- Only touched lines are re-rendered
- Style diffs minimize SGR sequence length
- Cursor movement uses shortest path (absolute, relative, tabs, backspace)
- Supports synchronized updates (mode 2026) to prevent flicker
- Hash-based scroll detection for efficient content shifts

## Minimal Hello World

```go
package main

import (
    "log"
    uv "github.com/charmbracelet/ultraviolet"
    "github.com/charmbracelet/ultraviolet/screen"
)

func main() {
    t := uv.DefaultTerminal()
    scr := t.Screen()
    scr.EnterAltScreen()

    if err := t.Start(); err != nil {
        log.Fatal(err)
    }
    defer t.Stop()

    ctx := screen.NewContext(scr)

    for ev := range t.Events() {
        switch ev := ev.(type) {
        case uv.WindowSizeEvent:
            scr.Resize(ev.Width, ev.Height)
        case uv.KeyPressEvent:
            if ev.MatchString("q", "ctrl+c") {
                return
            }
        }

        screen.Clear(scr)
        ctx.DrawString("Hello, World!", 0, 0)
        scr.Render()
        scr.Flush()
    }
}
```

## Relationship to Bubble Tea v2

```
ultraviolet (primitives)
    |
    +-- Lip Gloss v2 (styling, composition)
    |
    +-- Bubble Tea v2 (framework: Elm architecture, state management, commands)
            |
            +-- Bubbles (components: text input, list, table, etc.)
```

- Ultraviolet provides: cells, buffers, screen management, input decoding, rendering
- Bubble Tea v2 provides: `Program`, `Model`, `Update`, `View`, commands, subscriptions
- Lip Gloss v2 provides: `Style`, layout, borders, padding, composition

If you are building a TUI application, start with Bubble Tea v2. Only reach for ultraviolet when Bubble Tea's abstractions get in your way.

## Checklist

Before using ultraviolet directly, confirm:

- [ ] Bubble Tea v2 genuinely cannot solve your problem
- [ ] You need cell-level rendering control
- [ ] You accept API instability risk
- [ ] You understand the rendering pipeline (SetCell -> Render -> Flush)
- [ ] You handle WindowSizeEvent and call Resize yourself
- [ ] You manage terminal raw mode and cleanup (Start/Stop)
- [ ] You have read the examples in the `examples/` directory
