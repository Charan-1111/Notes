# Liskov Substitution Principle (LSP)

## 1. What is LSP?

The **Liskov Substitution Principle (LSP)** states:

> Objects that follow the same contract should be replaceable with one another without breaking the expected behaviour of the application.

A common definition is:

> If `S` is a subtype of `T`, then an object of type `T` should be replaceable with an object of type `S` without changing the correctness of the program.

In simple terms:

**If two types promise the same behaviour, the caller should be able to use either of them without needing special handling.**

### Simple Example

Imagine we have a `Shape`:

```go
type Shape interface {
    Area() int
}
```

Both `Rectangle` and `Square` implement `Area()`.

```text
             Shape
               |
        +------+------+
        |             |
   Rectangle        Square
```

If some function accepts a `Shape`, it should work correctly with both:

```go
func PrintArea(s Shape) {
    fmt.Println(s.Area())
}
```

The caller should not need code such as:

```go
if shapeIsRectangle {
    // do something
} else if shapeIsSquare {
    // do something different
}
```

That ability to safely replace one implementation with another is the main idea behind LSP.

### LSP in Go

Go does not have traditional class inheritance like Java or C++.

Instead, LSP is most commonly relevant to **interfaces and their implementations**.

For example:

```go
type Storage interface {
    Save(data []byte) error
}
```

Different implementations could be:

```text
Storage
   |
   +-- LocalStorage
   |
   +-- S3Storage
   |
   +-- DatabaseStorage
```

Every implementation should respect the behaviour promised by `Storage`.

---

## 2. Why Do We Need LSP?

LSP helps us build systems where implementations can be replaced without surprising the caller.

Consider a backend service:

```text
UserService
     |
     v
UserRepository
     |
 +---+-------------+
 |                 |
PostgresRepo    MockRepo
```

`UserService` should work correctly regardless of whether it receives `PostgresRepo` or `MockRepo`.

If `PostgresRepo` follows the repository contract but `MockRepo` behaves completely differently, replacing one with the other may break the service.

LSP helps make implementations:

- Predictable
- Replaceable
- Easier to test
- Easier to extend

---

## 3. When to Use LSP

LSP becomes important when:

- Designing interfaces.
- Creating multiple implementations of the same interface.
- Using dependency injection.
- Designing repository or service abstractions.
- Implementing different storage providers.
- Implementing different cache providers.
- Implementing payment or notification providers.
- Building plugin-style architectures.
- We want polymorphism to work correctly.

### Backend Example

Suppose we have:

```go
type Cache interface {
    Get(key string) (string, error)
    Set(key, value string) error
}
```

We may have:

```text
Cache
 |
 +-- RedisCache
 |
 +-- MemoryCache
```

Our application should be able to switch between `RedisCache` and `MemoryCache` without changing the code that uses `Cache`.

---

## 4. How Can We Break LSP?

LSP is violated when an implementation technically satisfies a type or interface but does not respect its expected behaviour.

### 4.1 Changing Expected Behaviour

Suppose a method promises to save data:

```go
Save(data []byte) error
```

An implementation that silently ignores the data and returns `nil` technically satisfies the interface, but violates the expected behaviour.

### 4.2 Strengthening Preconditions

A subtype should not unexpectedly demand **more** from the caller.

For example, if the contract accepts any non-empty username:

```text
CreateUser("charan")
```

one implementation should not suddenly require:

```text
username must contain at least 20 characters
```

unless that requirement is part of the original contract.

### 4.3 Weakening Postconditions

An implementation should not provide **less** than what the contract promises.

If:

```go
FindUser(id int) (User, error)
```

promises to return the requested user when it exists, an implementation should not return an empty `User` with `nil` error.

### 4.4 Breaking Invariants

An **invariant** is something that should always remain true.

For example, suppose a bank account guarantees:

```text
balance >= 0
```

An implementation that allows the balance to become negative would break that invariant.

### 4.5 Returning Unexpected Errors

Suppose:

```go
type Storage interface {
    Save(data []byte) error
}
```

If one implementation always returns:

```text
operation not supported
```

for valid input, we should question whether it really belongs behind this interface.

---

## 5. How to Apply LSP

### 1. Define Clear Contracts

Interfaces should represent clear behaviour.

```go
type Reader interface {
    Read() ([]byte, error)
}
```

Every implementation should behave according to what `Read()` promises.

### 2. Keep Interfaces Focused

Prefer:

```go
type Reader interface {
    Read() ([]byte, error)
}

type Writer interface {
    Write([]byte) error
}
```

instead of forcing every implementation to support:

```go
type Storage interface {
    Read() ([]byte, error)
    Write([]byte) error
    Delete() error
    Compress() error
    Encrypt() error
}
```

A smaller contract is usually easier to implement correctly.

### 3. Don't Depend on Implementation-Specific Behaviour

Avoid writing callers that know too much about concrete implementations.

Bad idea:

```go
if storage == "s3" {
    // special behaviour
}
```

Prefer letting the abstraction define the required behaviour.

### 4. Use Composition When Types Don't Share the Same Behaviour

Don't force two types into the same abstraction simply because they look similar.

### 5. Test the Contract

Run the same behavioural tests against different implementations.

```text
Repository Contract Tests
          |
     +----+----+
     |         |
 Postgres    Memory
```

If both pass the same contract tests, substitution becomes much safer.

---

## 6. Pros

- Ensures polymorphism works correctly.
- Makes implementations predictable.
- Makes implementations easier to replace.
- Reduces implementation-specific conditions.
- Makes testing easier.
- Encourages clean interface design.
- Helps extend systems without breaking existing callers.

---

## 7. Cons / Challenges

- Requires clear behavioural contracts.
- Poorly designed interfaces make LSP difficult to follow.
- Behavioural violations are not always caught by the compiler.
- May require redesigning abstractions.
- Can introduce unnecessary abstraction when applied to very small/simple codebases.

---

## 8. Example 1 — Rectangle and Square

The Rectangle/Square example is a classic way of understanding LSP.

> **Go note:** Go does not support classical inheritance. The first example intentionally models the classic inheritance problem conceptually. The second version shows a more Go-friendly design.

### Violating LSP

Consider the expectation for a rectangle:

```text
SetWidth(4)
SetHeight(5)

Area = 20
```

Now imagine treating `Square` as a special kind of `Rectangle`.

A square has an additional rule:

```text
width == height
```

Changing the width of a square therefore also changes its height.

Conceptually:

```go
type Rectangle struct {
    width  int
    height int
}

func (r *Rectangle) SetWidth(width int) {
    r.width = width
}

func (r *Rectangle) SetHeight(height int) {
    r.height = height
}

func (r Rectangle) Area() int {
    return r.width * r.height
}

type Square struct {
    side int
}

func (s *Square) SetWidth(width int) {
    s.side = width
}

func (s *Square) SetHeight(height int) {
    s.side = height
}

func (s Square) Area() int {
    return s.side * s.side
}
```

The important issue is not whether Go considers `Square` a subtype of `Rectangle`—it does not.

The problem is the **behavioural contract**.

If callers expect width and height to be independently configurable:

```text
SetWidth(4)
SetHeight(5)
Expected Area = 20
```

a square cannot satisfy that contract because:

```text
width == height
```

must always remain true.

### Step-by-Step

**Step 1 — Rectangle allows independent dimensions**

```go
r.SetWidth(4)
r.SetHeight(5)
```

Now:

```text
width  = 4
height = 5
area   = 20
```

**Step 2 — Square has a different invariant**

For a square:

```text
width == height
```

Setting:

```text
width = 4
height = 5
```

would mean it is no longer a square.

**Step 3 — The contracts conflict**

A rectangle-like contract says:

```text
width and height can change independently
```

A square says:

```text
width and height must always be equal
```

Therefore a square cannot safely replace something whose contract requires independent width and height.

That is the LSP problem.

---

### Applying LSP

Instead of forcing both types into a Rectangle abstraction, identify the behaviour they actually share:

```text
Both have an area.
```

Define that contract:

```go
package main

import "fmt"

type Shape interface {
    Area() int
}

type Rectangle struct {
    width  int
    height int
}

func (r Rectangle) Area() int {
    return r.width * r.height
}

type Square struct {
    side int
}

func (s Square) Area() int {
    return s.side * s.side
}

func PrintArea(shape Shape) {
    fmt.Println("Area:", shape.Area())
}

func main() {
    rectangle := Rectangle{
        width:  4,
        height: 5,
    }

    square := Square{
        side: 4,
    }

    PrintArea(rectangle)
    PrintArea(square)
}
```

### Step-by-Step

**Step 1 — Define the contract**

```go
type Shape interface {
    Area() int
}
```

Any shape only promises:

```text
I can calculate my area.
```

It does **not** promise anything about width or height.

**Step 2 — Rectangle implements the contract**

```go
func (r Rectangle) Area() int {
    return r.width * r.height
}
```

For:

```text
width = 4
height = 5
```

the result is:

```text
20
```

**Step 3 — Square implements the same contract**

```go
func (s Square) Area() int {
    return s.side * s.side
}
```

For:

```text
side = 4
```

the result is:

```text
16
```

**Step 4 — `PrintArea` depends only on the contract**

```go
func PrintArea(shape Shape)
```

It doesn't care whether it receives:

```text
Rectangle
Square
```

Both can safely substitute for `Shape`.

This satisfies LSP.

---

## 9. Example 2 — Backend Example in Go

The Rectangle/Square example explains the concept, but LSP becomes more useful when designing backend interfaces.

Consider file storage.

### Bad Design

Suppose we define:

```go
type Storage interface {
    Read(name string) ([]byte, error)
    Write(name string, data []byte) error
}
```

Now we create:

```text
Storage
   |
   +-- LocalStorage
   |
   +-- S3Storage
   |
   +-- ReadOnlyStorage
```

`ReadOnlyStorage` cannot actually support `Write()`.

Someone might implement it like this:

```go
type ReadOnlyStorage struct{}

func (r ReadOnlyStorage) Read(name string) ([]byte, error) {
    return []byte("data"), nil
}

func (r ReadOnlyStorage) Write(name string, data []byte) error {
    return errors.New("write operation not supported")
}
```

Technically, `ReadOnlyStorage` satisfies the `Storage` interface.

But consider this function:

```go
func SaveReport(storage Storage, data []byte) error {
    return storage.Write("report.txt", data)
}
```

If we pass:

```text
LocalStorage
```

it works.

If we pass:

```text
ReadOnlyStorage
```

it always fails because writing is fundamentally unsupported.

### Step-by-Step

**Step 1 — `Storage` promises both reading and writing**

```go
type Storage interface {
    Read(...)
    Write(...)
}
```

**Step 2 — `ReadOnlyStorage` claims to satisfy that contract**

Because it implements both methods, the Go compiler accepts it.

**Step 3 — But its behaviour doesn't match the contract**

```go
Write(...)
```

can never perform the operation.

**Step 4 — Substitution breaks**

A caller expecting writable `Storage` cannot safely receive `ReadOnlyStorage`.

The important lesson is:

> Satisfying a Go interface at compile time does not automatically mean that the implementation satisfies LSP.

---

### Better Design

Separate the capabilities:

```go
package main

import (
    "errors"
    "fmt"
)

type Reader interface {
    Read(name string) ([]byte, error)
}

type Writer interface {
    Write(name string, data []byte) error
}

type LocalStorage struct {
    files map[string][]byte
}

func NewLocalStorage() *LocalStorage {
    return &LocalStorage{
        files: make(map[string][]byte),
    }
}

func (s *LocalStorage) Read(name string) ([]byte, error) {
    data, exists := s.files[name]
    if !exists {
        return nil, errors.New("file not found")
    }

    return data, nil
}

func (s *LocalStorage) Write(name string, data []byte) error {
    s.files[name] = data
    return nil
}

type ReadOnlyStorage struct {
    files map[string][]byte
}

func (s *ReadOnlyStorage) Read(name string) ([]byte, error) {
    data, exists := s.files[name]
    if !exists {
        return nil, errors.New("file not found")
    }

    return data, nil
}

func SaveReport(writer Writer, data []byte) error {
    return writer.Write("report.txt", data)
}

func main() {
    storage := NewLocalStorage()

    err := SaveReport(storage, []byte("Monthly Report"))
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    data, err := storage.Read("report.txt")
    if err != nil {
        fmt.Println("Error:", err)
        return
    }

    fmt.Println(string(data))
}
```

### Step-by-Step

**Step 1 — Separate reading and writing**

```go
type Reader interface {
    Read(name string) ([]byte, error)
}

type Writer interface {
    Write(name string, data []byte) error
}
```

Each interface represents one clear capability.

**Step 2 — `LocalStorage` supports both**

It implements:

```text
Read()
Write()
```

Therefore it can be used as either:

```text
Reader
Writer
```

**Step 3 — `ReadOnlyStorage` implements only what it supports**

It implements:

```text
Read()
```

It does not pretend that it can write.

**Step 4 — `SaveReport` asks for exactly what it needs**

```go
func SaveReport(writer Writer, data []byte) error
```

The function requires a type that genuinely supports writing.

A read-only implementation cannot accidentally be passed to it.

This design makes invalid substitution harder and gives us a cleaner behavioural contract.

---

## 10. LSP in Real Backend Systems

LSP appears frequently in backend development.

### Repository Implementations

```go
type UserRepository interface {
    FindByID(id int) (User, error)
}
```

Implementations:

```text
PostgresUserRepository
MemoryUserRepository
MockUserRepository
```

All should follow the same behavioural expectations.

### Cache Implementations

```go
type Cache interface {
    Get(key string) (string, error)
    Set(key, value string) error
}
```

Implementations:

```text
RedisCache
MemoryCache
```

The application should not need special logic depending on which cache implementation is being used.

### Notification Providers

```go
type Notifier interface {
    Send(message string) error
}
```

Implementations might include:

```text
EmailNotifier
SMSNotifier
PushNotifier
```

Each implementation should honour the meaning of `Send()`.

### Storage Providers

```go
type Storage interface {
    Upload(data []byte) error
}
```

Implementations:

```text
S3Storage
GCSStorage
LocalStorage
```

Business logic can depend on `Storage` instead of knowing which provider is underneath it.

---

## 11. How LSP Relates to Other SOLID Principles

### LSP + OCP

The **Open-Closed Principle** says we should be able to add new behaviour without modifying stable code.

Suppose:

```go
type Notifier interface {
    Send(message string) error
}
```

We already have:

```text
EmailNotifier
SMSNotifier
```

Later we add:

```text
PushNotifier
```

OCP allows us to extend the system with `PushNotifier`.

LSP ensures `PushNotifier` behaves according to the existing `Notifier` contract.

So:

```text
OCP -> allows new implementations

LSP -> ensures those implementations are safe replacements
```

### LSP + ISP

The **Interface Segregation Principle (ISP)** encourages small, focused interfaces.

Instead of:

```go
type Storage interface {
    Read()
    Write()
    Delete()
    Compress()
    Encrypt()
}
```

we may have:

```go
type Reader interface {
    Read()
}

type Writer interface {
    Write()
}
```

Smaller interfaces make it easier for implementations to genuinely support their contracts.

Therefore:

```text
Good interface design
        |
        v
Easier implementation
        |
        v
Easier LSP compliance
```

---

## 12. Quick Rules to Remember

1. **Subtypes/implementations should respect the behaviour promised by the abstraction.**

2. **Replacing one valid implementation with another should not require the caller to change its logic.**

3. **Implementing a Go interface is not enough — the behavioural contract matters too.**

4. **Don't make implementations support operations they fundamentally cannot perform.**

5. **Prefer small, focused interfaces.**

6. **Don't strengthen preconditions unexpectedly.**

7. **Don't weaken postconditions promised by the contract.**

8. **Preserve important invariants.**

9. **If callers constantly check which concrete implementation they received, inspect the abstraction.**

10. **Use contract tests when multiple implementations should behave consistently.**

---

## 13. Final Mental Model

The easiest way to remember LSP is:

```text
Interface      = Promise

Implementation = Someone fulfilling the promise

LSP            = Every implementation must keep the promise
```

For example:

```go
type PaymentProcessor interface {
    Pay(amount float64) error
}
```

If we have:

```text
StripeProcessor
RazorpayProcessor
MockProcessor
```

then the business logic should be able to say:

```go
processor.Pay(100)
```

without caring which valid processor it received.

```text
               PaymentProcessor
                       |
           +-----------+-----------+
           |           |           |
        Stripe      Razorpay      Mock
           |           |           |
           +-----------+-----------+
                       |
              Same behavioural
                  contract
```

That is the core idea of the **Liskov Substitution Principle**:

> **If an implementation promises to follow an abstraction, callers should be able to trust that promise.**