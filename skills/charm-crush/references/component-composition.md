# Component Composition Patterns from Crush

How Crush composes bubbletea + lipgloss + bubbles + ultraviolet into a production multi-pane TUI.

## Table of Contents

1. [Interface Hierarchy](#interface-hierarchy)
2. [Chat Item System](#chat-item-system)
3. [Dialog Stack](#dialog-stack)
4. [List Component](#list-component)
5. [Shared Context Pattern](#shared-context-pattern)
6. [Tool Renderer Factory](#tool-renderer-factory)
7. [Completions Popup](#completions-popup)
8. [Notification System](#notification-system)

## Interface Hierarchy

**Source:** `internal/ui/list/item.go`, `internal/ui/chat/messages.go`, `internal/ui/chat/tools.go`

Crush builds a layered interface system through composition, not inheritance:

```
list.Item (base)
  Render(width int) string

list.RawRenderable
  RawRender(width int) string

list.Focusable
  SetFocused(focused bool)

list.Highlightable
  SetHighlight(startLine, startCol, endLine, endCol int)
  Highlight() (startLine, startCol, endLine, endCol int)

list.MouseClickable
  HandleMouseClick(btn MouseButton, x, y int) bool
```

Chat extends these:

```
chat.MessageItem = list.Item + list.RawRenderable + Identifiable
  ID() string

chat.ToolMessageItem extends MessageItem with:
  ToolCall() message.ToolCall
  ToolResult() *message.ToolResult
  MessageID() string
  SetToolResult(result *message.ToolResult)
  ToolStatus() string
```

Opt-in capabilities (components implement only what they need):

```
chat.Animatable
  StartAnimation() tea.Cmd
  Animate(msg anim.StepMsg) tea.Cmd

chat.Expandable
  ToggleExpanded() bool

chat.Compactable
  SetCompact(compact bool)

chat.KeyEventHandler
  HandleKeyEvent(key tea.KeyMsg) (bool, tea.Cmd)

chat.NestedToolContainer
  SetNestedTools(tools []ToolMessageItem)
```

This design lets the Chat and List check capabilities at runtime:

```go
// In UI.Update when handling animation:
for _, item := range items {
    if animatable, ok := item.(chat.Animatable); ok {
        if cmd := animatable.StartAnimation(); cmd != nil {
            cmds = append(cmds, cmd)
        }
    }
}

// In rendering, check if item supports compact mode:
if simplifiable, ok := nestedToolItem.(chat.Compactable); ok {
    simplifiable.SetCompact(true)
}
```

## Chat Item System

**Source:** `internal/ui/chat/`

The Chat wraps a `list.List` and provides message-specific behavior. Items are created by `ExtractMessageItems()` which parses a `message.Message` into one or more `MessageItem`s.

### Message Types

| File | Handles | Key Feature |
|------|---------|-------------|
| `chat/user.go` | User messages | Shows input text + attachments |
| `chat/assistant.go` | Assistant text, thinking, errors | Streaming markdown, reasoning blocks, info footer |
| `chat/bash.go` | Bash, JobOutput, JobKill | Command display, output truncation, job tracking |
| `chat/file.go` | View, Write, Edit, MultiEdit, Download | Diff rendering, file path display |
| `chat/search.go` | Glob, Grep, LS, Sourcegraph | Result lists with truncation |
| `chat/fetch.go` | Fetch, WebFetch, WebSearch | URL display, content preview |
| `chat/agent.go` | Agent, AgenticFetch | Nested tool containers |
| `chat/mcp.go` | MCP tools (mcp_ prefix) | Generic tool rendering with server name |
| `chat/generic.go` | Fallback | Any unrecognized tool |
| `chat/diagnostics.go` | LSP diagnostics | Error/warning lists |
| `chat/todos.go` | Todo lists | Checkbox rendering |

### Cached Rendering

Items cache their rendered output and invalidate when data changes. This is critical because the list only renders visible items (lazy rendering), but those items may be re-rendered on every frame during streaming.

```go
// Pattern from chat/messages.go - embedded cache struct
type cachedMessageItem struct {
    cachedRender string
    cachedWidth  int
    dirty        bool
}

func (c *cachedMessageItem) invalidate() {
    c.dirty = true
}

func (c *cachedMessageItem) render(width int, renderFn func(int) string) string {
    if !c.dirty && c.cachedWidth == width {
        return c.cachedRender
    }
    c.cachedRender = renderFn(width)
    c.cachedWidth = width
    c.dirty = false
    return c.cachedRender
}
```

### Tool Renderer Factory

**Source:** `internal/ui/chat/tools.go`

`NewToolMessageItem` is a central factory routing tool names to specific types:

```go
func NewToolMessageItem(sty *styles.Styles, msg *message.Message, tc message.ToolCall, result *message.ToolResult) ToolMessageItem {
    switch tc.Name {
    case "bash":
        return newBashItem(sty, msg, tc, result)
    case "edit", "multiedit":
        return newEditItem(sty, msg, tc, result)
    case "view":
        return newViewItem(sty, msg, tc, result)
    // ... etc
    default:
        if strings.HasPrefix(tc.Name, "mcp_") {
            return newMCPItem(sty, msg, tc, result)
        }
        return newGenericItem(sty, msg, tc, result)
    }
}
```

Each tool renderer implements `RenderTool(sty *styles.Styles, width int, opts *ToolRenderOpts) string` which produces the styled output.

## Dialog Stack

**Source:** `internal/ui/dialog/dialog.go`

Dialogs are an overlay system managed by `dialog.Overlay`:

```go
type Dialog interface {
    ID() string
    HandleMsg(msg tea.Msg) Action  // returns typed action
    Draw(scr uv.Screen, area uv.Rectangle) *tea.Cursor
}

type Overlay struct {
    dialogs []Dialog  // stack
}

func (d *Overlay) OpenDialog(dialog Dialog)      // push
func (d *Overlay) CloseFrontDialog()             // pop
func (d *Overlay) ContainsDialog(id string) bool // check
func (d *Overlay) HasDialogs() bool              // any open?
```

Dialog implementations in `internal/ui/dialog/`:

| Dialog | ID | Purpose |
|--------|----|---------|
| `Models` | "models" | Model picker with provider groups |
| `Sessions` | "sessions" | Session browser with search |
| `Commands` | "commands" | Slash command picker |
| `Permissions` | "permissions" | Tool approval with diff view |
| `APIKeyInput` | "api_key_input" | API key entry |
| `OAuthCopilot` | "oauth_copilot" | GitHub Copilot OAuth flow |
| `FilePicker` | "filepicker" | File browser |
| `Reasoning` | "reasoning" | Extended thinking view |
| `Quit` | "quit" | Quit confirmation |
| `Arguments` | "arguments" | Skill argument input |

### Dialog Actions

Dialogs return typed `Action` values that the parent UI handles:

```go
// In UI.handleDialogMsg:
action := m.dialog.Update(msg)
switch a := action.(type) {
case dialog.ModelSelectedAction:
    // user picked a model
case dialog.PermissionAction:
    // user allowed/denied tool
case dialog.SessionSelectedAction:
    // user switched session
}
```

This decouples dialog logic from the parent. Dialogs don't need to know about the UI - they just return what happened.

### Dialog Rendering

Dialogs use `RenderContext` for consistent layout:

```go
rc := dialog.NewRenderContext(styles, width)
rc.Title = "Select Model"
rc.Parts = []string{modelList, helpText}
rc.Help = helpView
rendered := rc.Render()
```

`RenderContext` handles title gradient, content alignment, help bar positioning, and onboarding-mode layout (bottom-left instead of centered).

## List Component

**Source:** `internal/ui/list/list.go` (650 lines)

A custom lazy-rendered scrollable list. Not the standard bubbles list - this is crush-specific.

Key design decisions:
- **Lazy rendering**: only visible items are rendered
- **No internal cache**: items cache their own rendering (see cached render pattern above)
- **Scroll by lines, not items**: viewport tracks `offsetIdx` (first visible item) + `offsetLine` (lines scrolled within that item)
- **Render callbacks**: the parent can modify items before rendering via `RegisterRenderCallback()`

```go
type List struct {
    width, height int
    items         []Item
    gap           int
    reverse       bool
    focused       bool
    selectedIdx   int
    offsetIdx     int
    offsetLine    int
    renderCallbacks []func(idx, selectedIdx int, item Item) Item
}
```

The `Render()` method walks visible items, renders each to a string, joins with gap, and truncates to viewport height. This is the string that Chat then draws onto the ultraviolet screen buffer.

The list supports both forward and reverse rendering (`reverse` flag) for chat-style bottom-up layouts.

## Shared Context Pattern

**Source:** `internal/ui/common/common.go`

```go
type Common struct {
    App    *app.App
    Styles *styles.Styles
}

func (c *Common) Config() *config.Config { return c.App.Config() }
func (c *Common) Store() *config.ConfigStore { ... }
```

Every component that needs styles or app access receives `*common.Common` at creation:

```go
ch := NewChat(com)       // chat
status := NewStatus(com, ui)  // status bar
header := newHeader(com)      // header
```

This avoids passing individual dependencies and makes it easy to add new shared state.

## Completions Popup

**Source:** `internal/ui/completions/completions.go`

The `@` completions popup shows files and MCP resources. It's positioned relative to the cursor:

```go
// In UI.Draw():
if m.completionsOpen && m.completions.HasItems() {
    w, h := m.completions.Size()
    x := m.completionsPositionStart.X
    y := m.completionsPositionStart.Y - h  // above cursor

    // Keep within screen bounds
    if x+w > screenW { x = screenW - w }
    x = max(0, x)
    y = max(0, y+1)

    completionsView := uv.NewStyledString(m.completions.Render())
    completionsView.Draw(scr, image.Rectangle{
        Min: image.Pt(x, y),
        Max: image.Pt(x+w, y+h),
    })
}
```

The completions component has its own key handling that returns whether it consumed the event:

```go
// Non-standard Update signature
func (c *Completions) Update(msg tea.Msg) bool {
    // returns true if it handled the key (consumed)
}
```

Items are loaded asynchronously via `completions.CompletionItemsLoadedMsg`.

## Notification System

**Source:** `internal/ui/notification/`

Desktop notifications are sent when:
- A tool needs permission approval
- The agent finishes its turn
- But ONLY when the terminal window is unfocused AND the terminal supports focus reporting

```go
func (m *UI) shouldSendNotification() bool {
    if cfg.Options.DisableNotifications { return false }
    return m.caps.ReportFocusEvents && !m.notifyWindowFocused
}
```

Focus tracking uses bubbletea's `tea.FocusMsg` / `tea.BlurMsg`:

```go
case tea.FocusMsg:
    m.notifyWindowFocused = true
case tea.BlurMsg:
    m.notifyWindowFocused = false
```

The notification backend is swapped based on terminal capability:

```go
case tea.ModeReportMsg:
    if m.caps.ReportFocusEvents {
        m.notifyBackend = notification.NewNativeBackend(notification.Icon)
    }
```

If the terminal doesn't support focus reporting, notifications stay disabled (NoopBackend).
