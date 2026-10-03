# Prototype Design Pattern

Prototype is a **creational design pattern** that creates new objects by cloning existing objects instead of initializing them from scratch.

The existing object acts as a **prototype**: a starting point for creating similar objects.

### Simple Example

Suppose we already have a circle:

```text
Original: Radius = 5, Color = Red
```

We clone it and change the clone’s color:

```text
Original: Radius = 5, Color = Red
Clone:    Radius = 5, Color = Green
```

Both initially have the same state, but they are separate objects.

> Prototype lets us reuse an existing object's state and customize the copy.

Cloning does not rerun the original initialization logic. It copies state according to the rules implemented in `Clone()`.

## When to Use

### 1. Initialization Is Expensive

Use it when initialization performs expensive work and the resulting state can safely be copied.

**Example:** Load and prepare a complex template once, then clone its prepared configuration for different users.

Cloning is not automatically faster than constructing an object. Its benefit depends on the initialization cost and the amount of data being copied.

### 2. Reusing Complex Initial Configuration

Use it when several objects need the same initial settings.

**Example:** Create a configured report prototype containing a title, layout, colors, and default options. Clone it for each new report.

The clone reuses the configuration without repeating the setup.

### 3. Creating Many Similar Objects

Use it when objects differ in only a few fields.

**Example:** Clone a red circle and change its color to create a green circle with the same radius.

For a small struct, ordinary construction may be simpler. Prototype becomes useful when copying rules or initialization are more involved.

## Components

| Component | Responsibility | Go example |
|---|---|---|
| Prototype Interface | Defines the cloning operation | `Shape` |
| Concrete Prototype | Implements how an object copies itself | `Circle`, `Rectangle` |
| Client | Requests and uses clones | `main()` |

In Go, the prototype interface can look like this:

```go
type Shape interface {
	Clone() Shape
	Draw()
}
```

- `Clone()` returns a copy through the `Shape` interface.
- `Draw()` exposes behavior shared by the shapes.
- Each concrete type decides how its state should be copied.

## Pros

### 1. Reuses Prepared State

Cloning can avoid repeating expensive initialization.

**Example:** Copy a prepared template instead of loading and configuring it again.

### 2. Centralizes Copying Logic

The copying rules live inside `Clone()`.

Callers do not need to remember which fields require independent copies.

### 3. Supports Runtime Flexibility

An application can select an existing prototype at runtime and clone it.

**Example:** Store named shape prototypes in a map and select one based on user input.

### 4. Reduces Types Created Only for Configuration Variants

You do not need separate types such as `RedCircle`, `GreenCircle`, and `BlueCircle`.

Use one `Circle` type, clone an existing circle, and change its color.

### 5. Supports Cloning Through an Interface

A client can call `Clone()` without knowing the concrete shape type.

However, changing a concrete field such as `Radius` requires access to the concrete type or another suitable interface.

## Cons

### 1. Cloning Can Be Complex

Simple fields are easy to copy. Nested pointers, slices, and maps require more care.

You must decide which data should be independent and which data may remain shared.

### 2. Shallow Copies Can Share Mutable Data

In Go, copying a struct copies its field values.

For fields such as integers and strings, this is sufficient for independent field updates. But slices, maps, and pointers may still refer to shared data.

#### Shallow Copy Example

```go
type Template struct {
	Name string
	Tags []string
}

original := Template{
	Name: "Default",
	Tags: []string{"backend", "go"},
}

cloned := original
cloned.Tags[0] = "ai"

fmt.Println(original.Tags[0]) // ai
```

**Explanation:**

1. `cloned := original` copies the struct.
2. The copied slice still refers to the same underlying array.
3. Updating an element through `cloned.Tags` also changes what `original.Tags` sees.

#### Independent Copy of the Slice

```go
cloned := original
cloned.Tags = append([]string(nil), original.Tags...)
```

This creates a separate slice backing array and copies its elements.

Because these elements are strings, this is enough for independent element updates. A slice containing pointers or other nested mutable data may require additional copying.

### 3. Memory Overhead

Independent copies of large data structures can consume substantial memory.

Cloning a large object many times may be more expensive than expected.

### 4. Some Fields Cannot Be Copied Safely

Objects containing mutexes, active connections, or other resources need special handling.

For example, a `sync.Mutex` must not be copied after it has been used.

A clone method should explicitly define whether such resources are recreated, shared safely, or excluded from cloning.

### 5. Prototype Changes Affect Future Clones

If the prototype’s configuration changes, future clones use its updated state.

Existing independent clones remain unchanged. However, shared mutable data in shallow copies can allow changes to affect multiple objects.

## Sample Implementation in Golang

We will implement:

- A `Circle` prototype.
- A `Rectangle` prototype.
- A `Shape` interface for cloning and drawing.

These structs contain only integers and strings, so a struct copy is sufficient.

### Step 1: Define the Prototype Interface

```go
type Shape interface {
	Clone() Shape
	Draw()
}
```

**Explanation:**

1. `Clone()` returns a cloned object as a `Shape`.
2. `Draw()` provides common behavior.
3. Both `Circle` and `Rectangle` will implement these methods.

### Step 2: Implement the Circle Prototype

```go
type Circle struct {
	Radius int
	Color  string
}

func (c *Circle) Clone() Shape {
	cloned := *c
	return &cloned
}

func (c *Circle) Draw() {
	fmt.Printf(
		"Circle: radius=%d, color=%s\n",
		c.Radius,
		c.Color,
	)
}
```

**Explanation:**

1. `c` points to the original circle.
2. `*c` reads the struct value that the pointer refers to.
3. `cloned := *c` copies that struct into a new variable.
4. `&cloned` returns a pointer to the copy.
5. Updating the clone’s `Radius` or `Color` does not change the original.

Returning a pointer to a local variable is safe in Go. Go keeps the value alive for as long as it is needed.

**Why return `&cloned` instead of `cloned`?**

Both methods use pointer receivers:

```go
func (c *Circle) Clone() Shape
func (c *Circle) Draw()
```

Therefore, `*Circle` implements `Shape`, while `Circle` does not.

The returned value must satisfy the interface.

### Step 3: Implement the Rectangle Prototype

```go
type Rectangle struct {
	Width  int
	Height int
	Color  string
}

func (r *Rectangle) Clone() Shape {
	cloned := *r
	return &cloned
}

func (r *Rectangle) Draw() {
	fmt.Printf(
		"Rectangle: width=%d, height=%d, color=%s\n",
		r.Width,
		r.Height,
		r.Color,
	)
}
```

**Explanation:**

1. `Rectangle` stores its dimensions and color.
2. `cloned := *r` copies those fields.
3. `&cloned` returns a pointer to the new rectangle.
4. The clone can be modified independently.

Both clone methods assume the receiver is non-nil.

### Step 4: Create the Original Objects

```go
circle1 := &Circle{
	Radius: 5,
	Color:  "Red",
}

rectangle1 := &Rectangle{
	Width:  5,
	Height: 4,
	Color:  "Blue",
}
```

These objects become the prototypes we will clone.

### Step 5: Clone the Objects

```go
circle2 := circle1.Clone().(*Circle)
rectangle2 := rectangle1.Clone().(*Rectangle)
```

**Explanation:**

1. `Clone()` returns a value with the interface type `Shape`.
2. Its underlying concrete value is a `*Circle` or `*Rectangle`.
3. `.(*Circle)` and `.(*Rectangle)` are **type assertions**.
4. They let us access concrete fields such as `Color` and `Width`.

These assertions are valid because we know what these clone methods return. An incorrect single-value assertion causes a panic.

When the concrete type is uncertain, use the checked form:

```go
circle, ok := shape.(*Circle)
if ok {
	circle.Color = "Green"
}
```

If we only need shared behavior, no assertion is necessary:

```go
clonedShape := circle1.Clone()
clonedShape.Draw()
```

### Step 6: Modify the Clones

```go
circle2.Color = "Green"

rectangle2.Width = 10
rectangle2.Color = "Yellow"
```

Only the cloned objects change.

- `circle1` remains red.
- `rectangle1` remains blue with width `5`.

### Step 7: Display Originals and Clones

```go
fmt.Println("Original objects:")
circle1.Draw()
rectangle1.Draw()

fmt.Println("\nCloned objects:")
circle2.Draw()
rectangle2.Draw()
```

This demonstrates that the clones retain unchanged fields while allowing independent modifications.

### Complete Runnable Code

```go
package main

import "fmt"

// Prototype interface.
type Shape interface {
	Clone() Shape
	Draw()
}

// Concrete prototype: Circle.
type Circle struct {
	Radius int
	Color  string
}

func (c *Circle) Clone() Shape {
	cloned := *c
	return &cloned
}

func (c *Circle) Draw() {
	fmt.Printf(
		"Circle: radius=%d, color=%s\n",
		c.Radius,
		c.Color,
	)
}

// Concrete prototype: Rectangle.
type Rectangle struct {
	Width  int
	Height int
	Color  string
}

func (r *Rectangle) Clone() Shape {
	cloned := *r
	return &cloned
}

func (r *Rectangle) Draw() {
	fmt.Printf(
		"Rectangle: width=%d, height=%d, color=%s\n",
		r.Width,
		r.Height,
		r.Color,
	)
}

func main() {
	// Create the original objects.
	circle1 := &Circle{
		Radius: 5,
		Color:  "Red",
	}

	rectangle1 := &Rectangle{
		Width:  5,
		Height: 4,
		Color:  "Blue",
	}

	// Clone the original objects.
	circle2 := circle1.Clone().(*Circle)
	rectangle2 := rectangle1.Clone().(*Rectangle)

	// Modify the clones.
	circle2.Color = "Green"

	rectangle2.Width = 10
	rectangle2.Color = "Yellow"

	// Display originals and clones.
	fmt.Println("Original objects:")
	circle1.Draw()
	rectangle1.Draw()

	fmt.Println("\nCloned objects:")
	circle2.Draw()
	rectangle2.Draw()
}
```

### Expected Output

```text
Original objects:
Circle: radius=5, color=Red
Rectangle: width=5, height=4, color=Blue

Cloned objects:
Circle: radius=5, color=Green
Rectangle: width=10, height=4, color=Yellow
```

The clones start with the originals’ state. Their modifications leave the originals unchanged because these structs contain no shared mutable fields.
