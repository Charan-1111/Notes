# Command Design Pattern

The **Command pattern** is a behavioral design pattern that represents a request as a standalone object.

The command stores the information needed to perform an action, such as:

- The object that performs the work.
- The operation to invoke.
- Any required arguments.

This separates the object that **triggers an action** from the object that **performs it**.

**Simple example:**

- A remote triggers a command.
- A `LightOnCommand` calls the light’s `On()` method.
- A `LightOffCommand` calls the light’s `Off()` method.

The remote only knows how to execute a command. It does not need to understand how a light works.

## When to Use

- When an invoker should support different actions through the same interface.
- When requests need to be queued, delayed, or scheduled.
- When you need execution history or optional undo/redo support.
- When several actions should be grouped into one command.

**Backend example:**

A `GenerateReportCommand` contains a report ID and calls a report service.

A worker can execute that command without knowing the details of report generation.

Command objects help organize execution, but queueing, persistence, retries, and scheduling require additional implementation.

## Components

| Component | Responsibility | Light example |
|---|---|---|
| **Command Interface** | Defines the operation used to execute commands. | `Command`, with `Execute()`. |
| **Concrete Command** | Connects a particular action to its receiver and arguments. | `LightOnCommand` and `LightOffCommand`. |
| **Receiver** | Performs the actual work. | `Light`. |
| **Invoker** | Triggers a command through the shared interface. | `RemoteControl`. |
| **Client** | Creates the objects and connects them. | `main()`. |

The command knows the receiver. The invoker only depends on the command interface.

## Working

1. The client creates the receiver.
2. The client creates commands that reference the receiver.
3. The client assigns a command to the invoker.
4. The invoker calls the command’s `Execute()` method.
5. The command calls the appropriate receiver method.

For turning on the light:

1. The client presses the remote’s button.
2. The remote executes `LightOnCommand`.
3. The command calls `Light.On()`.
4. The light performs the operation.

The client can replace the assigned command without changing the remote’s button logic.

## Pros

- **Decouples invoker and receiver:** The invoker does not depend on device-specific operations.
- **Supports extension:** New commands can implement the same interface.
- **Enables deferred execution:** Commands can be stored and executed later.
- **Supports history:** An application can record commands and their outcomes.
- **Enables optional undo/redo:** Commands can capture the information needed to reverse or repeat operations.
- **Supports macro commands:** A command can execute a group of other commands.

These are capabilities the design supports, not features provided automatically.

## Cons

- **Adds types and objects:** A large number of actions may introduce many commands.
- **Can overcomplicate simple code:** A direct function call may be sufficient.
- **Makes undo difficult for some operations:** Reversing an action may require previous state or compensation.
- **Adds queue and history costs:** Retaining commands consumes memory or storage.
- **Makes replay sensitive:** Repeating a command may repeat external side effects.

For example, replaying a payment command could charge a customer twice unless the operation has suitable idempotency protection.

## Sample Implementation in Golang

This example demonstrates command execution and runtime replacement. It does not implement undo/redo.

```go
package main

import (
	"errors"
	"fmt"
)

// Command interface.
type Command interface {
	Execute()
}

// Receiver: performs the actual operations.
type Light struct{}

func (l *Light) On() {
	fmt.Println("Light is ON")
}

func (l *Light) Off() {
	fmt.Println("Light is OFF")
}

// Concrete command: turns the light on.
type LightOnCommand struct {
	light *Light
}

func (c *LightOnCommand) Execute() {
	c.light.On()
}

// Concrete command: turns the light off.
type LightOffCommand struct {
	light *Light
}

func (c *LightOffCommand) Execute() {
	c.light.Off()
}

// Invoker: triggers whichever command is assigned.
type RemoteControl struct {
	command Command
}

func (r *RemoteControl) SetCommand(command Command) {
	r.command = command
}

func (r *RemoteControl) PressButton() error {
	if r.command == nil {
		return errors.New("no command assigned")
	}

	r.command.Execute()
	return nil
}

// Client.
func main() {
	light := &Light{}

	lightOn := &LightOnCommand{light: light}
	lightOff := &LightOffCommand{light: light}

	remote := &RemoteControl{}

	// No command has been assigned yet.
	if err := remote.PressButton(); err != nil {
		fmt.Println("Remote:", err)
	}

	// Assign and execute the ON command.
	remote.SetCommand(lightOn)
	if err := remote.PressButton(); err != nil {
		fmt.Println("Remote:", err)
	}

	// Replace it with the OFF command.
	remote.SetCommand(lightOff)
	if err := remote.PressButton(); err != nil {
		fmt.Println("Remote:", err)
	}
}
```

**Output:**

```text
Remote: no command assigned
Light is ON
Light is OFF
```

### Step 1: Define the Command Interface

```go
type Command interface {
	Execute()
}
```

The invoker uses this common method to trigger any command.

Go satisfies interfaces implicitly. Both concrete command pointer types implement `Command` because they provide `Execute()`.

### Step 2: Create the Receiver

```go
type Light struct{}
```

The light provides `On()` and `Off()`.

It performs the actual operations and does not need to know about commands or the remote.

### Step 3: Bind an Action to the Receiver

```go
type LightOnCommand struct {
	light *Light
}

func (c *LightOnCommand) Execute() {
	c.light.On()
}
```

The command stores the receiver and selects the operation to invoke.

`LightOffCommand` uses the same structure but calls `Off()`.

Commands can also store arguments—for example, a brightness command could hold a light reference and a brightness level.

### Step 4: Make the Invoker Depend on the Interface

```go
type RemoteControl struct {
	command Command
}
```

The remote stores a `Command`, rather than a particular light command.

Its execution logic stays the same when a different command is assigned.

### Step 5: Handle an Unset Command

```go
func (r *RemoteControl) PressButton() error {
	if r.command == nil {
		return errors.New("no command assigned")
	}

	r.command.Execute()
	return nil
}
```

The check handles a remote whose command interface is unset.

The example assumes valid command objects with valid receivers. A typed nil pointer stored inside an interface is a separate case and is not caught by this check.

The returned error only reports an unset command. For operations that can fail, the command interface could instead define `Execute() error`, with the invoker forwarding that error.

### Step 6: Replace Commands at Runtime

```go
remote.SetCommand(lightOn)
remote.SetCommand(lightOff)
```

Each call changes what the next button press will do.

Replacing the command does not execute it; execution happens when `PressButton()` is called.

### Step 7: Understand Undo and Redo

Undo requires additional behavior and often previous state.

Suppose the light was already on before an ON command ran. Undoing that command by simply turning the light off would restore the wrong state.

A reversible command may need to:

- Capture the previous state.
- Perform the action.
- Restore that state when undone.

An application also needs history management to decide which command to undo or redo.

Some operations, such as sending an email, cannot be directly undone.

### Step 8: Understand Queueing and Persistence

These command objects contain Go pointers, so they are suitable for in-memory execution.

A persistent queue normally stores serializable data such as:

- Command type.
- Receiver or resource ID.
- Arguments.
- An idempotency key when needed.

A worker reads that data, obtains the required dependencies, and executes the operation.

### Step 9: Keep Go Implementations Simple

A separate struct for every action is optional.

For small commands, a function type with an `Execute()` method can adapt functions to the command interface.

Use dedicated structs when commands need explicit arguments, state, metadata, or undo information.