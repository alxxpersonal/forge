# Design System from Crush

The visual language of Crush, extracted from `internal/ui/styles/styles.go` and component rendering code.

## Table of Contents

1. [Charmtone Color Palette](#charmtone-color-palette)
2. [Semantic Color Roles](#semantic-color-roles)
3. [The Styles Struct](#the-styles-struct)
4. [Icon System](#icon-system)
5. [Typography Patterns](#typography-patterns)
6. [Layout Constants](#layout-constants)
7. [Gradient Effects](#gradient-effects)

## Charmtone Color Palette

**Source:** `github.com/charmbracelet/x/exp/charmtone`

Crush uses Charm's proprietary color palette "Charmtone" with food-themed names. These are true-color values designed to look good on dark terminal backgrounds.

| Role | Charmtone Name | Usage |
|------|---------------|-------|
| Primary | `Charple` | Focus borders, highlights, accent |
| Secondary | `Dolly` | Cursor color, secondary highlights |
| Tertiary | `Bok` | Editor prompt, tertiary accent |
| Background | `Pepper` | Main background |
| Background lighter | `BBQ` | Slightly lighter bg for contrast |
| Background subtle | `Charcoal` | Borders, separators |
| Background overlay | `Iron` | Dialog overlays |
| Foreground | `Ash` | Primary text |
| Foreground muted | `Squid` | Secondary text |
| Foreground half-muted | `Smoke` | Between muted and base |
| Foreground subtle | `Oyster` | Placeholders, hints |
| Error | `Sriracha` | Error states |
| Warning | `Zest` | Warning states |
| Info | `Malibu` | Info states, blue accents |
| White | `Butter` | High-contrast text |
| Blue | `Malibu` | Tool names, links |
| Blue light | `Sardine` | Light blue accents |
| Blue dark | `Damson` | Dark blue accents |
| Green | `Julep` | Success states |
| Green light | `Bok` | Light green accents |
| Green dark | `Guac` | Pending job icons |
| Red | `Coral` | Error text |
| Red dark | `Sriracha` | Error backgrounds |
| Yellow | `Mustard` | Warning text, note tags |

## Semantic Color Roles

**Source:** `internal/ui/styles/styles.go`

Colors are organized by semantic role, not by hue:

```go
var (
    primary   = charmtone.Charple   // "brand" color
    secondary = charmtone.Dolly     // cursor, secondary accent
    tertiary  = charmtone.Bok       // editor prompt

    bgBase        = charmtone.Pepper    // app background
    bgBaseLighter = charmtone.BBQ       // content blocks
    bgSubtle      = charmtone.Charcoal  // borders
    bgOverlay     = charmtone.Iron      // dialog bg

    fgBase      = charmtone.Ash     // text
    fgMuted     = charmtone.Squid   // secondary text
    fgHalfMuted = charmtone.Smoke   // in-between
    fgSubtle    = charmtone.Oyster  // hints

    border      = charmtone.Charcoal  // unfocused borders
    borderFocus = charmtone.Charple   // focused borders

    error   = charmtone.Sriracha
    warning = charmtone.Zest
    info    = charmtone.Malibu
)
```

Every component references these semantic colors rather than hard-coding values. This makes theming possible by swapping the palette.

## The Styles Struct

**Source:** `internal/ui/styles/styles.go`

The `Styles` struct is a massive (~500 fields) centralized style registry. It's organized in nested groups:

```go
type Styles struct {
    // Base text
    Base, Muted, HalfMuted, Subtle lipgloss.Style

    // Tags
    TagBase, TagError, TagInfo lipgloss.Style

    // Semantic colors (as color.Color for components that need raw colors)
    Primary, Secondary, Tertiary color.Color
    BgBase, BgSubtle, BgOverlay  color.Color
    FgBase, FgMuted, FgSubtle    color.Color
    Error, Warning, Info         color.Color

    // Nested groups
    Header struct { Charm, Diagonals, Percentage, Keystroke, WorkingDir lipgloss.Style }
    Chat   struct { Message struct { UserFocused, AssistantFocused, Thinking... } }
    Tool   struct { IconPending, IconSuccess, NameNormal, ContentLine, Body... }
    Dialog struct { Title, View, Help... }
    Pills  struct { ... }

    // Component-specific styles
    TextInput textinput.Styles
    TextArea  textarea.Styles
    Help      help.Styles
    Diff      diffview.Style
    FilePicker filepicker.Styles

    // Markdown rendering config
    Markdown      ansi.StyleConfig
    PlainMarkdown ansi.StyleConfig
}
```

Styles are created once in `DefaultStyles()` and passed via `common.Common`:

```go
type Common struct {
    App    *app.App
    Styles *styles.Styles
}
```

Every component receives `*common.Common` at creation time.

## Icon System

**Source:** `internal/ui/styles/styles.go`

Crush uses Unicode characters as icons, defined as constants:

```go
const (
    CheckIcon   = "✓"
    SpinnerIcon = "⋯"
    LoadingIcon = "⟳"
    ModelIcon   = "◇"
    ArrowRightIcon = "→"

    // Tool status
    ToolPending = "●"
    ToolSuccess = "✓"
    ToolError   = "×"

    // Radio buttons
    RadioOn  = "◉"
    RadioOff = "○"

    // Borders
    BorderThin  = "│"
    BorderThick = "▌"

    // Separators
    SectionSeparator = "─"

    // Todos
    TodoCompletedIcon  = "✓"
    TodoPendingIcon    = "•"
    TodoInProgressIcon = "→"

    // Attachments
    ImageIcon = "■"
    TextIcon  = "≡"

    // Scrollbar
    ScrollbarThumb = "┃"
    ScrollbarTrack = "│"

    // LSP diagnostics
    LSPErrorIcon   = "E"
    LSPWarningIcon = "W"
    LSPInfoIcon    = "I"
    LSPHintIcon    = "H"
)
```

Icons are styled using the corresponding `lipgloss.Style` from the Styles struct. For example:

```go
s.Tool.IconPending = lipgloss.NewStyle().Foreground(blue)
s.Tool.IconSuccess = lipgloss.NewStyle().Foreground(green)
s.Tool.IconError   = lipgloss.NewStyle().Foreground(red)
```

## Typography Patterns

### Text Hierarchy

| Level | Style | Usage |
|-------|-------|-------|
| Primary | `Base` (fgBase/Ash) | Main content, user messages |
| Secondary | `Muted` (fgMuted/Squid) | Metadata, timestamps, secondary info |
| Tertiary | `HalfMuted` (fgHalfMuted/Smoke) | Less important metadata |
| Hint | `Subtle` (fgSubtle/Oyster) | Placeholders, keystroke hints |

### Message Rendering

User messages and assistant messages have distinct focused/blurred styles:

```go
Chat.Message.UserFocused     // highlight border on left
Chat.Message.UserBlurred     // muted border on left
Chat.Message.AssistantFocused
Chat.Message.AssistantBlurred
```

Tool calls get three states: focused, blurred, and compact (nested tools):

```go
Chat.Message.ToolCallFocused  // full detail
Chat.Message.ToolCallBlurred  // muted
Chat.Message.ToolCallCompact  // minimal, for nested agent tools
```

### Tags

Small colored labels for status indicators:

```go
TagBase  // default tag
TagError // red background
TagInfo  // blue/info background
Tool.ErrorTag    // "ERROR" label
Tool.NoteTag     // "NOTE" label (yellow bg)
Tool.AgentTaskTag // "TASK" label (blue bg)
```

### Section Headers

Used to separate logical sections within tool output:

```go
Chat.Message.SectionHeader  // styled divider text
Section.Title               // section title style
Section.Line                // horizontal line style
```

## Layout Constants

```go
// Global
const defaultMargin     = 2
const defaultListIndent = 2

// Compact mode breakpoints
const compactModeWidthBreakpoint  = 120
const compactModeHeightBreakpoint = 30

// Text
const maxTextWidth = 120  // max readable width for chat messages
const MessageLeftPaddingTotal = 2  // border + padding

// Dialogs
const defaultDialogMaxWidth = 70
const defaultDialogHeight   = 20

// Permissions dialog
const diffMaxWidth      = 180
const simpleMaxWidth    = 100
const splitModeMinWidth = 140

// Sidebar
const sidebarWidth = 30

// Editor
const editorHeight = 5

// Paste threshold
const pasteLinesThreshold = 10  // lines before treating paste as attachment
```

## Gradient Effects

**Source:** `internal/ui/styles/grad.go`

Dialog titles use color gradients rendered character-by-character:

```go
type RenderContext struct {
    TitleGradientFromColor color.Color  // defaults to Primary
    TitleGradientToColor   color.Color  // defaults to Secondary
}
```

The title text is rendered with a smooth gradient from the primary color (Charple) to the secondary color (Dolly), giving dialogs their distinctive polished look.

The logo also uses gradient colors:

```go
LogoTitleColorA  color.Color  // gradient start
LogoTitleColorB  color.Color  // gradient end
LogoCharmColor   color.Color  // "Charm" text
LogoVersionColor color.Color  // version number
```

This creates the signature Charm aesthetic - dark backgrounds with warm, saturated accent gradients.
