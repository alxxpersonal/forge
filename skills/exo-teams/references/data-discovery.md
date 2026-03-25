# Data Discovery Guide

How to find IDs and data buried in Microsoft Teams.

## "How do I find which teams I'm in?"

```bash
exo-teams list-teams
```

Output shows team names and all their channels with IDs. The group ID (for SharePoint/files) is in the JSON:
```bash
exo-teams list-teams --json | jq '.[].teamSiteInformation.groupId'
```

## "How do I find a channel ID?"

```bash
exo-teams list-teams
# Look for: #channel-name   19:abc123@thread.tacv2
```

Channel IDs look like `19:abc123@thread.tacv2`. Pass them directly to `get-messages`.

## "How do I find a conversation/chat ID?"

```bash
exo-teams list-chats
# Look for: [DM]  Person Name   19:abc@unq.gbl.spaces
```

Chat IDs look like `19:abc@unq.gbl.spaces`. Pass them to `send`, `get-chat`, `mark-read`, `send-file`.

## "How do I find a file in SharePoint?"

```bash
# List default drive
exo-teams files "Team Name"

# List a specific subfolder
exo-teams files "Team Name" --path "General/Week 1"

# See class materials (education teams)
exo-teams files "Team Name" --drive "Class Materials"

# See ALL drives at once
exo-teams files "Team Name" --all-drives
```

The file path shown in `files` output is what you pass to `--path` for `download`.

## "How do I find assignment details?"

```bash
# List all classes
exo-teams assignments --classes

# List all assignments with status
exo-teams assignments

# Get full JSON including classId, assignment ID, submission ID
exo-teams assignments --json
```

Assignment status icons:
- `[ ]` - not submitted
- `[~]` - working / in progress
- `[x]` - submitted
- `[>]` - returned (graded)

## "How do I check unread messages?"

```bash
# Quick overview
exo-teams unread

# Full list with IDs
exo-teams list-chats
# * prefix = unread
```

Unread is based on `isRead == false` in chat metadata. Note: `isRead` can be null (treated as read) or bool.

## "How do I find someone's user ID?"

```bash
# Try new-dm - it will print the found user ID to stderr
exo-teams new-dm "their name" "test" 2>&1 | head
# stderr: found user: Alice Smith (alice@school.edu)
# stderr: conversation created: 19:...

# Or search via JSON
exo-teams list-chats --json | jq '.[].members[] | select(.friendlyName | test("Alice"))'
# Returns: {"mri":"8:orgid:uuid-here","friendlyName":"Alice Smith","role":"User"}
```

MRI format is `8:orgid:<uuid>`. The UUID is the Azure AD object ID used in Graph API calls.

## "How do I find what I submitted for an assignment?"

```bash
exo-teams assignments --json | jq '.[] | select(.displayName | test("Essay")) | {status: .submissionStatus, submitted: .submittedDateTime, resources: .submittedResources}'
```

Or just run `exo-teams assignments` - submitted resources are printed under each assignment.

## "How do I see all my classes?"

```bash
exo-teams assignments --classes
# Output: ClassName   class-uuid
```

The class UUID is what the assignments API uses internally.

## ID Format Reference

| ID type | Format example | Where to get it |
|---------|---------------|-----------------|
| Channel ID | `19:abc@thread.tacv2` | `list-teams` |
| Chat/DM ID | `19:abc@unq.gbl.spaces` | `list-chats` |
| Group ID (team) | UUID | `list-teams --json` -> `teamSiteInformation.groupId` |
| User MRI | `8:orgid:uuid` | `list-chats --json` -> `members[].mri` |
| Class ID | UUID | `assignments --classes` |
| Assignment ID | UUID | `assignments --json` -> `id` |
| Submission ID | UUID | `assignments --json` -> `submissionId` |
