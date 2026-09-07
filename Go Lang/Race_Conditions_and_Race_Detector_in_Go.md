# Race Conditions and the Race Detector in Go

Go makes concurrency easy with goroutines, but multiple goroutines can accidentally access the same data at the same time. This can create a **race condition**.

---

## 1. What Is a Race Condition?

A race condition happens when a program's result depends on which goroutine runs first.

Imagine two cashiers updating the same account balance:

```text
Starting balance: 100

Cashier A reads 100 and adds 50  -> wants to save 150
Cashier B reads 100 and adds 20  -> wants to save 120

Final balance may become 120 instead of 170.
```

One update was lost because both cashiers worked with the old value.

## 2. What Is a Data Race?

A **data race** occurs when:

1. Two or more goroutines access the same memory at the same time.
2. At least one access is a write.
3. The accesses are not synchronized.

A data race is one type of race condition. Go's race detector finds data races, but it cannot find every logical concurrency bug.

---

## 3. A Simple Data Race

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()
			counter++
		}()
	}

	wg.Wait()
	fmt.Println("Counter:", counter)
}
```

You may expect `1000`, but the result can be smaller.

### Why isn't `counter++` safe?

It looks like one operation, but internally it is roughly:

```go
value := counter
value = value + 1
counter = value
```

Two goroutines can read the same old value and overwrite each other's update.

`WaitGroup` only waits for goroutines to finish. It does **not** protect `counter`.

---

## 4. Detecting the Race

Run the program with Go's race detector:

```bash
go run -race main.go
```

For tests:

```bash
go test -race ./...
```

For a compiled binary:

```bash
go build -race -o app .
./app
```

The output will contain information similar to:

```text
WARNING: DATA RACE
Read at ...
Previous write at ...
Goroutine ...
```

The report shows:

- Where one goroutine accessed the variable
- Where another goroutine accessed it
- Where those goroutines were created

Start by finding the two conflicting stack traces and the shared variable they access.

### Important limitation

The detector finds races that happen while the program is running. If a risky code path is never executed, its race will not be reported. Good tests and realistic workloads are still necessary.

The `-race` option also makes programs slower and use more memory, so it is commonly used during development, testing, and staging—not usually in every production instance.

---

## 5. Fix 1: Protect Shared Data with a Mutex

A mutex allows only one goroutine at a time into a protected section of code.

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0
	var mu sync.Mutex
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()

			mu.Lock()
			counter++
			mu.Unlock()
		}()
	}

	wg.Wait()
	fmt.Println("Counter:", counter)
}
```

Think of `mu.Lock()` as taking the only key to a room. Other goroutines must wait until `mu.Unlock()` returns the key.

A common pattern is:

```go
mu.Lock()
defer mu.Unlock()

// Read or modify protected data.
```

Use `defer` when it makes the critical section clear and ensures the mutex is unlocked on every return path.

### Keep the critical section small

Avoid slow operations while holding a lock:

```go
mu.Lock()
counter++
mu.Unlock()

// Slow network call happens after unlocking.
callAnotherService()
```

---

## 6. Mutexes Must Protect Reads Too

This code still has a race:

```go
mu.Lock()
counter++
mu.Unlock()

fmt.Println(counter) // Unprotected read
```

If another goroutine might write at the same time, the read must also be protected:

```go
mu.Lock()
value := counter
mu.Unlock()

fmt.Println(value)
```

The rule is simple: use the same synchronization strategy for **every** concurrent access to the shared variable.

---

## 7. Fix 2: Use `sync.RWMutex` for Read-Heavy Data

`RWMutex` supports:

- Multiple readers at the same time with `RLock()`
- Only one writer with `Lock()`

```go
type UserCache struct {
	mu    sync.RWMutex
	users map[string]string
}

func (c *UserCache) Get(id string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	name, ok := c.users[id]
	return name, ok
}

func (c *UserCache) Set(id, name string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.users[id] = name
}
```

A normal Go map is not safe for concurrent reads and writes. Protect it with synchronization or redesign ownership.

Use `RWMutex` only when concurrent reads provide a real benefit. A regular `Mutex` is often simpler and sufficient.

---

## 8. Fix 3: Use Atomic Operations for Simple Values

For simple counters or flags, `sync/atomic` can update a value as one indivisible operation.

```go
package main

import (
	"fmt"
	"sync"
	"sync/atomic"
)

func main() {
	var counter atomic.Int64
	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()
			counter.Add(1)
		}()
	}

	wg.Wait()
	fmt.Println("Counter:", counter.Load())
}
```

Atomics work well for independent counters and flags. Use a mutex when several fields must be changed together as one consistent operation.

---

## 9. Fix 4: Own the Data in One Goroutine

Instead of sharing a variable, one goroutine can own it. Other goroutines send messages through a channel.

```go
func main() {
	increments := make(chan int)
	result := make(chan int)

	go func() {
		counter := 0

		for value := range increments {
			counter += value
		}

		result <- counter
	}()

	var wg sync.WaitGroup

	for i := 0; i < 1000; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()
			increments <- 1
		}()
	}

	wg.Wait()
	close(increments)

	fmt.Println("Counter:", <-result)
}
```

Only the owner goroutine accesses `counter`, so no mutex is needed.

Channels are useful when the design is naturally message-based. Do not use them only to avoid learning mutexes; choose whichever makes ownership and behavior clearer.

---

## 10. Race in an HTTP Server

HTTP handlers run concurrently. This server contains a race:

```go
type Server struct {
	requests int
}

func (s *Server) handleRequest(w http.ResponseWriter, r *http.Request) {
	s.requests++
	fmt.Fprintf(w, "request number: %d", s.requests)
}
```

Several requests can update `s.requests` simultaneously.

One solution is an atomic counter:

```go
type Server struct {
	requests atomic.Int64
}

func (s *Server) handleRequest(w http.ResponseWriter, r *http.Request) {
	count := s.requests.Add(1)
	fmt.Fprintf(w, "request number: %d", count)
}
```

This is especially important for shared state in handlers, middleware, in-memory caches, and background workers.

---

## 11. A Logical Race the Detector May Not Find

Consider reserving the last product:

```go
if inventory.Load() > 0 {
	inventory.Add(-1)
}
```

Each atomic operation is individually safe, so the race detector may report nothing. But two goroutines can both observe inventory as `1`, then both subtract, producing `-1`.

This is a logical race because the **check and update together** must be one operation. A mutex can protect the whole rule:

```go
mu.Lock()
if inventory > 0 {
	inventory--
}
mu.Unlock()
```

Passing `go test -race` does not prove that all concurrency logic is correct.

---

## 12. Common Mistakes

### Copying a mutex

Do not copy a struct after it starts using a mutex. Prefer pointer receivers:

```go
type Store struct {
	mu   sync.Mutex
	data map[string]string
}

func (s *Store) Set(key, value string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.data[key] = value
}
```

### Protecting only the writer

Both reads and writes need synchronization when they can occur concurrently.

### Locking with different mutexes

All access to one shared resource must follow the same locking rule. Two different mutexes do not protect the resource from each other.

### Assuming timing fixes the problem

Adding `time.Sleep` may hide a race temporarily, but it does not synchronize goroutines.

### Forgetting that tests can run concurrently

Parallel tests, shared package variables, and reused test fixtures can race too. Run the entire test suite with `-race`.

---

## 13. Choosing a Solution

| Situation | Usually use |
|---|---|
| Protecting a struct or several related fields | `sync.Mutex` |
| Many reads and fewer writes | `sync.RWMutex` |
| Simple independent counter or flag | `sync/atomic` |
| Passing work or transferring ownership | Channel |
| Waiting for goroutines to finish | `sync.WaitGroup` |

`WaitGroup` coordinates completion; it does not protect shared data.

---

## 14. Practical Race-Detection Workflow

Run these commands regularly:

```bash
go test -race ./...
go vet ./...
```

When the race detector reports a problem:

1. Find the shared variable in the report.
2. Identify every goroutine that reads or writes it.
3. Decide who should own the data.
4. Protect all accesses with one clear strategy.
5. Run the same test again with `-race`.

For bugs that appear only under concurrency, repeat tests:

```bash
go test -race -count=20 ./...
```

The race detector needs the problematic operations to execute, so write tests that actually run work concurrently.

---

## Quick Summary

- A data race happens when goroutines access the same memory concurrently, at least one access is a write, and there is no synchronization.
- `counter++` is not automatically safe.
- Detect races with `go test -race ./...` or `go run -race main.go`.
- Fix shared access with a mutex, an atomic operation, or clear channel ownership.
- Protect reads as well as writes.
- The detector only finds races executed during that run.
- Race-free code can still contain logical concurrency bugs.

The main question to ask is: **who owns this data, and how are concurrent accesses coordinated?**
