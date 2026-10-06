# Facade Design Pattern

The **Facade pattern** is a structural design pattern that provides a simple, high-level API for a complex subsystem.

Instead of making the client coordinate several components, the facade handles the common workflow through one convenient method.

**Simple example:**

To watch a movie, you need to:

- Dim the lights.
- Turn on the projector and enable widescreen mode.
- Turn on the amplifier and set its volume.
- Turn on the DVD player and play the movie.

A facade lets the client perform this workflow with:

```go
homeTheater.WatchMovie("Inception")
```

In Go, a facade can be a **struct with methods**. It does not require a Go interface declaration.

## When to Use

- When clients must coordinate several subsystem components.
- When common workflows involve multiple low-level operations.
- When you want to provide an API that is easier to use.
- When a legacy subsystem is too complex for most clients to use directly.

**Backend example:**

An order subsystem may contain inventory, payment, and order-storage components.

A facade can expose `PlaceOrder()` to coordinate them, so API handlers do not repeat the same sequence of calls.

However, the facade still needs explicit handling for failures. It does not automatically undo a payment if saving the order fails.

## Components

| Component | Responsibility | Home theater example |
|---|---|---|
| **Facade** | Exposes convenient operations and coordinates subsystem calls. | `HomeTheaterFacade`. |
| **Subsystem Classes** | Implement individual capabilities. In Go, these can be structs with methods. | `Amplifier`, `DVDPlayer`, `Projector`, and `Lights`. |
| **Client** | Uses the facade to perform a common workflow. | `main()`, which calls `WatchMovie()`. |

The subsystem components do not need to know that the facade exists.

A facade also does not have to block direct subsystem access. Specialized clients may still use individual components when needed.

## Working

1. The client calls a high-level facade method.
2. The facade coordinates the required subsystem operations.
3. Each subsystem performs its own work.
4. The facade returns a result or error when appropriate.

In the home theater example, `WatchMovie()` prepares the equipment in order and starts playback.

The facade **coordinates the workflow**; each component remains responsible for its own operation.

## Pros

- **Simplifies usage:** Turns a sequence of calls into one convenient operation.
- **Reduces client dependencies:** Clients using the facade need less knowledge of subsystem details.
- **Centralizes coordination:** Keeps common operation sequences in one place.
- **Improves readability:** `WatchMovie()` communicates the client's intent clearly.
- **Improves maintainability:** Internal changes can be handled inside the facade while preserving its public API and behavior.

## Cons

- **Risk of a God Object:** A facade becomes difficult to maintain if it handles too many unrelated workflows.
- **Limited exposed functionality:** Specialized subsystem features may not be available through the facade.
- **Additional abstraction:** Adds unnecessary structure when the subsystem is already simple.
- **Reduced flexibility if enforced everywhere:** Requiring every client to use the facade can restrict fine-grained control.

Keep each facade focused on a related set of operations. It should coordinate components rather than absorb all their responsibilities.

## Sample Implementation in Golang

We will build a home theater facade that coordinates four components.

```go
package main

import "fmt"

// Subsystem: amplifier.
type Amplifier struct{}

func (a *Amplifier) On() {
	fmt.Println("Amplifier on")
}

func (a *Amplifier) SetVolume(level int) {
	fmt.Printf("Volume set to %d\n", level)
}

// Subsystem: DVD player.
type DVDPlayer struct{}

func (d *DVDPlayer) On() {
	fmt.Println("DVD player on")
}

func (d *DVDPlayer) Play(movie string) {
	fmt.Printf("Playing movie: %s\n", movie)
}

// Subsystem: projector.
type Projector struct{}

func (p *Projector) On() {
	fmt.Println("Projector on")
}

func (p *Projector) WideScreenMode() {
	fmt.Println("Projector in widescreen mode")
}

// Subsystem: lights.
type Lights struct{}

func (l *Lights) Dim(level int) {
	fmt.Printf("Lights dimmed to %d%%\n", level)
}

// Facade: coordinates the subsystem components.
type HomeTheaterFacade struct {
	amp       *Amplifier
	dvd       *DVDPlayer
	projector *Projector
	lights    *Lights
}

func NewHomeTheaterFacade(
	amp *Amplifier,
	dvd *DVDPlayer,
	projector *Projector,
	lights *Lights,
) *HomeTheaterFacade {
	return &HomeTheaterFacade{
		amp:       amp,
		dvd:       dvd,
		projector: projector,
		lights:    lights,
	}
}

// High-level operation exposed to the client.
func (h *HomeTheaterFacade) WatchMovie(movie string) {
	fmt.Println("Get ready to watch a movie...")

	h.lights.Dim(10)

	h.projector.On()
	h.projector.WideScreenMode()

	h.amp.On()
	h.amp.SetVolume(5)

	h.dvd.On()
	h.dvd.Play(movie)
}

// Client.
func main() {
	amp := &Amplifier{}
	dvd := &DVDPlayer{}
	projector := &Projector{}
	lights := &Lights{}

	homeTheater := NewHomeTheaterFacade(
		amp,
		dvd,
		projector,
		lights,
	)

	homeTheater.WatchMovie("Inception")
}
```

**Output:**

```text
Get ready to watch a movie...
Lights dimmed to 10%
Projector on
Projector in widescreen mode
Amplifier on
Volume set to 5
DVD player on
Playing movie: Inception
```

### Step 1: Define the Subsystems

Each component provides its own operations:

- `Amplifier`: turns on and sets volume.
- `DVDPlayer`: turns on and plays a movie.
- `Projector`: turns on and enables widescreen mode.
- `Lights`: adjusts brightness.

These components work independently of the facade.

### Step 2: Store the Components in the Facade

```go
type HomeTheaterFacade struct {
	amp       *Amplifier
	dvd       *DVDPlayer
	projector *Projector
	lights    *Lights
}
```

The facade holds references to the components it coordinates.

This is **composition**: one object contains and uses other objects.

The lowercase field names make them unexported, so another Go package cannot access them directly through the facade.

### Step 3: Supply Dependencies Through a Constructor

`NewHomeTheaterFacade()` receives the components and stores them.

The client supplies the dependencies instead of the facade creating them internally. This lets the client choose which component instances to use.

In Go, `New...` is a naming convention, not a special constructor keyword.

This example assumes all required dependencies are supplied.

### Step 4: Define the Common Workflow

```go
func (h *HomeTheaterFacade) WatchMovie(movie string) {
	fmt.Println("Get ready to watch a movie...")

	h.lights.Dim(10)

	h.projector.On()
	h.projector.WideScreenMode()

	h.amp.On()
	h.amp.SetVolume(5)

	h.dvd.On()
	h.dvd.Play(movie)
}
```

The method hides the preparation sequence from the client.

The settings—brightness of 10% and volume of 5—are defaults chosen for this workflow. The subsystem methods still perform the individual operations.

### Step 5: Use the Facade

```go
homeTheater.WatchMovie("Inception")
```

The client expresses what it wants to do without coordinating every component.

`main()` still creates and connects the components during setup. After that, movie playback uses the facade's simple API.

### Step 6: Understand Error Handling

The example only prints messages, so its methods do not return errors.

Real device or backend operations can fail. A production facade should handle those errors and decide whether earlier steps need cleanup or compensation.

For example, if playback fails after the amplifier turns on, the facade does not automatically turn it off. That behavior must be implemented explicitly.
