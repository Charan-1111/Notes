# Iterator Design Pattern
- Behavioural design pattern that provides a way to sequentially access elements of a collection ( like list, tree or custom aggregate ) without exposing its underlying representation.
- It separates the traversal logic from the collection itself, allowing clients to iterate over elements without depending on the collection's internal structure.

- ## When to Use
	- When we need to traverse a collection ( like list, tree, graph... ) in a consistent and flexible way.
	- When we want to support multiple ways to iterate ( eg :- forward, backward, filtering or skipping elements ).
	- When we want to decouple traversal logic from collection structure, so the client doesn't depend on the internal representation.

- ## Components
	- **Iterator Interface** - Declares methods for travelling or traversing the collection, such as **hasNext()** to check if more elements exists, and **next()** to get the next element.
	- **Concrete Iterator** - Implements the iterator interface and keeps track of the current position in the traversal.
	- **Aggregate Interface** - Defines a method to create an iterator.
	- **Concrete Aggregate** - Implements the aggregate interface and returns the instance of the concrete interface.

- ## Working
	1. Client requests an iterator from the collection ( aggregate ).
	2. Iterator abstracts the traversal mechanism, letting the client navigate through elements sequentially.
	3. Underlying collection's structure remains hidden from the client.
	4. Multiple iterators can independently traverse the same collection simultaneously.

- ## Pros
	- Encapsulation of traversal - Client doesn't know how the collection is implemented ( array, list, tree etc ).
	- Uniform access to collections - Different collections ( list, set, tree... ) can be traversed in the same way using iterators.
	- Supports multiple traversals - We can create multiple iterators to traverse the same collection simultaneously, as well as we can define multiple iteration strategies ( forward, backward, skip ... ).
	- Open/Closed Principle ( OCP ) - We can introduce new iterators ( like reverse or random order ) wihtout modifying the collection

- ## Cons
	- Increased complexity - Adds extra classes or interfaces ( Iterator, Aggregator ), which can feel like boilerplate for simple collections.
	- Overhead in small / simple cases - If we just need a simple for loop over an array, an iterator can be overkill.
	- Not always efficient - Some specialized collection operations ( like random access in arrays ) may be less efficient if forced through an iterator abstraction.
	- Can break encapsulation indirectly - Although the pattern hides the structure, iterators still expose elements one by one, which might allow unwanted modifications if not handled carefully.

- ## Sample implementation in Golang

```go
package main

import "fmt"

// Iterator interface
type Iterator interface {
	hasNext() bool
	next() interface{}
}

// Concrete Iterator
type ListIterator struct {
	items []string
	index int
}

func (it *ListIterator) hasNext() bool {
	return it.index < len(it.items)
}

func (it *ListIterator) next() interface{} {
	if it.hasNext() {
		value := it.items[it.index]
		it.index = it.index + 1
		return  value
	}
	return nil
}

// Aggregate ( collection )
type Aggregate interface {
	CreateIterator() Iterator
}

// Concrete aggregate
type StringCollection struct {
	items []string
}

func (c *StringCollection) CreateIterator() Iterator {
	return &ListIterator{
		items : c.items,
		index: 0,
	}
}

// Client Usage
func main() {
	collection := &StringCollection{
		items: []string{"A", "B", "C"},
	}

	iterator := collection.CreateIterator()

	for iterator.hasNext() {
		fmt.Println(iterator.next())
	}
}
```
