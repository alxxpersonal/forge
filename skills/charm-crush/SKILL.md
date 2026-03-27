---
name: charm-crush
description: Architecture patterns from Crush, charmbracelet's production agentic coding CLI built on bubbletea v2, lipgloss v2, bubbles v2, glamour v2, and ultraviolet. Use when building production bubbletea apps, composing charm TUI components at scale, designing agentic CLI tools, implementing streaming LLM UIs, or asking about crush internals. NOT for individual charm library basics.
argument-hint: "[pattern or component to look up]"
---

# Charm Crush - Production Agentic TUI Patterns

Crush is charmbracelet's agentic coding CLI - their answer to Claude Code, built entirely with their own TUI stack. This skill captures the architecture patterns, component composition strategies, and design decisions from a production app with 10+ composed components, real-time LLM streaming, dialog overlays, and a full agent loop.

Source: `github.com/charmbracelet/crush` (FSL-1.1-MIT)

## Architecture Overview

Crush follows a clear layered architecture:

```
main.go (cobra CLI)
  -> internal/app/app.go (wiring: DB, config, agents, LSP, MCP, events)
    -> internal/agent/ (LLM conversations, tool execution, streaming)
    -> internal/ui/ (bubbletea v2 TUI)
    -> internal/permission/ (tool approval via pubsub)
    -> internal/skills/ (Agent Skills open standard)
    -> internal/pubsub/ (generic typed broker for cross-component messaging)
    -> internal/session/ (SQLite persistence via sqlc)
```

### Key Dependencies (go.mod)

| Library | Version | Role |
|---------|---------|------|
| `charm.land/bubbletea/v2` | v2.0.2 | TUI framework |
| `charm.land/lipgloss/v2` | v2.0.2 | Terminal styling |
| `charm.land/bubbles/v2` | v2.0.0 | Reusable components (textarea, viewport, spinner, help) |
| `charm.land/glamour/v2` | v2.0.0 | Markdown rendering |
| `charm.land/fantasy` | v0.16.0 | LLM provider abstraction (Anthropic, OpenAI, Gemini, Bedrock, etc.) |
| `charmbracelet/ultraviolet` | - | Screen-buffer rendering system |
| `charm.land/catwalk` | v0.31.0 | Model registry + golden-file testing |
| `charmbracelet/x/exp/charmtone` | - | Color palette system |

## Decision Tree: How Crush Solves Common Problems

Use $ARGUMENTS to find the relevant pattern, or browse by category:

### TUI Architecture
- **"How do I structure a large bubbletea app?"** -> Read `references/tui-patterns.md` (Centralized Model pattern)
- **"How do I handle window resize?"** -> Read `references/tui-patterns.md` (Layout System)
- **"How do I compose 10+ components?"** -> Read `references/component-composition.md`
- **"How do I manage focus between panes?"** -> Read `references/tui-patterns.md` (Focus Management)
- **"How do I render streaming content?"** -> Read `references/tui-patterns.md` (Streaming Rendering)
- **"How do I do overlays/modals?"** -> Read `references/component-composition.md` (Dialog System)

### Agent Loop
- **"How does the LLM loop work?"** -> Read `references/agent-loop.md`
- **"How do I handle tool approval?"** -> Read `references/agent-loop.md` (Permission System)
- **"How do I stream LLM responses to the TUI?"** -> Read `references/agent-loop.md` (Streaming Bridge)

### Design & Styling
- **"How do I build a polished terminal UI?"** -> Read `references/design-system.md`
- **"What colors does charm use?"** -> Read `references/design-system.md` (Charmtone Palette)
- **"How do I structure styles for a large app?"** -> Read `references/design-system.md` (Styles Struct)

## Core Pattern: The Centralized Model

The single most important architectural decision in Crush. The `UI` struct in `internal/ui/model/ui.go` is the **sole bubbletea model**. Sub-components are NOT bubbletea models - they're stateful structs with imperative methods.

```
UI (the only tea.Model)
  |-- Chat (stateful struct, no Update method)
  |-- textarea (bubbles/v2 textarea.Model)
  |-- dialog.Overlay (stack of Dialog interfaces)
  |-- completions (non-standard Update returning bool)
  |-- attachments (non-standard Update)
  |-- status, header, pills (render methods on UI)
```

This means:
- All message routing happens in one giant `switch msg.(type)` in `UI.Update()`
- Focus state determines key routing (editor vs chat)
- Components expose methods like `HandleMouseDown()`, `ScrollBy()`, `SetMessages()` instead of `Update(tea.Msg)`
- Side effects return `tea.Cmd` from methods, not from Update

### Why This Works

Traditional Elm architecture (each component gets its own Update/View) breaks down at scale because:
1. Message routing becomes a maze of forwarding
2. Shared state requires complex message passing
3. Focus management needs a central coordinator anyway

Crush sidesteps this by making the parent the single source of truth for all state transitions.

## Rendering Pipeline: Ultraviolet Screen Buffer

Crush uses a hybrid rendering approach instead of pure string concatenation:

1. `View()` creates an `ultraviolet.ScreenBuffer` sized to terminal dimensions
2. Layout is computed as `image.Rectangle` regions via `ultraviolet/layout.SplitVertical/SplitHorizontal`
3. Components draw into sub-regions: `uv.NewStyledString(str).Draw(scr, rect)`
4. Dialogs draw last (overlay on top of everything)
5. `canvas.Render()` flattens to a string for bubbletea

This enables:
- Overlapping content (dialogs, completions popup)
- Precise cursor positioning across components
- Screen-based layout math instead of string width guessing

## Communication: Typed Pub/Sub

Agent and TUI run on separate goroutines. They communicate through a generic typed broker (`internal/pubsub`) that publishes events which bubbletea converts to `tea.Msg`. See `references/agent-loop.md` for the full pattern and event types.

## File Map

| What | Where | Read for |
|------|-------|----------|
| Main TUI model | `internal/ui/model/ui.go` | Centralized model, Update loop, layout |
| Chat component | `internal/ui/model/chat.go` | List wrapping, animation, mouse handling |
| Tool renderers | `internal/ui/chat/*.go` | Per-tool rendering (bash, file, search, etc.) |
| Dialog system | `internal/ui/dialog/` | Modal overlays, permissions, models picker |
| List component | `internal/ui/list/list.go` | Lazy-rendered scrollable list |
| Styles | `internal/ui/styles/styles.go` | Full design system |
| Agent loop | `internal/agent/agent.go` | LLM streaming, tool execution |
| Coordinator | `internal/agent/coordinator.go` | Multi-provider setup, model management |
| Tool definitions | `internal/agent/tools/*.go` | Tool implementations (.go + .md pairs) |
| Permission system | `internal/permission/permission.go` | Tool approval flow |
| Skills loader | `internal/skills/skills.go` | SKILL.md parsing and discovery |
| Pub/sub | `internal/pubsub/` | Typed event broker |
| App wiring | `internal/app/app.go` | Service initialization |
| Config | `internal/config/` | crush.json loading, provider config |
| UI architecture guide | `internal/ui/AGENTS.md` | Charm's own UI development instructions |

## Reference Files

For detailed patterns with code examples from the actual source:

- `references/tui-patterns.md` - Layout system, focus management, key handling, streaming, responsive design
- `references/agent-loop.md` - Agent loop, fantasy SDK, tool system, permission flow, pubsub bridge
- `references/design-system.md` - Charmtone colors, Styles struct, icon system, typography
- `references/component-composition.md` - Interface hierarchy, dialog stack, list composition, chat items
