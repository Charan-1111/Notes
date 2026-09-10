# Open-Closed Principle (OCP)

## What is the Open-Closed Principle?

The **Open-Closed Principle (OCP)** states:

> Software entities such as classes, modules, structs, or functions should be **open for extension, but closed for modification**.

In simple terms:

- **Open for extension** → We should be able to add new functionality or behaviour.
- **Closed for modification** → We should try not to modify existing, tested, and stable code whenever we add a new variation of an existing behaviour.

### Simple Example

Imagine an application supports payments using:

- Credit Card
- PayPal
- UPI

Later, the business asks us to support **Net Banking**.

If adding Net Banking requires us to modify a large existing `PaymentService`, we increase the chance of accidentally breaking Credit Card, PayPal, or UPI payments.

With OCP, we design the payment system so that Net Banking can be added as a **new implementation**, while the existing payment-processing logic remains unchanged.

### Important: "Closed for Modification" Does Not Mean "Never Change Code"

OCP does **not** mean existing code can never be changed.

We may still modify code to:

- Fix bugs.
- Refactor poor design.
- Improve performance.
- Handle genuinely changed requirements.

The idea is that adding another variation of an already-known behaviour should ideally require **adding new code rather than repeatedly changing stable code**.

---

## When to Use

OCP is especially useful when:

- Requirements are likely to evolve.
- We expect multiple implementations of the same behaviour.
- We want to avoid breaking existing features while adding new ones.
- We are designing frameworks or libraries that others may extend.
- An `if/else` or `switch` statement keeps growing whenever a new type is introduced.

Common backend examples include:

- Payment methods: Credit Card, UPI, PayPal, Net Banking.
- Notifications: Email, SMS, Push Notification.
- Storage providers: Local Storage, AWS S3, Google Cloud Storage.
- Authentication: Password, Google OAuth, GitHub OAuth.

### A Note About `if` and `switch`

Using `if` or `switch` does **not automatically violate OCP**.

The problem appears when the same conditional logic must be modified every time a new implementation is added.

For example:

```go
switch paymentMethod {
case "credit":
    // ...
case "paypal":
    // ...
case "upi":
    // ...
case "netbanking":
    // ...
}
```

If this switch keeps growing whenever we add a payment method, it is a sign that the behaviour may be better represented using an interface.

---

## How Can We Break OCP?

We commonly violate OCP by:

- Modifying stable code every time a new requirement is introduced.
- Creating long `if/else` or `switch` chains that grow with every new implementation.
- Hardcoding specific implementations inside business logic.
- Making high-level services depend directly on every concrete implementation.

For example:

```go
func (p PaymentService) Pay(method string, amount float64) {
    if method == "credit" {
        // Credit card payment
    } else if method == "paypal" {
        // PayPal payment
    } else if method == "upi" {
        // UPI payment
    }
}
```

Suppose we now need Net Banking.

We have to open this existing function and add:

```go
} else if method == "netbanking" {
    // Net Banking payment
}
```

The more payment methods we add, the larger this function becomes and the more frequently stable code must be changed.

---

## How to Apply OCP

A simple approach is:

1. Identify the behaviour that can vary.
2. Create an interface representing that behaviour.
3. Create separate implementations of the interface.
4. Make the main service depend on the interface.
5. Add new implementations without changing the main service.

In our payment example, the behaviour that changes is:

```text
How the payment is performed
```

So we can represent it using:

```go
type PaymentMethod interface {
    Pay(amount float64)
}
```

Different payment methods can then implement this interface.

This uses several important Go/design concepts:

- **Interfaces** define the behaviour required by the service.
- **Polymorphism** lets different types be used through the same interface.
- **Composition** lets `PaymentService` contain/use a `PaymentMethod`.
- **Dependency Inversion** encourages the service to depend on an abstraction rather than a specific payment implementation.

---

## Pros

- Reduces the risk of breaking existing functionality.
- Makes systems easier to extend.
- Keeps stable business logic from changing frequently.
- Makes individual implementations easier to test.
- Encourages reusable and plug-and-play components.

---

## Cons

- Can introduce additional interfaces, structs, and files.
- Can make simple applications unnecessarily complex.
- Poorly chosen abstractions can reduce readability.
- Trying to predict every possible future requirement can lead to over-engineering.

OCP should therefore be applied where **variation is expected**, rather than creating interfaces for everything.

---

## Sample Implementation in Go

### Violates OCP

```go
package main

import "fmt"

type PaymentService struct{}

func (p PaymentService) Pay(method string, amount float64) {
    if method == "credit" {
        fmt.Println("Paid", amount, "using Credit Card")
    } else if method == "paypal" {
        fmt.Println("Paid", amount, "using PayPal")
    } else if method == "upi" {
        fmt.Println("Paid", amount, "using UPI")
    }
}

func main() {
    service := PaymentService{}

    service.Pay("credit", 100)
    service.Pay("upi", 150)
}
```

### Step-by-Step Explanation

#### Step 1: Create `PaymentService`

```go
type PaymentService struct{}
```

`PaymentService` is responsible for processing payments.

#### Step 2: Pass the payment type as a string

```go
func (p PaymentService) Pay(method string, amount float64)
```

The caller tells the service which payment method should be used.

For example:

```go
service.Pay("credit", 100)
```

#### Step 3: Decide the behaviour using conditions

```go
if method == "credit" {
    // ...
} else if method == "paypal" {
    // ...
}
```

`PaymentService` knows about every supported payment type.

This is the main problem.

#### Step 4: A new requirement arrives

Suppose we need:

```text
Net Banking
```

We must modify `PaymentService.Pay()`:

```go
} else if method == "netbanking" {
    fmt.Println("Paid", amount, "using Net Banking")
}
```

Now every new payment method requires changing the existing payment service.

Therefore, this design does not follow OCP well.

---

## Applying OCP

We can move the changing behaviour behind an interface.

```go
package main

import "fmt"

// Abstraction
type PaymentMethod interface {
    Pay(amount float64)
}

// Credit Card implementation
type CreditCard struct{}

func (CreditCard) Pay(amount float64) {
    fmt.Println("Paid", amount, "using Credit Card")
}

// PayPal implementation
type PayPal struct{}

func (PayPal) Pay(amount float64) {
    fmt.Println("Paid", amount, "using PayPal")
}

// UPI implementation
type UPI struct{}

func (UPI) Pay(amount float64) {
    fmt.Println("Paid", amount, "using UPI")
}

// PaymentService depends on the abstraction.
type PaymentService struct {
    method PaymentMethod
}

func (p PaymentService) Process(amount float64) {
    p.method.Pay(amount)
}

func main() {
    creditService := PaymentService{
        method: CreditCard{},
    }

    creditService.Process(100)

    upiService := PaymentService{
        method: UPI{},
    }

    upiService.Process(150)
}
```

### Step-by-Step Explanation

#### Step 1: Define the behaviour using an interface

```go
type PaymentMethod interface {
    Pay(amount float64)
}
```

The interface says:

> Any payment method used by our system must know how to `Pay`.

`PaymentService` does not need to know *how* Credit Card, UPI, or PayPal performs the payment.

It only needs something that satisfies `PaymentMethod`.

---

#### Step 2: Implement Credit Card

```go
type CreditCard struct{}

func (CreditCard) Pay(amount float64) {
    fmt.Println("Paid", amount, "using Credit Card")
}
```

Because `CreditCard` has:

```go
Pay(amount float64)
```

it automatically satisfies `PaymentMethod`.

Go does not require us to explicitly write something like:

```text
CreditCard implements PaymentMethod
```

Interface implementation in Go is **implicit**.

---

#### Step 3: Implement PayPal and UPI

```go
type PayPal struct{}

func (PayPal) Pay(amount float64) {
    fmt.Println("Paid", amount, "using PayPal")
}

type UPI struct{}

func (UPI) Pay(amount float64) {
    fmt.Println("Paid", amount, "using UPI")
}
```

Both types provide the same required behaviour:

```go
Pay(amount float64)
```

Therefore, all three can be used as a `PaymentMethod`.

---

#### Step 4: Make `PaymentService` depend on the interface

```go
type PaymentService struct {
    method PaymentMethod
}
```

Notice that we did **not** write:

```go
method CreditCard
```

If we did that, `PaymentService` would be tied specifically to Credit Card.

Instead, it accepts:

```go
PaymentMethod
```

So it can work with:

```text
CreditCard
PayPal
UPI
NetBanking
or any future payment method
```

as long as the type implements `Pay()`.

---

#### Step 5: Process the payment

```go
func (p PaymentService) Process(amount float64) {
    p.method.Pay(amount)
}
```

`PaymentService` does not contain:

```go
if credit ...
if paypal ...
if upi ...
```

It simply asks the configured payment method to perform the payment.

For:

```go
creditService := PaymentService{
    method: CreditCard{},
}

creditService.Process(100)
```

the flow is:

```text
creditService.Process(100)
        |
        v
PaymentService.Process()
        |
        v
p.method.Pay(100)
        |
        v
CreditCard.Pay(100)
        |
        v
Paid 100 using Credit Card
```

If `method` contains `UPI{}`, the same `PaymentService.Process()` function automatically calls `UPI.Pay()` instead.

This is polymorphism.

---

## Adding a New Payment Method

Now suppose the business asks us to support **Net Banking**.

We create a new implementation:

```go
type NetBanking struct{}

func (NetBanking) Pay(amount float64) {
    fmt.Println("Paid", amount, "using Net Banking")
}
```

Then use it:

```go
service := PaymentService{
    method: NetBanking{},
}

service.Process(500)
```

We did **not** modify:

```go
type PaymentService struct {
    method PaymentMethod
}

func (p PaymentService) Process(amount float64) {
    p.method.Pay(amount)
}
```

The existing service remains unchanged.

The system was:

```text
OPEN for extension
```

because we added `NetBanking`.

At the same time, the stable payment-processing logic was:

```text
CLOSED for modification
```

because `PaymentService` did not need to change.

That is the Open-Closed Principle.

---

## Key Takeaway

A useful way to remember OCP is:

> **Add new behaviour by adding new code instead of repeatedly changing stable code.**

Without OCP:

```text
New Requirement
      |
      v
Modify Existing Service
      |
      v
Add another if/switch case
      |
      v
Risk affecting existing behaviour
```

With OCP:

```text
New Requirement
      |
      v
Create New Implementation
      |
      v
Implement Existing Interface
      |
      v
Existing Service Remains Unchanged
```

For Go backend development, a common OCP-friendly design is:

```text
             PaymentMethod
              (interface)
                  |
       +----------+----------+
       |          |          |
       v          v          v
 CreditCard     PayPal      UPI
       |
       +------ future implementations
                  |
                  v
             NetBanking

                  ^
                  |
          PaymentService
        depends on interface
```

The goal of OCP is **not to create an interface for everything**. Use it when you have a behaviour that is likely to gain multiple implementations or change frequently.