---
name: charm-bubbles
description: "Pre-built TUI components for Bubble Tea apps - spinner, text input, textarea, list, table, viewport, paginator, progress bar. Use when adding Go TUI components, bubbles, or terminal widgets to a Bubble Tea app. NOT for the core TUI framework (use bubbletea)."
---

# Bubbles - TUI Components for Bubble Tea

Bubbles (`charm.land/bubbles/v2`) is a component library for [Bubble Tea](https://github.com/charmbracelet/bubbletea) applications. Each bubble is a self-contained Model with `Update` and `View` methods you embed in your own model.

## Quick Start

Embed a bubble (e.g. spinner) inside your Bubble Tea app:

```go
package main

import (
    "fmt"
    tea "charm.land/bubbletea/v2"
    "charm.land/bubbles/v2/spinner"
)

type model struct {
    spinner spinner.Model
}

func initialModel() model {
    return model{spinner: spinner.New(spinner.WithSpinner(spinner.Dot))}
}

func (m model) Init() tea.Cmd {
    return m.spinner.Tick // start the spinner
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        if msg.String() == "q" {
            return m, tea.Quit
        }
    }
    var cmd tea.Cmd
    m.spinner, cmd = m.spinner.Update(msg)
    return m, cmd
}

func (m model) View() string {
    return fmt.Sprintf("\n  %s Loading...\n", m.spinner.View())
}

func main() {
    tea.NewProgram(initialModel()).Run()
}
```

## How Bubbles Work

### The Update/View Contract

Every bubble follows the same pattern:

1. **Model** - a struct holding component state
2. **New()** - constructor returning a configured Model (usually with functional options)
3. **Update(msg tea.Msg) (Model, tea.Cmd)** - processes messages, returns updated model + commands
4. **View() string** - renders current state to a string

Bubbles return their own Model type from Update (not `tea.Model`), so you assign back to the embedded field:

```go
// Correct: assign back to the field
m.textinput, cmd = m.textinput.Update(msg)

// Wrong: this loses the update
m.textinput.Update(msg)
```

### Message Flow

Bubbles communicate via typed messages. When you embed a bubble, you pass all messages through to its Update:

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd

    // Handle your own messages first
    switch msg := msg.(type) {
    case tea.KeyPressMsg:
        // your key handling
    }

    // Forward to embedded bubbles
    var cmd tea.Cmd
    m.spinner, cmd = m.spinner.Update(msg)
    cmds = append(cmds, cmd)

    m.textinput, cmd = m.textinput.Update(msg)
    cmds = append(cmds, cmd)

    return m, tea.Batch(cmds...)
}
```

### Commands and Init

Some bubbles require commands to start (spinner needs `Tick`, timer needs `Init`). Return these from your top-level `Init()`:

```go
func (m model) Init() tea.Cmd {
    return tea.Batch(
        m.spinner.Tick,
        m.timer.Init(),
    )
}
```

## Common Patterns

### Focus Management

Interactive bubbles (textinput, textarea, table, filepicker) have Focus/Blur methods. Only focused components process keyboard input.

```go
// Focus returns a tea.Cmd for textinput (starts cursor blinking)
cmd := m.textinput.Focus()

// Table focus is simpler, no cmd needed
m.table.Focus()

// Blur removes focus
m.textinput.Blur()
```

When managing multiple inputs, blur all then focus the active one:

```go
for i := range m.inputs {
    m.inputs[i].Blur()
}
m.inputs[m.focusIndex].Focus()
```

### Functional Options

Most constructors accept variadic options:

```go
s := spinner.New(spinner.WithSpinner(spinner.Dot), spinner.WithStyle(myStyle))
t := table.New(table.WithColumns(cols), table.WithRows(rows), table.WithHeight(10))
v := viewport.New(viewport.WithWidth(80), viewport.WithHeight(24))
p := progress.New(progress.WithDefaultBlend(), progress.WithoutPercentage())
tmr := timer.New(30*time.Second, timer.WithInterval(100*time.Millisecond))
```

### Styling with Lipgloss

All visual bubbles accept lipgloss styles. Common pattern:

```go
// Spinner: direct Style field
s := spinner.New()
s.Style = lipgloss.NewStyle().Foreground(lipgloss.Color("205"))

// Table: Styles struct
t := table.New(table.WithStyles(table.Styles{
    Header:   lipgloss.NewStyle().Bold(true).Padding(0, 1),
    Cell:     lipgloss.NewStyle().Padding(0, 1),
    Selected: lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("212")),
}))

// TextInput/TextArea: SetStyles method with Focused/Blurred states
ti := textinput.New()
ti.SetStyles(textinput.DefaultDarkStyles())

// Viewport: Style field for borders/padding
vp := viewport.New(viewport.WithWidth(80), viewport.WithHeight(24))
vp.Style = lipgloss.NewStyle().Border(lipgloss.RoundedBorder())
```

### Key Bindings

Use the `key` package for remappable bindings that integrate with the `help` bubble:

```go
type KeyMap struct {
    Quit key.Binding
    Help key.Binding
}

var keys = KeyMap{
    Quit: key.NewBinding(
        key.WithKeys("q", "ctrl+c"),
        key.WithHelp("q", "quit"),
    ),
    Help: key.NewBinding(
        key.WithKeys("?"),
        key.WithHelp("?", "help"),
    ),
}

// In Update:
case tea.KeyPressMsg:
    switch {
    case key.Matches(msg, keys.Quit):
        return m, tea.Quit
    }

// For help integration, implement help.KeyMap interface:
func (k KeyMap) ShortHelp() []key.Binding { return []key.Binding{k.Quit, k.Help} }
func (k KeyMap) FullHelp() [][]key.Binding { return [][]key.Binding{{k.Quit, k.Help}} }
```

### Composing Multiple Bubbles

The list component is a good example of composition - it internally uses spinner, textinput, paginator, and help:

```go
type model struct {
    list     list.Model
    viewport viewport.Model
    help     help.Model
    spinner  spinner.Model
}
```

Combine their views with lipgloss layout:

```go
func (m model) View() string {
    left := m.list.View()
    right := m.viewport.View()
    return lipgloss.JoinHorizontal(lipgloss.Top, left, right)
}
```

### Window Size Handling

Resize bubbles when the terminal size changes:

```go
case tea.WindowSizeMsg:
    m.viewport.SetWidth(msg.Width)
    m.viewport.SetHeight(msg.Height - headerHeight)
    m.list.SetSize(msg.Width, msg.Height)
    m.table.SetWidth(msg.Width)
    m.table.SetHeight(msg.Height)
    m.progress.SetWidth(msg.Width - padding)
    m.help.SetWidth(msg.Width)
```

### ID-Based Message Routing

Animated bubbles (spinner, progress, timer, stopwatch) use internal IDs so multiple instances don't interfere. Each instance only processes messages with its own ID. Just forward all messages to all instances:

```go
m.spinner1, cmd1 = m.spinner1.Update(msg)
m.spinner2, cmd2 = m.spinner2.Update(msg)
```

## Integration with Bubbletea and Lipgloss

### Import Paths (v2)

```go
import (
    tea "charm.land/bubbletea/v2"
    "charm.land/bubbles/v2/spinner"
    "charm.land/bubbles/v2/textinput"
    "charm.land/lipgloss/v2"
)
```

### Key Message Types

Bubbles v2 uses `tea.KeyPressMsg` (not `tea.KeyMsg` from v1). Match with `key.Matches`:

```go
case tea.KeyPressMsg:
    switch {
    case key.Matches(msg, m.KeyMap.Up):
        // handle up
    }
```

### Progress Bar - Static vs Animated

Progress supports two modes:

```go
// Animated: use SetPercent (returns cmd), Update processes FrameMsg
cmd := m.progress.SetPercent(0.75)

// Static: use ViewAs directly, no Update needed
view := m.progress.ViewAs(0.75)
```

### List Item Interface

The list component requires items to implement the `Item` interface:

```go
type Item interface {
    FilterValue() string
}
```

And a delegate implementing `ItemDelegate`:

```go
type ItemDelegate interface {
    Render(w io.Writer, m Model, index int, item Item)
    Height() int
    Spacing() int
    Update(msg tea.Msg, m *Model) tea.Cmd
}
```

### Filepicker Selection

Check for file selection in your Update:

```go
case tea.KeyPressMsg:
    m.filepicker, cmd = m.filepicker.Update(msg)
    if didSelect, path := m.filepicker.DidSelectFile(msg); didSelect {
        m.selectedFile = path
    }
```

## Common Mistakes

1. **Not returning commands from Update.** Bubbles like spinner and timer stop working if you drop their commands. Always capture and return: `m.spinner, cmd = m.spinner.Update(msg)`

2. **Forgetting to call Init/Tick.** Spinner needs `m.spinner.Tick` returned from Init. Timer needs `m.timer.Init()`. Without these, the component never starts animating.

3. **Not assigning Update result back.** Bubbles return value types (not pointers). `m.spinner.Update(msg)` without assignment discards the update.

4. **Forwarding messages only to focused bubble.** Most bubbles self-filter (spinners ignore wrong IDs, unfocused inputs ignore keys). Forward all messages to all bubbles and let them decide.

5. **Using v1 message types.** In v2, it's `tea.KeyPressMsg` not `tea.KeyMsg`. Check the upgrade guide if migrating.

6. **Not handling WindowSizeMsg.** Components with fixed dimensions (viewport, list, table, progress) need resizing or they clip/overflow.

7. **Setting width/height to 0.** Viewport and table render empty strings when dimensions are 0. Always set dimensions before first render.

8. **Calling Focus() without using the cmd.** `textinput.Focus()` returns a `tea.Cmd` for cursor blinking. If you drop it, the cursor won't blink.

## Checklist

- [ ] Embed bubble Model as a field in your model (not a pointer)
- [ ] Call constructor with `New()` or `New(opts...)`
- [ ] Return Init commands from your `Init()` (Tick for spinner, Init for timer/stopwatch)
- [ ] Forward messages to bubble's `Update` and capture both return values
- [ ] Collect commands with `tea.Batch` when using multiple bubbles
- [ ] Call `Focus()` on interactive components and use returned cmd
- [ ] Handle `tea.WindowSizeMsg` to resize dimension-aware components
- [ ] Implement `Item` and `ItemDelegate` interfaces when using list
- [ ] Use `key.Matches(msg, binding)` for key matching in v2
- [ ] Style with lipgloss via Style fields or SetStyles methods
