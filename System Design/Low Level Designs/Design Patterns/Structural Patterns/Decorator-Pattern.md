# Decorator Design Pattern

The **Decorator pattern** is a structural design pattern that adds behavior to an object by wrapping it in another object.

The wrapper implements the same interface as the original object, so the client can use either one in the same way.

**Simple example:**

- Plain coffee costs ₹50.
- A milk decorator adds ₹10.
- A sugar decorator adds ₹5.
- Wrapping plain coffee with both produces a total cost of ₹65.

Each decorator uses the wrapped object's behavior and adds its own contribution.

**Adapter vs Decorator:**

- **Adapter:** Converts an implementation to the interface the client expects.
- **Decorator:** Preserves the interface while adding behavior.

In Go, decorators use **interfaces and composition**, rather than class inheritance.

## When to Use

- When optional features should be added without modifying the original implementation.
- When behaviors need to be combined in different ways at runtime.
- When creating a separate type for every feature combination would duplicate code.
- When optional functionality would otherwise make one type difficult to maintain.

**Backend example:**

An HTTP handler processes a request. Wrappers can add:

- Authentication.
- Logging.
- Metrics.

Each wrapper implements `http.Handler` and delegates to another handler. This is a common decorator-style implementation of HTTP middleware.

## Components

| Component | Responsibility | Coffee example |
|---|---|---|
| **Component Interface** | Defines operations shared by original objects and decorators. | `Coffee`. |
| **Concrete Component** | Provides the original behavior. | `PlainCoffee`. |
| **Decorator** | Holds a component and delegates operations to it. | `CoffeeDecorator`. |
| **Concrete Decorator** | Adds a specific behavior while preserving the interface. | `MilkDecorator` and `SugarDecorator`. |

A separate base decorator is optional in Go. Each concrete decorator can directly hold a `Coffee` field.

Here, we keep the base struct to match your example.

## Working

1. The client creates the original component.
2. A decorator wraps that component.
3. Additional decorators can wrap the existing decorated object.
4. The client calls methods through the shared interface.
5. Each decorator delegates to its wrapped component and adds behavior.

For coffee wrapped first with milk and then sugar:

- Sugar asks the milk decorator for its cost.
- Milk asks plain coffee for its cost.
- Plain coffee returns ₹50.
- Milk adds ₹10.
- Sugar adds ₹5.
- The client receives ₹65.

Wrapping does not inherently modify the original object. Other clients can still use plain coffee directly.

However, decorators that mutate shared state or introduce side effects can affect other users of that state.

## Pros

- **Adds behavior dynamically:** Chooses wrappers when assembling objects at runtime.
- **Supports the Open–Closed Principle:** Adds decorators without changing the original component.
- **Avoids combination-specific types:** Combines features through wrappers.
- **Encourages focused responsibilities:** Each decorator can handle one concern.
- **Preserves client usage:** Wrapped and unwrapped objects satisfy the same interface.

## Cons

- **Adds layers:** Many wrappers can make the call flow harder to follow.
- **Order can matter:** Different wrapping orders may produce different results.
- **Makes debugging harder:** Behavior is distributed across several objects.
- **Allows invalid combinations:** Some decorators may depend on a particular order or another feature.

For example, a logging wrapper outside authentication can observe rejected requests. A logging wrapper inside authentication may only run for requests that pass authentication.

In the coffee example, reversing milk and sugar changes the description order, but the total cost remains ₹65.

## Sample Implementation in Golang

This example wraps plain coffee with milk and sugar.

```go
package main

import "fmt"

// Component interface.
type Coffee interface {
	Cost() int
	Description() string
}

// Concrete component.
type PlainCoffee struct{}

func (p *PlainCoffee) Cost() int {
	return 50
}

func (p *PlainCoffee) Description() string {
	return "Plain Coffee"
}

// Base decorator: delegates to a wrapped component.
type CoffeeDecorator struct {
	coffee Coffee
}

func (c *CoffeeDecorator) Cost() int {
	return c.coffee.Cost()
}

func (c *CoffeeDecorator) Description() string {
	return c.coffee.Description()
}

// Concrete decorator: adds milk.
type MilkDecorator struct {
	CoffeeDecorator
}

func (m *MilkDecorator) Cost() int {
	return m.CoffeeDecorator.Cost() + 10
}

func (m *MilkDecorator) Description() string {
	return m.CoffeeDecorator.Description() + ", Milk"
}

// Concrete decorator: adds sugar.
type SugarDecorator struct {
	CoffeeDecorator
}

func (s *SugarDecorator) Cost() int {
	return s.CoffeeDecorator.Cost() + 5
}

func (s *SugarDecorator) Description() string {
	return s.CoffeeDecorator.Description() + ", Sugar"
}

// Client.
func main() {
	plain := &PlainCoffee{}
	var coffee Coffee = plain

	coffee = &MilkDecorator{
		CoffeeDecorator: CoffeeDecorator{
			coffee: coffee,
		},
	}

	coffee = &SugarDecorator{
		CoffeeDecorator: CoffeeDecorator{
			coffee: coffee,
		},
	}

	fmt.Printf("Order: %s\n", coffee.Description())
	fmt.Printf("Total cost: ₹%d\n", coffee.Cost())
	fmt.Printf("Plain coffee still costs: ₹%d\n", plain.Cost())
}
```

**Output:**

```text
Order: Plain Coffee, Milk, Sugar
Total cost: ₹65
Plain coffee still costs: ₹50
```

### Step 1: Define the Shared Interface

```go
type Coffee interface {
	Cost() int
	Description() string
}
```

Both plain coffee and decorators provide these methods.

The client can therefore use any of them through a `Coffee` variable.

### Step 2: Implement the Original Behavior

`PlainCoffee` returns:

- A cost of ₹50.
- A description of `"Plain Coffee"`.

It does not know about milk, sugar, or decorators.

### Step 3: Hold the Wrapped Component

```go
type CoffeeDecorator struct {
	coffee Coffee
}
```

The field accepts any `Coffee` implementation, including another decorator.

The base decorator's methods simply forward calls to that wrapped object.

### Step 4: Add Specific Behavior

```go
func (m *MilkDecorator) Cost() int {
	return m.CoffeeDecorator.Cost() + 10
}
```

The milk decorator gets the wrapped component's cost and adds ₹10.

Its `Description()` method similarly appends `", Milk"`.

The sugar decorator follows the same approach, adding ₹5 and `", Sugar"`.

Each concrete decorator preserves the `Coffee` interface.

### Step 5: Understand Embedding

```go
type MilkDecorator struct {
	CoffeeDecorator
}
```

This embeds the base decorator for reuse.

Go embedding is **composition, not class inheritance**. The milk decorator defines its own methods and explicitly calls the embedded decorator's methods.

The base decorator is a convenience; it is not required by the pattern.

### Step 6: Stack the Wrappers

The client begins with plain coffee:

```go
plain := &PlainCoffee{}
var coffee Coffee = plain
```

It then wraps the current object with milk, followed by sugar.

During each assignment, the new wrapper stores the previous value of `coffee`. Reassigning the variable does not replace the component inside earlier wrappers or create a cycle.

The final object is a sugar decorator wrapping a milk decorator wrapping plain coffee.

### Step 7: Trace the Result

For `Cost()`:

1. Plain coffee returns `50`.
2. Milk adds `10`, producing `60`.
3. Sugar adds `5`, producing `65`.

For `Description()`:

1. Plain coffee returns `"Plain Coffee"`.
2. Milk appends `", Milk"`.
3. Sugar appends `", Sugar"`.

The original `plain` object still returns ₹50 because these decorators do not mutate it.

### Step 8: Keep Wrappers Valid

Every decorator in this example needs a non-nil wrapped component.

Calling methods on an uninitialized decorator would fail when it tries to delegate. Constructors can help ensure that wrappers receive valid components.

For backend decorators, also preserve relevant return values, errors, and cancellation behavior when forwarding calls.
