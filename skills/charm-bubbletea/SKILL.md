---
name: charm-bubbletea
description: "Build terminal UIs in Go with Bubble Tea v2's Elm Architecture (Model/Update/View). Use when building Go TUI apps, tea.Model, tea.Cmd, Elm architecture, or terminal applications. NOT for pre-built TUI components (use bubbles)."
argument-hint: "[component or pattern name]"
---

# Bubble Tea v2

Build terminal UIs in Go using the Elm Architecture. Import as `tea "charm.land/bubbletea/v2"`.

**$ARGUMENTS context**: If a specific component or pattern is requested, focus guidance on that area.

## Quick Start

Minimal working program - a counter:

```go
package main

import (
    "fmt"
    "os"

    tea "charm.land/bubbletea/v2"
)

type model struct{ count int }

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        switch msg.String() {
        case "q", "ctrl+c":
            return m, tea.Quit
        case "up", "k":
            m.count++
        case "down", "j":
            m.count--
        }
    }
    return m, nil
}

func (m model) View() tea.View {
    return tea.NewView(fmt.Sprintf("Count: %d\n\nup/down to change, q to quit\n", m.count))
}

func main() {
    if _, err := tea.NewProgram(model{}).Run(); err != nil {
        fmt.Println("Error:", err)
        os.Exit(1)
    }
}
```

## Core API Reference

### The Elm Architecture

Bubble Tea follows the Elm Architecture - a unidirectional data flow pattern:

1. **Model** holds all application state
2. **Update** receives messages, returns updated model + optional command
3. **View** renders the model to a string (called after every Update)
4. **Commands** perform async I/O, returning messages back to Update

The runtime owns the loop. You never mutate state outside Update. You never call View yourself.

### tea.Model Interface

```go
type Model interface {
    Init() Cmd                      // initial command (return nil for none)
    Update(Msg) (Model, Cmd)        // handle messages, return new state + command
    View() View                     // render UI
}
```

### tea.Cmd and tea.Msg

```go
type Msg = any                      // messages can be any type
type Cmd func() Msg                 // commands are functions that return a message
```

**A Cmd is a function that performs I/O and returns a Msg.** Return `nil` for no command.

### tea.View

```go
// Create a view from a string
v := tea.NewView("Hello, World!")

// View struct fields (set after creation):
v.AltScreen = true                          // fullscreen mode
v.MouseMode = tea.MouseModeCellMotion       // enable mouse
v.ReportFocus = true                        // get FocusMsg/BlurMsg
v.Cursor = tea.NewCursor(x, y)             // show cursor at position
v.WindowTitle = "My App"                    // set terminal title
v.BackgroundColor = someColor               // set terminal bg
v.KeyboardEnhancements.ReportEventTypes = true // key release events
```

### Program

```go
p := tea.NewProgram(model{}, opts...)       // create program
m, err := p.Run()                           // run (blocks until quit)
p.Send(msg)                                 // send message from outside
p.Quit()                                    // quit from outside
p.Kill()                                    // force kill
p.Wait()                                    // block until shutdown
```

### Program Options

```go
tea.WithContext(ctx)                         // cancellable context
tea.WithInput(reader)                        // custom input (nil to disable)
tea.WithOutput(writer)                       // custom output
tea.WithFilter(func(Model, Msg) Msg)         // intercept/filter messages
tea.WithFPS(fps)                             // custom FPS (default 60, max 120)
tea.WithEnvironment([]string)                // custom env vars (SSH)
tea.WithWindowSize(w, h)                     // initial size (testing)
tea.WithColorProfile(profile)                // force color profile
tea.WithoutRenderer()                        // no TUI rendering
tea.WithoutSignalHandler()                   // handle signals yourself
tea.WithoutCatchPanics()                     // disable panic recovery
```

### Built-in Messages

| Message | When |
|---------|------|
| `tea.KeyPressMsg` | Key pressed. Use `msg.String()` to match (e.g. `"ctrl+c"`, `"enter"`, `"a"`) |
| `tea.KeyReleaseMsg` | Key released (needs keyboard enhancements) |
| `tea.WindowSizeMsg` | Terminal resized. Fields: `Width`, `Height`. Sent on startup + resize |
| `tea.MouseClickMsg` | Mouse click. Fields: `X`, `Y`, `Button`, `Mod` |
| `tea.MouseReleaseMsg` | Mouse button released |
| `tea.MouseWheelMsg` | Scroll wheel |
| `tea.MouseMotionMsg` | Mouse moved (needs AllMotion mode) |
| `tea.FocusMsg` | Terminal gained focus (needs `ReportFocus`) |
| `tea.BlurMsg` | Terminal lost focus |
| `tea.PasteMsg` | Bracketed paste. Field: `Content` |
| `tea.ColorProfileMsg` | Terminal color profile on startup |
| `tea.BackgroundColorMsg` | Response to `RequestBackgroundColor`. Has `IsDark()` |
| `tea.ResumeMsg` | Program resumed after suspend |
| `tea.KeyboardEnhancementsMsg` | Terminal keyboard capabilities |

### Built-in Commands

| Command | What it does |
|---------|-------------|
| `tea.Quit` | Exit the program |
| `tea.Suspend` | Suspend (ctrl+z behavior) |
| `tea.Interrupt` | Interrupt (returns `ErrInterrupted`) |
| `tea.ClearScreen` | Clear terminal |
| `tea.Batch(cmds...)` | Run commands concurrently |
| `tea.Sequence(cmds...)` | Run commands in order |
| `tea.Every(dur, fn)` | Tick synced with system clock |
| `tea.Tick(dur, fn)` | Tick from invocation time |
| `tea.Println(args...)` | Print above TUI (persists across renders) |
| `tea.Printf(tmpl, args...)` | Printf above TUI |
| `tea.SetClipboard(s)` | Set system clipboard (OSC52) |
| `tea.ReadClipboard` | Read system clipboard |
| `tea.RequestWindowSize` | Query current window size |
| `tea.RequestBackgroundColor` | Query terminal background color |
| `tea.ExecProcess(cmd, callback)` | Run external process (e.g. editor) |
| `tea.Raw(seq)` | Send raw escape sequence |

### Key Handling

```go
case tea.KeyPressMsg:
    switch msg.String() {
    case "ctrl+c", "q":        // string matching (most common)
        return m, tea.Quit
    case "up", "k":            // arrow keys have string names
    case "enter", "space":     // special keys
    case "ctrl+s":             // modifier combos
    case "a":                  // regular characters
    }

// Or use the Key struct for type-safe matching:
case tea.KeyPressMsg:
    key := msg.Key()
    switch key.Code {
    case tea.KeyEnter:          // typed constant
    case tea.KeyTab:
    case tea.KeyEsc:
    }
    // key.Mod for modifiers: tea.ModCtrl, tea.ModAlt, tea.ModShift
    // key.Text for printable characters
```

### Mouse Handling

Enable mouse in View, handle in Update:

```go
func (m model) View() tea.View {
    v := tea.NewView(m.render())
    v.MouseMode = tea.MouseModeCellMotion  // or tea.MouseModeAllMotion
    return v
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.MouseClickMsg:
        x, y := msg.X, msg.Y
        if msg.Button == tea.MouseLeft { /* handle click */ }
    case tea.MouseWheelMsg:
        if msg.Button == tea.MouseWheelUp { /* scroll up */ }
    case tea.MouseMsg:
        // catches all mouse events
        mouse := msg.Mouse()
    }
    return m, nil
}
```

## Common Patterns

### Pattern 1: Async I/O (HTTP, file, etc.)

Define a custom Msg type. Write a Cmd that does the I/O and returns it.

```go
type statusMsg int
type errMsg struct{ error }

func checkServer() tea.Msg {
    res, err := http.Get("https://example.com")
    if err != nil {
        return errMsg{err}
    }
    defer res.Body.Close()
    return statusMsg(res.StatusCode)
}

func (m model) Init() tea.Cmd { return checkServer } // note: pass function, don't call it

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case statusMsg:
        m.status = int(msg)
        return m, tea.Quit
    case errMsg:
        m.err = msg.error
        return m, nil
    }
    return m, nil
}
```

### Pattern 2: Ticking / Timers

`tea.Tick` and `tea.Every` send a single message. Re-dispatch to loop.

```go
type tickMsg time.Time

func doTick() tea.Cmd {
    return tea.Tick(time.Second, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}

func (m model) Init() tea.Cmd { return doTick() }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg.(type) {
    case tickMsg:
        m.elapsed++
        return m, doTick()  // re-dispatch to keep ticking
    }
    return m, nil
}
```

**`tea.Every`** syncs with the system clock (wall-clock aligned ticks). **`tea.Tick`** starts from invocation.

### Pattern 3: Composing Child Components

Child components follow the same Model/Update/View pattern. Parent delegates messages.

```go
type parentModel struct {
    spinner spinner.Model
    input   textinput.Model
    focus   int
}

func (m parentModel) Init() tea.Cmd {
    return tea.Batch(m.spinner.Tick, m.input.Focus())
}

func (m parentModel) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    // Always update spinner (it needs ticks regardless of focus)
    var cmd tea.Cmd
    m.spinner, cmd = m.spinner.Update(msg)
    cmds = append(cmds, cmd)

    // Only update focused component for key events
    switch msg.(type) {
    case tea.KeyPressMsg:
        if m.focus == 0 {
            m.input, cmd = m.input.Update(msg)
            cmds = append(cmds, cmd)
        }
    }

    return m, tea.Batch(cmds...)
}

func (m parentModel) View() tea.View {
    return tea.NewView(m.spinner.View() + "\n" + m.input.View())
}
```

### Pattern 4: External Messages via Channel

Use `p.Send()` to push messages from goroutines, or use a channel-based Cmd.

```go
// Channel-based approach (preferred for subscription-like behavior)
func waitForActivity(sub chan resultMsg) tea.Cmd {
    return func() tea.Msg {
        return <-sub  // blocks until message arrives
    }
}

func (m model) Init() tea.Cmd {
    return waitForActivity(m.sub)
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case resultMsg:
        m.results = append(m.results, msg)
        return m, waitForActivity(m.sub)  // re-subscribe
    }
    return m, nil
}

// p.Send approach (from outside the program)
go func() {
    p.Send(myMsg{data: "hello"})  // thread-safe
}()
```

### Pattern 5: Fullscreen with Alt Screen

Set `AltScreen` in your View. No Program option needed in v2.

```go
func (m model) View() tea.View {
    v := tea.NewView(fmt.Sprintf("Fullscreen app! Size: %dx%d", m.width, m.height))
    v.AltScreen = true
    return v
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width = msg.Width
        m.height = msg.Height
    }
    return m, nil
}
```

## Integration Patterns

### Lip Gloss (Styling)

```go
import "charm.land/lipgloss/v2"

var style = lipgloss.NewStyle().
    Bold(true).
    Foreground(lipgloss.Color("205")).
    Padding(0, 1)

func (m model) View() tea.View {
    return tea.NewView(style.Render("styled text"))
}
```

### Bubbles (Components)

Standard components: `spinner`, `textinput`, `textarea`, `list`, `table`, `viewport`, `paginator`, `timer`, `stopwatch`, `help`, `key`, `filepicker`, `progress`.

Import: `"charm.land/bubbles/v2/<component>"`

Each bubble follows the same Init/Update/View pattern. See Pattern 3 above.

### Huh (Forms)

```go
import "charm.land/huh/v2"

// Huh can run standalone or embed in a bubbletea program
form := huh.NewForm(
    huh.NewGroup(
        huh.NewInput().Title("Name").Value(&name),
    ),
)
```

## Common Mistakes

1. **Calling a Cmd instead of passing it.** `Init` returns `tea.Cmd`, not `tea.Msg`. Write `return checkServer` not `return checkServer()`. The runtime calls the function.

2. **Forgetting to return a Cmd from tick handlers.** `tea.Tick` and `tea.Every` fire once. You must return a new tick command in your Update handler to keep ticking.

3. **Mutating the model outside Update.** The Elm Architecture requires all state changes go through Update. Don't share model pointers with goroutines. Use `p.Send()` or channel-based Cmds to communicate.

4. **Not handling WindowSizeMsg.** It's sent on startup and every resize. If you do any layout math, store the dimensions and use them in View.

5. **Using fmt.Println for output.** stdout is owned by the TUI. Use `tea.LogToFile()` for debug logging. Use `tea.Println()` / `tea.Printf()` to print above the TUI.

6. **Returning `tea.Batch()` with nil commands.** This is safe - `tea.Batch` filters nil commands - but returning a plain `nil` is cleaner when you have no commands.

7. **AltScreen in v2 is a View property, not a Program option.** Set `v.AltScreen = true` in your View method. Same for mouse mode - set `v.MouseMode` in View.

8. **Blocking in Update.** Update must return quickly. Any I/O (HTTP calls, file reads, sleeps) belongs in a Cmd, not directly in Update.

9. **Not handling ctrl+c.** The terminal is in raw mode - ctrl+c won't kill your app automatically. Always match `"ctrl+c"` in your KeyPressMsg handler and return `tea.Quit` or `tea.Interrupt`.

## Checklist

- [ ] Model implements `Init() tea.Cmd`, `Update(tea.Msg) (tea.Model, tea.Cmd)`, `View() tea.View`
- [ ] View returns `tea.NewView(s)`, not a raw string
- [ ] ctrl+c / q handling exists in Update
- [ ] WindowSizeMsg handled if doing any layout
- [ ] All I/O in Cmds, never in Update
- [ ] Tick commands re-dispatched in Update handler
- [ ] Child component updates collected with `tea.Batch(cmds...)`
- [ ] No `fmt.Println` - use `tea.LogToFile` for debugging
- [ ] AltScreen and MouseMode set in View, not as Program options

## Reference

For full message/type/constant listings, see [references/api.md](references/api.md).
