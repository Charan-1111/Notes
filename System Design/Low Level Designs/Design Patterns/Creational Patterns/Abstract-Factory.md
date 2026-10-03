# Abstract Factory Design Pattern

Abstract Factory is a **creational design pattern** that provides an interface for creating **families of related objects**, without requiring client code to know their concrete implementations.

### Simple Example

Suppose an application supports two UI styles:

| Family | Button | Checkbox |
|---|---|---|
| Windows | Windows Button | Windows Checkbox |
| Mac | Mac Button | Mac Checkbox |

When the application selects the **Mac factory**, it receives a Mac button and a Mac checkbox. Selecting the **Windows factory** produces the corresponding Windows products.

The client uses the same interfaces in both cases.

> Abstract Factory creates a family of related products through a common factory interface.

It is sometimes called a **“factory of factories”**, but that phrase can be misleading: an abstract factory usually creates **products**, not other factories.

## When to Use

### 1. Families of Related Objects

Use it when several objects belong together and should be created as a group.

**Example:** A UI toolkit needs buttons and checkboxes that follow the same platform style.

### 2. Consistency Matters

Use it when products should come from the same family.

**Example:** A Mac factory creates both Mac buttons and Mac checkboxes, keeping the appearance consistent.

**Important:** This consistency depends on correct factory implementations and how callers use them. Go does not automatically prevent someone from manually mixing Windows and Mac products.

### 3. Switching Between Variants

Use it when the application needs to select a family based on configuration or its environment.

**Example:**

- Select `MacFactory` on macOS.
- Select `WinFactory` on Windows.

Changing the factory affects **objects created afterward**. It does not convert existing objects into another family.

### 4. Decoupling from Concrete Implementations

Use it when client logic should depend on interfaces rather than specific types.

**Example:** A rendering function accepts a `UIFactory`. It does not need to know whether the factory creates Mac or Windows products.

### 5. Cross-Platform Systems

Use it when different platforms require different implementations of the same related components.

**Example:** Each platform provides its own button and checkbox implementations, while the application's rendering logic stays the same.

## Components

| Component | Responsibility | Example |
|---|---|---|
| Abstract Factory | Defines methods for creating related products | `UIFactory` |
| Concrete Factories | Create products belonging to a particular family | `WinFactory`, `MacFactory` |
| Abstract Products | Define the behavior available to clients | `Button`, `Checkbox` |
| Concrete Products | Implement that behavior for a family | `WinButton`, `WinCheckbox`, `MacButton`, `MacCheckbox` |

In Go:

- Abstract factories and abstract products are usually **interfaces**.
- Concrete factories and products are usually **structs with methods**.
- A type implements an interface implicitly by providing its required methods.

## Pros

### 1. Encourages Consistent Product Families

Using one factory to create all related products helps keep them in the same family.

**Example:** `MacFactory` returns a Mac button and a Mac checkbox.

### 2. Groups Related Creation Logic

The factory keeps the creation of related products in one place.

**Example:** Windows-specific product creation belongs inside `WinFactory`.

### 3. Decouples Client Logic

The client depends on interfaces rather than concrete product types.

**Example:** The client calls `button.Paint()` without knowing whether `button` is a `WinButton` or `MacButton`.

### 4. Makes Switching Families Straightforward

Pass a different factory to the same client function to create another family.

### 5. Supports the Open/Closed Principle for New Families

You can add another family, such as Linux, by introducing new products and a new factory.

Existing client logic can remain unchanged. Factory-selection code may still need updating.

## Cons

### 1. Additional Complexity

The pattern introduces more interfaces, structs, and methods.

For an application that creates only one simple object, this may be unnecessary.

### 2. Adding New Product Types Is Harder

Suppose every family must now provide a `Slider`.

You need to:

- Define a `Slider` interface.
- Implement Windows and Mac sliders.
- Add `CreateSlider()` to `UIFactory`.
- Update every concrete factory.

> Adding a new family is usually easy. Adding a new product type affects every factory.

### 3. Indirection Can Make Tracing Harder

The client sees a `Button` interface, so identifying the concrete implementation requires checking which factory was selected.

### 4. Assumes a Shared Product Structure

Each factory must provide the products required by the factory interface.

The pattern becomes awkward when different families need substantially different sets of products.

## Sample Implementation in Go

We will build two product families:

- **Windows:** `WinButton` + `WinCheckbox`
- **Mac:** `MacButton` + `MacCheckbox`

### Step 1: Define Product Interfaces

```go
type Button interface {
	Paint()
}

type Checkbox interface {
	Paint()
}
```

**Explanation:**

1. `Button` defines the behavior expected from any button.
2. `Checkbox` defines the behavior expected from any checkbox.
3. Both expose `Paint()`, which represents rendering the component.
4. Clients can call these methods without depending on a concrete product type.

For this small example, both interfaces have the same method set. Therefore, Go treats their implementations as structurally interchangeable. Real product interfaces can include distinct operations when those operations are meaningful.

### Step 2: Implement Concrete Products

```go
// Windows products.
type WinButton struct{}

func (WinButton) Paint() {
	fmt.Println("Rendering Windows button")
}

type WinCheckbox struct{}

func (WinCheckbox) Paint() {
	fmt.Println("Rendering Windows checkbox")
}

// Mac products.
type MacButton struct{}

func (MacButton) Paint() {
	fmt.Println("Rendering Mac button")
}

type MacCheckbox struct{}

func (MacCheckbox) Paint() {
	fmt.Println("Rendering Mac checkbox")
}
```

**Explanation:**

1. Each struct represents a concrete UI component.
2. Its `Paint()` method provides the platform-specific behavior.
3. `WinButton` and `MacButton` satisfy the `Button` interface.
4. `WinCheckbox` and `MacCheckbox` satisfy the `Checkbox` interface.
5. Go requires no `implements` keyword.

The empty structs are sufficient because this demonstration does not store component state.

### Step 3: Define the Abstract Factory

```go
type UIFactory interface {
	CreateButton() Button
	CreateCheckbox() Checkbox
}
```

**Explanation:**

1. `UIFactory` defines how to create the related products.
2. `CreateButton()` returns the `Button` interface.
3. `CreateCheckbox()` returns the `Checkbox` interface.
4. The interface does not specify which concrete products must be returned.

### Step 4: Implement Concrete Factories

```go
type WinFactory struct{}

func (WinFactory) CreateButton() Button {
	return WinButton{}
}

func (WinFactory) CreateCheckbox() Checkbox {
	return WinCheckbox{}
}

type MacFactory struct{}

func (MacFactory) CreateButton() Button {
	return MacButton{}
}

func (MacFactory) CreateCheckbox() Checkbox {
	return MacCheckbox{}
}
```

**Explanation:**

1. `WinFactory` creates Windows products.
2. `MacFactory` creates Mac products.
3. Both implement the methods required by `UIFactory`.
4. The method signatures return interfaces, while the returned values are concrete products.

For example:

```go
func (MacFactory) CreateButton() Button {
	return MacButton{}
}
```

This is valid because `MacButton` provides the `Paint()` method required by `Button`.

### Step 5: Write Reusable Client Code

```go
func RenderUI(factory UIFactory) {
	button := factory.CreateButton()
	checkbox := factory.CreateCheckbox()

	button.Paint()
	checkbox.Paint()
}
```

**Explanation:**

1. `RenderUI` receives a factory through the `UIFactory` interface.
2. It asks that factory to create a button and a checkbox.
3. It calls the behavior exposed by their interfaces.
4. It does not construct or reference Windows or Mac product types.

The chosen factory determines which products this function uses.

### Step 6: Select the Product Family

```go
func main() {
	var factory UIFactory = MacFactory{}

	fmt.Println("Mac UI:")
	RenderUI(factory)

	factory = WinFactory{}

	fmt.Println("\nWindows UI:")
	RenderUI(factory)
}
```

**Explanation:**

1. `factory` initially holds a `MacFactory`.
2. The first `RenderUI` call creates and paints Mac products.
3. We assign a `WinFactory` to the same interface variable.
4. The second call creates and paints Windows products.

Changing `factory` does not modify the products created by the first call.

### Complete Runnable Code

```go
package main

import "fmt"

// Abstract products.
type Button interface {
	Paint()
}

type Checkbox interface {
	Paint()
}

// Windows products.
type WinButton struct{}

func (WinButton) Paint() {
	fmt.Println("Rendering Windows button")
}

type WinCheckbox struct{}

func (WinCheckbox) Paint() {
	fmt.Println("Rendering Windows checkbox")
}

// Mac products.
type MacButton struct{}

func (MacButton) Paint() {
	fmt.Println("Rendering Mac button")
}

type MacCheckbox struct{}

func (MacCheckbox) Paint() {
	fmt.Println("Rendering Mac checkbox")
}

// Abstract factory.
type UIFactory interface {
	CreateButton() Button
	CreateCheckbox() Checkbox
}

// Windows factory.
type WinFactory struct{}

func (WinFactory) CreateButton() Button {
	return WinButton{}
}

func (WinFactory) CreateCheckbox() Checkbox {
	return WinCheckbox{}
}

// Mac factory.
type MacFactory struct{}

func (MacFactory) CreateButton() Button {
	return MacButton{}
}

func (MacFactory) CreateCheckbox() Checkbox {
	return MacCheckbox{}
}

// Client logic depends on interfaces.
func RenderUI(factory UIFactory) {
	button := factory.CreateButton()
	checkbox := factory.CreateCheckbox()

	button.Paint()
	checkbox.Paint()
}

func main() {
	var factory UIFactory = MacFactory{}

	fmt.Println("Mac UI:")
	RenderUI(factory)

	factory = WinFactory{}

	fmt.Println("\nWindows UI:")
	RenderUI(factory)
}
```

### Expected Output

```text
Mac UI:
Rendering Mac button
Rendering Mac checkbox

Windows UI:
Rendering Windows button
Rendering Windows checkbox
```

The first pair comes from `MacFactory`; the second pair comes from `WinFactory`. Both use the same `RenderUI` function.
