# Thread Pool Design Pattern

A **thread pool** is a concurrency pattern that starts a group of reusable workers to process tasks. Each worker takes a task, runs it, and then picks up another. The pool size limits how many tasks can run at once.

In Go, this is usually called a **worker pool** because the workers are goroutines. A pool of three worker goroutines can process up to three jobs at once, but it does not create or reserve three OS threads. The Go runtime schedules goroutines onto OS threads.

**How it relates to Producer–Consumer:** Producer–Consumer describes how work is handed off through a queue. A worker pool defines how many consumers process that work concurrently. The two patterns often appear together.

## When to Use

- When many independent jobs need to be processed concurrently.
- When you want to limit pressure on a database, an external API, or another shared resource.
- When jobs arrive in bursts and can wait in a bounded queue.
- When request handling and job processing can happen separately.

**Backend example:** An API queues report jobs for three workers. If each worker generates one report at a time, at most three reports are being generated. This limits concurrent work, but it does not enforce a requests-per-second limit.

## Components

| Component | Responsibility | Go example |
|---|---|---|
| **Workers** | Receive jobs and process them one at a time. | Three `worker()` goroutines |
| **Job Queue** | Holds jobs that are waiting for a worker. | Buffered `jobs` channel |
| **Pool Coordination** | Starts workers and coordinates shutdown. | `main()` and `sync.WaitGroup` |
| **Result Channel** | Sends completed results back to the caller, when results are needed. | `results` channel |

## Working

1. Start a fixed number of workers. Each waits for a job.
2. Submit jobs to the shared channel. Available workers receive jobs directly; queued jobs wait for a worker to become free.
3. If every worker is busy and the buffer fills, the next send blocks. This is **backpressure**.
4. After a worker finishes a job, it sends the result and returns to receive another.
5. When submission ends, close `jobs`. Workers finish the queued jobs and exit. Once all workers have exited, close `results` so the result loop can stop.

With **three workers and a queue capacity of two**, up to three jobs can be held by workers and two more can wait in the queue. The worker count and the queue capacity control different things.

## Pros

- **Bounds concurrency:** Limits how many tasks can use a shared resource at once.
- **Reuses workers:** Each worker processes multiple jobs over its lifetime.
- **Absorbs short bursts:** The buffer gives incoming work somewhere to wait.
- **Centralizes lifecycle management:** Worker startup and shutdown can be managed in one place.

## Cons

- **Adds queue delay:** Jobs wait when all workers are busy, and submission can block when the queue fills.
- **Long-running jobs occupy workers:** If all workers are stuck on slow jobs, queued jobs cannot progress.
- **Results may arrive out of order:** A later job can finish before an earlier one.
- **Requires shutdown coordination:** Closing a channel too early, or leaving workers blocked on sends, can break the pool.
- **Requires tuning:** Too few workers can limit throughput; too many can overload the resource the pool was meant to protect.

## Sample Implementation in Golang

This example starts **three workers**, submits **five jobs**, and uses a jobs channel with a **buffer of two**. Each worker doubles its input and returns a result that identifies the original job.

```go
package main

import (
	"fmt"
	"sync"
	"time"
)

type Result struct {
	JobID    int
	WorkerID int
	Value    int
}

func worker(id int, jobs <-chan int, results chan<- Result, wg *sync.WaitGroup) {
	defer wg.Done()

	for job := range jobs {
		time.Sleep(30 * time.Millisecond) // Simulate processing.

		results <- Result{
			JobID:    job,
			WorkerID: id,
			Value:    job * 2,
		}
	}
}

func main() {
	const (
		numWorkers = 3
		numJobs    = 5
	)

	jobs := make(chan int, 2)
	results := make(chan Result)

	var wg sync.WaitGroup
	wg.Add(numWorkers)

	for id := 1; id <= numWorkers; id++ {
		go worker(id, jobs, results, &wg)
	}

	// Submit jobs while main reads results.
	go func() {
		defer close(jobs)

		for job := 1; job <= numJobs; job++ {
			jobs <- job
		}
	}()

	// Only close results after every worker has stopped sending.
	go func() {
		wg.Wait()
		close(results)
	}()

	for result := range results {
		fmt.Printf("Worker %d: job %d -> %d\n",
			result.WorkerID, result.JobID, result.Value)
	}

	fmt.Println("All jobs processed")
}
```

**One possible output:**

```text
Worker 2: job 2 -> 4
Worker 1: job 1 -> 2
Worker 3: job 3 -> 6
Worker 2: job 4 -> 8
Worker 1: job 5 -> 10
All jobs processed
```

### Step-by-Step Explanation

1. **Start three reusable workers.** `wg.Add(numWorkers)` counts the workers before they start. Each worker loops over `jobs`, processes one job at a time, and calls `wg.Done()` when it exits.

2. **Submit jobs while collecting results.** A separate goroutine sends jobs while `main()` reads results. If `main()` tried to submit every job before reading any results, workers could block sending results while `main()` blocks sending to a full jobs queue. Neither side could make progress.

3. **Close `jobs` when submission ends.** The submitting goroutine owns this close. Workers can still receive jobs already in the buffer. Their loops end once the channel is closed and empty.

4. **Close `results` after all workers exit.** `wg.Wait()` ensures no worker can still send a result. Closing `results` lets the loop in `main()` exit, so `All jobs processed` always prints last.

5. **Use job IDs to match results.** Worker assignment and result order can vary between runs. `JobID` tells the caller which input produced each result.

In a backend service, size the pool around the resource it protects. Also decide what to do when the queue is full: wait, stop waiting when the request is cancelled, or reject the submission. A bounded queue limits waiting work in the channel; that decision determines how overload reaches the caller.



and