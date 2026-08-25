# Latency, Throughput, and Availability in System Design

Imagine you have developed a food-delivery application. Thousands of users open the application, search for restaurants, place orders, and make payments.

To understand how well your backend works, you need to answer three important questions:

1. How quickly does the system respond? — **Latency**
2. How much traffic can the system handle? — **Throughput**
3. How reliably can users access the system? — **Availability**

---

# 1. Latency

## What is latency?

**Latency is the amount of time a system takes to complete a single operation or request.**

```text
User sends request
        ↓
Server processes request
        ↓
User receives response
```

If the complete operation takes 200 milliseconds:

```text
Latency = 200 ms
```

Latency tells us how long one request waits before receiving a response.

---

## Simple analogy

Imagine ordering coffee at a coffee shop.

You place your order at 10:00 AM and receive your coffee at 10:05 AM.

```text
Latency = 5 minutes
```

Latency focuses on the waiting time experienced by one customer or one request.

---

## Backend API example

Suppose a user sends the following request:

```http
GET /api/restaurants
```

The request might involve these operations:

```text
Request travels through network     20 ms
Authentication                      10 ms
Database query                      80 ms
Application processing              20 ms
Response travels through network    20 ms
-----------------------------------------
Total latency                      150 ms
```

Therefore:

```text
Request latency = 150 ms
```

---

## Sources of latency

A backend request may experience latency in multiple places:

```text
Client
  ↓
Network
  ↓
Load balancer
  ↓
Application server
  ↓
Database
  ↓
External service
  ↓
Response reaches client
```

Examples include:

| Operation | Example latency |
|---|---:|
| Reading from memory | Very low |
| Redis lookup | 1–5 ms |
| Database query | 10–100 ms |
| Calling another service | 50–500 ms |
| Uploading a large file | Several seconds |

> These are illustrative values. Actual latency depends on the infrastructure, network, workload, query complexity, and system design.

---

## Average latency

Suppose your API receives five requests:

```text
Request 1: 100 ms
Request 2: 110 ms
Request 3: 120 ms
Request 4: 130 ms
Request 5: 3000 ms
```

The average latency is:

```text
Average latency
= Total request time / Number of requests

= (100 + 110 + 120 + 130 + 3000) / 5

= 692 ms
```

However, this number can be misleading.

Four requests completed quickly, while one request took three seconds. The average does not clearly communicate this situation.

Because of this, production systems also monitor latency percentiles.

---

## Latency percentiles

Common latency percentiles include:

- **p50**
- **p90**
- **p95**
- **p99**

Consider these measurements:

```text
p50 latency = 100 ms
p95 latency = 300 ms
p99 latency = 2 seconds
```

### p50 latency

```text
p50 = 100 ms
```

This means that 50% of requests completed within 100 ms.

It is also known as the **median latency**.

### p95 latency

```text
p95 = 300 ms
```

This means that 95% of requests completed within 300 ms.

The slowest 5% took longer than 300 ms.

### p99 latency

```text
p99 = 2 seconds
```

This means that 99% of requests completed within two seconds.

The slowest 1% took longer than two seconds.

---

## Tail latency

The latency experienced by the slowest requests is called **tail latency**.

For example:

```text
Most requests:   100–200 ms
Slowest 5%:      More than 500 ms
Slowest 1%:      More than 2 seconds
```

Even if most requests are fast, the slowest requests still affect real users.

This is why production systems commonly monitor:

```text
p50
p95
p99
```

For user-facing APIs, p95 and p99 latency are usually more informative than average latency.

---

## What causes high latency?

Common causes include:

- Slow database queries
- Missing database indexes
- Too many calls between services
- Slow external APIs
- Network delays
- Lock contention
- Insufficient CPU or memory
- Garbage collection
- Database connection-pool exhaustion
- Large request or response payloads
- Overloaded servers
- Cold caches
- Long queues
- Slow disk operations

---

## How can latency be reduced?

Common techniques include:

- Cache frequently accessed data
- Add proper database indexes
- Optimize database queries
- Reduce unnecessary service calls
- Use database connection pooling
- Process non-essential work asynchronously
- Place servers closer to users
- Use a CDN for static content
- Compress large responses
- Add appropriate timeouts
- Scale overloaded services
- Reduce response payload size

---

## Example: synchronous processing

Suppose an order API performs these operations:

```text
1. Save order
2. Send confirmation email
3. Generate invoice
4. Notify restaurant
5. Return response
```

If every operation runs synchronously:

```text
Save order                 100 ms
Send email                 500 ms
Generate invoice           300 ms
Notify restaurant          200 ms
--------------------------------
Total latency             1100 ms
```

The user waits 1.1 seconds.

Instead, non-essential operations can be processed asynchronously:

```text
1. Save order
2. Add background jobs to a queue
3. Return response
```

Now the request may take:

```text
Save order                 100 ms
Add job to queue            20 ms
--------------------------------
Total latency              120 ms
```

The email, invoice, and notification are processed later by background workers.

This reduces user-facing latency.

---

# 2. Throughput

## What is throughput?

**Throughput is the amount of work a system completes during a particular period.**

It tells us how much traffic the system can process.

Throughput is commonly measured using:

- Requests per second — RPS
- Queries per second — QPS
- Transactions per second — TPS
- Messages per second
- Jobs per minute
- Megabytes per second

---

## Example

Suppose your backend successfully processes 5,000 requests in 10 seconds.

```text
Throughput
= Completed requests / Total time

= 5000 / 10

= 500 requests per second
```

Therefore:

```text
Throughput = 500 RPS
```

---

## Simple analogy

Consider a coffee shop.

One customer receives coffee in five minutes:

```text
Latency = 5 minutes
```

The coffee shop prepares 100 coffees in one hour:

```text
Throughput = 100 coffees per hour
```

Therefore:

```text
Latency
= Time required to prepare one order

Throughput
= Number of orders completed during a period
```

---

## Latency and throughput are different

Consider two restaurants:

| Restaurant | Time per order | Orders completed per hour |
|---|---:|---:|
| Restaurant A | 5 minutes | 20 |
| Restaurant B | 8 minutes | 100 |

Restaurant A has lower latency because it completes one order faster.

Restaurant B has higher throughput because it completes more total orders per hour.

Restaurant B may accomplish this by processing many orders concurrently.

The same idea applies to backend systems.

---

## Concurrency and throughput

A server usually handles multiple requests concurrently.

Therefore, throughput is not always simply:

```text
Throughput = 1 / Latency
```

Concurrency also matters.

A useful approximation is:

```text
Concurrent requests
≈ Throughput × Average latency
```

Suppose:

```text
Throughput      = 1,000 requests/second
Average latency = 200 ms
```

First, convert milliseconds to seconds:

```text
200 ms = 0.2 seconds
```

Now calculate concurrency:

```text
Concurrent requests
≈ 1000 × 0.2

≈ 200 requests
```

Approximately 200 requests may be in progress at any given time.

---

## What happens when traffic exceeds capacity?

Suppose your backend can safely process:

```text
Maximum capacity = 2,000 requests/second
```

But it receives:

```text
Incoming traffic = 3,000 requests/second
```

The system cannot immediately process all incoming requests. The extra requests begin waiting in queues.

This may produce the following sequence:

```text
Traffic exceeds system capacity
              ↓
Request queues increase
              ↓
Requests wait longer
              ↓
Latency increases
              ↓
Requests time out
              ↓
Clients retry requests
              ↓
Retries create more traffic
              ↓
System becomes overloaded
              ↓
Availability decreases
```

This is why latency often increases sharply when a system approaches its maximum capacity.

---

## Throughput under different loads

A service may behave like this during a load test:

```text
1,000 RPS → Stable
2,000 RPS → Stable
3,000 RPS → Stable
4,000 RPS → Latency starts increasing
5,000 RPS → Timeouts begin
6,000 RPS → Service becomes unstable
```

Although the service may accept 6,000 RPS for a short time, that does not mean 6,000 RPS is its safe capacity.

A safer operating capacity might be:

```text
Safe capacity = 3,000–3,500 RPS
```

This leaves headroom for:

- Sudden traffic spikes
- Server failures
- Background jobs
- Database maintenance
- Slow external dependencies

---

## How can throughput be increased?

Common techniques include:

- Add more application instances
- Use a load balancer
- Increase request concurrency
- Use asynchronous processing
- Introduce message queues
- Optimize database queries
- Use database read replicas
- Partition or shard data
- Cache frequently accessed data
- Batch operations
- Reduce unnecessary processing
- Upgrade CPU, memory, disk, or network capacity
- Reuse database and network connections

---

## Example: horizontal scaling

Suppose one application instance can process:

```text
1 server = 1,000 requests/second
```

If traffic increases, you can add more instances:

```text
3 servers ≈ 3,000 requests/second
```

The architecture may look like this:

```text
                     ┌── Server 1
Users → Load Balancer ├── Server 2
                     └── Server 3
```

The load balancer distributes requests across the available instances.

This is called **horizontal scaling**.

> Capacity may not scale perfectly because all instances could still depend on the same database or downstream service.

---

# 3. Availability

## What is availability?

**Availability is the percentage of time a system is operational and capable of successfully serving requests.**

A basic formula is:

```text
Availability
= Uptime / Total time × 100
```

Suppose a service is operational for 23 hours and unavailable for one hour:

```text
Availability
= 23 / 24 × 100

= 95.83%
```

Therefore:

```text
Availability = 95.83%
```

---

## Simple analogy

Imagine an ATM.

The ATM may process a transaction quickly when it is working. But if it is frequently out of service, it has poor availability.

For the ATM:

```text
Latency:
How long does one transaction take?

Throughput:
How many transactions can it process per minute?

Availability:
How often is it operational and usable?
```

---

## Availability is not only about the server running

Suppose this endpoint is running:

```http
POST /api/payments
```

However, 40% of payment requests return:

```http
HTTP/1.1 500 Internal Server Error
```

Technically, the server process is running. But from the user's perspective, the payment service is not properly available.

A more practical measurement is:

```text
Availability
= Successful valid requests / Total valid requests × 100
```

Suppose the system receives 10,000 valid requests and 9,990 complete successfully:

```text
Availability
= 9990 / 10000 × 100

= 99.9%
```

What counts as successful must be defined for your system. It might mean:

- The response is not a server error
- The request completes before a timeout
- The response contains correct data
- The critical user operation completes successfully

---

## The nines of availability

Availability is commonly described using the number of nines:

| Availability | Approximate downtime per year |
|---:|---:|
| 99% | 3 days 15 hours |
| 99.9% | 8 hours 46 minutes |
| 99.99% | 52 minutes 36 seconds |
| 99.999% | 5 minutes 15 seconds |

The difference between 99.9% and 99.99% may look small.

However:

```text
99.9%  → Almost 9 hours of downtime per year
99.99% → Less than 1 hour of downtime per year
```

Each additional nine becomes significantly more difficult and expensive to achieve.

---

## Single point of failure

Suppose your application runs on only one server:

```text
Users
  ↓
Application server
  ↓
Database
```

If the application server fails, the entire system becomes unavailable.

The server is a **single point of failure**.

A single point of failure is a component whose failure can make the entire system unavailable.

Examples include:

- A single application server
- A single database instance
- A single load balancer
- A single network connection
- A single availability zone

---

## How is high availability achieved?

### 1. Multiple application instances

Instead of using one application instance, run multiple instances:

```text
                     ┌── Server 1
Users → Load Balancer ├── Server 2
                     └── Server 3
```

If Server 1 fails, the load balancer can send traffic to Servers 2 and 3.

This improves availability.

---

### 2. Health checks

A load balancer regularly checks whether each application instance is healthy.

For example:

```http
GET /health
```

A healthy response might be:

```http
HTTP/1.1 200 OK
```

If an instance becomes unhealthy, the load balancer stops sending traffic to it:

```text
Server 1 → Healthy   → Receive traffic
Server 2 → Unhealthy → Do not receive traffic
Server 3 → Healthy   → Receive traffic
```

---

### 3. Database replication

A primary database can have one or more replicas:

```text
              ┌── Read Replica 1
Primary DB ───┼── Read Replica 2
              └── Read Replica 3
```

Read queries can be distributed across replicas.

Depending on the architecture, a replica may also be promoted if the primary database fails.

---

### 4. Multiple availability zones

Running every service in one data centre or availability zone creates a large failure risk.

Instead, services can be distributed across multiple availability zones:

```text
Region
├── Availability Zone A
│   ├── Application Server 1
│   └── Database Replica
│
└── Availability Zone B
    ├── Application Server 2
    └── Database Replica
```

If Availability Zone A fails, Availability Zone B may continue serving users.

---

### 5. Timeouts

A service should not wait forever for another service.

For example, in Go:

```go
client := &http.Client{
    Timeout: 2 * time.Second,
}
```

If the downstream service does not respond within two seconds, the request is cancelled.

Timeouts prevent resources from remaining blocked indefinitely.

---

### 6. Retries

A retry attempts a failed operation again:

```text
First request → Failed
Wait
Second request → Successful
```

Retries are useful for temporary failures such as:

- Short network interruptions
- Temporary service unavailability
- Brief database connection failures

However, retries should be limited.

A common approach is:

```text
Attempt 1
Wait 100 ms

Attempt 2
Wait 200 ms

Attempt 3
Wait 400 ms
```

This is called **exponential backoff**.

Randomness called **jitter** is often added so that all clients do not retry at exactly the same time.

Uncontrolled retries can overload an already struggling system.

---

### 7. Circuit breakers

A circuit breaker temporarily stops calls to a consistently failing service.

When the dependency is healthy:

```text
Service healthy
      ↓
Circuit closed
      ↓
Requests are allowed
```

When too many requests fail:

```text
Many failures
      ↓
Circuit opens
      ↓
Requests are rejected quickly
      ↓
Dependency gets time to recover
```

After some time, a few test requests are allowed. If they succeed, normal traffic resumes.

Circuit breakers prevent one failing service from bringing down other services.

---

### 8. Graceful degradation

A system should continue providing important functionality even if a non-critical service fails.

Suppose an e-commerce system has a recommendation service.

Normally:

```text
User opens homepage
        ↓
Display personalized recommendations
```

If the recommendation service fails:

```text
User opens homepage
        ↓
Display popular products
```

The user can still use the application.

This is called **graceful degradation**.

---

# Understanding all three together

Consider a ticket-booking system during a popular event.

It receives 10,000 requests every second.

| Metric | Question answered | Example |
|---|---|---|
| Latency | How long does one request take? | p95 is 300 ms |
| Throughput | How many requests can it handle? | 10,000 RPS |
| Availability | How reliably does it work? | 99.99% |

A good system should be:

- Fast enough for users
- Capable of handling expected traffic
- Reliable when failures occur

However, improving one metric does not automatically improve all the others.

---

# Relationships and trade-offs

## 1. Higher traffic can increase latency

As throughput demand approaches system capacity, resources become busy:

```text
More traffic
     ↓
More waiting for CPU, connections and locks
     ↓
Higher latency
     ↓
More timeouts
     ↓
More failed requests
     ↓
Lower availability
```

For example, suppose a database connection pool contains 100 connections.

If 500 requests simultaneously require a database connection:

```text
First 100 requests → Receive connections
Remaining requests → Wait for connections
```

The waiting time increases request latency.

If requests wait too long, they time out and availability decreases.

---

## 2. Replication can improve reliability but increase write latency

Suppose a payment must be written to three database replicas before the system returns success:

```text
Client
  ↓
Primary database
  ├── Replica 1 acknowledgement
  └── Replica 2 acknowledgement
```

Waiting for multiple acknowledgements improves data durability.

However, the request must wait for the replicas:

```text
More synchronous acknowledgements
                ↓
Better durability and failure tolerance
                ↓
Potentially higher write latency
```

---

## 3. Batching can increase throughput but also increase latency

Suppose a worker collects 100 messages and inserts them into the database together.

Without batching:

```text
100 messages
    ↓
100 database calls
```

With batching:

```text
100 messages
    ↓
1 batch database call
```

Batching reduces database calls and increases throughput.

However, the first message may need to wait until the batch fills:

```text
Batching
  → Higher throughput
  → Potentially higher latency per message
```

---

## 4. Caching can improve latency and throughput

Without caching:

```text
Every request
     ↓
Database
```

With caching:

```text
Request
   ↓
Cache lookup
   ├── Cache hit  → Fast response
   └── Cache miss → Database → Update cache → Response
```

Caching:

- Reduces database load
- Reduces latency
- Increases throughput
- Helps the system handle more requests

However, caching introduces challenges such as:

- Stale data
- Cache invalidation
- Cache failures
- Memory usage
- Cache stampedes

---

## 5. Retries can improve availability but increase load

Retries may help a request succeed after a temporary failure.

However:

```text
Service becomes slow
        ↓
Clients retry
        ↓
Service receives more traffic
        ↓
Service becomes even slower
```

This is called a **retry storm**.

Retries should therefore use:

- Retry limits
- Exponential backoff
- Jitter
- Timeouts
- Circuit breakers
- Idempotency where required

---

# Practical Go backend example

Suppose you have a Go backend endpoint:

```http
GET /api/products/:id
```

You observe these metrics:

```text
Traffic:       2,000 requests/second
p50 latency:   40 ms
p95 latency:   180 ms
p99 latency:   1.5 seconds
Availability:  99.8%
```

Let us understand what these values mean.

---

## Understanding latency

```text
p50 = 40 ms
```

Half of the requests complete within 40 ms.

```text
p95 = 180 ms
```

95% of the requests complete within 180 ms.

```text
p99 = 1.5 seconds
```

99% of requests complete within 1.5 seconds, while the slowest 1% take even longer.

The large difference between p95 and p99 indicates a tail-latency problem.

Possible causes include:

- Slow database queries
- Database connection-pool exhaustion
- Cache misses
- Slow downstream services
- Server overload
- Lock contention
- Garbage collection

---

## Understanding throughput

```text
Throughput = 2,000 requests/second
```

The service currently processes 2,000 requests every second.

However, this does not tell you the service's maximum safe capacity. You need a load test.

A load-test result might look like this:

```text
1,000 RPS → Stable latency
2,000 RPS → Stable latency
3,000 RPS → Stable latency
4,000 RPS → Latency begins increasing
5,000 RPS → Timeouts begin
6,000 RPS → Service becomes unstable
```

The safe capacity may be approximately:

```text
3,000–3,500 RPS
```

This leaves capacity for sudden traffic spikes or server failures.

---

## Understanding availability

```text
Availability = 99.8%
```

This means some requests are failing, timing out, or the service is experiencing downtime.

You should investigate:

- Application crashes
- Failed deployments
- Database outages
- Dependency timeouts
- Incorrect health checks
- Traffic spikes
- CPU or memory exhaustion
- Database connection exhaustion

---

# Measuring these metrics

Useful production-backend metrics include:

```text
Request count
Request duration
Successful request count
Error count
Active requests
Queue size
Timeout count
CPU usage
Memory usage
Database connection-pool usage
```

Prometheus-style metrics might include:

```text
http_requests_total
http_request_duration_seconds
http_requests_in_flight
http_request_errors_total
```

Grafana dashboards can display:

- Requests per second
- p50 latency
- p95 latency
- p99 latency
- Error rate
- Availability
- CPU usage
- Memory usage
- Database connection-pool usage
- Queue size

---

# SLI, SLO, and SLA

Latency and availability are commonly used when defining reliability targets.

## SLI — Service Level Indicator

An SLI is the actual measured value.

Examples:

```text
Current availability = 99.95%
Current p95 latency  = 250 ms
```

## SLO — Service Level Objective

An SLO is an internal reliability target.

Examples:

```text
Availability should be at least 99.9%.

95% of requests should complete within 300 ms.
```

## SLA — Service Level Agreement

An SLA is a formal agreement with customers.

It may state:

```text
The service will provide 99.9% monthly availability.
```

If the SLA is violated, the company may provide service credits or compensation.

An easy way to remember these terms is:

```text
SLI = What are we currently measuring?

SLO = What target do we want to achieve?

SLA = What have we formally promised customers?
```

---

# Quick comparison

| Concept | Meaning | Common units | Example |
|---|---|---|---|
| Latency | Time required for one operation | Milliseconds or seconds | 200 ms per request |
| Throughput | Work completed during a period | RPS, QPS or TPS | 5,000 RPS |
| Availability | Percentage of time the service works | Percentage | 99.99% |

---

# Easy way to remember

Use the restaurant analogy:

```text
Latency:
How long does one customer wait for food?

Throughput:
How many orders can the restaurant complete per hour?

Availability:
How often is the restaurant open and capable of serving customers?
```

For a backend API:

```text
Latency
= Time taken per request

Throughput
= Requests processed per unit of time

Availability
= Percentage of time requests can be served successfully
```

---

# Final example

Suppose your API:

- Responds in 100 ms
- Handles 5,000 requests per second
- Works successfully 99.99% of the time

Then:

```text
Latency      = 100 ms
Throughput   = 5,000 RPS
Availability = 99.99%
```

These metrics answer different questions, but they are closely connected:

```text
Traffic exceeds capacity
          ↓
Requests begin waiting
          ↓
Latency increases
          ↓
Requests time out
          ↓
Availability decreases
```

Therefore, when designing a backend system, you should evaluate **latency, throughput, and availability together**.
