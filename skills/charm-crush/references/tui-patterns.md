# TUI Architecture Patterns from Crush

Extracted from `internal/ui/` of charmbracelet/crush. Every pattern here is battle-tested in production.

## Table of Contents

1. [The Centralized Model Pattern](#the-centralized-model-pattern)
2. [Layout System](#layout-system)
3. [Focus Management](#focus-management)
4. [Key Event Routing](#key-event-routing)
5. [Streaming Rendering](#streaming-rendering)
6. [Responsive Design](#responsive-design)
7. [Mouse Handling](#mouse-handling)
8. [Animation System](#animation-system)
9. [Screen Buffer Rendering](#screen-buffer-rendering)

## The Centralized Model Pattern

**Source:** `internal/ui/model/ui.go`

The UI struct is the sole `tea.Model`. It owns all state directly:

```go
type UI struct {
    com          *common.Common
    width, height int
    layout       uiLayout
    state        uiState       // uiOnboarding | uiInitialize | uiLanding | uiChat
    focus        uiFocusState  // uiFocusNone | uiFocusEditor | uiFocusMain

    textarea    textarea.Model      // bubbles textarea
    chat        *Chat               // custom list wrapper
    dialog      *dialog.Overlay     // stacked dialog system
    completions *completions.Completions
    attachments *attachments.Attachments
    status      *Status
    header      *header
    // ... more fields
}
```

Sub-components do NOT implement `tea.Model`. They expose imperative methods:

```go
// Chat has no Update method. The parent calls these directly:
m.chat.HandleMouseDown(x, y)
m.chat.ScrollBy(n)
m.chat.SetMessages(items...)
m.chat.Animate(msg)

// Completions has a non-standard Update returning bool (consumed?):
consumed := m.completions.Update(msg)

// Sidebar is just a render method on UI:
m.drawSidebar(scr, layout.sidebar)
```

The Update method is one large switch statement routing all messages:

```go
func (m *UI) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    var cmds []tea.Cmd
    switch msg := msg.(type) {
    case tea.WindowSizeMsg:
        m.width, m.height = msg.Width, msg.Height
        m.updateLayoutAndSize()
    case tea.KeyPressMsg:
        if cmd := m.handleKeyPressMsg(msg); cmd != nil { cmds = append(cmds, cmd) }
    case pubsub.Event[message.Message]:
        // route to chat
    case pubsub.Event[permission.PermissionRequest]:
        // open dialog
    case anim.StepMsg:
        // route to chat animation
    case spinner.TickMsg:
        // route to spinner
    // ... 30+ message types
    }
    return m, tea.Batch(cmds...)
}
```

**Why not standard Elm per-component?** When you have 10+ components that share state (focus, session, layout), message forwarding becomes a maze. The centralized model keeps all transitions in one place. Components become simple render+method structs.

## Layout System

**Source:** `internal/ui/model/ui.go`

Layout uses `image.Rectangle` from Go's standard library via ultraviolet's layout helpers:

```go
type uiLayout struct {
    area           uv.Rectangle  // full terminal
    header         uv.Rectangle
    main           uv.Rectangle  // chat area
    pills          uv.Rectangle  // todo pills
    editor         uv.Rectangle  // text input
    sidebar        uv.Rectangle  // session details (non-compact)
    status         uv.Rectangle  // help/status bar
    sessionDetails uv.Rectangle  // compact mode overlay
}
```

Layout is computed in `generateLayout()` based on state. Different states get different layouts:

```go
func (m *UI) generateLayout(w, h int) uiLayout {
    area := image.Rect(0, 0, w, h)
    helpHeight := 1
    editorHeight := 5
    sidebarWidth := 30

    // Add margins
    appRect, helpRect := layout.SplitVertical(area, layout.Fixed(area.Dy()-helpHeight))
    appRect.Min.Y += 1; appRect.Max.Y -= 1
    appRect.Min.X += 1; appRect.Max.X -= 1

    switch m.state {
    case uiChat:
        if m.isCompact {
            // Compact: header | main | editor | help (no sidebar)
            headerRect, mainRect := layout.SplitVertical(appRect, layout.Fixed(1))
            mainRect, editorRect := layout.SplitVertical(mainRect, layout.Fixed(mainRect.Dy()-editorHeight))
        } else {
            // Full: main+editor | sidebar, with help bar at bottom
            mainRect, sideRect := layout.SplitHorizontal(appRect, layout.Fixed(appRect.Dx()-sidebarWidth))
            mainRect, editorRect := layout.SplitVertical(mainRect, layout.Fixed(mainRect.Dy()-editorHeight))
        }
    case uiLanding:
        // header | main | editor | help
    case uiOnboarding, uiInitialize:
        // header | main | help
    }
}
```

Key pattern: `layout.SplitVertical` and `layout.SplitHorizontal` take an area and a constraint (`layout.Fixed(n)`) and return two non-overlapping rectangles.

After computing layout, `updateSize()` propagates sizes to child components:

```go
func (m *UI) updateSize() {
    m.status.SetWidth(m.layout.status.Dx())
    m.chat.SetSize(m.layout.main.Dx(), m.layout.main.Dy())
    m.textarea.SetWidth(m.layout.editor.Dx())
    m.textarea.SetHeight(m.layout.editor.Dy() - 2) // account for margins
}
```

`updateLayoutAndSize()` is called on every `WindowSizeMsg` and state change.

## Focus Management

**Source:** `internal/ui/model/ui.go`

Focus is a simple enum:

```go
type uiFocusState uint8
const (
    uiFocusNone uiFocusState = iota
    uiFocusEditor  // keys go to textarea
    uiFocusMain    // keys go to chat list
)
```

Tab switches focus. Focus determines:
1. Where key events route
2. Which cursor shows
3. Which component gets highlight/border styling

```go
// In the key handler:
case key.Matches(msg, m.keyMap.Tab):
    if m.focus == uiFocusEditor {
        m.focus = uiFocusMain
    } else {
        m.focus = uiFocusEditor
    }
```

Cursor position is returned from `Draw()` based on focus:

```go
func (m *UI) Draw(scr uv.Screen, area uv.Rectangle) *tea.Cursor {
    // ... draw all components ...
    if m.dialog.HasDialogs() {
        return m.dialog.Draw(scr, scr.Bounds()) // dialog cursor takes priority
    }
    switch m.focus {
    case uiFocusEditor:
        if m.textarea.Focused() {
            cur := m.textarea.Cursor()
            cur.X += 1                            // app margin
            cur.Y += m.layout.editor.Min.Y + 1    // editor position + attachment row
            return cur
        }
    }
    return nil
}
```

## Key Event Routing

**Source:** `internal/ui/model/keys.go`, `internal/ui/model/ui.go`

Keys are defined as `key.Binding` structs organized in a `KeyMap`:

```go
type KeyMap struct {
    Quit     key.Binding
    Help     key.Binding
    Tab      key.Binding
    Commands key.Binding
    Models   key.Binding
    Editor   EditorKeyMap
    Chat     ChatKeyMap
}
```

The key handler delegates based on state and focus:

```go
func (m *UI) handleKeyPressMsg(msg tea.KeyPressMsg) tea.Cmd {
    // Dialogs intercept first
    if m.dialog.HasDialogs() {
        return m.handleDialogMsg(msg)
    }
    // Global keys (quit, help, commands, models)
    // Then state-specific:
    switch m.state {
    case uiChat:
        switch m.focus {
        case uiFocusEditor:
            return m.handleEditorKey(msg)
        case uiFocusMain:
            return m.handleChatKey(msg)
        }
    }
}
```

Keyboard enhancements are detected and key help is updated dynamically:

```go
case tea.KeyboardEnhancementsMsg:
    m.keyenh = msg
    if msg.SupportsKeyDisambiguation() {
        m.keyMap.Models.SetHelp("ctrl+m", "models")
        m.keyMap.Editor.Newline.SetHelp("shift+enter", "newline")
    }
```

## Streaming Rendering

**Source:** `internal/agent/agent.go`, `internal/ui/model/ui.go`

The agent loop runs in a goroutine. It uses fantasy's streaming callbacks:

```go
result, err := agent.Stream(ctx, fantasy.AgentStreamCall{
    OnTextDelta: func(id string, text string) error {
        currentAssistant.AppendContent(text)
        return a.messages.Update(ctx, *currentAssistant)  // persists and publishes
    },
    // ... other callbacks
})
```

`messages.Update()` triggers a pubsub event. The app converts pubsub events to `tea.Msg` via a channel that bubbletea reads. The UI receives `pubsub.Event[message.Message]` and updates the chat:

```go
case pubsub.Event[message.Message]:
    switch msg.Type {
    case pubsub.UpdatedEvent:
        cmds = append(cmds, m.updateSessionMessage(msg.Payload))
    }
```

The chat item re-renders its cached content on update. Glamour renders markdown in real-time as chunks arrive.

The chat has a `follow` flag - when true (user hasn't scrolled up), it auto-scrolls to bottom on every new content:

```go
if m.chat.Follow() {
    if cmd := m.chat.ScrollToBottomAndAnimate(); cmd != nil {
        cmds = append(cmds, cmd)
    }
}
```

## Responsive Design

**Source:** `internal/ui/model/ui.go`

Compact mode triggers automatically at breakpoints:

```go
const (
    compactModeWidthBreakpoint  = 120
    compactModeHeightBreakpoint = 30
)

func (m *UI) updateLayoutAndSize() {
    if m.state == uiChat {
        if m.forceCompactMode {
            m.isCompact = true
            return
        }
        if m.width < compactModeWidthBreakpoint || m.height < compactModeHeightBreakpoint {
            m.isCompact = true
        } else {
            m.isCompact = false
        }
    }
    m.layout = m.generateLayout(m.width, m.height)
    m.updateSize()
}
```

Compact mode removes the sidebar and replaces it with a compact header. Users can also force compact mode via config (`options.tui.compact_mode`).

Text width is capped for readability:

```go
const maxTextWidth = 120  // in chat/messages.go
```

## Mouse Handling

**Source:** `internal/ui/model/ui.go`

Mouse events are handled per-type with coordinate translation:

```go
case tea.MouseClickMsg:
    // Dialogs first
    if m.dialog.HasDialogs() {
        m.dialog.Update(msg)
        return m, tea.Batch(cmds...)
    }
    // Focus click (editor vs chat)
    if cmd := m.handleClickFocus(msg); cmd != nil { ... }
    // Translate to chat-local coordinates
    x := msg.X - m.layout.main.Min.X
    y := msg.Y - m.layout.main.Min.Y
    if handled, cmd := m.chat.HandleMouseDown(x, y); handled { ... }

case tea.MouseWheelMsg:
    switch msg.Button {
    case tea.MouseWheelUp:
        m.chat.ScrollByAndAnimate(-MouseScrollThreshold)
    case tea.MouseWheelDown:
        m.chat.ScrollByAndAnimate(MouseScrollThreshold)
    }
```

The chat tracks drag state for text selection with double/triple click detection:

```go
case tea.MouseReleaseMsg:
    if m.chat.HandleMouseUp(x, y) && m.chat.HasHighlight() {
        // Delay to detect double-click vs single-click
        cmds = append(cmds, tea.Tick(doubleClickThreshold, func(t time.Time) tea.Msg {
            if time.Since(m.lastClickTime) >= doubleClickThreshold {
                return copyChatHighlightMsg{}
            }
            return nil
        }))
    }
```

## Animation System

**Source:** `internal/ui/anim/anim.go`

Animations use a `StepMsg` tick message. The chat routes it to animatable items:

```go
case anim.StepMsg:
    if m.state == uiChat {
        if cmd := m.chat.Animate(msg); cmd != nil {
            cmds = append(cmds, cmd)
        }
        if m.chat.Follow() {
            m.chat.ScrollToBottomAndAnimate()
        }
    }
```

Items that support animation implement the `Animatable` interface:

```go
type Animatable interface {
    StartAnimation() tea.Cmd
    Animate(msg anim.StepMsg) tea.Cmd
}
```

Scrolling is animated - methods like `ScrollByAndAnimate()` and `ScrollToBottomAndAnimate()` smooth the transition.

## Screen Buffer Rendering

**Source:** `internal/ui/model/ui.go`

The View() -> Draw() pipeline:

```go
func (m *UI) View() tea.View {
    var v tea.View
    v.AltScreen = true
    v.BackgroundColor = m.com.Styles.Background
    v.MouseMode = tea.MouseModeCellMotion

    canvas := uv.NewScreenBuffer(m.width, m.height)
    v.Cursor = m.Draw(canvas, canvas.Bounds())

    content := canvas.Render()
    // Normalize newlines, trim trailing spaces
    v.Content = content
    return v
}

func (m *UI) Draw(scr uv.Screen, area uv.Rectangle) *tea.Cursor {
    screen.Clear(scr)
    switch m.state {
    case uiChat:
        if m.isCompact {
            m.drawHeader(scr, layout.header)
        } else {
            m.drawSidebar(scr, layout.sidebar)
        }
        m.chat.Draw(scr, layout.main)
        uv.NewStyledString(m.renderEditorView(editorWidth)).Draw(scr, layout.editor)
    }
    // Status bar
    m.status.Draw(scr, layout.status)
    // Completions popup (overlays)
    if m.completionsOpen { ... }
    // Dialogs last (always on top)
    if m.dialog.HasDialogs() {
        return m.dialog.Draw(scr, scr.Bounds())
    }
    // Return cursor based on focus
}
```

The screen buffer handles:
- Z-ordering (components drawn later overlay earlier ones)
- Cursor from the last-drawn focused component
- Efficient diffing (bubbletea handles this)

Terminal progress bar is enabled for supported terminals:

```go
if m.progressBarEnabled && m.sendProgressBar && m.isAgentBusy() {
    v.ProgressBar = tea.NewProgressBar(tea.ProgressBarIndeterminate, rand.Intn(100))
}
```
