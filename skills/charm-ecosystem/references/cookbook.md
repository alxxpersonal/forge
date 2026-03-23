# Integration Cookbook

Working multi-library code examples. Import paths verified against go.mod.

## 1. Standard TUI App (bubbletea + lipgloss + bubbles)

A list with styled header and footer. The three core libraries working together.

```go
package main

import (
	"fmt"
	"os"

	tea "charm.land/bubbletea/v2"
	"charm.land/bubbles/v2/list"
	"charm.land/lipgloss/v2"
)

var (
	titleStyle = lipgloss.NewStyle().
		Bold(true).
		Foreground(lipgloss.Color("#FAFAFA")).
		Background(lipgloss.Color("#7D56F4")).
		Padding(0, 1)

	statusStyle = lipgloss.NewStyle().
		Foreground(lipgloss.Color("241"))
)

type item string

func (i item) FilterValue() string { return string(i) }
func (i item) Title() string       { return string(i) }
func (i item) Description() string { return "" }

type model struct {
	list   list.Model
	width  int
	height int
}

func initialModel() model {
	items := []list.Item{
		item("Buy groceries"),
		item("Clean the house"),
		item("Write some Go"),
		item("Touch grass"),
	}
	l := list.New(items, list.NewDefaultDelegate(), 40, 15)
	l.Title = "Tasks"
	return model{list: l}
}

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyPressMsg:
		if msg.String() == "q" || msg.String() == "ctrl+c" {
			return m, tea.Quit
		}
	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.height = msg.Height
		m.list.SetSize(msg.Width, msg.Height-2)
	}

	var cmd tea.Cmd
	m.list, cmd = m.list.Update(msg)
	return m, cmd
}

func (m model) View() tea.View {
	header := titleStyle.Render("My App")
	body := m.list.View()
	footer := statusStyle.Render("q to quit")
	v := tea.NewView(fmt.Sprintf("%s\n%s\n%s", header, body, footer))
	v.AltScreen = true
	return v
}

func main() {
	if _, err := tea.NewProgram(initialModel()).Run(); err != nil {
		fmt.Println("Error:", err)
		os.Exit(1)
	}
}
```

## 2. Forms Embedded in TUI (bubbletea + huh)

A bubbletea app that shows a huh form, then displays results.

```go
package main

import (
	"fmt"
	"os"

	tea "charm.land/bubbletea/v2"
	"charm.land/huh/v2"
	"charm.land/lipgloss/v2"
)

type state int

const (
	stateForm state = iota
	stateDone
)

type model struct {
	form  *huh.Form
	state state
	name  string
	lang  string
}

func newModel() model {
	var name, lang string

	form := huh.NewForm(
		huh.NewGroup(
			huh.NewInput().
				Key("name").
				Title("Your name").
				Value(&name),
			huh.NewSelect[string]().
				Key("lang").
				Title("Favorite language").
				Options(huh.NewOptions("Go", "Rust", "Python", "TypeScript")...).
				Value(&lang),
		),
	)

	return model{form: form, name: name, lang: lang}
}

func (m model) Init() tea.Cmd {
	return m.form.Init()
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyPressMsg:
		if msg.String() == "ctrl+c" {
			return m, tea.Quit
		}
	}

	form, cmd := m.form.Update(msg)
	if f, ok := form.(*huh.Form); ok {
		m.form = f
	}

	if m.form.State == huh.StateCompleted {
		m.state = stateDone
		m.name = m.form.GetString("name")
		m.lang = m.form.GetString("lang")
		return m, tea.Quit
	}

	return m, cmd
}

func (m model) View() tea.View {
	if m.state == stateDone {
		result := lipgloss.NewStyle().Bold(true).Foreground(lipgloss.Color("212")).
			Render(fmt.Sprintf("%s loves %s!", m.name, m.lang))
		return tea.NewView(result + "\n")
	}
	return tea.NewView(m.form.View())
}

func main() {
	if _, err := tea.NewProgram(newModel()).Run(); err != nil {
		fmt.Println("Error:", err)
		os.Exit(1)
	}
}
```

## 3. Scrollable Markdown Viewer (bubbletea + glamour + viewport)

Renders markdown with glamour, displays in a scrollable viewport.

```go
package main

import (
	"fmt"
	"os"

	tea "charm.land/bubbletea/v2"
	"charm.land/bubbles/v2/viewport"
	"charm.land/glamour/v2"
	"charm.land/lipgloss/v2"
)

const sampleMD = `# Welcome

This is a **scrollable** markdown viewer built with:

- Bubble Tea (framework)
- Glamour (markdown rendering)
- Bubbles viewport (scrolling)

## Features

Scroll with j/k or arrow keys. Press q to quit.

> "The terminal is the ultimate UI." - someone, probably

## Code Example

` + "```go" + `
fmt.Println("Hello from glamour!")
` + "```" + `

More content here to make it scrollable...
`

type model struct {
	viewport    viewport.Model
	rawMarkdown string
	ready       bool
}

func (m model) Init() tea.Cmd { return nil }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyPressMsg:
		if msg.String() == "q" || msg.String() == "ctrl+c" {
			return m, tea.Quit
		}
	case tea.WindowSizeMsg:
		if !m.ready {
			m.viewport = viewport.New(
				viewport.WithWidth(msg.Width),
				viewport.WithHeight(msg.Height-2),
			)
			m.ready = true
		} else {
			m.viewport.SetWidth(msg.Width)
			m.viewport.SetHeight(msg.Height - 2)
		}

		r, _ := glamour.NewTermRenderer(
			glamour.WithStandardStyle("dark"),
			glamour.WithWordWrap(msg.Width-4),
		)
		rendered, _ := r.Render(m.rawMarkdown)
		m.viewport.SetContent(rendered)
	}

	var cmd tea.Cmd
	m.viewport, cmd = m.viewport.Update(msg)
	return m, cmd
}

func (m model) View() tea.View {
	if !m.ready {
		return tea.NewView("Loading...")
	}

	header := lipgloss.NewStyle().Bold(true).
		Foreground(lipgloss.Color("205")).
		Render("Markdown Viewer")
	footer := lipgloss.NewStyle().
		Foreground(lipgloss.Color("241")).
		Render(fmt.Sprintf("scroll: %d%%", int(m.viewport.ScrollPercent()*100)))

	v := tea.NewView(header + "\n" + m.viewport.View() + "\n" + footer)
	v.AltScreen = true
	return v
}

func main() {
	m := model{rawMarkdown: sampleMD}
	if _, err := tea.NewProgram(m).Run(); err != nil {
		fmt.Println("Error:", err)
		os.Exit(1)
	}
}
```

## 4. Animated Transitions (bubbletea + harmonica)

Spring-animated horizontal bar that follows a target position.

```go
package main

import (
	"fmt"
	"math"
	"os"
	"strings"
	"time"

	tea "charm.land/bubbletea/v2"
	"charm.land/lipgloss/v2"
	"github.com/charmbracelet/harmonica"
)

const fps = 60

type frameMsg time.Time

func animate() tea.Cmd {
	return tea.Tick(time.Second/fps, func(t time.Time) tea.Msg {
		return frameMsg(t)
	})
}

type model struct {
	spring harmonica.Spring
	x      float64
	xVel   float64
	target float64
	width  int
}

func (m model) Init() tea.Cmd { return animate() }

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
	switch msg := msg.(type) {
	case tea.KeyPressMsg:
		switch msg.String() {
		case "q", "ctrl+c":
			return m, tea.Quit
		case "left":
			m.target = float64(m.width) * 0.2
		case "right":
			m.target = float64(m.width) * 0.8
		case "space":
			m.target = float64(m.width) * 0.5
		}
	case tea.WindowSizeMsg:
		m.width = msg.Width
		m.target = float64(msg.Width) * 0.5
	case frameMsg:
		m.x, m.xVel = m.spring.Update(m.x, m.xVel, m.target)

		if math.Abs(m.x-m.target) < 0.01 && math.Abs(m.xVel) < 0.01 {
			return m, nil // stop animating when settled
		}
		return m, animate()
	}
	return m, nil
}

func (m model) View() tea.View {
	if m.width == 0 {
		return tea.NewView("")
	}

	pos := int(m.x)
	if pos < 0 {
		pos = 0
	}
	if pos >= m.width {
		pos = m.width - 1
	}

	bar := lipgloss.NewStyle().
		Foreground(lipgloss.Color("205")).
		Bold(true).
		Render("@")

	line := strings.Repeat(" ", pos) + bar
	help := lipgloss.NewStyle().Foreground(lipgloss.Color("241")).
		Render("\nleft/right/space to move, q to quit")

	return tea.NewView(line + help)
}

func main() {
	m := model{
		spring: harmonica.NewSpring(harmonica.FPS(fps), 7.0, 0.35),
	}
	if _, err := tea.NewProgram(m).Run(); err != nil {
		fmt.Println("Error:", err)
		os.Exit(1)
	}
}
```

## 5. Scripted Demo Recording (gum + vhs)

A VHS tape that demos a gum-powered script.

```tape
Output demo.gif

Set FontSize 14
Set Width 1000
Set Height 500
Set Theme "Catppuccin Frappe"
Set WindowBar Colorful
Set TypingSpeed 0.06
Set Framerate 30

Require gum

# Show a styled banner
Type `gum style --foreground 212 --border rounded --padding "1 2" "Deploy Helper"`
Enter
Sleep 1s

# Choose environment
Type `ENV=$(gum choose "staging" "production")`
Enter
Sleep 500ms
Down
Sleep 300ms
Enter
Sleep 1s

# Confirm
Type `gum confirm "Deploy to $ENV?" && echo "Deploying..."  || echo "Cancelled"`
Enter
Sleep 500ms
Enter
Sleep 2s
```
