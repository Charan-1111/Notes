# Factory Method Design Pattern in Go

Factory Method is a **creational design pattern** where an interface defines a method for creating an object, and different implementations decide which concrete object to create.

Imagine a checkout system that can pay by credit card or PayPal. The checkout flow should call `Pay()` without needing payment-specific code. A creator supplies the appropriate payment method.

## Types of Factory Patterns

| Pattern | Who decides what to create? | Typical shape |
|---|---|---|
| Simple Factory | One function selects a type, often with a `switch` | `NewPaymentMethod("credit")` |
| Factory Method | Each creator implements a creation method | `CreditCardCreator.CreatePayment()` |
| Abstract Factory | A factory creates a family of related objects | `UIFactory.CreateButton()` and `CreateCheckbox()` |

Your original `PaymentFactory(method string)` example is a **Simple Factory**. Its `switch` chooses between `CreditCard` and `PayPal`.

## Key Concepts

- **Product interface:** Describes what the created objects can do, such as `Pay()`.
- **Factory method:** A method that returns a product through that interface.
- **Concrete creators:** Each implements the factory method and chooses a concrete product.
- **Client:** Uses the creator and the product interfaces.

In languages with classes, Factory Method is often explained through subclass inheritance. **Go uses interfaces and composition** to express the same idea.

## When to Use

Use Factory Method when:

- Multiple creators need to supply different implementations of the same product interface.
- A shared workflow should use the product without knowing its concrete type.
- You expect to add creators while keeping that shared workflow unchanged.

If all you need is to choose a payment method from a string, a **Simple Factory** may be easier. If construction is straightforward and there is no shared creator workflow, a regular constructor may be enough.

## Components

| Component | Payment example | Responsibility |
|---|---|---|
| Product | `PaymentMethod` | Defines `Pay()` |
| Concrete Products | `CreditCard`, `PayPal` | Implement `Pay()` |
| Creator | `PaymentCreator` | Defines `CreatePayment()` |
| Concrete Creators | `CreditCardCreator`, `PayPalCreator` | Create the corresponding products |

## Pros

- **Less coupling in the shared workflow:** Checkout depends on interfaces instead of concrete payment types.
- **Focused creation logic:** Each creator knows how to construct its product.
- **Easy to extend the workflow:** A new creator can be passed to checkout without changing checkout's code.

## Cons

- **More types and code:** Each product may need a corresponding creator.
- **More steps to follow:** To find the concrete product, you may need to trace which creator was passed in.
- **Can be unnecessary:** A constructor or Simple Factory is often clearer for a small application.

A large `switch` becoming bloated is mainly a **Simple Factory** concern. Factory Method avoids putting every product choice inside one factory method, though the application still needs to decide which creator to use.

## Sample Implementation in Go

```go
package main

import "fmt"

// Step 1: Define the product interface.
type PaymentMethod interface {
	Pay(amount float64)
}

// Step 2: Implement concrete products.
type CreditCard struct{}

func (CreditCard) Pay(amount float64) {
	fmt.Printf("Paid %.2f using Credit Card\n", amount)
}

type PayPal struct{}

func (PayPal) Pay(amount float64) {
	fmt.Printf("Paid %.2f using PayPal\n", amount)
}

// Step 3: Define the creator interface and its factory method.
type PaymentCreator interface {
	CreatePayment() PaymentMethod
}

// Step 4: Implement concrete creators.
type CreditCardCreator struct{}

func (CreditCardCreator) CreatePayment() PaymentMethod {
	return CreditCard{}
}

type PayPalCreator struct{}

func (PayPalCreator) CreatePayment() PaymentMethod {
	return PayPal{}
}

// Step 5: Write the shared workflow.
func Checkout(creator PaymentCreator, amount float64) {
	payment := creator.CreatePayment()
	payment.Pay(amount)
}

func main() {
	Checkout(CreditCardCreator{}, 100.50)
	Checkout(PayPalCreator{}, 250.00)
}
```

### How the code works, step by step

1. `PaymentMethod` says that every payment product must have a `Pay()` method.
2. `CreditCard` and `PayPal` each implement `Pay()` differently.
3. `PaymentCreator` declares the factory method, `CreatePayment()`.
4. `CreditCardCreator` returns a `CreditCard`; `PayPalCreator` returns a `PayPal`.
5. `Checkout` asks its creator for a payment method, then calls `Pay()`. It does not check whether the product is a credit card or PayPal.
6. `main` chooses the creator and passes it to `Checkout`.

**Output:**

```text
Paid 100.50 using Credit Card
Paid 250.00 using PayPal
```

To add another payment type, implement `PaymentMethod`, add its creator, and pass that creator to `Checkout`. The `Checkout` function does not need to change.

> For a real payment system, avoid `float64` for money because floating-point values can introduce rounding errors. Use an integer representing the smallest currency unit or an appropriate decimal type.
