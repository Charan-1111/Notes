# Go Synchronization: `Mutex`, `RWMutex`, `WaitGroup`, and `Once`

When several goroutines run concurrently, they may access the same data at the same time. Without coordination, this can produce incorrect results, data races, deadlocks, and unpredictable bugs.

Go's `sync` package provides synchronization tools:

| Tool | Purpose |
|---|---|
| `sync.Mutex` | Allows only one goroutine at a time to access protected data |
| `sync.RWMutex` | Allows multiple readers or one writer |
| `sync.WaitGroup` | Waits for a collection of goroutines to finish |
| `sync.Once` | Runs an operation exactly once |

---

## 1. Why synchronization is needed

Consider this shared counter:

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

You may expect `1000`, but sometimes get a smaller value. This happens because `counter++` is conceptually three operations:

1. Read the current value.
2. Add one.
3. Write the new value.

Two goroutines can overlap:

```text
Counter starts at 10

Goroutine A reads 10
Goroutine B reads 10
Goroutine A writes 11
Goroutine B writes 11
```

The correct result should be `12`, but it becomes `11`. This is a **race condition**.

A **data race** occurs when multiple goroutines access the same memory concurrently, at least one access is a write, and the accesses are not synchronized.

Use Go's race detector:

```bash
go run -race main.go
go test -race ./...
```

---

## 2. `sync.Mutex`

`Mutex` means **mutual exclusion**. It allows only one goroutine at a time to execute a protected section of code.

Imagine a room with one key. A goroutine takes the key, enters the room, completes its work, and returns the key. Other goroutines must wait for it.

```go
var mu sync.Mutex

mu.Lock()
// Access shared data.
mu.Unlock()
```

The code between `Lock()` and `Unlock()` is called the **critical section**.

### Safe counter example

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

Now only one goroutine can execute `counter++` at a time, so the result is `1000`.

### Prefer `defer` when practical

```go
func increment() {
	mu.Lock()
	defer mu.Unlock()

	counter++
}
```

`defer` makes sure the mutex is released even when the function returns early:

```go
func updateUser(id string) error {
	mu.Lock()
	defer mu.Unlock()

	if id == "" {
		return errors.New("empty user ID")
	}

	// Update shared data.
	return nil
}
```

### Thread-safe in-memory store

```go
package main

import (
	"fmt"
	"sync"
)

type UserStore struct {
	mu    sync.Mutex
	users map[string]string
}

func NewUserStore() *UserStore {
	return &UserStore{users: make(map[string]string)}
}

func (s *UserStore) Set(id, name string) {
	s.mu.Lock()
	defer s.mu.Unlock()

	s.users[id] = name
}

func (s *UserStore) Get(id string) (string, bool) {
	s.mu.Lock()
	defer s.mu.Unlock()

	name, exists := s.users[id]
	return name, exists
}

func main() {
	store := NewUserStore()
	var wg sync.WaitGroup

	for i := 1; i <= 5; i++ {
		wg.Add(1)

		go func(id int) {
			defer wg.Done()
			key := fmt.Sprintf("user-%d", id)
			store.Set(key, fmt.Sprintf("User %d", id))
		}(i)
	}

	wg.Wait()
	name, exists := store.Get("user-3")
	fmt.Println(name, exists)
}
```

Go maps are not safe when reads and writes happen concurrently, so both `Get` and `Set` must use the same lock.

### Keep critical sections small

Avoid holding a lock while making a slow network or database call:

```go
// Poor: the lock is held during the network call.
mu.Lock()
defer mu.Unlock()

response, err := callExternalAPI()
if err != nil {
	return err
}

sharedData = response
```

Do the independent work first and lock only for the shared update:

```go
response, err := callExternalAPI()
if err != nil {
	return err
}

mu.Lock()
sharedData = response
mu.Unlock()
```

### Common `Mutex` mistakes

#### Forgetting to unlock

```go
mu.Lock()
counter++
// Missing mu.Unlock()
```

Waiting goroutines may remain blocked forever.

#### Locking the same mutex twice

```go
mu.Lock()
mu.Lock() // Deadlock
```

Go mutexes are not reentrant. The second call waits for a lock that the same goroutine already holds.

#### Copying a mutex

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

// Bad: value receiver copies Counter and its mutex.
func (c Counter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

Use a pointer receiver:

```go
func (c *Counter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

#### Creating a different mutex for each call

```go
func increment(counter *int) {
	var mu sync.Mutex
	mu.Lock()
	defer mu.Unlock()

	*counter = *counter + 1
}
```

Each call has its own mutex, so goroutines are not coordinated. Every goroutine accessing the data must share the same mutex.

---

## 3. `sync.RWMutex`

`RWMutex` means **reader/writer mutex**. It supports two kinds of locks:

```go
mu.RLock()   // Acquire read lock
mu.RUnlock() // Release read lock

mu.Lock()    // Acquire write lock
mu.Unlock()  // Release write lock
```

Its rules are:

- Multiple readers may access the data simultaneously.
- Only one writer may access it at a time.
- Readers cannot access it while a writer holds the lock.
- A writer cannot access it while readers hold the lock.

```text
Reader + Reader = Allowed
Writer + Writer = Not allowed
Reader + Writer = Not allowed
```

It is useful for read-heavy data such as caches, configuration, feature flags, or service registries.

### Thread-safe cache

```go
package main

import (
	"fmt"
	"sync"
)

type Cache struct {
	mu   sync.RWMutex
	data map[string]string
}

func NewCache() *Cache {
	return &Cache{data: make(map[string]string)}
}

func (c *Cache) Set(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.data[key] = value
}

func (c *Cache) Get(key string) (string, bool) {
	c.mu.RLock()
	defer c.mu.RUnlock()

	value, exists := c.data[key]
	return value, exists
}

func main() {
	cache := NewCache()
	cache.Set("language", "Go")

	value, exists := cache.Get("language")
	fmt.Println(value, exists)
}
```

`Set` changes the data, so it uses the exclusive write lock. `Get` only reads, so it uses a read lock. Multiple goroutines can therefore call `Get` concurrently.

### Use the write lock for every mutation

Incorrect:

```go
func (c *Cache) Set(key, value string) {
	c.mu.RLock()
	defer c.mu.RUnlock()
	c.data[key] = value
}
```

Correct:

```go
func (c *Cache) Set(key, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.data[key] = value
}
```

### Do not upgrade a read lock directly

This can deadlock:

```go
mu.RLock()
defer mu.RUnlock()

mu.Lock() // Waits for all readers, including this goroutine
```

Release the read lock before requesting a write lock, then check the condition again because another goroutine may have changed the data in between:

```go
mu.RLock()
_, exists := data[key]
mu.RUnlock()

if !exists {
	mu.Lock()

	if _, stillMissing := data[key]; stillMissing {
		data[key] = "default"
	}

	mu.Unlock()
}
```

### `Mutex` or `RWMutex`?

Start with `Mutex` when unsure. Choose `RWMutex` when reads greatly outnumber writes and concurrent reading provides a meaningful benefit. `RWMutex` has extra bookkeeping and is not automatically faster.

---

## 4. `sync.WaitGroup`

A `WaitGroup` lets one goroutine wait until a collection of other goroutines finishes. Think of it as a task counter:

```text
Add(3) -> 3 tasks remain
Done() -> 2 tasks remain
Done() -> 1 task remains
Done() -> 0 tasks remain
Wait() -> Continue
```

Its primary methods are:

```go
wg.Add(numberOfTasks)
wg.Done()
wg.Wait()
```

### Basic worker example

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Printf("Worker %d started\n", id)
	time.Sleep(time.Second)
	fmt.Printf("Worker %d completed\n", id)
}

func main() {
	var wg sync.WaitGroup

	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go worker(i, &wg)
	}

	wg.Wait()
	fmt.Println("All workers completed")
}
```

- `Add(1)` records one new task.
- `Done()` subtracts one when the task finishes.
- `Wait()` blocks until the counter reaches zero.

### Always call `Add` before starting the goroutine

Correct:

```go
wg.Add(1)
go func() {
	defer wg.Done()
}()
```

Avoid:

```go
go func() {
	wg.Add(1)
	defer wg.Done()
}()
```

If `Add` is inside the goroutine, `Wait` may run before the counter is increased.

### Fetching independent data concurrently

```go
func fetchUser(wg *sync.WaitGroup) {
	defer wg.Done()
	time.Sleep(500 * time.Millisecond)
	fmt.Println("User fetched")
}

func fetchOrders(wg *sync.WaitGroup) {
	defer wg.Done()
	time.Sleep(800 * time.Millisecond)
	fmt.Println("Orders fetched")
}

func fetchRecommendations(wg *sync.WaitGroup) {
	defer wg.Done()
	time.Sleep(300 * time.Millisecond)
	fmt.Println("Recommendations fetched")
}

func main() {
	var wg sync.WaitGroup
	wg.Add(3)

	go fetchUser(&wg)
	go fetchOrders(&wg)
	go fetchRecommendations(&wg)

	wg.Wait()
	fmt.Println("All data is ready")
}
```

### `WaitGroup` does not protect shared data

This is unsafe because multiple goroutines append to the same slice:

```go
var wg sync.WaitGroup
results := make([]string, 0)

for i := 0; i < 10; i++ {
	wg.Add(1)

	go func(id int) {
		defer wg.Done()
		results = append(results, fmt.Sprintf("result-%d", id))
	}(i)
}

wg.Wait()
```

Use a mutex as well:

```go
var wg sync.WaitGroup
var mu sync.Mutex
results := make([]string, 0)

for i := 0; i < 10; i++ {
	wg.Add(1)

	go func(id int) {
		defer wg.Done()
		result := fmt.Sprintf("result-%d", id)

		mu.Lock()
		results = append(results, result)
		mu.Unlock()
	}(i)
}

wg.Wait()
```

Here, `WaitGroup` waits for completion while `Mutex` protects the slice.

### Common `WaitGroup` mistakes

- Calling `Done` more times than `Add` causes `panic: sync: negative WaitGroup counter`.
- Forgetting `Done` can make `Wait` block forever.
- Passing a `WaitGroup` by value creates a copy; pass `*sync.WaitGroup` instead.
- Reuse a `WaitGroup` only after the previous `Wait` has completed.

---

## 5. `sync.Once`

`sync.Once` ensures that a function runs exactly once, even when several goroutines call it concurrently.

```go
var once sync.Once

once.Do(func() {
	fmt.Println("Executed once")
})
```

### Concurrent example

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	var wg sync.WaitGroup

	initialize := func() {
		fmt.Println("Application initialized")
	}

	for i := 1; i <= 10; i++ {
		wg.Add(1)

		go func() {
			defer wg.Done()
			once.Do(initialize)
		}()
	}

	wg.Wait()
}
```

Although ten goroutines call `once.Do`, `initialize` runs only once. Other callers wait until the first execution finishes.

### Initialize configuration once

```go
type Config struct {
	Environment string
	Port        int
}

var (
	config     *Config
	configOnce sync.Once
)

func GetConfig() *Config {
	configOnce.Do(func() {
		fmt.Println("Loading configuration")
		config = &Config{
			Environment: "production",
			Port:        8080,
		}
	})

	return config
}
```

Every caller receives the same initialized configuration.

### Initialize a database handle once

```go
var (
	db     *sql.DB
	dbErr  error
	dbOnce sync.Once
)

func GetDB() (*sql.DB, error) {
	dbOnce.Do(func() {
		db, dbErr = sql.Open(
			"postgres",
			"postgres://user:password@localhost/app?sslmode=disable",
		)
	})

	return db, dbErr
}
```

In a real service, validate connectivity with `PingContext` and close the database during graceful shutdown.

### What if initialization fails?

`sync.Once` does not understand errors. It considers the operation complete as soon as the function returns.

Store the result if every caller should receive the same initialization error:

```go
var (
	once    sync.Once
	initErr error
)

func Initialize() error {
	once.Do(func() {
		initErr = performInitialization()
	})

	return initErr
}
```

If the first attempt fails, later calls do not retry. If retry behavior is required, `sync.Once` alone is not the correct tool.

If the function passed to `Do` panics, the `Once` is also considered used, so future calls do not run it again.

One `Once` instance means one execution total—not one execution per function:

```go
once.Do(functionA)
once.Do(functionB) // Does not run
```

Use separate `Once` values for separate initialization operations.

---

## 6. Combining the tools

```go
package main

import (
	"fmt"
	"sync"
)

type Service struct {
	initOnce sync.Once

	cacheMu sync.RWMutex
	cache   map[string]string

	workers sync.WaitGroup
}

func NewService() *Service {
	return &Service{}
}

func (s *Service) Initialize() {
	s.initOnce.Do(func() {
		fmt.Println("Initializing service")
		s.cache = make(map[string]string)
	})
}

func (s *Service) Set(key, value string) {
	s.cacheMu.Lock()
	defer s.cacheMu.Unlock()
	s.cache[key] = value
}

func (s *Service) Get(key string) (string, bool) {
	s.cacheMu.RLock()
	defer s.cacheMu.RUnlock()
	value, exists := s.cache[key]
	return value, exists
}

func (s *Service) StartWorker(id int) {
	s.workers.Add(1)

	go func() {
		defer s.workers.Done()
		key := fmt.Sprintf("worker-%d", id)
		s.Set(key, fmt.Sprintf("result-from-worker-%d", id))
	}()
}

func (s *Service) Wait() {
	s.workers.Wait()
}

func main() {
	service := NewService()

	service.Initialize()
	service.Initialize() // Initialization is not repeated.

	for i := 1; i <= 5; i++ {
		service.StartWorker(i)
	}

	service.Wait()

	value, exists := service.Get("worker-3")
	fmt.Println(value, exists)
}
```

Responsibilities in this example:

| Tool | Responsibility |
|---|---|
| `sync.Once` | Initializes the service once |
| `sync.RWMutex` | Protects the shared cache |
| `sync.WaitGroup` | Waits for workers to complete |

---

## 7. How to choose

### Use `Mutex` when

- Multiple goroutines access shared mutable data.
- At least one goroutine changes that data.
- You need simple exclusive access.

### Use `RWMutex` when

- Shared data is read far more frequently than it is written.
- Concurrent reads provide a real benefit.

### Use `WaitGroup` when

- You start several goroutines.
- You need to wait until all of them finish.

### Use `Once` when

- Initialization must occur exactly once.
- Multiple goroutines might request the initialization.

---

## 8. Important rules

1. Synchronize shared mutable data.
2. Every `Lock` must eventually have an `Unlock`.
3. Keep critical sections small.
4. Use the same lock for the same shared state.
5. Do not copy synchronization primitives after first use.
6. Remember that `WaitGroup` waits but does not protect data.
7. Call `WaitGroup.Add` before starting the goroutine.
8. Use `RLock` only for reading and `Lock` for modification.
9. Do not use `Once` when failed operations must be retried.
10. Test concurrent code with `go test -race ./...`.

---

## 9. Final mental model

```text
Mutex     -> One goroutine at a time
RWMutex   -> Many readers or one writer
WaitGroup -> Wait until all goroutines finish
Once      -> Run an operation exactly once
```

Or remember this real-world analogy:

```text
Mutex     -> One key for one room
RWMutex   -> Many readers allowed; a writer needs the room alone
WaitGroup -> Wait until every assigned worker reports completion
Once      -> Perform initialization only on the first request
```
