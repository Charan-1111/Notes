# Builder Design Pattern in Go

The **Builder Pattern** is a creational design pattern used to construct a complex object step-by-step.

Instead of passing many arguments to a single function, the client configures the object through readable builder methods and calls `Build()` when finished.

For example, this is difficult to understand:

```go
house := NewHouse("concrete", "brick", "tile", true, false, 3)
```

With a builder, the purpose of every value is clear:

```go
house, err := NewHouseBuilder().
    SetFoundation("concrete").
    SetWalls("brick").
    SetRoof("tile").
    SetGarage(true).
    SetFloors(3).
    Build()
```

Think of it like ordering a custom meal: you choose the base, size, toppings, and extras one step at a time. The kitchen produces the final meal only after your configuration is complete.

## When to Use

Use the Builder Pattern when:

- An object has many optional fields.
- Passing many arguments makes object creation difficult to read.
- The object must be constructed in multiple steps.
- You need to validate the complete configuration before creating the object.
- You want to create different variations of the same product.

Typical backend examples include:

- Configuring an HTTP client
- Building a database query
- Creating a complex API request
- Configuring a server or application
- Constructing test data

> Go does not support constructor overloading. Functions such as `NewServer` or `NewClient` are ordinary factory functions written by convention.

## When Not to Use

Avoid the Builder Pattern when:

- The object contains only a few fields.
- All fields are required and a normal constructor is already clear.
- A struct literal communicates the configuration clearly.
- The additional builder type and methods provide no useful validation or readability.

For a simple object, this is enough:

```go
user := User{Name: "Charan", Age: 25}
```

Adding a builder here would introduce unnecessary code.

## Components

### 1. Product

The final complex object that we want to create.

```go
type House struct {
    Foundation string
    Walls      string
    Roof       string
}
```

### 2. Builder

Defines the steps used to configure and construct the product.

In Go, the builder can be:

- A concrete struct when there is only one building strategy
- An interface when multiple builders must follow the same construction process

### 3. Concrete Builder

Implements the construction steps and stores the object being assembled.

### 4. Director (Optional)

Uses a builder to execute construction steps in a predefined order. Many Go implementations do not need a director because the client can call builder methods directly.

### 5. Client

Configures the builder and requests the final product by calling `Build()`.

## How the Builder Pattern Works

1. The client creates a builder.
2. Builder methods configure the object one field at a time.
3. Each method returns the builder, allowing method chaining.
4. The client calls `Build()` after configuration is complete.
5. `Build()` validates the configuration and returns the final product.

The builder separates **how an object is constructed** from **the object that is eventually used**.

## Implementation in Go

```go
package main

import (
    "errors"
    "fmt"
)

// Product: the final object returned to the client.
type House struct {
    Foundation string
    Walls      string
    Roof       string
    Garage     bool
    Floors     int
}

// Concrete builder: stores the values while the house is being configured.
type HouseBuilder struct {
    foundation string
    walls      string
    roof       string
    garage     bool
    floors     int
}

// NewHouseBuilder creates a builder with sensible defaults.
func NewHouseBuilder() *HouseBuilder {
    return &HouseBuilder{
        floors: 1,
    }
}

func (b *HouseBuilder) SetFoundation(foundation string) *HouseBuilder {
    b.foundation = foundation
    return b
}

func (b *HouseBuilder) SetWalls(walls string) *HouseBuilder {
    b.walls = walls
    return b
}

func (b *HouseBuilder) SetRoof(roof string) *HouseBuilder {
    b.roof = roof
    return b
}

func (b *HouseBuilder) SetGarage(hasGarage bool) *HouseBuilder {
    b.garage = hasGarage
    return b
}

func (b *HouseBuilder) SetFloors(floors int) *HouseBuilder {
    b.floors = floors
    return b
}

// Build validates the configuration and creates the final House.
func (b *HouseBuilder) Build() (House, error) {
    if b.foundation == "" {
        return House{}, errors.New("foundation is required")
    }

    if b.walls == "" {
        return House{}, errors.New("walls are required")
    }

    if b.roof == "" {
        return House{}, errors.New("roof is required")
    }

    if b.floors < 1 {
        return House{}, errors.New("house must have at least one floor")
    }

    return House{
        Foundation: b.foundation,
        Walls:      b.walls,
        Roof:       b.roof,
        Garage:     b.garage,
        Floors:     b.floors,
    }, nil
}

func main() {
    house, err := NewHouseBuilder().
        SetFoundation("concrete").
        SetWalls("brick").
        SetRoof("tile").
        SetGarage(true).
        SetFloors(2).
        Build()

    if err != nil {
        fmt.Println("could not build house:", err)
        return
    }

    fmt.Printf("Built house: %+v\n", house)
}
```

## Code Explanation

### Step 1: Define the Product

`House` is the final product. It contains required fields such as `Foundation`, `Walls`, and `Roof`, together with optional configuration such as `Garage` and `Floors`.

### Step 2: Define the Builder

`HouseBuilder` temporarily stores the configuration. Its fields are unexported, so code outside the package cannot modify them directly.

The example does not define a builder interface because there is only one type of house builder. An interface would be useful only if multiple construction strategies needed to be interchangeable.

### Step 3: Provide Defaults

`NewHouseBuilder()` creates the builder and sets the default number of floors to `1`.

Defaults reduce the number of methods the client must call for common cases.

### Step 4: Configure the Object

Each setter updates one builder field and returns `*HouseBuilder`:

```go
func (b *HouseBuilder) SetRoof(roof string) *HouseBuilder {
    b.roof = roof
    return b
}
```

Returning the same builder enables method chaining:

```go
builder.SetWalls("brick").SetRoof("tile")
```

### Step 5: Validate and Build

`Build()` checks that the required fields are present and that the floor count is valid. If validation fails, it returns an error instead of producing an invalid `House`.

If the configuration is valid, `Build()` copies the values into a new `House` and returns it.

### Step 6: Use the Builder

The client chooses the desired configuration and calls `Build()`. Because `Build()` can fail, the client handles the returned error before using the house.

## Director Example (Optional)

A director contains predefined construction steps for common product variations.

```go
func BuildStandardHouse() (House, error) {
    return NewHouseBuilder().
        SetFoundation("concrete").
        SetWalls("brick").
        SetRoof("tile").
        SetFloors(1).
        Build()
}

func BuildLuxuryHouse() (House, error) {
    return NewHouseBuilder().
        SetFoundation("reinforced concrete").
        SetWalls("stone").
        SetRoof("slate").
        SetGarage(true).
        SetFloors(3).
        Build()
}
```

Here, `BuildStandardHouse()` and `BuildLuxuryHouse()` act as directors. They reuse the same builder while defining different construction recipes.

Use a director when the same predefined configurations are created repeatedly. Otherwise, allowing the client to use the builder directly is simpler.

## Pros

- **Readable construction:** Method names clearly describe each configured value.
- **Step-by-step creation:** The object can be configured gradually.
- **Handles optional fields:** The client sets only the options it needs.
- **Centralized validation:** `Build()` prevents invalid products from being returned.
- **Useful defaults:** The builder can provide sensible initial values.
- **Easy variations:** The same builder can create multiple versions of a product.
- **Encapsulated construction:** The client does not need to understand the internal assembly logic.

## Cons

- **More boilerplate:** The product may require a builder struct and several methods.
- **Additional mutable state:** The builder changes as methods are called.
- **Unnecessary for simple objects:** A constructor or struct literal may be clearer.
- **Can increase complexity:** Overusing the pattern makes small codebases harder to maintain.

## Builder vs Constructor vs Functional Options

| Approach | Best suited for | Main characteristic |
| --- | --- | --- |
| Constructor | Small objects with required fields | Simple and direct |
| Builder | Complex, multi-step construction with validation | Configure first, then call `Build()` |
| Functional options | Go APIs with defaults and optional settings | Pass option functions to a constructor |

### Constructor

Use a constructor when the object has a small number of required values:

```go
func NewUser(name string, age int) User {
    return User{Name: name, Age: age}
}
```

### Functional Options

Functional options are a popular Go alternative when configuration is optional but a separate `Build()` step is unnecessary:

```go
type Server struct {
    port    int
    timeout int
}

type ServerOption func(*Server)

func WithPort(port int) ServerOption {
    return func(s *Server) {
        s.port = port
    }
}

func WithTimeout(timeout int) ServerOption {
    return func(s *Server) {
        s.timeout = timeout
    }
}

func NewServer(options ...ServerOption) *Server {
    server := &Server{
        port:    8080,
        timeout: 30,
    }

    for _, option := range options {
        option(server)
    }

    return server
}
```

Client code:

```go
server := NewServer(
    WithPort(9000),
    WithTimeout(60),
)
```

Choose a builder when construction has meaningful stages or requires final validation. Choose functional options when you mainly need defaults and optional configuration during a single constructor call.

## Key Takeaways

- The Builder Pattern constructs complex objects step-by-step.
- It improves readability when an object has many optional settings.
- Returning the builder from setter methods enables fluent method chaining.
- `Build()` is the right place for final validation.
- A builder interface and director are optional in Go; add them only when they solve a real problem.
- For simple objects, prefer a constructor or struct literal.
- For many Go APIs, functional options may be a lighter alternative.