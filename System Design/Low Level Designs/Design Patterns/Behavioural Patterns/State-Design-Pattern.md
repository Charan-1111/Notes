# State Design Pattern

The **State pattern** is a behavioral design pattern that lets an object change its behavior when its internal state changes.

The object delegates state-specific operations to its current **state object**.

Its Go type stays the same. Its behavior changes because a different state implementation receives the calls.

**Simple example:**

A vending machine handles `SelectItem()` differently depending on its state:

- **No coin:** Ask the user to insert a coin.
- **Has coin:** Accept the selection.
- **Ready to dispense:** Reject another selection until dispensing finishes.

## When to Use

- When an object has distinct states with different behavior.
- When valid actions and transitions depend on the current state.
- When repeated state checks make methods difficult to maintain.
- When state-specific logic is substantial enough to benefit from separate types.

**Backend example:**

An order may be pending, paid, shipped, or cancelled.

A cancellation request could be accepted while pending but rejected after shipment.

For a small workflow with straightforward rules, a switch statement may be simpler. The number of states alone does not determine whether this pattern is useful.

## Components

| Component | Responsibility | Vending machine example |
|---|---|---|
| **Context** | Holds the current state and delegates actions to it. | `VendingMachine`. |
| **State Interface** | Defines the actions supported by state implementations. | `State`. |
| **Concrete States** | Implement behavior for each state. | `NoCoinState`, `HasCoinState`, and `ReadyToDispenseState`. |
| **State Transitions** | Replace the current state when an action permits it. | Calling `setState()`. |

A transition is part of the workflow, not necessarily a separate type.

## Working

1. The machine starts in `NoCoinState`.
2. The client calls an action on the machine.
3. The machine delegates to its current state.
4. That state accepts or rejects the action.
5. An accepted action may transition the machine to another state.

| Current state | Action | Result | Next state |
|---|---|---|---|
| No coin | Insert coin | Accept coin | Has coin |
| No coin | Select item or dispense | Ask for a coin | No coin |
| Has coin | Insert coin | Reject extra coin | Has coin |
| Has coin | Select item | Accept selection | Ready to dispense |
| Has coin | Dispense | Ask for a selection | Has coin |
| Ready to dispense | Insert coin or select item | Reject action | Ready to dispense |
| Ready to dispense | Dispense | Dispense item | No coin |

The same method call can therefore produce different results depending on the current state.

## Pros

- **Encapsulates state-specific behavior:** Each state groups its own rules.
- **Reduces repeated conditions:** The context delegates instead of checking every possible state.
- **Makes transitions explicit:** State changes occur where the relevant action is handled.
- **Improves maintainability:** Complex state behavior can be changed in focused types.
- **Supports extension:** New states can implement the existing interface.

Adding a state may still require updating existing transitions. Adding a new method to the state interface requires updating every implementation.

## Cons

- **Introduces additional types:** Many states can make the design larger.
- **Adds overhead for simple workflows:** A small switch may be easier to understand.
- **Distributes workflow logic:** Understanding the full lifecycle may require reading several state types.
- **Creates dependencies:** States may reference the context and other concrete states.
- **Requires transition discipline:** Allowing arbitrary state changes can bypass workflow rules.

## Sample Implementation in Golang

This example initializes the machine through a constructor and lets states control transitions.

It assumes stock is available and dispensing succeeds.

```go
package main

import "fmt"

// State interface.
type State interface {
	InsertCoin()
	SelectItem()
	Dispense()
}

// Context.
type VendingMachine struct {
	state State
}

func NewVendingMachine() *VendingMachine {
	vm := &VendingMachine{}
	vm.state = &NoCoinState{vm: vm}
	return vm
}

// Internal transition method.
func (vm *VendingMachine) setState(state State) {
	vm.state = state
}

func (vm *VendingMachine) InsertCoin() {
	vm.state.InsertCoin()
}

func (vm *VendingMachine) SelectItem() {
	vm.state.SelectItem()
}

func (vm *VendingMachine) Dispense() {
	vm.state.Dispense()
}

// Concrete state: waiting for a coin.
type NoCoinState struct {
	vm *VendingMachine
}

func (s *NoCoinState) InsertCoin() {
	fmt.Println("Coin inserted.")
	s.vm.setState(&HasCoinState{vm: s.vm})
}

func (s *NoCoinState) SelectItem() {
	fmt.Println("Insert a coin first.")
}

func (s *NoCoinState) Dispense() {
	fmt.Println("Insert a coin first.")
}

// Concrete state: coin received.
type HasCoinState struct {
	vm *VendingMachine
}

func (s *HasCoinState) InsertCoin() {
	fmt.Println("Coin already inserted.")
}

func (s *HasCoinState) SelectItem() {
	fmt.Println("Item selected.")
	s.vm.setState(&ReadyToDispenseState{vm: s.vm})
}

func (s *HasCoinState) Dispense() {
	fmt.Println("Select an item first.")
}

// Concrete state: selection accepted, ready to dispense.
type ReadyToDispenseState struct {
	vm *VendingMachine
}

func (s *ReadyToDispenseState) InsertCoin() {
	fmt.Println("Complete the current purchase first.")
}

func (s *ReadyToDispenseState) SelectItem() {
	fmt.Println("An item is already selected.")
}

func (s *ReadyToDispenseState) Dispense() {
	fmt.Println("Item dispensed.")
	s.vm.setState(&NoCoinState{vm: s.vm})
}

// Client.
func main() {
	vm := NewVendingMachine()

	vm.SelectItem() // Rejected: no coin.

	vm.InsertCoin()
	vm.InsertCoin() // Rejected: coin already received.

	vm.Dispense() // Rejected: no selection.

	vm.SelectItem()
	vm.SelectItem() // Rejected: selection already made.

	vm.Dispense()

	vm.SelectItem() // Rejected: machine is waiting for a coin again.
}
```

**Output:**

```text
Insert a coin first.
Coin inserted.
Coin already inserted.
Select an item first.
Item selected.
An item is already selected.
Item dispensed.
Insert a coin first.
```

### Step 1: Define the State Interface

```go
type State interface {
	InsertCoin()
	SelectItem()
	Dispense()
}
```

Each state implements the same actions.

An action does not have to succeed in every state. A state can reject it according to the workflow rules.

### Step 2: Store the Current State

```go
type VendingMachine struct {
	state State
}
```

The machine holds one active state.

Its methods delegate to that state without checking which concrete type it contains.

### Step 3: Initialize a Valid Starting State

```go
func NewVendingMachine() *VendingMachine {
	vm := &VendingMachine{}
	vm.state = &NoCoinState{vm: vm}
	return vm
}
```

The constructor creates the machine and connects its initial state back to it.

The state needs that reference so it can request transitions.

Use this constructor: a directly created zero-value `VendingMachine` has a nil state and cannot process actions.

### Step 4: Delegate Client Requests

```go
func (vm *VendingMachine) SelectItem() {
	vm.state.SelectItem()
}
```

The client always calls the machine.

The active state determines whether selection is accepted, rejected, or followed by a transition.

### Step 5: Transition After Receiving a Coin

```go
func (s *NoCoinState) InsertCoin() {
	fmt.Println("Coin inserted.")
	s.vm.setState(&HasCoinState{vm: s.vm})
}
```

The state accepts the action and replaces the machine’s current state.

Future calls are handled by `HasCoinState`.

### Step 6: Reject Invalid Actions Without Transitioning

```go
func (s *HasCoinState) Dispense() {
	fmt.Println("Select an item first.")
}
```

The machine already has a coin, but no item has been selected.

The action is rejected, and the machine remains in `HasCoinState`.

### Step 7: Complete the Purchase

After selection, the machine enters `ReadyToDispenseState`.

Calling `Dispense()` prints the result and returns the machine to `NoCoinState`.

The name describes this example accurately: the machine is ready to dispense until the method is called. It does not model dispensing as a background operation.

### Step 8: Keep Transitions Controlled

```go
func (vm *VendingMachine) setState(state State) {
	vm.state = state
}
```

The lowercase method is unexported, so clients in other packages cannot call it directly.

Go privacy is package-level. Code in the same package can still access it.

Keeping transition methods internal helps clients follow the intended sequence instead of assigning arbitrary states.

### Step 9: Extend for Real Operations

A production vending machine would also need rules for:

- Inventory and item selection.
- Payment amounts and refunds.
- Dispensing failures.
- Concurrent requests.

Methods can return errors when callers need to respond programmatically to rejected or failed actions.

If dispensing fails, the correct next state depends on the business rules. It should not automatically reset as though the purchase succeeded.

The State pattern organizes these rules; it does not automatically provide synchronization, persistence, or transactional behavior.
