# Channels and Channel Patterns in Go

A **channel** is a communication mechanism that allows goroutines to safely exchange values and coordinate their work.

A simple mental model is:

> A goroutine performs some work and sends the result through a channel.  
> Another goroutine waits for and receives that result.

Channels are not merely containers for data. They are also **synchronization points** between goroutines.

---

## 1. Why do we need channels?

Consider a backend service that needs to:

- Process background jobs
- Send emails asynchronously
- Fetch data from multiple services concurrently
- Limit the number of concurrent requests
- Stream events
- Coordinate graceful shutdown

Goroutines provide concurrency, but goroutines must often:

- Exchange data
- Signal completion
- Report errors
- Stop other goroutines
- Coordinate access to resources

Channels help us implement these operations safely.

Without communication:

```go
go processPayment()
go sendEmail()
go updateInventory()
```

These goroutines start concurrently, but:

- How do we know when they finish?
- How do they return results?
- How do they report errors?
- How can we stop them?

Channels provide answers to these questions.

---

## 2. Creating a channel

A channel is created using `make`.

```go
ch := make(chan int)
```

This creates a channel that carries `int` values.

Other examples:

```go
stringChannel := make(chan string)
errorChannel := make(chan error)
userChannel := make(chan User)
```

A channel is strongly typed. A `chan int` can carry only integers.

---

## 3. Sending and receiving values

The `<-` operator is used for channel operations.

### Sending

```go
ch <- 10
```

This sends `10` into the channel.

### Receiving

```go
value := <-ch
```

This receives a value from the channel.

### Complete example

```go
package main

import "fmt"

func main() {
	ch := make(chan int)

	go func() {
		ch <- 10
	}()

	value := <-ch

	fmt.Println(value)
}
```

Output:

```text
10
```

### What happens internally?

1. The main goroutine creates the channel.
2. A new goroutine is started.
3. The new goroutine tries to send `10`.
4. The main goroutine waits to receive a value.
5. The value is transferred.
6. Both goroutines continue.

---

## 4. Channels are blocking by default

Channel operations normally block until the other side is ready.

For an unbuffered channel:

- A send waits for a receiver.
- A receive waits for a sender.

```go
ch := make(chan int)

ch <- 10 // Blocks because no receiver is ready
```

This program causes a deadlock:

```go
package main

func main() {
	ch := make(chan int)

	ch <- 10
}
```

Runtime error:

```text
fatal error: all goroutines are asleep - deadlock!
```

The send happens in the main goroutine, but no other goroutine exists to receive the value.

A correct version is:

```go
package main

import "fmt"

func main() {
	ch := make(chan int)

	go func() {
		ch <- 10
	}()

	fmt.Println(<-ch)
}
```

---

## 5. Unbuffered channels

An unbuffered channel has no internal storage.

```go
ch := make(chan int)
```

A value can be sent only when a receiver is ready.

Think of it like handing a document directly to another person:

- The sender must be present.
- The receiver must be present.
- The handoff occurs directly.

```go
package main

import "fmt"

func worker(done chan bool) {
	fmt.Println("Worker started")

	// Perform some work.

	fmt.Println("Worker completed")
	done <- true
}

func main() {
	done := make(chan bool)

	go worker(done)

	<-done

	fmt.Println("Application completed")
}
```

Output:

```text
Worker started
Worker completed
Application completed
```

The receive operation:

```go
<-done
```

blocks the main goroutine until the worker sends a completion signal.

### When are unbuffered channels useful?

Use them when:

- The sender must wait for the receiver.
- You need strict synchronization.
- You want a direct handoff.
- Continuing without successful delivery would be incorrect.

---

## 6. Buffered channels

A buffered channel contains an internal queue.

```go
ch := make(chan int, 3)
```

This channel can temporarily hold three integers.

```go
package main

import "fmt"

func main() {
	ch := make(chan string, 2)

	ch <- "first"
	ch <- "second"

	fmt.Println(<-ch)
	fmt.Println(<-ch)
}
```

Output:

```text
first
second
```

The sends do not immediately block because the buffer has available space.

### When does a buffered send block?

A send blocks when the buffer is full.

```go
ch := make(chan int, 2)

ch <- 10
ch <- 20
ch <- 30 // Blocks because the buffer is full
```

### When does a buffered receive block?

A receive blocks when the buffer is empty.

```go
ch := make(chan int, 2)

value := <-ch // Blocks because the channel is empty
```

### Capacity and length

```go
ch := make(chan int, 3)

ch <- 10
ch <- 20

fmt.Println(len(ch)) // 2
fmt.Println(cap(ch)) // 3
```

- `len(ch)` is the number of queued values.
- `cap(ch)` is the buffer capacity.

These values are usually useful for diagnostics. Avoid using `len(ch)` to make concurrency decisions because the value can change immediately.

### Buffered channel mental model

Think of a buffered channel as a small waiting room:

- Senders add values to the queue.
- Receivers remove values from the queue.
- A sender waits when the queue is full.
- A receiver waits when the queue is empty.

---

## 7. Unbuffered vs buffered channels

| Feature | Unbuffered | Buffered |
|---|---|---|
| Creation | `make(chan int)` | `make(chan int, 10)` |
| Internal queue | No | Yes |
| Send blocks | Until a receiver is ready | When the buffer is full |
| Receive blocks | Until a sender is ready | When the buffer is empty |
| Primary purpose | Synchronization and direct handoff | Queueing and limited decoupling |
| Common examples | Completion signals | Job queues |

A buffer should have a reason.

Good reasons include:

- There can be at most `N` pending jobs.
- A burst of `N` events should be absorbed.
- A producer is allowed to run slightly ahead.
- The buffer represents a real resource limit.

Avoid adding a huge buffer just to hide blocking. It may only move the problem from blocking to excessive memory usage or delayed processing.

---

## 8. Closing a channel

Closing a channel means:

> No more values will be sent through this channel.

```go
close(ch)
```

Example:

```go
package main

import "fmt"

func produce(ch chan int) {
	for i := 1; i <= 3; i++ {
		ch <- i
	}

	close(ch)
}

func main() {
	ch := make(chan int)

	go produce(ch)

	for value := range ch {
		fmt.Println(value)
	}
}
```

Output:

```text
1
2
3
```

The loop continues receiving values until the channel is:

1. Closed
2. Completely drained

### Who should close a channel?

The sending side should usually close the channel.

A useful rule is:

> The goroutine that owns the channel and knows that no more values will be sent should close it.

Receivers normally should not close channels because they may not know whether another sender still needs to send.

### Closing is not always required

You do not need to close every channel.

Closing is useful when receivers need to know:

- No more values are coming.
- A `range` loop should terminate.
- A stream or pipeline stage has completed.
- A broadcast cancellation signal should be sent.

If a channel becomes unreachable and is garbage-collected, it does not need to be closed just for cleanup.

---

## 9. Receiving from a closed channel

A receive operation can return two values:

```go
value, ok := <-ch
```

- `value` is the received value.
- `ok` is `true` if a real value was received.
- `ok` is `false` when the channel is closed and empty.

Example:

```go
package main

import "fmt"

func main() {
	ch := make(chan int, 1)

	ch <- 100
	close(ch)

	value, ok := <-ch
	fmt.Println(value, ok)

	value, ok = <-ch
	fmt.Println(value, ok)
}
```

Output:

```text
100 true
0 false
```

After a closed channel is drained, receives return immediately with the type’s zero value.

Examples:

- `int` → `0`
- `string` → `""`
- `bool` → `false`
- Pointer → `nil`

This is why the `ok` value matters when a zero value could also be valid data.

---

## 10. Important closing rules

### Sending to a closed channel

This panics:

```go
close(ch)
ch <- 10
```

Runtime result:

```text
panic: send on closed channel
```

### Closing an already closed channel

This also panics:

```go
close(ch)
close(ch)
```

### Receiving from a closed channel

This is safe:

```go
value, ok := <-ch
```

### Closing a nil channel

This panics:

```go
var ch chan int
close(ch)
```

---

## 11. Nil channels

The zero value of a channel is `nil`.

```go
var ch chan int
```

A nil channel is not usable for normal communication:

```go
ch <- 10    // Blocks forever
value := <-ch // Blocks forever
```

This behavior can cause bugs, but it is also useful in advanced `select` patterns because a `select` case involving a nil channel is disabled.

Example:

```go
package main

import "fmt"

func main() {
	first := make(chan string)
	second := make(chan string)

	go func() {
		first <- "first completed"
		close(first)
	}()

	go func() {
		second <- "second completed"
		close(second)
	}()

	for first != nil || second != nil {
		select {
		case value, ok := <-first:
			if !ok {
				first = nil
				continue
			}

			fmt.Println(value)

		case value, ok := <-second:
			if !ok {
				second = nil
				continue
			}

			fmt.Println(value)
		}
	}
}
```

Setting a closed channel to `nil` disables that case.

Without doing this, receiving from the closed channel would always be immediately ready and could cause a busy loop.

---

## 12. Directional channels

Function parameters can specify whether a channel is used for sending or receiving.

### Send-only channel

```go
chan<- int
```

### Receive-only channel

```go
<-chan int
```

Example:

```go
package main

import "fmt"

func produce(out chan<- int) {
	for i := 1; i <= 3; i++ {
		out <- i
	}

	close(out)
}

func consume(in <-chan int) {
	for value := range in {
		fmt.Println(value)
	}
}

func main() {
	ch := make(chan int)

	go produce(ch)

	consume(ch)
}
```

Directional channels improve clarity and prevent accidental misuse.

This function can only send:

```go
func produce(out chan<- int) {
	out <- 10
}
```

This function can only receive:

```go
func consume(in <-chan int) {
	value := <-in
	fmt.Println(value)
}
```

Use channel direction in function signatures whenever possible.

---

## 13. The `select` statement

`select` allows a goroutine to wait on multiple channel operations.

It resembles `switch`, but it works with channels.

```go
select {
case value := <-channelOne:
	fmt.Println("Received:", value)

case value := <-channelTwo:
	fmt.Println("Received:", value)
}
```

Whichever channel operation becomes ready first is executed.

Example:

```go
package main

import (
	"fmt"
	"time"
)

func main() {
	first := make(chan string)
	second := make(chan string)

	go func() {
		time.Sleep(1 * time.Second)
		first <- "response from first service"
	}()

	go func() {
		time.Sleep(500 * time.Millisecond)
		second <- "response from second service"
	}()

	select {
	case result := <-first:
		fmt.Println(result)

	case result := <-second:
		fmt.Println(result)
	}
}
```

Output:

```text
response from second service
```

If multiple cases are ready at the same time, Go chooses one pseudo-randomly. You must not rely on a fixed priority.

---

## 14. Non-blocking channel operations

A `default` case makes `select` non-blocking.

### Non-blocking receive

```go
select {
case value := <-ch:
	fmt.Println("Received:", value)

default:
	fmt.Println("No value available")
}
```

### Non-blocking send

```go
select {
case ch <- value:
	fmt.Println("Value sent")

default:
	fmt.Println("Channel is not ready")
}
```

This is useful when dropping work is explicitly acceptable, such as best-effort metrics:

```go
func recordMetric(metrics chan<- string, metric string) {
	select {
	case metrics <- metric:
		// Metric queued.

	default:
		// Queue full; deliberately drop the metric.
	}
}
```

Do not use `default` automatically. It changes the meaning from:

> Wait until communication succeeds.

to:

> Try once, and skip if it cannot happen immediately.

---

## 15. Channel synchronization and visibility

Channels do more than transfer values. They establish synchronization between goroutines.

Conceptually:

```go
data := 0

go func() {
	data = 100
	done <- struct{}{}
}()

<-done
fmt.Println(data)
```

The receive ensures that the write before the send is visible after the receive.

A better version avoids shared mutable data entirely:

```go
result := make(chan int)

go func() {
	data := 100
	result <- data
}()

fmt.Println(<-result)
```

The safest principle is:

> Prefer transferring ownership of data instead of having several goroutines mutate the same data.

Channels do not make every shared variable automatically safe. If goroutines access unrelated shared state concurrently, you may still need a mutex or another synchronization mechanism.

---

## 16. Using `struct{}` for signals

Sometimes a channel carries no actual data. It only represents an event.

Use:

```go
chan struct{}
```

An empty struct occupies zero bytes.

```go
package main

import (
	"fmt"
	"time"
)

func worker(done chan<- struct{}) {
	time.Sleep(time.Second)
	fmt.Println("Work completed")

	close(done)
}

func main() {
	done := make(chan struct{})

	go worker(done)

	<-done

	fmt.Println("Main completed")
}
```

Closing the channel is especially useful because every receiver waiting on it is released.

---

# Channel Patterns

A channel pattern is a commonly used arrangement of goroutines and channels that solves a recurring concurrency problem.

---

## 17. Pattern 1: Result channel

Use a channel to return a result from a goroutine.

```go
package main

import "fmt"

func calculateSquare(number int, result chan<- int) {
	result <- number * number
}

func main() {
	result := make(chan int)

	go calculateSquare(5, result)

	square := <-result

	fmt.Println(square)
}
```

Output:

```text
25
```

This pattern is useful for executing an operation concurrently and waiting for its result.

---

## 18. Pattern 2: Result with error

Backend operations often return both a value and an error.

Instead of using two separate channels, define a result type:

```go
package main

import (
	"errors"
	"fmt"
)

type Result struct {
	Value int
	Err   error
}

func divide(a, b int) <-chan Result {
	resultChannel := make(chan Result, 1)

	go func() {
		defer close(resultChannel)

		if b == 0 {
			resultChannel <- Result{
				Err: errors.New("division by zero"),
			}
			return
		}

		resultChannel <- Result{
			Value: a / b,
		}
	}()

	return resultChannel
}

func main() {
	result := <-divide(10, 2)

	if result.Err != nil {
		fmt.Println("Error:", result.Err)
		return
	}

	fmt.Println("Result:", result.Value)
}
```

Keeping the value and error together prevents mismatching results from different operations.

In many ordinary functions, returning `(value, error)` directly remains simpler. Use a channel result when the work truly needs to run asynchronously or participate in a concurrent workflow.

---

## 19. Pattern 3: Done channel

A done channel tells a goroutine or caller that work has completed.

```go
package main

import (
	"fmt"
	"time"
)

func process(done chan<- struct{}) {
	defer close(done)

	fmt.Println("Processing started")
	time.Sleep(time.Second)
	fmt.Println("Processing finished")
}

func main() {
	done := make(chan struct{})

	go process(done)

	<-done

	fmt.Println("Safe to exit")
}
```

Because the channel is closed, multiple goroutines can wait for the same completion event:

```go
close(done)
```

All receivers waiting on `done` will be released.

---

## 20. Pattern 4: Cancellation with context

In production backend code, `context.Context` is usually the preferred cancellation mechanism.

It handles:

- Client disconnection
- Request cancellation
- Deadlines
- Timeouts
- Service shutdown
- Cancellation across multiple function calls

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
			fmt.Println("Worker stopped:", ctx.Err())
			return

		default:
			fmt.Println("Working...")
			time.Sleep(300 * time.Millisecond)
		}
	}
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())

	go worker(ctx)

	time.Sleep(time.Second)

	cancel()

	time.Sleep(100 * time.Millisecond)
}
```

A more production-friendly worker avoids sleeping inside the `default` case:

```go
func worker(ctx context.Context) {
	ticker := time.NewTicker(300 * time.Millisecond)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			fmt.Println("Worker stopped:", ctx.Err())
			return

		case <-ticker.C:
			fmt.Println("Working...")
		}
	}
}
```

### Important cancellation rule

A goroutine must actively observe cancellation:

```go
select {
case <-ctx.Done():
	return

case value := <-input:
	// Process value.
}
```

Calling `cancel()` does not forcefully terminate a goroutine. The goroutine must check `ctx.Done()` and return.

---

## 21. Pattern 5: Timeout

A channel operation can be limited with a timeout.

```go
package main

import (
	"fmt"
	"time"
)

func fetchData(result chan<- string) {
	time.Sleep(2 * time.Second)
	result <- "data received"
}

func main() {
	result := make(chan string, 1)

	go fetchData(result)

	select {
	case value := <-result:
		fmt.Println(value)

	case <-time.After(time.Second):
		fmt.Println("Operation timed out")
	}
}
```

Output:

```text
Operation timed out
```

### Why is the result channel buffered?

```go
result := make(chan string, 1)
```

If the caller times out, `fetchData` eventually attempts to send its result.

With an unbuffered channel, it could block forever because the caller is no longer receiving.

The buffer allows the single result to be delivered even after the caller has left.

However, buffering alone does not cancel the underlying work. Prefer context-aware operations when possible:

```go
func fetchData(ctx context.Context) (string, error) {
	select {
	case <-time.After(2 * time.Second):
		return "data received", nil

	case <-ctx.Done():
		return "", ctx.Err()
	}
}
```

Usage:

```go
ctx, cancel := context.WithTimeout(
	context.Background(),
	time.Second,
)
defer cancel()

result, err := fetchData(ctx)
if err != nil {
	fmt.Println("Error:", err)
	return
}

fmt.Println(result)
```

For repeated timeout logic, `time.NewTimer` is generally easier to manage than repeatedly calling `time.After`.

---

## 22. Pattern 6: Producer–consumer

One or more producers create work, and one or more consumers process it.

```go
package main

import (
	"fmt"
	"time"
)

func producer(jobs chan<- int) {
	defer close(jobs)

	for job := 1; job <= 5; job++ {
		fmt.Println("Produced job:", job)
		jobs <- job
	}
}

func consumer(jobs <-chan int) {
	for job := range jobs {
		fmt.Println("Processing job:", job)
		time.Sleep(300 * time.Millisecond)
	}
}

func main() {
	jobs := make(chan int, 2)

	go producer(jobs)

	consumer(jobs)
}
```

The buffered channel acts as a bounded queue.

This pattern is useful for:

- Email jobs
- Notification jobs
- Image processing
- Event processing
- Database write batching
- Audit log processing

---

## 23. Pattern 7: Worker pool

A worker pool uses a fixed number of goroutines to process many jobs.

Without a worker pool, we might create one goroutine for every request:

```go
for _, job := range jobs {
	go process(job)
}
```

If there are one million jobs, this creates one million goroutines. Goroutines are lightweight, but they are not free. The downstream database or service may also be unable to handle that concurrency.

A worker pool limits concurrency.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Job struct {
	ID int
}

type Result struct {
	JobID  int
	Output string
}

func worker(
	id int,
	jobs <-chan Job,
	results chan<- Result,
	wg *sync.WaitGroup,
) {
	defer wg.Done()

	for job := range jobs {
		fmt.Printf("Worker %d processing job %d\n", id, job.ID)

		time.Sleep(500 * time.Millisecond)

		results <- Result{
			JobID:  job.ID,
			Output: fmt.Sprintf("processed by worker %d", id),
		}
	}
}

func main() {
	const workerCount = 3
	const jobCount = 10

	jobs := make(chan Job, jobCount)
	results := make(chan Result, jobCount)

	var wg sync.WaitGroup

	for workerID := 1; workerID <= workerCount; workerID++ {
		wg.Add(1)
		go worker(workerID, jobs, results, &wg)
	}

	for jobID := 1; jobID <= jobCount; jobID++ {
		jobs <- Job{ID: jobID}
	}
	close(jobs)

	go func() {
		wg.Wait()
		close(results)
	}()

	for result := range results {
		fmt.Printf(
			"Job %d: %s\n",
			result.JobID,
			result.Output,
		)
	}
}
```

### How it works

1. `jobs` distributes work.
2. Three worker goroutines receive from the same channel.
3. Each job is received by only one worker.
4. Workers send completed work to `results`.
5. `WaitGroup` tracks when all workers finish.
6. After all workers finish, `results` is closed.
7. The main goroutine ranges over results until completion.

### Why not close `results` inside a worker?

Multiple workers send to `results`.

One worker cannot know whether every other worker has finished. If one worker closes the channel too early, other workers may panic while sending.

Therefore, a coordinator waits for all workers and closes `results`.

---

## 24. Pattern 8: Fan-out

Fan-out means distributing work from one channel across multiple goroutines.

```text
                    ┌─ Worker 1
Jobs channel ───────┼─ Worker 2
                    └─ Worker 3
```

When multiple goroutines receive from the same channel, each value is delivered to one receiver.

```go
for i := 1; i <= 3; i++ {
	go worker(i, jobs)
}
```

Fan-out is useful when:

- Jobs are independent.
- Processing is CPU-intensive or I/O-intensive.
- You want controlled parallelism.
- Order of completion does not matter.

Be careful: results may complete in a different order than the jobs were submitted.

---

## 25. Pattern 9: Fan-in

Fan-in combines multiple input channels into one output channel.

```text
Source 1 ──┐
Source 2 ──┼──> Combined output
Source 3 ──┘
```

```go
package main

import (
	"fmt"
	"sync"
)

func fanIn(inputs ...<-chan string) <-chan string {
	output := make(chan string)

	var wg sync.WaitGroup
	wg.Add(len(inputs))

	for _, input := range inputs {
		go func(ch <-chan string) {
			defer wg.Done()

			for value := range ch {
				output <- value
			}
		}(input)
	}

	go func() {
		wg.Wait()
		close(output)
	}()

	return output
}

func produce(name string, values ...string) <-chan string {
	output := make(chan string)

	go func() {
		defer close(output)

		for _, value := range values {
			output <- fmt.Sprintf("%s: %s", name, value)
		}
	}()

	return output
}

func main() {
	first := produce("service-one", "A", "B")
	second := produce("service-two", "C", "D")

	for value := range fanIn(first, second) {
		fmt.Println(value)
	}
}
```

The order is not guaranteed because the sources run concurrently.

Fan-in is useful for:

- Combining events from multiple services
- Collecting results from workers
- Aggregating log streams
- Combining multiple data sources

---

## 26. Pattern 10: Pipeline

A pipeline separates processing into stages.

Each stage:

1. Receives values from an input channel.
2. Transforms the values.
3. Sends them to an output channel.

Example:

```go
package main

import "fmt"

func generate(numbers ...int) <-chan int {
	output := make(chan int)

	go func() {
		defer close(output)

		for _, number := range numbers {
			output <- number
		}
	}()

	return output
}

func square(input <-chan int) <-chan int {
	output := make(chan int)

	go func() {
		defer close(output)

		for number := range input {
			output <- number * number
		}
	}()

	return output
}

func filterEven(input <-chan int) <-chan int {
	output := make(chan int)

	go func() {
		defer close(output)

		for number := range input {
			if number%2 == 0 {
				output <- number
			}
		}
	}()

	return output
}

func main() {
	numbers := generate(1, 2, 3, 4, 5)
	squares := square(numbers)
	evenSquares := filterEven(squares)

	for value := range evenSquares {
		fmt.Println(value)
	}
}
```

Output:

```text
4
16
```

Pipeline flow:

```text
generate → square → filterEven → main
```

Pipelines are useful for:

- Data processing
- Event transformation
- ETL systems
- Log processing
- Request enrichment
- Streaming workloads

### Pipeline danger: early consumer exit

Suppose the final consumer reads only one value:

```go
value := <-evenSquares
fmt.Println(value)
```

Upstream stages may remain blocked while trying to send additional values. This causes goroutine leaks.

Production pipelines should support cancellation.

---

## 27. Cancellation-aware pipeline

```go
package main

import (
	"context"
	"fmt"
)

func generate(
	ctx context.Context,
	numbers ...int,
) <-chan int {
	output := make(chan int)

	go func() {
		defer close(output)

		for _, number := range numbers {
			select {
			case output <- number:
			case <-ctx.Done():
				return
			}
		}
	}()

	return output
}

func square(
	ctx context.Context,
	input <-chan int,
) <-chan int {
	output := make(chan int)

	go func() {
		defer close(output)

		for {
			select {
			case <-ctx.Done():
				return

			case number, ok := <-input:
				if !ok {
					return
				}

				select {
				case output <- number * number:
				case <-ctx.Done():
					return
				}
			}
		}
	}()

	return output
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	numbers := generate(ctx, 1, 2, 3, 4, 5)
	squares := square(ctx, numbers)

	fmt.Println(<-squares)

	// We do not need the remaining values.
	cancel()
}
```

Every potentially blocking send checks cancellation:

```go
select {
case output <- value:
case <-ctx.Done():
	return
}
```

This prevents an upstream goroutine from remaining blocked after the consumer stops.

---

## 28. Pattern 11: Semaphore for limiting concurrency

A buffered channel can act as a semaphore.

Suppose a service needs to process 100 requests, but only five should run concurrently.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

func process(id int) {
	fmt.Println("Started:", id)
	time.Sleep(time.Second)
	fmt.Println("Finished:", id)
}

func main() {
	const maxConcurrency = 5

	semaphore := make(chan struct{}, maxConcurrency)

	var wg sync.WaitGroup

	for id := 1; id <= 20; id++ {
		wg.Add(1)

		go func(jobID int) {
			defer wg.Done()

			semaphore <- struct{}{}
			defer func() {
				<-semaphore
			}()

			process(jobID)
		}(id)
	}

	wg.Wait()
}
```

### How it works

Acquiring a slot:

```go
semaphore <- struct{}{}
```

Releasing a slot:

```go
<-semaphore
```

When five slots are occupied, the next send blocks.

Always release the slot with `defer` after acquiring it:

```go
semaphore <- struct{}{}
defer func() {
	<-semaphore
}()
```

This ensures the slot is released even if the function returns early.

### Backend use cases

- Limit outbound HTTP calls.
- Limit database operations.
- Limit file processing.
- Protect a rate-limited dependency.
- Prevent CPU or memory overload.

This limits concurrency, not request rate. For requests-per-second rate limiting, use a ticker, token bucket or rate limiter.

---

## 29. Pattern 12: Bounded job queue and backpressure

A buffered channel can implement a bounded job queue.

```go
type JobQueue struct {
	jobs chan Job
}

func NewJobQueue(size int) *JobQueue {
	return &JobQueue{
		jobs: make(chan Job, size),
	}
}

func (q *JobQueue) Submit(job Job) error {
	select {
	case q.jobs <- job:
		return nil

	default:
		return errors.New("job queue is full")
	}
}
```

If the queue is full, the service rejects the job immediately.

This is called **backpressure**.

Backpressure prevents a fast producer from overwhelming a slow consumer.

Possible backpressure policies include:

- Wait until queue space is available.
- Reject the new job.
- Drop the new job.
- Drop the oldest queued job.
- Retry for a limited time.
- Persist the job in an external queue.

The correct policy depends on the business requirement.

For example:

- Dropping metrics might be acceptable.
- Dropping payment events is not acceptable.
- Email jobs may need durable storage.
- HTTP requests may return `429` or `503`.

An in-memory Go channel is not a durable queue. All queued work is lost if the process crashes.

For critical jobs, use systems such as Kafka, RabbitMQ, SQS or a database-backed queue.

---

## 30. Pattern 13: First successful response

Sometimes a backend calls multiple replicas and uses the first successful response.

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"time"
)

type Response struct {
	Data string
	Err  error
}

func callService(
	ctx context.Context,
	name string,
	delay time.Duration,
	shouldFail bool,
) Response {
	select {
	case <-time.After(delay):
		if shouldFail {
			return Response{
				Err: errors.New(name + " failed"),
			}
		}

		return Response{
			Data: "response from " + name,
		}

	case <-ctx.Done():
		return Response{
			Err: ctx.Err(),
		}
	}
}

func fastestResponse(ctx context.Context) (string, error) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	responses := make(chan Response, 3)

	services := []struct {
		name       string
		delay      time.Duration
		shouldFail bool
	}{
		{"replica-one", 800 * time.Millisecond, false},
		{"replica-two", 300 * time.Millisecond, true},
		{"replica-three", 500 * time.Millisecond, false},
	}

	for _, service := range services {
		service := service

		go func() {
			response := callService(
				ctx,
				service.name,
				service.delay,
				service.shouldFail,
			)

			select {
			case responses <- response:
			case <-ctx.Done():
			}
		}()
	}

	var lastErr error

	for range services {
		select {
		case response := <-responses:
			if response.Err == nil {
				cancel()
				return response.Data, nil
			}

			lastErr = response.Err

		case <-ctx.Done():
			return "", ctx.Err()
		}
	}

	return "", lastErr
}

func main() {
	result, err := fastestResponse(context.Background())
	if err != nil {
		fmt.Println("Error:", err)
		return
	}

	fmt.Println(result)
}
```

This is sometimes called a hedged-request or first-response pattern. Use it carefully because it creates extra load on downstream services.

---

## 31. Pattern 14: Periodic worker

A ticker channel can trigger repeated work.

```go
package main

import (
	"context"
	"fmt"
	"time"
)

func runCleanup(ctx context.Context) {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ticker.C:
			fmt.Println("Running cleanup")

		case <-ctx.Done():
			fmt.Println("Cleanup worker stopped")
			return
		}
	}
}

func main() {
	ctx, cancel := context.WithTimeout(
		context.Background(),
		12*time.Second,
	)
	defer cancel()

	runCleanup(ctx)
}
```

Use `time.NewTicker` when work must run repeatedly.

Use `time.NewTimer` for a one-time delay.

Always stop tickers you create:

```go
defer ticker.Stop()
```

---

## 32. Pattern 15: Graceful shutdown

Channels and context are frequently used during server shutdown.

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
	server := &http.Server{
		Addr: ":8080",
	}

	go func() {
		log.Println("Server started on :8080")

		err := server.ListenAndServe()
		if err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Fatal(err)
		}
	}()

	shutdownSignal := make(
		chan os.Signal,
		1,
	)

	signal.Notify(
		shutdownSignal,
		syscall.SIGINT,
		syscall.SIGTERM,
	)

	<-shutdownSignal

	log.Println("Shutdown signal received")

	ctx, cancel := context.WithTimeout(
		context.Background(),
		10*time.Second,
	)
	defer cancel()

	if err := server.Shutdown(ctx); err != nil {
		log.Printf("Graceful shutdown failed: %v", err)
		return
	}

	log.Println("Server stopped gracefully")
}
```

The channel receives operating-system signals. When a signal arrives, the server stops accepting new requests and waits for existing requests to complete within the deadline.

In modern Go, `signal.NotifyContext` is often cleaner:

```go
ctx, stop := signal.NotifyContext(
	context.Background(),
	syscall.SIGINT,
	syscall.SIGTERM,
)
defer stop()

<-ctx.Done()
```

---

## 33. Channels vs mutexes

Channels and mutexes solve related but different problems.

### Use a channel when:

- Transferring ownership of data
- Passing jobs to workers
- Streaming values
- Returning asynchronous results
- Signalling completion or cancellation
- Coordinating pipeline stages

### Use a mutex when:

- Protecting shared in-memory state
- Multiple goroutines need direct access to the same data
- The operation is a short critical section
- No stream of values needs to be communicated

Example using a mutex:

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.value++
}
```

Forcing this into channels may make the code unnecessarily complex.

A useful principle is:

> Use channels for communication and ownership transfer.  
> Use mutexes for protecting shared state.

Do not interpret “share memory by communicating” as “mutexes are bad.” Both are first-class Go tools.

---

## 34. Common channel mistakes

### Mistake 1: Sending without a receiver

```go
ch := make(chan int)
ch <- 10
```

This deadlocks.

Ensure another goroutine can receive, or intentionally use a suitable buffer.

---

### Mistake 2: Reading forever from a channel that is never closed

```go
for value := range ch {
	fmt.Println(value)
}
```

The loop stops only when `ch` is closed and drained.

If the sender never closes it, the receiver waits forever.

---

### Mistake 3: Closing a channel from the receiver

```go
func consumer(ch chan int) {
	value := <-ch
	close(ch)
}
```

Another producer may still send and panic.

The sender or coordinating owner should usually close it.

---

### Mistake 4: Multiple goroutines closing the same channel

```go
go func() {
	close(ch)
}()

go func() {
	close(ch)
}()
```

One close succeeds; the other panics.

Channel closure needs one clear owner.

---

### Mistake 5: Using a channel as an unbounded queue

A channel has finite capacity.

If producers permanently outpace consumers:

- Producers eventually block.
- Memory grows if the buffer is enormous.
- Latency increases.
- The system becomes unhealthy.

Choose an explicit overload policy.

---

### Mistake 6: Goroutine leak

```go
func generate() <-chan int {
	ch := make(chan int)

	go func() {
		for i := 0; ; i++ {
			ch <- i
		}
	}()

	return ch
}

func main() {
	ch := generate()

	fmt.Println(<-ch)
}
```

The consumer reads one value and stops.

The producer remains blocked forever trying to send another value.

Fix it with cancellation:

```go
func generate(ctx context.Context) <-chan int {
	ch := make(chan int)

	go func() {
		defer close(ch)

		for i := 0; ; i++ {
			select {
			case ch <- i:
			case <-ctx.Done():
				return
			}
		}
	}()

	return ch
}
```

---

### Mistake 7: Assuming channel buffer size improves everything

```go
jobs := make(chan Job, 1_000_000)
```

A huge buffer can:

- Consume significant memory.
- Hide slow consumers.
- Increase job latency.
- Delay discovery of overload.
- Lose more pending work during a crash.

Size buffers based on a real system constraint.

---

### Mistake 8: Forgetting cancellation during sends

This can leak:

```go
output <- result
```

A safer version in a cancellable workflow is:

```go
select {
case output <- result:
case <-ctx.Done():
	return
}
```

Both receives and sends can block, so cancellation may need to be considered around both operations.

---

### Mistake 9: Copying a `WaitGroup`

Pass a pointer:

```go
func worker(wg *sync.WaitGroup) {
	defer wg.Done()
}
```

Do not pass it by value:

```go
func worker(wg sync.WaitGroup) {
	defer wg.Done()
}
```

The copied `WaitGroup` is separate from the original and can lead to incorrect coordination.

---

### Mistake 10: Capturing loop variables incorrectly

Write the value explicitly into the goroutine:

```go
for _, job := range jobs {
	job := job

	go func() {
		process(job)
	}()
}
```

Or pass it as an argument:

```go
for _, job := range jobs {
	go func(current Job) {
		process(current)
	}(job)
}
```

This is explicit and remains easy to understand across Go versions and different loop forms.

---

## 35. Channel ownership

Clear ownership prevents most channel bugs.

A function that creates an output channel should normally:

- Start the goroutine that writes to it.
- Return a receive-only channel.
- Close the channel when production finishes.

```go
func generate(numbers ...int) <-chan int {
	output := make(chan int)

	go func() {
		defer close(output)

		for _, number := range numbers {
			output <- number
		}
	}()

	return output
}
```

The caller can receive but cannot accidentally send or close it through the receive-only value:

```go
numbers := generate(1, 2, 3)
```

Ask these questions for every channel:

1. Who creates it?
2. Who sends to it?
3. Who receives from it?
4. Who closes it?
5. What happens if sending blocks?
6. What happens if receiving blocks?
7. How is cancellation handled?
8. Is data allowed to be dropped?
9. What is the buffer supposed to represent?

If these answers are unclear, the design is probably incomplete.

---

## 36. Ordering guarantees

A single channel preserves send order.

```go
ch <- 1
ch <- 2
ch <- 3
```

A receiver observes:

```text
1, 2, 3
```

However, with multiple concurrent senders, execution order is nondeterministic:

```go
go func() {
	ch <- "A"
}()

go func() {
	ch <- "B"
}()
```

Either `A` or `B` may arrive first.

A worker pool also does not normally preserve completion order. If result order matters, attach sequence numbers:

```go
type Job struct {
	Index int
	Value int
}

type Result struct {
	Index int
	Value int
}
```

The collector can reorder results using `Index`.

---

## 37. A realistic backend example

The following example represents an asynchronous email service with:

- A bounded queue
- Multiple workers
- Context cancellation
- Graceful shutdown
- Backpressure

```go
package main

import (
	"context"
	"errors"
	"fmt"
	"sync"
	"time"
)

var ErrQueueFull = errors.New("email queue is full")

type EmailJob struct {
	ID      int
	Address string
	Message string
}

type EmailService struct {
	jobs chan EmailJob
	wg   sync.WaitGroup
}

func NewEmailService(queueSize int) *EmailService {
	return &EmailService{
		jobs: make(chan EmailJob, queueSize),
	}
}

func (s *EmailService) Start(
	ctx context.Context,
	workerCount int,
) {
	for workerID := 1; workerID <= workerCount; workerID++ {
		s.wg.Add(1)

		go s.worker(ctx, workerID)
	}
}

func (s *EmailService) Submit(job EmailJob) error {
	select {
	case s.jobs <- job:
		return nil

	default:
		return ErrQueueFull
	}
}

func (s *EmailService) worker(
	ctx context.Context,
	workerID int,
) {
	defer s.wg.Done()

	for {
		select {
		case <-ctx.Done():
			fmt.Printf(
				"Worker %d stopping: %v\n",
				workerID,
				ctx.Err(),
			)
			return

		case job, ok := <-s.jobs:
			if !ok {
				fmt.Printf(
					"Worker %d finished\n",
					workerID,
				)
				return
			}

			s.sendEmail(workerID, job)
		}
	}
}

func (s *EmailService) sendEmail(
	workerID int,
	job EmailJob,
) {
	fmt.Printf(
		"Worker %d sending email job %d to %s\n",
		workerID,
		job.ID,
		job.Address,
	)

	time.Sleep(500 * time.Millisecond)
}

func (s *EmailService) Shutdown() {
	close(s.jobs)
	s.wg.Wait()
}

func main() {
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	emailService := NewEmailService(10)
	emailService.Start(ctx, 3)

	for id := 1; id <= 8; id++ {
		err := emailService.Submit(EmailJob{
			ID:      id,
			Address: fmt.Sprintf("user%d@example.com", id),
			Message: "Welcome!",
		})

		if err != nil {
			fmt.Printf(
				"Could not submit job %d: %v\n",
				id,
				err,
			)
		}
	}

	emailService.Shutdown()

	fmt.Println("Email service stopped")
}
```

### Design observations

- The queue accepts at most ten pending jobs.
- Three workers process jobs concurrently.
- `Submit` fails immediately if the queue is full.
- Only `Shutdown` closes the jobs channel.
- The `WaitGroup` waits for workers.
- Workers stop when the queue is closed and drained.
- Workers can also stop when the context is cancelled.

There is an important business decision here:

- Closing `jobs` allows already queued jobs to drain.
- Cancelling the context may stop workers before queued jobs are processed.

Production code should explicitly decide whether shutdown should:

1. Drain pending work.
2. Stop immediately.

Also remember that an in-memory queue loses jobs if the process crashes. Critical emails should normally be backed by durable storage.

---

## 38. Testing channel-based code

Channel tests should use timeouts so a bug causes a test failure instead of hanging forever.

```go
func TestWorker(t *testing.T) {
	result := make(chan int, 1)

	go func() {
		result <- 42
	}()

	select {
	case value := <-result:
		if value != 42 {
			t.Fatalf(
				"expected 42, got %d",
				value,
			)
		}

	case <-time.After(time.Second):
		t.Fatal("timed out waiting for result")
	}
}
```

Also run the race detector:

```bash
go test -race ./...
```

The race detector helps find unsafe shared-memory access. It does not prove that your channel design cannot deadlock or leak goroutines, so tests still need to exercise cancellation and shutdown paths.

---

## 39. How to choose the correct pattern

| Requirement | Suitable pattern |
|---|---|
| Return work from a goroutine | Result channel |
| Report value and error | Result struct channel |
| Wait for completion | Done channel or `WaitGroup` |
| Stop work | `context.Context` |
| Wait for several operations | `select` |
| Limit execution time | Context timeout |
| Process queued background jobs | Producer–consumer |
| Limit parallel processing | Worker pool |
| Distribute work | Fan-out |
| Combine several streams | Fan-in |
| Transform streaming data | Pipeline |
| Limit concurrent access | Semaphore |
| Handle overload | Bounded queue and backpressure |
| Execute work periodically | Ticker |
| Stop a server safely | Signal context and graceful shutdown |

---

## 40. Practical rules to remember

1. A send can block.
2. A receive can block.
3. An unbuffered channel performs a direct synchronized handoff.
4. A buffered channel is a bounded queue, not an unlimited one.
5. The sending owner usually closes the channel.
6. Do not close a channel merely because you finished receiving.
7. Receiving from a closed channel is safe.
8. Sending to a closed channel panics.
9. Closing an already closed channel panics.
10. Nil-channel sends and receives block forever.
11. Use directional channels in function signatures.
12. Use `context.Context` for cancellation in backend applications.
13. Every long-running goroutine needs a clear exit path.
14. Protect every potentially blocking pipeline operation with cancellation when early exit is possible.
15. Use a `WaitGroup` to wait for a collection of goroutines.
16. Use channels for communication; use mutexes for shared state.
17. Do not use an enormous buffer to hide slow consumers.
18. Decide explicitly what should happen during overload.
19. Run tests with `go test -race`.
20. Prefer the simplest synchronous implementation unless concurrency provides a real benefit.

The most important design principle is:

> Do not start by asking, “Where can I use a channel?”  
> Start by asking, “Which goroutines need to communicate, what do they exchange, and how do they stop?”
