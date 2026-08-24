# Packages and Modules in Go

When building a Go application, you need a way to:

- Organize related code.
- Reuse code in different files.
- Separate different responsibilities.
- Use third-party libraries.
- Manage library versions.

Go provides two important concepts for this:

1. **Packages** organize Go source code.
2. **Modules** organize and manage one or more packages.

A simple way to remember this is:

```text
Module
 ├── Package
 ├── Package
 └── Package
```

> A package is a collection of related Go files.  
> A module is a collection of related Go packages with a `go.mod` file.

---

## 1. What Is a Package?

A **package** is a collection of Go files that belong together and provide related functionality.

For example, a backend application may contain packages such as:

```text
handler
service
repository
middleware
config
database
```

Each package has a specific responsibility:

| Package | Responsibility |
|---|---|
| `handler` | Handles HTTP requests and responses |
| `service` | Contains business logic |
| `repository` | Communicates with the database |
| `middleware` | Contains HTTP middleware |
| `config` | Loads application configuration |
| `database` | Creates and manages database connections |

---

## 2. Creating a Simple Package

Consider this project:

```text
calculator/
├── main.go
└── mathutil/
    └── math.go
```

The `mathutil` folder represents a package.

### `mathutil/math.go`

```go
package mathutil

func Add(a, b int) int {
	return a + b
}

func Subtract(a, b int) int {
	return a - b
}
```

The first line declares the package name:

```go
package mathutil
```

This tells Go:

> This file belongs to the `mathutil` package.

### Using the Package

In `main.go`:

```go
package main

import (
	"fmt"

	"calculator/mathutil"
)

func main() {
	result := mathutil.Add(10, 20)

	fmt.Println(result)
}
```

Output:

```text
30
```

The function is accessed using:

```go
mathutil.Add(10, 20)
```

The general format is:

```text
packageName.FunctionName()
```

---

## 3. The `main` Package

The `main` package is a special package in Go.

It represents an executable application:

```go
package main

import "fmt"

func main() {
	fmt.Println("Application started")
}
```

For a Go application to be directly executable, it needs:

1. A package named `main`.
2. A function named `main`.

```go
package main

func main() {
	// Application starts here.
}
```

When you run:

```bash
go run .
```

Go looks for the `main` package and starts executing the `main()` function.

### Normal Package vs `main` Package

Normal package:

```go
package user

func CreateUser() {
	// Reusable application logic
}
```

This package provides reusable functionality.

Main package:

```go
package main

func main() {
	// Application entry point
}
```

This package produces an executable program.

---

## 4. One Folder Usually Represents One Package

In Go, all regular `.go` files inside the same folder must normally declare the same package.

Consider:

```text
user/
├── handler.go
├── service.go
└── repository.go
```

All three files should use the same package declaration.

### `user/handler.go`

```go
package user
```

### `user/service.go`

```go
package user
```

### `user/repository.go`

```go
package user
```

Together, these files form the `user` package:

```text
user package
├── handler.go
├── service.go
└── repository.go
```

Although the code is divided into multiple files, Go treats it as one package.

Therefore, code inside `handler.go` can directly use functions and types from `service.go` without importing it.

### `user/service.go`

```go
package user

func validateUser(name string) bool {
	return name != ""
}
```

### `user/handler.go`

```go
package user

func CreateUser(name string) {
	if !validateUser(name) {
		return
	}

	// Create the user.
}
```

No import is required because both files belong to the same package.

---

## 5. Exported and Unexported Names

Go uses capitalization to control whether a name is accessible from another package.

### Exported Names

Names beginning with an uppercase letter are exported:

```go
func CreateUser() {
}
```

```go
type User struct {
}
```

```go
var ErrUserNotFound = errors.New("user not found")
```

These can be accessed from another package:

```go
user.CreateUser()
```

### Unexported Names

Names beginning with a lowercase letter are unexported:

```go
func validateUser() {
}
```

```go
type userConfig struct {
}
```

```go
var defaultTimeout = 5 * time.Second
```

These can only be accessed inside the same package.

### Example

```go
package mathutil

func Add(a, b int) int {
	return a + b
}

func subtract(a, b int) int {
	return a - b
}
```

From another package:

```go
result := mathutil.Add(10, 20)
```

This works because `Add` begins with an uppercase letter.

This does not work:

```go
result := mathutil.subtract(20, 10)
```

It fails because `subtract` begins with a lowercase letter.

A useful mental model is:

```text
Uppercase name → Exported outside the package
Lowercase name → Accessible only inside the package
```

Technically, Go uses the terms **exported** and **unexported**, not public and private.

---

## 6. Importing Packages

The `import` statement allows one package to use code from another package.

```go
import "fmt"
```

For multiple packages:

```go
import (
	"context"
	"fmt"
	"net/http"
)
```

You can import three general types of packages.

### Standard-Library Packages

These are included with Go:

```go
import (
	"context"
	"encoding/json"
	"errors"
	"fmt"
	"net/http"
)
```

### Packages from Your Own Module

```go
import "github.com/charan/user-api/internal/service"
```

### Third-Party Packages

```go
import "github.com/gofiber/fiber/v3"
```

---

## 7. Package Name vs Import Path

The **import path** tells Go where the package is located.

The **package name** is used to access the package inside the code.

Consider:

```go
import "github.com/charan/user-api/internal/user"
```

The import path is:

```text
github.com/charan/user-api/internal/user
```

If the imported file declares:

```go
package user
```

You use it as:

```go
user.Create()
```

The last part of an import path and the package name usually match, but they are technically separate concepts.

---

## 8. Import Aliases

You can give an imported package a different local name:

```go
import userService "github.com/charan/user-api/internal/service"
```

Usage:

```go
userService.CreateUser()
```

Aliases are useful when two packages have the same name:

```go
import (
	userHandler "github.com/charan/user-api/internal/user/handler"
	orderHandler "github.com/charan/user-api/internal/order/handler"
)
```

Usage:

```go
userHandler.RegisterRoutes()
orderHandler.RegisterRoutes()
```

Use clear aliases. Avoid aliases when the original package name is already understandable.

---

## 9. Special Import Types

### Blank Import

A blank import uses `_`:

```go
import _ "github.com/lib/pq"
```

This imports a package only for its initialization side effects.

You cannot directly call functions from it.

Blank imports are sometimes used for:

- Database drivers.
- Plugin registration.
- Automatic initialization.

They should be used carefully because the package's behavior is less obvious.

### Dot Import

A dot import looks like this:

```go
import . "fmt"
```

It allows you to call exported names without the package prefix:

```go
Println("Hello")
```

Instead of:

```go
fmt.Println("Hello")
```

Dot imports are usually discouraged because it becomes difficult to identify where a function came from.

Prefer:

```go
fmt.Println("Hello")
```

---

## 10. What Is a Module?

A **module** is a collection of related Go packages that are versioned and managed together.

A module is identified by a `go.mod` file.

Example project:

```text
user-api/
├── go.mod
├── go.sum
├── main.go
├── handler/
│   └── user_handler.go
├── service/
│   └── user_service.go
└── repository/
    └── user_repository.go
```

The entire `user-api` project is a module.

Inside it are several packages:

```text
Module: user-api
├── Package: main
├── Package: handler
├── Package: service
└── Package: repository
```

---

## 11. Creating a Go Module

Create a project directory:

```bash
mkdir user-api
cd user-api
```

Initialize the module:

```bash
go mod init github.com/charan/user-api
```

This creates a `go.mod` file:

```go
module github.com/charan/user-api

go 1.25
```

The module path is:

```text
github.com/charan/user-api
```

This path becomes the prefix used to import packages from this module.

For example:

```go
import "github.com/charan/user-api/internal/service"
```

---

## 12. Understanding the Module Path

Suppose your `go.mod` contains:

```go
module github.com/charan/user-api
```

And your project contains:

```text
user-api/
├── go.mod
└── internal/
    └── service/
        └── user_service.go
```

The complete import path for the service package is:

```text
github.com/charan/user-api/internal/service
```

It is formed as:

```text
Module path + package folder path
```

```text
github.com/charan/user-api + /internal/service
```

Result:

```text
github.com/charan/user-api/internal/service
```

---

## 13. Module Path Does Not Have to Be a Real Website

You can initialize a local learning project using:

```bash
go mod init calculator
```

Then import a package like this:

```go
import "calculator/mathutil"
```

This is acceptable for local practice.

For a real project that will be published, use a globally unique module path:

```bash
go mod init github.com/your-username/project-name
```

For example:

```bash
go mod init github.com/charan/multitenant-api
```

The repository does not necessarily need to exist when you run the command, but the path should match where you intend to publish it.

---

## 14. What Is `go.mod`?

The `go.mod` file describes the module.

Example:

```go
module github.com/charan/user-api

go 1.25

require (
	github.com/gofiber/fiber/v3 v3.0.0
	github.com/google/uuid v1.6.0
)
```

It contains:

- The module path.
- The Go language version.
- Direct and indirect dependencies.
- Dependency versions.
- Optional replacement or exclusion rules.

### Module Declaration

```go
module github.com/charan/user-api
```

This identifies the module.

### Go Version

```go
go 1.25
```

This identifies the Go language and module behavior expected by the project.

### Dependencies

```go
require github.com/google/uuid v1.6.0
```

This tells Go that the application depends on version `v1.6.0` of the UUID package.

---

## 15. What Is `go.sum`?

The `go.sum` file contains checksums for downloaded module versions.

Example:

```text
github.com/google/uuid v1.6.0 h1:NIvaJDMOsjHA8...
github.com/google/uuid v1.6.0/go.mod h1:TIyPZe4Mgqv...
```

It helps Go verify that downloaded dependency files have not unexpectedly changed.

You normally should:

- Commit `go.mod` to Git.
- Commit `go.sum` to Git.
- Avoid manually editing `go.sum`.
- Let Go commands update it.

A simple way to remember:

```text
go.mod → Which dependencies are required?
go.sum → Are the downloaded dependencies trustworthy and unchanged?
```

---

## 16. Adding a Dependency

Suppose you want to use Fiber:

```bash
go get github.com/gofiber/fiber/v3
```

Go will:

1. Download the module.
2. Add the required version to `go.mod`.
3. Add checksums to `go.sum`.

You can then import it:

```go
package main

import "github.com/gofiber/fiber/v3"

func main() {
	app := fiber.New()

	app.Get("/", func(c fiber.Ctx) error {
		return c.SendString("Hello, World!")
	})

	app.Listen(":8080")
}
```

---

## 17. The `go mod tidy` Command

`go mod tidy` synchronizes your dependency files with your source code.

```bash
go mod tidy
```

It:

- Adds dependencies that your code imports but `go.mod` is missing.
- Removes unused dependencies from `go.mod`.
- Updates missing entries in `go.sum`.

You should commonly run it after:

- Adding new imports.
- Removing a library.
- Refactoring packages.
- Pulling a project from Git.
- Resolving dependency problems.

A common workflow is:

```bash
go mod tidy
go test ./...
go build ./...
```

---

## 18. Direct and Indirect Dependencies

Your `go.mod` may contain:

```go
require (
	github.com/gofiber/fiber/v3 v3.0.0
	github.com/example/library v1.2.0 // indirect
)
```

### Direct Dependency

Your code imports it directly:

```go
import "github.com/gofiber/fiber/v3"
```

### Indirect Dependency

Your code does not directly import it, but one of your dependencies needs it.

```text
Your application
      ↓
Fiber
      ↓
Another library
```

That other library is an indirect dependency of your application.

---

## 19. Package vs Module

This is the most important distinction:

| Package | Module |
|---|---|
| Organizes related Go code | Organizes one or more packages |
| Usually corresponds to a folder | Usually corresponds to a project or repository |
| Declared using `package` | Declared using `module` in `go.mod` |
| Imported by other packages | Versioned as a dependency |
| Examples: `service`, `repository` | Example: `github.com/charan/user-api` |
| Does not require its own `go.mod` | Must have a `go.mod` file |

Example:

```text
github.com/charan/user-api       ← Module
├── go.mod
├── cmd/
│   └── api/
│       └── main.go              ← main package
├── internal/
│   ├── handler/                 ← handler package
│   ├── service/                 ← service package
│   └── repository/              ← repository package
└── pkg/
    └── response/                ← response package
```

---

## 20. Package vs Module vs Repository

These terms are related but not identical.

### Package

A group of related Go files:

```text
internal/service
```

### Module

A collection of packages managed by a `go.mod` file:

```text
github.com/charan/user-api
```

### Repository

A source-control location, usually a Git repository:

```text
https://github.com/charan/user-api
```

A common setup is:

```text
One Git repository
    ↓
One Go module
    ↓
Multiple Go packages
```

However, a repository can contain multiple modules if each module has its own `go.mod`.

That is useful in some large projects, but most applications should begin with one module.

---

## 21. A Backend Project Example

Consider this structure:

```text
user-api/
├── go.mod
├── go.sum
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── config/
│   │   └── config.go
│   ├── handler/
│   │   └── user_handler.go
│   ├── service/
│   │   └── user_service.go
│   ├── repository/
│   │   └── user_repository.go
│   └── model/
│       └── user.go
└── pkg/
    └── response/
        └── response.go
```

This is one module containing multiple packages.

### `go.mod`

```go
module github.com/charan/user-api

go 1.25
```

### `internal/model/user.go`

```go
package model

type User struct {
	ID    int64
	Name  string
	Email string
}
```

### `internal/repository/user_repository.go`

```go
package repository

import (
	"context"

	"github.com/charan/user-api/internal/model"
)

type UserRepository struct {
}

func (r *UserRepository) FindByID(
	ctx context.Context,
	id int64,
) (*model.User, error) {
	return &model.User{
		ID:    id,
		Name:  "Charan",
		Email: "charan@example.com",
	}, nil
}
```

### `internal/service/user_service.go`

```go
package service

import (
	"context"
	"fmt"

	"github.com/charan/user-api/internal/model"
	"github.com/charan/user-api/internal/repository"
)

type UserService struct {
	repository *repository.UserRepository
}

func NewUserService(
	repository *repository.UserRepository,
) *UserService {
	return &UserService{
		repository: repository,
	}
}

func (s *UserService) GetUser(
	ctx context.Context,
	id int64,
) (*model.User, error) {
	if id <= 0 {
		return nil, fmt.Errorf("invalid user ID: %d", id)
	}

	return s.repository.FindByID(ctx, id)
}
```

### `cmd/api/main.go`

```go
package main

import (
	"context"
	"fmt"

	"github.com/charan/user-api/internal/repository"
	"github.com/charan/user-api/internal/service"
)

func main() {
	userRepository := &repository.UserRepository{}
	userService := service.NewUserService(userRepository)

	user, err := userService.GetUser(context.Background(), 1)
	if err != nil {
		fmt.Println("error:", err)
		return
	}

	fmt.Println(user)
}
```

The dependency flow is:

```text
main package
    ↓
service package
    ↓
repository package
    ↓
model package
```

---

## 22. The `internal` Directory

The `internal` directory has special meaning in Go.

```text
user-api/
└── internal/
    ├── service/
    ├── repository/
    └── handler/
```

Packages inside `internal` cannot be imported freely by unrelated external modules.

For example:

```text
github.com/charan/user-api/internal/service
```

is intended for use inside the `user-api` project, not by completely unrelated applications.

This is enforced by the Go compiler.

Use `internal` for application-specific code such as:

- Handlers.
- Services.
- Repositories.
- Database logic.
- Middleware.
- Configuration.
- Domain logic.

It communicates:

> This code belongs to this application and is not a public reusable library.

---

## 23. The `pkg` Directory

Some Go projects use a `pkg` directory:

```text
pkg/
├── logger/
├── response/
└── validator/
```

Unlike `internal`, `pkg` has no special compiler behavior.

It is only a project convention.

Packages under `pkg` usually represent code that may be imported by external projects.

However, you do not need to create `pkg` automatically. Use it only when you intentionally provide reusable public packages.

For application-specific code, `internal` is generally more appropriate.

---

## 24. The `cmd` Directory

The `cmd` directory commonly contains application entry points.

```text
cmd/
├── api/
│   └── main.go
└── worker/
    └── main.go
```

This module produces two executable programs:

1. An HTTP API.
2. A background worker.

Run the API:

```bash
go run ./cmd/api
```

Run the worker:

```bash
go run ./cmd/worker
```

Each directory contains its own `main` package:

```go
package main

func main() {
	// Start the application.
}
```

The `cmd` directory is a convention, not a special Go keyword.

---

## 25. Package Initialization

Packages can contain an `init()` function:

```go
package config

import "fmt"

func init() {
	fmt.Println("config package initialized")
}
```

Go automatically calls `init()` before calling `main()`.

A package can technically have multiple `init()` functions, although that can make behavior difficult to understand.

A simplified initialization order is:

```text
Imported dependencies
        ↓
Package-level variables
        ↓
init() functions
        ↓
main()
```

Example:

```go
package main

import "fmt"

var appName = loadAppName()

func loadAppName() string {
	fmt.Println("initializing variable")
	return "User API"
}

func init() {
	fmt.Println("running init")
}

func main() {
	fmt.Println("running main")
}
```

Output:

```text
initializing variable
running init
running main
```

Use `init()` carefully. Explicit initialization is usually easier to understand and test:

```go
func main() {
	config, err := LoadConfig()
	if err != nil {
		log.Fatal(err)
	}

	// Continue starting the application.
}
```

---

## 26. Import Cycles

Go does not allow circular imports.

Suppose:

```text
service imports repository
repository imports service
```

This creates a cycle:

```text
service
   ↓
repository
   ↓
service
```

Go reports an error similar to:

```text
import cycle not allowed
```

### Example of a Cycle

Service package:

```go
package service

import "myapp/repository"
```

Repository package:

```go
package repository

import "myapp/service"
```

This is not allowed.

### How to Avoid Import Cycles

Create a clear dependency direction:

```text
handler
   ↓
service
   ↓
repository
   ↓
database
```

Lower-level packages should not import higher-level packages.

Interfaces can also help break inappropriate dependencies:

```go
package service

type UserRepository interface {
	FindByID(id int64) (*User, error)
}
```

The service depends on an abstraction rather than a concrete repository implementation.

Another solution is to move shared types into a separate package:

```text
handler ──────┐
              ↓
            model
              ↑
repository ───┘
```

Do not create a generic shared package merely to hide a poor dependency design.

First, decide which package should own the type or behavior.

---

## 27. Good Package Design

A good package should have one clear responsibility.

Good examples:

```text
auth
user
order
payment
config
database
```

Avoid unclear package names such as:

```text
utils
common
helpers
misc
```

These packages often become dumping grounds for unrelated functions.

Instead of this:

```text
utils/
├── jwt.go
├── password.go
├── email.go
├── response.go
└── date.go
```

Prefer focused packages:

```text
auth/
├── jwt.go
└── password.go

email/
└── sender.go

response/
└── response.go
```

The package name should explain its responsibility.

---

## 28. Organizing by Technical Layer vs Feature

There are two common ways to organize backend packages.

### Layer-Based Organization

```text
internal/
├── handler/
├── service/
├── repository/
├── model/
└── middleware/
```

This structure is easy to understand for a small or medium application.

However, as the project grows, each directory may contain files for many unrelated features:

```text
service/
├── user_service.go
├── order_service.go
├── payment_service.go
└── product_service.go
```

### Feature-Based Organization

```text
internal/
├── user/
│   ├── handler.go
│   ├── service.go
│   └── repository.go
├── order/
│   ├── handler.go
│   ├── service.go
│   └── repository.go
└── payment/
    ├── handler.go
    ├── service.go
    └── repository.go
```

Each business feature owns its related code.

This often works better for larger applications because all user-related code stays together.

Neither structure is automatically correct for every project.

A practical approach is:

- Use layer-based organization for a smaller learning project.
- Consider feature-based organization when the project contains many business domains.
- Avoid creating many packages before the application actually needs them.

---

## 29. Package APIs

The exported names of a package form its API.

Consider:

```go
package user

type Service struct {
	repository Repository
}

func NewService(repository Repository) *Service {
	return &Service{
		repository: repository,
	}
}

func (s *Service) GetByID(id int64) (*User, error) {
	return s.repository.FindByID(id)
}

func validateID(id int64) bool {
	return id > 0
}
```

The package exposes:

```go
Service
NewService
GetByID
```

It hides:

```go
validateID
repository
```

A good package exports only what callers actually need.

This reduces coupling and makes the package easier to change.

---

## 30. Package Naming Best Practices

Good package names are:

- Short.
- Clear.
- Lowercase.
- Usually singular.
- Related to what the package provides.

Good:

```text
user
order
auth
config
database
middleware
```

Avoid:

```text
user_service_package
MyPackage
commonUtilities
all_helpers
```

Avoid repeating the package name in exported names.

Less idiomatic:

```go
user.UserService
user.NewUserService()
```

Often better:

```go
user.Service
user.NewService()
```

The package already provides the `user` context.

Similarly, the standard library uses names such as:

```go
http.Client
json.Decoder
sql.DB
```

Not:

```go
http.HTTPClient
json.JSONDecoder
sql.SQLDatabase
```

---

## 31. Common Go Module Commands

### Initialize a Module

```bash
go mod init github.com/charan/user-api
```

### Add or Update a Dependency

```bash
go get github.com/google/uuid
```

### Get a Specific Version

```bash
go get github.com/google/uuid@v1.6.0
```

### Synchronize Dependencies

```bash
go mod tidy
```

### Download Dependencies

```bash
go mod download
```

### List Modules

```bash
go list -m all
```

### Explain Why a Dependency Is Needed

```bash
go mod why github.com/example/dependency
```

### Verify Downloaded Dependencies

```bash
go mod verify
```

### Show Available Module Versions

```bash
go list -m -versions github.com/google/uuid
```

---

## 32. Local Module Replacement

Suppose you are developing two modules locally:

```text
projects/
├── user-api/
│   └── go.mod
└── shared-logger/
    └── go.mod
```

The API depends on the logger module, but the logger is not published yet.

You can temporarily add this to `user-api/go.mod`:

```go
replace github.com/charan/shared-logger => ../shared-logger
```

This tells Go:

> When this module imports `github.com/charan/shared-logger`, use the local directory instead.

You can also add it using:

```bash
go mod edit \
  -replace github.com/charan/shared-logger=../shared-logger
```

The `replace` directive is especially useful during local development.

Local filesystem replacements should normally not be relied on for production builds.

---

## 33. Versioning Go Modules

Go modules usually follow semantic versioning:

```text
vMAJOR.MINOR.PATCH
```

Example:

```text
v1.4.2
```

- `MAJOR`: Breaking changes.
- `MINOR`: New backward-compatible functionality.
- `PATCH`: Backward-compatible bug fixes.

```text
v1.4.2
│ │ └── Patch version
│ └──── Minor version
└────── Major version
```

For major version 2 and above, the module path normally includes the major version:

```go
module github.com/charan/mylibrary/v2
```

Consumers import it using:

```go
import "github.com/charan/mylibrary/v2"
```

This allows different major versions to be used as different module paths.

---

## 34. Common Mistakes

### Mistake 1: Using Relative Imports

Do not use:

```go
import "../service"
```

Use the complete module-based import path:

```go
import "github.com/charan/user-api/internal/service"
```

### Mistake 2: Creating a Module Inside Every Package

You usually do not need this:

```text
user-api/
├── go.mod
├── handler/
│   └── go.mod
├── service/
│   └── go.mod
└── repository/
    └── go.mod
```

For most applications, use one module:

```text
user-api/
├── go.mod
├── handler/
├── service/
└── repository/
```

Only use multiple modules when there is a clear need for independent versioning or dependency management.

### Mistake 3: Mixing Package Names in One Folder

This is incorrect for ordinary source files:

```text
user/
├── handler.go       → package handler
└── service.go       → package service
```

Put them in separate directories:

```text
handler/
└── handler.go       → package handler

service/
└── service.go       → package service
```

Alternatively, if both files belong to one cohesive package, give them the same package name.

### Mistake 4: Exporting Everything

Avoid making every function, field, and type uppercase:

```go
type UserService struct {
	Repository UserRepository
}
```

If callers do not need direct access to the repository field, hide it:

```go
type Service struct {
	repository Repository
}
```

Export only the package API that callers require.

### Mistake 5: Using One Huge Package

Avoid putting the entire application into `package main`:

```text
main.go
user.go
order.go
payment.go
database.go
email.go
middleware.go
```

This may work initially, but it becomes difficult to understand and test as the application grows.

Separate cohesive responsibilities into packages.

### Mistake 6: Creating Too Many Tiny Packages

Not every file needs its own package.

Avoid:

```text
getuser/
createuser/
updateuser/
deleteuser/
```

If these operations belong together, use one package:

```text
user/
├── get.go
├── create.go
├── update.go
└── delete.go
```

A package should represent a cohesive responsibility, not merely one function.

### Mistake 7: Creating Import Cycles

Avoid dependencies such as:

```text
handler → service → repository → handler
```

Prefer a one-directional dependency flow:

```text
handler → service → repository
```

---

## 35. Easy Mental Model

Imagine a company.

### Files Are Employees

Each file contains code performing some work:

```text
handler.go
service.go
repository.go
```

### A Package Is a Team

Employees working on the same responsibility belong to one team:

```text
user package
├── handler.go
├── service.go
└── repository.go
```

### A Module Is the Company

The company contains multiple teams:

```text
user-api module
├── user package
├── auth package
├── order package
└── database package
```

### `go.mod` Is the Company Registration Document

It identifies:

- The module's name.
- The expected Go version.
- The external modules it depends on.

```text
Files → Members of a package
Packages → Parts of a module
Module → Managed by go.mod
```

---

## 36. Complete Summary

```text
Go file
   ↓ belongs to
Package
   ↓ belongs to
Module
   ↓ defined by
go.mod
```

Example:

```text
File:
internal/service/user_service.go

Package:
service

Module:
github.com/charan/user-api

Import path:
github.com/charan/user-api/internal/service
```

Important rules:

1. A package is a collection of related Go files.
2. A module is a collection of related packages.
3. A module is identified by a `go.mod` file.
4. One folder normally represents one package.
5. Files in the same folder normally use the same package name.
6. Uppercase names are exported.
7. Lowercase names are accessible only inside the package.
8. Use the module path as the prefix for internal imports.
9. Use `internal` for application-private packages.
10. Use `cmd` for executable entry points.
11. Avoid circular imports.
12. Keep package responsibilities clear and cohesive.
13. Commit both `go.mod` and `go.sum`.
14. Run `go mod tidy` after changing dependencies.
15. Start with one module unless multiple modules are genuinely necessary.
