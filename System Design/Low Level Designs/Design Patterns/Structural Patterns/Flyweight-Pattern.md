# Flyweight Design Pattern

The **Flyweight pattern** is a structural design pattern that reduces memory usage by sharing common data across many objects.

It separates an object's state into:

- **Intrinsic state:** Shared data that is independent of where the object is used.
- **Extrinsic state:** Data specific to an occurrence, maintained by the client and supplied when needed.

**Simple example:**

The text `"ABABA"` contains five character occurrences but only two unique symbols.

We can reuse:

- One character object for `A`.
- One character object for `B`.

Each occurrence supplies its own position.

| State | Example | Where it belongs |
|---|---|---|
| **Intrinsic** | Character symbol: `A` or `B`. | Shared character object. |
| **Extrinsic** | Position: `0`, `1`, `2`, etc. | Client. |

In a real text renderer, shared data might include expensive font outlines or glyph images, rather than just a symbol.

## When to Use

- When there are many objects containing repeated data.
- When the repeated data consumes significant memory.
- When shared data can be separated from occurrence-specific state.
- When shared objects can generally remain immutable.

**Example:**

A forest simulation contains thousands of trees.

- **Intrinsic state:** Tree species, texture, and model.
- **Extrinsic state:** Position, rotation, and size.

Trees of the same species share the model and texture while keeping their individual placement information.

Use Flyweight when the savings justify the extra structure. A few small objects usually do not need it.

## Components

| Component | Responsibility | Character example |
|---|---|---|
| **Flyweight Interface** | Defines operations that accept extrinsic state. | `Character`, which declares `Display(position int)`. |
| **Concrete Flyweight** | Stores intrinsic state and uses supplied extrinsic state. | `ConcreteCharacter`. |
| **Flyweight Factory** | Creates and reuses flyweights by their intrinsic state. | `CharacterFactory`. |
| **Client** | Requests flyweights and supplies occurrence-specific state. | `main()`. |

The factory's key must identify all relevant intrinsic state.

For example, if characters differ by symbol, font, and font size, the key must include all three—not just the symbol.

## Working

1. The client requests a flyweight using its intrinsic state.
2. The factory checks its pool for a matching object.
3. If one exists, the factory returns it.
4. Otherwise, the factory creates, stores, and returns a new object.
5. The client supplies extrinsic state when calling the object's methods.

For `"ABABA"`:

- The first `A` creates an object.
- The first `B` creates another object.
- Later occurrences reuse those two objects.

Each call supplies a different position without storing that position inside the shared object.

## Pros

- **Reduces duplication:** Stores common data once per unique combination.
- **Can save memory:** Particularly useful when shared data is large.
- **Can reduce allocations:** Reuses existing objects instead of repeatedly allocating them.
- **Can reduce GC pressure:** Fewer allocations may mean less garbage-collection work.

These benefits depend on the workload. Factory lookups and bookkeeping also have a cost.

## Cons

- **Adds complexity:** Requires careful separation of intrinsic and extrinsic state.
- **Moves responsibility to clients:** Clients must maintain occurrence-specific data.
- **Makes mutation risky:** Changing a shared object affects all clients using it.
- **Retains pooled objects:** Objects remain reachable while the factory keeps references to them.
- **Requires synchronization for concurrent access:** Multiple goroutines cannot safely read and modify an ordinary map without coordination.

Shared intrinsic state should generally remain immutable. For a factory with many unique keys, consider its lifetime and whether entries need eviction.

## Sample Implementation in Golang

This example shares character objects while passing each occurrence's position separately.

```go
package main

import "fmt"

// Flyweight interface.
type Character interface {
	Display(position int)
}

// Concrete flyweight.
type ConcreteCharacter struct {
	symbol rune // Intrinsic state: shared symbol.
}

func (c *ConcreteCharacter) Display(position int) {
	// Extrinsic state: supplied for this occurrence.
	fmt.Printf("Character: %c at position %d\n", c.symbol, position)
}

// Flyweight factory.
// This example is intended for use by one goroutine.
type CharacterFactory struct {
	pool map[rune]*ConcreteCharacter
}

func NewCharacterFactory() *CharacterFactory {
	return &CharacterFactory{
		pool: make(map[rune]*ConcreteCharacter),
	}
}

func (f *CharacterFactory) GetCharacter(symbol rune) Character {
	if character, exists := f.pool[symbol]; exists {
		return character
	}

	character := &ConcreteCharacter{
		symbol: symbol,
	}

	f.pool[symbol] = character
	return character
}

func (f *CharacterFactory) Count() int {
	return len(f.pool)
}

// Client.
func main() {
	factory := NewCharacterFactory()

	text := "ABABA"
	position := 0

	for _, symbol := range text {
		character := factory.GetCharacter(symbol)
		character.Display(position)
		position++
	}

	fmt.Printf("Shared objects created: %d\n", factory.Count())

	firstA := factory.GetCharacter('A')
	secondA := factory.GetCharacter('A')

	fmt.Printf("Same A object reused: %t\n", firstA == secondA)
}
```

**Output:**

```text
Character: A at position 0
Character: B at position 1
Character: A at position 2
Character: B at position 3
Character: A at position 4
Shared objects created: 2
Same A object reused: true
```

### Step 1: Define the Flyweight Interface

```go
type Character interface {
	Display(position int)
}
```

The method accepts `position` as extrinsic state.

This lets the same shared character appear at different positions.

### Step 2: Store Only Intrinsic State

```go
type ConcreteCharacter struct {
	symbol rune
}
```

The shared object stores its symbol but does not store a position.

A `rune` represents a Unicode code point.

The field is unexported, preventing other packages from changing it directly. It is not automatically immutable; this implementation preserves immutability by never changing it after creation.

### Step 3: Combine Shared and Supplied State

```go
func (c *ConcreteCharacter) Display(position int) {
	fmt.Printf("Character: %c at position %d\n", c.symbol, position)
}
```

The method uses:

- `c.symbol`: shared intrinsic state.
- `position`: occurrence-specific extrinsic state.

It does not modify the shared object.

### Step 4: Initialize the Factory's Pool

```go
func NewCharacterFactory() *CharacterFactory {
	return &CharacterFactory{
		pool: make(map[rune]*ConcreteCharacter),
	}
}
```

The map associates each symbol with its shared object.

For example, the key `'A'` points to the character object used for every `A`.

### Step 5: Reuse or Create an Object

```go
func (f *CharacterFactory) GetCharacter(symbol rune) Character {
	if character, exists := f.pool[symbol]; exists {
		return character
	}

	character := &ConcreteCharacter{
		symbol: symbol,
	}

	f.pool[symbol] = character
	return character
}
```

The factory first checks for an existing object.

A new object is allocated only when the symbol is absent. The method returns the `Character` interface, keeping client usage focused on `Display()`.

### Step 6: Supply the Position from the Client

```go
position := 0

for _, symbol := range text {
	character := factory.GetCharacter(symbol)
	character.Display(position)
	position++
}
```

The client tracks each occurrence's position.

Go's `range` over a string normally provides a byte offset as its index. Here, we ignore that index and use a separate counter to count Unicode code points.

A code point is not always a complete visible character; some visible characters contain multiple code points.

### Step 7: Verify Object Reuse

```go
firstA := factory.GetCharacter('A')
secondA := factory.GetCharacter('A')

fmt.Printf("Same A object reused: %t\n", firstA == secondA)
```

Both interface values contain the same `*ConcreteCharacter` pointer, so the comparison returns `true`.

The factory contains only two objects even though five occurrences were displayed.

This demonstrates sharing. It does not prove a memory improvement for such a small example—the map and interface overhead may outweigh the savings.

### Step 8: Consider Concurrent Usage

The factory performs map reads and writes, so this version should not be shared across goroutines without synchronization.

For concurrent usage, protect the lookup-and-create operation with a mutex. Otherwise, concurrent requests can cause data races or unsafe map access.

The character objects themselves only read immutable state, which makes them suitable for sharing once safely initialized and published.
