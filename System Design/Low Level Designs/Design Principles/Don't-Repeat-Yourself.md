# Don't Repeat Yourself (DRY) Principle

## 1. What Is the DRY Principle?

**DRY** stands for **Don't Repeat Yourself**.

The principle states:

> Every piece of knowledge or business logic should have a single, clear, and authoritative representation in the system.

In simple terms, if the same rule or logic is required in multiple places, it should usually be defined once and reused.

For example, suppose an application calculates a discount using this rule:

```text
Customers with an order value above ₹5,000 receive a 10% discount.
```

If this rule is written separately in the order service, payment service, and invoice service, changing the discount requires updating all three places.

If one place is missed, the system may produce inconsistent results.

A better approach is to define the discount rule once and reuse it wherever required.

> DRY is mainly about avoiding duplicated knowledge and business rules—not merely reducing repeated lines of code.

---

## 2. Why Is Duplication a Problem?

Duplication makes software harder to change and maintain.

Consider the following tax calculation:

```go
tax := price * 0.18
```

If this calculation is duplicated in five different places and the tax rate changes, all five places must be updated.

This can cause several problems:

- A developer may forget to update one location.
- Different parts of the application may produce different results.
- Testing and debugging become more difficult.
- Future changes require more effort.
- The codebase becomes harder to understand.

With DRY, the tax calculation can be defined in one function and reused:

```go
const taxRate = 0.18

func calculateTax(price float64) float64 {
	return price * taxRate
}
```

Now, the tax rule has one authoritative representation.

---

## 3. When to Apply DRY

Consider applying DRY in the following situations:

- The same business rule exists in multiple modules.
- The same validation is performed by multiple handlers.
- A logic change requires updating several places.
- Constants or configuration values are repeatedly hard-coded.
- The same database query appears in multiple repositories.
- Multiple functions contain the same processing logic.
- Reusing a common implementation can reduce bugs and maintenance effort.

Before creating an abstraction, verify that the duplicated code represents the **same responsibility or knowledge**.

---

## 4. How We Break the DRY Principle

The DRY principle is commonly broken by:

- Copying and pasting the same logic into multiple functions.
- Duplicating validation rules across handlers or services.
- Hard-coding the same constant in multiple places.
- Repeating the same database query in different repositories.
- Creating multiple utility functions that perform nearly identical operations.
- Implementing the same business rule differently across services.
- Duplicating configuration values across source files.

### Example

Suppose multiple handlers independently validate a username:

```go
if len(username) < 3 {
	return errors.New("username must contain at least 3 characters")
}
```

If the minimum username length later changes to five characters, every copy must be found and updated.

This validation rule should instead have one authoritative implementation.

---

## 5. How to Apply the DRY Principle

### 5.1 Extract Common Logic

Move repeated logic into a reusable function:

```go
func calculateDiscount(price float64) float64 {
	return price * 0.10
}
```

### 5.2 Centralize Constants

Instead of repeating a value:

```go
finalPrice := price + price*0.18
```

Define it once:

```go
const taxRate = 0.18
```

### 5.3 Centralize Business Rules

Business rules should be placed in a suitable service or domain layer instead of being duplicated across handlers.

For example:

```go
func IsEligibleForDiscount(orderAmount float64) bool {
	return orderAmount >= 5000
}
```

### 5.4 Reuse Repository Methods

If multiple services need to find a user by email, create one repository method:

```go
func (r *UserRepository) FindByEmail(email string) (*User, error) {
	// Execute the database query.
	return nil, nil
}
```

### 5.5 Create Shared Components Carefully

Shared components should represent a meaningful common responsibility.

Do not combine unrelated code simply because it looks similar.

---

## 6. Go Example: Breaking the DRY Principle

```go
package main

import "fmt"

func areaCircle(radius float64) float64 {
	return 3.14159 * radius * radius
}

func printCircleArea(radius float64) {
	area := 3.14159 * radius * radius
	fmt.Println("Area of the circle is:", area)
}

func main() {
	fmt.Println(areaCircle(5))
	printCircleArea(5)
}
```

### Step-by-Step Explanation

1. `areaCircle` calculates the area using:

   ```go
   3.14159 * radius * radius
   ```

2. `printCircleArea` repeats the same calculation.

3. The value of π is also hard-coded in both places.

4. Therefore, the formula and constant have multiple representations.

5. If the formula or value changes, both functions must be updated.

6. If only one function is updated, they may return different results.

This violates DRY because the same knowledge is duplicated.

---

## 7. Go Example: Applying the DRY Principle

```go
package main

import "fmt"

const pi = 3.14159

func areaCircle(radius float64) float64 {
	return pi * radius * radius
}

func printCircleArea(radius float64) {
	area := areaCircle(radius)
	fmt.Println("Area of the circle is:", area)
}

func main() {
	printCircleArea(5)
}
```

### Step-by-Step Explanation

1. The value of π is stored in one constant:

   ```go
   const pi = 3.14159
   ```

2. `areaCircle` contains the circle-area formula:

   ```go
   return pi * radius * radius
   ```

3. `printCircleArea` does not repeat the formula.

4. Instead, it calls `areaCircle`:

   ```go
   area := areaCircle(radius)
   ```

5. If the calculation changes, only `areaCircle` needs to be updated.

6. All callers automatically use the updated implementation.

The formula now has one authoritative representation.

> In real Go programs, `math.Pi` can be used instead of declaring a custom `pi` constant.

For example:

```go
import "math"

func areaCircle(radius float64) float64 {
	return math.Pi * radius * radius
}
```

---

## 8. Backend Example: Repeated Validation

Consider two HTTP handlers that validate an email address separately.

### Breaking DRY

```go
func createUser(email string) error {
	if email == "" {
		return errors.New("email is required")
	}

	if !strings.Contains(email, "@") {
		return errors.New("invalid email address")
	}

	// Create the user.
	return nil
}

func updateUser(email string) error {
	if email == "" {
		return errors.New("email is required")
	}

	if !strings.Contains(email, "@") {
		return errors.New("invalid email address")
	}

	// Update the user.
	return nil
}
```

### Problem

The same email-validation rules are repeated in both functions.

If the rules change, both functions must be updated.

### Applying DRY

```go
package main

import (
	"errors"
	"strings"
)

func validateEmail(email string) error {
	if email == "" {
		return errors.New("email is required")
	}

	if !strings.Contains(email, "@") {
		return errors.New("invalid email address")
	}

	return nil
}

func createUser(email string) error {
	if err := validateEmail(email); err != nil {
		return err
	}

	// Create the user.
	return nil
}

func updateUser(email string) error {
	if err := validateEmail(email); err != nil {
		return err
	}

	// Update the user.
	return nil
}
```

### Step-by-Step Explanation

1. `validateEmail` contains the email-validation rules.

2. It checks whether the email is empty.

3. It checks whether the email contains `@`.

4. Both `createUser` and `updateUser` call the same validation function.

5. If the validation rules change, only `validateEmail` needs to be modified.

6. Both operations will then use the updated rules.

> This example uses simplified email validation for learning purposes. Production applications should use validation appropriate to their requirements.

---

## 9. Advantages of DRY

- Changes usually need to be made in only one place.
- Reduces bugs caused by inconsistent updates.
- Improves maintainability.
- Promotes code reuse.
- Makes business rules easier to locate.
- Makes testing easier because shared logic can be tested independently.
- Keeps the codebase more consistent and organized.

---

## 10. Disadvantages and Risks

### 10.1 Over-Abstraction

Trying to remove every repeated line can create complicated abstractions that are harder to understand than the duplication.

### 10.2 Tight Coupling

Unrelated modules may become dependent on the same shared component, making them harder to change independently.

### 10.3 Premature Abstraction

Creating a generic solution before understanding the actual requirements can slow development and introduce unnecessary complexity.

### 10.4 Difficult-to-Understand Helpers

A shared function that handles too many unrelated cases may become difficult to read, test, and maintain.

DRY should improve clarity. If an abstraction makes the code significantly harder to understand, it may not be useful.

---

## 11. DRY vs Over-Abstraction

DRY does not mean that every repeated line must immediately be extracted.

Consider these functions:

```go
func activateUser(user *User) {
	user.Status = "active"
}

func publishArticle(article *Article) {
	article.Status = "active"
}
```

Both functions assign `"active"` to a status field, but they represent different business operations:

- One activates a user.
- The other publishes an article.

Creating one generic function only because the code looks similar may incorrectly couple unrelated concepts.

### Important Question

Before extracting duplicated code, ask:

> Do these pieces of code represent the same business rule, or do they only look similar?

If they represent the same rule, abstraction may be useful.

If they have different reasons to change, keeping them separate may be safer.

---

## 12. Practical Guidelines

### Understand the Duplication

Determine whether the repeated code represents the same knowledge or only has a similar structure.

### Avoid Premature Abstraction

Do not create a reusable framework for something used only once.

### Use the Rule of Three as Guidance

A common guideline is to consider abstraction when similar logic appears three times.

This is not a strict rule. If a critical business rule is duplicated twice, centralizing it earlier may still be appropriate.

### Prefer Clear Duplication Over a Bad Abstraction

Small duplication is sometimes easier to maintain than a complicated shared function.

### Keep Business Rules Centralized

Important rules such as pricing, permissions, discounts, taxes, and eligibility should have one authoritative implementation.

### Refactor When the Pattern Becomes Clear

It is easier to design a useful abstraction after understanding what is genuinely common between the implementations.

---

## 13. Quick Summary

- **DRY** means **Don't Repeat Yourself**.
- A rule or piece of knowledge should have one authoritative representation.
- DRY reduces inconsistent changes and maintenance effort.
- Repeated business rules, validations, constants, and queries are common DRY violations.
- Shared logic can be moved into reusable functions, services, repositories, or configuration.
- DRY is not about removing every repeated line.
- Similar-looking code may represent different business concepts.
- Avoid premature or overly complex abstractions.
- Apply DRY when it makes the code easier and safer to change.