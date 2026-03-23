---
name: charm-pop
description: "Send emails from the terminal with pop - TUI and CLI modes, SMTP and Resend support, attachments. Use when sending email from terminal, pop, CLI email, or piping email content from shell scripts."
---

# charm-pop

CLI and TUI tool for sending emails from the terminal. Requires either a `RESEND_API_KEY` or SMTP config.

## Launch

```bash
pop          # open TUI
```

## CLI Usage

```bash
pop < message.md \
    --from "me@example.com" \
    --to "you@example.com" \
    --subject "Hello" \
    --attach invoice.pdf

pop --preview    # preview before sending
```

Body is read from stdin. `--preview` opens TUI for review before send.

## Auth: Resend

```bash
export RESEND_API_KEY=re_xxxx
```

Get key at https://resend.com/api-keys. No custom domain: use `onboarding@resend.dev` as sender.

## Auth: SMTP

```bash
export POP_SMTP_HOST=smtp.gmail.com
export POP_SMTP_PORT=587
export POP_SMTP_USERNAME=you@gmail.com
export POP_SMTP_PASSWORD=hunter2
```

## Env Defaults

```bash
export POP_FROM=you@example.com
export POP_SIGNATURE="Sent with Pop!"
```

Saves typing `--from` every time.

## Attachments

```bash
pop --attach file.pdf --attach image.png < body.txt
```

Multiple `--attach` flags supported.

## Pipelines

```bash
# AI-written body via mods
pop <<< "$(mods -f 'Write a status update')" \
    --subject "Weekly update" --preview

# Pick sender with gum
pop --from $(gum choose "a@x.com" "b@x.com") \
    --to $(gum filter < contacts.txt)

# Generate and send invoice
invoice generate --item "Work" --rate 100 --output inv.pdf
pop --attach inv.pdf --body "Invoice attached."
```

## Flags Reference

| Flag | Description |
|------|-------------|
| `--from` | Sender address |
| `--to` | Recipient (repeatable) |
| `--subject` | Email subject |
| `--body` | Body text (or use stdin) |
| `--attach` | File path to attach (repeatable) |
| `--preview` | Open TUI preview before sending |

## Install

```bash
brew install pop                      # macOS/Linux
go install github.com/charmbracelet/pop@latest
```
