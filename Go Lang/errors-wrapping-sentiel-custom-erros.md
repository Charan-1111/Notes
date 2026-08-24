# Error Handling in Go

Errors are unavoidable in backend systems. A database may be unavailable, input may be invalid, a requested resource may not exist, or an external API may time out.

Go treats errors as ordinary values. Instead of throwing exceptions, a function returns an `error`, and the caller decides how to handle it.

## 1. The `error` Interface

In Go, `error` is a built-in interface:

```go
type error interface {
	Error() string
}
```

Any type that implements an `Error() string` method can be used as an error.

```go
func divide(a, b float64) (float64, error) {
	if b == 0 {
		return 0, errors.New("cannot divide by zero")
	}

	return a / b, nil
}
```

Usage:

```go
result, err := divide(10, 0)
if err != nil {
	fmt.Println("operation failed:", err)
	return
}

fmt.Println(result)
```

- `err == nil` means the operation succeeded.
- `err != nil` means the operation failed.

## 2. Creating Simple Errors

### Using `errors.New`

Use `errors.New` when the error message is static:

```go
func withdraw(balance, amount float64) error {
	if amount > balance {
		return errors.New("insufficient balance")
	}

	return nil
}
```

### Using `fmt.Errorf`

Use `fmt.Errorf` when the message needs dynamic information:

```go
func withdraw(balance, amount float64) error {
	if amount > balance {
		return fmt.Errorf(
			"cannot withdraw %.2f: available balance is %.2f",
			amount,
			balance,
		)
	}

	return nil
}
```

Example output:

```text
cannot withdraw 5000.00: available balance is 2000.00
```

## 3. Handling Errors

Check errors immediately after calling a function:

```go
user, err := getUser(10)
if err != nil {
	return err
}
```

You can add context before returning the error:

```go
user, err := getUser(10)
if err != nil {
	return fmt.Errorf("getting user: %w", err)
}
```

## 4. Error Wrapping

Suppose a database operation returns:

```text
connection refused
```

This message does not tell us which application operation failed. Wrapping preserves the original error while adding useful context.

```go
func getUser(id int) error {
	err := queryDatabase(id)
	if err != nil {
		return fmt.Errorf("getting user %d: %w", id, err)
	}

	return nil
}
```

The final message becomes:

```text
getting user 10: connection refused
```

The `%w` verb wraps and preserves the original error.

### Wrapping Through Multiple Layers

```text
HTTP Handler
    ↓
User Service
    ↓
User Repository
    ↓
Database
```

Repository layer:

```go
func (r *UserRepository) FindByID(id int) error {
	err := queryDatabase(id)
	if err != nil {
		return fmt.Errorf("querying user %d: %w", id, err)
	}

	return nil
}
```

Service layer:

```go
func (s *UserService) GetUser(id int) error {
	err := s.repository.FindByID(id)
	if err != nil {
		return fmt.Errorf("getting user: %w", err)
	}

	return nil
}
```

The complete error could be:

```text
getting user: querying user 10: connection refused
```

Each layer adds information about what it was doing.

### `%w` Versus `%v`

`%w` preserves the original error:

```go
return fmt.Errorf("getting user: %w", err)
```

The error can later be inspected with `errors.Is`, `errors.As`, or `errors.Unwrap`.

`%v` includes only the original error's message:

```go
return fmt.Errorf("getting user: %v", err)
```

It does not preserve the original error in the error chain. Use `%w` when adding context to an error that you are returning.

## 5. Sentinel Errors

A **sentinel error** is a predefined error value representing a known condition.

```go
var ErrUserNotFound = errors.New("user not found")
```

Several domain errors can be grouped together:

```go
var (
	ErrUserNotFound      = errors.New("user not found")
	ErrEmailAlreadyUsed  = errors.New("email already used")
	ErrInvalidCredential = errors.New("invalid credentials")
)
```

A function can return one of these known values:

```go
func findUser(id int) error {
	if id != 1 {
		return ErrUserNotFound
	}

	return nil
}
```

Use `errors.Is` to identify it:

```go
err := findUser(10)

if errors.Is(err, ErrUserNotFound) {
	fmt.Println("the requested user does not exist")
}
```

### Why Not Compare Errors with `==`?

Direct comparison works only when the error has not been wrapped:

```go
if err == ErrUserNotFound {
	// Handle it.
}
```

After wrapping, direct comparison fails:

```go
err := fmt.Errorf("repository failure: %w", ErrUserNotFound)
```

Use `errors.Is` instead:

```go
if errors.Is(err, ErrUserNotFound) {
	// This still works after wrapping.
}
```

`errors.Is` walks through the complete error chain.

## 6. Custom Errors

Sentinel errors identify a condition but normally do not carry extra details. A custom error type can include information such as the invalid field, resource ID, or retry duration.

```go
type ValidationError struct {
	Field   string
	Message string
}
```

Implement the `error` interface:

```go
func (e *ValidationError) Error() string {
	return fmt.Sprintf(
		"validation failed for field %q: %s",
		e.Field,
		e.Message,
	)
}
```

Return the custom error:

```go
func createUser(email string) error {
	if email == "" {
		return &ValidationError{
			Field:   "email",
			Message: "email is required",
		}
	}

	return nil
}
```

## 7. Inspecting Custom Errors with `errors.As`

Use `errors.As` to find a particular error type and access its fields:

```go
err := createUser("")

var validationErr *ValidationError
if errors.As(err, &validationErr) {
	fmt.Println("Field:", validationErr.Field)
	fmt.Println("Message:", validationErr.Message)
}
```

It also works when the custom error has been wrapped:

```go
func registerUser(email string) error {
	err := createUser(email)
	if err != nil {
		return fmt.Errorf("registering user: %w", err)
	}

	return nil
}
```

```go
err := registerUser("")

var validationErr *ValidationError
if errors.As(err, &validationErr) {
	fmt.Println(validationErr.Field)
}
```

## 8. `errors.Is` Versus `errors.As`

Use `errors.Is` to match a known error **value**:

```go
if errors.Is(err, ErrUserNotFound) {
	// The chain contains ErrUserNotFound.
}
```

Use `errors.As` to find and extract an error **type**:

```go
var validationErr *ValidationError
if errors.As(err, &validationErr) {
	fmt.Println(validationErr.Field)
}
```

| Function | Purpose | Example |
|---|---|---|
| `errors.Is` | Match a known error value | `ErrUserNotFound` |
| `errors.As` | Find and extract an error type | `*ValidationError` |
| `errors.Unwrap` | Retrieve the directly wrapped error | Low-level inspection |

Common `errors.Is` checks include:

```go
errors.Is(err, sql.ErrNoRows)
errors.Is(err, context.DeadlineExceeded)
errors.Is(err, context.Canceled)
errors.Is(err, os.ErrNotExist)
```

## 9. Unwrapping an Error

`errors.Unwrap` returns the error wrapped directly inside another error:

```go
originalErr := errors.New("database unavailable")
wrappedErr := fmt.Errorf("getting user: %w", originalErr)

fmt.Println(wrappedErr)
fmt.Println(errors.Unwrap(wrappedErr))
```

Output:

```text
getting user: database unavailable
database unavailable
```

Normally, prefer `errors.Is` and `errors.As` because they can inspect the complete chain.

## 10. Custom Errors That Wrap Another Error

A custom error can store its underlying cause:

```go
type DatabaseError struct {
	Operation string
	Err       error
}

func (e *DatabaseError) Error() string {
	return fmt.Sprintf(
		"database operation %q failed: %v",
		e.Operation,
		e.Err,
	)
}

func (e *DatabaseError) Unwrap() error {
	return e.Err
}
```

Usage:

```go
func queryUser() error {
	originalErr := errors.New("connection refused")

	return &DatabaseError{
		Operation: "find user",
		Err:       originalErr,
	}
}
```

Because `DatabaseError` implements `Unwrap()`, `errors.Is` and `errors.As` can continue walking through the chain.

## 11. Combining Sentinel and Custom Errors

Sometimes you want both a general error category and detailed information.

```go
var ErrValidation = errors.New("validation failed")

type FieldError struct {
	Field string
	Value string
}

func (e *FieldError) Error() string {
	return fmt.Sprintf("%s: invalid value %q", e.Field, e.Value)
}

func (e *FieldError) Unwrap() error {
	return ErrValidation
}
```

Return it:

```go
func validateAge(age string) error {
	return &FieldError{
		Field: "age",
		Value: age,
	}
}
```

Check the general category:

```go
err := validateAge("-10")

if errors.Is(err, ErrValidation) {
	fmt.Println("this is a validation error")
}
```

Extract the details:

```go
var fieldErr *FieldError
if errors.As(err, &fieldErr) {
	fmt.Println("invalid field:", fieldErr.Field)
	fmt.Println("invalid value:", fieldErr.Value)
}
```

## 12. Practical Backend Example

### Domain Errors

```go
package domain

import "errors"

var (
	ErrUserNotFound     = errors.New("user not found")
	ErrEmailAlreadyUsed = errors.New("email already used")
)
```

### Repository Layer

The repository translates database-specific errors into application-level errors:

```go
func (r *UserRepository) FindByID(
	ctx context.Context,
	id int64,
) (*User, error) {
	var user User

	err := r.db.QueryRowContext(
		ctx,
		`SELECT id, name, email FROM users WHERE id = $1`,
		id,
	).Scan(&user.ID, &user.Name, &user.Email)

	if err != nil {
		if errors.Is(err, sql.ErrNoRows) {
			return nil, domain.ErrUserNotFound
		}

		return nil, fmt.Errorf("querying user %d: %w", id, err)
	}

	return &user, nil
}
```

### Service Layer

```go
func (s *UserService) GetUser(
	ctx context.Context,
	id int64,
) (*User, error) {
	if id <= 0 {
		return nil, &ValidationError{
			Field:   "id",
			Message: "must be greater than zero",
		}
	}

	user, err := s.repository.FindByID(ctx, id)
	if err != nil {
		return nil, fmt.Errorf("getting user: %w", err)
	}

	return user, nil
}
```

### HTTP Handler

```go
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
	user, err := h.service.GetUser(r.Context(), 10)
	if err != nil {
		var validationErr *ValidationError

		switch {
		case errors.Is(err, domain.ErrUserNotFound):
			http.Error(w, "user not found", http.StatusNotFound)

		case errors.As(err, &validationErr):
			http.Error(w, validationErr.Error(), http.StatusBadRequest)

		case errors.Is(err, context.DeadlineExceeded):
			http.Error(w, "request timed out", http.StatusGatewayTimeout)

		case errors.Is(err, context.Canceled):
			// The client disconnected or cancelled the request.
			return

		default:
			log.Printf("unexpected error: %v", err)
			http.Error(w, "internal server error", http.StatusInternalServerError)
		}

		return
	}

	json.NewEncoder(w).Encode(user)
}
```

Typical HTTP mappings:

| Error condition | HTTP status |
|---|---:|
| Validation error | `400 Bad Request` |
| Invalid credentials | `401 Unauthorized` |
| Permission denied | `403 Forbidden` |
| Resource not found | `404 Not Found` |
| Duplicate/conflict | `409 Conflict` |
| Deadline exceeded | `504 Gateway Timeout` |
| Unexpected error | `500 Internal Server Error` |

## 13. When to Use Each Approach

### Use a Simple Error When

- The error is local.
- The caller does not need to identify it programmatically.
- No structured information is required.

```go
return errors.New("configuration is empty")
```

### Use a Sentinel Error When

- Multiple callers need to recognize a known condition.
- The error does not need additional data.

```go
var ErrUserNotFound = errors.New("user not found")
```

### Use a Custom Error When

- The caller needs structured details.
- You need data such as a field name, resource ID, or retry delay.

```go
type ValidationError struct {
	Field   string
	Message string
}
```

### Wrap an Error When

- You want to add operation-specific context.
- You need to preserve the original cause.

```go
return fmt.Errorf("creating order: %w", err)
```

## 14. Common Mistakes

### Ignoring Errors

Bad:

```go
user, _ := getUser(10)
```

Better:

```go
user, err := getUser(10)
if err != nil {
	return err
}
```

### Comparing Error Messages

Bad:

```go
if err.Error() == "user not found" {
	// Handle it.
}
```

Better:

```go
if errors.Is(err, ErrUserNotFound) {
	// Handle it.
}
```

### Using `%v` When Wrapping Is Required

Bad:

```go
return fmt.Errorf("getting user: %v", err)
```

Better:

```go
return fmt.Errorf("getting user: %w", err)
```

### Logging and Returning the Same Error at Every Layer

If every layer logs the same error, logs become duplicated. A useful rule is:

- Lower layers add context and return errors.
- The boundary layer logs the final error once.

A boundary might be an HTTP handler, background worker, or message consumer.

```go
func service() error {
	err := repository()
	if err != nil {
		return fmt.Errorf("processing user: %w", err)
	}

	return nil
}
```

### Exposing Internal Details to Clients

Do not return internal errors like this to an API client:

```text
dial tcp 10.0.2.15:5432: connection refused
```

Return a safe response:

```json
{
  "error": "internal server error"
}
```

Log the complete wrapped error internally:

```text
getting user: querying PostgreSQL: dial tcp 10.0.2.15:5432: connection refused
```

### Wrapping Without Useful Context

Avoid:

```go
return fmt.Errorf("error happened: %w", err)
```

Prefer:

```go
return fmt.Errorf("fetching user %d: %w", userID, err)
```

## 15. Easy Mental Model

Think of an error as a package moving through your application:

```text
Database error
    ↓
Repository adds: "querying user 10"
    ↓
Service adds: "getting user"
    ↓
Handler identifies the error
    ↓
Handler selects the HTTP response
```

The final internal error could be:

```text
getting user: querying user 10: connection refused
```

But the client receives only:

```json
{
  "error": "internal server error"
}
```

The chain preserves the technical cause while each layer adds useful context.

## Quick Reference

```go
// Create a static error.
err := errors.New("something failed")

// Create a formatted error.
err := fmt.Errorf("user %d not found", id)

// Wrap an existing error.
err := fmt.Errorf("getting user: %w", originalErr)

// Check for a known error value.
if errors.Is(err, ErrUserNotFound) {
}

// Extract a custom error type.
var validationErr *ValidationError
if errors.As(err, &validationErr) {
}

// Get the directly wrapped error.
originalErr := errors.Unwrap(err)
```

## Key Rules to Remember

1. Treat errors as normal return values.
2. Check errors explicitly.
3. Add useful context using `%w`.
4. Use `errors.Is` for sentinel error values.
5. Use `errors.As` for custom error types.
6. Do not compare error messages.
7. Log an error once at the application boundary.
8. Do not expose internal errors directly to API clients.
