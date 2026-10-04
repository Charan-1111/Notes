# Adapter Design Pattern

The **Adapter pattern** is a structural design pattern that allows incompatible interfaces to work together.

It wraps an existing implementation and exposes the interface your application expects, without modifying the original implementation.

**Simple example:**

- Your application expects a printer with a `PrintText()` method.
- An old printer only provides a `PrintOld()` method.
- An adapter implements `PrintText()` and internally calls `PrintOld()`.

Think of a plug adapter: it connects a device to a socket that would otherwise be incompatible.

In Go, we typically implement this pattern using **interfaces, structs, and composition**.

## When to Use

- When an existing implementation does not match the interface your application needs.
- When reusing legacy code without modifying it.
- When integrating third-party libraries whose code you cannot change.

**Backend example:**

Your application expects:

```go
SendNotification(message string) error
```

But an external SMS library provides:

```go
PublishMessage(message string) error
```

An adapter can expose `SendNotification()` and internally call `PublishMessage()`.

This keeps the external library's details out of your application logic.

## Components

| Component | Responsibility | Printer example |
|---|---|---|
| **Target Interface** | Defines the operations the client expects. | `Printer`, which declares `PrintText()`. |
| **Adaptee** | Existing implementation with an incompatible interface. | `OldPrinter`, which provides `PrintOld()`. |
| **Adapter** | Implements the target interface and translates calls to the adaptee. | `PrinterAdapter`. |
| **Client** | Uses the target interface to perform an operation. | `PrintDocument()`. |

The client depends on the target interface. It does not need to understand the adaptee's methods or implementation.

## Working

1. The client calls a method through the target interface.
2. The adapter receives the call.
3. The adapter translates it into a call the adaptee understands.
4. The adaptee performs the operation.
5. If the operation returns a result or error, the adapter passes it back, converting it when necessary.

**Printer example:**

```text
PrintDocument() calls PrintText().
PrinterAdapter.PrintText() calls OldPrinter.PrintOld().
OldPrinter prints the text.
```

Translation may involve more than changing a method name. An adapter can also convert arguments, data formats, results, or errors.

## Pros

- **Reuses existing code:** Integrates legacy implementations and third-party libraries without changing their source.
- **Improves compatibility:** Connects implementations with different interfaces.
- **Supports the Open–Closed Principle:** Introduces adapters to support new implementations without changing client logic.
- **Reduces coupling:** The client depends on the target interface instead of a particular external implementation.
- **Centralizes translation:** Keeps integration details inside the adapter.

## Cons

- **Adds complexity:** Introduces another type and an extra layer of calls.
- **Requires maintenance:** Multiple integrations may require multiple adapters.
- **May add conversion overhead:** Data transformation can have a cost, although a simple forwarding call usually has negligible overhead.
- **Cannot remove behavioral differences:** Some APIs behave differently even after their interfaces are adapted.

**Example of a behavioral difference:**

Your application expects a notification to be sent immediately, but the external service only queues it. An adapter must represent this difference honestly; renaming the method alone does not solve it.

## Sample Implementation in Golang

Suppose our application expects `PrintText()`, but an existing printer only supports `PrintOld()`.

```go
package main

import "fmt"

// Target interface: what our application expects.
type Printer interface {
	PrintText(text string)
}

// Adaptee: an existing implementation.
type OldPrinter struct{}

func (op *OldPrinter) PrintOld(text string) {
	fmt.Println("Old Printer:", text)
}

// Adapter: wraps the existing printer.
type PrinterAdapter struct {
	oldPrinter *OldPrinter
}

// Implements Printer by translating the method call.
func (pa *PrinterAdapter) PrintText(text string) {
	pa.oldPrinter.PrintOld(text)
}

// Compile-time check that *PrinterAdapter implements Printer.
var _ Printer = (*PrinterAdapter)(nil)

// Client: depends only on the Printer interface.
func PrintDocument(printer Printer, text string) {
	printer.PrintText(text)
}

func main() {
	oldPrinter := &OldPrinter{}

	adapter := &PrinterAdapter{
		oldPrinter: oldPrinter,
	}

	PrintDocument(adapter, "Hello, Adapter pattern!")
}
```

**Output:**

```text
Old Printer: Hello, Adapter pattern!
```

### Step 1: Define the Target Interface

```go
type Printer interface {
	PrintText(text string)
}
```

This describes what the application needs.

Any type with a matching `PrintText(text string)` method satisfies `Printer` automatically. Go does not require an `implements` keyword.

### Step 2: Keep the Existing Implementation

```go
type OldPrinter struct{}

func (op *OldPrinter) PrintOld(text string) {
	fmt.Println("Old Printer:", text)
}
```

`OldPrinter` already performs the required operation.

However, it does not satisfy `Printer` because its method is named `PrintOld()`, not `PrintText()`.

We leave this implementation unchanged.

### Step 3: Wrap the Adaptee

```go
type PrinterAdapter struct {
	oldPrinter *OldPrinter
}
```

The adapter stores a pointer to the existing printer.

This is **composition**: one object contains and uses another object.

### Step 4: Translate the Call

```go
func (pa *PrinterAdapter) PrintText(text string) {
	pa.oldPrinter.PrintOld(text)
}
```

The adapter provides the method the application expects and delegates the work to the old printer.

Because this method has a pointer receiver, `*PrinterAdapter` satisfies `Printer`.

### Step 5: Verify Interface Compatibility

```go
var _ Printer = (*PrinterAdapter)(nil)
```

This asks the compiler to verify that `*PrinterAdapter` satisfies `Printer`.

It does not create a printer or call a method. If the required method is missing or has the wrong signature, compilation fails.

### Step 6: Let the Client Use the Interface

```go
func PrintDocument(printer Printer, text string) {
	printer.PrintText(text)
}
```

The client accepts any implementation of `Printer`.

It does not need to know whether it receives an adapter or a printer that directly implements the interface.

### Step 7: Connect the Objects

```go
oldPrinter := &OldPrinter{}

adapter := &PrinterAdapter{
	oldPrinter: oldPrinter,
}

PrintDocument(adapter, "Hello, Adapter pattern!")
```

We create the old printer, wrap it in the adapter, and pass the adapter to the client.

The application can now use the old printer through its expected interface.
