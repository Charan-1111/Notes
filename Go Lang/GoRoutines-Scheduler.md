# Goroutines and the Go Scheduler

Goroutines allow a Go application to perform multiple tasks concurrently without manually creating and managing operating-system threads.

A simple definition is:

> A goroutine is a lightweight task managed by the Go runtime.

The **Go scheduler** decides which goroutine executes, when it executes and which operating-system thread executes it.

---

## 1. Normal Function Execution

Consider a normal function call:

```go
package main

import (
	"fmt"
	"time"
)

func downloadFile() {
	fmt.Println("Downloading file...")
	time.Sleep(2 * time.Second)
	fmt.Println("Download completed")
}

func main() {
	downloadFile()
	fmt.Println("Main function completed")
}
```

The execution is sequential:

```text
Downloading file...
(wait for 2 seconds)
Download completed
Main function completed
```

The `main()` function waits for `downloadFile()` to finish before continuing.

---

## 2. Starting a Goroutine

Add the `go` keyword before a function call to run it as a goroutine:

```go
package main

import (
	"fmt"
	"time"
)

func downloadFile() {
	fmt.Println("Downloading file...")
	time.Sleep(2 * time.Second)
	fmt.Println("Download completed")
}

func main() {
	go downloadFile()

	fmt.Println("Main function completed")
	time.Sleep(3 * time.Second)
}
```

Possible output:

```text
Main function completed
Downloading file...
Download completed
```

The exact order is not guaranteed because the Go scheduler decides which goroutine runs first.

---

## 3. The Main Goroutine

The `main()` function itself runs inside a goroutine called the **main goroutine**:

```go
func main() {
	// This code is already running inside a goroutine.
}
```

When the main goroutine finishes, the entire program terminates—even if other goroutines are still running.

For example:

```go
func main() {
	go downloadFile()
}
```

The program may terminate before `downloadFile()` gets an opportunity to execute.

Using `time.Sleep()` to wait for goroutines is unreliable. Proper synchronization mechanisms such as `sync.WaitGroup` or channels should be used.

---

## 4. Anonymous Goroutines

An anonymous function can also be started as a goroutine:

```go
go func() {
	fmt.Println("Running inside a goroutine")
}()
```

Arguments can be passed to it:

```go
go func(id int) {
	fmt.Println("Worker:", id)
}(1)
```

---

## 5. Goroutines vs OS Threads

Goroutines are not operating-system threads.

They are lightweight tasks managed by the Go runtime and scheduled onto a smaller number of OS threads.

| Goroutine | OS thread |
|---|---|
| Managed by the Go runtime | Managed by the operating system |
| Starts with a small stack | Usually has a much larger stack |
| Cheap to create | More expensive to create |
| Can be created in large numbers | Large numbers consume significant resources |
| Scheduled by the Go scheduler | Scheduled by the OS scheduler |

It is practical to create thousands of goroutines:

```go
for i := 1; i <= 10_000; i++ {
	go func(id int) {
		fmt.Println("Worker:", id)
	}(i)
}
```

However, goroutines are lightweight—not free. Each goroutine still consumes memory and runtime resources.

---

## 6. Concurrency vs Parallelism

### Concurrency

Concurrency means multiple tasks make progress during the same period.

A single CPU core might switch between tasks:

```text
Task A → Task B → Task C → Task A
```

Only one task may execute at an exact moment, but all tasks are in progress.

### Parallelism

Parallelism means multiple tasks execute at exactly the same time:

```text
CPU Core 1 → Task A
CPU Core 2 → Task B
CPU Core 3 → Task C
```

Goroutines provide a concurrency model. They may run in parallel when multiple CPU cores and scheduler processors are available.

---

## 7. Why Go Needs a Scheduler

Imagine that an application has:

- 100,000 goroutines
- 8 CPU cores
- A smaller number of OS threads

All 100,000 goroutines cannot execute simultaneously.

The scheduler must decide:

- Which goroutine should run?
- On which OS thread should it run?
- What should happen when a goroutine blocks?
- How should work be distributed between CPU cores?
- When should a running goroutine be paused?

The Go runtime scheduler handles these decisions automatically.

---

## 8. The G-M-P Scheduler Model

The Go scheduler is usually explained using three components:

- **G — Goroutine**
- **M — Machine**
- **P — Processor**

The basic relationship is:

```text
Goroutine (G)
    ↓ scheduled through
Processor (P)
    ↓ attached to
Machine/OS Thread (M)
    ↓ executes on
CPU Core
```

### G: Goroutine

A `G` represents a goroutine.

For example:

```go
go processOrder()
```

The Go runtime creates a `G` containing information such as:

- The goroutine’s stack
- Its current state
- The next instruction to execute
- Scheduling information

A goroutine can be:

- Runnable
- Running
- Waiting
- Blocked in a system call
- Completed

### M: Machine

An `M` represents an operating-system thread.

The operating system schedules this thread on a physical CPU core.

An `M` normally needs a `P` to execute Go code.

### P: Processor

A `P` is a logical scheduler resource. It is not a physical CPU.

It contains the resources required to execute Go code, including a local queue of runnable goroutines.

The number of `P`s is controlled by `GOMAXPROCS`.

---

## 9. How G, M and P Work Together

Assume:

```text
GOMAXPROCS = 2
```

The runtime can have two logical processors:

```text
P1 local queue → G1, G2, G3
P2 local queue → G4, G5
```

Each processor can be attached to an OS thread:

```text
M1 + P1 → Executes G1
M2 + P2 → Executes G4
```

When `G1` stops running, `M1` can select another goroutine from `P1`:

```text
M1 + P1 → Executes G2
```

---

## 10. Local and Global Run Queues

The scheduler stores runnable goroutines in queues.

### Local Run Queue

Each `P` has its own local run queue:

```text
P1 → G1, G2, G3
P2 → G4, G5
```

Local queues reduce lock contention because every processor does not need to access one shared queue continuously.

### Global Run Queue

The scheduler also maintains a global queue:

```text
Global queue → G6, G7
```

The scheduler periodically checks the global queue so that its goroutines get an opportunity to execute.

---

## 11. Work Stealing

Suppose `P1` has many runnable goroutines:

```text
P1 → G1, G2, G3, G4, G5, G6
```

But `P2` has no work:

```text
P2 → Empty
```

The scheduler can move some goroutines from `P1` to `P2`:

```text
P1 → G1, G2, G3
P2 → G4, G5, G6
```

This mechanism is called **work stealing**.

It helps distribute work and use available CPU resources efficiently.

---

## 12. What Happens When a Goroutine Blocks?

A goroutine may block while waiting for:

- A channel operation
- A mutex
- A timer
- Network I/O
- A system call
- A `select` case

Consider a channel operation:

```go
package main

import "fmt"

func main() {
	ch := make(chan string)

	go func() {
		message := <-ch
		fmt.Println(message)
	}()

	ch <- "Hello from main"
}
```

The simplified flow is:

1. The receiver executes `<-ch`.
2. If no value is available, the goroutine enters a waiting state.
3. Its OS thread can execute another runnable goroutine.
4. The main goroutine sends a value.
5. The waiting goroutine becomes runnable.
6. The scheduler eventually executes it again.

---

## 13. Network I/O and the Network Poller

Backend applications spend significant time waiting for:

- HTTP requests
- Database responses
- Redis operations
- TCP connections
- Downstream services

Go uses a **network poller** integrated with the scheduler.

Consider:

```go
resp, err := http.Get("https://example.com")
```

While the goroutine waits for the network:

1. The runtime puts the goroutine into a waiting state.
2. The OS thread can execute another goroutine.
3. The network poller monitors the network operation.
4. When the response is ready, the goroutine becomes runnable again.

This allows Go servers to handle many concurrent network connections efficiently.

---

## 14. Blocking System Calls

Some system calls block the underlying OS thread.

Suppose `G1` is running on `M1` with `P1`:

```text
G1 → M1 + P1
```

If `G1` enters a blocking system call:

```text
M1 → Blocked with G1
P1 → Detached
```

The runtime can attach `P1` to another OS thread:

```text
M2 + P1 → Executes other goroutines
```

When the system call finishes, `G1` can become runnable and continue later.

---

## 15. Scheduler Preemption

Consider a long-running CPU loop:

```go
func calculateForever() {
	for {
		// CPU-intensive work
	}
}
```

The scheduler should not allow this goroutine to prevent every other goroutine from running.

Go supports **preemption**. The runtime can pause a running goroutine and allow another goroutine to execute:

```text
G1 runs
→ G1 is paused
→ G2 runs
→ G3 runs
→ G1 resumes later
```

You should still avoid uncontrolled busy loops because they waste CPU.

When waiting for an event, use:

- Channels
- Timers
- Context cancellation
- Synchronization primitives

---

## 16. Waiting with `sync.WaitGroup`

Do not use arbitrary sleeps to wait for goroutines.

### Incorrect approach

```go
go doWork()
time.Sleep(2 * time.Second)
```

### Correct approach

```go
package main

import (
	"fmt"
	"sync"
)

func worker(id int, wg *sync.WaitGroup) {
	defer wg.Done()

	fmt.Printf("Worker %d started\n", id)
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

How it works:

1. `wg.Add(1)` registers a task.
2. `go worker(...)` starts the task.
3. `wg.Done()` marks the task as completed.
4. `wg.Wait()` waits until the counter becomes zero.

Call `Add()` before starting the goroutine.

---

## 17. Communicating Through Channels

Channels allow goroutines to send and receive values safely.

```go
package main

import "fmt"

func calculate(a, b int, result chan int) {
	result <- a + b
}

func main() {
	result := make(chan int)

	go calculate(10, 20, result)

	answer := <-result
	fmt.Println(answer)
}
```

Output:

```text
30
```

A channel provides:

- Data transfer
- Synchronization between goroutines

---

## 18. Unbuffered Channels

Create an unbuffered channel using:

```go
ch := make(chan int)
```

A send waits until another goroutine is ready to receive:

```go
ch <- 10
```

Similarly, a receive waits until a sender is available:

```go
value := <-ch
```

The sender and receiver synchronize directly.

---

## 19. Buffered Channels

Create a buffered channel by specifying its capacity:

```go
ch := make(chan int, 3)
```

It can temporarily hold three values:

```go
ch <- 10
ch <- 20
ch <- 30
```

A send blocks when the buffer is full.

A receive blocks when the buffer is empty.

Buffered channels are useful for:

- Bounded job queues
- Limiting concurrency
- Handling small differences between producer and consumer speeds

---

## 20. Concurrent API Calls

Suppose an API needs data from three independent services.

### Sequential execution

```go
user := fetchUser()
orders := fetchOrders()
recommendations := fetchRecommendations()
```

If every call takes one second, the total duration may be approximately three seconds.

### Concurrent execution

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func fetch(name string, wg *sync.WaitGroup) {
	defer wg.Done()

	time.Sleep(time.Second)
	fmt.Println(name, "completed")
}

func main() {
	var wg sync.WaitGroup

	services := []string{
		"user service",
		"order service",
		"recommendation service",
	}

	for _, service := range services {
		wg.Add(1)
		go fetch(service, &wg)
	}

	wg.Wait()
	fmt.Println("All data fetched")
}
```

Because the operations are independent and execute concurrently, the total duration may be closer to one second.

---

## 21. CPU-Bound Work Behaves Differently

CPU-bound work includes operations such as:

- Image processing
- Video encoding
- Data compression
- Encryption
- Complex calculations
- Large data transformations

With one available `P`, goroutines can execute concurrently, but only one goroutine can execute Go code at a particular instant.

With multiple `P`s and CPU cores, CPU-bound goroutines may execute in parallel.

You can inspect the scheduler configuration:

```go
package main

import (
	"fmt"
	"runtime"
)

func main() {
	fmt.Println("Logical CPUs:", runtime.NumCPU())
	fmt.Println("GOMAXPROCS:", runtime.GOMAXPROCS(0))
}
```

`runtime.GOMAXPROCS(0)` returns the current value without changing it.

You can change it:

```go
runtime.GOMAXPROCS(2)
```

In most applications, leave the runtime default unchanged unless performance measurements provide a reason to tune it.

---

## 22. Worker Pool Example

Creating one goroutine for every item can be dangerous when the number of items is unbounded:

```go
for _, job := range millionsOfJobs {
	go process(job)
}
```

This can:

- Consume excessive memory
- Overload a database
- Overload downstream services
- Exhaust connection pools
- Create too much scheduling work

A worker pool limits concurrency:

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func worker(id int, jobs <-chan int, wg *sync.WaitGroup) {
	defer wg.Done()

	for job := range jobs {
		fmt.Printf("Worker %d processing job %d\n", id, job)
		time.Sleep(500 * time.Millisecond)
	}
}

func main() {
	const workerCount = 3

	jobs := make(chan int)
	var wg sync.WaitGroup

	for id := 1; id <= workerCount; id++ {
		wg.Add(1)
		go worker(id, jobs, &wg)
	}

	for job := 1; job <= 10; job++ {
		jobs <- job
	}

	close(jobs)
	wg.Wait()

	fmt.Println("All jobs completed")
}
```

Only three jobs are processed concurrently because there are three workers.

---

## 23. Goroutines in an HTTP Server

Go’s `net/http` package handles incoming requests concurrently using goroutines internally.

```go
package main

import (
	"fmt"
	"net/http"
	"time"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Println("Processing:", r.URL.Path)

	time.Sleep(time.Second)

	fmt.Fprintln(w, "Request completed")
}

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":8080", nil)
}
```

Multiple users can send requests, and their handlers can make progress concurrently.

Shared data used by handlers must be synchronized correctly.

---

## 24. Race Conditions

A race condition can occur when multiple goroutines access shared data concurrently and at least one modifies it.

### Incorrect example

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
	fmt.Println(counter)
}
```

`counter++` is conceptually:

1. Read the current value.
2. Add one.
3. Write the new value.

Multiple goroutines can interfere with one another.

### Protecting data with a mutex

```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	counter := 0

	var wg sync.WaitGroup
	var mu sync.Mutex

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
	fmt.Println(counter)
}
```

Detect race conditions using:

```bash
go test -race ./...
```

Or:

```bash
go run -race main.go
```

---

## 25. Goroutine Leaks

A goroutine leak occurs when a goroutine remains blocked forever and cannot perform useful work.

```go
func process() {
	ch := make(chan int)

	go func() {
		value := <-ch
		fmt.Println(value)
	}()
}
```

Nothing sends a value to `ch`, so the goroutine waits forever.

Repeated leaks in a backend service can increase:

- Memory usage
- Goroutine count
- Resource consumption
- System instability

---

## 26. Preventing Leaks with Context

Use context cancellation to tell goroutines when to stop:

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func worker(ctx context.Context) {
	select {
	case <-time.After(5 * time.Second):
		fmt.Println("Work completed")

	case <-ctx.Done():
		fmt.Println("Work cancelled:", ctx.Err())
	}
}

func main() {
	ctx, cancel := context.WithTimeout(
		context.Background(),
		2*time.Second,
	)
	defer cancel()

	worker(ctx)
}
```

After two seconds, the context is cancelled, allowing the worker to stop.

In HTTP handlers, request-related goroutines should usually observe `r.Context()` or a derived context.

---

## 27. Loop Variables and Goroutines

Pass loop values explicitly when starting goroutines:

```go
for i := 1; i <= 3; i++ {
	go func(id int) {
		fmt.Println(id)
	}(i)
}
```

Each function receives its own value through the `id` parameter.

This avoids accidental sharing and remains clear when the code is refactored.

---

## 28. Goroutine Lifecycle

A simplified goroutine lifecycle is:

```text
Created
   ↓
Runnable
   ↓
Running
   ↓
Waiting / Runnable / Completed
```

### Runnable

The goroutine is ready to execute but is waiting for scheduler time.

### Running

The goroutine is currently executing on an `M` attached to a `P`.

### Waiting

The goroutine is waiting for something, such as:

- A channel value
- A mutex
- A timer
- A network response

### Completed

The function returns, and the runtime eventually reclaims the goroutine’s resources.

---

## 29. Complete Scheduler Flow Example

Consider:

```go
func main() {
	ch := make(chan string)

	go func() {
		ch <- "done"
	}()

	message := <-ch
	fmt.Println(message)
}
```

A simplified execution flow is:

1. The runtime starts the main goroutine.
2. `main` creates a channel.
3. The `go` statement creates another goroutine.
4. The new goroutine becomes runnable.
5. `main` tries to receive from the channel.
6. If no value is ready, `main` enters a waiting state.
7. The scheduler runs the new goroutine.
8. The new goroutine sends `"done"`.
9. The main goroutine becomes runnable.
10. The scheduler runs the main goroutine again.
11. `main` receives and prints the value.
12. The program terminates.

The exact execution order is controlled by the scheduler, not by the order in which goroutines are created.

---

## 30. `runtime.Gosched()`

A goroutine can voluntarily yield its execution:

```go
runtime.Gosched()
```

Example:

```go
package main

import (
	"fmt"
	"runtime"
	"sync"
)

func main() {
	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()

		for i := 0; i < 3; i++ {
			fmt.Println("A:", i)
			runtime.Gosched()
		}
	}()

	go func() {
		defer wg.Done()

		for i := 0; i < 3; i++ {
			fmt.Println("B:", i)
			runtime.Gosched()
		}
	}()

	wg.Wait()
}
```

Most application code does not need to call `runtime.Gosched()` directly.

Channels, mutexes, I/O operations and runtime preemption already give the scheduler opportunities to run other goroutines.

---

## 31. Goroutines Do Not Always Make Code Faster

Goroutines introduce costs:

- Scheduling overhead
- Synchronization overhead
- Additional memory
- Race-condition risk
- Debugging complexity
- Downstream-service pressure

For a tiny operation, this:

```go
go add(1, 2)
```

may be slower and more complicated than:

```go
add(1, 2)
```

Use goroutines when tasks are meaningfully independent or spend time waiting.

---

## 32. Practical Backend Guidelines

### Use goroutines for independent work

Good use cases include:

- Calling independent downstream APIs
- Processing background jobs
- Consuming messages
- Handling network connections
- Performing independent I/O
- Running periodic cleanup tasks

### Always define how a goroutine stops

```go
func worker(ctx context.Context, jobs <-chan Job) {
	for {
		select {
		case <-ctx.Done():
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

### Limit concurrency

Use mechanisms such as:

- Worker pools
- Buffered channels as semaphores
- Rate limiters
- Connection-pool limits
- `errgroup` concurrency limits

### Protect shared memory

Use:

- `sync.Mutex`
- `sync.RWMutex`
- Atomic operations
- Channels
- Ownership-based designs

### Do not assume execution order

This code does not guarantee that `first()` runs before `second()`:

```go
go first()
go second()
```

If order matters, express it using synchronization.

### Do not ignore errors

A fire-and-forget goroutine can silently lose errors:

```go
go saveAuditLog()
```

Use an error channel, `errgroup`, logging or a durable background-job system, depending on the importance of the work.

---

## 33. Final Mental Model

When you write:

```go
go task()
```

Think:

> I am creating a lightweight runnable task. The Go runtime will place it in a scheduling queue. A logical processor will eventually execute it on an OS thread. If it blocks, the runtime will try to use the available resources to execute other runnable goroutines.

The complete relationship is:

```text
Application
    ↓ creates
Goroutines (G)
    ↓ scheduled through
Logical Processors (P)
    ↓ attached to
OS Threads (M)
    ↓ scheduled by the operating system on
CPU Cores
```

The Go scheduler is responsible for:

- Scheduling runnable goroutines
- Parking waiting goroutines
- Waking goroutines when events occur
- Balancing work using local and global queues
- Stealing work between processors
- Cooperating with the network poller
- Handling blocking system calls
- Preempting long-running goroutines
- Using available CPU parallelism

The most important practical lesson is:

> Creating a goroutine is easy. Managing its lifetime, synchronization, errors and resource limits is the real engineering work.
