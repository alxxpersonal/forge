---
name: charm-huh
description: "Build interactive terminal forms and prompts in Go with huh - input, select, confirm, multiselect, validation, theming. Use when building Go terminal forms, huh, interactive Go prompts, or form fields with validation. NOT for shell script prompts (use gum)."
---

# charmbracelet/huh

Interactive terminal forms and prompts for Go. Built on Bubble Tea.

Import: `charm.land/huh/v2`

## Quick Start

```go
package main

import (
    "fmt"
    "log"

    "charm.land/huh/v2"
)

func main() {
    var name string
    var confirm bool

    err := huh.NewForm(
        huh.NewGroup(
            huh.NewInput().
                Title("What's your name?").
                Value(&name).
                Validate(huh.ValidateNotEmpty()),
            huh.NewConfirm().
                Title("Ready?").
                Value(&confirm),
        ),
    ).Run()
    if err != nil {
        log.Fatal(err)
    }
    fmt.Printf("Hello, %s!\n", name)
}
```

Single field shorthand (no Form/Group wrapper needed):

```go
var name string
huh.NewInput().Title("Name?").Value(&name).Run()
```

## Core API

### Architecture

`Form` > `Group` (pages) > `Field` (inputs)

Groups are displayed one at a time. The form advances to the next group when all fields in the current group pass validation.

### Form

```go
form := huh.NewForm(groups...*Group) *Form
```

Key methods:

| Method | Purpose |
|--------|---------|
| `.Run()` | Block and run the form |
| `.RunWithContext(ctx)` | Run with context (supports cancellation) |
| `.WithTheme(theme)` | Set theme |
| `.WithWidth(w)` / `.WithHeight(h)` | Set dimensions |
| `.WithAccessible(bool)` | Screen reader mode |
| `.WithShowHelp(bool)` | Toggle help bar |
| `.WithShowErrors(bool)` | Toggle error display |
| `.WithTimeout(duration)` | Auto-cancel after duration |
| `.WithLayout(layout)` | Set group layout |
| `.WithKeyMap(keymap)` | Custom keybindings |
| `.WithOutput(w)` / `.WithInput(r)` | Custom IO |

Retrieve values by key after completion:

```go
form.GetString("key")
form.GetInt("key")
form.GetBool("key")
form.Get("key") // any
```

Form states: `huh.StateNormal`, `huh.StateCompleted`, `huh.StateAborted`

Errors: `huh.ErrUserAborted` (ctrl+c), `huh.ErrTimeout`

### Group

```go
group := huh.NewGroup(fields ...Field) *Group
```

| Method | Purpose |
|--------|---------|
| `.Title(s)` / `.Description(s)` | Group header |
| `.WithHide(bool)` | Skip this group |
| `.WithHideFunc(func() bool)` | Conditionally skip group |
| `.WithShowHelp(bool)` | Toggle help for group |

### Field Types

Every field supports: `.Title(s)`, `.Description(s)`, `.Key(s)`, `.Value(&v)`, `.Validate(fn)`, `.Run()`.

Dynamic variants exist for most properties: `.TitleFunc(fn, binding)`, `.DescriptionFunc(fn, binding)`, etc.

#### Input

Single line text. Type: `string`.

```go
huh.NewInput().
    Title("Email").
    Placeholder("you@example.com").
    Prompt("> ").
    CharLimit(100).
    Suggestions([]string{"gmail.com", "outlook.com"}).
    EchoMode(huh.EchoModePassword). // or EchoModeNone
    Inline(true). // title and input on same line
    Validate(huh.ValidateNotEmpty()).
    Value(&email)
```

#### Text

Multi-line textarea. Type: `string`.

```go
huh.NewText().
    Title("Description").
    Lines(5).
    CharLimit(500).
    Placeholder("Enter details...").
    ShowLineNumbers(true).
    Editor("vim"). // external editor support (ctrl+e)
    EditorExtension("md").
    ExternalEditor(false). // disable external editor
    Value(&description)
```

#### Select

Pick one from a list. Generic: `Select[T comparable]`.

```go
huh.NewSelect[string]().
    Title("Country").
    Options(
        huh.NewOption("United States", "US"),
        huh.NewOption("Canada", "CA"),
    ).
    Height(8).      // scrollable if options exceed height
    Inline(true).   // horizontal left/right navigation
    Filtering(true). // start with filter active
    Value(&country)
```

Shorthand for simple options:

```go
Options(huh.NewOptions("Warrior", "Mage", "Rogue")...)
```

#### MultiSelect

Pick zero or more. Generic: `MultiSelect[T comparable]`.

```go
huh.NewMultiSelect[string]().
    Title("Toppings").
    Options(
        huh.NewOption("Lettuce", "lettuce").Selected(true),
        huh.NewOption("Tomato", "tomato"),
        huh.NewOption("Cheese", "cheese"),
    ).
    Limit(3).
    Height(6).
    Filterable(false). // disable "/" filter
    Value(&toppings)
```

Space to toggle, `a` to select all (when no limit), `/` to filter.

#### Confirm

Yes/No. Type: `bool`.

```go
huh.NewConfirm().
    Title("Continue?").
    Affirmative("Yes!").
    Negative("No way").
    Inline(true).
    Value(&ok)
```

Keys: `h`/`l` or left/right to toggle, `y` to accept, `n` to reject.

#### Note

Display-only. Not interactive by default (auto-skipped).

```go
huh.NewNote().
    Title("Welcome").
    Description("This form collects your _preferences_.").
    Height(10).
    Next(true).        // show a "Next" button, makes it interactive
    NextLabel("Continue")
```

Description supports basic markdown: `_italic_`, `*bold*`, `` `code` ``.

## Common Patterns

### Multi-Step Forms

Groups act as pages. Users navigate forward/back between them.

```go
huh.NewForm(
    huh.NewGroup(/* step 1 fields */).Title("Step 1"),
    huh.NewGroup(/* step 2 fields */).Title("Step 2"),
    huh.NewGroup(/* step 3 fields */).Title("Step 3"),
).Run()
```

### Conditional Groups

Hide groups based on previous answers:

```go
var wantExtras bool

huh.NewForm(
    huh.NewGroup(
        huh.NewConfirm().Title("Want extras?").Value(&wantExtras),
    ),
    huh.NewGroup(
        huh.NewInput().Title("Extra details").Value(&details),
    ).WithHideFunc(func() bool { return !wantExtras }),
).Run()
```

### Dynamic Fields

Use `*Func` variants to recompute properties when bindings change. Pass a pointer to the bound variable.

```go
var country string

huh.NewSelect[string]().
    Value(&state).
    TitleFunc(func() string {
        if country == "Canada" { return "Province" }
        return "State"
    }, &country).
    OptionsFunc(func() []huh.Option[string] {
        return huh.NewOptions(statesByCountry[country]...)
    }, &country)
```

The binding (`&country`) tells huh when to recompute. Results are cached per binding hash.

### Validation

Built-in validators:

```go
huh.ValidateNotEmpty()
huh.ValidateMinLength(3)
huh.ValidateMaxLength(100)
huh.ValidateLength(3, 100)  // min and max
huh.ValidateOneOf("a", "b", "c")
```

Custom validation:

```go
.Validate(func(s string) error {
    if !strings.Contains(s, "@") {
        return fmt.Errorf("must be a valid email")
    }
    return nil
})
```

Validation runs on blur (when leaving a field). Forms block progression if any field in the current group has errors.

### Theming

Built-in themes: `ThemeCharm` (default), `ThemeDracula`, `ThemeCatppuccin`, `ThemeBase16`, `ThemeDefault`.

```go
form.WithTheme(huh.ThemeFunc(huh.ThemeDracula))
```

Custom theme - implement the `Theme` interface:

```go
type Theme interface {
    Theme(isDark bool) *Styles
}
```

Or use `ThemeFunc`:

```go
form.WithTheme(huh.ThemeFunc(func(isDark bool) *huh.Styles {
    s := huh.ThemeCharm(isDark) // start from a base
    s.Focused.Title = lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("212"))
    return s
}))
```

### Layouts

```go
form.WithLayout(huh.LayoutDefault)         // one group at a time (default)
form.WithLayout(huh.LayoutStack)           // all groups stacked vertically
form.WithLayout(huh.LayoutColumns(2))      // groups in 2 columns
form.WithLayout(huh.LayoutGrid(2, 3))      // 2 rows, 3 columns
```

### Accessibility

```go
accessible := os.Getenv("ACCESSIBLE") != ""
form.WithAccessible(accessible)
```

When `TERM=dumb`, accessible mode activates automatically. Replaces TUI with plain text prompts. Timeout is not supported in accessible mode.

### Spinner

Separate package for loading indicators after form submission:

```go
import "charm.land/huh/v2/spinner"

err := spinner.New().
    Title("Processing...").
    Action(func() { /* do work */ }).
    Run()
```

Or with context:

```go
go doWork()
spinner.New().Title("Working...").Context(ctx).Run()
```

## Integration: Standalone vs Bubble Tea

### Standalone

Call `.Run()` on a form or individual field. Blocks until complete.

```go
form.Run()
// or
huh.NewInput().Title("Name?").Value(&name).Run()
```

### Embedded in Bubble Tea

`*huh.Form` implements `tea.Model`. Use it as a component in your Bubble Tea app.

```go
type Model struct {
    form *huh.Form
}

func NewModel() Model {
    return Model{
        form: huh.NewForm(
            huh.NewGroup(
                huh.NewSelect[string]().
                    Key("class").
                    Options(huh.NewOptions("Warrior", "Mage", "Rogue")...).
                    Title("Choose your class"),
            ),
        ),
    }
}

func (m Model) Init() tea.Cmd {
    return m.form.Init()
}

func (m Model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    form, cmd := m.form.Update(msg)
    if f, ok := form.(*huh.Form); ok {
        m.form = f
    }

    if m.form.State == huh.StateCompleted {
        return m, tea.Quit
    }
    return m, cmd
}

func (m Model) View() string {
    if m.form.State == huh.StateCompleted {
        return fmt.Sprintf("You picked: %s", m.form.GetString("class"))
    }
    return m.form.View()
}
```

Key differences when embedded:
- Do NOT call `.Run()`, use Init/Update/View cycle instead
- Set `SubmitCmd` and `CancelCmd` if you want custom behavior on form completion
- Use `.Key("name")` on fields, retrieve with `form.GetString("name")`
- Check `form.State` to know when the form is done
- Type assert the Update result: `form.(*huh.Form)`

Navigation methods available for programmatic control:
- `form.NextGroup()`, `form.PrevGroup()`
- `form.NextField()`, `form.PrevField()`
- `form.GetFocusedField()`

## Common Mistakes

1. **Forgetting `.Value(&v)`** - Without it, answers go nowhere. The field uses an internal `EmbeddedAccessor` that you cannot read after form completes unless you use `.Key()` + `form.GetString()`.

2. **Using `.Run()` inside Bubble Tea** - Never call `.Run()` on an embedded form. Use the Init/Update/View pattern.

3. **Missing type parameter on Select/MultiSelect** - `huh.NewSelect[string]()` not `huh.NewSelect()`. The generic parameter determines the option value type.

4. **Dynamic binding without pointer** - `TitleFunc(fn, country)` will not work. Must be `TitleFunc(fn, &country)` with a pointer so huh can detect changes.

5. **Timeout in accessible mode** - `WithTimeout()` returns `ErrTimeoutUnsupported` in accessible mode. Guard it.

6. **Not handling ErrUserAborted** - `form.Run()` returns `huh.ErrUserAborted` when user presses ctrl+c. Always check the error.

7. **Form outputs to stderr** - By default, the TUI renders to stderr (stdout stays clean for piping). Use `.WithOutput(os.Stdout)` to change.

## Checklist

- [ ] Import `charm.land/huh/v2`
- [ ] Every field that stores data has `.Value(&var)` or `.Key("name")`
- [ ] Custom validators return `nil` on success, `error` on failure
- [ ] Dynamic fields use `*Func` variants with pointer bindings
- [ ] Accessible mode handled via env var or config flag
- [ ] `ErrUserAborted` handled after `.Run()`
- [ ] Embedded forms use Init/Update/View, not `.Run()`
- [ ] Form state checked via `form.State == huh.StateCompleted`
