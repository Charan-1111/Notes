# Coupling and Cohesion Principles

Cohesion and coupling describe how responsibilities and dependencies are organized in a software system.

They help us answer two questions:

1. **Cohesion:** Does a module contain responsibilities that belong together?
2. **Coupling:** How strongly does one module depend on another?

> **Good software design aims for high cohesion and low coupling.**

---

## 1. Simple Real-World Analogy

Consider a restaurant:

- The **chef** is responsible for preparing food.
- The **cashier** is responsible for accepting orders and payments.
- The **waiter** is responsible for serving customers.

Each person has one focused responsibility. This is **high cohesion**.

The cashier communicates with the chef using an order ticket. The cashier does not need to know:

- Which knife the chef uses
- How the stove works
- How the ingredients are stored

This is **low coupling** because they communicate through a simple contract: the order ticket.

If the cashier had to control the chef's stove and cooking tools directly, they would be tightly connected. A change in the kitchen could then affect the cashier. This would be **high coupling**.

---

## 2. Cohesion

Cohesion measures how closely related the responsibilities inside a module, function, package, or class are.

### High Cohesion

A module has high cohesion when it focuses on one well-defined responsibility.

Examples:

- `UserRepository` handles user database operations.
- `EmailService` sends emails.
- `PaymentService` processes payments.
- `AuthMiddleware` verifies authentication.

```go
type EmailService struct{}

func (s EmailService) SendWelcomeEmail(email string) error {
    // Send a welcome email.
    return nil
}
```

This service has high cohesion because its responsibility is related to email operations.

### Low Cohesion

A module has low cohesion when it handles several unrelated responsibilities.

```go
type UserManager struct {
    db *sql.DB
}

func (m UserManager) ProcessUser(id int) {
    // Fetch user from the database.
    // Validate the user.
    // Send an email.
    // Generate an invoice.
    // Write application logs.
    // Construct an HTTP response.
}
```

This module has low cohesion because database access, email, invoicing, logging, and HTTP responses are different responsibilities.

### Simple Rule

> Things that change for the same reason should stay together.  
> Things that change for different reasons should be separated.

---

## 3. Coupling

Coupling measures how strongly one module depends on another module.

### Low Coupling

Modules have low coupling when they communicate through small, clear contracts and do not depend on each other's internal implementation.

For example:

```go
type UserRepository interface {
    GetUserByID(ctx context.Context, id int) (User, error)
}
```

A service using this interface does not need to know whether the user is fetched from:

- PostgreSQL
- MySQL
- Redis
- An external API
- An in-memory test implementation

### High Coupling

Modules have high coupling when one module directly depends on the internal details or concrete implementation of another.

```go
type UserService struct {
    db *sql.DB
}
```

Here, `UserService` directly depends on `sql.DB`.

Because of this:

- The service knows that SQL is being used.
- Database queries may become mixed with business logic.
- Unit testing requires a database or SQL mocking.
- Changing the storage implementation affects the service.

---

## 4. Cohesion vs Coupling

Cohesion and coupling measure different parts of the design.

| Principle | What it measures | Desired result |
|---|---|---|
| Cohesion | How closely related responsibilities inside a module are | High |
| Coupling | How strongly different modules depend on each other | Low |

A module can be cohesive but still tightly coupled.

For example, a `PaymentService` may contain only payment logic, giving it high cohesion. However, if it directly depends on a specific payment provider's SDK throughout the code, it can still have high coupling.

---

## 5. When to Apply These Principles

Cohesion and coupling should be considered when:

- Designing packages, services, or microservices
- Implementing business logic
- Separating database code from service code
- Following the Single Responsibility Principle
- Refactoring god classes or large modules
- Improving unit testability
- Reducing the ripple effect of changes
- Replacing one implementation with another

These principles apply to systems of every size—not only microservices.

---

## 6. Signs of Poor Design

### Signs of Low Cohesion

A module may have low cohesion when:

- It performs several unrelated tasks.
- It has many unrelated reasons to change.
- Its name is vague, such as `Manager`, `Helper`, or `Common`.
- It mixes database, business, transport, and logging logic.
- It becomes difficult to describe its responsibility in one sentence.

### Signs of High Coupling

Modules may be highly coupled when:

- One module directly depends on another module's internal details.
- Concrete dependencies are created inside business logic.
- Global variables are used for shared dependencies.
- A small change requires modifications in many modules.
- Code contains long dependency chains such as:

```go
order.User.Account.Settings.Language
```

- Unit testing requires real infrastructure.
- Replacing a database or external service requires changing business logic.

---

## 7. How to Improve the Design

### Increase Cohesion

- Give each module a clear responsibility.
- Split unrelated responsibilities into focused components.
- Keep business logic separate from database and HTTP logic.
- Use names that clearly describe the module's purpose.

### Reduce Coupling

- Define clear contracts using interfaces.
- Inject dependencies instead of creating them inside services.
- Hide infrastructure details behind repositories or clients.
- Avoid unnecessary global variables.
- Keep interfaces small and focused.
- Depend on required behaviour instead of concrete implementations.

---

## 8. Advantages

High cohesion and low coupling provide several benefits:

- Responsibilities are easier to understand.
- Changes have a smaller impact on other modules.
- Components are easier to test.
- Implementations can be replaced more easily.
- Modules can be reused.
- Developers can work on different modules independently.
- Large applications become easier to maintain.
- Components are easier to extract into separate services when necessary.

---

## 9. Trade-offs and Common Mistakes

These principles should not be applied blindly.

### Too Many Small Modules

Separating every function into a different package can make the project difficult to navigate.

### Unnecessary Interfaces

An interface should represent a useful boundary or replaceable behaviour.

Creating an interface for every struct adds boilerplate without always improving the design.

### Too Many Abstraction Layers

Code becomes harder to understand when a simple operation passes through many unnecessary layers.

```text
Handler → Manager → Service → Processor → Executor → Repository
```

Create an abstraction when it provides a real benefit, such as:

- Testability
- Multiple implementations
- Separation between business logic and infrastructure
- Protection from external implementation details

> The goal is clear separation—not the maximum possible number of files and interfaces.

---

## 10. Poor Go Implementation

The following example has low cohesion and high coupling:

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "log"

    _ "github.com/lib/pq"
)

type UserService struct {
    db *sql.DB
}

func (s UserService) ProcessUser(ctx context.Context, id int) error {
    var name string
    var active bool

    err := s.db.QueryRowContext(
        ctx,
        "SELECT name, active FROM users WHERE id = $1",
        id,
    ).Scan(&name, &active)
    if err != nil {
        return fmt.Errorf("fetch user: %w", err)
    }

    if !active {
        return fmt.Errorf("user is inactive")
    }

    fmt.Println("Processing user:", name)
    return nil
}

func main() {
    db, err := sql.Open(
        "postgres",
        "user=postgres dbname=test sslmode=disable",
    )
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    service := UserService{db: db}

    if err := service.ProcessUser(context.Background(), 1); err != nil {
        log.Fatal(err)
    }
}
```

### Step-by-Step Explanation

1. `UserService` directly depends on `*sql.DB`.
2. It contains the SQL query used to fetch the user.
3. It applies the business rule that the user must be active.
4. It also prints the result.

Therefore, it handles three different concerns:

- Database access
- Business logic
- Output

It also cannot be unit-tested easily without involving SQL-related infrastructure.

---

## 11. Improved Go Implementation

```go
package main

import (
    "context"
    "database/sql"
    "errors"
    "fmt"
    "log"

    _ "github.com/lib/pq"
)

var ErrInactiveUser = errors.New("user is inactive")

type User struct {
    ID     int
    Name   string
    Active bool
}

// UserRepository defines the behaviour required by UserService.
type UserRepository interface {
    GetByID(ctx context.Context, id int) (User, error)
}

// PostgresUserRepository handles PostgreSQL operations.
type PostgresUserRepository struct {
    db *sql.DB
}

func NewPostgresUserRepository(db *sql.DB) *PostgresUserRepository {
    return &PostgresUserRepository{db: db}
}

func (r *PostgresUserRepository) GetByID(
    ctx context.Context,
    id int,
) (User, error) {
    var user User

    err := r.db.QueryRowContext(
        ctx,
        `SELECT id, name, active
         FROM users
         WHERE id = $1`,
        id,
    ).Scan(&user.ID, &user.Name, &user.Active)

    if err != nil {
        return User{}, fmt.Errorf("get user by ID: %w", err)
    }

    return user, nil
}

// UserService handles user-related business logic.
type UserService struct {
    repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
    return &UserService{repo: repo}
}

func (s *UserService) ProcessUser(
    ctx context.Context,
    id int,
) (User, error) {
    user, err := s.repo.GetByID(ctx, id)
    if err != nil {
        return User{}, err
    }

    if !user.Active {
        return User{}, ErrInactiveUser
    }

    return user, nil
}

func main() {
    db, err := sql.Open(
        "postgres",
        "user=postgres dbname=test sslmode=disable",
    )
    if err != nil {
        log.Fatal(err)
    }
    defer db.Close()

    repo := NewPostgresUserRepository(db)
    service := NewUserService(repo)

    user, err := service.ProcessUser(context.Background(), 1)
    if err != nil {
        log.Fatal(err)
    }

    fmt.Println("Processing user:", user.Name)
}
```

### Step-by-Step Explanation

#### Step 1: Create the Domain Model

```go
type User struct {
    ID     int
    Name   string
    Active bool
}
```

The `User` struct represents the data used by the application.

It does not contain SQL or HTTP-specific details.

#### Step 2: Define the Repository Contract

```go
type UserRepository interface {
    GetByID(ctx context.Context, id int) (User, error)
}
```

The interface describes what `UserService` needs.

The service needs the ability to fetch a user, but it does not need to know how that user is fetched.

#### Step 3: Implement the PostgreSQL Repository

```go
type PostgresUserRepository struct {
    db *sql.DB
}
```

`PostgresUserRepository` is responsible only for PostgreSQL operations.

Its `GetByID` method:

1. Executes the query.
2. Scans the result into a `User`.
3. Returns the user or an error.

This gives the repository high cohesion.

#### Step 4: Inject the Repository into the Service

```go
type UserService struct {
    repo UserRepository
}
```

`UserService` depends on the interface instead of `*sql.DB`.

This is dependency injection: the required dependency is provided from outside the service.

#### Step 5: Keep Business Logic in the Service

```go
if !user.Active {
    return User{}, ErrInactiveUser
}
```

The rule that only active users can be processed belongs to the business layer.

The service does not execute SQL or print output.

#### Step 6: Construct Dependencies in `main`

```go
repo := NewPostgresUserRepository(db)
service := NewUserService(repo)
```

The `main` function connects the concrete implementations.

This is sometimes called the **composition root** because it is where the application's dependencies are assembled.

#### Step 7: Keep Output Outside the Service

```go
fmt.Println("Processing user:", user.Name)
```

Printing is handled by the calling layer.

In a web application, this layer would usually be an HTTP handler that converts the result into an HTTP response.

---

## 12. Testing Benefit

Because `UserService` depends on an interface, it can be tested without PostgreSQL.

```go
type FakeUserRepository struct {
    User User
    Err  error
}

func (f FakeUserRepository) GetByID(
    ctx context.Context,
    id int,
) (User, error) {
    return f.User, f.Err
}
```

The fake repository can be used in a unit test:

```go
func TestProcessUserReturnsActiveUser(t *testing.T) {
    repo := FakeUserRepository{
        User: User{
            ID:     1,
            Name:   "Charan",
            Active: true,
        },
    }

    service := NewUserService(repo)

    user, err := service.ProcessUser(context.Background(), 1)
    if err != nil {
        t.Fatalf("expected no error, got %v", err)
    }

    if user.Name != "Charan" {
        t.Fatalf("expected Charan, got %s", user.Name)
    }
}
```

### How the Test Works

1. `FakeUserRepository` satisfies the `UserRepository` interface.
2. It returns predefined data without connecting to PostgreSQL.
3. The fake repository is injected into `UserService`.
4. The test checks only the service's business behaviour.
5. The test is fast and independent of external infrastructure.

This is one of the main practical benefits of low coupling.

---

## 13. Practical Checklist

Before finalizing a module, ask:

- Does this module have one clear responsibility?
- Can I describe its responsibility in one sentence?
- Does it have several unrelated reasons to change?
- Is business logic mixed with SQL or HTTP code?
- Does the module depend on behaviour or concrete implementation details?
- Can I replace its dependency during testing?
- Does a small change affect many unrelated modules?
- Is an interface creating a useful boundary?
- Am I adding unnecessary layers or packages?

---

## 14. Summary

### Cohesion

Cohesion describes how closely related the responsibilities inside a module are.

- **High cohesion:** The module has one focused responsibility.
- **Low cohesion:** The module contains unrelated responsibilities.

### Coupling

Coupling describes how strongly different modules depend on one another.

- **Low coupling:** Modules communicate through clear contracts.
- **High coupling:** Modules depend on concrete implementations or internal details.

The desired design is:

> **High cohesion inside modules and low coupling between modules.**

In practical Go applications:

- Keep database logic in repositories.
- Keep business rules in services.
- Keep HTTP logic in handlers.
- Use small interfaces at meaningful boundaries.
- Inject dependencies from outside.
- Avoid adding abstractions that do not solve a real problem.
