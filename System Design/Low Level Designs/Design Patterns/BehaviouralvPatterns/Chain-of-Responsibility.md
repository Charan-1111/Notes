# Chain of Responsibility Design Pattern

The **Chain of Responsibility pattern** is a behavioral design pattern that passes a request through a sequence of handlers.

Each handler decides whether to:

- Handle the request.
- Forward it to the next handler.

The client sends the request to the first handler without needing to know which handler will process it.

**Simple example:**

We have three logging handlers connected in this order:

**INFO → DEBUG → ERROR**

For an `ERROR` request:

1. The INFO handler does not support it and forwards it.
2. The DEBUG handler does not support it and forwards it.
3. The ERROR handler processes it.

This example uses **first-match handling**: the chain stops when a handler accepts the request.

## When to Use

- When several handlers could process a request, and the appropriate handler is determined at runtime.
- When the sender should not depend on a particular handler.
- When handlers need to be added, removed, or reordered without changing how the client submits requests.

**Backend example:**

A support request passes through:

- Basic support.
- Technical support.
- Specialist support.

Each handler resolves requests within its scope and forwards unsupported requests.

Another variation lets handlers perform a step and continue, such as a middleware pipeline containing validation, authentication, and request processing.

## Components

| Component | Responsibility | Logging example |
|---|---|---|
| **Handler Interface** | Defines request handling and chain configuration. | `Logger`. |
| **Concrete Handlers** | Handle supported requests or forward them. | `InfoLogger`, `DebugLogger`, and `ErrorLogger`. |
| **Client** | Sends requests to the first handler. | `main()`. |

Our example also uses a `BaseLogger` struct to store the next handler and reuse forwarding logic.

In Go, the interface defines methods; the struct stores the reference to the next handler.

## Working

1. The client sends a request to the first handler.
2. The handler checks whether it supports the request.
3. If supported, it processes the request and returns `true`.
4. Otherwise, it forwards the request to the next handler.
5. If the chain ends without handling it, the result is `false`.

**Important distinctions:**

- Forwarding means the current handler does not accept the request.
- Retrying means repeating an operation after failure.
- An unhandled request needs an explicit policy, such as reporting it or using a default handler.

The pattern does not automatically provide retries or fallback behavior.

## Pros

- **Decouples sender and receiver:** The client does not select the concrete handler.
- **Supports flexible chains:** Handlers can be added, removed, or reordered during setup.
- **Encourages focused responsibilities:** Each handler handles a specific concern.
- **Supports extension:** New handlers can be introduced without modifying existing handlers.
- **Makes unhandled requests explicit:** A result or terminal handler can define what happens when nothing matches.

Changing the chain still requires updating its configuration, but the client’s request-submission code can remain the same.

## Cons

- **Requests may go unhandled:** No handler is guaranteed to accept every request.
- **Order can affect results:** When multiple handlers match, the first one wins in a first-match chain.
- **Tracing can be harder:** A request may pass through several handlers.
- **Long chains add work:** A request may visit every handler before finding a match.
- **Cycles can break execution:** A handler forwarding back to an earlier handler can cause unbounded recursion.

For a chain of `n` handlers, a request may require up to O(n) handler checks. Recursive forwarding can also use O(n) call-stack depth.

## Sample Implementation in Golang

This example handles exact logging levels and explicitly reports unsupported levels.

```go
package main

import "fmt"

// Handler interface.
// Log returns true when a handler accepts the request.
type Logger interface {
	SetNext(Logger)
	Log(level, message string) bool
}

// Shared chaining behavior.
type BaseLogger struct {
	next Logger
}

func (b *BaseLogger) SetNext(next Logger) {
	b.next = next
}

func (b *BaseLogger) forward(level, message string) bool {
	if b.next == nil {
		return false
	}

	return b.next.Log(level, message)
}

// Concrete handler: INFO.
type InfoLogger struct {
	BaseLogger
}

func (i *InfoLogger) Log(level, message string) bool {
	if level == "INFO" {
		fmt.Printf("INFO: %s\n", message)
		return true
	}

	return i.forward(level, message)
}

// Concrete handler: DEBUG.
type DebugLogger struct {
	BaseLogger
}

func (d *DebugLogger) Log(level, message string) bool {
	if level == "DEBUG" {
		fmt.Printf("DEBUG: %s\n", message)
		return true
	}

	return d.forward(level, message)
}

// Concrete handler: ERROR.
type ErrorLogger struct {
	BaseLogger
}

func (e *ErrorLogger) Log(level, message string) bool {
	if level == "ERROR" {
		fmt.Printf("ERROR: %s\n", message)
		return true
	}

	return e.forward(level, message)
}

// Client.
func main() {
	info := &InfoLogger{}
	debug := &DebugLogger{}
	errorLog := &ErrorLogger{}

	// Build the chain: INFO → DEBUG → ERROR.
	info.SetNext(debug)
	debug.SetNext(errorLog)

	var logger Logger = info

	logger.Log("INFO", "This is an info message")
	logger.Log("DEBUG", "Debugging mode enabled")
	logger.Log("ERROR", "Something went wrong")

	if !logger.Log("WARNING", "Unknown log level") {
		fmt.Println("Unhandled log level: WARNING")
	}
}
```

**Output:**

```text
INFO: This is an info message
DEBUG: Debugging mode enabled
ERROR: Something went wrong
Unhandled log level: WARNING
```

### Step 1: Define the Handler Interface

```go
type Logger interface {
	SetNext(Logger)
	Log(level, message string) bool
}
```

- `SetNext()` connects a handler to another handler.
- `Log()` processes or forwards the request.
- The boolean reports whether any handler accepted it.

This boolean represents handled versus unhandled. It does not represent operational errors; real implementations may need an additional `error` result.

### Step 2: Store the Next Handler

```go
type BaseLogger struct {
	next Logger
}
```

The field uses the `Logger` interface, so it can reference any handler implementing that interface.

```go
func (b *BaseLogger) SetNext(next Logger) {
	b.next = next
}
```

This assigns the next link in the chain.

### Step 3: Reuse Forwarding Logic

```go
func (b *BaseLogger) forward(level, message string) bool {
	if b.next == nil {
		return false
	}

	return b.next.Log(level, message)
}
```

If another handler exists, the request is forwarded.

Otherwise, the chain has ended and returns `false`.

The result travels back through the earlier handlers to the client.

### Step 4: Implement Each Handler’s Decision

```go
func (i *InfoLogger) Log(level, message string) bool {
	if level == "INFO" {
		fmt.Printf("INFO: %s\n", message)
		return true
	}

	return i.forward(level, message)
}
```

The INFO handler accepts only `"INFO"`.

It returns immediately after handling the request. Unsupported levels continue through the chain.

DEBUG and ERROR follow the same structure.

### Step 5: Understand Embedding

```go
type InfoLogger struct {
	BaseLogger
}
```

Embedding lets `InfoLogger` reuse the base struct’s chaining methods.

Go embedding is composition, not class inheritance.

`*InfoLogger` satisfies `Logger` through its own `Log()` method and the promoted `SetNext()` method.

`BaseLogger` alone does not satisfy `Logger`, because it has no `Log()` method.

### Step 6: Connect the Handlers

```go
info.SetNext(debug)
debug.SetNext(errorLog)
```

The resulting order is INFO, then DEBUG, then ERROR.

`errorLog.next` remains nil, making it the final handler.

Adding a new handler requires implementing `Logger` and connecting it into the chain.

### Step 7: Trace an ERROR Request

```go
logger.Log("ERROR", "Something went wrong")
```

1. INFO does not match and forwards.
2. DEBUG does not match and forwards.
3. ERROR prints the message and returns `true`.
4. That result returns through DEBUG and INFO to the client.

### Step 8: Handle an Unsupported Request

```go
if !logger.Log("WARNING", "Unknown log level") {
	fmt.Println("Unhandled log level: WARNING")
}
```

No handler supports WARNING, so the chain returns `false`.

The client explicitly reports the unsupported level instead of silently ignoring it.

This example routes exact levels. It does not implement severity thresholds, such as logging everything at ERROR level or above.

### Step 9: Keep the Chain Valid

Avoid connecting the final handler back to the first:

```go
errorLog.SetNext(info)
```

An unsupported request would then circulate indefinitely.

Configure an acyclic chain before use. If multiple goroutines use it, avoid changing links concurrently without synchronization.
