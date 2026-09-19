# Separation of Concerns Principle

## What Is Separation of Concerns?

**Separation of Concerns (SoC)** is a software design principle that divides an application into distinct sections or modules.

Each section focuses on one particular responsibility or **concern**.

A concern can be:

- Handling an HTTP request
- Validating user input
- Applying business rules
- Communicating with a database
- Logging application events
- Formatting an HTTP response

Instead of placing all these responsibilities inside one function, we separate them into different layers or packages.

### Simple example

Consider a restaurant:

- The waiter accepts the order.
- The chef prepares the food.
- The cashier handles the payment.

Each person has a separate responsibility.

Similarly, in a backend application:

- The **handler** handles HTTP communication.
- The **service** handles business logic.
- The **repository** handles database communication.

---

## Why Separation of Concerns Matters

Imagine an HTTP handler that:

1. Reads the request.
2. Validates the input.
3. Executes an SQL query.
4. Applies business rules.
5. Logs the operation.
6. Creates the HTTP response.

This function has too many responsibilities.

If the database, validation rule, or response format changes, the same function must be modified repeatedly. It also becomes difficult to test and understand.

With Separation of Concerns, each responsibility is placed in the appropriate layer.

```text
Client
  ↓
Handler
  ↓
Service
  ↓
Repository
  ↓
Database
```

---

## When to Use Separation of Concerns

Separation of Concerns is useful:

- When building layered applications:

  ```text
  Handler → Service → Repository
  ```

- When the application contains different responsibilities such as validation, logging, business logic, and database access.
- When the code needs to be modular, maintainable, and testable.
- When multiple developers work on different parts of the application.
- When the application is expected to grow over time.

Even small applications can use SoC, but they may not need many layers or abstractions.

---

## How the SoC Principle Is Broken

The principle is commonly broken by:

- Writing database queries directly inside HTTP handlers.
- Mixing business rules with database operations.
- Formatting HTTP responses inside the service layer.
- Adding logging, validation, and database access directly inside core business logic.
- Creating a large class or function that handles everything.
- Making one package responsible for unrelated functionality.

Such code may work initially, but it becomes difficult to change, test, and maintain as the application grows.

---

## How to Apply Separation of Concerns

A common Go backend can be divided into the following layers.

### 1. Handler or controller layer

Responsible for HTTP-related work:

- Reading request parameters
- Decoding JSON
- Performing basic input validation
- Calling the service
- Converting errors into HTTP responses
- Encoding the response

It should not contain SQL queries or important business rules.

### 2. Service layer

Responsible for business logic:

- Applying application rules
- Coordinating multiple repositories
- Making business decisions
- Returning domain-related results and errors

It should not know about HTTP status codes or SQL queries.

### 3. Repository layer

Responsible for data access:

- Executing database queries
- Reading and saving records
- Converting database rows into Go values
- Returning data-access errors

It should not format HTTP responses or apply presentation logic.

### 4. Supporting packages

Other concerns can also be separated into packages:

- `config` for application configuration
- `middleware` for authentication and request logging
- `model` for domain structures
- `validator` for reusable validation
- `logger` for structured logging

Do not create a separate package for every small function. Separate responsibilities only when the separation makes the code easier to understand or change.

---

## Example: Breaking Separation of Concerns

In the following example, everything is mixed inside the HTTP handler:

```go
package main

import (
	"database/sql"
	"fmt"
	"net/http"

	_ "github.com/lib/pq"
)

func getGreetingHandler(w http.ResponseWriter, r *http.Request) {
	// Database initialization inside the handler
	db, err := sql.Open(
		"postgres",
		"user=postgres dbname=test sslmode=disable",
	)
	if err != nil {
		http.Error(w, "database initialization failed", http.StatusInternalServerError)
		return
	}
	defer db.Close()

	// Database access inside the handler
	var name string
	err = db.QueryRow(
		"SELECT name FROM users WHERE id = $1",
		1,
	).Scan(&name)

	if err != nil {
		http.Error(w, "user not found", http.StatusNotFound)
		return
	}

	// Business and presentation logic inside the handler
	greeting := "Welcome, " + name
	fmt.Fprintln(w, greeting)
}
```

### What is wrong with this code?

The handler is responsible for:

1. Creating the database connection.
2. Executing the SQL query.
3. Handling database errors.
4. Creating the greeting.
5. Writing the HTTP response.

These are different concerns placed inside one function.

This causes several problems:

- The business logic cannot be tested easily without a database.
- Changing the database affects the HTTP handler.
- The function becomes larger as more requirements are added.
- The same database logic may be duplicated in other handlers.

---

## Example: Applying Separation of Concerns

The following implementation separates the application into repository, service, and handler layers.

```go
package main

import (
	"database/sql"
	"errors"
	"fmt"
	"log"
	"net/http"
	"strconv"

	_ "github.com/lib/pq"
)

// Domain errors
var ErrUserNotFound = errors.New("user not found")

// Repository contract required by the service.
type UserRepository interface {
	GetUserName(id int) (string, error)
}

// PostgreSQL repository implementation.
type PostgresUserRepository struct {
	db *sql.DB
}

func NewPostgresUserRepository(db *sql.DB) *PostgresUserRepository {
	return &PostgresUserRepository{db: db}
}

func (r *PostgresUserRepository) GetUserName(id int) (string, error) {
	var name string

	err := r.db.QueryRow(
		"SELECT name FROM users WHERE id = $1",
		id,
	).Scan(&name)

	if errors.Is(err, sql.ErrNoRows) {
		return "", ErrUserNotFound
	}

	if err != nil {
		return "", fmt.Errorf("get user name: %w", err)
	}

	return name, nil
}

// Service contains the business logic.
type UserService struct {
	repo UserRepository
}

func NewUserService(repo UserRepository) *UserService {
	return &UserService{repo: repo}
}

func (s *UserService) GetGreeting(id int) (string, error) {
	name, err := s.repo.GetUserName(id)
	if err != nil {
		return "", err
	}

	return "Welcome, " + name, nil
}

// Handler deals only with HTTP-related work.
type UserHandler struct {
	service *UserService
}

func NewUserHandler(service *UserService) *UserHandler {
	return &UserHandler{service: service}
}

func (h *UserHandler) GetGreeting(w http.ResponseWriter, r *http.Request) {
	id, err := strconv.Atoi(r.URL.Query().Get("id"))
	if err != nil || id <= 0 {
		http.Error(w, "invalid user id", http.StatusBadRequest)
		return
	}

	greeting, err := h.service.GetGreeting(id)
	if errors.Is(err, ErrUserNotFound) {
		http.Error(w, "user not found", http.StatusNotFound)
		return
	}

	if err != nil {
		http.Error(w, "internal server error", http.StatusInternalServerError)
		return
	}

	fmt.Fprintln(w, greeting)
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

	if err := db.Ping(); err != nil {
		log.Fatal(err)
	}

	repository := NewPostgresUserRepository(db)
	service := NewUserService(repository)
	handler := NewUserHandler(service)

	http.HandleFunc("/greeting", handler.GetGreeting)

	log.Println("server running on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## Step-by-Step Explanation

### Step 1: Define the repository interface

```go
type UserRepository interface {
	GetUserName(id int) (string, error)
}
```

The service needs something that can retrieve a user's name.

It does not need to know whether the data comes from:

- PostgreSQL
- MySQL
- An external API
- An in-memory map

The interface describes only the capability required by the service.

---

### Step 2: Implement the repository

```go
type PostgresUserRepository struct {
	db *sql.DB
}
```

`PostgresUserRepository` contains the PostgreSQL-specific dependency.

Its `GetUserName` method executes the SQL query:

```go
func (r *PostgresUserRepository) GetUserName(id int) (string, error) {
	var name string

	err := r.db.QueryRow(
		"SELECT name FROM users WHERE id = $1",
		id,
	).Scan(&name)

	if errors.Is(err, sql.ErrNoRows) {
		return "", ErrUserNotFound
	}

	if err != nil {
		return "", fmt.Errorf("get user name: %w", err)
	}

	return name, nil
}
```

This method is concerned only with data access:

1. Execute the query.
2. Read the returned value.
3. Convert database errors into meaningful application errors.
4. Return the result.

---

### Step 3: Implement the service

```go
type UserService struct {
	repo UserRepository
}
```

The service depends on the `UserRepository` interface instead of directly depending on PostgreSQL.

```go
func (s *UserService) GetGreeting(id int) (string, error) {
	name, err := s.repo.GetUserName(id)
	if err != nil {
		return "", err
	}

	return "Welcome, " + name, nil
}
```

The service:

1. Requests the user's name from the repository.
2. Handles the returned error.
3. Applies the business rule for creating the greeting.
4. Returns the result.

The service does not know anything about SQL or HTTP.

---

### Step 4: Implement the HTTP handler

```go
type UserHandler struct {
	service *UserService
}
```

The handler depends on the service.

Its `GetGreeting` method:

1. Reads the `id` query parameter.
2. Validates the value.
3. Calls the service.
4. Maps application errors to HTTP status codes.
5. Writes the HTTP response.

For example:

```text
GET /greeting?id=1
```

A successful response could be:

```text
Welcome, Charan
```

The handler does not execute SQL or create the greeting itself.

---

### Step 5: Connect the layers

The `main` function creates the dependencies:

```go
repository := NewPostgresUserRepository(db)
service := NewUserService(repository)
handler := NewUserHandler(service)
```

The dependency flow is:

```text
UserHandler
    ↓ depends on
UserService
    ↓ depends on
UserRepository interface
    ↑ implemented by
PostgresUserRepository
```

This process is often called **dependency injection**.

The dependencies are created outside the layers and passed through constructors.

---

## Request Flow

When the client sends:

```text
GET /greeting?id=1
```

The request follows this sequence:

1. The handler reads and validates `id`.
2. The handler calls `UserService.GetGreeting(1)`.
3. The service calls `UserRepository.GetUserName(1)`.
4. The PostgreSQL repository executes the query.
5. The repository returns the user's name.
6. The service creates the greeting.
7. The handler writes the HTTP response.

Each layer handles one category of responsibility.

---

## Testing Benefits

Because the service depends on an interface, its business logic can be tested without starting PostgreSQL.

```go
type FakeUserRepository struct {
	name string
	err  error
}

func (f FakeUserRepository) GetUserName(id int) (string, error) {
	return f.name, f.err
}
```

Example service test:

```go
func TestGetGreeting(t *testing.T) {
	repo := FakeUserRepository{name: "Charan"}
	service := NewUserService(repo)

	greeting, err := service.GetGreeting(1)
	if err != nil {
		t.Fatal(err)
	}

	expected := "Welcome, Charan"

	if greeting != expected {
		t.Fatalf("expected %q, got %q", expected, greeting)
	}
}
```

### Why is this easier to test?

The test does not require:

- A running PostgreSQL server
- Database credentials
- Database tables
- Test records

The fake repository returns controlled data, allowing the test to focus only on the service's business logic.

---

## Advantages

### Improved maintainability

A change in one concern has less impact on other concerns.

For example, modifying a PostgreSQL query usually requires changes only in the repository.

### Better testability

Business logic can be tested independently by replacing dependencies with fakes or mocks.

### Improved reusability

The same service can be used by:

- An HTTP handler
- A gRPC handler
- A command-line application
- A background job

### Easier team collaboration

Different developers can work on handlers, services, and repositories with fewer conflicts.

### Easier debugging

When a problem occurs, the responsible layer is easier to identify.

---

## Disadvantages

### Additional boilerplate

Separating layers introduces more:

- Types
- Constructors
- Interfaces
- Files
- Packages

### Risk of over-engineering

A small program may not need a handler, service, repository, and interface for every operation.

### More navigation

Developers may need to move through multiple files to understand one request flow.

The goal is not to create the maximum number of layers. The goal is to keep unrelated responsibilities separate.

---

## Practical Guidelines

- Keep HTTP-specific code inside handlers.
- Keep business rules inside services.
- Keep SQL and database operations inside repositories.
- Define interfaces where they provide useful decoupling or testability.
- Do not create an interface for every struct automatically.
- Keep interfaces small and focused on what the consumer needs.
- Avoid placing every helper function in a generic `utils` package.
- Separate code based on responsibilities, not only file size.
- Do not add layers that have no meaningful responsibility.
- Start simple and introduce separation as the application grows.

---

## Separation of Concerns vs Single Responsibility Principle

These principles are related but have slightly different focuses.

- **Separation of Concerns** divides the overall system into sections responsible for different concerns.
- **Single Responsibility Principle** says a module should have one primary reason to change.

For example:

- Separating HTTP handling, business logic, and database access follows SoC.
- Ensuring `UserService` changes only when user-related business rules change follows SRP.

They are often applied together.

---

## Summary

Separation of Concerns means organizing software so that each part focuses on a specific responsibility.

In a typical Go backend:

```text
Handler    → HTTP communication
Service    → Business logic
Repository → Database access
```

The main benefits are:

- Easier maintenance
- Better testability
- Improved reusability
- Simpler collaboration
- Reduced coupling between modules

However, the principle should be applied according to the application's complexity. A small application does not need unnecessary layers, but mixing unrelated responsibilities will make a growing application harder to maintain.