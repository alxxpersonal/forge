# Code Examples

Working code for spawning Claude headless, parsing NDJSON, and handling permissions. TypeScript and Go.

## TypeScript: Spawn and Parse

### NDJSON Stream Parser

Buffers incoming data and emits complete JSON objects line by line.

```typescript
import { Readable } from 'stream'
import { EventEmitter } from 'events'

interface ClaudeEvent {
  type: string
  [key: string]: unknown
}

class StreamParser extends EventEmitter {
  private buffer = ''

  feed(chunk: string): void {
    this.buffer += chunk
    const lines = this.buffer.split('\n')
    this.buffer = lines.pop() || '' // keep incomplete trailing line

    for (const line of lines) {
      const trimmed = line.trim()
      if (!trimmed) continue
      try {
        const parsed = JSON.parse(trimmed) as ClaudeEvent
        this.emit('event', parsed)
      } catch {
        this.emit('parse-error', trimmed)
      }
    }
  }

  flush(): void {
    const trimmed = this.buffer.trim()
    if (trimmed) {
      try {
        this.emit('event', JSON.parse(trimmed))
      } catch {
        this.emit('parse-error', trimmed)
      }
    }
    this.buffer = ''
  }

  static fromStream(stream: Readable): StreamParser {
    const parser = new StreamParser()
    stream.setEncoding('utf-8')
    stream.on('data', (chunk: string) => parser.feed(chunk))
    stream.on('end', () => parser.flush())
    return parser
  }
}
```

### Spawning Claude

```typescript
import { spawn, ChildProcess } from 'child_process'

interface RunOptions {
  prompt: string
  cwd: string
  sessionId?: string
  model?: string
  hookSettingsPath?: string
  allowedTools?: string[]
  maxTurns?: number
}

function startRun(options: RunOptions): ChildProcess {
  const args: string[] = [
    '-p',
    '--input-format', 'stream-json',
    '--output-format', 'stream-json',
    '--verbose',
    '--include-partial-messages',
    '--permission-mode', 'default',
  ]

  if (options.sessionId) {
    args.push('--resume', options.sessionId)
  }
  if (options.model) {
    args.push('--model', options.model)
  }
  if (options.hookSettingsPath) {
    args.push('--settings', options.hookSettingsPath)
  }
  if (options.allowedTools?.length) {
    args.push('--allowedTools', options.allowedTools.join(','))
  }
  if (options.maxTurns) {
    args.push('--max-turns', String(options.maxTurns))
  }

  // Remove CLAUDECODE env var to avoid subprocess conflicts
  const env = { ...process.env }
  delete env.CLAUDECODE

  const child = spawn('claude', args, {
    stdio: ['pipe', 'pipe', 'pipe'],
    cwd: options.cwd,
    env,
  })

  // Write initial prompt
  const userMessage = JSON.stringify({
    type: 'user',
    message: {
      role: 'user',
      content: [{ type: 'text', text: options.prompt }],
    },
  })
  child.stdin!.write(userMessage + '\n')

  return child
}
```

### Full Lifecycle Example

```typescript
async function runPrompt(prompt: string, cwd: string): Promise<string> {
  const child = startRun({ prompt, cwd })

  const parser = StreamParser.fromStream(child.stdout!)
  let sessionId: string | null = null
  let resultText = ''
  const textChunks: string[] = []

  return new Promise((resolve, reject) => {
    parser.on('event', (event: ClaudeEvent) => {
      switch (event.type) {
        case 'system':
          if ((event as any).subtype === 'init') {
            sessionId = (event as any).session_id
          }
          break

        case 'stream_event': {
          const sub = (event as any).event
          if (sub?.type === 'content_block_delta' && sub.delta?.type === 'text_delta') {
            textChunks.push(sub.delta.text)
            process.stdout.write(sub.delta.text) // stream to terminal
          }
          break
        }

        case 'result':
          resultText = (event as any).result || textChunks.join('')
          // Close stdin to trigger clean exit
          try { child.stdin?.end() } catch {}
          break
      }
    })

    child.on('close', (code) => {
      if (code === 0) {
        resolve(resultText)
      } else {
        reject(new Error(`claude exited with code ${code}`))
      }
    })

    child.on('error', reject)
  })
}
```

## TypeScript: Permission Hook Server

### Minimal HTTP Hook Server

```typescript
import { createServer, IncomingMessage, ServerResponse } from 'http'
import { writeFileSync, mkdirSync, unlinkSync } from 'fs'
import { tmpdir } from 'os'
import { join } from 'path'
import { randomUUID } from 'crypto'

const PORT = 19836
const APP_SECRET = randomUUID()

interface HookRequest {
  session_id: string
  hook_event_name: string
  tool_name: string
  tool_input: Record<string, unknown>
  tool_use_id: string
  cwd: string
}

// Tools that need user approval
const DANGEROUS_TOOLS = new Set(['Bash', 'Edit', 'Write', 'MultiEdit'])

// Tools to auto-approve via --allowedTools
const SAFE_TOOLS = [
  'Read', 'Glob', 'Grep', 'LS',
  'TodoRead', 'TodoWrite',
  'Agent', 'Task', 'TaskOutput',
  'Notebook', 'WebSearch', 'WebFetch',
]

function allowResponse(reason: string) {
  return {
    hookSpecificOutput: {
      hookEventName: 'PreToolUse',
      permissionDecision: 'allow',
      permissionDecisionReason: reason,
    },
  }
}

function denyResponse(reason: string) {
  return {
    hookSpecificOutput: {
      hookEventName: 'PreToolUse',
      permissionDecision: 'deny',
      permissionDecisionReason: reason,
    },
  }
}

// Your approval callback - wire this to your UI
type ApprovalCallback = (tool: string, input: Record<string, unknown>) => Promise<boolean>

function startHookServer(onApproval: ApprovalCallback) {
  const server = createServer(async (req: IncomingMessage, res: ServerResponse) => {
    if (req.method !== 'POST') {
      res.writeHead(404)
      res.end(JSON.stringify(denyResponse('Not found')))
      return
    }

    // Validate URL path: /hook/pre-tool-use/<secret>/<token>
    const segments = (req.url || '').split('/').filter(Boolean)
    if (segments.length < 3 || segments[2] !== APP_SECRET) {
      res.writeHead(403)
      res.end(JSON.stringify(denyResponse('Invalid credentials')))
      return
    }

    // Read body
    let body = ''
    for await (const chunk of req) body += chunk

    const toolReq: HookRequest = JSON.parse(body)

    // Auto-approve if not a dangerous tool (belt-and-suspenders with matcher)
    if (!DANGEROUS_TOOLS.has(toolReq.tool_name)) {
      res.writeHead(200, { 'Content-Type': 'application/json' })
      res.end(JSON.stringify(allowResponse('Safe tool')))
      return
    }

    // Ask user via your UI
    const approved = await onApproval(toolReq.tool_name, toolReq.tool_input)

    res.writeHead(200, { 'Content-Type': 'application/json' })
    res.end(JSON.stringify(
      approved
        ? allowResponse('User approved')
        : denyResponse('User denied')
    ))
  })

  server.listen(PORT, '127.0.0.1')
  return server
}

// Generate settings file for a run
function generateSettingsFile(runToken: string): string {
  const matcher = '^(Bash|Edit|Write|MultiEdit|mcp__.*)$'
  const settings = {
    hooks: {
      PreToolUse: [{
        matcher,
        hooks: [{
          type: 'http',
          url: `http://127.0.0.1:${PORT}/hook/pre-tool-use/${APP_SECRET}/${runToken}`,
          timeout: 300,
        }],
      }],
    },
  }

  const dir = join(tmpdir(), 'my-app-hooks')
  mkdirSync(dir, { recursive: true, mode: 0o700 })
  const filePath = join(dir, `hook-${runToken}.json`)
  writeFileSync(filePath, JSON.stringify(settings), { mode: 0o600 })
  return filePath
}
```

## Go: Spawn and Parse

### NDJSON Parser

```go
package claude

import (
    "bufio"
    "encoding/json"
    "io"
)

// Event represents any NDJSON event from Claude's stdout.
type Event struct {
    Type      string          `json:"type"`
    Subtype   string          `json:"subtype,omitempty"`
    SessionID string          `json:"session_id,omitempty"`
    Raw       json.RawMessage `json:"-"` // full original JSON
}

// StreamSubEvent is the nested event inside stream_event.
type StreamSubEvent struct {
    Type         string       `json:"type"`
    Index        int          `json:"index,omitempty"`
    ContentBlock ContentBlock `json:"content_block,omitempty"`
    Delta        Delta        `json:"delta,omitempty"`
}

type ContentBlock struct {
    Type string `json:"type"`
    Text string `json:"text,omitempty"`
    ID   string `json:"id,omitempty"`
    Name string `json:"name,omitempty"`
}

type Delta struct {
    Type        string `json:"type"`
    Text        string `json:"text,omitempty"`
    PartialJSON string `json:"partial_json,omitempty"`
}

// StreamEvent is the full stream_event with its nested event.
type StreamEvent struct {
    Type            string         `json:"type"`
    Event           StreamSubEvent `json:"event"`
    SessionID       string         `json:"session_id"`
    ParentToolUseID *string        `json:"parent_tool_use_id"`
}

// ResultEvent is the final event of a run.
type ResultEvent struct {
    Type         string  `json:"type"`
    Subtype      string  `json:"subtype"`
    IsError      bool    `json:"is_error"`
    Result       string  `json:"result"`
    SessionID    string  `json:"session_id"`
    TotalCostUSD float64 `json:"total_cost_usd"`
    DurationMs   int     `json:"duration_ms"`
    NumTurns     int     `json:"num_turns"`
}

// ParseEvents reads NDJSON lines from a reader and sends parsed events to a channel.
func ParseEvents(r io.Reader, events chan<- json.RawMessage, errs chan<- error) {
    scanner := bufio.NewScanner(r)
    // Increase buffer for large events
    scanner.Buffer(make([]byte, 0, 1024*1024), 1024*1024)

    for scanner.Scan() {
        line := scanner.Bytes()
        if len(line) == 0 {
            continue
        }
        // Make a copy since scanner reuses the buffer
        cp := make([]byte, len(line))
        copy(cp, line)
        events <- json.RawMessage(cp)
    }

    if err := scanner.Err(); err != nil {
        errs <- err
    }
    close(events)
}
```

### Spawning Claude

```go
package claude

import (
    "encoding/json"
    "fmt"
    "io"
    "os"
    "os/exec"
    "strings"
)

type RunConfig struct {
    Prompt       string
    Cwd          string
    SessionID    string
    Model        string
    SettingsPath string
    AllowedTools []string
    MaxTurns     int
}

// UserMessage is the NDJSON input format for stdin.
type UserMessage struct {
    Type    string         `json:"type"`
    Message MessagePayload `json:"message"`
}

type MessagePayload struct {
    Role    string        `json:"role"`
    Content []ContentPart `json:"content"`
}

type ContentPart struct {
    Type string `json:"type"`
    Text string `json:"text"`
}

// SpawnResult holds the started process and its I/O handles.
type SpawnResult struct {
    Cmd    *exec.Cmd
    Stdin  io.WriteCloser
    Stdout io.ReadCloser
}

func SpawnClaude(cfg RunConfig) (*SpawnResult, error) {
    args := []string{
        "-p",
        "--input-format", "stream-json",
        "--output-format", "stream-json",
        "--verbose",
        "--include-partial-messages",
        "--permission-mode", "default",
    }

    if cfg.SessionID != "" {
        args = append(args, "--resume", cfg.SessionID)
    }
    if cfg.Model != "" {
        args = append(args, "--model", cfg.Model)
    }
    if cfg.SettingsPath != "" {
        args = append(args, "--settings", cfg.SettingsPath)
    }
    if len(cfg.AllowedTools) > 0 {
        args = append(args, "--allowedTools", strings.Join(cfg.AllowedTools, ","))
    }
    if cfg.MaxTurns > 0 {
        args = append(args, "--max-turns", fmt.Sprintf("%d", cfg.MaxTurns))
    }

    cmd := exec.Command("claude", args...)
    cmd.Dir = cfg.Cwd

    // Clean env
    env := os.Environ()
    filtered := make([]string, 0, len(env))
    for _, e := range env {
        if !strings.HasPrefix(e, "CLAUDECODE=") {
            filtered = append(filtered, e)
        }
    }
    cmd.Env = filtered
    cmd.Stderr = os.Stderr // or capture separately

    // Get pipes before Start - cannot mix StdinPipe with cmd.Stdin assignment
    stdin, err := cmd.StdinPipe()
    if err != nil {
        return nil, err
    }
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return nil, err
    }

    if err := cmd.Start(); err != nil {
        return nil, err
    }

    return &SpawnResult{Cmd: cmd, Stdin: stdin, Stdout: stdout}, nil
}

// WritePrompt sends a user message to Claude's stdin.
func WritePrompt(stdin io.WriteCloser, prompt string) error {
    msg := UserMessage{
        Type: "user",
        Message: MessagePayload{
            Role: "user",
            Content: []ContentPart{
                {Type: "text", Text: prompt},
            },
        },
    }

    data, err := json.Marshal(msg)
    if err != nil {
        return err
    }

    _, err = fmt.Fprintf(stdin, "%s\n", data)
    return err
}
```

### Bubbletea Integration Skeleton

The key pattern: each `tea.Cmd` reads one event and returns it. Bubbletea calls `Update` with that message, then the model returns the next read command - creating a pull loop without goroutines racing against the program.

```go
package main

import (
    "bufio"
    "encoding/json"
    "fmt"
    "io"
    "os/exec"
    "strings"

    tea "github.com/charmbracelet/bubbletea"
)

type eventMsg struct{ raw json.RawMessage }
type doneMsg struct{ err error }

type model struct {
    output    strings.Builder
    scanner   *bufio.Scanner
    stdin     io.WriteCloser
    done      bool
    err       error
}

func (m model) Init() tea.Cmd {
    cmd := exec.Command("claude", "-p",
        "--input-format", "stream-json",
        "--output-format", "stream-json",
        "--verbose", "--include-partial-messages",
    )
    stdin, _ := cmd.StdinPipe()
    stdout, _ := cmd.StdoutPipe()
    if err := cmd.Start(); err != nil {
        return func() tea.Msg { return doneMsg{err: err} }
    }

    prompt, _ := json.Marshal(map[string]any{
        "type": "user",
        "message": map[string]any{
            "role":    "user",
            "content": []map[string]string{{"type": "text", "text": "Explain what a goroutine is"}},
        },
    })
    fmt.Fprintf(stdin, "%s\n", prompt)

    m.stdin = stdin
    scanner := bufio.NewScanner(stdout)
    scanner.Buffer(make([]byte, 0, 1024*1024), 1024*1024)
    m.scanner = scanner

    return readNext(scanner)
}

// readNext returns a tea.Cmd that blocks until the next NDJSON line arrives.
func readNext(s *bufio.Scanner) tea.Cmd {
    return func() tea.Msg {
        for s.Scan() {
            line := s.Bytes()
            if len(line) == 0 {
                continue
            }
            cp := make([]byte, len(line))
            copy(cp, line)
            return eventMsg{raw: cp}
        }
        return doneMsg{err: s.Err()}
    }
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case eventMsg:
        var base struct {
            Type  string `json:"type"`
            Event struct {
                Type  string `json:"type"`
                Delta struct {
                    Type string `json:"type"`
                    Text string `json:"text"`
                } `json:"delta"`
            } `json:"event"`
        }
        json.Unmarshal(msg.raw, &base)
        if base.Type == "stream_event" &&
            base.Event.Type == "content_block_delta" &&
            base.Event.Delta.Type == "text_delta" {
            m.output.WriteString(base.Event.Delta.Text)
        }
        if base.Type == "result" {
            m.done = true
            m.stdin.Close()
        }
        return m, readNext(m.scanner)

    case doneMsg:
        m.done = true
        m.err = msg.err
        return m, tea.Quit

    case tea.KeyMsg:
        if msg.String() == "q" || msg.String() == "ctrl+c" {
            return m, tea.Quit
        }
    }
    return m, nil
}

func (m model) View() string {
    status := "streaming..."
    if m.done {
        status = "done"
    }
    if m.err != nil {
        return fmt.Sprintf("error: %v\n", m.err)
    }
    return fmt.Sprintf("[%s]\n\n%s\n\nPress q to quit.", status, m.output.String())
}

func main() {
    p := tea.NewProgram(model{})
    if _, err := p.Run(); err != nil {
        fmt.Printf("error: %v\n", err)
    }
}
```

## Patterns

### Cancellation (TypeScript)

```typescript
function cancelRun(child: ChildProcess): void {
  child.kill('SIGINT')

  // Fallback: SIGKILL after 5s if SIGINT didn't work
  setTimeout(() => {
    if (child.exitCode === null) {
      child.kill('SIGKILL')
    }
  }, 5000)
}
```

### Follow-up Message (TypeScript)

```typescript
function sendFollowUp(child: ChildProcess, text: string): void {
  const msg = JSON.stringify({
    type: 'user',
    message: {
      role: 'user',
      content: [{ type: 'text', text }],
    },
  })
  child.stdin!.write(msg + '\n')
}
```

### Track Tool Calls (TypeScript)

```typescript
interface ToolCall {
  id: string
  name: string
  inputFragments: string[]
  complete: boolean
}

const activeTools = new Map<number, ToolCall>() // index -> tool

function handleStreamEvent(event: any): void {
  const sub = event.event
  if (!sub) return

  switch (sub.type) {
    case 'content_block_start':
      if (sub.content_block.type === 'tool_use') {
        activeTools.set(sub.index, {
          id: sub.content_block.id,
          name: sub.content_block.name,
          inputFragments: [],
          complete: false,
        })
      }
      break

    case 'content_block_delta':
      if (sub.delta.type === 'input_json_delta') {
        const tool = activeTools.get(sub.index)
        if (tool) {
          tool.inputFragments.push(sub.delta.partial_json)
        }
      }
      break

    case 'content_block_stop': {
      const tool = activeTools.get(sub.index)
      if (tool) {
        tool.complete = true
        const fullInput = JSON.parse(tool.inputFragments.join(''))
        console.log(`Tool ${tool.name}: ${JSON.stringify(fullInput)}`)
      }
      break
    }
  }
}
```

### Diagnostic Ring Buffer (TypeScript)

Keep a ring buffer of the last N stderr lines for error reporting.

```typescript
const MAX_LINES = 100

class RingBuffer {
  private lines: string[] = []

  push(line: string): void {
    this.lines.push(line)
    if (this.lines.length > MAX_LINES) {
      this.lines.shift()
    }
  }

  tail(n: number): string[] {
    return this.lines.slice(-n)
  }
}

// Usage:
const stderrBuf = new RingBuffer()
child.stderr?.setEncoding('utf-8')
child.stderr?.on('data', (data: string) => {
  for (const line of data.split('\n').filter(l => l.trim())) {
    stderrBuf.push(line)
  }
})

// On error, include last 20 stderr lines in diagnostics
child.on('close', (code) => {
  if (code !== 0) {
    console.error('Last stderr:', stderrBuf.tail(20))
  }
})
```
