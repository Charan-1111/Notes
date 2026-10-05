# Bridge Design Pattern

The **Bridge pattern** is a structural design pattern that separates high-level functionality from its underlying implementation, allowing both to evolve independently.

It connects them through **composition**: the high-level object holds a reference to an implementation and delegates work to it.

**Simple example:**

We have two independent dimensions:

- **Remote types:** Basic remote and advanced remote.
- **TV brands:** Sony and Samsung.

Without this separation, we might create a separate type for every combination:

- Sony basic remote.
- Sony advanced remote.
- Samsung basic remote.
- Samsung advanced remote.

With Bridge, we define remote types and TV implementations separately. Any remote can work with any TV that satisfies the required interface.

**Adapter vs Bridge:**

- **Adapter:** Connects existing implementations with incompatible interfaces.
- **Bridge:** Organizes two independent dimensions so they can evolve separately.

In Go, we use **interfaces and composition** rather than class inheritance.

## When to Use

- When a design has two independent dimensions that may grow.
  - Example: Remote types and TV brands.
  - Example: Shapes and rendering technologies.
- When creating a separate type for every combination would duplicate code.
- When you need to select an implementation at runtime.

**Backend example:**

A reporting system has:

- **Report types:** Sales report and inventory report.
- **Output formats:** JSON and HTML.

Each report prepares its data and delegates formatting to an output implementation.

Adding an HTML formatter should not require creating a new type for every report.

> Using an interface alone does not necessarily make a design Bridge. The key is separating two dimensions that need to vary independently.

## Components

| Component | Responsibility | Remote and TV example |
|---|---|---|
| **Abstraction** | Provides high-level operations and holds an implementation reference. | `RemoteControl`. |
| **Refined Abstraction** | Adds high-level functionality while using the same implementation interface. | `AdvancedRemote`. |
| **Implementor** | Defines the operations underlying implementations must provide. | `TV` interface. |
| **Concrete Implementor** | Supplies a particular implementation. | `SonyTV` and `SamsungTV`. |

The abstraction and implementor interfaces do not have to be identical.

For example, an advanced remote might provide `WatchNews()`, which internally calls the TV's `On()` and `SetChannel()` methods.

## Working

1. The client calls an operation on a remote.
2. The remote holds a TV implementation through the `TV` interface.
3. The remote delegates the required operations to that TV.
4. The concrete TV performs them.

**Example:**

A basic remote connected to Sony calls `SonyTV.On()`.

The same remote type connected to Samsung calls `SamsungTV.On()`.

We can add:

- A new remote type without modifying the existing TV implementations.
- A new TV brand without modifying existing remote types.

This works as long as the TV implementations continue to satisfy the interface the remotes need.

## Pros

- **Separates responsibilities:** Remotes define control behavior; TVs implement device operations.
- **Avoids combination-specific types:** Reuses remote types across TV brands.
- **Supports independent extension:** Adds new remote types or TV implementations separately.
- **Provides runtime flexibility:** Selects the TV implementation when creating the remote.
- **Supports the Open–Closed Principle:** Introduces new implementations without changing existing remote logic.

## Cons

- **Adds structure:** Introduces separate abstraction and implementation types.
- **Requires design judgment:** The two independent dimensions must be identified clearly.
- **Adds indirection:** Understanding an operation may require following a delegated call.
- **Can be unnecessary:** A small system with one stable implementation may not benefit from this separation.

A simple delegated call usually has negligible performance overhead. The additional design complexity is often the more relevant trade-off.

## Sample Implementation in Golang

We will create basic and advanced remotes that can control either Sony or Samsung TVs.

```go
package main

import "fmt"

// Implementor: operations every supported TV must provide.
type TV interface {
	On()
	Off()
	SetChannel(channel int)
}

// Concrete implementor: Sony TV.
type SonyTV struct{}

func (s *SonyTV) On() {
	fmt.Println("Sony TV is ON")
}

func (s *SonyTV) Off() {
	fmt.Println("Sony TV is OFF")
}

func (s *SonyTV) SetChannel(channel int) {
	fmt.Printf("Sony TV set to channel %d\n", channel)
}

// Concrete implementor: Samsung TV.
type SamsungTV struct{}

func (s *SamsungTV) On() {
	fmt.Println("Samsung TV is ON")
}

func (s *SamsungTV) Off() {
	fmt.Println("Samsung TV is OFF")
}

func (s *SamsungTV) SetChannel(channel int) {
	fmt.Printf("Samsung TV set to channel %d\n", channel)
}

// Abstraction: holds a TV implementation.
type RemoteControl struct {
	tv TV
}

func (r *RemoteControl) On() {
	r.tv.On()
}

func (r *RemoteControl) Off() {
	r.tv.Off()
}

// Refined abstraction: reuses basic control behavior.
type AdvancedRemote struct {
	RemoteControl
}

func (a *AdvancedRemote) SetChannel(channel int) {
	a.tv.SetChannel(channel)
}

// Combines multiple implementation operations.
func (a *AdvancedRemote) WatchNews() {
	a.tv.On()
	a.tv.SetChannel(5)
}

// Client: chooses the remote and TV combinations.
func main() {
	sony := &SonyTV{}
	samsung := &SamsungTV{}

	basicRemote := &RemoteControl{
		tv: sony,
	}

	advancedRemote := &AdvancedRemote{
		RemoteControl: RemoteControl{
			tv: samsung,
		},
	}

	basicRemote.On()
	basicRemote.Off()

	advancedRemote.WatchNews()
	advancedRemote.SetChannel(10)
	advancedRemote.Off()
}
```

**Output:**

```text
Sony TV is ON
Sony TV is OFF
Samsung TV is ON
Samsung TV set to channel 5
Samsung TV set to channel 10
Samsung TV is OFF
```

### Step 1: Define the Implementor Interface

```go
type TV interface {
	On()
	Off()
	SetChannel(channel int)
}
```

This defines the operations remotes need from a TV.

Any Go type with matching methods satisfies this interface automatically.

### Step 2: Create Concrete Implementations

```go
type SonyTV struct{}
type SamsungTV struct{}
```

Both implement the methods required by `TV`.

Because their methods use pointer receivers, `*SonyTV` and `*SamsungTV` satisfy the interface.

The example prints messages; a real implementation could send commands through a device SDK or network connection.

### Step 3: Create the Abstraction

```go
type RemoteControl struct {
	tv TV
}
```

The remote stores a `TV` interface instead of depending on a particular brand.

This field is the **bridge** between remote behavior and TV implementation.

```go
func (r *RemoteControl) On() {
	r.tv.On()
}
```

The remote delegates the operation to whichever TV it contains.

### Step 4: Create the Refined Abstraction

```go
type AdvancedRemote struct {
	RemoteControl
}
```

`AdvancedRemote` embeds `RemoteControl`, allowing it to reuse the basic remote's methods.

**Go embedding is composition, not class inheritance.**

Calling:

```go
advancedRemote.Off()
```

uses the embedded `RemoteControl`'s `Off()` method.

The advanced remote adds more functionality:

```go
func (a *AdvancedRemote) WatchNews() {
	a.tv.On()
	a.tv.SetChannel(5)
}
```

One high-level operation combines two underlying TV operations.

This shows why the abstraction's methods do not have to match the implementor's methods exactly.

### Step 5: Choose the Combinations

```go
basicRemote := &RemoteControl{
	tv: sony,
}
```

This connects a basic remote to Sony.

```go
advancedRemote := &AdvancedRemote{
	RemoteControl: RemoteControl{
		tv: samsung,
	},
}
```

This connects an advanced remote to Samsung.

We could also connect the advanced remote to Sony by using `tv: sony`. No new remote type is required.

### Step 6: Understand Runtime Selection

The example chooses a TV when each remote is created.

Changing an existing remote's TV requires an explicit assignment or a method that updates the field. Bridge does not switch implementations automatically.

Also, the `tv` field must contain a valid implementation before calling remote methods. In this example, `main()` supplies it during construction.
