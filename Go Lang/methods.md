# Methods in Go

## 1. What is a Method?

A **method** is a function associated with a particular type.

A normal function operates on values passed as arguments:

```go
func IsUserActive(user User) bool {
	return user.Active
}
```

A method attaches that behavior directly to the `User` type:

```go
func (u User) IsActive() bool {
	return u.Active
}
```

We call it using the value:

```go
user.IsActive()
```

Methods help us keep related **data and behavior together**.

For example, checking whether a user is active is behavior related to a user. Therefore, expressing it as `user.IsActive()` is usually clearer than `IsUserActive(user)`.

---

## 2. Basic Method Syntax

```go
func (receiver ReceiverType) MethodName(parameters) ReturnType {
	// method body
}
```

Example:

```go
package main

import "fmt"

type User struct {
	Name   string
	Active bool
}

func (u User) IsActive() bool {
	return u.Active
}

func main() {
	user := User{
		Name:   "Rahul",
		Active: true,
	}

	fmt.Println(user.IsActive())
}
```

Output:

```text
true
```

In this method:

```go
func (u User) IsActive() bool
```

- `u` is the **receiver variable**
- `User` is the **receiver type**
- `IsActive` is the **method name**
- `bool` is the **return type**

The receiver tells Go that the `IsActive` method belongs to the `User` type.

Inside the method, `u` represents the value on which the method was called.

```go
user.IsActive()
```

Here, `user` becomes the receiver value `u`.

---

## 3. Normal Function vs Method

Consider this type:

```go
type User struct {
	Name   string
	Active bool
}
```

### Using a normal function

```go
func IsUserActive(user User) bool {
	return user.Active
}

func main() {
	user := User{
		Name:   "Rahul",
		Active: true,
	}

	fmt.Println(IsUserActive(user))
}
```

### Using a method

```go
func (u User) IsActive() bool {
	return u.Active
}

func main() {
	user := User{
		Name:   "Rahul",
		Active: true,
	}

	fmt.Println(user.IsActive())
}
```

Both approaches work. The method version communicates that `IsActive` is behavior associated with `User`.

A simple way to think about it is:

```text
Function: IsUserActive(user)
Method:   user.IsActive()
```

---

## 4. What is a Receiver?

A receiver is the value on which a method operates.

```go
func (u User) Greet() string {
	return "Hello, " + u.Name
}
```

Here:

- `u` is the receiver variable
- `User` is the receiver type

Usage:

```go
user := User{Name: "Rahul"}

message := user.Greet()

fmt.Println(message)
```

Output:

```text
Hello, Rahul
```

The receiver is similar to a normal function parameter, but it appears between the `func` keyword and the method name.

Conceptually, this method:

```go
func (u User) Greet() string {
	return "Hello, " + u.Name
}
```

is similar to:

```go
func Greet(u User) string {
	return "Hello, " + u.Name
}
```

The important difference is that the first version is associated with the `User` type.

---

## 5. Receiver Naming Convention

Receiver names are normally short and related to the type.

```go
func (u User) Greet() string {
	return "Hello, " + u.Name
}
```

Common examples:

```go
func (u User) IsActive() bool
func (o Order) Total() float64
func (c Customer) FullName() string
func (s Server) Start() error
```

Avoid using names such as `this` or `self`:

```go
// Not idiomatic Go
func (this User) IsActive() bool {
	return this.Active
}
```

Go normally uses a short abbreviation of the type name:

```go
// Idiomatic Go
func (u User) IsActive() bool {
	return u.Active
}
```

Use the same receiver name consistently for all methods of a type.

---

# Value Receivers and Pointer Receivers

Go supports two main kinds of method receivers:

1. Value receiver
2. Pointer receiver

Understanding their difference is very important.

---

## 6. Value Receiver

A value receiver receives a copy of the value.

```go
func (u User) ChangeName(name string) {
	u.Name = name
}
```

Complete example:

```go
package main

import "fmt"

type User struct {
	Name string
}

func (u User) ChangeName(name string) {
	u.Name = name
}

func main() {
	user := User{Name: "Rahul"}

	user.ChangeName("Charan")

	fmt.Println(user.Name)
}
```

Output:

```text
Rahul
```

The original name did not change.

### Why?

When we call:

```go
user.ChangeName("Charan")
```

Go copies `user` into the receiver variable `u`.

The method changes only that copy:

```text
Original user        Copied receiver
Name: Rahul   --->   Name: Rahul
                     Name changed to Charan
```

After the method finishes, the copied receiver is discarded. The original value is still unchanged.

### When should we use a value receiver?

A value receiver is commonly used when:

- The method does not need to modify the original value.
- The type is small and inexpensive to copy.
- The type represents a value, such as a coordinate, date, or measurement.
- You want the method to behave like a read-only operation.

Example:

```go
type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

The `Area` method only calculates and returns a value. It does not modify the rectangle.

```go
rectangle := Rectangle{
	Width:  10,
	Height: 5,
}

fmt.Println(rectangle.Area())
```

Output:

```text
50
```

---

## 7. Pointer Receiver

A pointer receiver receives the address of the original value.

Its receiver type starts with `*`:

```go
func (u *User) ChangeName(name string) {
	u.Name = name
}
```

Complete example:

```go
package main

import "fmt"

type User struct {
	Name string
}

func (u *User) ChangeName(name string) {
	u.Name = name
}

func main() {
	user := User{Name: "Rahul"}

	user.ChangeName("Charan")

	fmt.Println(user.Name)
}
```

Output:

```text
Charan
```

This time, the original value changes because `u` points to the original `user`.

Conceptually:

```text
u ───────> original user
           Name: Rahul
```

When the method executes:

```go
u.Name = name
```

it updates the original object.

### When should we use a pointer receiver?

Use a pointer receiver when:

- The method needs to modify the original value.
- The struct is large and copying it would be inefficient.
- The type contains synchronization values such as `sync.Mutex`.
- Other methods on the same type already use pointer receivers.
- The type represents an object with changing state.

Example:

```go
type BankAccount struct {
	Balance float64
}

func (b *BankAccount) Deposit(amount float64) {
	b.Balance += amount
}

func (b *BankAccount) Withdraw(amount float64) bool {
	if amount > b.Balance {
		return false
	}

	b.Balance -= amount
	return true
}
```

Usage:

```go
account := BankAccount{
	Balance: 1000,
}

account.Deposit(500)

success := account.Withdraw(200)

fmt.Println(success)
fmt.Println(account.Balance)
```

Output:

```text
true
1300
```

---

## 8. Value Receiver vs Pointer Receiver

| Value receiver | Pointer receiver |
|---|---|
| Written as `(u User)` | Written as `(u *User)` |
| Receives a copy of the value | Receives a pointer to the original value |
| Usually cannot modify the original struct | Can modify the original struct |
| Suitable for small, read-only values | Suitable for mutable or large values |
| Both `User` and `*User` can call it | Usually called on an addressable `User` or `*User` |

Example:

```go
func (u User) GetName() string {
	return u.Name
}

func (u *User) SetName(name string) {
	u.Name = name
}
```

Usage:

```go
user := User{Name: "Rahul"}

fmt.Println(user.GetName())

user.SetName("Charan")

fmt.Println(user.GetName())
```

Output:

```text
Rahul
Charan
```

A useful initial rule is:

```text
Reading data without changing it → value receiver may be appropriate
Changing the original data       → pointer receiver
```

However, consistency also matters. If several methods of a type require pointer receivers, it is common to use pointer receivers for most or all methods of that type.

---

## 9. Go Automatically Takes the Address

Suppose a method has a pointer receiver:

```go
func (u *User) ChangeName(name string) {
	u.Name = name
}
```

Normally, a pointer could be passed like this:

```go
user := User{Name: "Rahul"}

(&user).ChangeName("Charan")
```

But Go allows us to write:

```go
user.ChangeName("Charan")
```

For an addressable variable, Go automatically treats it approximately as:

```go
(&user).ChangeName("Charan")
```

Therefore, both of these calls work:

```go
user.ChangeName("Charan")
(&user).ChangeName("Charan")
```

The shorter form is normally preferred.

---

## 10. Go Automatically Dereferences Pointers

A value receiver method can also be called using a pointer.

```go
func (u User) GetName() string {
	return u.Name
}
```

Usage:

```go
user := User{Name: "Rahul"}
userPointer := &user

fmt.Println(userPointer.GetName())
```

Go automatically dereferences `userPointer` when making this method call.

Conceptually, Go handles it approximately like this:

```go
(*userPointer).GetName()
```

This automatic conversion makes method calls convenient, but it does not change the method-set rules used when satisfying interfaces.

---

## 11. Important Detail: A Value Receiver Makes a Shallow Copy

It is common to say that a value receiver copies the object. That is correct, but the copy is generally a **shallow copy**.

Consider a struct containing a slice:

```go
type Team struct {
	Name    string
	Members []string
}
```

Now define a value receiver method:

```go
func (t Team) Update() {
	t.Name = "Backend Team"
	t.Members[0] = "Charan"
}
```

Usage:

```go
team := Team{
	Name:    "Development Team",
	Members: []string{"Rahul", "John"},
}

team.Update()

fmt.Println(team.Name)
fmt.Println(team.Members)
```

Output:

```text
Development Team
[Charan John]
```

Why did this happen?

- The `Team` struct itself was copied.
- Therefore, changing `t.Name` did not affect the original `team.Name`.
- A slice contains a reference to an underlying array.
- The copied slice still refers to the same underlying array.
- Therefore, modifying an existing slice element affected the original data.

This means that a value receiver should not automatically be considered completely immutable.

The same kind of behavior can occur with fields such as:

- Slices
- Maps
- Pointers
- Channels
- Functions
- Interfaces containing reference-like values

If a type owns mutable state, pointer receivers usually make the intention clearer.

---

# Methods on Named Types

## 12. Methods Are Not Restricted to Structs

Methods can be defined on named types, not only structs.

Example:

```go
type Celsius float64

func (c Celsius) ToFahrenheit() float64 {
	return float64(c)*9/5 + 32
}
```

Usage:

```go
temperature := Celsius(30)

fmt.Println(temperature.ToFahrenheit())
```

Output:

```text
86
```

Another example:

```go
type UserID string

func (id UserID) IsEmpty() bool {
	return id == ""
}
```

Usage:

```go
id := UserID("user-123")

fmt.Println(id.IsEmpty())
```

Output:

```text
false
```

Methods can be defined on named:

- Struct types
- Integer types
- String types
- Boolean types
- Slice types
- Map types
- Array types
- Other user-defined types

---

## 13. Methods on a Named Slice Type

```go
type Numbers []int

func (n Numbers) Sum() int {
	total := 0

	for _, number := range n {
		total += number
	}

	return total
}
```

Usage:

```go
numbers := Numbers{10, 20, 30}

fmt.Println(numbers.Sum())
```

Output:

```text
60
```

Here, `Numbers` is a named slice type, and `Sum` is associated with it.

---

## 14. Receiver-Type Restrictions

A method receiver must use a defined type from the same package where the method is declared.

For example, you cannot directly add a method to Go's built-in `string` type:

```go
// Invalid
func (s string) IsEmpty() bool {
	return s == ""
}
```

You also cannot add methods to a type declared in another package.

Instead, create your own named type:

```go
type MyString string

func (s MyString) IsEmpty() bool {
	return s == ""
}
```

Usage:

```go
value := MyString("")

fmt.Println(value.IsEmpty())
```

Output:

```text
true
```

The receiver's base type cannot itself be a pointer or interface type.

For example, define methods on `User` or use `*User` as the receiver—not on a separately defined pointer type.

---

# Methods and Interfaces

## 15. Methods Allow Types to Satisfy Interfaces

Interfaces in Go describe behavior through method signatures.

```go
type Speaker interface {
	Speak() string
}
```

Any type with a matching `Speak()` method satisfies this interface automatically.

```go
type Person struct {
	Name string
}

func (p Person) Speak() string {
	return "Hello, my name is " + p.Name
}
```

Now `Person` satisfies `Speaker`:

```go
func Introduce(s Speaker) {
	fmt.Println(s.Speak())
}

func main() {
	person := Person{Name: "Charan"}

	Introduce(person)
}
```

Output:

```text
Hello, my name is Charan
```

Go does not require us to explicitly declare:

```text
Person implements Speaker
```

The implementation is implicit. If the required method exists with the correct signature, the type satisfies the interface.

---

## 16. Method Sets

A type's **method set** determines which methods belong to that type, especially when interfaces are involved.

Consider:

```go
type User struct {
	Name string
}

func (u User) GetName() string {
	return u.Name
}

func (u *User) ChangeName(name string) {
	u.Name = name
}
```

The method sets are:

| Type | Methods in its method set |
|---|---|
| `User` | Methods declared with receiver `User` |
| `*User` | Methods declared with receiver `User` and `*User` |

Therefore:

- A `User` value has the value-receiver method `GetName`.
- A `*User` pointer has both `GetName` and `ChangeName`.

This is especially important for interfaces.

---

## 17. Pointer Receiver and Interface Satisfaction

Consider this interface:

```go
type NameChanger interface {
	ChangeName(name string)
}
```

The method is defined with a pointer receiver:

```go
func (u *User) ChangeName(name string) {
	u.Name = name
}
```

Therefore, `*User` satisfies `NameChanger`, but `User` does not.

```go
func Rename(changer NameChanger) {
	changer.ChangeName("Charan")
}

func main() {
	user := User{Name: "Rahul"}

	Rename(&user)

	fmt.Println(user.Name)
}
```

Output:

```text
Charan
```

This will not compile:

```go
Rename(user)
```

Even though Go can automatically take the address for a direct call like:

```go
user.ChangeName("Charan")
```

it does not use that convenience to change the method set when assigning a value to an interface.

Remember:

```text
Value receiver method:
User  satisfies the interface
*User also satisfies the interface

Pointer receiver method:
Only *User satisfies the interface
```

---

# Practical Method Examples

## 18. User Activation Example

```go
package main

import "fmt"

type User struct {
	ID     string
	Name   string
	Active bool
}

func (u User) IsActive() bool {
	return u.Active
}

func (u *User) Activate() {
	u.Active = true
}

func (u *User) Deactivate() {
	u.Active = false
}

func main() {
	user := User{
		ID:     "user-101",
		Name:   "Rahul",
		Active: false,
	}

	fmt.Println(user.IsActive())

	user.Activate()
	fmt.Println(user.IsActive())

	user.Deactivate()
	fmt.Println(user.IsActive())
}
```

Output:

```text
false
true
false
```

Here:

- `IsActive` only reads the state, so it uses a value receiver.
- `Activate` modifies the state, so it uses a pointer receiver.
- `Deactivate` also modifies the state, so it uses a pointer receiver.

For consistency, it would also be valid to use a pointer receiver for `IsActive`:

```go
func (u *User) IsActive() bool {
	return u.Active
}
```

---

## 19. Methods Can Accept Parameters

A receiver is separate from the method's normal parameters.

```go
type Product struct {
	Name  string
	Price float64
}

func (p Product) PriceAfterDiscount(discountPercentage float64) float64 {
	discount := p.Price * discountPercentage / 100
	return p.Price - discount
}
```

Usage:

```go
product := Product{
	Name:  "Keyboard",
	Price: 1000,
}

finalPrice := product.PriceAfterDiscount(10)

fmt.Println(finalPrice)
```

Output:

```text
900
```

In this method:

- `p` is the receiver.
- `discountPercentage` is a normal parameter.
- `float64` is the return type.

---

## 20. Methods Can Return Multiple Values

Like functions, methods can return multiple values.

```go
type BankAccount struct {
	Balance float64
}

func (b *BankAccount) Withdraw(amount float64) (float64, error) {
	if amount <= 0 {
		return b.Balance, fmt.Errorf("amount must be greater than zero")
	}

	if amount > b.Balance {
		return b.Balance, fmt.Errorf("insufficient balance")
	}

	b.Balance -= amount

	return b.Balance, nil
}
```

Usage:

```go
account := BankAccount{
	Balance: 1000,
}

balance, err := account.Withdraw(300)
if err != nil {
	fmt.Println("withdrawal failed:", err)
	return
}

fmt.Println("remaining balance:", balance)
```

Output:

```text
remaining balance: 700
```

---

## 21. Methods Can Call Other Methods

A method can call another method belonging to the same type.

```go
type User struct {
	Name   string
	Active bool
}

func (u User) IsActive() bool {
	return u.Active
}

func (u User) Status() string {
	if u.IsActive() {
		return "active"
	}

	return "inactive"
}
```

Usage:

```go
user := User{
	Name:   "Charan",
	Active: true,
}

fmt.Println(user.Status())
```

Output:

```text
active
```

---

## 22. Methods With Validation

Methods are useful for protecting the state of an object.

Instead of modifying fields from anywhere:

```go
account.Balance -= amount
```

we can put the rules inside a method:

```go
type BankAccount struct {
	balance float64
}

func (b *BankAccount) Deposit(amount float64) error {
	if amount <= 0 {
		return fmt.Errorf("deposit amount must be greater than zero")
	}

	b.balance += amount
	return nil
}

func (b *BankAccount) Balance() float64 {
	return b.balance
}
```

Usage:

```go
account := BankAccount{}

err := account.Deposit(1000)
if err != nil {
	fmt.Println(err)
	return
}

fmt.Println(account.Balance())
```

The method ensures that invalid deposits cannot change the account balance.

This is called **encapsulation**: keeping data and the rules that operate on that data together.

---

# Nil Pointer Receivers

## 23. Can a Method Be Called on a Nil Pointer?

Yes, a method with a pointer receiver can be called on a nil pointer. However, the method must handle it safely.

```go
type User struct {
	Name string
}

func (u *User) DisplayName() string {
	if u == nil {
		return "unknown user"
	}

	return u.Name
}
```

Usage:

```go
var user *User

fmt.Println(user.DisplayName())
```

Output:

```text
unknown user
```

Without the nil check, accessing `u.Name` would cause a runtime panic:

```go
func (u *User) DisplayName() string {
	return u.Name // Panics if u is nil
}
```

Nil receiver handling can be useful in some designs, but it should be intentional and clearly documented.

---

# Common Mistakes

## 24. Expecting a Value Receiver to Modify the Original Struct

Incorrect:

```go
func (u User) ChangeName(name string) {
	u.Name = name
}
```

If the original value must change, use a pointer receiver:

```go
func (u *User) ChangeName(name string) {
	u.Name = name
}
```

---

## 25. Forgetting the Nil Check

Potentially unsafe:

```go
func (u *User) GetName() string {
	return u.Name
}
```

If `u` can be nil, handle it:

```go
func (u *User) GetName() string {
	if u == nil {
		return ""
	}

	return u.Name
}
```

Not every method needs a nil check. Add one only when nil is a meaningful or possible receiver value in your design.

---

## 26. Mixing Receiver Types Without a Reason

Avoid randomly mixing value and pointer receivers:

```go
func (u User) GetName() string
func (u *User) IsActive() bool
func (u User) GetID() string
func (u *User) Display() string
```

It makes the method set and interface behavior harder to understand.

A better approach is to choose based on the type's behavior:

```go
func (u *User) GetName() string
func (u *User) IsActive() bool
func (u *User) GetID() string
func (u *User) Display() string
```

Using pointer receivers consistently is common for mutable domain entities such as:

- Users
- Accounts
- Orders
- Servers
- Database repositories
- Service objects

Value receivers are common for small value-like types such as:

- Coordinates
- Dates
- Durations
- Money values
- Measurements

---

## 27. Copying Types That Must Not Be Copied

Some structs contain fields that should not be copied, such as `sync.Mutex`.

```go
type Counter struct {
	mu    sync.Mutex
	value int
}
```

Use pointer receivers for this type:

```go
func (c *Counter) Increment() {
	c.mu.Lock()
	defer c.mu.Unlock()

	c.value++
}

func (c *Counter) Value() int {
	c.mu.Lock()
	defer c.mu.Unlock()

	return c.value
}
```

A value receiver would copy the mutex, which can lead to incorrect synchronization behavior.

---

# Choosing the Correct Receiver

## 28. Receiver Decision Guide

Ask these questions:

### Does the method need to modify the original value?

Use a pointer receiver:

```go
func (u *User) ChangeName(name string)
```

### Is the struct large or expensive to copy?

Use a pointer receiver:

```go
func (r *LargeReport) Generate()
```

### Does the type contain a mutex or another value that must not be copied?

Use a pointer receiver:

```go
func (c *Counter) Increment()
```

### Is the type small and value-like?

A value receiver may be appropriate:

```go
func (p Point) DistanceFromOrigin() float64
```

### Do some methods already require pointer receivers?

Consider using pointer receivers consistently for the type.

### Is the type intended to be immutable?

A value receiver may communicate that intention, although reference-like fields still need careful handling.

---

## 29. Complete Backend-Oriented Example

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

type User struct {
	ID     string
	Name   string
	Email  string
	Active bool
}

func (u User) IsActive() bool {
	return u.Active
}

func (u User) DisplayName() string {
	if strings.TrimSpace(u.Name) == "" {
		return "Unknown User"
	}

	return u.Name
}

func (u *User) ChangeName(name string) error {
	name = strings.TrimSpace(name)

	if name == "" {
		return errors.New("name cannot be empty")
	}

	u.Name = name
	return nil
}

func (u *User) Activate() {
	u.Active = true
}

func (u *User) Deactivate() {
	u.Active = false
}

func main() {
	user := User{
		ID:     "user-101",
		Name:   "Rahul",
		Email:  "rahul@example.com",
		Active: false,
	}

	fmt.Println("Name:", user.DisplayName())
	fmt.Println("Active:", user.IsActive())

	if err := user.ChangeName("Charan"); err != nil {
		fmt.Println("failed to change name:", err)
		return
	}

	user.Activate()

	fmt.Println("Updated name:", user.DisplayName())
	fmt.Println("Active:", user.IsActive())
}
```

Output:

```text
Name: Rahul
Active: false
Updated name: Charan
Active: true
```

This example demonstrates:

- Value receiver methods for reading data
- Pointer receiver methods for modifying data
- Validation inside a method
- Multiple methods associated with one type
- Behavior kept close to the data it operates on

---

# Summary

- A method is a function associated with a named type.
- A receiver tells Go which type the method belongs to.
- Methods are called using `value.MethodName()`.
- A value receiver receives a copy of the value.
- A pointer receiver can modify the original value.
- Go automatically takes addresses and dereferences pointers for many direct method calls.
- Methods are not restricted to structs; they can be defined on other locally defined named types.
- Methods allow types to satisfy interfaces implicitly.
- Method sets determine whether `T` or `*T` satisfies an interface.
- A value receiver creates a shallow copy, so reference-like fields may still share underlying data.
- Use pointer receivers for mutable, large, or non-copyable types.
- Choose receiver types intentionally and consistently.

A simple mental model is:

```text
Function:
Behavior receives data from the outside.

Method:
Behavior belongs to a type.

Value receiver:
Work with a copy.

Pointer receiver:
Work with the original value.
```