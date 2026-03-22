# Prompt Forge - Agentic Patterns Reference

Read this file when building agentic systems with tool use, subagents, or real-world actions.

## Conditional Section Assembly

Production prompts are not static. Sections should be injected or omitted based on runtime state.

**Tool-aware instructions.** Only include rules about a tool when that tool is available:
```
if "search" in available_tools:
    sections.append("For codebase searches, use the Search tool instead of grep.")
```

**Mode-aware sections.** Skip domain-specific instructions when the agent operates in a different mode:
```
if mode == "code":
    sections.append(coding_guidelines)
elif mode == "support":
    sections.append(support_guidelines)
```

**Feature-flagged sections.** Gate experimental instructions behind flags:
```
if feature_flags.get("verbose_output"):
    sections.append(output_efficiency_section)
```

Every unnecessary token in the system prompt costs money and dilutes attention. A prompt with 20 tools that includes instructions for all 20 wastes tokens on tools the model may never use.

## Mid-Conversation Context Injection

Not all instructions belong in the system prompt. Some context should be injected at runtime as tagged messages within the conversation.

**System-reminder pattern.** Wrap injected context in XML tags as a user message:
```xml
<system-reminder>
Today's date is March 22, 2026.

IMPORTANT: this context may or may not be relevant to your tasks.
You should not respond to this context unless it is highly relevant.
</system-reminder>
```

**When to use system-reminders vs system prompt:**

| Content | Where |
|---------|-------|
| Static rules, identity, tool instructions | System prompt |
| User config files, project instructions | First user message (system-reminder wrapper) |
| Date changes, background task results | Injected mid-conversation as system-reminder |
| Warnings about specific tool results | Appended to the tool result content |
| Memory staleness alerts | Appended after memory content |

Tell the model explicitly that these tags "bear no direct relation to the specific tool results or user messages in which they appear." Otherwise the model may attribute the reminder to the wrong context.

## Tool Prompt Architecture

When building agentic systems with tool use, instructions for tools live in three separate places.

| Layer | What goes here | Example |
|-------|---------------|---------|
| **Tool `description`** | What the tool does, when to use it. Model reads this to decide whether to call it. | "Reads a file from the local filesystem." |
| **Tool `parameter.description`** | Per-parameter guidance. Model reads this when filling in arguments. | "The absolute path to the file to read" |
| **System prompt tool section** | Cross-tool preferences, delegation rules, parallelism instructions. | "Use Read instead of cat. Call multiple tools in parallel when independent." |

**Anti-pattern:** putting usage strategy in tool descriptions. "Always prefer this tool over Bash" belongs in the system prompt, not the tool's description.

**Anti-pattern:** duplicating instructions across tool descriptions and system prompt. Pick one location.

## Subagent Prompt Minimalism

Subagents should NOT inherit the full parent prompt. Strip everything except identity, task scope, correctness constraints, and environment info. Omit tone/style, methodology, tool preferences, action safety rules.

**Subagent prompt structure:**
```
1. Tight identity (what you are, scope of work, how to report back)
2. Operational constraints (absolute paths, no emojis, concise responses)
3. Minimal environment context (CWD, platform, model)
```

**Subagent identity pattern:**
```
Given the user's message, use the tools available to complete the task.
Do what has been asked; nothing more, nothing less.
When you complete the task, respond with a concise report covering what was done
and any key findings.
```

This prevents subagents from over-explaining, expanding scope, or duplicating parent work.

## Action Safety (Reversibility Spectrum)

For agents that take real-world actions (file edits, git operations, API calls, deployments), include a reversibility framework in the system prompt.

**Reversibility spectrum:**
```
Freely take: local, reversible actions (editing files, running tests)
Confirm first: hard-to-reverse actions (force push, delete branch, deploy)
Never without explicit ask: actions visible to others (push, comment on PRs, send messages)
```

**Key rules:**
- "A user approving an action once does NOT mean they approve it in all contexts"
- "Authorization stands for the scope specified, not beyond"
- "Do not use destructive actions as a shortcut to make obstacles go away"
- "If you discover unexpected state, investigate before deleting or overwriting"

**Anti-pattern:** binary "ask for everything" or "do everything autonomously." The spectrum approach gives the model judgment within guardrails.

## Tool Result Shrinking (Microcompaction)

Large tool outputs (file contents, search results, command output) bloat context fast. Apply automatic replacement:

- Set a threshold (e.g., 2000 chars). Tool results exceeding it get summarized.
- Track replacements in state so you can restore if needed.
- Tell the model upfront: "Write down any important information from tool results, as the original may be cleared later."

This prompts the model to self-extract key facts before compaction removes the raw output.
