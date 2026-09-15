# YAGNI Principle in Go

## 1. What Is YAGNI?

**YAGNI** stands for **You Aren't Gonna Need It**.

It means:

> Do not build functionality until there is a real, current requirement for it.

Developers often think:

- “We may need another database later.”
- “Perhaps we should support five authentication methods.”
- “This service might eventually handle millions of requests.”

Planning is useful, but implementing every possible future requirement is usually not. Many of those requirements may change or never arrive.

Building code for an imagined requirement is called **speculative development**.

---

## 2. Why YAGNI Matters

Every feature has a cost beyond writing the initial code. It must also be:

- Understood by other developers
- Tested and reviewed
- Documented
- Maintained when dependencies change
- Debugged when something fails

Unused functionality makes a system more complicated without providing value to its users.

### Simple example

Suppose the current requirement is:

> Store users in PostgreSQL.

Implement PostgreSQL support.

Do not automatically create a database framework supporting PostgreSQL, MySQL, MongoDB, and Oracle merely because the application *might* use them someday.

---

## 3. Simple Real-World Example

Suppose a product story says:

> Send an email after a user registers.

A YAGNI-friendly implementation sends the registration email through the provider currently used by the application.

An over-engineered implementation might add:

- SMS and push-notification support
- Multiple email-provider adapters
- A rule engine for selecting a notification channel
- Configuration for providers that are not being used

These capabilities may sound useful, but they do not solve the current requirement. Add them when a confirmed requirement makes them necessary.

---

## 4. When to Apply YAGNI

Use YAGNI in the following situations:

### Designing APIs

Implement only the endpoints required by current use cases.

### Developing services

Avoid creating a new microservice for a feature that does not need independent scaling or deployment.

### Refactoring

Improve the problem that currently exists instead of generalizing the code for every possible variation.

### Adding configuration

Expose a configuration setting only when something genuinely needs to vary.

### Working iteratively

Deliver what the current user story requires, gather feedback, and implement the next requirement afterward.

---

## 5. Common Ways Developers Violate YAGNI

### Premature abstractions

Creating several interfaces, factories, and layers before there are multiple real implementations.

### Unnecessary configuration

Adding options such as selectable cache engines when the application has only one cache implementation.

### Supporting unused technologies

Building support for several databases or message brokers when the system uses only one.

### Unused API endpoints

Creating endpoints for hypothetical clients or operations that no current user needs.

### Premature scaling

Introducing distributed services, queues, or database sharding before measurements show that the simpler design is insufficient.

---

## 6. How to Apply YAGNI

Follow these steps:

1. Write down the current requirement clearly.
2. Implement the simplest maintainable solution that satisfies it.
3. Test the current behavior.
4. Observe real usage and collect feedback.
5. Extend or refactor the design when a new requirement actually appears.

Before adding something, ask:

- Which current requirement needs this?
- Who will use it now?
- What problem does it solve today?
- What is the cost of adding and maintaining it?
- Can it be added safely later?

If there is no concrete requirement or user, the feature probably does not belong in the codebase yet.

---

## 7. YAGNI in Backend Development

### API design

Suppose clients currently need only:

```text
POST /users
GET /users/{id}
```

Do not create update, delete, bulk-import, and export endpoints only because they might be useful later.

Every additional endpoint requires:

- Request validation
- Authorization
- Tests
- Documentation
- Error handling
- Security maintenance

Implement those endpoints when clients actually need them.

### Database design

Create tables and indexes for known requirements and access patterns.

Do not add columns, relationships, and indexes for reports that nobody has requested.

### Configuration

Hard-coding secrets is unsafe, so secrets should be configurable.

However, making every small constant configurable creates unnecessary operational complexity. Configuration should be used for values that genuinely differ across environments or deployments.

### Microservices

Do not split a small application into many microservices only because it might grow.

A well-structured monolith is often easier to:

- Develop
- Test
- Deploy
- Monitor
- Debug

Split it when clear service boundaries, team ownership, independent deployments, or scaling requirements justify the change.

### Performance

Basic performance practices are valuable, but complex caching, replication, or database sharding should be driven by measurements.

First identify the actual bottleneck. Then optimize it.

---

## 8. Go Example: Violating YAGNI

### Current requirement

The program needs to calculate only the area of a circle.

```go
package main

import (
	"fmt"
	"math"
)

type Shape interface {
	Area() float64
}

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}

// There is currently no requirement for a square.
type Square struct {
	Side float64
}

func (s Square) Area() float64 {
	return s.Side * s.Side
}

func printArea(shape Shape) {
	fmt.Printf("Area: %.2f\n", shape.Area())
}

func main() {
	circle := Circle{Radius: 5}
	printArea(circle)
}
```

### Step-by-step explanation

#### Step 1: Define an interface

```go
type Shape interface {
	Area() float64
}
```

The `Shape` interface represents anything that can calculate an area.

However, the current requirement needs only one shape: a circle.

#### Step 2: Define the circle

```go
type Circle struct {
	Radius float64
}
```

The `Circle` structure stores the radius of a circle.

#### Step 3: Implement the area calculation

```go
func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}
```

This method calculates the circle's area using:

```text
Area = π × radius × radius
```

#### Step 4: Add an unused square

```go
type Square struct {
	Side float64
}

func (s Square) Area() float64 {
	return s.Side * s.Side
}
```

The `Square` implementation was added even though the current application does not need it.

This is speculative development.

#### Step 5: Accept the interface

```go
func printArea(shape Shape) {
	fmt.Printf("Area: %.2f\n", shape.Area())
}
```

The function can print the area of any shape, although the application currently needs only circles.

#### Step 6: Use only the circle

```go
func main() {
	circle := Circle{Radius: 5}
	printArea(circle)
}
```

Although support for other shapes was created, `main` uses only a circle.

### What is wrong with this implementation?

The program works correctly, but it contains:

- An unused `Square` type
- An unnecessary extension mechanism
- An abstraction that does not solve a current problem
- More concepts for developers to understand and maintain

The `Shape` interface is not inherently bad.

It becomes useful when:

- The application genuinely supports multiple shapes
- Several shapes must be handled through the same function
- The interface creates a meaningful boundary around a dependency

The problem is creating the abstraction only for an imagined future requirement.

---

## 9. Go Example: Applying YAGNI

```go
package main

import (
	"fmt"
	"math"
)

func circleArea(radius float64) float64 {
	return math.Pi * radius * radius
}

func main() {
	area := circleArea(5)
	fmt.Printf("Area of the circle: %.2f\n", area)
}
```

### Step-by-step explanation

#### Step 1: Import the required packages

```go
import (
	"fmt"
	"math"
)
```

- `fmt` is used to print the result.
- `math` provides the accurate value of `Pi`.

#### Step 2: Create the required function

```go
func circleArea(radius float64) float64 {
	return math.Pi * radius * radius
}
```

The function:

1. Accepts the circle's radius.
2. Calculates the area using `π × radius × radius`.
3. Returns the calculated value.

#### Step 3: Call the function

```go
area := circleArea(5)
```

This calculates the area of a circle whose radius is `5`.

#### Step 4: Print the result

```go
fmt.Printf("Area of the circle: %.2f\n", area)
```

`%.2f` prints the result with two digits after the decimal point.

### Why is this implementation better?

This version directly satisfies the current requirement.

It does not contain:

- Unused shapes
- Unnecessary interfaces
- Speculative extension mechanisms

If the application later needs several shapes, we can refactor the code using the actual requirements.

At that point, those requirements will help us design the correct abstraction.

---

## 10. YAGNI Does Not Mean Poor Design

YAGNI does **not** mean:

- Writing careless or unreadable code
- Skipping validation and error handling
- Ignoring security
- Avoiding useful tests
- Refusing all architectural planning
- Choosing code that is intentionally difficult to change

For example, validating a request body is not speculative.

Invalid input is a real and predictable condition.

Similarly, adding authentication to a private endpoint is not speculative. Authentication is part of the endpoint's current security requirement.

YAGNI is about avoiding **unrequired functionality**, not avoiding engineering quality.

It may also be reasonable to make a small design decision today when:

- A future requirement is strongly supported by evidence
- The requirement is expected very soon
- Changing the design later would be extremely expensive

The decision should be based on evidence and cost—not merely on “maybe someday.”

---

## 11. Relationship with KISS and DRY

### YAGNI — You Aren't Gonna Need It

Do not build functionality that is not currently required.

### KISS — Keep It Simple

Choose the simplest solution that clearly and reliably solves the current problem.

### DRY — Don't Repeat Yourself

Avoid duplicating the same knowledge or business rule in multiple places.

These principles should be balanced.

For example, do not create a complicated generic framework just to remove two small pieces of similar code.

A little duplication can be easier to understand until a stable and repeated pattern becomes clear.

---

## 12. Advantages and Trade-offs

### Advantages

- Keeps the codebase lean and focused
- Reduces development and review time
- Lowers testing and maintenance effort
- Makes the code easier to understand
- Allows real feedback to guide future designs
- Reduces unused features and possible bugs

### Trade-offs

- A new requirement may require refactoring later
- A narrowly designed solution may become difficult to extend if written carelessly
- Ignoring a highly likely and expensive future constraint may create additional work

These trade-offs are reasons to use good judgment, not reasons to abandon YAGNI.

Write clean and changeable code for today's requirement without implementing tomorrow's imagined features.

---

## 13. Practical Decision Checklist

Before adding a feature or abstraction, check:

- [ ] Is it required by a current user story or technical requirement?
- [ ] Is there a real user or component that needs it now?
- [ ] Do we have evidence rather than a hypothetical possibility?
- [ ] Is the simpler implementation maintainable and safe?
- [ ] Have measurements shown that this optimization is necessary?
- [ ] Would delaying this work create a serious and expensive problem?
- [ ] Can we add or refactor it later when we understand the requirement better?

If most answers point toward “not needed now,” postpone the implementation and record the idea for later.

---

## 14. Key Takeaways

- Build what is required today.
- Avoid writing code only for “maybe someday.”
- Keep the current solution simple, clean, tested, and maintainable.
- Use real requirements and measurements to guide abstractions and optimizations.
- Refactor when a new need becomes real.
- Balance YAGNI with KISS, DRY, security, and sound engineering judgment.

> The goal of YAGNI is not to prevent future change. It is to avoid paying today for a future that may never happen.