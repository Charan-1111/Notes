# Producer–Consumer Design Pattern

The **Producer–Consumer pattern** is a concurrency pattern that separates the component that **creates work** from the component that **processes it**. A queue between them holds work until a consumer is ready.

For example, an API can enqueue a report-generation job, and a background worker can generate the report later. The API and the worker agree on what a job contains; the API does not need to know which worker will process it.

A buffer lets producers get ahead temporarily. If a producer keeps creating work faster than consumers can process it, the buffer eventually fills. In Go, a send to a full buffered channel blocks until space becomes available. This is **backpressure**.

## When to Use

- When one part of the system creates jobs, messages, or events that another part processes.
- When work can run in the background, such as generating reports or resizing images.
- When work arrives in short bursts and consumers need time to catch up.
- When you want to control how many jobs run at once by limiting the number of consumers.

**Backend example:** A Go API accepts a report request and queues a job containing its report ID. A worker reads the job and generates the report. The API can respond once the job is accepted instead of waiting for the report to finish. If accepted jobs must survive a process restart, use a durable queue; a Go channel only holds jobs in memory.

## Components

| Component | Responsibility | Go example |
|---|---|---|
| **Producer** | Creates jobs and sends them to the queue. | `produce()` |
| **Consumer** | Receives jobs and processes them. | `consume()` |
| **Buffer / Queue** | Holds jobs waiting to be received. | `make(chan int, 2)` |

The buffer capacity controls how many jobs can **wait in the channel**. It does not count jobs a consumer has already received and is still processing.

## Working

| Channel state | What happens |
|---|---|
| **Buffer has room** | The producer sends a job without waiting for the consumer to finish its current job. |
| **Buffer is full** | The next send waits until a consumer receives a job. |
| **Channel is open and empty** | The consumer waits for a job. |
| **Channel is closed with jobs remaining** | The consumer can still receive the queued jobs. |
| **Channel is closed and empty** | The consumer’s `range` loop exits. |

The producer closes the channel **after its last send**. If several producers share the channel, a coordinator can use a `sync.WaitGroup` to wait for all of them and then close it once. Sending to a closed channel panics.

## Pros

- **Decouples job creation from processing:** Producers and consumers can change independently as long as they agree on the job format.
- **Absorbs short bursts:** A bounded buffer lets a producer enqueue some work while the consumer is busy.
- **Limits concurrent processing:** A fixed number of consumers can cap how many jobs run at once.
- **Can improve throughput:** Multiple consumers can process independent jobs concurrently when the workload benefits.

## Cons

- **A full queue can delay producers:** A buffer buys time during a burst; it does not fix a sustained gap between production and processing.
- **Shutdown needs coordination:** Closing too early can panic. Forgetting to close can leave a consumer waiting. If a consumer may stop early, producers need a way to cancel blocked sends.
- **Completion order can change:** Multiple consumers can receive jobs in queue order but finish them in a different order.
- **Adds concurrency complexity:** The channel safely coordinates sends and receives. Any separate mutable state shared by workers still needs its own protection.

## Sample Implementation in Golang

This example runs one producer in a separate goroutine and one consumer in the main goroutine. The consumer simulates slower work, and the channel can hold up to two waiting jobs.

```go
package main

import (
	"fmt"
	"time"
)

// Producer: send-only access to the channel.
func produce(jobs chan<- int) {
	defer close(jobs)

	for job := 1; job <= 5; job++ {
		fmt.Printf("Trying to enqueue job %d\n", job)
		jobs <- job // Blocks if the buffer is full.
	}
}

// Consumer: receive-only access to the channel.
func consume(jobs <-chan int) {
	for job := range jobs {
		time.Sleep(40 * time.Millisecond) // Simulate processing.
		fmt.Printf("Finished job %d\n", job)
	}
}

func main() {
	jobs := make(chan int, 2)

	go produce(jobs)
	consume(jobs)

	fmt.Println("All jobs processed")
}
```

**One possible output:**

```text
Trying to enqueue job 1
Trying to enqueue job 2
Trying to enqueue job 3
Trying to enqueue job 4
Finished job 1
Trying to enqueue job 5
Finished job 2
Finished job 3
Finished job 4
Finished job 5
All jobs processed
```

### Step-by-Step Explanation

1. **Create a bounded buffer.** `make(chan int, 2)` creates a channel with room for two queued jobs. If the consumer is processing job `1` while jobs `2` and `3` are waiting, the producer cannot enqueue job `4` until the consumer receives job `2`.

2. **Make channel direction explicit.** `chan<- int` lets `produce()` send jobs, and `<-chan int` lets `consume()` receive them. Both functions use the same channel.

3. **Run the producer and consumer concurrently.** `go produce(jobs)` starts a goroutine. `consume(jobs)` runs in the main goroutine, so `main()` waits there while the consumer finishes the work. The sleep only simulates processing; it is not needed for synchronization.

4. **Distinguish a send attempt from a successful send.** The producer prints `Trying to enqueue` **before** `jobs <- job`. Seeing that line means the producer reached the send, not that the job is already queued. The next line from either goroutine depends on scheduling.

5. **Close the channel and drain it.** `defer close(jobs)` runs when the producer returns after sending all five jobs. The consumer’s `range` continues until the channel is both closed and empty.

6. **Know what order is guaranteed.** With one producer sending `1` through `5` and one consumer processing one job at a time, the `Finished job` lines are always in that order. The producer and consumer logs can interleave differently. `All jobs processed` prints last because `consume()` has returned.
