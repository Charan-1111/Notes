# Observer Design Pattern

The **Observer pattern** is a behavioral design pattern that defines a one-to-many relationship between objects.

When a **subject** publishes a change, it notifies its registered **observers**. Each observer decides how to react.

The subject depends on a shared observer interface rather than concrete observer types.

**Simple example:**

- A news publisher is the subject.
- Alice and Bob are observers.
- Both receive published news.
- If Alice unsubscribes, only Bob receives future news.

Notifications trigger observer code; they do not automatically synchronize every observer’s state.

## When to Use

- When several components need to react to the same change or event.
- When the publisher should not depend on specific subscriber implementations.
- When subscriptions need to change at runtime.
- When updates should trigger reactions instead of requiring repeated polling.

**Backend example:**

After an order is created, interested observers might:

- Update application metrics.
- Refresh an in-memory dashboard.
- Record an audit event.

In-process Observer does not provide durable delivery. Critical workflows may require additional mechanisms such as persistent queues.

## Components

| Component | Responsibility | News example |
|---|---|---|
| **Subject** | Defines subscription and notification operations. | `Subject` interface. |
| **Observer** | Defines how notifications are received. | `Observer` interface. |
| **Concrete Subject** | Stores subscriptions and publishes updates. | `NewsPublisher`. |
| **Concrete Observer** | Implements a reaction to notifications. | `Subscriber`. |

An observer does not always need a reference to the subject.

Two common notification approaches are:

- **Push:** The subject passes data to the observer.
- **Pull:** The observer retrieves data from the subject after notification.

Our example uses push: `Update(news string)` receives the news directly.

## Working

1. The client registers observers with the publisher.
2. Each registration returns a subscription ID.
3. The publisher receives new news.
4. It calls `Update()` on the registered observers.
5. The client can unsubscribe using a subscription ID.

Notifications in this example are **synchronous**: each observer finishes before the next one is called.

The Observer pattern does not automatically create goroutines or background tasks.

## Pros

- **Loose coupling:** The publisher knows the observer interface, not individual subscriber implementations.
- **Broadcasting:** One event can notify multiple interested components.
- **Dynamic subscriptions:** Observers can subscribe and unsubscribe at runtime.
- **Extensibility:** New observer types can be added without changing publisher logic.
- **Avoids polling:** Observers receive notifications when the publisher emits them.

## Cons

- **Cascading updates:** Observers may trigger more events, creating complicated flows.
- **Retained subscriptions:** A publisher holding observer references can keep them reachable longer than intended.
- **Notification overhead:** Frequent events and many observers increase work.
- **Slow observers:** Synchronous observers delay publishing and later observers.
- **Ordering dependencies:** Correctness becomes fragile if observers rely on an unspecified order.
- **Error-handling decisions:** The application must decide what happens when an observer fails.

Go’s garbage collector can collect unreachable cycles. The retention concern is a still-reachable publisher holding subscriptions that are no longer needed.

## Sample Implementation in Golang

This example uses subscription IDs and notifies observers in registration order.

It is intended for use by one goroutine.

```go
package main

import "fmt"

// Observer interface.
type Observer interface {
	Update(news string)
}

// Subject interface.
type Subject interface {
	Register(observer Observer) int
	Unregister(id int) bool
	NotifyAll()
}

// A registration connects an ID to an observer.
type subscription struct {
	id       int
	observer Observer
}

// Concrete subject.
type NewsPublisher struct {
	subscriptions []subscription
	nextID        int
	news          string
}

func (p *NewsPublisher) Register(observer Observer) int {
	p.nextID++

	p.subscriptions = append(p.subscriptions, subscription{
		id:       p.nextID,
		observer: observer,
	})

	return p.nextID
}

func (p *NewsPublisher) Unregister(id int) bool {
	for i, sub := range p.subscriptions {
		if sub.id != id {
			continue
		}

		// Shift later subscriptions left, preserving order.
		copy(p.subscriptions[i:], p.subscriptions[i+1:])

		// Clear the unused slot to release its observer reference.
		last := len(p.subscriptions) - 1
		p.subscriptions[last] = subscription{}
		p.subscriptions = p.subscriptions[:last]

		return true
	}

	return false
}

func (p *NewsPublisher) NotifyAll() {
	// Capture recipients and news for this notification.
	recipients := append([]subscription(nil), p.subscriptions...)
	news := p.news

	for _, sub := range recipients {
		sub.observer.Update(news)
	}
}

func (p *NewsPublisher) AddNews(news string) {
	p.news = news
	p.NotifyAll()
}

// Concrete observer.
type Subscriber struct {
	name string
}

func (s *Subscriber) Update(news string) {
	fmt.Printf("%s received news: %s\n", s.name, news)
}

// Client.
func main() {
	publisher := &NewsPublisher{}

	alice := &Subscriber{name: "Alice"}
	bob := &Subscriber{name: "Bob"}

	aliceID := publisher.Register(alice)
	publisher.Register(bob)

	publisher.AddNews("New Go release announced")

	publisher.Unregister(aliceID)

	publisher.AddNews("Observer pattern explained")
}
```

**Output:**

```text
Alice received news: New Go release announced
Bob received news: New Go release announced
Bob received news: Observer pattern explained
```

### Step 1: Define the Observer Interface

```go
type Observer interface {
	Update(news string)
}
```

Every observer implements this method.

The publisher can notify different observer types through the same interface.

### Step 2: Define Subscription Operations

```go
type Subject interface {
	Register(observer Observer) int
	Unregister(id int) bool
	NotifyAll()
}
```

- `Register()` returns an ID for the new subscription.
- `Unregister()` reports whether a matching subscription was removed.
- `NotifyAll()` sends the current news to subscribers.

A separate subject interface is optional if clients do not need that abstraction.

### Step 3: Store Registrations

```go
type subscription struct {
	id       int
	observer Observer
}
```

Each registration has its own ID.

Registering the same observer twice creates two subscriptions and therefore two notifications per publication.

IDs avoid comparing arbitrary observer interface values. Such comparisons can panic if the underlying concrete value is not comparable.

The original pointer-based subscribers are comparable, but the interface could also be implemented by other types.

### Step 4: Register an Observer

`Register()` increments the ID counter and appends the observer to the subscription slice.

Appending preserves registration order.

This example assumes callers supply valid, non-nil observers.

### Step 5: Remove a Subscription

`Unregister()` finds the matching ID, shifts later entries left, and shortens the slice.

Clearing the final unused slot removes the leftover observer reference from the backing array.

Other subscriptions remain in their original order.

### Step 6: Capture the Notification’s Recipients

```go
recipients := append([]subscription(nil), p.subscriptions...)
news := p.news
```

The publisher copies the subscription entries before invoking callbacks.

This defines a simple policy:

- Observers registered during notification receive future publications.
- Observers removed during notification may still receive the current publication.
- Every recipient in this notification receives the captured news value.

The observer objects themselves are not copied.

### Step 7: Notify Observers

```go
for _, sub := range recipients {
	sub.observer.Update(news)
}
```

Each callback runs synchronously in registration order.

A slow callback delays the remaining callbacks. A panic is not recovered here and interrupts normal notification flow.

The pattern itself does not guarantee ordering; this implementation explicitly preserves it.

### Step 8: Publish a Change

```go
func (p *NewsPublisher) AddNews(news string) {
	p.news = news
	p.NotifyAll()
}
```

The publisher stores the news and triggers notification.

Calling `AddNews()` publishes every time, even if the text matches the previous value.

### Step 9: Understand Concurrency and Delivery

Copying the subscription slice supports subscription changes from synchronous callbacks. It does not make concurrent access safe.

If multiple goroutines access the publisher:

- Protect its state with synchronization.
- Capture recipients while holding the lock.
- Release the lock before calling observers.

Calling observers while holding a lock can cause deadlocks if they call back into the publisher.

Observer callbacks that publish more news also create nested notifications. Applications should define how to handle that behavior.

### Step 10: Distinguish Observer from Broker-Based Pub/Sub

This example directly invokes objects inside one process.

A broker-based publish/subscribe system sends events through separate infrastructure and introduces concerns such as:

- Serialization.
- Delivery guarantees.
- Retries.
- Duplicate messages.
- Ordering.
- Independent consumer failures.

Observer is useful for local notifications, but it does not automatically provide those distributed-system capabilities.
