---
name: claude-headless
description: Build custom UIs on top of Claude Code's headless mode. Covers spawning, NDJSON protocol, permission hooks, and session management. Use when building a desktop app, TUI, web UI, or any custom interface that wraps Claude Code as a subprocess.
---

# Claude Headless

Build custom UIs and applications on top of Claude Code by running it as a headless subprocess. Claude Code exposes a bidirectional NDJSON protocol over stdin/stdout that gives you full control over prompts, streaming responses, tool approvals, and session continuity.

For complete event type catalog, read `references/event-types.md`.
For working code examples in Go and TypeScript, read `references/code-examples.md`.

## Architecture Overview

```
Your App (any language/framework)
  |
  ├── spawn: claude -p --input-format stream-json --output-format stream-json --verbose
  |
  ├── stdin  → write NDJSON messages (prompts, permission responses)
  ├── stdout ← read NDJSON events (text chunks, tool calls, results)
  └── stderr ← diagnostic logs (not structured, for debugging only)
```

Claude Code runs as a child process. You write JSON lines to stdin, read JSON lines from stdout. No API key needed - it uses the existing OAuth login from `claude login`. Same subscription limits as the interactive CLI.

## Spawning Claude in Headless Mode

### Required Flags

```
claude -p \
  --input-format stream-json \
  --output-format stream-json \
  --verbose \
  --include-partial-messages
```

| Flag | Purpose |
|------|---------|
| `-p` | Print mode - non-interactive, reads from stdin |
| `--input-format stream-json` | Accept NDJSON on stdin (bidirectional) |
| `--output-format stream-json` | Emit NDJSON on stdout |
| `--verbose` | Include streaming events (content deltas, tool call updates) |
| `--include-partial-messages` | Emit partial content block events during streaming |

### Optional Flags

| Flag | Purpose |
|------|---------|
| `--resume <session-id>` | Continue an existing session |
| `--model <model>` | Choose model (e.g. `claude-sonnet-4-20250514`) |
| `--permission-mode default` | Use default permission behavior |
| `--allowedTools <tools>` | Comma-separated list of pre-approved tools |
| `--settings <path>` | Path to settings JSON with hook config |
| `--system-prompt <text>` | Replace the default system prompt entirely |
| `--append-system-prompt <text>` | Append to the default system prompt (additive, can coexist with `--system-prompt`) |
| `--max-turns <n>` | Limit number of agentic turns |
| `--max-budget-usd <n>` | Set spending cap per run |
| `--add-dir <path>` | Add extra directories to context (repeatable) |

### Stdio Config

Spawn with all three pipes: `stdin`, `stdout`, `stderr` as `pipe`. Stdin must stay open for follow-up messages. Set stdout encoding to UTF-8.

### Environment

Delete the `CLAUDECODE` env var if it exists in your process - it interferes with subprocess spawning. Ensure the `claude` binary is on `PATH` or use an absolute path.

## NDJSON Input Protocol (stdin)

### Sending a Prompt

Write a single JSON line to stdin:

```json
{"type":"user","message":{"role":"user","content":[{"type":"text","text":"Your prompt here"}]}}
```

**Important:** append `\n` after each JSON object. Stdin stays open - do not close it after writing. The process accepts multiple messages over its lifetime.

### Content Types

Text message:
```json
{
  "type": "user",
  "message": {
    "role": "user",
    "content": [{"type": "text", "text": "Explain this code"}]
  }
}
```

The content array follows the Anthropic messages API format. Each element has a `type` field.

### Permission Response

When Claude requests tool approval and you're using the stdin-based permission flow (not HTTP hooks):

```json
{
  "type": "permission_response",
  "question_id": "the-question-id-from-the-event",
  "option_id": "allow"
}
```

Valid option IDs: `allow`, `allow-session`, `deny`. The `question_id` comes from the `permission_request` event.

### Follow-up Messages

Write additional user messages to stdin at any time. Claude processes them sequentially. After receiving a `result` event, close stdin to trigger a clean process exit.

## NDJSON Output Protocol (stdout)

Every line on stdout is a JSON object with a `type` field. Events arrive in this lifecycle order:

```
system (init) -> stream_event* -> assistant -> result
                      ^                |
                      |   (tool loop)  |
                      +----------------+
```

### Event Lifecycle

1. **`system`** (subtype `init`) - first event, contains session metadata
2. **`stream_event`** - streaming content: text deltas, tool call starts/updates/stops
3. **`assistant`** - assembled message with all content blocks (after streaming completes)
4. **`result`** - final event, contains cost/usage/session_id

Between steps 2-3, tool calls may trigger `permission_request` events (if using stdin-based permissions) or HTTP hook requests (if using a hook server).

Rate limits produce `rate_limit_event` at any point.

### Parsing Strategy

Buffer incoming stdout data. Split on `\n`. Parse each non-empty line as JSON. Handle incomplete lines by keeping a buffer of the trailing fragment.

```
buffer += chunk
lines = buffer.split('\n')
buffer = lines.pop()  // keep incomplete trailing line
for each line in lines:
    if line.trim() is empty: skip
    event = JSON.parse(line.trim())
    handle(event)
```

On stream end, flush the buffer (parse any remaining content).

### Detecting Completion

The `result` event signals the run is complete. After receiving it, close stdin to trigger process exit. The process stays alive in `stream-json` input mode waiting for more input - closing stdin is what triggers the clean shutdown.

```json
{"type":"result","subtype":"success","result":"...","session_id":"...","total_cost_usd":0.003,...}
```

Check `is_error` and `subtype` on the result event. If `is_error` is true or `subtype` is `"error"`, the run failed.

## Permission Hook Server

For production UIs, use an HTTP-based PreToolUse hook instead of stdin-based permission flow. This gives you a proper request/response cycle with timeouts and scoped approvals.

### How It Works

1. Start a local HTTP server before spawning Claude
2. Generate a per-run settings JSON file pointing Claude to your hook URL
3. Pass the settings file via `--settings <path>`
4. When Claude wants to use a tool, it POSTs to your hook URL
5. Your server returns allow/deny
6. Claude proceeds or skips the tool

### Settings File Format

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "^(Bash|Edit|Write|MultiEdit|mcp__.*)$",
        "hooks": [
          {
            "type": "http",
            "url": "http://127.0.0.1:19836/hook/pre-tool-use/<app-secret>/<run-token>",
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

The `matcher` is a regex against tool names. Only matched tools trigger the hook - unmatched tools need `--allowedTools` to run.

**Security pattern:** embed a per-launch app secret and per-run token in the URL path. Validate both on every request. This prevents local spoofing and cross-run confusion.

**File lifecycle:** write the settings file to a temp directory with restrictive permissions (0o600), clean it up when the run ends.

### Hook Request (POST body from Claude)

```json
{
  "session_id": "abc-123",
  "hook_event_name": "PreToolUse",
  "tool_name": "Bash",
  "tool_input": {"command": "rm -rf /tmp/test"},
  "tool_use_id": "toolu_xyz",
  "cwd": "/Users/me/project",
  "permission_mode": "default",
  "transcript_path": "/path/to/transcript.jsonl"
}
```

### Hook Response (your server returns)

Allow:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "Approved by user"
  }
}
```

Deny:
```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "User denied"
  }
}
```

### Tool Safety Tiers

Split tools into safe (auto-approve) and dangerous (require approval):

**Safe tools** (pass via `--allowedTools`):
`Read`, `Glob`, `Grep`, `LS`, `TodoRead`, `TodoWrite`, `Agent`, `Task`, `TaskOutput`, `Notebook`, `WebSearch`, `WebFetch`

**Dangerous tools** (route through hook server):
`Bash`, `Edit`, `Write`, `MultiEdit`, and any `mcp__*` tools

You can additionally auto-approve read-only Bash commands by inspecting `tool_input.command` before prompting the user.

### Timeout Behavior

The hook has a `timeout` field in seconds (300 = 5 minutes). If your server doesn't respond in time, Claude treats it as a denial. Always deny-by-default on every failure path (parse errors, invalid tokens, timeouts).

### Scoped Approvals

Track user decisions to reduce permission fatigue:

- **Session-scoped:** user approves "Edit" once, auto-allow for the rest of the session. Key: `session:<id>:tool:<name>`
- **Domain-scoped:** for WebFetch, approve a domain once. Key: `session:<id>:webfetch:<domain>`
- **Per-command:** Bash commands are too diverse for blanket approval - review each individually

## Session Management

### Session IDs

The `system` init event returns a `session_id`. Store it. Pass it back via `--resume <session-id>` on subsequent runs to continue the conversation.

### Multiple Concurrent Sessions

Each session is a separate `claude -p` child process. You can run many in parallel. Track each by a unique request ID mapped to its process handle.

### Session Lifecycle

```
idle -> connecting -> running -> completed
                        |           |
                        v           v
                      failed      idle (new prompt)
                        |
                        v
                       dead (unrecoverable)
```

- **connecting:** process spawned, waiting for `system` init event
- **running:** init received, streaming in progress
- **completed:** `result` event received with `subtype: "success"`
- **failed:** non-zero exit, SIGINT/SIGKILL, or error result
- **dead:** process error (binary not found, spawn failure)

### Tab Pattern

For multi-tab UIs, maintain a registry mapping tab IDs to session state:

```
Tab Registry:
  tabId -> {
    claudeSessionId: string | null,
    status: TabStatus,
    activeRequestId: string | null,
    promptCount: number,
  }
```

Queue prompts if a tab already has an active run. Process the queue when the current run completes.

### Cancellation

Send SIGINT to the child process. If it hasn't exited after 5 seconds, send SIGKILL.

## Model Routing

Pass `--model <model-id>` when spawning. To switch models mid-conversation, start a new process with `--resume <session-id> --model <new-model>`. The session context carries over.

## Common Patterns

### Streaming Text to UI

Listen for `stream_event` events where the inner event type is `content_block_delta` with `delta.type === "text_delta"`. Append `delta.text` to your display buffer.

### Tracking Tool Calls

1. `content_block_start` with `content_block.type === "tool_use"` - tool call begins, extract `name` and `id`
2. `content_block_delta` with `delta.type === "input_json_delta"` - partial tool input JSON arrives
3. `content_block_stop` - tool call input is complete

The `assistant` event arrives after all content blocks, containing the fully assembled message with all tool calls and their complete inputs.

### Idempotent Request IDs

Use unique request IDs for each prompt submission. If a duplicate ID is submitted while inflight, return the existing promise instead of spawning a new process. This prevents double-submissions from UI race conditions.

### Request Queuing

If a tab already has an active run, queue the new request. Process the queue (FIFO) when the current run's exit event fires. Set a max queue depth (32 is reasonable) and reject with backpressure when full.

### Warm-up Init

To pre-populate session metadata (available tools, model, MCP servers) without showing a visible message, fire a minimal prompt like `"hi"` with `--max-turns 1` at tab creation. Suppress all events except the `session_init` from this request.

## What NOT to Do

1. **Don't close stdin after the first prompt.** The process stays alive for follow-up messages. Only close stdin after receiving the `result` event to trigger clean exit.

2. **Don't parse stderr as structured data.** It contains diagnostic logs, not NDJSON. Read it for debugging only.

3. **Don't use `--output-format json`** (non-streaming). You get a single JSON blob at the end with no intermediate events. Always use `stream-json`.

4. **Don't skip `--verbose` and `--include-partial-messages`.** Without these, you miss streaming content deltas and tool call updates. Your UI will appear frozen until the full response completes.

5. **Don't auto-approve all tools without a hook server.** If you pass every tool in `--allowedTools`, Claude will execute destructive operations (file writes, shell commands) without user consent.

6. **Don't ignore the `CLAUDECODE` env var.** If your app is itself running inside Claude Code, this var will be set and can interfere with subprocess spawning. Delete it from the child's environment.

7. **Don't forget request ID idempotency.** UI double-clicks and network retries can cause duplicate submissions. Always check if a request ID is already inflight or queued before spawning.
