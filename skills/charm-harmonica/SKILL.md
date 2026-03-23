---
name: charm-harmonica
description: "Physics-based animation for Go TUIs - damped spring oscillator and projectile motion. Use when adding spring animations, physics-based motion, or smooth transitions to Go terminal apps. No Ease function exists in this library."
---

# charm-harmonica

Physics-based animation primitives: spring (damped harmonic oscillator) and projectile motion.

## Quick Start

```go
import "github.com/charmbracelet/harmonica"

// Create spring once - expensive coefficients computed here
spring := harmonica.NewSpring(harmonica.FPS(60), 6.0, 0.5)

// Per-frame state - YOU own these
var pos, vel float64

// Each frame:
pos, vel = spring.Update(pos, vel, targetPos)
```

## Core API

### FPS(n int) float64

Converts a frame rate to a delta time in seconds. Pass directly to `NewSpring` or `NewProjectile`.

```go
dt := harmonica.FPS(60) // 0.01666...
```

Use the engine's actual delta time instead when available (e.g. real elapsed time between frames).

### NewSpring(deltaTime, angularFrequency, dampingRatio float64) Spring

Pre-computes spring coefficients. Call once, reuse across frames.

| Parameter | Effect |
|-----------|--------|
| `deltaTime` | Frame duration in seconds - use `FPS(n)` |
| `angularFrequency` | Speed of motion. Typical range: 1-20 |
| `dampingRatio` | Oscillation behavior |

**Damping ratio guide:**
- `< 1.0` - under-damped, overshoots and oscillates
- `= 1.0` - critically damped, fastest without overshoot
- `> 1.0` - over-damped, slow and sluggish

Practical starting points: frequency `6.0`, damping `0.5` (smooth). For bouncy: frequency `8.0`, damping `0.15`. For snappy: frequency `12.0`, damping `1.0`.

### Spring.Update(pos, vel, target float64) (newPos, newVel float64)

Advances one frame. Returns new position and velocity - always capture both.

```go
// One spring can drive multiple independent axes
x, xVel = spring.Update(x, xVel, targetX)
y, yVel = spring.Update(y, yVel, targetY)
radius, radiusVel = spring.Update(radius, radiusVel, targetRadius)
```

### NewProjectile(deltaTime float64, pos Point, vel, acc Vector) *Projectile

Kinematic projectile - no spring, just position + velocity + acceleration per frame.

```go
p := harmonica.NewProjectile(
    harmonica.FPS(60),
    harmonica.Point{X: 0, Y: 0, Z: 0},
    harmonica.Vector{X: 5, Y: 0, Z: 0},   // initial velocity
    harmonica.TerminalGravity,              // acceleration: {0, 9.81, 0}
)

// Each frame:
pos := p.Update() // returns harmonica.Point
```

**Gravity constants:**
- `harmonica.Gravity` - `{0, -9.81, 0}` - origin bottom-left
- `harmonica.TerminalGravity` - `{0, 9.81, 0}` - origin top-left (standard TUI)

**Projectile accessors:** `p.Position()`, `p.Velocity()`, `p.Acceleration()`

### No Ease Function

harmonica does not have an Ease function. It has Spring and Projectile only.

## bubbletea Integration Pattern

The canonical pattern uses a `frameMsg` sentinel type and `tea.Tick` to drive the loop.

```go
const fps = 60

type frameMsg time.Time

// Schedules the next frame tick
func animate() tea.Cmd {
    return tea.Tick(time.Second/fps, func(t time.Time) tea.Msg {
        return frameMsg(t)
    })
}

type model struct {
    x      float64
    xVel   float64
    spring harmonica.Spring
}

func (m model) Init() tea.Cmd {
    return animate() // kick off the loop
}

func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg.(type) {
    case frameMsg:
        const target = 60.0
        m.x, m.xVel = m.spring.Update(m.x, m.xVel, target)

        // Stop ticking when close enough
        if math.Abs(m.x-target) < 0.01 {
            return m, nil
        }

        return m, animate() // schedule next frame
    }
    return m, nil
}

func main() {
    m := model{
        spring: harmonica.NewSpring(harmonica.FPS(fps), 7.0, 0.15),
    }
    tea.NewProgram(m).Run()
}
```

### Changing spring parameters mid-animation

When frequency or damping changes, call `NewSpring` again with the same `deltaTime`. The current `pos` and `vel` carry over - do not reset them.

```go
// User changed settings - recompute spring, keep state
spring = harmonica.NewSpring(harmonica.FPS(fps), newFreq, newDamp)
// x, xVel unchanged - animation continues smoothly from current state
```

### Stopping the animation loop

Return `nil` (no command) instead of `animate()` to stop. Resume by returning `animate()` again on the next relevant message.

## Common Mistakes

**Forgetting to capture velocity.** `spring.Update` returns two values. Discarding the velocity breaks the simulation on the next frame.
```go
// wrong
pos, _ = spring.Update(pos, vel, target)

// right
pos, vel = spring.Update(pos, vel, target)
```

**Creating Spring inside the update loop.** `NewSpring` is expensive - it computes trig/exp coefficients. Create it once in `main` or `Init`, store it on the model.

**Using `Gravity` instead of `TerminalGravity` in TUIs.** TUI coordinate systems have Y increasing downward. Use `TerminalGravity` (`{0, 9.81, 0}`) so things fall down the screen, not up.

**Not passing real delta time in non-fixed-FPS contexts.** In game loops with variable frame time, pass the actual `time.Since(last).Seconds()` to `NewSpring` each frame instead of `FPS(60)`. Recreating the spring with the real dt each frame is correct and expected.

**Calling `animate()` unconditionally.** Always check if the animation has converged before scheduling the next frame, or it runs forever at 60fps.

## Checklist

- [ ] `NewSpring` called once, stored on model struct
- [ ] Both return values from `Update` captured (`pos, vel = ...`)
- [ ] `animate()` returns `nil` when animation is done (convergence check)
- [ ] Using `TerminalGravity` for TUI projectiles (Y-down coordinate space)
- [ ] `frameMsg` type defined as `type frameMsg time.Time`
- [ ] Spring recomputed (not state reset) when parameters change at runtime
