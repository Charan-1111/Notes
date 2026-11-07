# Strategy Design Pattern

The **Strategy pattern** is a behavioral design pattern that lets you choose between different ways to perform the same task. Each strategy implements a shared interface, and the object using it delegates the work to whichever strategy is configured.

**Example:** A checkout can accept a credit card or PayPal. The caller chooses the payment method, but it always uses the same `Checkout()` method.

**Strategy vs State:**

| Pattern | What determines the behavior? | Example |
|---|---|---|
| **Strategy** | The caller or configuration chooses how to perform a task. | Pay by credit card or PayPal. |
| **State** | The object's current lifecycle state determines how it responds to an action. | An order can be cancelled before shipment but rejects cancellation after shipment. |

## When to Use

- When a task has multiple implementations that share the same contract.
- When callers need to choose an implementation at runtime.
- When the logic for each variation is substantial enough to test and maintain separately.

**Backend example:** A rate limiter can use a token bucket or a sliding window. Both can implement `Allow(key string) bool`, so the HTTP middleware can call `Allow()` without knowing which algorithm evaluates the request.

A small `switch` may be clearer when the variations are simple. Strategy gives the implementations a place to live; you may still use a `switch` to choose one.

## Components

| Component | Responsibility | Payment example |
|---|---|---|
| **Context** | Stores the selected strategy and delegates work to it. | `PaymentProcessor` |
| **Strategy Interface** | Defines the contract shared by the implementations. | `PaymentStrategy` |
| **Concrete Strategies** | Implement different ways to perform the task. | `CreditCardPayment`, `PayPalPayment` |
| **Client** | Chooses and supplies a strategy to the context. | `main()` |

## Working

1. The client selects a strategy and passes it to `SetStrategy()`.
2. The client calls `Checkout()` on the processor.
3. The processor validates the amount and delegates payment to the selected strategy.
4. The strategy performs the operation or returns an error.

If the client switches from credit card to PayPal, the next checkout uses PayPal. The checkout method itself stays the same.

## Pros

- **Keeps variations separate:** Each strategy can be understood and tested independently.
- **Simplifies the context:** The processor handles common checkout rules and delegates payment-specific work.
- **Supports runtime selection:** The caller can choose a strategy based on the user's payment method or application configuration.
- **Makes extension easier:** A new strategy can implement the existing interface without changing `Checkout()`. The code that selects strategies may still need updating.

## Cons

- **Adds abstraction:** An interface and separate implementations can be excessive for a couple of simple branches.
- **Requires a selection rule:** The caller or configuration needs to know which strategy to choose.
- **Requires a shared contract:** Implementations must agree on details such as amount units, currency, and what counts as success.
- **Shared configuration can be risky:** Changing the strategy on a processor shared by concurrent requests can cause a data race or route a request to the wrong strategy. Choose the strategy per request or use a processor configured with a fixed strategy.

## Sample Implementation in Golang

This example uses integer **paise** to represent INR amounts: `100_000` paise is ₹1,000. The payment methods simulate success or failure; they do not call real payment providers.

```go
package main

import (
	"errors"
	"fmt"
)

// Strategy interface: all amounts are in INR paise.
type PaymentStrategy interface {
	Pay(amountPaise int64) error
}

// Concrete strategy: credit card.
type CreditCardPayment struct{}

func (CreditCardPayment) Pay(amountPaise int64) error {
	fmt.Printf("Paid %d paise using a credit card.\n", amountPaise)
	return nil
}

// Concrete strategy: PayPal.
type PayPalPayment struct {
	available bool
}

func (p PayPalPayment) Pay(amountPaise int64) error {
	if !p.available {
		return errors.New("PayPal is unavailable")
	}

	fmt.Printf("Paid %d paise using PayPal.\n", amountPaise)
	return nil
}

// Context.
type PaymentProcessor struct {
	strategy PaymentStrategy
}

func (p *PaymentProcessor) SetStrategy(strategy PaymentStrategy) {
	p.strategy = strategy
}

func (p *PaymentProcessor) Checkout(amountPaise int64) error {
	if amountPaise <= 0 {
		return errors.New("payment amount must be positive")
	}

	if p.strategy == nil {
		return errors.New("no payment strategy configured")
	}

	if err := p.strategy.Pay(amountPaise); err != nil {
		return fmt.Errorf("payment failed: %w", err)
	}

	return nil
}

// Client.
func main() {
	processor := &PaymentProcessor{}

	if err := processor.Checkout(100_000); err != nil {
		fmt.Println("Checkout:", err)
	}

	processor.SetStrategy(CreditCardPayment{})
	if err := processor.Checkout(100_000); err != nil {
		fmt.Println("Checkout:", err)
	}

	processor.SetStrategy(PayPalPayment{available: true})
	if err := processor.Checkout(20_000); err != nil {
		fmt.Println("Checkout:", err)
	}

	processor.SetStrategy(PayPalPayment{available: false})
	if err := processor.Checkout(20_000); err != nil {
		fmt.Println("Checkout:", err)
	}
}
```

**Expected output:**

```text
Checkout: no payment strategy configured
Paid 100000 paise using a credit card.
Paid 20000 paise using PayPal.
Checkout: payment failed: PayPal is unavailable
```

### Step-by-Step Explanation

1. **Define the shared contract.** `PaymentStrategy` requires `Pay(amountPaise int64) error`. Both implementations accept the same amount unit and return an error if payment fails. Integer minor units let you represent amounts exactly without floating-point rounding.

2. **Implement each payment method.** `CreditCardPayment` simulates a successful charge. `PayPalPayment` checks `available` so the example can demonstrate both success and failure. Go recognizes that both types satisfy `PaymentStrategy` because their `Pay()` methods match the interface.

3. **Store the selected strategy.** `PaymentProcessor` holds a `PaymentStrategy` field. `SetStrategy()` replaces that field; it does not run the payment.

4. **Validate, then delegate.** `Checkout()` rejects invalid amounts and handles an unset strategy. It then calls `p.strategy.Pay(amountPaise)`. This is where the selected implementation takes over.

5. **Return failures to the caller.** If `Pay()` fails, `Checkout()` adds context with `fmt.Errorf("payment failed: %w", err)`. `%w` preserves the original error for callers that need to inspect it.

6. **Switch strategies without rewriting checkout.** `main()` runs the same `Checkout()` method with a credit card and with PayPal. To add another payment method, implement `Pay()` and make the new strategy available to the caller.
