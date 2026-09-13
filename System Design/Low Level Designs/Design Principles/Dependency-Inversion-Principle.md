# Dependency Inversion Principle (DIP)

## 1. What Is the Dependency Inversion Principle?

The Dependency Inversion Principle states that:

1. High-level modules should not depend directly on low-level modules. Both should depend on abstractions.
2. Abstractions should not depend on implementation details. Implementation details should depend on abstractions.

In simple terms:

> Depend on interfaces that describe required behaviour, not directly on concrete implementations.

### What does this mean?

Imagine that a `UserService` needs to save a user:

- `UserService` contains the important business logic, so it is a **high-level module**.
- `MySQLDatabase` contains database-specific code, so it is a **low-level module**.
- A `UserRepository` interface describes what the business logic needs, so it is an **abstraction**.

Instead of making `UserService` depend directly on MySQL, we make it depend on the `UserRepository` interface. MySQL, MongoDB, or an in-memory repository can implement that interface.

## 2. Why Does DIP Matter?

Without DIP, business logic becomes tightly connected to infrastructure such as databases, payment gateways, and external APIs.

For example, if `UserService` directly creates a MySQL connection, changing to MongoDB requires changing `UserService`. Testing it may also require a real MySQL database.

With DIP, `UserService` only knows that user data can be saved. It does not need to know where or how it is saved.

### Simple analogy

Think of a wall socket:

- An appliance depends on the socket's standard contract.
- It does not depend on a particular power station.
- The electricity provider can change without changing the appliance.

The socket acts like an interface between the appliance and the electricity provider.

## 3. Important Terms

### High-level module

Contains business rules or application logic.

Examples: registering a user, placing an order, or processing a payment.

### Low-level module

Handles technical or infrastructure details.

Examples: MySQL, Redis, an email provider, or a payment SDK.

### Abstraction

Defines the behaviour required by the high-level module. In Go, this is commonly a small interface.

### Concrete implementation

Contains the actual code that performs the behaviour described by an abstraction.

### Dependency injection

Provides a dependency to a type from outside instead of creating it inside that type.

## 4. Without DIP

```go
type MySQLDatabase struct{}

func (db *MySQLDatabase) Save(name string) error {
	// MySQL-specific logic
	return nil
}

type UserService struct {
	db *MySQLDatabase
}

func NewUserService() *UserService {
	return &UserService{
		db: &MySQLDatabase{},
	}
}
```

### What is the problem?

1. `UserService` depends directly on `MySQLDatabase`.
2. `UserService` creates its own dependency.
3. Switching to MongoDB requires changing `UserService`.
4. Unit testing is difficult because we cannot easily replace MySQL with a fake implementation.

The business logic is tightly coupled to a low-level database detail.

## 5. With DIP

We can introduce an interface that describes only what `UserService` needs:

```go
type UserRepository interface {
	Save(name string) error
}

type UserService struct {
	repository UserRepository
}

func NewUserService(repository UserRepository) *UserService {
	return &UserService{repository: repository}
}
```

Now `UserService` depends on the `UserRepository` abstraction. Any type with a matching `Save` method can be injected.

## 6. When to Use DIP

Use DIP when:

- Business logic should remain independent of infrastructure.
- A dependency may have multiple implementations.
- External systems may change in the future.
- Code needs to be tested without real external services.
- A dependency has side effects, such as database writes, emails, or payments.

Common examples include:

- MySQL, PostgreSQL, or MongoDB repositories
- Redis or in-memory caching
- Stripe or Razorpay payment gateways
- SMTP or third-party email providers
- Local or cloud-based file storage

DIP is not required for every struct. Introduce an abstraction when it provides a real testing, separation, or replacement benefit.

## 7. How DIP Can Be Violated

DIP is commonly violated by:

- Creating infrastructure dependencies inside business logic
- Using concrete dependency types in high-level modules
- Mixing database or framework code with business rules
- Making business logic depend directly on an external SDK
- Designing large interfaces containing unrelated methods

Example of tight coupling:

```go
func (s *UserService) RegisterUser(name string) error {
	db := &MySQLDatabase{}
	return db.Save(name)
}
```

`RegisterUser` decides both the business action and the database implementation. These responsibilities should be separated.

## 8. How to Apply DIP in Go

### 1. Identify what the high-level module needs

Ask: “What behaviour does this service require?”

For `UserService`, the answer may be: “It needs to save a user.”

### 2. Define a small interface

```go
type UserRepository interface {
	Save(name string) error
}
```

### 3. Make the high-level module accept the interface

```go
type UserService struct {
	repository UserRepository
}
```

### 4. Inject the implementation through a constructor

```go
func NewUserService(repository UserRepository) *UserService {
	return &UserService{repository: repository}
}
```

### 5. Create concrete dependencies at the application entry point

The `main` function commonly creates and connects the application's dependencies. This is sometimes called the **composition root**.

## 9. Sample Implementation in Go

```go
package main

import "fmt"

// UserRepository describes the behaviour required by UserService.
type UserRepository interface {
	Save(name string) error
}

// MySQLRepository is a low-level implementation.
type MySQLRepository struct{}

func (repository *MySQLRepository) Save(name string) error {
	fmt.Println("Saving user to MySQL:", name)
	return nil
}

// MongoRepository is another low-level implementation.
type MongoRepository struct{}

func (repository *MongoRepository) Save(name string) error {
	fmt.Println("Saving user to MongoDB:", name)
	return nil
}

// UserService contains high-level business logic.
type UserService struct {
	repository UserRepository
}

// NewUserService receives its dependency from outside.
func NewUserService(repository UserRepository) *UserService {
	return &UserService{repository: repository}
}

func (service *UserService) RegisterUser(name string) error {
	fmt.Println("Registering user:", name)
	return service.repository.Save(name)
}

func main() {
	mysqlRepository := &MySQLRepository{}
	mysqlUserService := NewUserService(mysqlRepository)

	if err := mysqlUserService.RegisterUser("Alice"); err != nil {
		fmt.Println("failed to register Alice:", err)
	}

	mongoRepository := &MongoRepository{}
	mongoUserService := NewUserService(mongoRepository)

	if err := mongoUserService.RegisterUser("Bob"); err != nil {
		fmt.Println("failed to register Bob:", err)
	}
}
```

## 10. Step-by-Step Code Explanation

### Step 1: Define the contract

```go
type UserRepository interface {
	Save(name string) error
}
```

The interface says that a user repository must provide a `Save` method. It describes the required behaviour without specifying a database.

### Step 2: Implement the contract using MySQL

```go
func (repository *MySQLRepository) Save(name string) error
```

Because `MySQLRepository` has the required method, it automatically satisfies `UserRepository`. Go does not require an explicit `implements` keyword.

### Step 3: Provide another implementation

`MongoRepository` has the same method signature, so it also satisfies `UserRepository`.

### Step 4: Depend on the abstraction

```go
type UserService struct {
	repository UserRepository
}
```

`UserService` knows only about the interface. It does not know whether MySQL or MongoDB is being used.

### Step 5: Inject the dependency

```go
func NewUserService(repository UserRepository) *UserService
```

The constructor receives the repository from outside. This is **constructor injection**.

### Step 6: Use the dependency

`RegisterUser` performs its business operation and calls `repository.Save(name)`. The injected implementation handles the actual storage.

### Step 7: Swap implementations

In `main`, we can inject either `MySQLRepository` or `MongoRepository` without modifying `UserService`.

## 11. Unit Testing Benefit

We can test `UserService` using a fake repository instead of a real database:

```go
package main

import "testing"

type FakeUserRepository struct {
	savedName string
}

func (repository *FakeUserRepository) Save(name string) error {
	repository.savedName = name
	return nil
}

func TestRegisterUser(t *testing.T) {
	fakeRepository := &FakeUserRepository{}
	service := NewUserService(fakeRepository)

	err := service.RegisterUser("Alice")
	if err != nil {
		t.Fatalf("expected no error, got %v", err)
	}

	if fakeRepository.savedName != "Alice" {
		t.Fatalf("expected Alice to be saved, got %q", fakeRepository.savedName)
	}
}
```

### How the test works

1. `FakeUserRepository` records the received name instead of accessing a database.
2. It satisfies the same `UserRepository` interface.
3. The fake is injected into `UserService`.
4. The test calls `RegisterUser`.
5. It verifies that the service asked the repository to save `Alice`.

The test is fast and does not require MySQL or MongoDB.

## 12. Pros

- **Flexibility:** Dependencies can be replaced easily.
- **Testability:** Real databases and external services can be replaced with fakes.
- **Maintainability:** Infrastructure changes have less effect on business logic.
- **Separation of concerns:** Business rules and technical details stay separate.
- **Clean architecture:** Dependency direction points toward business rules and abstractions.

## 13. Cons

- Introduces additional interfaces and constructors.
- Can make a small program unnecessarily complex.
- Requires developers to understand abstractions and dependency injection.
- Poorly designed interfaces can create more maintenance work.

## 14. Common Mistakes

### Creating an interface for every struct

Not every concrete type needs an interface. Create one when the consuming code needs an abstraction for separation, testing, or multiple implementations.

### Creating large interfaces

Prefer small interfaces based on the consumer's needs.

```go
type UserRepository interface {
	Save(name string) error
}
```

This is usually better than forcing `UserService` to depend on many unrelated database operations.

### Defining interfaces around implementation details

An interface should describe what the business logic requires, not copy every method provided by a database library.

### Confusing DIP with dependency injection

They are related, but they are not the same concept.

## 15. DIP vs Dependency Injection

| Concept | Meaning |
| --- | --- |
| Dependency Inversion Principle | A design principle that makes high-level and low-level modules depend on abstractions |
| Dependency injection | A technique for supplying dependencies from outside a type |

Constructor injection helps us apply DIP, but merely injecting a concrete type does not automatically follow DIP.

For example:

```go
func NewUserService(database *MySQLDatabase) *UserService
```

The dependency is injected, but the service still depends on a concrete MySQL type. Using a suitable interface removes that coupling.

## 16. Key Takeaways

- High-level business logic should not depend directly on infrastructure details.
- Define small interfaces based on the behaviour the consuming code needs.
- Make concrete infrastructure types implement those interfaces.
- Supply dependencies from outside, commonly through constructors.
- In Go, interfaces are satisfied implicitly.
- Apply DIP where it improves separation, replaceability, or testing—not automatically everywhere.

> DIP keeps business rules stable while allowing implementation details to change.