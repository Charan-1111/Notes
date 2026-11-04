# Mediator Design Pattern

The **Mediator pattern** is a behavioral design pattern that centralizes interactions between a group of objects.

Instead of calling each other directly, these objects—called **colleagues**—communicate through a mediator.

The mediator decides how to route requests and coordinate their behavior.

**Simple example:**

In a chat room:

- Alice sends a message to the room.
- The room delivers it to Bob and Charlie.
- Alice does not need references to Bob or Charlie.

Users handle sending and receiving. The chat room handles communication rules.

**Mediator vs Observer:**

- **Mediator:** Coordinates interactions between colleagues.
- **Observer:** Notifies subscribers when an event or state change occurs.

Their implementations can overlap, but their primary purposes differ.

## When to Use

- When several objects communicate through many direct dependencies.
- When changing interaction rules requires editing multiple objects.
- When coordination logic should be managed separately from individual component behavior.
- When the same components need different interaction rules in different contexts.

**Backend example:**

An upload coordinator manages interactions between a validator, storage component, and metadata repository.

It decides which step runs next and how failures affect the workflow. Each component remains responsible for its own operation.

A central service is not automatically a mediator; the key is that it coordinates interactions that would otherwise create dependencies between colleagues.

## Components

| Component | Responsibility | Chat example |
|---|---|---|
| **Mediator Interface** | Defines how colleagues communicate through the mediator. | `ChatMediator`. |
| **Concrete Mediator** | Implements routing and coordination rules. | `ChatRoom`. |
| **Colleagues** | Perform their own work and use the mediator for interactions. | `ChatUser`, implementing `User`. |

The mediator knows its colleagues. Each colleague knows the mediator and does not need references to the other colleagues.

## Working

1. Users register with the chat room.
2. Registration connects each user to the mediator.
3. A user sends a message through the mediator.
4. The mediator checks that the sender is registered.
5. It delivers the message to every registered user except the sender.

The routing rule belongs to the mediator. Users do not implement recipient selection themselves.

## Pros

- **Reduces dependencies:** Colleagues do not need direct references to one another.
- **Centralizes coordination:** Interaction rules can be changed in one place.
- **Simplifies colleagues:** Each component focuses on its own behavior.
- **Supports reuse:** Components can work with other compatible mediator implementations.
- **Improves testability:** Coordination rules and individual component behavior can be tested separately.

Adding or changing a colleague may still require mediator changes, depending on the interaction rules.

## Cons

- **Risk of a God Object:** A mediator can become difficult to maintain if it coordinates too many unrelated workflows.
- **Indirect communication:** Understanding a request may require tracing through the mediator.
- **Concentrated complexity:** Colleagues become simpler, but coordination complexity moves into the mediator.
- **Unnecessary abstraction:** Simple interactions may be clearer with direct calls.

An extra method call usually has negligible overhead. Expensive routing, blocking operations, and complicated coordination are more relevant concerns.

## Sample Implementation in Golang

This example uses explicit user IDs and reports attempts to send before registration.

Messages are delivered synchronously in registration order.

```go
package main

import (
	"errors"
	"fmt"
)

// Mediator interface.
type ChatMediator interface {
	Register(user User) error
	SendMessage(message, senderID string) error
}

// Colleague interface.
type User interface {
	ID() string
	GetName() string
	Send(message string) error
	Receive(message string)
	SetMediator(mediator ChatMediator)
}

// Concrete mediator.
type ChatRoom struct {
	users []User
}

func (c *ChatRoom) Register(user User) error {
	if user.ID() == "" {
		return errors.New("user ID must not be empty")
	}

	for _, existing := range c.users {
		if existing.ID() == user.ID() {
			return fmt.Errorf("user %q is already registered", user.ID())
		}
	}

	c.users = append(c.users, user)
	user.SetMediator(c)

	return nil
}

func (c *ChatRoom) SendMessage(message, senderID string) error {
	var sender User

	for _, user := range c.users {
		if user.ID() == senderID {
			sender = user
			break
		}
	}

	if sender == nil {
		return fmt.Errorf("sender %q is not registered", senderID)
	}

	formatted := fmt.Sprintf("%s: %s", sender.GetName(), message)

	for _, user := range c.users {
		if user.ID() != senderID {
			user.Receive(formatted)
		}
	}

	return nil
}

// Concrete colleague.
type ChatUser struct {
	id       string
	name     string
	mediator ChatMediator
}

func (u *ChatUser) ID() string {
	return u.id
}

func (u *ChatUser) GetName() string {
	return u.name
}

func (u *ChatUser) SetMediator(mediator ChatMediator) {
	u.mediator = mediator
}

func (u *ChatUser) Send(message string) error {
	if u.mediator == nil {
		return errors.New("user has not joined a chat room")
	}

	return u.mediator.SendMessage(message, u.id)
}

func (u *ChatUser) Receive(message string) {
	fmt.Printf("%s received: %s\n", u.name, message)
}

// Client.
func main() {
	room := &ChatRoom{}

	alice := &ChatUser{id: "alice-1", name: "Alice"}
	bob := &ChatUser{id: "bob-1", name: "Bob"}
	charlie := &ChatUser{id: "charlie-1", name: "Charlie"}

	// Sending before registration returns an error.
	if err := alice.Send("Is anyone here?"); err != nil {
		fmt.Println("Send failed:", err)
	}

	for _, user := range []User{alice, bob, charlie} {
		if err := room.Register(user); err != nil {
			fmt.Println("Registration failed:", err)
			return
		}
	}

	if err := alice.Send("Hello everyone!"); err != nil {
		fmt.Println("Send failed:", err)
	}

	if err := bob.Send("Hi Alice!"); err != nil {
		fmt.Println("Send failed:", err)
	}
}
```

**Output:**

```text
Send failed: user has not joined a chat room
Bob received: Alice: Hello everyone!
Charlie received: Alice: Hello everyone!
Alice received: Bob: Hi Alice!
Charlie received: Bob: Hi Alice!
```

### Step 1: Define the Mediator Contract

```go
type ChatMediator interface {
	Register(user User) error
	SendMessage(message, senderID string) error
}
```

The mediator exposes registration and message routing.

The interface lets users depend on communication behavior rather than a specific `ChatRoom` implementation.

### Step 2: Define the Colleague Contract

The `User` interface provides:

- Identity through `ID()`.
- A display name through `GetName()`.
- Sending and receiving behavior.
- A method for connecting the user to a mediator.

User IDs identify participants. Display names do not need to be unique.

The example assumes registered users keep their IDs unchanged.

### Step 3: Register and Connect Users

```go
c.users = append(c.users, user)
user.SetMediator(c)
```

Registration stores the user and gives it a mediator reference.

The room rejects empty or duplicate IDs.

This example assumes valid, non-nil user objects and one room per user.

### Step 4: Send Through the Mediator

```go
func (u *ChatUser) Send(message string) error {
	if u.mediator == nil {
		return errors.New("user has not joined a chat room")
	}

	return u.mediator.SendMessage(message, u.id)
}
```

The user delegates communication to the mediator.

It does not find recipients or call other users.

The nil check handles a user whose mediator has not been assigned.

### Step 5: Validate the Sender

The room searches its registered users for the supplied sender ID.

If no match exists, it returns an error.

This is a membership check for the demonstration, not authentication. A real chat backend must derive sender identity from a trusted session rather than accepting an arbitrary ID.

### Step 6: Apply the Routing Rule

```go
for _, user := range c.users {
	if user.ID() != senderID {
		user.Receive(formatted)
	}
}
```

The room delivers the message to everyone except the sender.

Comparing string IDs avoids comparing observer-like interface values, which may contain non-comparable concrete types.

The mediator could implement different rules, such as private messages or role-based delivery, without putting recipient-selection logic into each user.

### Step 7: Let Recipients Handle Their Own Behavior

```go
func (u *ChatUser) Receive(message string) {
	fmt.Printf("%s received: %s\n", u.name, message)
}
```

The user decides what receiving means.

Here it prints the message. Another implementation could update a UI or record it in local history.

The room coordinates delivery; the recipient implements its reaction.

### Step 8: Understand Execution and Concurrency

Delivery is synchronous. A slow recipient delays later recipients and the sender’s return.

This implementation assumes registrations are stable during delivery and uses one goroutine. Concurrent access requires synchronization and a policy for registration changes.

For production systems, also define how delivery errors, callback panics, and nested sends should behave.

The Mediator pattern organizes interactions; concurrency, networking, and reliable delivery require additional implementation.
