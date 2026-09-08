# Single Responsibility Principle (SRP)

## 1. What is SRP?

The **Single Responsibility Principle (SRP)** states:

> A class, struct, module, or function should have **one responsibility** and therefore **one reason to change**.

In simple terms:

**One piece of code should focus on one job.**

For example, imagine a `UserService` that:

- validates user data,
- inserts users into PostgreSQL,
- sends emails,
- writes HTTP responses.

This component has several responsibilities. It may need to change when:

- business rules change,
- the database changes,
- email requirements change,
- the API response format changes.

That is a sign that the component is doing too much.

A better design separates those responsibilities:

```text
UserHandler     → HTTP responsibility
UserService     → business responsibility
UserRepository  → database responsibility
EmailService    → email responsibility
```

### What does "one reason to change" mean?

This is the most important part of SRP.

Consider:

```text
UserRepository
```

Its responsibility is storing and retrieving users.

It might contain several methods:

```go
Create()
FindByID()
Update()
Delete()
```

Having multiple methods **does not violate SRP**, because all of them belong to the same responsibility:

> User persistence / database operations.

But if we add:

```go
SendWelcomeEmail()
```

to `UserRepository`, we introduce another responsibility.

Now the repository can change because:

1. the database implementation changes, or
2. the email implementation changes.

That is what SRP tries to prevent.

---

## 2. Why SRP Matters

Consider a large function that:

```text
Validate Request
      ↓
Apply Business Rules
      ↓
Execute SQL
      ↓
Send Email
      ↓
Return HTTP Response
```

A small change to email handling may require modifying the same function that contains database and business logic.

This increases the chance of accidentally breaking unrelated functionality.

With SRP:

```text
Handler
   ↓
Service
   ↓
Repository
```

Changes are more isolated.

For example:

```text
Database query changes
        ↓
UserRepository changes
```

The HTTP handler usually does not need to change.

Similarly:

```text
Business rule changes
        ↓
UserService changes
```

The repository usually does not need to change.

---

## 3. When to Use SRP

SRP becomes especially useful when:

### 1. We have a God Object

A **God Object** is a struct or class that handles too many unrelated responsibilities.

Example:

```go
type UserManager struct {
    db *sql.DB
}
```

Imagine `UserManager` contains methods for:

```text
CreateUser()
ValidateUser()
SaveUser()
SendEmail()
GenerateJWT()
WriteLogs()
```

This component has too many reasons to change.

---

### 2. A function performs multiple unrelated operations

Example:

```go
func RegisterUser() {
    // validate request
    // execute SQL
    // apply business rules
    // send email
    // create HTTP response
}
```

The function is responsible for several different concerns.

---

### 3. Business logic and infrastructure logic are mixed

For example:

```text
Business Rules + SQL + HTTP + Email
```

inside the same component.

These responsibilities usually change for different reasons and should generally be separated.

---

### 4. We want code that is easier to test and maintain

Smaller, focused components are easier to understand and test independently.

---

## 4. How We Break SRP

Common ways SRP gets violated include:

### Adding unrelated responsibilities to one struct

For example:

```go
type UserService struct {
    db *sql.DB
}
```

and then making it responsible for:

```text
Database access
Business rules
Email
Logging
HTTP responses
```

---

### Mixing business logic and database logic

Example:

```go
func RegisterUser(name string) {
    // business validation

    // SQL query

    // business processing
}
```

Database access and business decisions are separate concerns.

---

### Mixing HTTP logic with business logic

For example:

```go
func RegisterUser(c *fiber.Ctx) error {
    // parse HTTP request
    // validate business rules
    // execute SQL
    // send email
    // return HTTP response
}
```

The handler has become responsible for almost the entire use case.

---

### Writing large functions with multiple behaviours

A large function is not automatically an SRP violation.

The real question is:

> Does this function have multiple unrelated reasons to change?

If yes, it probably needs to be separated.

---

## 5. How to Apply SRP

A useful process is:

### Step 1: Identify responsibilities

Ask:

```text
What jobs is this component doing?
```

For example:

```text
Parse HTTP request
Validate business rules
Store user
Send email
```

These are different responsibilities.

---

### Step 2: Ask why the code might change

Suppose one function handles:

```text
SQL
Business Rules
HTTP
```

Ask:

```text
Could SQL change independently?       → Yes
Could business rules change?          → Yes
Could API response format change?     → Yes
```

There are multiple reasons to change.

---

### Step 3: Separate responsibilities

We could create:

```text
UserHandler
UserService
UserRepository
```

Each component has a clear purpose.

---

### Step 4: Introduce interfaces where useful

Interfaces can help separate the business layer from implementation details.

For example:

```go
type UserRepository interface {
    Create(name string) error
}
```

Now `UserService` depends on the capability it needs rather than directly depending on PostgreSQL.

Do not create interfaces only because SRP exists. Use them when abstraction, substitution, or testing benefits from them.

---

### Step 5: Refactor large functions into focused functions

Instead of:

```go
func RegisterUser() {
    // everything
}
```

we might have:

```go
func RegisterUser() {}
func validateUser() {}
func sendWelcomeEmail() {}
```

But splitting functions is useful only when the extracted functions represent meaningful responsibilities or steps.

---

## 6. Example: Violating SRP in Go

Consider this implementation:

```go
package main

import (
    "database/sql"
    "fmt"
)

type UserService struct {
    db *sql.DB
}

func (s UserService) RegisterUser(name string) error {
    if name == "" {
        return fmt.Errorf("name cannot be empty")
    }

    _, err := s.db.Exec(
        "INSERT INTO users(name) VALUES ($1)",
        name,
    )
    if err != nil {
        return err
    }

    fmt.Println("User registered:", name)

    return nil
}
```

### Step-by-step explanation

#### Step 1: `UserService` receives the database

```go
type UserService struct {
    db *sql.DB
}
```

The service directly knows about SQL/database infrastructure.

#### Step 2: It performs business validation

```go
if name == "" {
    return fmt.Errorf("name cannot be empty")
}
```

Checking whether a user can be registered is business logic.

#### Step 3: It performs database operations

```go
_, err := s.db.Exec(
    "INSERT INTO users(name) VALUES ($1)",
    name,
)
```

Now the same service also contains persistence logic.

#### Step 4: It handles output/logging

```go
fmt.Println("User registered:", name)
```

The same method is also producing output.

So `UserService` has multiple reasons to change:

```text
Business rule changes
        ↓
UserService changes

Database query changes
        ↓
UserService changes

Output/logging requirements change
        ↓
UserService changes
```

This is the problem SRP tries to avoid.

---

## 7. Example: Applying SRP in Go

We can separate the responsibilities into:

```text
HTTP Request
     ↓
UserHandler
     ↓
UserService
     ↓
UserRepository
     ↓
Database
```

### Repository

The repository handles user persistence.

```go
type UserRepository struct {
    db *sql.DB
}

func (r UserRepository) Create(name string) error {
    _, err := r.db.Exec(
        "INSERT INTO users(name) VALUES ($1)",
        name,
    )

    return err
}
```

### Step-by-step

#### Step 1

```go
type UserRepository struct {
    db *sql.DB
}
```

The repository receives the database connection.

#### Step 2

```go
func (r UserRepository) Create(name string) error
```

Its job is to persist the user.

#### Step 3

```go
_, err := r.db.Exec(...)
```

The SQL/database operation stays inside the repository.

#### Step 4

```go
return err
```

The repository reports whether persistence succeeded or failed.

It does **not** decide how the application should respond to that failure.

Its responsibility is:

```text
UserRepository → User persistence
```

---

### Repository Interface

The service does not necessarily need to know about the concrete database implementation.

```go
type UserRepositoryInterface interface {
    Create(name string) error
}
```

This describes what the service needs:

```text
"I need something capable of creating a user."
```

The service does not care whether the implementation uses:

```text
PostgreSQL
MySQL
In-memory storage
Mock repository
```

---

### Service

The service handles business logic.

```go
type UserService struct {
    repo UserRepositoryInterface
}

func (s UserService) RegisterUser(name string) error {
    if name == "" {
        return fmt.Errorf("name cannot be empty")
    }

    return s.repo.Create(name)
}
```

### Step-by-step

#### Step 1

```go
type UserService struct {
    repo UserRepositoryInterface
}
```

The service depends on a repository abstraction.

#### Step 2

```go
if name == "" {
    return fmt.Errorf("name cannot be empty")
}
```

The service performs the business validation.

#### Step 3

```go
return s.repo.Create(name)
```

The service asks the repository to store the user.

It does not execute SQL itself.

Its responsibility is:

```text
UserService → User registration business logic
```

---

### Handler

In a real backend, the HTTP layer can also be separated.

Using Go Fiber:

```go
type UserHandler struct {
    service *UserService
}

type RegisterUserRequest struct {
    Name string `json:"name"`
}

func (h UserHandler) RegisterUser(c *fiber.Ctx) error {
    var req RegisterUserRequest

    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(
            fiber.Map{"error": "invalid request"},
        )
    }

    if err := h.service.RegisterUser(req.Name); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(
            fiber.Map{"error": err.Error()},
        )
    }

    return c.Status(fiber.StatusCreated).JSON(
        fiber.Map{"message": "user registered successfully"},
    )
}
```

### Step-by-step

#### Step 1: Parse the HTTP request

```go
c.BodyParser(&req)
```

The handler converts the HTTP request body into Go data.

#### Step 2: Call the business layer

```go
h.service.RegisterUser(req.Name)
```

The handler does not implement registration rules itself.

It delegates them to `UserService`.

#### Step 3: Convert the result into an HTTP response

```go
return c.Status(...).JSON(...)
```

HTTP status codes and JSON responses belong to the HTTP layer.

The responsibility is:

```text
UserHandler → HTTP handling
```

---

### Complete Flow

When a request arrives:

```text
POST /users
     │
     ▼
UserHandler
     │
     │ Parse HTTP request
     ▼
UserService
     │
     │ Apply business rules
     ▼
UserRepository
     │
     │ Execute SQL
     ▼
PostgreSQL
```

Each layer has one main responsibility:

| Component | Responsibility |
|---|---|
| `UserHandler` | HTTP request/response handling |
| `UserService` | Business logic |
| `UserRepository` | Database persistence |

Now consider a change.

If the SQL query changes:

```text
UserRepository changes
```

If registration rules change:

```text
UserService changes
```

If the HTTP response changes:

```text
UserHandler changes
```

This is the main benefit of SRP: **changes are more isolated.**

---

## 8. SRP and Separation of Concerns

SRP and **Separation of Concerns (SoC)** are related, but they are not exactly the same thing.

### Separation of Concerns

SoC is a broader design idea:

> Different concerns of a system should be separated from each other.

For example:

```text
HTTP Layer
Business Layer
Database Layer
```

### Single Responsibility Principle

SRP focuses more specifically on individual software units:

```text
Does this struct/module/function have one clear responsibility?
```

A simple way to remember the relationship:

```text
Separation of Concerns
        ↓
Broad architectural/design principle

Single Responsibility Principle
        ↓
Focuses on keeping individual components responsible for one thing
```

---

## 9. Pros

### Easier to maintain

Changes are more isolated.

```text
Database change
     ↓
Repository
```

instead of modifying a giant service containing everything.

### Improves readability

When we see:

```go
UserRepository
```

we can reasonably expect database-related user operations.

When we see:

```go
UserService
```

we can expect user-related business logic.

### Easier to test

Business logic can be tested without requiring a real database when the dependency is abstracted appropriately.

For example:

```text
UserService
    ↓
Mock Repository
```

### Reduces side effects

Changing database code is less likely to accidentally affect HTTP handling or business logic.

### Easier collaboration

Different developers can work on separate components with clearer boundaries.

---

## 10. Cons / Things to Be Careful About

SRP is useful, but it can be overused.

### Too many tiny files or structs

We should not create a new struct for every tiny operation.

Bad over-engineering might look like:

```text
UserNameValidator
UserEmailValidator
UserAgeValidator
UserCreator
UserIDGenerator
```

when the application is small and these responsibilities naturally belong together.

### Extra boilerplate

Separating layers can introduce:

```text
interfaces
constructors
structs
dependency wiring
```

For small programs, this may be unnecessary.

### Unnecessary abstractions

SRP does **not** mean:

> Everything needs an interface.

Create abstractions when they provide a real benefit.

### Over-engineering

The goal is:

```text
Clear responsibilities
```

not:

```text
Maximum number of files
```

---

## 11. Important Things to Remember

### SRP does not mean "one method per struct"

This is perfectly reasonable:

```go
type UserRepository struct {
    db *sql.DB
}

func (r UserRepository) Create() {}
func (r UserRepository) FindByID() {}
func (r UserRepository) Update() {}
func (r UserRepository) Delete() {}
```

All these methods belong to:

```text
User persistence
```

So the repository can still have a single responsibility.

---

### SRP does not mean functions must always be small

A 50-line function can potentially have one responsibility.

A 10-line function can potentially have three unrelated responsibilities.

Do not judge SRP only by line count.

Ask:

> How many different reasons does this code have to change?

---

### Related operations can belong together

For example:

```text
CreateUser
UpdateUser
DeleteUser
FindUser
```

can all belong to a user repository because they are part of the same persistence responsibility.

---

### Split based on responsibility, not file size

Do not think:

```text
"This file has 200 lines, so it violates SRP."
```

Instead ask:

```text
"What responsibilities does this file have?"
```

If all 200 lines support one cohesive responsibility, SRP may still be satisfied.

---

## 12. Quick Summary

### Definition

> A component should have one responsibility and one reason to change.

### Main question to ask

```text
Why might this code need to change?
```

If there are several unrelated answers, the component may have too many responsibilities.

### Example

Instead of:

```text
UserService
 ├── HTTP handling
 ├── Business logic
 ├── SQL
 └── Email
```

prefer clear boundaries such as:

```text
UserHandler
     ↓
UserService
     ↓
UserRepository
```

where:

```text
UserHandler     → HTTP
UserService     → Business logic
UserRepository  → Database
```

### Remember

```text
SRP ≠ one method per struct
SRP ≠ tiny functions everywhere
SRP ≠ interfaces everywhere

SRP = one clear responsibility / one reason to change
```
