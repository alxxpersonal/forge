---
name: exo-teams
description: CLI tool for Microsoft Teams internal API - no admin consent required. Use when working with Teams messages, DMs, assignments, class materials, file uploads, calendar, or deadlines. Trigger on "teams", "exo-teams", "microsoft teams", "assignments", "class materials", "send message teams", "teams files", "uni automation", "deadlines", or "submit assignment".
argument-hint: "[command or 'help']"
---

# exo-teams

Go CLI for Microsoft Teams using the internal (desktop app) API. No admin consent, no Graph API registration, no IT department. Tokens stored at `~/.exo-teams/`.

## Auth

Five token scopes, all acquired via device code OAuth (`exo-teams auth`):

| Token | Used for |
|-------|----------|
| skype | messaging, activity feed, read receipts |
| chatsvcagg | teams/channels/chats listing |
| teams | middle tier ops |
| graph | calendar, files, search, user profiles |
| assignments | education assignments via `assignments.onenote.com` (bypasses admin consent) |

Tokens auto-refresh. Check status with `exo-teams whoami`.

**Skypetoken quirk**: messaging uses `Authentication: skypetoken=<value>` header, NOT `Authorization: Bearer`. The skypetoken is derived from the raw skype JWT via an authz exchange endpoint.

## Quick Start

```bash
exo-teams auth                              # device code login
exo-teams whoami                            # token status + account info
exo-teams list-teams                        # see all teams, channels, and channel IDs
exo-teams list-chats                        # all DMs and group chats with IDs
```

## Command Reference

See `references/commands.md` for full flag docs. Quick map:

| Command | What it does |
|---------|-------------|
| `auth [--import\|--refresh]` | login, import from fossteams, or force refresh |
| `whoami` | account info + per-token expiry |
| `list-teams` | all teams + channels with IDs |
| `list-chats` | all DMs + group chats with IDs |
| `get-messages <search>` | channel messages by name search or ID |
| `get-chat <search>` | DM/group chat messages |
| `send <conv-id> <message>` | send text to any conversation |
| `send-file <conv-id> --file <path>` | upload file to OneDrive, send share link |
| `new-dm <name> <message>` | find user by name, open DM, send message |
| `files <team-search>` | list SharePoint files |
| `upload <team-search> --path --file` | upload to SharePoint |
| `download <team-search> --path` | download from SharePoint |
| `calendar [--days N]` | upcoming calendar events (default 7 days) |
| `assignments [--classes]` | all assignments with submission status |
| `submit <class> <assignment> --file` | submit a file to an assignment |
| `deadlines` | pending assignments sorted by due date |
| `unread` | unread conversations at a glance |
| `activity` | activity feed (mentions, replies, reactions) |
| `search <query>` | search messages + files |
| `mark-read <conv-id>` | mark conversation as read |

All commands accept `--json` for machine-readable output.

## Data Discovery Guide

See `references/data-discovery.md` for full patterns. Critical ones:

**Find team/channel IDs:**
```bash
exo-teams list-teams
# Output: == Team Name == / #channel-name   19:abc123@thread.tacv2
```

**Find chat/DM IDs:**
```bash
exo-teams list-chats
# Output: * [DM]  Person Name   19:abc@unq.gbl.spaces
```

**Find your user ID:**
```bash
exo-teams whoami --json
# or: exo-teams new-dm "their name" "" (prints found user ID to stderr)
```

**Find assignment IDs:**
```bash
exo-teams assignments --classes     # list class IDs
exo-teams assignments --json        # includes classId, id, submissionId
```

## Key Gotchas

- Conversation IDs contain `:` and `@` - they are always URL-encoded internally, but pass them raw to CLI args
- `chat.hidden` is unreliable - most active DMs have `hidden=true`, use `list-chats` not hidden filter
- `annotationsSummary.emotions` returns either `[]any` or `map[string]any` depending on message
- File upload to DMs uses OneDrive ("Microsoft Teams Chat Files" folder) + share link, not AMS
- SharePoint upload (team files) uses Graph PUT to `/groups/{id}/drive/root:/{path}:/content`
- 423 on OneDrive upload means file is locked - tool auto-retries with deduped name up to 5 times
- Activity feed reads from a special conversation ID `48:notifications`
- Unread status comes from `isRead` field on Chat which can be bool or null - null means read
- Search uses Graph `/search/query` with `message` + `driveItem` entity types (no Chat.Read needed)
- `assignments` command uses `assignments.onenote.com` API not Graph `/education/` (bypasses admin consent)

## Common Patterns

```bash
# Check what needs submitting today
exo-teams deadlines

# Read a specific channel
exo-teams get-messages "Academic Writing" --replies --count 50

# Read all messages ever (paginated)
exo-teams get-messages "General" --all

# Messages after a date
exo-teams get-messages "General" --since 2026-03-01

# Send a file to a DM
exo-teams send-file "19:abc@unq.gbl.spaces" --file report.pdf --message "here it is"

# Multi-file send
exo-teams send-file "19:abc@unq.gbl.spaces" --file a.pdf --file b.docx

# Download class material
exo-teams files "Academic Writing" --drive "Class Materials"
exo-teams download "Academic Writing" --path "Class Materials/week1.pdf" --drive "Class Materials"

# Submit assignment
exo-teams submit "Academic Writing" "Essay 1" --file essay.docx

# Export channel to JSON
exo-teams get-messages "General" --all --json > general.json
```

## Scripting Patterns

See `references/scripting.md` for full automation examples including:
- Fetch all pending assignments + download class materials
- Monitor channel for new messages (poll loop)
- Export full conversation to markdown
- Bulk-send files across multiple conversations
