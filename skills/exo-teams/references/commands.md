# Command Reference

Full flag documentation for every exo-teams command.

## auth

```
exo-teams auth [--import] [--refresh]
```

- No flags: device code OAuth login (opens browser)
- `--import`: import tokens from fossteams (`~/.config/fossteams/`)
- `--refresh`: force refresh using saved refresh token

## whoami

```
exo-teams whoami [--json]
```

Shows display name, email, and expiry for all 5 tokens. JSON output is an array of `{name, valid, expiry}`.

## list-teams

```
exo-teams list-teams [--json]
```

Lists all teams with their channels. Output format:
```
== Team Name ==
   Description
   #channel-name                   19:channelid@thread.tacv2 (general)
```

The channel ID is what you pass to `get-messages`.

## list-chats

```
exo-teams list-chats [--json]
```

Lists DMs and group chats. Resolves MRI UUIDs to display names via Graph API.
```
* [DM   ] Person Name                       19:chatid@unq.gbl.spaces
           Person: last message preview
```

`*` prefix means unread. The chat ID is what you pass to `get-chat`, `send`, `send-file`, `mark-read`.

## get-messages

```
exo-teams get-messages <search> [--count N] [--all] [--replies] [--since DATE] [--json]
```

- `<search>`: matches against "TeamName ChannelName" (case-insensitive), or exact channel ID
- `--count N`: messages per page (default 200)
- `--all`: follow backwardLink pagination to get all messages
- `--replies`: show reply threads grouped and indented
- `--since DATE`: filter to messages on/after date, format `2026-03-15`

Output (oldest first):
```
[2026-03-20 14:22] Alice: message content
  > [2026-03-20 14:25] Bob: reply content  (with --replies)
```

## get-chat

```
exo-teams get-chat <search> [--count N] [--all] [--replies] [--since DATE] [--json]
```

- `<search>`: matches against title, member name, chat ID, or last message content
- Same flags as `get-messages`

## send

```
exo-teams send <conversation-id> <message>
```

Sends a plain text message. Works for both channel IDs and chat IDs.

## send-file

```
exo-teams send-file <conversation-id> --file <path> [--file <path>...] [--message <text>]
```

- `--file`: local file path (repeat for multiple files)
- `--message`: optional text with the first file

**How it works**: uploads to OneDrive `/me/drive/root:/Microsoft Teams Chat Files/<name>:/content`, creates an org-scoped share link, then sends a message with the file metadata. Retries up to 5 times on 423 (locked) with deduplicated names.

## new-dm

```
exo-teams new-dm <user-search> <message>
```

Searches tenant users via Graph `$filter=startswith(displayName,'...')`, creates a 1:1 conversation, sends the message. Prints found user name/email to stderr.

## files

```
exo-teams files <team-search> [--path <subfolder>] [--drive <drive-name>] [--all-drives] [--json]
```

- `<team-search>`: partial team name match
- `--path`: subfolder within drive (e.g. `General` or `General/Week 1`)
- `--drive`: specific drive by name (e.g. `Class Materials`)
- `--all-drives`: list files from ALL drives (Documents, Class Materials, etc.)

Education teams typically have two drives: `Documents` (default) and `Class Materials`.

Output:
```
F filename.pdf                          1234 KB  2026-03-20 14:22  Alice
D Subfolder                             3 items  2026-03-15 10:00  Bob
```

## upload

```
exo-teams upload <team-search> --path <remote-path> --file <local-file>
```

- `--path`: remote path in team drive (e.g. `General/report.pdf`)
- `--file`: local file to upload

Uses Graph PUT to `/groups/{groupId}/drive/root:/{path}:/content`.

## download

```
exo-teams download <team-search> --path <file-path> [--output <dir-or-file>] [--drive <drive-name>]
```

- `--path`: file path in team drive (e.g. `General/Cat Cafe (1).xlsx`)
- `--output`: local output directory or full file path (default: current directory)
- `--drive`: target a specific drive by name

## calendar

```
exo-teams calendar [--days N] [--json]
```

- `--days N`: days ahead to show (default 7)

Shows subject, time range, location, organizer, and body preview. Skips cancelled events.

## assignments

```
exo-teams assignments [--classes] [--json]
```

- `--classes`: list education classes with IDs instead of assignments

Default output shows all assignments across all classes with submission status:
```
[ ] Assignment Name               due: 2026-03-25 23:59  class: Academic Writing
    Instructions preview...
[ ] uncompleted
[~] in progress / working
[x] submitted
[>] returned (graded)
```

Submitted files are listed with their names and links.

## submit

```
exo-teams submit <class-search> <assignment-search> --file <path>
```

1. Finds class by partial name match
2. Finds assignment by partial name match
3. Gets your submission ID
4. Calls SetUpResourcesFolder
5. Uploads file as a submission resource
6. Calls submit endpoint

## deadlines

```
exo-teams deadlines [--json]
```

Shows only non-submitted assignments, sorted by due date. Calculates days remaining. Shows OVERDUE, TODAY, TOMORROW labels.

## unread

```
exo-teams unread [--json]
```

Filters chats where `isRead == false`. Shows title, last sender, and last message preview.

## activity

```
exo-teams activity [--json]
```

Reads from the `48:notifications` special conversation thread. Shows activityType (mention, reply, reaction), sender, and content.

## search

```
exo-teams search <query> [--json]
```

Searches using Graph `/search/query` with `message` and `driveItem` entity types. Returns up to 25 hits. Shows sender, timestamp, channel/location, and content preview.

## mark-read

```
exo-teams mark-read <conversation-id>
```

Sets consumptionHorizon to current timestamp via PUT to internal API.
