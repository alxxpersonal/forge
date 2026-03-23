# Event Types Reference

Complete catalog of all NDJSON events emitted by `claude -p --output-format stream-json --verbose`.

## Top-Level Event Types

Every stdout line is a JSON object with a `type` field. These are the possible values:

| Type | When | Contains |
|------|------|----------|
| `system` | First event of a run | Session metadata, tools, model |
| `stream_event` | During streaming | Content deltas, tool call fragments |
| `assistant` | After streaming completes | Fully assembled message |
| `result` | Final event | Cost, usage, session ID, errors |
| `rate_limit_event` | When rate limited | Reset time, limit type |
| `permission_request` | When tool needs approval | Tool name, input, options |

## system (init)

The first event. Subtype is always `init`.

```json
{
  "type": "system",
  "subtype": "init",
  "cwd": "/Users/me/project",
  "session_id": "abc-def-123",
  "tools": ["Read", "Write", "Edit", "Bash", "Glob", "Grep", "LS", "Agent", "Task", "WebSearch", "WebFetch"],
  "mcp_servers": [{"name": "my-server", "status": "connected"}],
  "model": "claude-sonnet-4-20250514",
  "permissionMode": "default",
  "agents": [],
  "skills": ["skill-name"],
  "plugins": [],
  "claude_code_version": "2.1.63",
  "fast_mode_state": "disabled",
  "uuid": "event-uuid"
}
```

Key fields:
- `session_id` - store this for `--resume`
- `tools` - available tool names (varies by installation and MCP servers)
- `model` - the active model
- `mcp_servers` - connected MCP servers and their status
- `claude_code_version` - for compatibility checks

## stream_event

Wraps Anthropic API streaming events. Has a nested `event` object.

```json
{
  "type": "stream_event",
  "event": { ... },
  "session_id": "abc-def-123",
  "parent_tool_use_id": null,
  "uuid": "event-uuid"
}
```

The `parent_tool_use_id` is non-null when the event comes from a subagent (tool-within-tool execution).

### Sub-event: message_start

Signals beginning of a new assistant message.

```json
{
  "type": "stream_event",
  "event": {
    "type": "message_start",
    "message": {
      "model": "claude-sonnet-4-20250514",
      "id": "msg_abc123",
      "role": "assistant",
      "content": [],
      "stop_reason": null,
      "usage": {"input_tokens": 100, "output_tokens": 0}
    }
  }
}
```

### Sub-event: content_block_start (text)

A text content block begins.

```json
{
  "type": "stream_event",
  "event": {
    "type": "content_block_start",
    "index": 0,
    "content_block": {"type": "text", "text": ""}
  }
}
```

### Sub-event: content_block_start (tool_use)

A tool call begins. Extract the tool `name` and `id`.

```json
{
  "type": "stream_event",
  "event": {
    "type": "content_block_start",
    "index": 1,
    "content_block": {
      "type": "tool_use",
      "id": "toolu_abc123",
      "name": "Read",
      "input": {}
    }
  }
}
```

### Sub-event: content_block_delta (text)

Streaming text fragment. Append `delta.text` to your display.

```json
{
  "type": "stream_event",
  "event": {
    "type": "content_block_delta",
    "index": 0,
    "delta": {"type": "text_delta", "text": "Here is the "}
  }
}
```

### Sub-event: content_block_delta (tool input)

Streaming tool input JSON fragment. Accumulate `partial_json` to reconstruct the full tool input.

```json
{
  "type": "stream_event",
  "event": {
    "type": "content_block_delta",
    "index": 1,
    "delta": {"type": "input_json_delta", "partial_json": "{\"file_path\":\""}
  }
}
```

### Sub-event: content_block_stop

A content block (text or tool_use) is complete.

```json
{
  "type": "stream_event",
  "event": {
    "type": "content_block_stop",
    "index": 0
  }
}
```

### Sub-event: message_delta

Message-level update with stop reason and usage.

```json
{
  "type": "stream_event",
  "event": {
    "type": "message_delta",
    "delta": {"stop_reason": "end_turn"},
    "usage": {"input_tokens": 100, "output_tokens": 250}
  }
}
```

Stop reasons: `"end_turn"` (normal), `"tool_use"` (needs to call a tool), `"max_tokens"` (hit limit).

### Sub-event: message_stop

Message streaming is complete.

```json
{
  "type": "stream_event",
  "event": {"type": "message_stop"}
}
```

## assistant

Assembled message after streaming completes. Contains all content blocks with their full content.

```json
{
  "type": "assistant",
  "message": {
    "model": "claude-sonnet-4-20250514",
    "id": "msg_abc123",
    "role": "assistant",
    "content": [
      {"type": "text", "text": "I'll read that file for you."},
      {"type": "tool_use", "id": "toolu_abc123", "name": "Read", "input": {"file_path": "/tmp/test.txt"}}
    ],
    "stop_reason": "tool_use",
    "usage": {"input_tokens": 100, "output_tokens": 50}
  },
  "parent_tool_use_id": null,
  "session_id": "abc-def-123",
  "uuid": "event-uuid"
}
```

Use this event to get complete tool inputs (instead of reconstructing from `input_json_delta` fragments).

## result

Final event of a run. Always present, even on errors.

### Success

```json
{
  "type": "result",
  "subtype": "success",
  "is_error": false,
  "duration_ms": 15234,
  "num_turns": 3,
  "result": "The file contains a list of...",
  "total_cost_usd": 0.0034,
  "session_id": "abc-def-123",
  "usage": {
    "input_tokens": 5000,
    "output_tokens": 1200,
    "cache_read_input_tokens": 3000,
    "cache_creation_input_tokens": 500
  },
  "permission_denials": [],
  "uuid": "event-uuid"
}
```

### Error

```json
{
  "type": "result",
  "subtype": "error",
  "is_error": true,
  "result": "Error: rate limit exceeded",
  "session_id": "abc-def-123",
  "duration_ms": 1200,
  "num_turns": 0,
  "total_cost_usd": 0,
  "usage": {},
  "permission_denials": [],
  "uuid": "event-uuid"
}
```

Key fields:
- `total_cost_usd` - cost of this run
- `session_id` - store for `--resume`
- `permission_denials` - tools denied during the run. Runtime sends `{tool_name, tool_use_id}` objects; types.ts declares it as `string[]`. Treat as `{tool_name: string, tool_use_id: string}[]` in practice.
- `num_turns` - number of agentic turns taken

## rate_limit_event

Emitted when the subscription rate limit is hit.

```json
{
  "type": "rate_limit_event",
  "rate_limit_info": {
    "status": "rate_limited",
    "resetsAt": 1700000000,
    "rateLimitType": "model"
  },
  "session_id": "abc-def-123",
  "uuid": "event-uuid"
}
```

`resetsAt` is a Unix timestamp. Show the user a countdown or retry after that time.

## permission_request

Emitted when Claude wants to use a tool that requires approval (stdin-based permission flow, not HTTP hooks).

```json
{
  "type": "permission_request",
  "tool": {
    "name": "Bash",
    "description": "Execute a bash command",
    "input": {"command": "npm install express"}
  },
  "question_id": "perm-abc-123",
  "options": [
    {"id": "allow", "label": "Allow Once", "kind": "allow"},
    {"id": "allow-session", "label": "Allow for Session", "kind": "allow"},
    {"id": "deny", "label": "Deny", "kind": "deny"}
  ],
  "session_id": "abc-def-123",
  "uuid": "event-uuid"
}
```

Respond via stdin with:
```json
{"type":"permission_response","question_id":"perm-abc-123","option_id":"allow"}
```

## Event Ordering Guarantees

1. `system` (init) is always the first event
2. `result` is always the last event
3. `stream_event` events arrive in order within a content block
4. `content_block_start` always precedes its corresponding deltas and stop
5. `assistant` arrives after all stream events for that message
6. `permission_request` can arrive at any point between init and result
7. `rate_limit_event` can arrive at any point
