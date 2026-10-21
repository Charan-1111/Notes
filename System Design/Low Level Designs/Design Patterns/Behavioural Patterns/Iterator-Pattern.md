# Iterator Design Pattern

The **Iterator pattern** is a behavioral design pattern that provides a way to access collection elements sequentially without exposing how the collection stores them.

It separates **traversal logic** from the code that uses the elements.

**Simple example:**

A collection contains `"A"`, `"B"`, and `"C"`.

The client asks an iterator for the next element. It does not need to manage indexes or know whether the collection uses a slice, linked list, or another structure.

The iterator tracks the traversal position.

## When to Use

- When clients need to traverse a custom collection without knowing its internal structure.
- When a collection supports different traversal strategies, such as forward, reverse, or filtered traversal.
- When separate clients need independent traversal positions.
- When traversal requires more logic than a simple loop.

**Backend example:**

An iterator can expose records from a paginated API. The client requests the next record while the iterator manages page boundaries.

Real API or database iterators also need a way to report errors and may require cleanup.

For a straightforward traversal over an available slice, a regular `for range` loop is usually sufficient.

## Components

| Component | Responsibility | String collection example |
|---|---|---|
| **Iterator Interface** | Defines how clients request the next element. | `Iterator`. |
| **Concrete Iterator** | Implements traversal and tracks its current position. | `ListIterator`. |
| **Aggregate Interface** | Defines how to create an iterator. | `Aggregate`. |
| **Concrete Aggregate** | Holds the collection and creates concrete iterators. | `StringCollection`. |

The concrete aggregate returns an iterator through the iterator interface.

A separate aggregate interface is optional in Go when clients do not need that abstraction.

## Working

1. The client requests an iterator from the collection.
2. The collection creates an iterator with its own traversal position.
3. The client repeatedly calls `Next()`.
4. The iterator returns an element and advances its position.
5. When no elements remain, it reports exhaustion.

Creating another iterator starts a separate traversal.

Independent positions do not imply independent copies of the underlying data.

## Pros

- **Encapsulates traversal:** Clients do not manage collection-specific navigation.
- **Provides consistent access:** Different collections can expose the same iterator interface for the same element type.
- **Supports independent traversals:** Each iterator maintains its own position.
- **Supports different strategies:** Forward, reverse, and filtered iterators can provide different views.
- **Keeps client logic focused:** Clients process elements while iterators handle navigation.

Adding a traversal strategy may still require a new factory method or configuration. The collection is not guaranteed to remain unchanged.

## Cons

- **Adds abstraction:** Extra types may be unnecessary for simple collections.
- **Can limit available operations:** Sequential traversal does not replace efficient indexed access.
- **Requires mutation rules:** Changes during iteration can affect results.
- **Does not guarantee concurrency safety:** Shared data or iterator state may require synchronization.
- **May expose mutable elements:** Hiding collection structure does not automatically protect the objects returned.

## Sample Implementation in Golang

This example uses `Next() (string, bool)`:

- The string is the next value.
- The boolean indicates whether a value exists.

A separate `HasNext()` method is unnecessary because `Next()` reports exhaustion directly.

```go
package main

import "fmt"

// Iterator interface.
type Iterator interface {
	Next() (string, bool)
}

// Concrete iterator.
type ListIterator struct {
	items []string
	index int
}

func (it *ListIterator) Next() (string, bool) {
	if it.index >= len(it.items) {
		return "", false
	}

	value := it.items[it.index]
	it.index++

	return value, true
}

// Aggregate interface.
type Aggregate interface {
	CreateIterator() Iterator
}

// Concrete aggregate.
type StringCollection struct {
	items []string
}

func (c *StringCollection) CreateIterator() Iterator {
	return &ListIterator{
		items: c.items,
	}
}

// Client.
func main() {
	var collection Aggregate = &StringCollection{
		items: []string{"A", "B", "C"},
	}

	iterator := collection.CreateIterator()

	for {
		value, ok := iterator.Next()
		if !ok {
			break
		}

		fmt.Println(value)
	}

	// A new iterator starts from the beginning.
	secondIterator := collection.CreateIterator()
	value, ok := secondIterator.Next()

	fmt.Printf("Second iterator: %s, available: %t\n", value, ok)

	// The original iterator remains exhausted.
	_, ok = iterator.Next()
	fmt.Printf("Original iterator has another value: %t\n", ok)
}
```

**Output:**

```text
A
B
C
Second iterator: A, available: true
Original iterator has another value: false
```

### Step 1: Define the Iterator Interface

```go
type Iterator interface {
	Next() (string, bool)
}
```

The interface provides type-safe access to strings.

Unlike `interface{}`, callers do not need a type assertion to use the result as a string.

The exported name `Next()` allows clients in other packages to call it.

### Step 2: Store Traversal State

```go
type ListIterator struct {
	items []string
	index int
}
```

- `items` references the elements being traversed.
- `index` identifies the next element to return.

Each iterator has its own `index`.

### Step 3: Return the Next Element

```go
func (it *ListIterator) Next() (string, bool) {
	if it.index >= len(it.items) {
		return "", false
	}

	value := it.items[it.index]
	it.index++

	return value, true
}
```

The method first checks whether the traversal has ended.

If an element exists, it returns that element and advances the index.

The boolean distinguishes these cases:

| Result | Meaning |
|---|---|
| `"A", true` | An element containing `"A"`. |
| `"", true` | A valid element containing an empty string. |
| `"", false` | No element remains. |

An empty collection immediately returns `false`. Repeated calls after exhaustion also return `false`.

### Step 4: Create an Iterator from the Collection

```go
func (c *StringCollection) CreateIterator() Iterator {
	return &ListIterator{
		items: c.items,
	}
}
```

Each call creates a new iterator.

The omitted `index` field starts at its zero value, `0`.

The collection returns the `Iterator` interface, so clients depend on traversal behavior rather than `ListIterator` fields.

### Step 5: Traverse Through the Interface

```go
for {
	value, ok := iterator.Next()
	if !ok {
		break
	}

	fmt.Println(value)
}
```

The client only asks for the next value and stops when none remains.

It does not access the slice or maintain an index.

### Step 6: Understand Independent Iterators

```go
secondIterator := collection.CreateIterator()
```

This creates a fresh traversal starting at index `0`.

Advancing it does not change the first iterator’s position.

However, sharing a single iterator between goroutines would involve concurrent reads and writes to its index and would require synchronization.

### Step 7: Understand Shared Data

```go
items: c.items
```

This copies the slice header, not its underlying array.

Consequently:

- Changes to existing elements in the shared array can be visible to the iterator.
- The iterator retains the slice length captured when it was created.
- Later appends to the collection are not automatically included in that iterator.

This implementation assumes the collection remains unchanged during traversal.

If snapshot behavior is required, the factory can copy the elements when creating the iterator. That uses additional memory.

### Step 8: Adapt the Contract to the Workload

The `(string, bool)` result is sufficient for this in-memory example because traversal has no expected operational errors.

Database or network iterators need additional error reporting. Some also need a `Close()` operation to release resources.

Choose the iterator contract according to the collection and its failure modes.
