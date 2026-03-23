---
name: charm-vhs
description: "Record terminal sessions as GIF/MP4/WebM from declarative .tape scripts with VHS. Use when creating terminal demos, recording CLI sessions, VHS tape files, or generating terminal GIFs."
---

# charm-vhs

VHS records terminal sessions from `.tape` scripts. Requires `ttyd` and `ffmpeg` on PATH.

```sh
brew install vhs       # also installs deps on macOS
vhs demo.tape          # run a tape file
vhs new demo.tape      # scaffold a new tape
vhs record > out.tape  # record interactively, then exit
vhs publish demo.gif   # host on vhs.charm.sh
```

## Tape File Structure

Order matters: `Output` and `Set` must come before action commands. `Require` goes at the very top.

```
Require <program>   # fail early if missing from PATH
Output <path>       # .gif / .mp4 / .webm / .ascii / frames/
Set <Setting> Value # terminal config (must precede actions)
<actions>           # Type, Enter, Sleep, etc.
```

## Output Formats

```elixir
Output demo.gif
Output demo.mp4
Output demo.webm
Output frames/        # PNG sequence
Output golden.ascii   # for CI golden file diffing
```

Multiple `Output` lines are fine - all render in one run.

## Settings Reference

```elixir
Set Shell "zsh"
Set FontSize 14
Set FontFamily "JetBrains Mono"
Set Width 1200
Set Height 600
Set Padding 20
Set Margin 40
Set MarginFill "#6B50FF"
Set BorderRadius 10
Set WindowBar Colorful        # Colorful, ColorfulRight, Rings, RingsRight
Set Theme "Catppuccin Frappe" # run `vhs themes` for full list
Set TypingSpeed 0.05          # seconds per keypress
Set Framerate 60
Set PlaybackSpeed 1.0
Set LoopOffset 50%            # where GIF loop starts
Set CursorBlink false
```

`TypingSpeed` is the only setting that can change mid-tape. All others are ignored after the first action command.

## Action Commands

### Typing + input

```elixir
Type "git status"           # types the string
Type@500ms "slowly..."      # override typing speed for this line
Type `VAR="backtick escapes quotes"`
Enter
Enter 2                     # press N times
Tab
Tab@200ms 3
Backspace 5
Space 2
```

### Navigation

```elixir
Up / Down / Left / Right    # arrow keys
Up 3                        # repeat N times
PageUp / PageDown
ScrollUp 10
ScrollDown@100ms 5
Ctrl+C
Ctrl+Alt+Delete
```

### Timing

```elixir
Sleep 500ms
Sleep 2s
Sleep 0.5       # seconds (float ok)
Wait /regex/    # wait until last line matches (default timeout 15s)
Wait+Screen /regex/         # check whole screen
Wait+Line@10ms /regex/      # poll every 10ms
```

`Wait` is better than `Sleep` for commands with unpredictable runtime (builds, network calls).

### Hide / Show

```elixir
Hide
Type "setup stuff not shown in recording"
Enter
Wait /\$/
Show
```

Use `Hide`/`Show` to run setup or cleanup without polluting the demo.

### Other

```elixir
Screenshot path/out.png     # capture current frame as PNG
Copy "text"                 # put text on clipboard
Paste                       # paste clipboard
Env KEY "value"             # set env var
Source config.tape          # include another tape
```

## Example Tapes

### 1. Simple CLI demo

```elixir
Output demo.gif

Set FontSize 14
Set Width 900
Set Height 400
Set Theme "Catppuccin Frappe"
Set WindowBar Colorful
Set TypingSpeed 0.05

Type "ls -la"
Sleep 300ms
Enter
Sleep 2s
```

### 2. Build + run with hidden setup

```elixir
Output demo.gif

Set FontSize 13
Set Width 1200
Set Height 600
Set Theme "Dracula"

Require go

Hide
Type "go build -o myapp . && clear"
Enter
Wait /\$/
Show

Type "./myapp --help"
Sleep 200ms
Enter
Sleep 3s

Hide
Type "rm myapp"
Enter
```

### 3. Interactive TUI demo with Wait

```elixir
Output tui-demo.gif
Output tui-demo.mp4

Set FontSize 14
Set Width 1000
Set Height 500
Set WindowBar Rings
Set Margin 30
Set MarginFill "#1a1b26"
Set BorderRadius 8
Set TypingSpeed 0.07

Require gum

Type "gum choose 'Option A' 'Option B' 'Option C'"
Enter
Sleep 500ms
Down
Sleep 300ms
Down
Sleep 500ms
Enter
Sleep 2s
```

## CI Integration

Use the [vhs-action](https://github.com/charmbracelet/vhs-action) GitHub Action to regenerate GIFs on push.

For integration testing, output `.ascii` and commit as golden files - diff them in CI to catch terminal output regressions.

```elixir
Output golden.ascii
```

## Tips

- `vhs record > cassette.tape` then edit the generated tape to add `Set` blocks and clean up timing
- Use `Source` to share a `config.tape` with common `Set` defaults across multiple tapes
- `Wait` beats `Sleep` for anything async - no need to guess how long a build takes
- `LoopOffset` makes the GIF preview frame more interesting than frame 0
- `vhs themes` lists all built-in theme names
