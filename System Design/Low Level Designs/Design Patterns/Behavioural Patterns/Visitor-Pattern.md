# Visitor Design Pattern

The **Visitor pattern** is a behavioral design pattern that lets you add operations to a set of related types without editing those types each time you add an operation.

For example, `Circle` and `Rectangle` hold their own dimensions. An `AreaVisitor` calculates their areas, and a `PerimeterVisitor` calculates their perimeters. Once both shapes support `Accept()`, you can add another operation, such as exporting shapes, by writing another visitor.

**The main trade-off:** Visitor makes new operations easy to add, but new element types are harder to add.

## When to Use

- When the element types are relatively stable but the operations on them change frequently.
- When one operation needs different logic for each concrete type.
- When you want to keep all the logic for one operation together instead of spreading it across several types.

**Backend example:** A Go source analysis tool might run several operations over the same syntax tree: collecting function names, detecting unused imports, or generating documentation. Each operation can handle the node types it cares about separately.

If you have only one small operation over a few types, a type switch may be simpler. If area is a core capability of every shape and you expect to add shapes often, an `Area()` method on `Shape` may be a better fit.

## Components

| Component | Responsibility | Shape example |
|---|---|---|
| **Element Interface** | Lets callers apply a visitor to an element. | `Shape`, with `Accept(Visitor)` |
| **Concrete Elements** | Pass themselves to the matching visitor method. | `Circle` and `Rectangle` |
| **Visitor Interface** | Defines a method for each supported element type. | `VisitCircle(*Circle)` and `VisitRectangle(*Rectangle)` |
| **Concrete Visitors** | Implement one operation across the supported element types. | `AreaVisitor` and `PerimeterVisitor` |
| **Object Structure** | Holds or organizes the elements to visit. | A `[]Shape`; no separate container type is needed here. |

Go uses distinct method names such as `VisitCircle()` and `VisitRectangle()` because it does not support overloading methods by parameter type.

## Working

Suppose a `Shape` contains a `*Circle` and the client calls `shape.Accept(areaVisitor)`:

1. The concrete **shape** determines which `Accept()` method runs: `(*Circle).Accept()`.
2. `Circle.Accept()` calls `v.VisitCircle(c)`, passing the circle itself.
3. The concrete **visitor** determines which `VisitCircle()` implementation runs: `AreaVisitor.VisitCircle()`.
4. The visitor uses the circle’s radius to calculate its area.

This is **double dispatch**: the result depends on both the concrete element and the concrete visitor. Pass a `PerimeterVisitor` to the same circle, and its `VisitCircle()` method runs instead.

**Traversal and visiting are separate.** The loop chooses which elements to visit and in what order. The visitor decides what operation to perform on each element.

## Pros

- **Easy to add operations:** Add a visitor without modifying elements that already support `Accept()`.
- **Keeps related logic together:** All area calculations live in `AreaVisitor`.
- **Reuses the same structure:** Different visitors can operate on the same collection or tree.
- **Avoids repeated type checks:** Each element calls its matching visitor method directly.

## Cons

- **Harder to add element types:** Adding `Triangle` requires adding `VisitTriangle()` to the visitor interface and updating existing visitors.
- **Adds indirection:** To trace an operation, follow `Accept()` to the matching visitor method.
- **Can pressure encapsulation:** A visitor in another package may need accessors for the element data it uses.
- **Can be excessive for simple cases:** A direct method or small type switch may be clearer.

## Sample Implementation in Golang

This example applies two visitors to the same set of shapes. It uses `math.Pi` for circle calculations and rounds only the displayed output.

```go
package main

import (
	"fmt"
	"math"
)

// Visitor: one method for each supported shape type.
type Visitor interface {
	VisitCircle(*Circle)
	VisitRectangle(*Rectangle)
}

// Element: a shape that accepts a visitor.
type Shape interface {
	Accept(Visitor)
}

type Circle struct {
	radius float64
}

func (c *Circle) Accept(v Visitor) {
	v.VisitCircle(c)
}

type Rectangle struct {
	width  float64
	height float64
}

func (r *Rectangle) Accept(v Visitor) {
	v.VisitRectangle(r)
}

// AreaVisitor calculates and prints each shape's area.
type AreaVisitor struct{}

func (AreaVisitor) VisitCircle(c *Circle) {
	area := math.Pi * c.radius * c.radius
	fmt.Printf("Circle area: %.2f\n", area)
}

func (AreaVisitor) VisitRectangle(r *Rectangle) {
	area := r.width * r.height
	fmt.Printf("Rectangle area: %.2f\n", area)
}

// PerimeterVisitor calculates and prints each shape's perimeter.
type PerimeterVisitor struct{}

func (PerimeterVisitor) VisitCircle(c *Circle) {
	perimeter := 2 * math.Pi * c.radius
	fmt.Printf("Circle perimeter: %.2f\n", perimeter)
}

func (PerimeterVisitor) VisitRectangle(r *Rectangle) {
	perimeter := 2 * (r.width + r.height)
	fmt.Printf("Rectangle perimeter: %.2f\n", perimeter)
}

func main() {
	shapes := []Shape{
		&Circle{radius: 5},
		&Rectangle{width: 4, height: 5},
	}

	area := AreaVisitor{}
	perimeter := PerimeterVisitor{}

	for _, shape := range shapes {
		shape.Accept(area)
		shape.Accept(perimeter)
	}
}
```

**Expected output:**

```text
Circle area: 78.54
Circle perimeter: 31.42
Rectangle area: 20.00
Rectangle perimeter: 18.00
```

### Step-by-Step Explanation

1. **Define the two contracts.** `Shape` exposes `Accept()`, so the client can handle circles and rectangles uniformly. `Visitor` exposes one method per shape type, so each visitor receives a concrete value without a type assertion.

2. **Let each shape select the matching method.** `Circle.Accept()` calls `VisitCircle(c)`, while `Rectangle.Accept()` calls `VisitRectangle(r)`. Both methods pass the shape itself to the visitor. Since `Accept()` uses pointer receivers, the slice contains `*Circle` and `*Rectangle`.

3. **Give each visitor one operation.** `AreaVisitor` knows how to calculate the area of both shapes. `PerimeterVisitor` does the same for perimeter. The shapes provide their dimensions; the visitors supply the formulas.

4. **Apply both visitors during traversal.** For each shape, `main()` calls `Accept()` twice. The shape stays the same, but switching the visitor changes the operation.

5. **Add operations by implementing `Visitor`.** For example, an `ExportVisitor` would implement both visit methods. It could collect export records in a slice and let the caller retrieve them after traversal. `Circle` and `Rectangle` would need no changes.
