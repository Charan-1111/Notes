# Mediator Design Pattern
- Behavioural design pattern that centralizes communication between objects, so that they don't need to reference each other directly.
- Instead of objects ( called Colleagues ) talking to each other, they communicate through a mediator, which reduces coupling and makes the system easier to maintain.
- It promotes loose coupling by preventing objects from referring to each other directly and lets us vary their interactions independently.

- ## When to Use
	- When we have a group of tightly coupled classes or UI components that need to communicate.
	- Changes in one component requires updates in multiple others.
	- When we want to centralize communication logic to simplify maintenance and testing.

- ## Components
	- **Mediator Interface** - Defines the contract for communication channels between colleague objects.
	- **Concrete Mediator** - Implements the mediator interface and coordinates communication and behaviour among colleague objects.
	- **Colleague Classes** - Objects that interact with each other but only communicate via the mediator, unaware of each other's concrete implementations.

- ## Working
	1. Instead of objects calling each other, they call the mediator.
	2. The mediator takes care of routing messages and controlling interactions, centralizing the coordination logic.
	3. Objects only knows the mediator, resulting in loose coupling.

- ## Pros
	- Reduces tight coupling - Objects ( colleagues ) no longer reference each other directly -> dependencies are minimized.
	- Centralized Control - All communication logic is in one place, making it easier to manage and modify.
	- Improved maintainability - Adding or removing or modifying colleagues doesn't affect other colleagues, only the mediator.
	- Simplifies object classes - Colleagues only handle their own behaviour, interaction logic is delegated to the mediator.
	- Promotes reusability - Same colleague classes can be reused in different systems by plugging them into different mediators.

- ## Cons
	- Mediator can become a God Object - If too much logic is centralized, the mediator itself may become overly complex and difficult to maintain.
	- Reduced transparency - Interactions between colleagues are hidden inside the mediator -> harder to trace the actual flow of communication.
	- Performance Overhead - Messages or events always go through the mediator, adding an extra layer.
	- Over engineering for simple cases - If there are only a few interactions, introducing a mediator adds unnecessary complexity.

- ## Sample implementation in Golang

```go
// Chat Room
package main

import "fmt"

// Mediator interface
type ChatMediator interface {
	SendMessage (msg string, user User)
	Register(user User)
}

// Colleague Interface
type User interface {
	Send(msg string)
	Receive(msg string)
	SetMediator(mediator ChatMediator)
	GetName() string
}

// Concrete Mediator
type ChatRoom struct {
	users []User
}

func (c *ChatRoom) Register(user User) {
	c.users = append(c.users, user)
	user.SetMediator(c)
}

func (c *ChatRoom) SendMessage(msg string, sender User) {
	for _, user := range c.users {
		if user != sender {
			user.Receive(sender.GetName() + ":" + msg)
		}
	}
}

// Concrete Colleague
type ChatUser struct {
	name string
	mediator ChatMediator
}

func (u *ChatUser) Send(msg string) {
	u.mediator.SendMessage(msg, u)
}

func (u *ChatUser) Receive(msg string) {
	fmt.Println(u.name , " received : ", msg)
}

func (u *ChatUser) SetMediator(mediator ChatMediator) {
	u.mediator = mediator
}

func (u *ChatUser) GetName() string {
	return u.name
}

// Client
func main() {
	chatRoom := &ChatRoom{}

	alice := &ChatUser{name : "Alice"}
	bob := &ChatUser{name: "Bob"}
	charlie := &ChatUser{name: "Charlie"}

	chatRoom.Register(alice)
	chatRoom.Register(bob)
	chatRoom.Register(charlie)

	alice.Send("Hello everyone!")
	bob.Send("Hi Alice!")
}
```
