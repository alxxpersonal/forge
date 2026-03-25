# Scripting Patterns

Automation examples using exo-teams.

## Fetch all assignments and check status

```bash
#!/bin/bash
echo "=== PENDING DEADLINES ==="
exo-teams deadlines

echo ""
echo "=== ALL ASSIGNMENTS ==="
exo-teams assignments
```

```bash
# Machine-readable pipeline
exo-teams assignments --json | jq '
  .[] | {
    class: .className,
    name: .displayName,
    due: .dueDateTime,
    status: .submissionStatus
  }
'
```

## Download all class materials

```bash
#!/bin/bash
TEAM="Academic Writing"
OUTPUT_DIR="./class-materials"
mkdir -p "$OUTPUT_DIR"

# List files
exo-teams files "$TEAM" --drive "Class Materials" --json | jq -r '.[] | select(.folder == null) | .name' | while read name; do
  exo-teams download "$TEAM" \
    --path "Class Materials/$name" \
    --drive "Class Materials" \
    --output "$OUTPUT_DIR/$name"
done
```

## Send a file to someone

```bash
# Step 1: find their chat ID
exo-teams list-chats | grep "Alice"
# * [DM   ] Alice Smith    19:abc123@unq.gbl.spaces

# Step 2: send
exo-teams send-file "19:abc123@unq.gbl.spaces" \
  --file report.pdf \
  --message "Here's the report"
```

## Submit an assignment

```bash
# Check what's pending
exo-teams deadlines

# Submit
exo-teams submit "Academic Writing" "Essay 1" --file my-essay.docx
```

## Monitor a channel for new messages (poll loop)

```bash
#!/bin/bash
CHANNEL="General"
LAST_DATE=$(date -u +"%Y-%m-%d")

while true; do
  echo "--- checking at $(date) ---"
  exo-teams get-messages "$CHANNEL" --since "$LAST_DATE"
  LAST_DATE=$(date -u +"%Y-%m-%d")
  sleep 60
done
```

## Export a conversation to markdown

```bash
#!/bin/bash
CHAT="Alice"
OUTPUT="chat-export.md"

echo "# Chat Export: $CHAT" > "$OUTPUT"
echo "Generated: $(date)" >> "$OUTPUT"
echo "" >> "$OUTPUT"

exo-teams get-chat "$CHAT" --all --json | jq -r '
  reverse[] |
  select(.messagetype == "RichText/Html" or .messagetype == "Text") |
  "**[\(.composetime | split("T")[0])] \(.imdisplayname):** \(.content | gsub("<[^>]*>"; ""))"
' >> "$OUTPUT"

echo "Exported to $OUTPUT"
```

## Check all unread and mark them

```bash
# See unread
exo-teams unread --json | jq -r '.[].id'

# Mark all unread as read
exo-teams unread --json | jq -r '.[].id' | while read id; do
  echo "marking $id as read..."
  exo-teams mark-read "$id"
done
```

## Bulk search and find files

```bash
# Search for files matching a query
exo-teams search "lecture notes" --json | jq '
  .[] |
  select(.resource.name != null) |
  {name: .resource.name, url: .resource.webUrl}
'
```

## Get today's calendar and assignments together

```bash
echo "=== CALENDAR TODAY ==="
exo-teams calendar --days 1

echo ""
echo "=== DUE SOON ==="
exo-teams deadlines | head -5
```

## Tips

- All commands accept `--json` - pipe into `jq` for filtering/transformation
- `get-messages` and `get-chat` print status to stderr, data to stdout - safe to redirect stdout
- `--all` flag on get-messages follows backwardLink pagination automatically
- `--since` accepts `YYYY-MM-DD` or `YYYY-MM-DD HH:MM:SS`
- Token refresh is automatic - no need to manually call `auth --refresh` in scripts
