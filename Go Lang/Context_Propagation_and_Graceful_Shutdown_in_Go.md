# Context Propagation and Graceful Shutdown in Go

Backend applications often perform work across multiple layers:

```text
HTTP Handler
    ↓
Service
    ↓
Repository
    ↓
Database / External API
```

Two important questions arise:

1. What happens if the client cancels the request?
2. What happens to active requests when the server is stopped or redeployed?

Go addresses these problems using:

- **Context propagation** — communicates cancellation, deadlines, and request-scoped values across function boundaries.
- **Graceful shutdown** — stops accepting new work while allowing current work to finish safely.

---

## 1. What Is Context?

The `context.Context` interface carries request-scoped information through your application. It mainly carries:

- Cancellation signals
- Deadlines
- Timeouts
- Small request-scoped values

Think of a context as a cancellation signal that travels with a request.

```text
Client request
      │
      ▼
HTTP Handler ─── ctx
      │
      ▼
Service      ─── ctx
      │
      ▼
Repository   ─── ctx
      │
      ▼
Database
```

If the client disconnects or the timeout expires, the same cancellation signal reaches every layer.

## 2. Why Context Propagation Is Important

Imagine an endpoint that fetches data from a database. If the database query takes 20 seconds but the client disconnects after 5 seconds, the query may continue unnecessarily unless cancellation is propagated.

This wastes database connections, CPU, memory, goroutines, and external-service capacity.

Pass the request context through every layer:

```go
func GetUserHandler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	user, err := userService.GetUser(ctx, 10)
	if err != nil {
		http.Error(w, "failed to get user", http.StatusInternalServerError)
		return
	}

	json.NewEncoder(w).Encode(user)
}
```

Now cancellation can flow from the HTTP request to the database or external service.

## 3. The `context.Context` Interface

```go
type Context interface {
	Deadline() (deadline time.Time, ok bool)
	Done() <-chan struct{}
	Err() error
	Value(key any) any
}
```

- `Deadline()` returns the time at which the operation should stop.
- `Done()` returns a channel that closes when the context is cancelled.
- `Err()` explains why it ended—usually `context.Canceled` or `context.DeadlineExceeded`.
- `Value()` retrieves small request-scoped values such as a request ID.

## 4. Creating Contexts

### `context.Background()`

Creates an empty root context:

```go
ctx := context.Background()
```

It has no deadline, cancellation, or values. It is commonly used at the beginning of an application or independent background operation.

### `context.TODO()`

Use this temporarily when a context is required but you have not decided which context should be passed:

```go
ctx := context.TODO()
```

## 5. Context With Cancellation

`context.WithCancel` creates a child context that can be cancelled manually.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func worker(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("worker stopped:", ctx.Err())
			return
		default:
			fmt.Println("worker is running")
			time.Sleep(time.Second)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	go worker(ctx)

	time.Sleep(3 * time.Second)
	cancel()
	time.Sleep(time.Second)
}
```

Always call the returned cancellation function because it releases resources associated with the child context.

## 6. Context With a Timeout

`context.WithTimeout` automatically cancels a context after a duration:

```go
func performOperation(ctx context.Context) error {
	select {
	case <-time.After(5 * time.Second):
		fmt.Println("operation completed")
		return nil
	case <-ctx.Done():
		return ctx.Err()
	}
}

func main() {
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	err := performOperation(ctx)
	fmt.Println(err) // context deadline exceeded
}
```

## 7. Context With a Deadline

A deadline specifies an exact time when cancellation should occur:

```go
deadline := time.Now().Add(5 * time.Second)
ctx, cancel := context.WithDeadline(context.Background(), deadline)
defer cancel()
```

- `WithTimeout(ctx, 5*time.Second)` means cancel five seconds from now.
- `WithDeadline(ctx, deadline)` means cancel at a specific time.

## 8. Propagating Context Through Application Layers

Handler:

```go
func handler(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	user, err := userService.GetUser(ctx, 10)
	// Handle result...
}
```

Service:

```go
type UserService struct {
	repository *UserRepository
}

func (s *UserService) GetUser(ctx context.Context, userID int) (*User, error) {
	return s.repository.GetByID(ctx, userID)
}
```

Repository:

```go
type UserRepository struct {
	db *sql.DB
}

func (r *UserRepository) GetByID(ctx context.Context, userID int) (*User, error) {
	query := `
		SELECT id, name, email
		FROM users
		WHERE id = $1
	`

	var user User
	err := r.db.QueryRowContext(ctx, query, userID).Scan(
		&user.ID,
		&user.Name,
		&user.Email,
	)
	if err != nil {
		return nil, err
	}

	return &user, nil
}
```

The flow is:

```text
r.Context()
    ↓
service.GetUser(ctx)
    ↓
repository.GetByID(ctx)
    ↓
db.QueryRowContext(ctx)
```

## 9. Propagating Context to an External API

Use `http.NewRequestWithContext`:

```go
func fetchProfile(ctx context.Context, userID int) ([]byte, error) {
	url := fmt.Sprintf("https://profile-service/users/%d", userID)

	req, err := http.NewRequestWithContext(ctx, http.MethodGet, url, nil)
	if err != nil {
		return nil, err
	}

	response, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer response.Body.Close()

	return io.ReadAll(response.Body)
}
```

If the original client cancels, the outgoing request can be cancelled too.

## 10. Parent and Child Contexts

Contexts form a tree:

```text
Request context
      │
      ├── Database timeout context
      └── External API timeout context
```

```go
dbCtx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
defer cancel()
```

Cancelling a parent cancels all its children. A child can impose a shorter deadline, but it cannot extend its parent’s lifetime.

## 11. Checking Cancellation in Long-Running Work

Database and HTTP APIs often understand context already. Your own loops must check it explicitly:

```go
func processItems(ctx context.Context, items []string) error {
	for _, item := range items {
		select {
		case <-ctx.Done():
			return ctx.Err()
		default:
			fmt.Println("processing:", item)
		}

		time.Sleep(500 * time.Millisecond)
	}

	return nil
}
```

For a worker waiting for jobs:

```go
func worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println("worker shutting down")
			return
		case job, ok := <-jobs:
			if !ok {
				return
			}
			process(job)
		}
	}
}
```

## 12. Request-Scoped Values

Contexts can carry small values such as request IDs, trace IDs, authenticated user IDs, and tenant IDs.

Use a private custom key type to prevent collisions:

```go
type contextKey string

const requestIDKey contextKey = "request-id"
```

Middleware can add a request ID:

```go
func requestIDMiddleware(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		requestID := generateRequestID()
		ctx := context.WithValue(r.Context(), requestIDKey, requestID)
		next.ServeHTTP(w, r.WithContext(ctx))
	})
}
```

Do not use context as a general-purpose parameter container. Business inputs belong in normal parameters or structs.

## 13. Context Best Practices

### Accept context as the first parameter

```go
func GetUser(ctx context.Context, id int) error
```

### Do not store request contexts in long-lived structs

Avoid:

```go
type UserService struct {
	ctx context.Context
}
```

Pass context explicitly to each operation instead.

### Never pass `nil`

Use `context.Background()` when no more appropriate context exists.

### Always call `cancel`

```go
ctx, cancel := context.WithTimeout(parent, 2*time.Second)
defer cancel()
```

### Propagate the existing context

Do not replace a request context with `context.Background()` inside a service or repository. Doing so disconnects downstream operations from cancellation.

---

## 14. What Is Graceful Shutdown?

A server might receive a termination signal because:

- You pressed `Ctrl+C`.
- Kubernetes is replacing a pod.
- A deployment is occurring.
- The operating system is stopping the process.

An immediate shutdown can break client connections, interrupt database operations, lose messages, stop jobs halfway, and prevent logs from being flushed.

Graceful shutdown means:

1. Receive the shutdown signal.
2. Stop accepting new work.
3. Allow current work to finish.
4. Stop background workers.
5. Close resources.
6. Exit the process.
7. Force an exit if shutdown takes too long.

```text
Shutdown signal
      ↓
Stop accepting new requests
      ↓
Wait for active work
      ↓
Close resources
      ↓
Exit
```

## 15. Basic HTTP Server Without Graceful Shutdown

```go
func main() {
	http.ListenAndServe(":8080", nil)
}
```

Instead, create an explicit server so you can call `Shutdown`:

```go
server := &http.Server{
	Addr:    ":8080",
	Handler: router,
}
```

## 16. Graceful Shutdown Implementation

```go
package main

import (
	"context"
	"errors"
	"log"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("healthy"))
	})

	server := &http.Server{
		Addr:              ":8080",
		Handler:           mux,
		ReadHeaderTimeout: 5 * time.Second,
		ReadTimeout:       10 * time.Second,
		WriteTimeout:      15 * time.Second,
		IdleTimeout:       60 * time.Second,
	}

	serverErrors := make(chan error, 1)
	go func() {
		log.Println("server listening on :8080")
		err := server.ListenAndServe()
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			serverErrors <- err
			return
		}
		serverErrors <- nil
	}()

	shutdownSignal := make(chan os.Signal, 1)
	signal.Notify(shutdownSignal, os.Interrupt, syscall.SIGTERM)

	select {
	case sig := <-shutdownSignal:
		log.Printf("received shutdown signal: %s", sig)
	case err := <-serverErrors:
		if err != nil {
			log.Fatalf("server failed: %v", err)
		}
		return
	}

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	log.Println("starting graceful shutdown")
	if err := server.Shutdown(shutdownCtx); err != nil {
		log.Printf("graceful shutdown failed: %v", err)
		if closeErr := server.Close(); closeErr != nil {
			log.Printf("forced close failed: %v", closeErr)
		}
	}

	log.Println("server stopped")
}
```

## 17. Understanding `server.Shutdown`

When `server.Shutdown(ctx)` is called, the HTTP server:

1. Closes its listeners.
2. Stops accepting new connections.
3. Closes idle connections.
4. Waits for active HTTP handlers to return.
5. Stops waiting when handlers finish or the context expires.

The timeout prevents a buggy or stuck handler from blocking shutdown forever.

## 18. `Shutdown` vs `Close`

### `server.Shutdown(ctx)`

- Gracefully stops accepting requests.
- Waits for active requests.
- Respects the supplied deadline.

### `server.Close()`

- Immediately closes active connections.
- Does not wait for requests to finish.

Common strategy:

```go
if err := server.Shutdown(shutdownCtx); err != nil {
	server.Close()
}
```

## 19. Using `signal.NotifyContext`

Go can convert operating-system signals directly into context cancellation:

```go
appCtx, stop := signal.NotifyContext(
	context.Background(),
	os.Interrupt,
	syscall.SIGTERM,
)
defer stop()

<-appCtx.Done()
```

When the program receives `SIGINT` or `SIGTERM`, `appCtx.Done()` closes.

## 20. Why Shutdown Needs a Fresh Context

After the signal context has been cancelled, do not derive the shutdown context from it:

```go
// Incorrect: appCtx is already cancelled.
shutdownCtx, cancel := context.WithTimeout(appCtx, 10*time.Second)
```

The child would also be cancelled immediately. Use a fresh root context:

```go
shutdownCtx, cancel := context.WithTimeout(
	context.Background(),
	10*time.Second,
)
defer cancel()
```

The responsibilities are different:

- The signal context tells the application **when to begin shutting down**.
- The shutdown context limits **how long cleanup may take**.

## 21. Gracefully Stopping Background Workers

`server.Shutdown` handles HTTP connections only. It does not stop custom goroutines, queue consumers, schedulers, or polling loops.

Use an application context and a `sync.WaitGroup`:

```go
func startWorker(ctx context.Context, wg *sync.WaitGroup) {
	defer wg.Done()

	for {
		select {
		case <-ctx.Done():
			log.Println("worker stopped")
			return
		case <-time.After(2 * time.Second):
			log.Println("processing background work")
		}
	}
}
```

Start and stop it:

```go
appCtx, cancelApp := context.WithCancel(context.Background())

var wg sync.WaitGroup
wg.Add(1)
go startWorker(appCtx, &wg)

// During shutdown:
cancelApp()
wg.Wait()
```

## 22. Recommended Shutdown Order

A typical backend shutdown flow is:

```text
1. Receive SIGTERM or SIGINT
2. Mark the application as not ready
3. Stop receiving new traffic
4. Stop producing new background work
5. Allow active requests and jobs to complete
6. Cancel work that exceeds the deadline
7. Close database, Redis, and message-broker clients
8. Flush logs, metrics, and traces
9. Exit
```

For Kubernetes applications, readiness is especially important:

```text
Kubernetes sends SIGTERM
        ↓
Application becomes unready
        ↓
Load balancer stops sending new traffic
        ↓
Existing requests finish
        ↓
Application exits
```

## 23. Common Mistakes

### Treating `http.ErrServerClosed` as a failure

`ListenAndServe` returns `http.ErrServerClosed` during normal shutdown:

```go
if err != nil && !errors.Is(err, http.ErrServerClosed) {
	log.Fatal(err)
}
```

### Using a cancelled context for cleanup

Create the shutdown timeout from `context.Background()`, not the already-cancelled signal context.

### Forgetting background goroutines

`server.Shutdown` does not stop your custom workers. Cancel and wait for them separately.

### Waiting forever

Avoid:

```go
server.Shutdown(context.Background())
```

Use a timeout so shutdown is bounded.

### Closing shared resources too early

Avoid closing the database before active requests finish. Prefer:

```text
Stop accepting traffic
    ↓
Wait for active requests
    ↓
Close database
```

### Detaching request work accidentally

Starting request-related work with `context.Background()` disconnects it from request cancellation. Propagate `r.Context()` when the operation belongs to the request.

If work must continue after the response, submit it to a proper background-job system with its own lifecycle rather than attaching it to the request.

## 24. Context Propagation vs Graceful Shutdown

| Concept | Primary purpose |
|---|---|
| Context propagation | Controls the lifetime of an individual operation |
| Graceful shutdown | Controls the lifetime of the entire application |
| Request context | Cancelled when the client disconnects or the request ends |
| Application context | Cancelled when the application begins shutting down |
| Shutdown context | Gives cleanup operations limited time to finish |

They work together:

```text
Operating-system signal
          ↓
Application begins shutdown
          ↓
HTTP server stops accepting requests
          ↓
Active requests use propagated contexts
          ↓
Workers receive cancellation
          ↓
Application waits within shutdown deadline
          ↓
Resources close and process exits
```

## 25. Simple Mental Model

Context answers:

> Should this operation still continue?

Examples:

- Did the client disconnect?
- Did the request time out?
- Is the application shutting down?
- Did the caller cancel the operation?

Graceful shutdown answers:

> How can the application stop without suddenly abandoning important work?

Context propagation allows each operation to hear cancellation. Graceful shutdown coordinates the cancellation and cleanup of the entire application.

The central context pattern is:

```go
func DoSomething(ctx context.Context) error {
	select {
	case <-ctx.Done():
		return ctx.Err()
	case result := <-performWork():
		return handle(result)
	}
}
```

The central shutdown pattern is:

```go
receiveSignal()
stopAcceptingNewWork()
waitForCurrentWorkWithTimeout()
closeResources()
exit()
```
