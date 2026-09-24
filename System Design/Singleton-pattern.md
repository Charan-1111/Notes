# Singleton Design Pattern

The Singleton is a **creational design pattern**: it controls how an object is created. It provides one shared instance and a common way to access it.

**Example:** Your Go API has user, payment, and order handlers. Each needs application settings. Instead of loading settings separately in every handler, they can access the same configuration instance.

“One instance” means **one per running application process**. If you run three replicas of your API, each replica has its own singleton. A singleton does not coordinate state across servers.

## Key Principles

### 1. Single Instance

The accessor creates an instance once and returns that same instance on later calls.

Think of one shared office noticeboard: everyone reads the same board instead of maintaining separate copies.

### 2. Global Access

A function such as `config.GetInstance()` gives different parts of the application access to the shared object.

This is convenient, but dependencies become less visible: a function may depend on configuration even though configuration is absent from its parameters.

### 3. Private Constructor

In class-based languages, a private constructor prevents callers from constructing the class directly.

Go has no classes or special constructor syntax. A function named `NewConfig` is just an ordinary function. Go uses **package visibility** instead:

- Names starting with an uppercase letter are exported.
- Names starting with a lowercase letter are unexported.

An unexported implementation type helps prevent other packages from constructing it directly by name. An accessor exposes the intended shared instance.

This is an API design convention, not an absolute ban on copies. Package code can still create values, and callers holding a concrete pointer can copy its value. Returning an interface can hide more implementation detail when necessary.

### 4. Lazy/Eager Initialization

| Approach | When initialization happens | Example |
|---|---|---|
| Lazy | On the first request for the instance | Load optional settings when first needed |
| Eager | During package initialization or explicit application startup | Load required settings before accepting HTTP requests |

Lazy initialization avoids work if the object is never used, but the first caller pays the initialization cost. Eager initialization makes startup failures easier to detect before serving traffic.

### 5. Thread Safety

In Go, multiple goroutines might request the instance simultaneously.

A simple “if the instance is nil, create it” check is unsafe without synchronization: two goroutines could both observe nil and create separate objects.

`sync.Once` makes initialization happen once, and concurrent callers wait for that initialization to finish.

**Safe initialization does not make later mutations safe.** A shared cache still needs synchronization, such as a mutex, around concurrent reads and writes. Configuration that never changes after initialization avoids that particular problem.

## When to Use

Consider a singleton when there is a clear reason for one shared instance within the process and global access is appropriate.

| Example | What is shared | Important detail |
|---|---|---|
| Database connection pool | A pool reused by handlers | One pool can contain many database connections |
| Logger | A configured logging instance | Request-specific information can still vary |
| Configuration loader | Settings loaded once | Reloadable settings need a separate update strategy |
| Cache manager | In-memory cached data | Each API replica has its own cache |

For example, creating a new database pool for every HTTP request wastes resources. Creating a pool once and reusing it is sensible.

However, **a shared resource does not require a Singleton implementation**. You can create the pool or configuration in `main()` and pass it to handlers. This is dependency injection: components receive their dependencies instead of finding them globally. It often makes testing and lifecycle management easier.

## Pros

- **Reuses one instance:** Callers of the accessor receive the same object.
- **Avoids repeated initialization:** Useful when constructing the object is expensive.
- **Centralizes access:** The application has a consistent entry point for the resource.
- **Supports lazy initialization:** Construction can wait until the first use.

A singleton guarantees reuse through its access path. It does not guarantee that every part of the application uses it, or that all global variables are the same.

## Cons

- **Global state:** A change made by one component may affect another unexpectedly.
- **Tight coupling:** Code calling a concrete global accessor is harder to substitute with a fake implementation.
- **Difficult test isolation:** Settings initialized in one test may remain in place for later tests within the same process.
- **Concurrency work remains:** Shared mutable fields need protection even after initialization.
- **Responsibility concerns:** Combining business behavior, instance creation, and global access can conflict with the Single Responsibility Principle.
- **Lifecycle limitations:** Reloading configuration, retrying initialization, or shutting down a shared resource needs explicit design.

For example, if one test initializes a singleton with production-like settings, another test cannot simply initialize the same `sync.Once` again with different settings.

## Sample Implementation in Go

We will create a configuration instance that is initialized lazily and read through methods. The example uses fixed values so the singleton behavior stays easy to follow.

### 1. Configuration package

Save this as `config/config.go` in a Go module named `singleton-demo`:

```go
package config

import (
    "fmt"
    "sync"
)

type settings struct {
    appName string
    port    int
}

func (s *settings) AppName() string {
    return s.appName
}

func (s *settings) Port() int {
    return s.port
}

var (
    instance *settings
    once     sync.Once
)

func GetInstance() *settings {
    once.Do(func() {
        fmt.Println("Initializing configuration")
        instance = &settings{
            appName: "Orders API",
            port:    8080,
        }
    })

    return instance
}
```

**Step by step:**

1. **`settings` holds the configuration.** Its lowercase name prevents another package from constructing it by naming `config.settings`.
2. **The fields are unexported.** Other packages cannot directly assign `appName` or `port`. The exported methods provide read access.
3. **`instance` stores a pointer.** It starts as nil and later points to the shared configuration.
4. **`once` controls initialization.** Its zero value is ready to use; no constructor is necessary.
5. **`once.Do(...)` runs the initialization function once.** Later calls skip that function.
6. **`return instance` returns the shared pointer.** Initialization has completed before a normal call returns.

The exported function can return an unexported type. Callers can use type inference with `:=` and call its exported methods.

### 2. Use the same instance twice

Save this as `main.go` in the module root:

```go
package main

import (
    "fmt"

    "singleton-demo/config"
)

func main() {
    first := config.GetInstance()
    second := config.GetInstance()

    fmt.Println("Same instance:", first == second)
    fmt.Println("Application:", first.AppName())
    fmt.Println("Port:", second.Port())
}
```

Expected output:

```text
Initializing configuration
Same instance: true
Application: Orders API
Port: 8080
```

**Step by step:**

1. The first call creates the configuration and prints the initialization message.
2. The second call returns the existing pointer without initializing again.
3. Comparing the pointers produces `true`: both refer to the same object.
4. Both variables can read settings through the exported methods.

To run this example, initialize the module with `go mod init singleton-demo`, then run `go run .`. If you already have a module, use its module path in the import instead.

### 3. What happens with concurrent calls?

Suppose three HTTP handlers call `GetInstance()` simultaneously:

| Caller | Behavior |
|---|---|
| Goroutine that starts initialization | Executes the function passed to `once.Do` |
| Other goroutines while initialization is running | Wait inside `once.Do` |
| All callers after initialization completes | Receive the same initialized pointer |

Which goroutine initializes the object is not predictable, but the initialization function runs only once.

### 4. Important limitations of `sync.Once`

- It protects initialization, not later reads and writes to mutable fields.
- It has no reset method and must not be copied after first use.
- It does not automatically retry a failed initialization. An initialization function that records an error and returns is still considered finished.
- If the initialization function panics, that call panics and the `Once` still considers the function finished. Later calls do not rerun it.

Consequently, avoid treating `sync.Once` as a retry mechanism for a database or network dependency. For essential resources, explicit startup initialization with error handling is often easier to manage.

The original sample also had a spelling mismatch: `instacne` was declared but `instance` was used. Those must match for the code to compile. The standard Go type is written `sync.Once`.

Reference: [Official Go documentation for sync.Once](https://go.dev/pkg/sync/#Once).