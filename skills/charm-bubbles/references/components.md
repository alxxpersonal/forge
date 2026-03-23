# Bubbles Component Catalog

All components live under `charm.land/bubbles/v2/<package>`.

## spinner

Animated loading indicator with preset and custom frame sets.

**Key Types:**
- `Model` - component state (Spinner, Style fields)
- `Spinner` - frame definition (`Frames []string`, `FPS time.Duration`)
- `TickMsg` - animation frame message

**Presets:** `Line`, `Dot`, `MiniDot`, `Jump`, `Pulse`, `Points`, `Globe`, `Moon`, `Monkey`, `Meter`, `Hamburger`, `Ellipsis`

**Constructor:**
```go
s := spinner.New(spinner.WithSpinner(spinner.Dot), spinner.WithStyle(style))
```

**Start:** Return `m.spinner.Tick` from your `Init()`.

**Usage:**
```go
m.spinner, cmd = m.spinner.Update(msg)
view := m.spinner.View()
```

---

## textinput

Single-line text input with cursor, placeholder, validation, echo modes, and autocomplete suggestions.

**Key Types:**
- `Model` - component state
- `KeyMap` - customizable keybindings
- `EchoMode` - `EchoNormal`, `EchoPassword`, `EchoNone`
- `ValidateFunc` - `func(string) error`
- `Styles` - focused/blurred state styles

**Constructor:**
```go
ti := textinput.New()
ti.Placeholder = "Type here..."
ti.CharLimit = 100
ti.SetWidth(40)
```

**Key Methods:**
- `Focus() tea.Cmd` / `Blur()` - focus management
- `Value() string` / `SetValue(string)` - get/set content
- `SetSuggestions([]string)` - autocomplete suggestions (set `ShowSuggestions = true`)
- `SetStyles(Styles)` - apply styles
- `Validate ValidateFunc` - assign validation function, errors go to `Err` field

---

## textarea

Multi-line text editor with line numbers, word wrapping, cursor navigation, and viewport scrolling.

**Key Types:**
- `Model` - component state
- `KeyMap` - extensive keybindings (character/word/line movement, case transforms, transpose)
- `LineInfo` - cursor position metadata

**Constructor:**
```go
ta := textarea.New()
ta.SetWidth(60)
ta.SetHeight(10)
ta.Placeholder = "Enter text..."
ta.ShowLineNumbers = true
```

**Key Methods:**
- `Focus() tea.Cmd` / `Blur()` - focus management
- `Value() string` / `SetValue(string)` - get/set content
- `Line() int` / `LineCount() int` - cursor line info
- `SetStyles(Styles)` - focused/blurred styles

---

## list

Batteries-included list browser with fuzzy filtering, pagination, help, spinner, and status messages.

**Key Types:**
- `Model` - component state
- `Item` interface - must implement `FilterValue() string`
- `ItemDelegate` interface - `Render`, `Height`, `Spacing`, `Update` methods
- `FilterState` - `Unfiltered`, `Filtering`, `FilterApplied`
- `FilterFunc` - custom filter function
- `Rank` - filter match result

**Constructor:**
```go
items := []list.Item{myItem1, myItem2}
l := list.New(items, myDelegate, width, height)
l.Title = "My List"
```

**Key Methods:**
- `SetItems([]Item)` / `Items() []Item` - manage items
- `SelectedItem() Item` / `Index() int` - get selection
- `SetSize(w, h int)` - resize
- `SetFilteringEnabled(bool)` - toggle filtering
- `NewStatusMessage(string) tea.Cmd` - show temporary status
- `SetShowHelp(bool)` / `SetShowTitle(bool)` / `SetShowFilter(bool)` - toggle UI elements

**Built-in Filter:** `list.DefaultFilter` (fuzzy), `list.UnsortedFilter`

---

## table

Tabular data display with row selection and keyboard navigation.

**Key Types:**
- `Model` - component state
- `Row` - `[]string`
- `Column` - `Title string`, `Width int`
- `KeyMap` - up/down/page/goto keybindings
- `Styles` - `Header`, `Cell`, `Selected`

**Constructor:**
```go
t := table.New(
    table.WithColumns([]table.Column{
        {Title: "Name", Width: 20},
        {Title: "Age", Width: 5},
    }),
    table.WithRows([]table.Row{
        {"Alice", "30"},
        {"Bob", "25"},
    }),
    table.WithHeight(10),
    table.WithFocused(true),
)
```

**Key Methods:**
- `Focus()` / `Blur()` - focus management
- `SelectedRow() Row` / `Cursor() int` - get selection
- `SetRows([]Row)` / `SetColumns([]Column)` - update data
- `SetWidth(int)` / `SetHeight(int)` - resize
- `SetCursor(int)` - set selection index
- `FromValues(value, separator string)` - parse string data
- `HelpView() string` - render help from keymap
- `UpdateViewport()` - refresh after data changes

---

## viewport

Scrollable content viewer with vertical/horizontal scrolling, soft wrap, mouse wheel, line gutters, and text highlighting.

**Key Types:**
- `Model` - component state
- `KeyMap` - scroll keybindings (Up/Down/Left/Right/PageUp/PageDown/HalfPageUp/HalfPageDown)
- `GutterFunc` / `GutterContext` - left gutter rendering (line numbers, etc.)

**Constructor:**
```go
vp := viewport.New(viewport.WithWidth(80), viewport.WithHeight(24))
vp.SetContent("long text content here...")
```

**Key Methods:**
- `SetContent(string)` / `GetContent() string` - set/get content
- `SetContentLines([]string)` - set content as lines
- `SetWidth(int)` / `SetHeight(int)` - resize
- `ScrollDown(n)` / `ScrollUp(n)` / `ScrollLeft(n)` / `ScrollRight(n)` - scroll
- `GotoTop()` / `GotoBottom()` - jump to extremes
- `AtTop() bool` / `AtBottom() bool` - position checks
- `ScrollPercent() float64` - scroll progress (0-1)
- `YOffset() int` / `SetYOffset(int)` - vertical position
- `SetHighlights([][]int)` / `HighlightNext()` / `HighlightPrevious()` - search highlighting
- `EnsureVisible(line, colstart, colend int)` - scroll to show position

**Fields:**
- `SoftWrap bool` - enable word wrap (disables horizontal scroll)
- `FillHeight bool` - pad with empty lines
- `MouseWheelEnabled bool` / `MouseWheelDelta int` - mouse config
- `LeftGutterFunc GutterFunc` - line number gutter
- `Style lipgloss.Style` - border/padding style
- `HighlightStyle` / `SelectedHighlightStyle` - search match styles
- `StyleLineFunc func(int) lipgloss.Style` - per-line styling

---

## paginator

Pagination logic and display (dot or numeric style).

**Key Types:**
- `Model` - component state
- `Type` - `Arabic` (1/5), `Dots` (bullet dots)
- `KeyMap` - `PrevPage`, `NextPage`

**Constructor:**
```go
p := paginator.New(paginator.WithPerPage(10), paginator.WithTotalPages(5))
```

**Key Methods:**
- `NextPage()` / `PrevPage()` - navigate
- `SetTotalPages(itemCount int) int` - calculate pages from item count
- `GetSliceBounds(length int) (start, end int)` - get slice indices for current page
- `ItemsOnPage(totalItems int) int` - items on current page
- `OnFirstPage() bool` / `OnLastPage() bool` - position checks

**Fields:** `Page`, `PerPage`, `TotalPages`, `ActiveDot`, `InactiveDot`, `ArabicFormat`

---

## progress

Animated progress bar with solid, gradient, and dynamic color fills.

**Key Types:**
- `Model` - component state
- `FrameMsg` - animation tick
- `ColorFunc` - `func(total, current float64) color.Color`

**Constructor:**
```go
p := progress.New(
    progress.WithDefaultBlend(),      // purple-to-pink gradient
    progress.WithWidth(40),
    progress.WithoutPercentage(),
)
// Or solid color:
p := progress.New(progress.WithColors(lipgloss.Color("#FF0000")))
// Or dynamic:
p := progress.New(progress.WithColorFunc(myColorFunc))
```

**Key Methods:**
- `SetPercent(float64) tea.Cmd` - animate to percentage (0-1), returns cmd
- `IncrPercent(float64) tea.Cmd` / `DecrPercent(float64) tea.Cmd` - relative changes
- `ViewAs(float64) string` - static render at given percentage (no animation)
- `View() string` - render current animated state
- `Percent() float64` - current target percentage
- `IsAnimating() bool` - whether animation is in progress
- `SetWidth(int)` / `Width() int` - resize
- `SetSpringOptions(frequency, damping float64)` - animation physics

**Fields:** `Full`, `Empty` (runes), `FullColor`, `EmptyColor`, `ShowPercentage`, `PercentFormat`, `PercentageStyle`

---

## help

Auto-generated horizontal help view from keybindings. Supports short (single line) and full (multi-column) modes.

**Key Types:**
- `Model` - component state
- `KeyMap` interface - `ShortHelp() []key.Binding`, `FullHelp() [][]key.Binding`
- `Styles` - separate styles for short/full key/desc/separator

**Constructor:**
```go
h := help.New()
h.SetWidth(80)
```

**Usage:**
```go
// Pass your KeyMap implementation
view := h.View(myKeyMap)

// Or render directly
view := h.ShortHelpView(bindings)
view := h.FullHelpView(bindingGroups)
```

**Fields:** `ShowAll bool` (toggle short/full), `ShortSeparator`, `FullSeparator`, `Ellipsis`

---

## filepicker

File system browser for selecting files/directories.

**Key Types:**
- `Model` - component state
- `KeyMap` - navigation keys (Up/Down/Back/Open/Select/PageUp/PageDown/GoToTop/GoToLast)
- `Styles` - styles for cursor, directory, file, symlink, permissions, etc.

**Constructor:**
```go
fp := filepicker.New()
fp.CurrentDirectory = "/home/user"
fp.AllowedTypes = []string{".go", ".md"}
fp.DirAllowed = false
fp.FileAllowed = true
fp.ShowHidden = true
```

**Key Methods:**
- `DidSelectFile(msg tea.Msg) (bool, string)` - check if user selected a valid file
- `DidSelectDisabledFile(msg tea.Msg) (bool, string)` - check if user tried to select a disallowed file

**Fields:** `CurrentDirectory`, `AllowedTypes`, `ShowPermissions`, `ShowSize`, `ShowHidden`, `DirAllowed`, `FileAllowed`, `AutoHeight`, `Path`, `FileSelected`

---

## timer

Countdown timer with configurable interval.

**Key Types:**
- `Model` - component state
- `TickMsg` - periodic tick (has `Timeout bool` flag)
- `TimeoutMsg` - sent once when timer expires
- `StartStopMsg` - control message

**Constructor:**
```go
t := timer.New(30*time.Second, timer.WithInterval(100*time.Millisecond))
```

**Start:** Return `m.timer.Init()` from your `Init()`.

**Key Methods:**
- `Start() tea.Cmd` / `Stop() tea.Cmd` / `Toggle() tea.Cmd` - control
- `Running() bool` / `Timedout() bool` - state queries
- `View() string` - renders remaining time

**Fields:** `Timeout time.Duration`, `Interval time.Duration`

---

## stopwatch

Count-up timer with start/stop/reset.

**Key Types:**
- `Model` - component state
- `TickMsg` - periodic tick
- `StartStopMsg` / `ResetMsg` - control messages

**Constructor:**
```go
sw := stopwatch.New(stopwatch.WithInterval(100*time.Millisecond))
```

**Start:** Return `m.stopwatch.Init()` from your `Init()`.

**Key Methods:**
- `Start() tea.Cmd` / `Stop() tea.Cmd` / `Toggle() tea.Cmd` / `Reset() tea.Cmd` - control
- `Running() bool` - state query
- `Elapsed() time.Duration` - get elapsed time
- `View() string` - renders elapsed time

**Fields:** `Interval time.Duration`

---

## cursor

Virtual cursor used internally by textinput and textarea. Supports blink, static, and hidden modes.

**Key Types:**
- `Model` - cursor state
- `Mode` - `CursorBlink`, `CursorStatic`, `CursorHide`
- `BlinkMsg` - blink tick

**Constructor:**
```go
c := cursor.New()
```

**Key Methods:**
- `Focus() tea.Cmd` / `Blur()` - focus management
- `SetMode(Mode) tea.Cmd` - set cursor behavior
- `Mode() Mode` - get current mode
- `SetChar(string)` - character under cursor
- `Blink() tea.Cmd` - start blink cycle

**Fields:** `Style`, `TextStyle`, `BlinkSpeed`, `IsBlinked`

---

## key

Non-visual utility for defining remappable keybindings with help text integration.

**Key Types:**
- `Binding` - a set of keys with optional help text
- `BindingOpt` - functional option for NewBinding
- `Help` - `Key string`, `Desc string`

**Constructor:**
```go
b := key.NewBinding(
    key.WithKeys("k", "up"),
    key.WithHelp("up/k", "move up"),
)
```

**Key Functions:**
- `key.Matches(key fmt.Stringer, bindings ...Binding) bool` - check if key matches any binding

**Key Methods on Binding:**
- `Enabled() bool` / `SetEnabled(bool)` - enable/disable
- `Keys() []string` / `SetKeys(...string)` - get/set keys
- `Help() Help` / `SetHelp(key, desc string)` - get/set help text
- `Unbind()` - remove all keys and help

---

## runeutil (internal)

Internal sanitizer for cleaning rune input (tab/newline replacement). Not part of the public API.
