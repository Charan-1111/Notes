# Load Balancer Explained with Go

## 1. What Is a Load Balancer?

A **load balancer** is a component that sits between clients and backend servers. It receives incoming requests and distributes them across multiple servers.

Imagine a bank with three counters. If everyone goes to Counter 1, it becomes overloaded while the other counters remain idle. A bank employee who directs each customer to a suitable counter acts like a load balancer.

```text
                         ┌──> Backend Server 1
Client ──> Load Balancer ├──> Backend Server 2
                         └──> Backend Server 3
```

Instead of calling a backend server directly, a client sends a request to the load balancer:

```http
GET https://api.example.com/users
```

The load balancer might forward it to:

```text
http://server-2:8080/users
```

The client does not need to know which backend server handled the request.

---

## 2. Why Do We Need a Load Balancer?

Suppose an application initially has one server:

```text
Client ──> Server
```

This works while traffic is low. If the server can handle 1,000 requests per second but the application receives 3,000 requests per second, it may become slow, consume all its resources, reject requests, or crash.

We can add more servers:

```text
Server 1: 1,000 requests/second
Server 2: 1,000 requests/second
Server 3: 1,000 requests/second
```

A load balancer gives clients one entry point and distributes the work among those servers.

It provides:

- Traffic distribution
- Better availability
- Horizontal scalability
- Failure handling
- A single entry point for clients

---

## 3. Request Flow

Assume that the backend servers run at:

```text
Server 1: http://localhost:8081
Server 2: http://localhost:8082
Server 3: http://localhost:8083
```

The load balancer runs at:

```text
http://localhost:8080
```

When a client sends:

```http
GET http://localhost:8080/users
```

the load balancer:

1. Receives the request.
2. Selects a healthy backend server.
3. Forwards the request to that server.
4. Receives the backend response.
5. Returns the response to the client.

```text
Client
  |
  | GET /users
  v
Load Balancer :8080
  |
  | GET /users
  v
Backend Server 2 :8082
  |
  | 200 OK
  v
Load Balancer
  |
  | 200 OK
  v
Client
```

---

## 4. Main Responsibilities

### 4.1 Distribute Traffic

```text
Request 1 ──> Server 1
Request 2 ──> Server 2
Request 3 ──> Server 3
Request 4 ──> Server 1
```

This prevents one server from receiving all the traffic.

### 4.2 Detect Unhealthy Servers

If Server 2 crashes, a load balancer with health checking stops sending requests to it:

```text
                         ┌──> Server 1: Healthy
Load Balancer ───────────┤
                         ├──> Server 2: Unhealthy — skipped
                         └──> Server 3: Healthy
```

### 4.3 Hide Backend Details

Clients use one public address:

```text
https://api.example.com
```

The actual servers may use private addresses:

```text
10.0.1.10:8080
10.0.1.11:8080
10.0.1.12:8080
```

Clients do not need to know when servers are added, removed, restarted, or replaced.

### 4.4 Improve Availability

If one backend fails, the remaining healthy servers can continue processing requests. In production, multiple load-balancer instances are also used so that the load balancer itself does not become a single point of failure.

---

## 5. Load-Balancing Algorithms

### 5.1 Round Robin

Round Robin selects servers one after another:

```text
Request 1 ──> Server A
Request 2 ──> Server B
Request 3 ──> Server C
Request 4 ──> Server A
```

It works well when servers have similar capacities and requests require roughly similar amounts of work.

Its limitation is that it treats a 10 ms request and a 20-second request equally.

### 5.2 Weighted Round Robin

More powerful servers receive a larger weight and therefore more requests.

```text
Server A weight: 2
Server B weight: 1

Request 1 ──> Server A
Request 2 ──> Server A
Request 3 ──> Server B
```

### 5.3 Least Connections

The next request goes to the server with the fewest active connections.

```text
Server A: 20 active connections
Server B: 7 active connections  <- selected
Server C: 12 active connections
```

This is useful for long-running requests, file uploads, report generation, and WebSocket connections.

### 5.4 Least Response Time

The load balancer prefers the server that is responding fastest, often while also considering its number of active connections.

### 5.5 Random Selection

A backend is selected randomly. It is simple and can work reasonably well when the system receives many requests.

### 5.6 IP Hash

The load balancer hashes the client's IP address:

```text
serverIndex = hash(clientIP) % numberOfServers
```

This can send requests from the same IP to the same server, but shared public IPs, changing mobile IPs, and changes to the backend list make it imperfect.

---

## 6. Layer 4 and Layer 7 Load Balancing

### Layer 4

A Layer 4 load balancer works mainly with IP addresses and TCP or UDP ports. It does not need to understand HTTP paths, headers, or cookies.

It is fast and is useful for TCP services, databases, game servers, and other non-HTTP protocols.

### Layer 7

A Layer 7 load balancer understands application protocols such as HTTP. It can inspect paths, methods, headers, cookies, hostnames, and query parameters.

For example:

```text
/users/*    ──> User Service
/orders/*   ──> Order Service
/payments/* ──> Payment Service
```

It can support content-based routing, authentication, rate limiting, TLS termination, and HTTP-specific observability.

---

## 7. Load Balancer vs Reverse Proxy

A **reverse proxy** receives a request and forwards it to a backend server:

```text
Client ──> Reverse Proxy ──> Backend
```

A load balancer is normally a reverse proxy that can select from multiple backend servers:

```text
                          ┌──> Backend 1
Client ──> Load Balancer ─┼──> Backend 2
                          └──> Backend 3
```

---

## 8. Building a Simple Load Balancer in Go

We will build a small Layer 7 HTTP load balancer using:

- `net/http`
- `net/http/httputil`
- Round Robin selection

This is an educational implementation, not a complete production load balancer.

### Project Structure

```text
load-balancer/
├── backend/
│   └── main.go
├── loadbalancer/
│   └── main.go
└── go.mod
```

---

## 9. Backend Servers

Create `backend/main.go`:

```go
package main

import (
	"encoding/json"
	"flag"
	"log"
	"net/http"
)

type Response struct {
	Server string `json:"server"`
	Path   string `json:"path"`
}

func main() {
	port := flag.String("port", "8081", "port used by the backend server")
	name := flag.String("name", "server-1", "backend server name")
	flag.Parse()

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		response := Response{
			Server: *name,
			Path:   r.URL.Path,
		}

		w.Header().Set("Content-Type", "application/json")

		if err := json.NewEncoder(w).Encode(response); err != nil {
			log.Printf("failed to encode response: %v", err)
		}
	})

	address := ":" + *port
	log.Printf("%s listening on %s", *name, address)

	if err := http.ListenAndServe(address, nil); err != nil {
		log.Fatal(err)
	}
}
```

### Step-by-Step Explanation

#### Step 1: Accept command-line configuration

```go
port := flag.String("port", "8081", "port used by the backend server")
name := flag.String("name", "server-1", "backend server name")
flag.Parse()
```

This lets us run the same program as several backend servers:

```bash
go run ./backend -port=8081 -name=server-1
go run ./backend -port=8082 -name=server-2
go run ./backend -port=8083 -name=server-3
```

#### Step 2: Register an HTTP handler

```go
http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
```

Registering `/` allows the handler to receive paths such as `/users`, `/products`, and `/orders/100`.

#### Step 3: Create the response

```go
response := Response{
	Server: *name,
	Path:   r.URL.Path,
}
```

The response tells us which backend processed the request and which path it received.

#### Step 4: Return JSON

```go
w.Header().Set("Content-Type", "application/json")
json.NewEncoder(w).Encode(response)
```

Example response:

```json
{
  "server": "server-2",
  "path": "/users"
}
```

#### Step 5: Start the server

```go
http.ListenAndServe(address, nil)
```

This starts an HTTP server on the configured port.

---

## 10. Understanding Round Robin

For three servers, their indexes are `0`, `1`, and `2`. We select an index using:

```text
counter % numberOfServers
```

```text
0 % 3 = 0 ──> Server 1
1 % 3 = 1 ──> Server 2
2 % 3 = 2 ──> Server 3
3 % 3 = 0 ──> Server 1
```

The modulo operator `%` makes the selection wrap back to the first server.

Because Go processes HTTP requests concurrently, the counter must be updated safely.

---

## 11. Complete Go Load Balancer

Create `loadbalancer/main.go`:

```go
package main

import (
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"sync/atomic"
)

type Backend struct {
	URL   *url.URL
	Proxy *httputil.ReverseProxy
}

type LoadBalancer struct {
	backends []*Backend
	counter  uint64
}

func NewLoadBalancer(addresses []string) (*LoadBalancer, error) {
	backends := make([]*Backend, 0, len(addresses))

	for _, address := range addresses {
		backendURL, err := url.Parse(address)
		if err != nil {
			return nil, err
		}

		proxy := httputil.NewSingleHostReverseProxy(backendURL)

		backends = append(backends, &Backend{
			URL:   backendURL,
			Proxy: proxy,
		})
	}

	return &LoadBalancer{backends: backends}, nil
}

func (lb *LoadBalancer) nextBackend() *Backend {
	position := atomic.AddUint64(&lb.counter, 1)
	index := (position - 1) % uint64(len(lb.backends))

	return lb.backends[index]
}

func (lb *LoadBalancer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	backend := lb.nextBackend()

	log.Printf(
		"forwarding %s %s to %s",
		r.Method,
		r.URL.Path,
		backend.URL,
	)

	backend.Proxy.ServeHTTP(w, r)
}

func main() {
	addresses := []string{
		"http://localhost:8081",
		"http://localhost:8082",
		"http://localhost:8083",
	}

	loadBalancer, err := NewLoadBalancer(addresses)
	if err != nil {
		log.Fatal(err)
	}

	log.Println("load balancer listening on :8080")

	if err := http.ListenAndServe(":8080", loadBalancer); err != nil {
		log.Fatal(err)
	}
}
```

---

## 12. Step-by-Step Load Balancer Explanation

### Step 1: Represent a Backend

```go
type Backend struct {
	URL   *url.URL
	Proxy *httputil.ReverseProxy
}
```

Each backend contains its address and a reverse proxy that forwards requests to that address.

### Step 2: Store Load-Balancer State

```go
type LoadBalancer struct {
	backends []*Backend
	counter  uint64
}
```

The load balancer stores all available backends and the counter used for Round Robin selection.

### Step 3: Create the Backends

`NewLoadBalancer` loops through the address strings, parses each address into a URL, creates a reverse proxy, and stores both in a `Backend` value.

```go
backendURL, err := url.Parse(address)
proxy := httputil.NewSingleHostReverseProxy(backendURL)
```

### Step 4: Select the Next Backend

```go
position := atomic.AddUint64(&lb.counter, 1)
index := (position - 1) % uint64(len(lb.backends))
```

Go HTTP requests run concurrently in separate goroutines. Using `lb.counter++` could create a data race, so `atomic.AddUint64` increments the counter safely.

We subtract one because the first atomic increment changes the counter from `0` to `1`; subtracting makes the first selected index `0`.

### Step 5: Implement `http.Handler`

```go
func (lb *LoadBalancer) ServeHTTP(w http.ResponseWriter, r *http.Request)
```

This method means that `LoadBalancer` implements Go's `http.Handler` interface and can be passed directly to `http.ListenAndServe`.

### Step 6: Forward the Request

```go
backend := lb.nextBackend()
backend.Proxy.ServeHTTP(w, r)
```

The reverse proxy forwards the incoming request and copies the backend's response back to the client. It handles the request body, headers, status code, and response body for us.

---

## 13. Running the Example

Initialize the module:

```bash
go mod init example.com/load-balancer
```

Run the three backend servers in separate terminals:

```bash
go run ./backend -port=8081 -name=server-1
go run ./backend -port=8082 -name=server-2
go run ./backend -port=8083 -name=server-3
```

Run the load balancer in another terminal:

```bash
go run ./loadbalancer
```

Send several requests:

```bash
curl http://localhost:8080/users
curl http://localhost:8080/users
curl http://localhost:8080/users
curl http://localhost:8080/users
```

Expected output:

```json
{"server":"server-1","path":"/users"}
{"server":"server-2","path":"/users"}
{"server":"server-3","path":"/users"}
{"server":"server-1","path":"/users"}
```

---

## 14. Health Checks

The basic implementation continues selecting a server even if that server has stopped. A production load balancer needs health checks.

### Active Health Check

The load balancer periodically calls an endpoint such as:

```http
GET /health
```

A healthy server returns `200 OK`. An unhealthy server may return `500`, refuse the connection, time out, or not respond.

A sensible policy might be:

```text
3 consecutive failures  ──> Mark unhealthy
2 consecutive successes ──> Mark healthy again
```

### Passive Health Check

The load balancer also observes real requests. If a backend repeatedly returns errors or timeouts, it can temporarily stop sending traffic to it.

### Liveness and Readiness

- **Liveness** asks: Is the application process alive?
- **Readiness** asks: Is the application ready to handle traffic?

If readiness fails, the load balancer should stop sending new requests to that instance.

---

## 15. Timeouts

A load balancer should not wait forever for a backend. Important timeouts include:

- Connection timeout
- Response-header timeout
- Idle connection timeout
- Overall request timeout

Without timeouts, slow backends can consume goroutines, memory, file descriptors, and network connections until the load balancer itself becomes unhealthy.

---

## 16. Retries

If one backend cannot be reached, the load balancer may retry against another backend:

```text
Request ──> Server 1 ──> Connection failure
        └─> Server 2 ──> Success
```

Retries must be limited and used carefully.

A `GET` request is normally safe to retry. Retrying a request such as `POST /payments` can process the payment twice if the first attempt succeeded but its response was lost.

Write operations can use an idempotency key:

```http
Idempotency-Key: payment-12345
```

The backend remembers the key and prevents the same operation from being performed twice.

---

## 17. Sticky Sessions

Sticky sessions send a user's requests to the same backend:

```text
User A ──> Server 1
User A ──> Server 1
User A ──> Server 1
```

They can be implemented using cookies, IP hashing, or session identifiers.

A more scalable design stores sessions in shared storage such as Redis. Then every backend can handle every request, making the servers stateless and easier to scale.

---

## 18. TLS Termination

A load balancer can receive encrypted HTTPS traffic, decrypt it, and forward it to a backend:

```text
Client ── HTTPS ──> Load Balancer ── HTTP/HTTPS ──> Backend
```

This centralizes certificate management. In security-sensitive environments, communication between the load balancer and backends should also use TLS.

---

## 19. Preserving Client Information

Because the backend receives the connection from the load balancer, it may see the load balancer's IP instead of the client's IP.

Proxies commonly add:

```http
X-Forwarded-For: 203.0.113.10
X-Forwarded-Proto: https
X-Forwarded-Host: api.example.com
```

Only trust these headers when they come from a trusted proxy, because a direct client can send fake forwarding headers.

---

## 20. Connection Draining

When deploying a new server version:

1. Stop sending new requests to the old instance.
2. Allow its current requests to finish.
3. Shut it down after completion or a deadline.
4. Start the new version.
5. Wait until it becomes ready.
6. Add it back to the backend pool.

This avoids terminating requests during deployments.

---

## 21. Load Balancer vs API Gateway

| Load Balancer | API Gateway |
|---|---|
| Distributes traffic | Manages and controls APIs |
| Performs health checks | Handles authentication and authorization |
| Selects backend instances | Applies API-specific rate limits |
| Improves availability | Can transform requests and responses |
| Can operate at Layer 4 or Layer 7 | Usually operates at Layer 7 |

A system can use both. The API gateway applies API policies, while the load balancer distributes traffic across service instances.

---

## 22. Load Balancing in Kubernetes

Several instances of an application normally run as pods. A Kubernetes `Service` provides a stable address and distributes traffic among matching pods.

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Ingress Controller
   |
   v
Kubernetes Service
   |
   ├──> Pod 1
   ├──> Pod 2
   └──> Pod 3
```

Load balancing can therefore happen at multiple levels.

---

## 23. Metrics to Monitor

A load balancer should expose metrics such as:

- Total requests
- Requests per second
- Active connections
- Response latency
- Responses grouped by status code
- Healthy and unhealthy backend counts
- Request count per backend
- Backend connection failures
- Timeout count
- Retry count

Prometheus can collect these metrics and Grafana can display them. Structured logs should record fields such as method, path, backend, response status, and duration.

---

## 24. What the Basic Implementation Is Missing

The example teaches the fundamental flow but does not yet include:

- Active health checks
- Passive failure detection
- Backend timeouts
- Retry rules
- Circuit breaking
- Rate limiting
- TLS termination
- Dynamic backend discovery
- Graceful shutdown
- Connection draining
- Authentication
- Metrics
- Structured logging
- Protection against malformed or oversized requests
- Multiple load-balancer instances

---

## 25. Practical Learning Path

Build the project in stages:

1. Start three backend servers.
2. Forward every request to one backend.
3. Add multiple backends.
4. Implement Round Robin.
5. Make concurrent selection race-safe.
6. Add active health checks.
7. Skip unhealthy backends.
8. Add connection and response timeouts.
9. Add limited retries for safe requests.
10. Add structured logging.
11. Add Prometheus metrics.
12. Add graceful shutdown and connection draining.
13. Load-test the application and observe its behavior.
14. Run multiple load-balancer instances.

---

## 26. Final Mental Model

> A load balancer is a traffic manager that receives requests, selects an appropriate healthy backend server, forwards the requests, and returns the backend responses to clients.

```text
Receive request
      |
      v
Find healthy backends
      |
      v
Select one using an algorithm
      |
      v
Forward the request
      |
      v
Return the backend response
```

A basic load balancer distributes requests. A production load balancer must also handle failures, timeouts, retries, health checks, security, monitoring, and its own high availability.
