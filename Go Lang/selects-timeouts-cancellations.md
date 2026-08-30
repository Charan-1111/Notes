# Go Concurrency: `select`, Timeouts, and Cancellation

Go uses goroutines and channels to perform concurrent work.

Three important concepts for controlling concurrent operations are:

1. `select` — wait for multiple channel operations.
2. Timeouts — stop waiting when an operation takes too long.
3. Cancellation — tell running goroutines to stop because their work is no longer needed.

These concepts are closely related and are commonly used together in backend applications.

---

# 1. Understanding `select`

## What is `select`?

A `select` statement lets a goroutine wait on multiple channel operations simultaneously.

It looks similar to a `switch`, but it is specifically designed for channels.

```go
select {
case value := <-channel1:
    fmt.Println("Received from channel1:", value)

case value := <-channel2:
    fmt.Println("Received from channel2:", value)
}
```

The `select` waits until one of the channel operations is ready.

When a channel becomes ready, the corresponding case executes.

---

## Why do we need `select`?

Imagine that your backend service requests information from two other services:

- User service
- Order service

You want to process whichever response arrives first.

Without `select`, you might wait for the user service first even if the order service has already responded.

With `select`, you can wait for both simultaneously.

---

## Basic `select` example

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    channel1 := make(chan string)
    channel2 := make(chan string)

    go func() {
        time.Sleep(2 * time.Second)
        channel1 <- "Response from channel 1"
    }()

    go func() {
        time.Sleep(1 * time.Second)
        channel2 <- "Response from channel 2"
    }()

    select {
    case message := <-channel1:
        fmt.Println(message)

    case message := <-channel2:
        fmt.Println(message)
    }
}
```

Output:

```text
Response from channel 2
```

The second goroutine finishes first, so the second case executes.

A `select` executes only one ready case. After executing that case, it exits unless it is placed inside a loop.

---

## What if multiple channels are ready?

If multiple cases are ready at the same time, Go chooses one of them pseudo-randomly.

```go
package main

import "fmt"

func main() {
    channel1 := make(chan string, 1)
    channel2 := make(chan string, 1)

    channel1 <- "Message from channel 1"
    channel2 <- "Message from channel 2"

    select {
    case message := <-channel1:
        fmt.Println(message)

    case message := <-channel2:
        fmt.Println(message)
    }
}
```

Both channels already contain a value.

Therefore, both cases are ready. Go selects one of them.

You should not depend on the order in which ready cases are selected.

---

## Using `default` with `select`

A `default` case makes a `select` non-blocking.

```go
select {
case message := <-channel:
    fmt.Println("Received:", message)

default:
    fmt.Println("No message available")
}
```

If the channel is not ready, the `default` case runs immediately.

### Example

```go
package main

import "fmt"

func main() {
    messages := make(chan string)

    select {
    case message := <-messages:
        fmt.Println("Received:", message)

    default:
        fmt.Println("No message is currently available")
    }
}
```

Output:

```text
No message is currently available
```

The program does not wait because the `default` case is available.

### Be careful with `default`

This code can consume unnecessary CPU:

```go
for {
    select {
    case message := <-messages:
        fmt.Println(message)

    default:
        // Repeats continuously without waiting
    }
}
```

Because `default` runs immediately, the loop may execute millions of times.

If you do not need non-blocking behavior, leave out the `default` case.

---

## Using `select` inside a loop

A single `select` handles only one event.

To continuously process events, place it inside a loop.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    messages := make(chan string)

    go func() {
        for i := 1; i <= 3; i++ {
            messages <- fmt.Sprintf("Message %d", i)
            time.Sleep(500 * time.Millisecond)
        }

        close(messages)
    }()

    for {
        select {
        case message, ok := <-messages:
            if !ok {
                fmt.Println("Channel closed")
                return
            }

            fmt.Println("Received:", message)
        }
    }
}
```

Output:

```text
Received: Message 1
Received: Message 2
Received: Message 3
Channel closed
```

The `ok` value tells us whether the channel is still open:

```go
message, ok := <-messages
```

- `ok == true`: a value was received normally.
- `ok == false`: the channel was closed and has no remaining values.

A receive operation on a closed channel is always ready. If you do not check `ok`, a loop may repeatedly receive the channel’s zero value forever.

---

# 2. Understanding Timeouts

## What is a timeout?

A timeout means:

> Wait for an operation only for a limited amount of time.

For example, suppose your backend calls a payment service.

Normally, the payment service responds within two seconds. If it takes 30 seconds, you probably do not want your request handler to wait indefinitely.

You can stop waiting after a specific duration.

---

## Timeout using `time.After`

`time.After` returns a channel that receives a value after the given duration.

```go
timeoutChannel := time.After(2 * time.Second)
```

It can be used inside `select`.

```go
select {
case result := <-resultChannel:
    fmt.Println("Result:", result)

case <-time.After(2 * time.Second):
    fmt.Println("Operation timed out")
}
```

---

## Complete timeout example

```go
package main

import (
    "fmt"
    "time"
)

func fetchData(resultChannel chan<- string) {
    time.Sleep(3 * time.Second)
    resultChannel <- "Data received"
}

func main() {
    resultChannel := make(chan string)

    go fetchData(resultChannel)

    select {
    case result := <-resultChannel:
        fmt.Println(result)

    case <-time.After(2 * time.Second):
        fmt.Println("Request timed out")
    }
}
```

Output:

```text
Request timed out
```

The worker takes three seconds, but the main goroutine waits for only two seconds.

---

## Important: A timeout does not automatically stop the worker

In the previous example, the caller stops waiting after two seconds, but `fetchData` is still running.

This distinction is extremely important:

- A timeout stops the caller from waiting.
- Cancellation tells the worker to stop.

There is another problem in the previous code. After three seconds, the worker tries to send a result through an unbuffered channel:

```go
resultChannel <- "Data received"
```

But nobody is receiving from it anymore. The goroutine becomes blocked.

One possible improvement is to use a buffered channel:

```go
resultChannel := make(chan string, 1)
```

Now the worker can send one result even if the caller has stopped waiting.

However, a better design is usually to make the worker cancellation-aware.

---

## Timeout using `time.NewTimer`

For more control, create a timer explicitly.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    resultChannel := make(chan string, 1)

    go func() {
        time.Sleep(1 * time.Second)
        resultChannel <- "Operation completed"
    }()

    timer := time.NewTimer(2 * time.Second)
    defer timer.Stop()

    select {
    case result := <-resultChannel:
        fmt.Println(result)

    case <-timer.C:
        fmt.Println("Operation timed out")
    }
}
```

`time.NewTimer` is useful when you may need to stop or reset the timer.

Always stop a timer when it is no longer needed:

```go
defer timer.Stop()
```

---

## Avoid repeatedly creating timers inside hot loops

This code creates a new timer on every loop iteration:

```go
for {
    select {
    case message := <-messages:
        fmt.Println(message)

    case <-time.After(5 * time.Second):
        fmt.Println("No messages received")
    }
}
```

For occasional operations, this may be acceptable. In performance-sensitive or long-running loops, explicitly manage a timer instead.

---

# 3. Understanding Cancellation

## What is cancellation?

Cancellation is a way to tell running work:

> Your result is no longer needed. Stop what you are doing.

Common reasons for cancellation include:

- The client disconnected.
- The HTTP request was cancelled.
- The request timed out.
- One operation failed, so related operations should stop.
- The application is shutting down.
- The user manually cancelled the operation.

In Go backend applications, cancellation is usually implemented using the `context` package.

---

# 4. Understanding `context.Context`

A `context.Context` carries request-scoped information such as:

- Cancellation signals
- Deadlines
- Timeouts
- Request-scoped values

The most important method for cancellation is:

```go
ctx.Done()
```

`ctx.Done()` returns a channel.

When the context is cancelled, this channel is closed. A goroutine can listen to it using `select`.

```go
select {
case <-ctx.Done():
    fmt.Println("Operation cancelled")

case result := <-resultChannel:
    fmt.Println("Result:", result)
}
```

You can find the reason for cancellation using:

```go
ctx.Err()
```

Common errors are:

```go
context.Canceled
context.DeadlineExceeded
```

---

# 5. Manual Cancellation with `context.WithCancel`

`context.WithCancel` creates a child context and a cancellation function.

```go
ctx, cancel := context.WithCancel(context.Background())
```

Calling the cancellation function cancels the context:

```go
cancel()
```

---

## Manual cancellation example

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
            fmt.Println("Worker is processing...")
            time.Sleep(500 * time.Millisecond)
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    go worker(ctx)

    time.Sleep(2 * time.Second)

    fmt.Println("Main: cancelling the worker")
    cancel()

    time.Sleep(500 * time.Millisecond)
}
```

Possible output:

```text
Worker is processing...
Worker is processing...
Worker is processing...
Worker is processing...
Main: cancelling the worker
Worker stopped: context canceled
```

### What happens here?

1. `context.Background()` creates the root context.
2. `context.WithCancel` creates a cancellable context.
3. The context is passed to the worker.
4. The worker monitors `ctx.Done()`.
5. The main goroutine calls `cancel()`.
6. `ctx.Done()` is closed.
7. The worker detects cancellation and returns.

---

## Better cancellation-aware worker

Using `default` with `time.Sleep` works for demonstration, but a ticker is cleaner for repeated work.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context) {
    ticker := time.NewTicker(500 * time.Millisecond)
    defer ticker.Stop()

    for {
        select {
        case <-ticker.C:
            fmt.Println("Worker is processing...")

        case <-ctx.Done():
            fmt.Println("Worker stopped:", ctx.Err())
            return
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()

    go worker(ctx)

    time.Sleep(2 * time.Second)

    fmt.Println("Main: cancelling the worker")
    cancel()

    time.Sleep(500 * time.Millisecond)
}
```

---

# 6. Timeout with `context.WithTimeout`

`context.WithTimeout` automatically cancels a context after a specified duration.

```go
ctx, cancel := context.WithTimeout(
    context.Background(),
    2*time.Second,
)
defer cancel()
```

Even though the timeout automatically cancels the context, you should still call `cancel()`.

This immediately releases resources associated with the context when the work finishes before the timeout.

---

## Complete `context.WithTimeout` example

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func fetchData(ctx context.Context) (string, error) {
    select {
    case <-time.After(3 * time.Second):
        return "Data received", nil

    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(
        context.Background(),
        2*time.Second,
    )
    defer cancel()

    result, err := fetchData(ctx)
    if err != nil {
        fmt.Println("Failed to fetch data:", err)
        return
    }

    fmt.Println(result)
}
```

Output:

```text
Failed to fetch data: context deadline exceeded
```

The function needs three seconds, but the context allows only two seconds.

When the deadline is reached:

1. The context is automatically cancelled.
2. `ctx.Done()` is closed.
3. The cancellation case becomes ready.
4. `fetchData` returns `context deadline exceeded`.

---

# 7. Making Long-Running Work Cancellable

A worker should periodically check whether its context has been cancelled.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func processItems(ctx context.Context, items []int) error {
    for _, item := range items {
        select {
        case <-ctx.Done():
            return ctx.Err()

        default:
        }

        fmt.Println("Processing item:", item)
        time.Sleep(500 * time.Millisecond)
    }

    return nil
}

func main() {
    ctx, cancel := context.WithTimeout(
        context.Background(),
        2*time.Second,
    )
    defer cancel()

    items := []int{1, 2, 3, 4, 5, 6}

    err := processItems(ctx, items)
    if err != nil {
        fmt.Println("Processing stopped:", err)
        return
    }

    fmt.Println("All items processed")
}
```

Possible output:

```text
Processing item: 1
Processing item: 2
Processing item: 3
Processing item: 4
Processing stopped: context deadline exceeded
```

The context is checked before processing each item.

Cancellation is cooperative. Go does not forcibly kill the goroutine. The goroutine must observe the cancellation signal and return.

---

# 8. Cancellation While Sending to a Channel

A goroutine can become stuck while attempting to send to a channel.

Consider:

```go
results <- result
```

If nobody receives the result, the goroutine may block forever.

Make the send operation cancellation-aware:

```go
select {
case results <- result:
    // Result delivered successfully

case <-ctx.Done():
    // Caller no longer wants the result
    return
}
```

---

## Complete example

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func calculate(ctx context.Context, results chan<- int) {
    time.Sleep(2 * time.Second)

    result := 100

    select {
    case results <- result:
        fmt.Println("Result sent successfully")

    case <-ctx.Done():
        fmt.Println("Result not sent:", ctx.Err())
        return
    }
}

func main() {
    ctx, cancel := context.WithTimeout(
        context.Background(),
        1*time.Second,
    )
    defer cancel()

    results := make(chan int)

    go calculate(ctx, results)

    select {
    case result := <-results:
        fmt.Println("Received:", result)

    case <-ctx.Done():
        fmt.Println("Main stopped waiting:", ctx.Err())
    }

    time.Sleep(1500 * time.Millisecond)
}
```

Possible output:

```text
Main stopped waiting: context deadline exceeded
Result not sent: context deadline exceeded
```

Both the caller and worker observe the same cancellation signal.

---

# 9. Cancellation While Receiving from a Channel

Receiving from a channel can also block.

Instead of doing this:

```go
value := <-jobs
```

Use `select` when the receive operation should be cancellable:

```go
select {
case value := <-jobs:
    fmt.Println("Received:", value)

case <-ctx.Done():
    return
}
```

---

## Worker-pool style example

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func worker(ctx context.Context, jobs <-chan int) {
    for {
        select {
        case job, ok := <-jobs:
            if !ok {
                fmt.Println("No more jobs")
                return
            }

            fmt.Println("Processing job:", job)
            time.Sleep(500 * time.Millisecond)

        case <-ctx.Done():
            fmt.Println("Worker cancelled:", ctx.Err())
            return
        }
    }
}

func main() {
    ctx, cancel := context.WithCancel(context.Background())

    jobs := make(chan int)

    go worker(ctx, jobs)

    go func() {
        for i := 1; i <= 10; i++ {
            select {
            case jobs <- i:
            case <-ctx.Done():
                return
            }
        }

        close(jobs)
    }()

    time.Sleep(2 * time.Second)

    fmt.Println("Cancelling all work")
    cancel()

    time.Sleep(500 * time.Millisecond)
}
```

The worker waits for either:

- A new job
- A cancellation signal

---

# 10. Context Cancellation Propagation

Contexts form a parent-child tree.

```text
Parent context
    |
    +-- Child context
            |
            +-- Grandchild context
```

When a parent context is cancelled, all its children are also cancelled.

Cancelling a child does not cancel its parent.

---

## Example

```go
package main

import (
    "context"
    "fmt"
)

func main() {
    parentCtx, cancelParent :=
        context.WithCancel(context.Background())

    childCtx, cancelChild :=
        context.WithCancel(parentCtx)

    defer cancelParent()
    defer cancelChild()

    cancelParent()

    <-childCtx.Done()

    fmt.Println("Child context:", childCtx.Err())
}
```

Output:

```text
Child context: context canceled
```

This is useful when one incoming HTTP request starts several related operations. When the request is cancelled, all operations using its context can stop.

---

# 11. Practical Backend Example

Suppose an HTTP endpoint needs data from a user service.

The service call should stop when:

- The client disconnects.
- The incoming request is cancelled.
- The operation exceeds two seconds.

```go
package main

import (
    "context"
    "encoding/json"
    "errors"
    "net/http"
    "time"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

func fetchUser(ctx context.Context, userID int) (User, error) {
    select {
    case <-time.After(3 * time.Second):
        return User{
            ID:   userID,
            Name: "Charan",
        }, nil

    case <-ctx.Done():
        return User{}, ctx.Err()
    }
}

func userHandler(w http.ResponseWriter, r *http.Request) {
    // The request context is cancelled if the client disconnects.
    requestCtx := r.Context()

    // Add our own two-second timeout.
    ctx, cancel := context.WithTimeout(
        requestCtx,
        2*time.Second,
    )
    defer cancel()

    user, err := fetchUser(ctx, 1)
    if err != nil {
        switch {
        case errors.Is(err, context.DeadlineExceeded):
            http.Error(
                w,
                "User service timed out",
                http.StatusGatewayTimeout,
            )

        case errors.Is(err, context.Canceled):
            // The client may already have disconnected.
            http.Error(
                w,
                "Request cancelled",
                http.StatusRequestTimeout,
            )

        default:
            http.Error(
                w,
                "Internal server error",
                http.StatusInternalServerError,
            )
        }

        return
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(user)
}

func main() {
    http.HandleFunc("/user", userHandler)

    http.ListenAndServe(":8080", nil)
}
```

In this example:

1. `r.Context()` represents the lifetime of the HTTP request.
2. `context.WithTimeout` adds a two-second deadline.
3. The context is passed to `fetchUser`.
4. `fetchUser` waits for completion or cancellation.
5. Because the simulated service takes three seconds, the timeout occurs first.
6. The handler returns HTTP `504 Gateway Timeout`.

In a real application, pass this context to context-aware libraries:

```go
request, err := http.NewRequestWithContext(
    ctx,
    http.MethodGet,
    serviceURL,
    nil,
)
```

For a SQL query:

```go
rows, err := db.QueryContext(
    ctx,
    "SELECT id, name FROM users",
)
```

These APIs can stop their underlying work when the context is cancelled.

---

# 12. Combining `select`, Timeout, and Cancellation

Here is an example that combines all three concepts.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func performTask(
    ctx context.Context,
    resultChannel chan<- string,
) {
    taskTimer := time.NewTimer(3 * time.Second)
    defer taskTimer.Stop()

    select {
    case <-taskTimer.C:
        result := "Task completed successfully"

        select {
        case resultChannel <- result:
        case <-ctx.Done():
            return
        }

    case <-ctx.Done():
        fmt.Println("Worker stopped:", ctx.Err())
        return
    }
}

func main() {
    ctx, cancel := context.WithTimeout(
        context.Background(),
        2*time.Second,
    )
    defer cancel()

    resultChannel := make(chan string)

    go performTask(ctx, resultChannel)

    select {
    case result := <-resultChannel:
        fmt.Println(result)

    case <-ctx.Done():
        fmt.Println("Main stopped waiting:", ctx.Err())
    }
}
```

Output:

```text
Main stopped waiting: context deadline exceeded
```

The task needs three seconds, but the context expires after two seconds.

The same context informs both the caller and worker that the operation should stop.

---

# 13. `select` with Multiple Results

Sometimes you need results from multiple goroutines rather than only the first result.

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func fetchUser(ctx context.Context) (string, error) {
    select {
    case <-time.After(1 * time.Second):
        return "User data", nil

    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func fetchOrders(ctx context.Context) (string, error) {
    select {
    case <-time.After(1500 * time.Millisecond):
        return "Order data", nil

    case <-ctx.Done():
        return "", ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(
        context.Background(),
        2*time.Second,
    )
    defer cancel()

    userChannel := make(chan string, 1)
    orderChannel := make(chan string, 1)

    go func() {
        user, err := fetchUser(ctx)
        if err == nil {
            userChannel <- user
        }
    }()

    go func() {
        orders, err := fetchOrders(ctx)
        if err == nil {
            orderChannel <- orders
        }
    }()

    var userData string
    var orderData string

    for received := 0; received < 2; {
        select {
        case userData = <-userChannel:
            received++

        case orderData = <-orderChannel:
            received++

        case <-ctx.Done():
            fmt.Println("Operation failed:", ctx.Err())
            return
        }
    }

    fmt.Println("User:", userData)
    fmt.Println("Orders:", orderData)
}
```

Output:

```text
User: User data
Orders: Order data
```

The loop continues until both results have been received or the context is cancelled.

---

# 14. Common Mistakes

## Mistake 1: Forgetting to call `cancel`

Incorrect:

```go
ctx, _ := context.WithTimeout(
    context.Background(),
    2*time.Second,
)
```

Correct:

```go
ctx, cancel := context.WithTimeout(
    context.Background(),
    2*time.Second,
)
defer cancel()
```

Calling `cancel` releases context-related resources as soon as they are no longer needed.

---

## Mistake 2: Creating a context but not passing it

```go
ctx, cancel := context.WithTimeout(
    context.Background(),
    2*time.Second,
)
defer cancel()

result := fetchData()
```

The context cannot cancel `fetchData` because it was never passed to the function.

Correct:

```go
result, err := fetchData(ctx)
```

---

## Mistake 3: Ignoring `ctx.Done()`

Simply accepting a context is not enough:

```go
func worker(ctx context.Context) {
    for {
        doSomething()
    }
}
```

The worker must observe the cancellation signal:

```go
func worker(ctx context.Context) {
    for {
        select {
        case <-ctx.Done():
            return

        default:
            doSomething()
        }
    }
}
```

---

## Mistake 4: Using context as a global variable

Avoid:

```go
var applicationContext context.Context
```

Pass context explicitly:

```go
func processRequest(ctx context.Context) error {
    return fetchData(ctx)
}
```

This keeps cancellation associated with the correct operation.

---

## Mistake 5: Storing context inside a struct

Usually avoid:

```go
type Service struct {
    ctx context.Context
}
```

Prefer passing it to each operation:

```go
type Service struct{}

func (s *Service) FetchUser(
    ctx context.Context,
    userID int,
) error {
    return nil
}
```

Contexts usually represent individual requests or operations, not the entire lifetime of a service object.

---

## Mistake 6: Passing `nil` as a context

Do not do this:

```go
fetchData(nil)
```

If you do not yet have a meaningful context, use:

```go
context.Background()
```

---

## Mistake 7: Assuming timeout kills a goroutine

A timeout does not forcibly terminate a goroutine.

The goroutine must listen for cancellation:

```go
select {
case <-ctx.Done():
    return

case result := <-resultChannel:
    process(result)
}
```

Cancellation in Go is cooperative.

---

## Mistake 8: Using `context.Value` for normal function parameters

Avoid:

```go
ctx = context.WithValue(ctx, "userID", 100)
```

when the value is simply a required argument.

Prefer:

```go
func fetchUser(ctx context.Context, userID int) {
}
```

`context.Value` is intended for request-scoped metadata that crosses API boundaries, such as:

- Request IDs
- Trace IDs
- Authentication metadata

---

# 15. Best Practices

## Accept context as the first parameter

```go
func FetchUser(
    ctx context.Context,
    userID int,
) (User, error) {
    // ...
}
```

The common convention is:

```go
func FunctionName(ctx context.Context, otherArguments...)
```

---

## Do not replace an existing request context unnecessarily

If a function already receives a context, derive from it:

```go
ctx, cancel := context.WithTimeout(
    parentCtx,
    2*time.Second,
)
defer cancel()
```

Do not disconnect the operation from its parent by starting with `context.Background()`:

```go
// Usually incorrect inside request-related code
ctx, cancel := context.WithTimeout(
    context.Background(),
    2*time.Second,
)
```

---

## Always clean up timers and tickers

```go
timer := time.NewTimer(time.Second)
defer timer.Stop()
```

```go
ticker := time.NewTicker(time.Second)
defer ticker.Stop()
```

---

## Make blocking channel operations cancellable

For sending:

```go
select {
case results <- result:
case <-ctx.Done():
    return ctx.Err()
}
```

For receiving:

```go
select {
case job := <-jobs:
    process(job)

case <-ctx.Done():
    return ctx.Err()
}
```

---

## Return the context error

```go
case <-ctx.Done():
    return ctx.Err()
```

This lets the caller distinguish between:

```go
context.Canceled
```

and:

```go
context.DeadlineExceeded
```

Use `errors.Is` when checking them:

```go
if errors.Is(err, context.DeadlineExceeded) {
    fmt.Println("The operation timed out")
}

if errors.Is(err, context.Canceled) {
    fmt.Println("The operation was cancelled")
}
```

---

## The sender should usually close the channel

A receiver normally should not close a channel because it cannot know whether another sender will attempt to use it.

```go
go func() {
    defer close(results)

    for _, item := range items {
        results <- process(item)
    }
}()
```

Closing a channel means:

> No more values will ever be sent through this channel.

---

# 16. Quick Comparison

| Concept | Purpose | Common Go mechanism |
|---|---|---|
| `select` | Wait for multiple channel operations | `select { case ... }` |
| Timeout | Limit how long an operation may take | `time.After`, `time.NewTimer`, `context.WithTimeout` |
| Cancellation | Tell ongoing work to stop | `context.WithCancel`, `ctx.Done()` |
| Deadline | Stop work at a specific time | `context.WithDeadline` |
| Cancellation reason | Determine why work stopped | `ctx.Err()` |

---

# 17. Simple Mental Model

Imagine ordering food at a restaurant.

## `select`

You are waiting for multiple possible events:

- Your food becomes ready.
- Your friend calls you.
- The restaurant announces a problem.

You respond to whichever event happens first.

## Timeout

You decide:

> If my food does not arrive within 30 minutes, I will stop waiting.

## Cancellation

Your friend tells you:

> We need to leave immediately.

You inform the restaurant that the order is no longer needed.

In Go:

```go
select {
case food := <-foodChannel:
    fmt.Println("Food received:", food)

case <-ctx.Done():
    fmt.Println("Stopped waiting:", ctx.Err())
}
```

---

# 18. Final Combined Example

```go
package main

import (
    "context"
    "errors"
    "fmt"
    "time"
)

func fetchData(
    ctx context.Context,
    resultChannel chan<- string,
) {
    timer := time.NewTimer(3 * time.Second)
    defer timer.Stop()

    select {
    case <-timer.C:
        result := "Backend response"

        select {
        case resultChannel <- result:
            fmt.Println("Worker sent the result")

        case <-ctx.Done():
            fmt.Println(
                "Worker could not send result:",
                ctx.Err(),
            )
        }

    case <-ctx.Done():
        fmt.Println("Worker cancelled:", ctx.Err())
    }
}

func main() {
    ctx, cancel := context.WithTimeout(
        context.Background(),
        2*time.Second,
    )
    defer cancel()

    resultChannel := make(chan string)

    go fetchData(ctx, resultChannel)

    select {
    case result := <-resultChannel:
        fmt.Println("Received:", result)

    case <-ctx.Done():
        switch {
        case errors.Is(ctx.Err(), context.DeadlineExceeded):
            fmt.Println("The operation timed out")

        case errors.Is(ctx.Err(), context.Canceled):
            fmt.Println("The operation was manually cancelled")
        }
    }

    // Only for this demonstration, so the worker's message is visible.
    time.Sleep(500 * time.Millisecond)
}
```

Output:

```text
The operation timed out
Worker cancelled: context deadline exceeded
```

This example demonstrates all three concepts:

- `select` waits for multiple possible events.
- `context.WithTimeout` sets a time limit.
- `ctx.Done()` communicates cancellation to the worker.
- `ctx.Err()` explains why the operation stopped.

---

# Summary

## `select`

Use `select` when a goroutine needs to wait for multiple channel operations.

```go
select {
case result := <-results:
    fmt.Println(result)

case <-ctx.Done():
    return ctx.Err()
}
```

## Timeout

Use a timeout when an operation must not take longer than a specific duration.

```go
ctx, cancel := context.WithTimeout(
    parentCtx,
    2*time.Second,
)
defer cancel()
```

## Cancellation

Pass the context to every related function and make blocking operations watch `ctx.Done()`.

```go
func worker(ctx context.Context) error {
    select {
    case <-ctx.Done():
        return ctx.Err()

    case result := <-performWork():
        fmt.Println(result)
        return nil
    }
}
```

The central idea is:

> `select` chooses between concurrent events, a timeout limits waiting time, and cancellation helps the entire operation stop cleanly when its result is no longer required.
