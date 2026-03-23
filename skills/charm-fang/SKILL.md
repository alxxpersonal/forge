---
name: charm-fang
description: "Wrap Cobra with styled help, error output, auto versioning, and manpage generation via fang. Use when building Go CLIs with fang, styled Cobra help, or adding lipgloss-rendered help pages to a Go CLI."
---

# charm-fang

Fang is NOT a Cobra replacement - it wraps Cobra. You still write `*cobra.Command` structs exactly as you would with plain Cobra. Fang intercepts execution to add:

- styled help/usage output (lipgloss-rendered)
- styled error output with an `ERROR` header block
- auto `--version` from build info or a string you provide
- hidden `man` subcommand (manpage via mango/roff)
- `completion` subcommand for shell completions
- `SilenceUsage = true` by default (no help dump on error)
- signal handling via `WithNotifySignal`

## Install

```bash
go get charm.land/fang/v2
```

Import path (v2, note the vanity domain):

```go
import "charm.land/fang/v2"
```

## Minimal App

```go
package main

import (
    "context"
    "os"

    "charm.land/fang/v2"
    "github.com/spf13/cobra"
)

func main() {
    root := &cobra.Command{
        Use:   "myapp",
        Short: "Does something useful",
    }
    if err := fang.Execute(context.Background(), root); err != nil {
        os.Exit(1)
    }
}
```

That's the full swap from `root.Execute()` to `fang.Execute()`.

## Complete CLI Skeleton

```go
package main

import (
    "context"
    "os"

    "charm.land/fang/v2"
    "github.com/spf13/cobra"
)

func main() {
    var name string
    var verbose bool

    root := &cobra.Command{
        Use:     "myapp [flags]",
        Short:   "One-line description",
        Long:    "Longer description shown in full help.",
        Version: "1.0.0", // overridden by fang if you use WithVersion
        Example: `
  # basic usage
  myapp --name alice

  # with subcommand
  myapp greet --name alice`,
        RunE: func(cmd *cobra.Command, args []string) error {
            cmd.Printf("Hello, %s\n", name)
            return nil
        },
    }

    // persistent flags available to all subcommands
    root.PersistentFlags().BoolVarP(&verbose, "verbose", "v", false, "verbose output")
    // local flags for root only
    root.Flags().StringVar(&name, "name", "world", "name to greet")

    // subcommand
    greet := &cobra.Command{
        Use:   "greet",
        Short: "Greet someone",
        RunE: func(cmd *cobra.Command, args []string) error {
            cmd.Println("greet subcommand")
            return nil
        },
    }
    root.AddCommand(greet)

    if err := fang.Execute(
        context.Background(),
        root,
        fang.WithVersion("1.2.3"),
        fang.WithCommit("abc1234"),
        fang.WithNotifySignal(os.Interrupt),
    ); err != nil {
        os.Exit(1)
    }
}
```

## Options Reference

| Option | Effect |
|--------|--------|
| `WithVersion(v string)` | Sets version string shown by `--version` |
| `WithCommit(sha string)` | Appends short commit SHA to version |
| `WithoutVersion()` | Disables `-v`/`--version` entirely |
| `WithoutCompletions()` | Removes the `completion` subcommand |
| `WithoutManpage()` | Removes the hidden `man` subcommand |
| `WithNotifySignal(signals...)` | Cancels context on given OS signals |
| `WithColorSchemeFunc(fn)` | Custom theme, light/dark-adaptive |
| `WithErrorHandler(fn)` | Custom error rendering |

If no `WithVersion` is passed, fang reads `debug.ReadBuildInfo()` automatically (works when installed via `go install`).

## Custom Theme

```go
import "charm.land/lipgloss/v2"

fang.Execute(ctx, root, fang.WithColorSchemeFunc(func(ld lipgloss.LightDarkFunc) fang.ColorScheme {
    return fang.ColorScheme{
        Title:   ld(lipgloss.Color("#FF6B6B"), lipgloss.Color("#4ECDC4")),
        Flag:    lipgloss.Color("#0CB37F"),
        Command: lipgloss.Color("#A550DF"),
        // ... other fields
    }
}))
```

`LightDarkFunc` lets you return different colors based on terminal background. Use `fang.DefaultColorScheme` or `fang.AnsiColorScheme` as a reference.

## Differences from Plain Cobra

| Cobra | Fang |
|-------|------|
| `root.Execute()` | `fang.Execute(ctx, root, opts...)` |
| plain text help | lipgloss-styled help |
| errors printed raw | styled error block |
| manual signal handling | `WithNotifySignal` |
| manual manpage setup | built-in hidden `man` cmd |
| `SilenceUsage` off by default | on by default |
| no version auto-detect | reads build info automatically |

## Flag Patterns (Cobra, unchanged)

```go
// string flag with short form
cmd.Flags().StringVarP(&val, "name", "n", "default", "description")

// persistent (inherited by subcommands)
cmd.PersistentFlags().BoolVar(&debug, "debug", false, "enable debug")

// hidden flag
cmd.Flags().String("internal", "", "internal use")
_ = cmd.Flags().MarkHidden("internal")

// required flag
cmd.Flags().String("config", "", "config file")
_ = cmd.MarkFlagRequired("config")
```

## Command Groups

```go
root.AddGroup(&cobra.Group{
    ID:    "core",
    Title: "Core Commands",
})
sub.GroupID = "core"
```

Groups show as sections in fang's styled help output.

## Context Usage

Subcommands get the context fang passes (with signal cancellation if configured):

```go
RunE: func(cmd *cobra.Command, args []string) error {
    select {
    case <-time.After(5 * time.Second):
        return nil
    case <-cmd.Context().Done():
        return cmd.Context().Err()
    }
},
```

## go.mod

```
require (
    charm.land/fang/v2 v2.x.x
    github.com/spf13/cobra v1.x.x
)
```
