---
name: charm-gum
description: "Interactive shell script prompts, fuzzy filters, spinners, and styled output with gum. Use when building bash/shell script UIs, gum commands, interactive shell prompts, or CLI script workflows. NOT for Go terminal forms (use huh)."
---

# charm-gum

gum is a CLI for glamorous shell scripts. All interaction writes to stdout; capture with `$()`. All commands render to stderr so stdout stays clean for piping.

## Quick Start

```bash
brew install gum           # macOS/Linux
go install github.com/charmbracelet/gum@latest

NAME=$(gum input --placeholder "your name")
gum confirm "Continue?" && echo "hello $NAME"
```

Every flag has an env var equivalent: `--placeholder` = `GUM_INPUT_PLACEHOLDER`. Export env vars to set defaults project-wide.

## Command Reference

### input - single-line prompt

```bash
gum input [flags]
# key flags:
#   --placeholder "text"   hint text
#   --value "text"         pre-filled value
#   --password             mask input
#   --header "text"        label above input
#   --width N              fixed width (0 = terminal width)
#   --char-limit N         max chars (default 400, 0 = unlimited)
#   --timeout 30s          auto-submit after duration

NAME=$(gum input --placeholder "full name" --header "Enter your name")
PASS=$(gum input --password --placeholder "password")
```

### write - multi-line textarea

```bash
gum write [flags]
# key flags:
#   --placeholder "text"
#   --header "text"
#   --width N, --height N
#   --show-line-numbers
#   --show-cursor-line
#   --max-lines N
# ctrl+d to submit, ctrl+c to cancel

BODY=$(gum write --placeholder "PR description..." --header "Description" --width 80)
```

### choose - pick from a list

```bash
gum choose [options...] [flags]
# pipe options or pass as args
# key flags:
#   --limit N              max selectable (default 1)
#   --no-limit             unlimited selection
#   --header "text"
#   --height N             visible rows (default 10)
#   --cursor "> "          cursor prefix
#   --selected "val"       pre-selected item
#   --ordered              preserve selection order
#   --timeout 30s

TYPE=$(gum choose "fix" "feat" "docs" "chore" "refactor")
PKGS=$(brew list | gum choose --no-limit --header "Remove packages")
```

### filter - fuzzy search a list

```bash
gum filter [options...] [flags]
# reads from stdin or args; fuzzy match by default
# key flags:
#   --limit N
#   --no-limit
#   --placeholder "text"
#   --header "text"
#   --height N
#   --value "text"         initial filter query
#   --no-fuzzy             prefix match only
#   --no-strict            return query if no match
#   --reverse              render from bottom

SESSION=$(tmux list-sessions -F '#S' | gum filter --placeholder "pick session...")
BRANCH=$(git branch | cut -c 3- | gum filter --placeholder "checkout...")
```

### confirm - yes/no prompt

```bash
gum confirm [prompt] [flags]
# exits 0 = yes, 1 = no; use with && / ||
# key flags:
#   --affirmative "Yes"    confirm button label
#   --negative "No"        cancel button label
#   --default              which is pre-selected (true = yes)
#   --timeout 30s
#   --show-output          echo chosen action to stdout

gum confirm "Delete branch?" && git branch -D "$BRANCH"
gum confirm "Overwrite?" --affirmative "Overwrite" --negative "Skip" || exit 0
```

### spin - spinner while command runs

```bash
gum spin [flags] -- <command>
# key flags:
#   --title "text"         message shown next to spinner
#   --spinner dot          type: line,dot,minidot,jump,pulse,points,globe,moon,monkey,meter,hamburger
#   --show-output          stream stdout/stderr live
#   --show-error           show output only on failure
#   --timeout 60s

gum spin --title "Installing deps..." -- npm install
OUTPUT=$(gum spin --show-output --title "Fetching..." -- curl -s https://api.example.com/data)
```

### style - styled text output

```bash
gum style [flags] "text" ["text2" ...]
# multiple strings are rendered as separate lines in one block
# key flags:
#   --foreground "#hex"|"256color"
#   --background "#hex"|"256color"
#   --border none|hidden|normal|rounded|thick|double
#   --border-foreground
#   --align left|center|right
#   --width N, --height N
#   --margin "T R B L" (css shorthand)
#   --padding "T R B L"
#   --bold, --italic, --underline, --strikethrough, --faint

gum style --foreground 212 --border rounded --padding "1 2" "Done!"
```

### format - render markdown, code, templates, emoji

```bash
gum format [flags] [text...]
# key flags:
#   -t markdown|template|code|emoji   (default: markdown)
#   -l python                          language hint for code type
#   --theme pink                       glamour theme for markdown

echo "# Hello\n- item 1\n- item 2" | gum format
cat script.sh | gum format -t code -l bash
echo '{{ Bold "OK" }} {{ Color "99" "0" " gum " }}' | gum format -t template
echo "I :heart: gum :candy:" | gum format -t emoji
```

### join - compose styled blocks side by side or stacked

```bash
gum join [flags] "block1" "block2"
# flags:
#   --vertical     stack top to bottom (default is horizontal)
#   --align left|center|right

# always quote gum style output to preserve newlines
A=$(gum style --border rounded --padding "0 2" "left")
B=$(gum style --border rounded --padding "0 2" "right")
gum join "$A" "$B"
```

### file - file picker from tree

```bash
gum file [path]   # defaults to current dir
# flags: --cursor, --height, --show-hidden

$EDITOR "$(gum file $HOME)"
```

### pager - scrollable viewer

```bash
gum pager < README.md
gum pager --show-line-numbers < file.txt
```

### table - pick a row from CSV

```bash
gum table < data.csv
gum table -c Name,Age,Role < users.csv | cut -d',' -f1
```

### log - structured log output

```bash
# levels: debug, info, warn, error, fatal
gum log --level info "Server started" port 8080
gum log --structured --level error "Failed" file foo.txt
gum log --time rfc822 --level warn "Slow response"
```

## Shell Script Patterns

### 1. conventional commit helper

```bash
#!/bin/bash
set -e

TYPE=$(gum choose "fix" "feat" "docs" "style" "refactor" "test" "chore")
SCOPE=$(gum input --placeholder "scope (optional)")
[ -n "$SCOPE" ] && SCOPE="($SCOPE)"

SUMMARY=$(gum input --value "$TYPE$SCOPE: " --placeholder "short summary")
BODY=$(gum write --placeholder "longer description (ctrl+d to skip)" --height 6)

gum confirm "Commit?" || exit 0
git commit -m "$SUMMARY" ${BODY:+-m "$BODY"}
```

### 2. interactive branch cleanup

```bash
#!/bin/bash
set -e

# pick branches to delete
BRANCHES=$(git branch | cut -c 3- | gum choose --no-limit --header "Branches to delete")
[ -z "$BRANCHES" ] && exit 0

gum style --foreground 196 --bold "Will delete:"
echo "$BRANCHES" | gum format

gum confirm "Delete these branches?" --affirmative "Delete" --negative "Cancel" || exit 0

echo "$BRANCHES" | while IFS= read -r branch; do
  gum spin --title "Deleting $branch..." -- git branch -D "$branch"
  gum log --level info "Deleted" branch "$branch"
done
```

### 3. deploy script with env selection

```bash
#!/bin/bash
set -e

ENV=$(gum choose "staging" "production" --header "Deploy target")
TAG=$(git tag --sort=-v:refname | gum filter --placeholder "pick version tag...")

gum style \
  --border rounded --border-foreground 214 \
  --padding "1 3" --margin "1" \
  "Deploy $TAG to $ENV?"

gum confirm "Proceed?" --default=false || exit 0

gum spin --title "Deploying $TAG to $ENV..." --show-error -- \
  ./deploy.sh "$ENV" "$TAG"

gum style --foreground 82 --bold "Deployed $TAG to $ENV"
```

### 4. quick file notes launcher

```bash
#!/bin/bash
# browse vault notes, open selected in editor
VAULT="$HOME/notes"

FILE=$(find "$VAULT" -name "*.md" | sed "s|$VAULT/||" \
  | gum filter --placeholder "search notes..." --height 20)

[ -z "$FILE" ] && exit 0
$EDITOR "$VAULT/$FILE"
```

### 5. package manager TUI

```bash
#!/bin/bash
set -e

ACTION=$(gum choose "install" "remove" "update" --header "Package action")

case "$ACTION" in
  install)
    PKG=$(gum input --placeholder "package name")
    gum spin --title "Installing $PKG..." -- brew install "$PKG"
    ;;
  remove)
    PKGS=$(brew list | gum choose --no-limit --header "Select packages to remove")
    [ -z "$PKGS" ] && exit 0
    gum confirm "Remove selected?" || exit 0
    echo "$PKGS" | xargs gum spin --title "Removing..." -- brew uninstall
    ;;
  update)
    gum spin --title "Updating Homebrew..." --show-error -- brew update
    gum spin --title "Upgrading packages..." --show-output -- brew upgrade
    ;;
esac

gum log --level info "Done" action "$ACTION"
```

## Styling

Colors accept ANSI 256 codes (`212`) or hex (`#FF79C6`). Padding/margin use CSS shorthand: `"1 2"` = top/bottom 1, left/right 2.

```bash
# banner helper pattern
banner() {
  gum style \
    --foreground 212 --border-foreground 212 \
    --border double --align center \
    --padding "1 4" --margin "1 2" \
    "$@"
}
banner "Build Complete" "v1.4.2"

# side by side layout
LEFT=$(gum style --border rounded --padding "0 3" --foreground 82 "OK")
RIGHT=$(gum style --border rounded --padding "0 3" --foreground 196 "ERR")
gum join "$LEFT" "$RIGHT"
```

Style env vars use `GUM_<CMD>_<FLAG>` format. Set in shell profile for persistent defaults:

```bash
export GUM_CHOOSE_CURSOR_FOREGROUND="#FF79C6"
export GUM_INPUT_PLACEHOLDER="..."
export GUM_SPIN_SPINNER="dot"
```

## Common Mistakes

- **capturing output**: gum writes the prompt to stderr, result to stdout. `VAL=$(gum input)` works correctly.
- **spin command separator**: `--` is required before the command: `gum spin --title "..." -- npm install`. Without it gum parses your command as its own flags.
- **confirm exit code**: `gum confirm` returns 0 for yes, 1 for no. Use `&&`/`||` not `if [ $? -eq 0 ]` - both work but `&&` is idiomatic.
- **join with newlines**: always quote `$(gum style ...)` in join args or newlines collapse: `gum join "$A" "$B"` not `gum join $A $B`.
- **filter with no match**: by default `--strict` is on - filter returns nothing if no match. Use `--no-strict` to return the query string instead.
- **choose vs filter**: `choose` = static list, cursor navigation. `filter` = fuzzy search while typing. Use filter for long lists.
- **multi-select output**: each selection on its own line. Iterate with `while IFS= read -r item; do ... done <<< "$SELECTION"`.

## Checklist

- [ ] capture interactive output with `$()`, not redirect
- [ ] add `-- command` separator for `gum spin`
- [ ] handle empty selection (`[ -z "$VAR" ] && exit 0`)
- [ ] use `--no-limit` for multi-select, iterate output line by line
- [ ] use `--default=false` on destructive confirms
- [ ] quote `$(gum style ...)` when passing to `gum join`
- [ ] set `--timeout` for unattended or CI-adjacent scripts
- [ ] test ctrl+c behavior - gum exits non-zero, handle with `set -e` or explicit checks
