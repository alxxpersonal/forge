# Bubble Tea v2 - Full API Reference

Import: `tea "charm.land/bubbletea/v2"`

## Key Constants

Special keys (use with `Key.Code`):

```
KeyUp, KeyDown, KeyLeft, KeyRight, KeyHome, KeyEnd, KeyPgUp, KeyPgDown
KeyInsert, KeyDelete, KeyFind, KeySelect, KeyBegin
KeyBackspace, KeyTab, KeyEnter, KeyReturn, KeyEscape, KeyEsc, KeySpace
KeyF1 through KeyF63
KeyCapsLock, KeyScrollLock, KeyNumLock, KeyPrintScreen, KeyPause, KeyMenu
```

Keypad keys: `KeyKpEnter`, `KeyKp0`-`KeyKp9`, `KeyKpPlus`, `KeyKpMinus`, `KeyKpMultiply`, `KeyKpDivide`, `KeyKpDecimal`, `KeyKpEqual`, `KeyKpComma`, `KeyKpSep`

Modifier keys: `KeyLeftShift`, `KeyRightShift`, `KeyLeftAlt`, `KeyRightAlt`, `KeyLeftCtrl`, `KeyRightCtrl`, `KeyLeftSuper`, `KeyRightSuper`, `KeyLeftMeta`, `KeyRightMeta`, `KeyLeftHyper`, `KeyRightHyper`

## Modifier Constants

```go
ModShift    // Shift key
ModAlt      // Alt/Option key
ModCtrl     // Control key
ModMeta     // Meta key
ModHyper    // Hyper key
ModSuper    // Super/Windows/Command key
ModCapsLock // Caps Lock state
ModNumLock  // Num Lock state
```

## Key Struct

```go
type Key struct {
    Text        string   // printable character(s) - empty for special keys
    Mod         KeyMod   // modifier keys bitmask
    Code        rune     // key code (e.g. KeyEnter, 'a')
    ShiftedCode rune     // shifted key (Kitty protocol only)
    BaseCode    rune     // base key per US layout (Kitty protocol only)
    IsRepeat    bool     // key repeat (Kitty protocol only)
}
```

Methods: `String()`, `Keystroke()`

String() returns the text if available, otherwise falls back to Keystroke().
Keystroke() returns modifier+key format: `"ctrl+shift+alt+a"` (modifiers always in this order: ctrl, alt, shift, meta, hyper, super).

## Mouse Constants

```go
MouseNone, MouseLeft, MouseMiddle, MouseRight
MouseWheelUp, MouseWheelDown, MouseWheelLeft, MouseWheelRight
MouseBackward, MouseForward, MouseButton10, MouseButton11
```

## Mouse Struct

```go
type Mouse struct {
    X, Y   int          // zero-based position, (0,0) = top-left
    Button MouseButton
    Mod    KeyMod
}
```

## Mouse Messages

- `MouseClickMsg` - button pressed
- `MouseReleaseMsg` - button released
- `MouseWheelMsg` - scroll wheel
- `MouseMotionMsg` - mouse moved

All satisfy `MouseMsg` interface with `.Mouse() Mouse` and `.String() string`.

## Mouse Modes

```go
MouseModeNone        // disabled (default)
MouseModeCellMotion  // click, release, wheel, drag
MouseModeAllMotion   // all above + movement without buttons
```

## View Struct

```go
type View struct {
    Content                   string
    OnMouse                   func(MouseMsg) Cmd
    Cursor                    *Cursor
    BackgroundColor           color.Color
    ForegroundColor           color.Color
    WindowTitle               string
    ProgressBar               *ProgressBar
    AltScreen                 bool
    ReportFocus               bool
    DisableBracketedPasteMode bool
    MouseMode                 MouseMode
    KeyboardEnhancements      KeyboardEnhancements
}
```

`NewView(s string) View` - create View from string.
`SetContent(s string)` - set view content.

## Cursor

```go
type Cursor struct {
    Position             // X, Y int
    Color    color.Color
    Shape    CursorShape // CursorBlock, CursorUnderline, CursorBar
    Blink    bool
}
```

`NewCursor(x, y int) *Cursor` - create cursor at position.

## Program Methods

```go
NewProgram(model Model, opts ...ProgramOption) *Program
(p *Program) Run() (Model, error)
(p *Program) Send(msg Msg)
(p *Program) Quit()
(p *Program) Kill()
(p *Program) Wait()
```

## Error Variables

```go
ErrProgramPanic  // recovered from panic
ErrProgramKilled // Kill() was called
ErrInterrupted   // SIGINT or InterruptMsg
```

## Clipboard

```go
SetClipboard(s string) Cmd         // set system clipboard
ReadClipboard() Msg                 // request clipboard content
SetPrimaryClipboard(s string) Cmd   // X11/Wayland primary
ReadPrimaryClipboard() Msg          // X11/Wayland primary
```

`ClipboardMsg` - received clipboard content. Fields: `Content string`, `Selection byte`.

## Color Queries

```go
RequestBackgroundColor() Msg    // returns BackgroundColorMsg
RequestForegroundColor() Msg    // returns ForegroundColorMsg
RequestCursorColor() Msg        // returns CursorColorMsg
```

Each response has `IsDark() bool` and `String() string`.

## Progress Bar

```go
type ProgressBar struct {
    State ProgressBarState  // ProgressBarNone/Default/Error/Indeterminate/Warning
    Value int               // 0-100
}

NewProgressBar(state ProgressBarState, value int) *ProgressBar
```

## Logging

```go
LogToFile(path, prefix string) (*os.File, error)
LogToFileWith(path, prefix string, log LogOptionsSetter) (*os.File, error)
```

## Exec

```go
Exec(cmd ExecCommand, fn ExecCallback) Cmd
ExecProcess(cmd *exec.Cmd, fn ExecCallback) Cmd

type ExecCallback func(error) Msg
```

Pauses the TUI, runs external process (e.g. vim), resumes on completion.
