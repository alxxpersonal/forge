# Prompt Forge - Code Patterns Reference

Read this file when you need SDK-specific code examples for implementing prompts.

## Claude SDK Patterns

### System Prompt with Caching

```python
system=[{
    "type": "text",
    "text": SYSTEM_PROMPT,
    "cache_control": {"type": "ephemeral"},
}]
```

- Writing to cache costs 25% more than base input price
- Reading from cache costs only 10% of base input price
- Breakeven after ~4 calls with the same cached content

### Structured Outputs (messages.parse)

```python
response = client.messages.parse(
    model="claude-haiku-4-5",
    max_tokens=1024,
    system=[{"type": "text", "text": SYSTEM_PROMPT, "cache_control": {"type": "ephemeral"}}],
    messages=[{"role": "user", "content": user_content}],
    output_format=OutputSchema,  # Pydantic model
)
result = response.parsed_output  # typed instance
```

This is NOT tool use. It enforces the schema at the API level.

### Prefilling

```python
messages=[
    {"role": "user", "content": "Extract the data from this document: ..."},
    {"role": "assistant", "content": "{"}  # Forces JSON output
]
```

Use sparingly - structured outputs are preferred for JSON.

## OpenAI SDK Patterns

### Structured Outputs

```python
response = client.chat.completions.create(
    model="gpt-5.2",
    messages=[...],
    response_format={"type": "json_schema", "json_schema": {...}}
)
```

### Reasoning Effort

```python
response = client.chat.completions.create(
    model="gpt-5.2",
    messages=[...],
    reasoning_effort="medium"  # none, minimal, low, medium, high, xhigh
)
```

## Confidence Schema Pattern

```python
class ConfidenceLevel(str, Enum):
    HIGH = "HIGH"       # Clearly stated, unambiguous
    MEDIUM = "MEDIUM"   # Reasonable inference from context
    LOW = "LOW"         # Weak signal, likely needs human review
    MISSING = "MISSING" # Not found in source

class ExtractedData(BaseModel):
    field_a: Optional[str] = None
    field_a_confidence: ConfidenceLevel = ConfidenceLevel.MISSING
    overall_confidence: ConfidenceLevel = ConfidenceLevel.MISSING
    fields_needing_review: list[str] = []
```

