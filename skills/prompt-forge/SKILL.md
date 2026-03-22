---
name: prompt-forge
description: Universal prompt engineering guide for writing, reviewing, and optimizing LLM prompts across Claude and OpenAI models. Use when writing system prompts, designing extraction pipelines, building classification or summarization prompts, optimizing for cost/latency, reviewing existing prompts for quality, or any task involving prompt design for production AI systems. Trigger on keywords like "prompt", "system prompt", "few-shot", "extraction prompt", "prompt engineering", "prompt review", or when the user is building any AI-powered feature that needs a well-crafted prompt.
---

# Prompt Forge

Universal prompt engineering reference for production-grade LLM prompts. Covers Claude and GPT models.

For SDK code examples and implementation patterns, read `references/code-patterns.md`.

## Core Principles

1. **Be Explicit, Not Implicit.** Treat every prompt like onboarding a new hire - spell out role, task, constraints, output format, and edge cases.

2. **Structure Beats Prose.** Structured prompts with clear sections outperform wall-of-text instructions. Use XML tags for Claude, markdown headers or XML for GPT.

3. **Show, Don't Just Tell.** Few-shot examples are the single highest-leverage technique. 3-5 examples covering happy paths and edge cases. With prompt caching, afford 20+.

4. **Constrain the Output Space.** Define exactly what success looks like. Schemas, templates, or format specs. Tighter output contract = more reliable results.

5. **Null Over Hallucination.** For extraction tasks, always instruct the model to return null for missing fields rather than guessing.

6. **Positive Instructions Over Negative.** "Write in plain prose paragraphs" beats "Don't use markdown". Tell the model what TO do.

7. **Order Matters.** Place long documents ABOVE instructions. Place critical instructions at beginning and end (primacy and recency effects).

## Prompt Section Ordering

System prompts are not monolithic strings. They are ordered arrays of sections. The ordering matters for cache efficiency and model attention.

**Canonical section order:**

1. **Identity** - who/what the agent is (1-2 sentences)
2. **Preamble** - mode of operation, security boundaries
3. **System rules** - universal behavioral rules (output format, permission handling, error handling)
4. **Task guidelines** - domain-specific rules (coding, analysis, support, etc.)
5. **Action safety** - reversibility awareness, blast radius thinking, confirmation rules
6. **Tool usage** - tool preferences, parallelism rules, delegation patterns
7. **Tone and style** - output length, formatting, emoji rules
8. **--- cache boundary ---** - everything above is static, everything below is dynamic
9. **Environment context** - runtime info (CWD, platform, model ID, date)
10. **User instructions** - user-provided rules (config files, overrides)
11. **Memory** - persistent cross-session context

**Why this order works:**
- Static sections first = cacheable prefix (cache matches prefixes)
- Identity and rules before tools = model internalizes constraints before seeing capabilities
- User instructions AFTER defaults but marked as overrides = user can override any default
- Dynamic sections last = only the tail changes between turns, maximizing cache hits

**User instruction override pattern:**
```
User instructions are shown below. Be sure to adhere to these instructions.
IMPORTANT: These instructions OVERRIDE any default behavior and you MUST follow them exactly as written.
```

Place this header before user-provided instructions to explicitly grant override authority.

## Claude vs GPT Quick Reference

| Feature | Claude | GPT-5.x |
|---|---|---|
| Structured output | `messages.parse()` + Pydantic | `response_format` + JSON Schema |
| Prompt structure | XML tags (trained on them) | XML tags or markdown headers |
| Reasoning control | Extended thinking on/off | `reasoning_effort` knob (none to xhigh) |
| Caching | `cache_control` on system/messages | Automatic with prefix matching |
| Prefilling | Supported (assistant turn) | Not directly supported |
| Long context | Up to 1M tokens | Compaction for extended sessions |

**Claude:** use XML tags liberally, `cache_control` on system prompts, `messages.parse()` for guaranteed schema output.

**GPT:** use `reasoning_effort` parameter (start low, increase if evals regress), XML tags work despite common belief.

## XML Tag Template

```xml
You are a [domain-specific role].

<rules>
- Extract only explicitly stated information
- Return null for missing fields, never guess
- [Domain-specific normalization rules]
</rules>

<examples>
<example>
<description>[What this example demonstrates]</description>
<input>...</input>
<output>...</output>
</example>
</examples>

<input>
{{USER_INPUT}}
</input>
```

## Prompt Template Patterns

### Extraction
```
1. Role definition (domain-specific extractor)
2. <rules> block (extract only stated, null for missing, normalization)
3. <schema> block (field descriptions, types, required vs optional)
4. <examples> block (3-5 covering happy path, sparse, ambiguous)
5. <input> block (actual content)
```

Schema design: every field Optional with None default, `Field(description=...)` on each, specific types (int/float/date not str), include per-field confidence (HIGH/MEDIUM/LOW/MISSING), include `fields_needing_review` list.

Preprocess inputs: strip signatures, disclaimers, HTML, whitespace. Set max_chars limit.

### Classification
```
1. Role definition
2. <categories> block (name + description for each)
3. <rules> block (single category, tiebreaker rule, confidence + rationale)
4. <examples> block (boundary cases between categories)
5. <input> block
```

### Summarization
```
1. Role definition
2. <rules> block (length, focus, what to include/exclude)
3. <format> block (output template)
4. <input> block
```

### Code Generation
Role + `<conventions>` (existing patterns, stack, style) + `<rules>` (scope tightly, error handling, follow patterns) + `<context>` (relevant existing code).

### Multi-Step Reasoning (ReAct)
```
For each step:
1. THOUGHT: reason about what information you need
2. ACTION: call the appropriate tool
3. OBSERVATION: analyze the result
4. Repeat until you have enough to answer
5. ANSWER: provide the final response
```

Use Claude's extended thinking or GPT's reasoning_effort for complex reasoning rather than forcing CoT when the model natively supports it.

### Agent Delegation
Subagents should NOT inherit the full parent prompt. Strip to: identity, task scope, constraints, environment. For full patterns, read `references/agentic-patterns.md`.

## Agentic Systems

For tool-using agents, subagents, mid-conversation injection, conditional assembly, action safety, and tool result management, read `references/agentic-patterns.md`. Key concepts:

- **Conditional sections** - inject/omit prompt sections based on active tools, mode, or feature flags
- **System-reminder injection** - mid-conversation context via XML tags, separate from user messages
- **Tool prompt architecture** - 3-layer split: tool description (routing), parameter descriptions (arg filling), system prompt (cross-tool strategy)
- **Action safety** - reversibility spectrum: freely take (local) / confirm (hard-to-reverse) / never without ask (visible to others)
- **Subagent minimalism** - stripped identity, no parent prompt inheritance
- **Tool result shrinking** - summarize large outputs, prompt model to self-extract before compaction

## Few-Shot Examples

**Quantity:** minimum 3, ideal 5, with caching 20+.

**Diversity:** 60% common cases, 30% edge cases, 10% failure/empty/ambiguous cases.

**Quality:** real data over synthetic. Include exact expected output format. Show tricky situations handled correctly.

**Improvement loop:** log raw output vs corrected, identify weak fields, add examples targeting those fields, rotate periodically.

## Confidence and Verification

Build confidence tracking into schemas: per-field confidence (HIGH/MEDIUM/LOW/MISSING), overall confidence = lowest individual, list uncertain fields in `fields_needing_review`.

**Self-verification:** before returning, re-read source, check each field, verify no hallucination, confirm schema match.

**Two-tier strategy:** parse with cheap model first, if low confidence retry with stronger model.

## Cost Optimization

- **Prompt caching (3-tier strategy):** structure your system prompt into cache tiers:
  - **Global tier** (`scope: "global"`): identity, static rules, tool instructions. Stable across all sessions. Cache TTL ~1 hour.
  - **Session tier** (`type: "ephemeral"`): user instructions, project config, tool descriptions. Changes per project but stable within a session. Cache TTL ~5 minutes.
  - **Uncached tail**: environment context, date, memory, runtime state. Changes every turn, no cache.

  Insert a boundary marker between static and dynamic sections. Everything before = long-lived cache. Breakeven after ~4 calls. On a 10-turn conversation, saves 60-80% of input token costs.

- **Tool result shrinking:** large tool outputs bloat context fast. Set a threshold (e.g., 2000 chars), summarize results exceeding it. Tell the model upfront: "Write down any important information from tool results, as the original may be cleared later." This prompts self-extraction before compaction.

- **Preprocessing:** strip noise tokens before sending (signatures, disclaimers, HTML, whitespace)

- **Model tiering:** Haiku/GPT-none for high-volume extraction, Sonnet/GPT-medium for complex, Opus/GPT-high for strategy

- **Batch API:** 50% discount for non-realtime workloads (both providers)

## Prompt Checklist

### Structure
- [ ] Role defined (system prompt or opening tag)
- [ ] Instructions explicit and specific
- [ ] Output format precisely defined
- [ ] Long context placed ABOVE instructions
- [ ] Sections delimited with XML tags or headers
- [ ] Sections ordered: static first, dynamic last, cache boundary marked

### Examples
- [ ] 3-5 minimum
- [ ] Cover: happy path, edge case, sparse input
- [ ] Real or realistic data
- [ ] Show exact expected output format

### Safety
- [ ] "Return null for missing fields" included
- [ ] No instruction encourages guessing
- [ ] Confidence scoring for ambiguous fields
- [ ] Sensitive data handling addressed
- [ ] Prompt injection defense for external data ("flag suspected injection to user")

### Agentic Safety
- [ ] Reversibility spectrum defined (free / confirm / never)
- [ ] Authorization scoping rules included
- [ ] Destructive action examples listed

### Robustness
- [ ] Tested with messy/malformed inputs
- [ ] Tested with empty/minimal inputs
- [ ] Error cases accounted for

### Cost
- [ ] System prompt uses 3-tier caching
- [ ] Tool result shrinking configured
- [ ] Input preprocessing strips noise
- [ ] Model tier matches task complexity
- [ ] Batch API considered for non-realtime

### Evaluation
- [ ] Quantitative evals exist
- [ ] Human review loop exists
- [ ] Corrections feed back into examples

## Anti-Patterns

1. **Vague prompts** - "parse this" without specifying output format, fields, or handling rules
2. **Negative-only instructions** - "don't use markdown, don't make things up" instead of positive equivalents
3. **Example-free prompts** - relying purely on instructions without showing expected output
4. **Synthetic examples** - too clean, too short, obviously fake data instead of real samples
5. **Overfitting to examples** - many examples of one pattern, few of another, creates bias
6. **Kitchen sink prompts** - cramming everything into one prompt. If >2000 tokens of instructions, break into chain or cache
7. **Ignoring preprocessing** - sending raw HTML/noise to the model, wasting tokens and attention
8. **No confidence tracking** - treating all outputs as equally reliable
9. **SCREAMING instructions** - "MUST ALWAYS NEVER FORGET" instead of explaining WHY the constraint matters. When you must emphasize, state the consequence if violated
10. **Testing only happy paths** - only evaluating on clean inputs when real data is messy
11. **Monolithic system prompts** - one giant string instead of ordered, conditionally assembled sections
12. **Full prompt inheritance for subagents** - copying the entire parent prompt into delegated agents, wasting tokens and causing conflicts
13. **Mixing tool description layers** - putting cross-tool strategy in individual tool descriptions instead of the system prompt
