# Template Method Design Pattern

The **Template Method pattern** is a behavioral design pattern that defines a fixed sequence of steps and lets implementations customize specific steps.

For example, tea and coffee follow the same preparation sequence:

1. Boil water.
2. Brew the beverage.
3. Pour it into a cup.
4. Add condiments.

The order stays the same. Tea is steeped and coffee is brewed through a filter, and each can use different condiments.

The classic pattern uses a base class and subclasses. **In Go, a shared function can own the sequence while an interface defines the steps that vary.**

| Pattern | What does it delegate? | Example |
|---|---|---|
| **Template Method** | Specific steps within a shared workflow. | The recipe controls the order and delegates brewing and adding condiments. |
| **Strategy** | How to perform a task through one interchangeable implementation. | Checkout delegates payment to a credit card or PayPal strategy. |

## When to Use

- When several implementations follow the same sequence of steps.
- When some steps are identical and others need to vary.
- When the order matters and you want to define it in one place.

**Backend example:** A file importer might follow `Read → Parse → Validate → Save`. CSV and JSON importers can supply different parsing logic while sharing validation and saving. The workflow can stop before saving if parsing or validation fails.

## Components

| Component | Responsibility | Go example |
|---|---|---|
| **Template Method** | Defines the order and calls each step. | `PrepareRecipe()` |
| **Shared Steps** | Perform operations common to every implementation. | `boilWater()` and `pourInCup()` |
| **Required Custom Steps** | Define what each implementation must supply. | `Brew()` and `AddCondiments()` on `Beverage` |
| **Concrete Implementations** | Supply behavior for the custom steps. | `Tea` and `Coffee` |
| **Optional Hooks** | Let an implementation customize behavior at a predefined point. | A recipe could optionally check whether to add condiments. |

A **required step** must be implemented to satisfy the interface. A **hook** is optional: for example, the template could add condiments by default and consult a preference only when the beverage provides one. The sample keeps both custom steps required.

## Working

1. The caller passes a beverage to `PrepareRecipe()`.
2. The template boils water using shared code.
3. It calls the beverage's `Brew()` method.
4. It pours into a cup using shared code.
5. It calls the beverage's `AddCondiments()` method.

The template decides **when** each step runs. The beverage decides **how** its custom steps work. This is a form of inversion of control: the workflow calls the implementation at the right point.

## Pros

- **Reuses common steps:** Shared operations are written once.
- **Keeps the workflow consistent:** Every call through the template follows the same order.
- **Limits duplication:** New implementations supply the steps that vary.
- **Simplifies the caller:** The caller passes a beverage to `PrepareRecipe()` without coordinating individual steps.

## Cons

- **Fixes the sequence:** A recipe that needs a different order may need another workflow.
- **Makes shared changes affect every implementation:** Changing the template changes all recipes that use it.
- **Can add unnecessary abstraction:** A direct function may be clearer when there are few variations or little shared logic.
- **Makes interface changes costly:** Adding a required step means updating every implementation.

The classic inheritance-based form couples subclasses to a base class. The Go version avoids that dependency, but its implementations still depend on the step contract.

## Sample Implementation in Golang

In the original example, `Beverage` requires `PrepareRecipe()`, but embedding `BaseBeverage` gives `Tea` a method with the signature `PrepareRecipe(bev Beverage)`. The signatures differ, so `*Tea` does not satisfy `Beverage`.

Here, `PrepareRecipe()` is a shared function, and `Beverage` describes only the steps that vary.

```go
package main

import "fmt"

// Beverage defines the steps that vary by recipe.
type Beverage interface {
	Brew()
	AddCondiments()
}

// PrepareRecipe owns the preparation sequence.
func PrepareRecipe(bev Beverage) {
	boilWater()
	bev.Brew()
	pourInCup()
	bev.AddCondiments()
}

func boilWater() {
	fmt.Println("Boiling water")
}

func pourInCup() {
	fmt.Println("Pouring into a cup")
}

type Tea struct{}

func (Tea) Brew() {
	fmt.Println("Steeping the tea")
}

func (Tea) AddCondiments() {
	fmt.Println("Adding sugar and milk")
}

type Coffee struct{}

func (Coffee) Brew() {
	fmt.Println("Brewing coffee through a filter")
}

func (Coffee) AddCondiments() {
	fmt.Println("Adding milk")
}

func main() {
	fmt.Println("Preparing tea:")
	PrepareRecipe(Tea{})

	fmt.Println("\nPreparing coffee:")
	PrepareRecipe(Coffee{})
}
```

**Expected output:**

```text
Preparing tea:
Boiling water
Steeping the tea
Pouring into a cup
Adding sugar and milk

Preparing coffee:
Boiling water
Brewing coffee through a filter
Pouring into a cup
Adding milk
```

### Step-by-Step Explanation

1. **Define the variable steps.** `Beverage` requires `Brew()` and `AddCondiments()`. `Tea` and `Coffee` satisfy the interface automatically because they implement both methods.

2. **Write the common steps once.** `boilWater()` and `pourInCup()` apply to both drinks, so the template calls them directly.

3. **Put the sequence in one place.** `PrepareRecipe()` decides that brewing happens after boiling and before pouring. It delegates brewing and adding condiments through the interface.

4. **Pass in a concrete beverage.** With `PrepareRecipe(Tea{})`, `bev.Brew()` steeps the tea. With `PrepareRecipe(Coffee{})`, the same line brews coffee through a filter. The surrounding steps run in the same order.

5. **Extend the workflow through its existing steps.** To add another beverage that follows this sequence, implement `Brew()` and `AddCondiments()` and pass it to `PrepareRecipe()`. If the new drink needs a different sequence, reconsider whether it belongs in this template.
