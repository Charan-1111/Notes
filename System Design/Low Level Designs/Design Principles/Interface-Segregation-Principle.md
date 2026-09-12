# Interface Segregation Principle (ISP) in Go

## What ISP Means

The **Interface Segregation Principle** states:

> No client should be forced to depend on methods it does not use.

In simple terms, prefer **small, focused interfaces** over one large interface containing unrelated methods.

An implementation should provide only the behavior it actually supports.

## Simple Real-World Example

Imagine two devices:

- An old printer can only print.
- A modern printer can print, scan, and fax.

If both devices must implement an interface containing `Print()`, `Scan()`, and `Fax()`, the old printer is forced to provide meaningless scan and fax methods.

ISP solves this by separating the behaviors into `Printer`, `Scanner`, and `Faxer` interfaces.

## Why ISP Matters

A large interface creates unnecessary dependencies. If an unrelated method is added or changed, every implementation may need to be updated.

Small interfaces make code easier to:

- Implement
- Test
- Change
- Reuse

In Go, a useful guideline is:

> Define interfaces around what the client needs, not everything a type can do.

## When to Use ISP

Use ISP when:

- An interface contains several unrelated methods.
- Different clients need different subsets of behavior.
- A struct needs empty, dummy, or unsupported methods to satisfy an interface.
- A small interface change affects many unrelated implementations.

## How ISP Is Violated

ISP is commonly violated by:

- Creating **fat interfaces** with unrelated responsibilities.
- Forcing structs to implement methods they do not support.
- Designing interfaces around a large concrete type instead of client needs.
- Adding methods to a widely used interface without considering its implementations.

## How to Apply ISP in Go

- Split large interfaces into small, focused interfaces.
- Keep each interface centered on one capability.
- Define interfaces close to the code that consumes them.
- Compose small interfaces when a client genuinely needs several capabilities.
- Apply the Single Responsibility Principle to interface design.

## Advantages

- Reduces unnecessary dependencies.
- Makes implementations simpler and more flexible.
- Makes mocks and unit tests smaller.
- Limits the effect of interface changes.
- Improves maintainability and reusability.

## Disadvantages

- Can create many small interfaces.
- Poorly chosen boundaries may fragment the design.
- May add unnecessary complexity in very small programs.

The goal is not to make every interface contain exactly one method. The goal is to group methods that belong together from the client's point of view.

## Go Example

### Violating ISP: A Fat Interface

```go
package main

import "fmt"

type MultiFunctionDevice interface {
	Print(document string)
	Scan(document string)
	Fax(document string)
}

type OldPrinter struct{}

func (OldPrinter) Print(document string) {
	fmt.Println("Printing:", document)
}

func (OldPrinter) Scan(document string) {
	fmt.Println("Scan is not supported")
}

func (OldPrinter) Fax(document string) {
	fmt.Println("Fax is not supported")
}
```

### Step-by-Step Explanation

1. `MultiFunctionDevice` requires printing, scanning, and faxing.
2. `OldPrinter` supports only printing.
3. Go still requires `OldPrinter` to implement all three methods before it can satisfy the interface.
4. Its `Scan()` and `Fax()` methods therefore contain behavior the device cannot perform.
5. The interface is designed around a multifunction device, not around what each client needs.

This violates ISP because `OldPrinter` is forced to depend on methods it does not need.

### Applying ISP: Small, Focused Interfaces

```go
package main

import "fmt"

type Printer interface {
	Print(document string)
}

type Scanner interface {
	Scan(document string)
}

type Faxer interface {
	Fax(document string)
}

type OldPrinter struct{}

func (OldPrinter) Print(document string) {
	fmt.Println("Printing:", document)
}

type ModernPrinter struct{}

func (ModernPrinter) Print(document string) {
	fmt.Println("Printing:", document)
}

func (ModernPrinter) Scan(document string) {
	fmt.Println("Scanning:", document)
}

func (ModernPrinter) Fax(document string) {
	fmt.Println("Faxing:", document)
}

func printDocument(printer Printer, document string) {
	printer.Print(document)
}

func scanDocument(scanner Scanner, document string) {
	scanner.Scan(document)
}

func main() {
	oldPrinter := OldPrinter{}
	modernPrinter := ModernPrinter{}

	printDocument(oldPrinter, "Report")
	printDocument(modernPrinter, "Invoice")
	scanDocument(modernPrinter, "Contract")
}
```

### Step-by-Step Explanation

1. `Printer`, `Scanner`, and `Faxer` each represent one focused capability.
2. `OldPrinter` implements only `Print()`, so it satisfies `Printer` without dummy methods.
3. `ModernPrinter` implements all three methods, so it satisfies all three interfaces automatically.
4. `printDocument()` asks only for a `Printer` because printing is all it needs.
5. `scanDocument()` asks only for a `Scanner` because scanning is all it needs.
6. Go's implicit interface implementation lets both devices satisfy the appropriate interfaces without an `implements` keyword.

The client depends on the smallest interface that provides the behavior it requires.

## Interface Composition in Go

Sometimes a client truly needs multiple capabilities. Small interfaces can be combined through embedding:

```go
type PrinterScanner interface {
	Printer
	Scanner
}

func copyDocument(device PrinterScanner, document string) {
	device.Scan(document)
	device.Print(document)
}
```

### Step-by-Step Explanation

1. `PrinterScanner` embeds the `Printer` and `Scanner` interfaces.
2. A type must have both `Print()` and `Scan()` to satisfy it.
3. `ModernPrinter` satisfies it because it implements both methods.
4. `OldPrinter` does not satisfy it, which is correct because it cannot scan.
5. `copyDocument()` requests both capabilities because its work genuinely requires both.

Composition lets us keep the basic interfaces small while supporting clients with broader requirements.

## Key Takeaways

- Clients should not depend on methods they do not use.
- Prefer small interfaces based on client needs.
- Avoid dummy or unsupported method implementations.
- Compose focused interfaces when several related capabilities are required.
- In Go, interfaces are implemented implicitly, making this principle natural to apply.

> **Practical rule:** Accept the smallest interface your function needs.