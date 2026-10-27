# State Design Pattern
- Behavioural design pattern that lets an object change its behaviour when its internal state changes, making it appear as if it changed it class.
- This pattern is especially helpful for managing objects with complex, state-dependent logic, such as vending machines, document lifecycles, or order-processing systems.

- ## When to Use
	- When an object can be in one of many distinct states, each with different behaviour.
	- When the objects behaviour depends on current context, and that context changes overtime.
	- When we want to avoid large monolithic if-else or switch statements to check for every possible state.

- ## Components
	- **Context** - The main object whose behaviour changes with state. It holds a reference to a current state object and delegates state-specific requests to it.
	- **State Interface** - Declares the behaviours or actions that each concrete state must implement.
	- **Concrete State Classes** - Each represents a specific state and implements state-specific behaviours declared in the interface.
	- **State Transition** - States can trigger transitions by assigning a different concrete state objects to the context.

- ## Pros
	- Removes complex conditions - Avoids huge if-else or switch blocks to handle state-specific logic.
	- Encapsulation of state-specific behaviour - Each state has its own class -> Cleaner, more maintainable code.
	- Open/Closed Principle - Adding a new state or behaviour doesn't require changing the context or existing states.
	- Improved readability and maintainability - State transitions and behaviours are explicit and easier to reason about.
	- Flexible State transitions - State objects can decide the next state dynamically, giving more control over workflows.

- ## Cons
	- Class Explosion - Each state requires a new class -> too many classes for systems with lot of states.
	- Complexity Overhead - For simple state management ( like 2-3 states ), using this pattern may be over engineering.
	- Harder debugging - Since behaviour is spread across multiple state classes, it can be harder to trace logic flow.
	- Tight coupling between Context and States - The context often needs to reference concrete states, which can reduce flexibility of not managed carefully.

- ## Sample implementation in Golang

```go
// Vending Machine

package main

import "fmt"

// State interface
type State interface {
	InsertCoin()
	SelectItem()
	Dispense()
}

// Context
type VendingMachine struct {
	state State
}

func (vm *VendingMachine) SetState(s State) {
	vm.state = s
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

// Concrete States
type NoCoinState struct {
	vm *VendingMachine
}

func (s *NoCoinState) InsertCoin() {
	fmt.Println("Coin Inserted.")
	s.vm.SetState(&HasCoinState{vm: s.vm})
}

func (s *NoCoinState) SelectItem() {
	fmt.Println("Insert Coin First")
}

func (s *NoCoinState) Dispense() {
	fmt.Println("Insert Coin First")
}

type HasCoinState struct {
	vm *VendingMachine
}

func (s *HasCoinState) InsertCoin() {
	fmt.Println("Coin already inserted")
}

func (s *HasCoinState) SelectItem() {
	fmt.Println("Item Selected")
	s.vm.SetState(&DispenseState{vm: s.vm})
}

func (s *HasCoinState) Dispense() {
	fmt.Println("Select item first!")
}

type DispenseState struct {
	vm *VendingMachine
}

func (s *DispenseState) InsertCoin() {
	fmt.Println("Please wait, dispensing...")
}

func (s *DispenseState) SelectItem() {
	fmt.Println("Already dispensing")
}

func (s *DispenseState) Dispense() {
	fmt.Println("Item Dispensed")
	s.vm.SetState(&NoCoinState{vm: s.vm})
}

// Client
func main() {
	vm := &VendingMachine{}
	nocoin := &NoCoinState{vm: vm}
	
	vm.SetState(nocoin)

	vm.InsertCoin()
	vm.SelectItem()
	vm.Dispense()
}
```
