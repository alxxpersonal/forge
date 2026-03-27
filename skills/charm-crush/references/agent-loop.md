# Agent Loop Patterns from Crush

How Crush orchestrates LLM conversations, tool execution, streaming, and permission handling.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [The Agent Loop](#the-agent-loop)
3. [Fantasy SDK Integration](#fantasy-sdk-integration)
4. [Streaming Bridge to TUI](#streaming-bridge-to-tui)
5. [Tool System](#tool-system)
6. [Permission System](#permission-system)
7. [Message Queue](#message-queue)
8. [Auto-Summarization](#auto-summarization)
9. [Coordinator Pattern](#coordinator-pattern)

## Architecture Overview

```
User types prompt
  -> UI.sendMessage() creates a tea.Cmd
    -> AgentCoordinator.Run(ctx, sessionID, prompt)
      -> SessionAgent.Run(ctx, call)
        -> fantasy.Agent.Stream(ctx, streamCall)
          -> LLM responds with text/tool calls
            -> Callbacks persist to DB via message.Service
              -> message.Service publishes via pubsub.Broker
                -> App.Events() channel delivers to bubbletea
                  -> UI.Update() receives pubsub.Event[message.Message]
                    -> Chat re-renders
```

## The Agent Loop

**Source:** `internal/agent/agent.go` `Run()` method

The core loop:

1. **Queue check** - if session is busy, queue the prompt and return immediately
2. **Prepare** - copy tools, model, system prompt (thread-safe via `csync.Value`)
3. **Create user message** - persist to DB, triggers pubsub event
4. **Generate title** - async goroutine on first message
5. **Stream** - call `fantasy.Agent.Stream()` with callbacks
6. **Handle result** - update session usage, check for summarization, process queue

```go
func (a *sessionAgent) Run(ctx context.Context, call SessionAgentCall) (*fantasy.AgentResult, error) {
    // Queue if busy
    if a.IsSessionBusy(call.SessionID) {
        a.messageQueue.Set(call.SessionID, append(existing, call))
        return nil, nil
    }

    // Thread-safe copies
    agentTools := a.tools.Copy()
    largeModel := a.largeModel.Get()
    systemPrompt := a.systemPrompt.Get()

    // Add MCP server instructions to system prompt
    for _, server := range mcp.GetStates() {
        if server.State == mcp.StateConnected {
            instructions.WriteString(server.Client.InitializeResult().Instructions)
        }
    }

    // Create fantasy agent
    agent := fantasy.NewAgent(
        largeModel.Model,
        fantasy.WithSystemPrompt(systemPrompt),
        fantasy.WithTools(agentTools...),
    )

    // Get history, create user message, then stream
    history, files := a.preparePrompt(msgs, call.Attachments...)
    result, err := agent.Stream(ctx, fantasy.AgentStreamCall{...})
}
```

## Fantasy SDK Integration

**Source:** `internal/agent/agent.go`

Fantasy (`charm.land/fantasy`) is Charm's multi-provider LLM abstraction. Crush uses the agent streaming API with rich callbacks:

```go
result, err := agent.Stream(genCtx, fantasy.AgentStreamCall{
    Prompt:          promptText,
    Messages:        history,
    Files:           files,
    ProviderOptions: call.ProviderOptions,
    MaxOutputTokens: &call.MaxOutputTokens,

    PrepareStep: func(ctx context.Context, opts fantasy.PrepareStepFunctionOptions) (context.Context, fantasy.PrepareStepResult, error) {
        // Called before each LLM call in the agentic loop
        // Refresh tools (MCP might have changed)
        prepared.Tools = a.tools.Copy()
        // Drain queued prompts and inject them
        for _, queued := range queuedCalls {
            prepared.Messages = append(prepared.Messages, userMessage.ToAIMessage()...)
        }
        // Create assistant message placeholder in DB
        assistantMsg, _ := a.messages.Create(ctx, sessionID, ...)
        currentAssistant = &assistantMsg
        return ctx, prepared, nil
    },

    OnReasoningStart: func(id string, reasoning fantasy.ReasoningContent) error {
        currentAssistant.AppendReasoningContent(reasoning.Text)
        return a.messages.Update(ctx, *currentAssistant)
    },
    OnReasoningDelta: func(id string, text string) error {
        currentAssistant.AppendReasoningContent(text)
        return a.messages.Update(ctx, *currentAssistant)
    },
    OnTextDelta: func(id string, text string) error {
        currentAssistant.AppendContent(text)
        return a.messages.Update(ctx, *currentAssistant)
    },
    OnToolInputStart: func(id string, toolName string) error {
        currentAssistant.AddToolCall(message.ToolCall{ID: id, Name: toolName})
        return a.messages.Update(ctx, *currentAssistant)
    },
    OnToolCall: func(tc fantasy.ToolCallContent) error {
        currentAssistant.AddToolCall(message.ToolCall{ID: tc.ToolCallID, Name: tc.ToolName, Input: tc.Input, Finished: true})
        return a.messages.Update(ctx, *currentAssistant)
    },
    OnToolResult: func(result fantasy.ToolResultContent) error {
        a.messages.Create(ctx, sessionID, message.CreateMessageParams{Role: message.Tool, Parts: [...]})
        return nil
    },
    OnStepFinish: func(stepResult fantasy.StepResult) error {
        // Update token usage on session
        a.updateSessionUsage(largeModel, &session, stepResult.Usage, ...)
        a.sessions.Save(ctx, session)
        return nil
    },

    StopWhen: []fantasy.StopCondition{
        // Stop when context window is nearly full
        func(_ []fantasy.StepResult) bool {
            remaining := contextWindow - tokensUsed
            return remaining <= threshold && !disableAutoSummarize
        },
        // Stop on repeated tool calls (loop detection)
        func(steps []fantasy.StepResult) bool {
            return hasRepeatedToolCalls(steps, windowSize, maxRepeats)
        },
    },
})
```

Key insight: `PrepareStep` runs before EACH step in the agentic loop (not just the first). This lets Crush:
- Inject queued user messages mid-conversation
- Refresh MCP tools dynamically
- Create a fresh assistant message for each step
- Apply Anthropic cache control to the right message positions

## Streaming Bridge to TUI

The flow from agent goroutine to TUI:

1. **Agent callback** (`OnTextDelta`, etc.) calls `a.messages.Update(ctx, msg)`
2. **message.Service** persists to SQLite, then publishes: `broker.Publish(pubsub.UpdatedEvent, msg)`
3. **App** has a goroutine converting pubsub channels to `tea.Msg` via `app.Events()` channel
4. **Bubbletea** reads from `app.Events()` and dispatches to `UI.Update()`
5. **UI** receives `pubsub.Event[message.Message]` and updates the chat item

```go
// In UI.Update():
case pubsub.Event[message.Message]:
    if msg.Payload.SessionID != m.session.ID {
        // Handle child session (agent tool)
        break
    }
    switch msg.Type {
    case pubsub.CreatedEvent:
        cmds = append(cmds, m.appendSessionMessage(msg.Payload))
    case pubsub.UpdatedEvent:
        cmds = append(cmds, m.updateSessionMessage(msg.Payload))
    }
```

The chat item uses cached rendering - it only re-renders when the underlying message data changes.

## Tool System

**Source:** `internal/agent/tools/`

Tools are self-documenting pairs: a `.go` implementation file and a `.md` description file in the same directory.

Built-in tools: bash, edit, multiedit, view, write, grep, glob, ls, diagnostics, references, fetch, download, lsp_restart, sourcegraph, job_output, job_kill, list_mcp_resources, todos, agent, agentic_fetch.

MCP tools are dynamically loaded and prefixed with `mcp_`.

Each tool implements `fantasy.AgentTool` which provides a JSON schema for the LLM and an execution function.

Tool results flow back through `OnToolResult` callback -> `message.Service.Create()` -> pubsub -> TUI.

## Permission System

**Source:** `internal/permission/permission.go`

The permission service mediates tool execution:

```go
type Service interface {
    Request(ctx context.Context, opts CreatePermissionRequest) (bool, error)
    Grant(permission PermissionRequest)
    GrantPersistent(permission PermissionRequest)
    Deny(permission PermissionRequest)
    AutoApproveSession(sessionID string)
    SetSkipRequests(skip bool)  // yolo mode
}
```

When a tool needs permission:
1. Tool calls `permissions.Request()` which blocks
2. Permission service publishes `pubsub.Event[permission.PermissionRequest]`
3. TUI receives event, opens `dialog.Permissions`
4. User chooses: Allow / Allow for session / Deny
5. TUI calls `permissions.Grant()` or `permissions.Deny()`
6. The blocked `Request()` call returns, tool continues or errors

Allow-lists can be configured in `crush.json` to skip prompting for specific tools.

Yolo mode (`--yolo` flag) sets `SkipRequests(true)` which auto-approves everything.

## Message Queue

**Source:** `internal/agent/agent.go`

If the user sends a new prompt while the agent is busy:

```go
if a.IsSessionBusy(call.SessionID) {
    existing, _ := a.messageQueue.Get(call.SessionID)
    existing = append(existing, call)
    a.messageQueue.Set(call.SessionID, existing)
    return nil, nil  // queued, not executed yet
}
```

Queued messages are drained in `PrepareStep` (called before each LLM step):

```go
PrepareStep: func(ctx context.Context, opts ...) (...) {
    queuedCalls, _ := a.messageQueue.Get(call.SessionID)
    a.messageQueue.Del(call.SessionID)
    for _, queued := range queuedCalls {
        userMessage, _ := a.createUserMessage(ctx, queued)
        prepared.Messages = append(prepared.Messages, userMessage.ToAIMessage()...)
    }
}
```

This means queued prompts are injected into the conversation at the next natural break point (between LLM steps).

## Auto-Summarization

When the context window is nearly full, Crush auto-summarizes:

```go
const (
    largeContextWindowThreshold = 200_000
    largeContextWindowBuffer    = 20_000
    smallContextWindowRatio     = 0.2
)

// In StopWhen condition:
remaining := contextWindow - tokensUsed
if cw > largeContextWindowThreshold {
    threshold = largeContextWindowBuffer  // 20K buffer for large models
} else {
    threshold = int64(float64(cw) * smallContextWindowRatio)  // 20% for small models
}
if remaining <= threshold && !disableAutoSummarize {
    shouldSummarize = true
    return true  // stop the loop
}
```

After the loop stops, if `shouldSummarize` is true, the coordinator triggers summarization using the small model.

## Coordinator Pattern

**Source:** `internal/agent/coordinator.go`

The Coordinator manages the lifecycle:

```go
type Coordinator interface {
    Run(ctx context.Context, sessionID, prompt string, attachments ...message.Attachment) (*fantasy.AgentResult, error)
    Cancel(sessionID string)
    IsSessionBusy(sessionID string) bool
    QueuedPrompts(sessionID string) int
    UpdateModels(ctx context.Context) error
    Model() Model
}
```

It handles:
- **Multi-provider setup** - creates `fantasy.LanguageModel` from config for each provider type (Anthropic, OpenAI, Google, Bedrock, Azure, OpenRouter, Vercel, etc.)
- **Model switching** - `UpdateModels()` reconfigures agents when user changes model mid-session
- **Tool registration** - collects built-in tools + MCP tools, passes to session agent
- **System prompt assembly** - loads Go templates from `internal/agent/templates/`, injects runtime data (working dir, OS, skills, context files)

The coordinator owns a map of named agents (currently "coder" and "task") and delegates to the current agent.

Thread safety throughout uses `internal/csync` which provides `Value[T]`, `Slice[T]`, and `Map[K,V]` - simple wrappers around values with mutex protection.
