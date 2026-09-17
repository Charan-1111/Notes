# Law of Demeter Principle in Go

## 1. Introduction

The **Law of Demeter (LoD)** states that a function, struct, or module should have **limited knowledge about the internal structure of other components**.

It is also called the:

> **Principle of Least Knowledge**

In simple terms:

> A component should communicate only with its immediate dependencies—not reach through them to access deeply nested objects.

### Simple analogy

Imagine ordering food at a restaurant.

You tell the waiter what you want. You do not:

1. Walk into the kitchen.
2. Find the chef.
3. Open the refrigerator.
4. Select the ingredients.
5. Prepare the food yourself.

You communicate with the waiter, and the waiter coordinates with the kitchen.

The same idea applies to software: communicate with the object directly responsible for the operation.

---

## 2. What Can an Object Communicate With?

According to the Law of Demeter, a method should generally communicate with:

1. **Itself**
2. **Its direct fields or components**
3. **Objects passed as parameters**
4. **Objects it creates internally**
5. **Its direct dependencies**

### Example

```go
type OrderService struct {
	repository OrderRepository
}

func (s *OrderService) CreateOrder(customer Customer) error {
	order := NewOrder(customer)

	return s.repository.Save(order)
}
```

### Step-by-step explanation

1. `OrderService` communicates with its direct dependency, `repository`.
2. It communicates with `customer`, which was passed as a parameter.
3. It communicates with `order`, which it created.
4. It does not reach into the internal fields of the repository or customer.

This keeps `OrderService` independent of unnecessary implementation details.

---

## 3. Why Is the Law of Demeter Important?

Consider the following code:

```go
city := order.Customer.Address.City
```

This code knows several internal details:

- An `Order` contains a `Customer`.
- A `Customer` contains an `Address`.
- An `Address` contains a `City`.

If the internal structure changes, every place using this chain may also need to change.

For example, suppose `Address` is moved into a `Profile`:

```go
type Customer struct {
	Profile Profile
}
```

The previous code will no longer compile:

```go
city := order.Customer.Address.City
```

It must become:

```go
city := order.Customer.Profile.Address.City
```

The caller is affected even though it only wanted the customer's city.

The Law of Demeter reduces this problem by hiding internal navigation behind meaningful methods.

---

## 4. When to Use the Law of Demeter

The principle is useful when:

- Code contains deep access chains such as `order.Customer.Address.City`.
- One module knows too much about another module's internal structure.
- Changes in one struct cause changes in several unrelated places.
- Business logic is being performed outside the object responsible for it.
- You want clearer and more stable APIs.
- You are designing service, repository, or domain layers.

Go does not use traditional classes, but the principle still applies through:

- Structs
- Methods
- Interfaces
- Composition
- Packages

---

## 5. How the Law of Demeter Is Violated

### 5.1 Accessing deeply nested fields

```go
city := order.Customer.Address.City
```

The caller navigates through multiple objects to obtain the value.

---

### 5.2 Manipulating another object's internals

```go
order.Customer.Address.City = "Hyderabad"
```

The caller directly changes an object buried inside another object.

This creates strong coupling between the caller and the complete object structure.

---

### 5.3 Passing an object only to access nested data

```go
func PrintCustomerCity(order Order) {
	fmt.Println(order.Customer.Address.City)
}
```

The function receives an entire `Order`, even though it only needs a city.

A simpler alternative may be:

```go
func PrintCity(city string) {
	fmt.Println(city)
}
```

Whether this change is appropriate depends on the responsibility of the function.

---

### 5.4 Depending on internal structures

```go
func SendOrder(order Order) {
	postalCode := order.Customer.Address.PostalCode
	// Send the order using the postal code.
}
```

`SendOrder` must understand how the order, customer, and address are connected.

That knowledge should usually remain inside the relevant domain objects.

---

## 6. How to Apply the Law of Demeter

### 6.1 Hide internal details

Keep fields unexported when outside packages should not access them directly.

```go
type Address struct {
	city string
}
```

Because `city` starts with a lowercase letter, it cannot be accessed directly from another package.

---

### 6.2 Expose meaningful behaviour

Instead of exposing an entire internal object, expose the operation callers actually need.

```go
func (o Order) DeliveryCity() string {
	return o.customer.City()
}
```

The caller asks the order for its delivery city without knowing how the order stores customer information.

---

### 6.3 Use delegation methods

Each object communicates with its direct component.

```go
func (c Customer) City() string {
	return c.address.City()
}

func (a Address) City() string {
	return a.city
}
```

`Customer` knows about its direct `Address`, while `Order` knows about its direct `Customer`.

---

### 6.4 Use small interfaces

A component should depend only on the behaviour it requires.

```go
type CityProvider interface {
	DeliveryCity() string
}
```

The caller does not need to know the complete structure of the object.

---

### 6.5 Follow “Tell, Don’t Ask”

Instead of retrieving data and deciding what to do:

```go
if order.Status() == "pending" {
	order.SetStatus("confirmed")
}
```

Tell the object to perform the business operation:

```go
if err := order.Confirm(); err != nil {
	return err
}
```

The `Order` object owns the rules for confirming an order.

---

## 7. Go Example: Violating the Law of Demeter

```go
package main

import "fmt"

type Address struct {
	City string
}

type Customer struct {
	Address Address
}

type Order struct {
	Customer Customer
}

func main() {
	order := Order{
		Customer: Customer{
			Address: Address{
				City: "Mumbai",
			},
		},
	}

	// Violates LoD by navigating through nested objects.
	fmt.Println("City:", order.Customer.Address.City)
}
```

### Step-by-step explanation

1. `Address` contains a `City`.
2. `Customer` contains an `Address`.
3. `Order` contains a `Customer`.
4. `main` reaches through `Order`, `Customer`, and `Address`.
5. Therefore, `main` knows the complete internal structure of `Order`.

The problematic part is:

```go
order.Customer.Address.City
```

If any level of this structure changes, `main` must also change.

---

## 8. Go Example: Applying the Law of Demeter

```go
package main

import "fmt"

type Address struct {
	city string
}

func NewAddress(city string) Address {
	return Address{city: city}
}

func (a Address) City() string {
	return a.city
}

type Customer struct {
	address Address
}

func NewCustomer(address Address) Customer {
	return Customer{address: address}
}

func (c Customer) City() string {
	return c.address.City()
}

type Order struct {
	customer Customer
}

func NewOrder(customer Customer) Order {
	return Order{customer: customer}
}

func (o Order) DeliveryCity() string {
	return o.customer.City()
}

func main() {
	address := NewAddress("Mumbai")
	customer := NewCustomer(address)
	order := NewOrder(customer)

	fmt.Println("City:", order.DeliveryCity())
}
```

### Step-by-step explanation

#### Step 1: Create an address

```go
address := NewAddress("Mumbai")
```

`NewAddress` creates an `Address`. The caller does not directly set its internal fields.

#### Step 2: Create a customer

```go
customer := NewCustomer(address)
```

The customer receives its address through a constructor.

#### Step 3: Create an order

```go
order := NewOrder(customer)
```

The order receives its direct dependency: the customer.

#### Step 4: Ask the order for the delivery city

```go
order.DeliveryCity()
```

The caller does not navigate through the customer and address.

#### Step 5: Delegate through direct relationships

The delegation happens as follows:

```text
Order.DeliveryCity()
        ↓
Customer.City()
        ↓
Address.City()
```

Each object communicates only with its direct component.

### Benefit

If the customer's address later moves into a profile, only the internal customer implementation may need to change.

The caller can continue using:

```go
order.DeliveryCity()
```

---

## 9. Practical Backend Example

Suppose an order service must send a shipping notification.

### Tightly coupled implementation

```go
func (s *OrderService) SendShippingNotification(order Order) error {
	email := order.Customer.ContactDetails.Email
	return s.emailClient.Send(email, "Your order has been shipped")
}
```

The service knows that:

- `Order` contains `Customer`.
- `Customer` contains `ContactDetails`.
- `ContactDetails` contains `Email`.

This creates unnecessary coupling.

### Improved implementation

```go
type ShippingNotifier interface {
	NotifyOrderShipped(orderID string, recipientEmail string) error
}

type Order struct {
	id       string
	customer Customer
}

func (o Order) ID() string {
	return o.id
}

func (o Order) CustomerEmail() string {
	return o.customer.Email()
}

type Customer struct {
	email string
}

func (c Customer) Email() string {
	return c.email
}

type OrderService struct {
	notifier ShippingNotifier
}

func (s *OrderService) Ship(order Order) error {
	return s.notifier.NotifyOrderShipped(
		order.ID(),
		order.CustomerEmail(),
	)
}
```

### Step-by-step explanation

1. `OrderService` depends on the small `ShippingNotifier` interface.
2. It does not know how notifications are sent.
3. It asks `Order` for the customer email.
4. It does not access `Customer` or its internal fields directly.
5. The notifier can be implemented using email, SMS, or another provider.

This reduces coupling between the service, domain model, and notification infrastructure.

---

## 10. Law of Demeter vs Method Chaining

Not every method chain violates the Law of Demeter.

### Potential violation

```go
order.Customer().Address().City()
```

This navigates through several domain objects.

The caller knows how the objects are connected.

### Usually acceptable fluent API

```go
query.
	Where("status = ?", "active").
	OrderBy("created_at DESC").
	Limit(10)
```

This is usually not a violation because each method operates on and returns the same logical query builder.

The caller is not navigating through the internal structure of unrelated objects.

### Useful question

When reviewing a method chain, ask:

> Am I describing behaviour on one abstraction, or navigating through several objects to access their internals?

Navigating through several objects is the main warning sign.

---

## 11. Advantages

### Reduced coupling

Components depend on fewer internal implementation details.

### Better encapsulation

Internal data structures remain hidden behind meaningful methods.

### Easier maintenance

Changing one object's internal structure is less likely to affect unrelated code.

### Clearer APIs

Methods such as `DeliveryCity()` clearly communicate their purpose.

### Easier testing

Small interfaces and focused dependencies are easier to mock or replace in tests.

---

## 12. Disadvantages and Trade-offs

### Additional delegation methods

You may need methods such as:

```go
func (o Order) DeliveryCity() string
func (c Customer) City() string
func (a Address) City() string
```

This can add boilerplate.

### Too many thin wrappers

If every field gets multiple forwarding methods, the design may become harder to follow.

### Reduced transparency

Developers may need to inspect several methods to understand where a value comes from.

### Unnecessary abstraction

For small DTOs or temporary data structures, direct field access may be simpler and perfectly acceptable.

The Law of Demeter is a design guideline—not a rule that must be followed blindly.

---

## 13. Common Mistakes

### 13.1 Creating meaningless getters

This does not necessarily improve the design:

```go
order.GetCustomer().GetAddress().GetCity()
```

The getters hide field access, but the caller still knows the entire object structure.

A more meaningful API is:

```go
order.DeliveryCity()
```

---

### 13.2 Applying the principle too strictly

Direct access may be reasonable for:

- Simple DTOs
- API request and response structures
- Configuration objects
- Small internal data structures
- Code where the structure is intentionally public

Example:

```go
type CreateUserRequest struct {
	Name  string `json:"name"`
	Email string `json:"email"`
}
```

Accessing `request.Email` is normal because the struct is designed to carry data.

---

### 13.3 Creating interfaces without a purpose

Do not create an interface for every struct only to claim that the code follows LoD.

Create an interface when:

- A caller requires a small set of behaviours.
- Multiple implementations are possible.
- It creates a useful boundary between components.
- It improves testing or separation of concerns.

---

### 13.4 Moving behaviour to the wrong object

Suppose an order determines whether it can be cancelled.

Avoid placing the rule in an unrelated service:

```go
if order.Status() == "pending" {
	order.SetStatus("cancelled")
}
```

Prefer keeping the business rule inside `Order`:

```go
if err := order.Cancel(); err != nil {
	return err
}
```

The object owning the data should usually protect the rules related to that data.

---

## 14. Quick Review Checklist

When reviewing code, ask:

- Does this code access deeply nested fields?
- Does it know too much about another object's structure?
- Will an internal structure change affect multiple callers?
- Can I expose a meaningful higher-level method?
- Can the operation be moved to the object that owns the data?
- Would a small interface reduce coupling?
- Am I adding useful abstraction or only unnecessary wrappers?
- Is direct field access acceptable because this is intentionally a simple DTO?

A long chain is a warning sign, not automatic proof of a design problem.

---

## 15. Summary

The Law of Demeter encourages components to have limited knowledge of other components.

Instead of this:

```go
order.Customer.Address.City
```

Prefer a meaningful operation:

```go
order.DeliveryCity()
```

The main idea is:

> Communicate with immediate dependencies and hide unnecessary internal details.

Remember:

- Avoid navigating through deeply nested objects.
- Expose behaviour instead of raw internal structures.
- Use delegation where it creates a clear boundary.
- Use small interfaces to reduce coupling.
- Follow “Tell, Don’t Ask” for business operations.
- Do not apply the principle blindly to simple DTOs or data containers.

The goal is not to eliminate every method chain. The goal is to produce code that is easier to change, test, and maintain.