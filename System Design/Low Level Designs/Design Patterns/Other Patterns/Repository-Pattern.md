# Repository Design Pattern

The **Repository pattern** is a data access pattern that gives business logic a consistent way to load and store domain objects.

A repository exposes operations such as `FindByID()` and `Save()`. The caller works with a collection of users, while the repository handles the details of looking them up in a map, querying a database, or calling an external API.

**Example:** A user service needs to rename a user. The service checks whether the new name is valid. The repository loads the user and saves the updated record. The service never needs to write SQL.

## When to Use

- When you want to keep SQL, ORM calls, and storage details out of business logic.
- When you want to test service rules with an in-memory implementation.
- When several callers need the same lookup or persistence behavior.
- When domain-specific queries give callers a clearer API, such as `FindPendingApprovals()`.

You don’t need multiple databases to justify a repository. A useful boundary between business rules and persistence can be enough. For a small CRUD application, an extra layer may add little if it only forwards calls to the ORM.

## Components

| Component | Responsibility | Go example |
|---|---|---|
| **Domain Model** | Represents the business data the application works with. | `User` |
| **Repository Interface** | Defines the persistence operations callers rely on. | `UserRepository` |
| **Concrete Repository** | Implements the contract using a specific data source. | `InMemoryUserRepository` |
| **Client / Service Layer** | Applies business rules and calls the repository. | `UserService` |

## Working

1. A caller asks `UserService` to rename a user.
2. The service validates the new name and calls `FindByID()`.
3. The repository returns the user or an error. If the user is missing, the error matches `ErrUserNotFound`.
4. The service updates the user and calls `Save()`.
5. The repository persists the change using its own storage mechanism.

A database repository would translate these operations into queries and map rows to `User` values. It can receive a shared connection pool during setup and reuse it across calls.

## Pros

- **Separates responsibilities:** The service owns business rules; the repository owns persistence.
- **Makes service tests easier:** A test can use an in-memory repository without starting a database.
- **Keeps behavior consistent:** Callers can share the same rules for missing records, ordering, or filters.
- **Hides storage changes behind a contract:** Query optimizations can stay inside the repository when they preserve the behavior callers rely on.

## Cons

- **Adds abstraction:** A repository that only forwards ORM calls may be unnecessary.
- **Can make complex reads awkward:** Reports and dashboards often need joins, aggregates, or specialized result types. A dedicated query method or read component can express those needs better than a generic CRUD interface.
- **Doesn’t make database swaps automatic:** A replacement must preserve the behavior the service relies on, including errors, ordering, and transaction guarantees.
- **Requires database tests too:** An in-memory implementation helps test service logic, but it cannot verify SQL, constraints, or transaction behavior.

## Sample Implementation in Golang

This example stores users in a map. `Save()` inserts a new user or replaces the existing user with the same ID. `FindAll()` returns users in ascending ID order.

```go
package main

import (
	"errors"
	"fmt"
	"log"
	"sort"
	"strings"
)

// Domain model.
type User struct {
	ID   int
	Name string
}

var ErrUserNotFound = errors.New("user not found")

// Repository contract.
type UserRepository interface {
	FindByID(id int) (User, error)
	Save(user User) error
	FindAll() ([]User, error)
}

// Concrete repository: in-memory storage.
type InMemoryUserRepository struct {
	users map[int]User
}

func NewInMemoryUserRepository() *InMemoryUserRepository {
	return &InMemoryUserRepository{
		users: make(map[int]User),
	}
}

func (r *InMemoryUserRepository) FindByID(id int) (User, error) {
	user, exists := r.users[id]
	if !exists {
		return User{}, fmt.Errorf("%w: id %d", ErrUserNotFound, id)
	}
	return user, nil
}

func (r *InMemoryUserRepository) Save(user User) error {
	r.users[user.ID] = user
	return nil
}

func (r *InMemoryUserRepository) FindAll() ([]User, error) {
	users := make([]User, 0, len(r.users))
	for _, user := range r.users {
		users = append(users, user)
	}

	// Map iteration order is unspecified.
	sort.Slice(users, func(i, j int) bool {
		return users[i].ID < users[j].ID
	})

	return users, nil
}

// Service: applies business rules through the repository contract.
type UserService struct {
	repo UserRepository
}

func (s UserService) RenameUser(id int, name string) error {
	name = strings.TrimSpace(name)
	if name == "" {
		return errors.New("name must not be blank")
	}

	user, err := s.repo.FindByID(id)
	if err != nil {
		return fmt.Errorf("rename user: %w", err)
	}

	user.Name = name
	if err := s.repo.Save(user); err != nil {
		return fmt.Errorf("save renamed user: %w", err)
	}

	return nil
}

func main() {
	repo := NewInMemoryUserRepository()
	service := UserService{repo: repo}

	for _, user := range []User{
		{ID: 2, Name: "Bob"},
		{ID: 1, Name: "Alice"},
	} {
		if err := repo.Save(user); err != nil {
			log.Fatal(err)
		}
	}

	if err := service.RenameUser(1, " Alicia "); err != nil {
		log.Fatal(err)
	}

	err := service.RenameUser(99, "Charlie")
	switch {
	case errors.Is(err, ErrUserNotFound):
		fmt.Println("Could not rename user 99: user not found")
	case err != nil:
		log.Fatal(err)
	}

	users, err := repo.FindAll()
	if err != nil {
		log.Fatal(err)
	}
	for _, user := range users {
		fmt.Printf("User %d: %s\n", user.ID, user.Name)
	}
}
```

**Expected output:**

```text
Could not rename user 99: user not found
User 1: Alicia
User 2: Bob
```

### Step-by-Step Explanation

1. **Define the contract.** `UserRepository` describes what callers can do with users. `UserService` depends on that interface, so it can work with an in-memory repository in a test and a database repository in the application.

2. **Implement storage in the repository.** `InMemoryUserRepository` uses a map keyed by user ID. `Save()` replaces the record if the ID already exists. `FindByID()` returns a `User` value, so changing its name in the service does not change the stored record until `Save()` runs.

3. **Keep the business rule in the service.** `RenameUser()` trims whitespace and rejects blank names before accessing storage. It then loads, changes, and saves the user.

4. **Distinguish missing records from storage failures.** The repository wraps `ErrUserNotFound` with the ID, and the service adds context using `%w`. `errors.Is()` still recognizes the original error. A database implementation could map its no-rows result to the same error while returning connection failures separately.

5. **Make ordering explicit.** `FindAll()` sorts by ID because map iteration has no guaranteed order. A SQL implementation preserving that contract would use `ORDER BY id`.

This example is designed for one goroutine. In a web server, protect a shared in-memory map from concurrent access. If a read followed by a write must be atomic, protect the whole operation or use a dedicated update method; locking each repository call separately does not make the sequence atomic.

Further reading: [Repository pattern example](https://github.com/Charan-1111/low-level-design/blob/main/design-patterns/examples/Repository-pattern.md)
