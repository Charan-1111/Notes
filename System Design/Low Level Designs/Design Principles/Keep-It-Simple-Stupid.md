# Keep It Simple, Stupid (KISS) Principle

## 1. What Is KISS?

KISS means **Keep It Simple, Stupid**. The principle says that a design or solution usually works best when it is kept as simple as possible.

The goal is not to write the fewest lines of code. The goal is to write the **simplest correct solution that is easy to understand, test, maintain, and change**.

### Simple analogy

Suppose you need to travel to a shop one kilometre away:

- Walking directly to the shop is simple and solves the problem.
- Building a new vehicle for the journey is unnecessary complexity.

Software is similar. Do not build a complicated system when a straightforward solution meets the current requirement.

---

## 2. Why KISS Matters in Backend Development

Backend applications often grow over time. If the initial design is unnecessarily complicated, every new feature becomes harder to implement.

KISS helps because simple code is usually:

- Easier to read and review.
- Easier to test and debug.
- Faster to modify.
- Less likely to contain hidden bugs.
- Easier for new developers to understand.

For example, if an API only needs to fetch a user by ID, start with a clear handler and service method. Do not immediately introduce multiple factories, adapters, event handlers, and generic repositories unless the requirements justify them.

---

## 3. When to Use KISS

### Designing features and APIs

Prefer clear endpoints and request formats over clever or highly generic designs.

Example:

```text
GET /users/42
```

This is clearer than creating one generic endpoint that performs many unrelated operations based on several request parameters.

### Writing business logic

Use direct conditions and meaningful names. Avoid complicated control flow when a few clear checks are enough.

### Refactoring

Remove duplicated logic, unnecessary layers, dead code, and abstractions that no longer provide value.

### Onboarding developers

A simple package structure and predictable request flow help a new developer become productive quickly.

### Optimizing performance

Start with clear, correct code. Optimize only after measurements show a real bottleneck.

---

## 4. How We Break KISS

We break KISS when we add complexity without a current, proven need.

Common examples include:

- Adding unnecessary abstractions or layers.
- Using a design pattern only because it is popular.
- Writing generic code for hypothetical future requirements.
- Using clever expressions that reduce readability.
- Splitting a simple operation across too many packages.
- Optimizing before measuring performance.

### Example

Suppose an application supports only email notifications. Creating a notification engine with factories, registries, plugins, and multiple interfaces may be unnecessary. A clear `SendEmail` function might be enough.

If the application later needs email, SMS, and push notifications with interchangeable providers, introducing an interface may then be justified.

KISS does not mean **never use abstractions**. It means **introduce them when they solve a real problem**.

---

## 5. How to Apply KISS

- Start with the simplest solution that correctly meets the requirement.
- Choose meaningful variable, function, and package names.
- Keep each function focused on one responsibility.
- Prefer straightforward control flow.
- Add abstractions when they remove real duplication or isolate real variation.
- Measure before optimizing.
- Delete unused code instead of keeping it for an imagined future.
- Use KISS together with **YAGNI: You Aren't Gonna Need It**.

A useful question during code review is:

> Can another developer understand why this code exists and how it works without a long explanation?

If the answer is no, check whether the design can be simplified.

---

## 6. KISS in Go

Go encourages simple and explicit programs.

### Use small interfaces

Define an interface where it is consumed, and include only the methods the consumer needs.

```go
type UserFinder interface {
	FindByID(ctx context.Context, id int64) (User, error)
}
```

This interface is easier to understand and implement than a large interface containing every possible user operation.

### Handle errors explicitly

```go
user, err := service.FindUser(ctx, id)
if err != nil {
	return err
}
```

The failure path is visible. A developer does not need to discover hidden exception behaviour.

### Keep the package structure proportional to the application

A small service may only need packages such as `handler`, `service`, and `repository`. Add more layers when the application develops a real need for them.

### Prefer the standard library when it is sufficient

External frameworks and libraries are useful, but every dependency introduces concepts, updates, and maintenance work. Use one when its benefit is greater than its cost.

---

## 7. Fibonacci Example in Go

Assume the current requirement is simply: **calculate a Fibonacci number for a small non-negative integer**.

### More complicated than necessary

```go
package main

import "fmt"

func fibonacci(n int, memo map[int]int) int {
	if value, exists := memo[n]; exists {
		return value
	}

	if n <= 1 {
		return n
	}

	memo[n] = fibonacci(n-1, memo) + fibonacci(n-2, memo)
	return memo[n]
}

func main() {
	memo := make(map[int]int)
	fmt.Println("Fibonacci(5):", fibonacci(5, memo))
}
```

### Step-by-step explanation

1. `main` creates a map called `memo`.
2. `fibonacci(5, memo)` calls the function recursively.
3. The base case returns `0` or `1` when `n <= 1`.
4. Every other result is calculated from `fibonacci(n-1)` and `fibonacci(n-2)`.
5. Calculated values are stored in the map so they are not calculated again.

Memoization avoids repeated calculations, so this version has approximately **O(n)** time complexity. However, it also uses recursion, a map, and additional function calls. For this requirement, those elements make the solution harder to follow than necessary.

> Recursion with memoization is not inherently bad. It is useful for many dynamic-programming problems. It is simply unnecessary for this small Fibonacci requirement.

### Applying KISS

```go
package main

import "fmt"

func fibonacci(n int) int {
	if n <= 1 {
		return n
	}

	a, b := 0, 1

	for i := 2; i <= n; i++ {
		a, b = b, a+b
	}

	return b
}

func main() {
	fmt.Println("Fibonacci(5):", fibonacci(5))
}
```

### Step-by-step explanation

1. If `n` is `0` or `1`, the function directly returns `n`.
2. `a` stores the previous Fibonacci number, initially `0`.
3. `b` stores the current Fibonacci number, initially `1`.
4. The loop starts at `2` and calculates each following number.
5. `a, b = b, a+b` moves both values forward by one position.
6. After the loop, `b` contains the required Fibonacci number.

For `n = 5`, the values change as follows:

| Iteration | `a` | `b` |
|---:|---:|---:|
| Initial | 0 | 1 |
| `i = 2` | 1 | 1 |
| `i = 3` | 1 | 2 |
| `i = 4` | 2 | 3 |
| `i = 5` | 3 | 5 |

### Comparison

| Approach | Time | Extra space | Main trade-off |
|---|---:|---:|---|
| Recursion with memoization | O(n) | O(n) | Useful technique, but unnecessary machinery here |
| Iterative | O(n) | O(1) | Direct, readable, and memory-efficient |

---

## 8. Backend API Example

Suppose an API must check whether a user can update a task.

### Unnecessarily complicated

```go
type PermissionStrategy interface {
	CanExecute(user User, task Task) bool
}

type TaskUpdatePermissionStrategy struct{}

func (TaskUpdatePermissionStrategy) CanExecute(user User, task Task) bool {
	return user.IsAdmin || user.ID == task.OwnerID
}

type PermissionExecutor struct {
	strategy PermissionStrategy
}

func (e PermissionExecutor) Execute(user User, task Task) bool {
	return e.strategy.CanExecute(user, task)
}
```

### Step-by-step explanation

1. An interface represents a permission strategy.
2. A concrete type implements that interface.
3. An executor stores the strategy.
4. The executor delegates the permission check to it.
5. The actual rule is still only one Boolean expression.

This structure may be useful when the application truly has many interchangeable permission strategies. For one stable rule, it adds several types without making the rule clearer.

### Simpler implementation

```go
func canUpdateTask(user User, task Task) bool {
	return user.IsAdmin || user.ID == task.OwnerID
}
```

### Step-by-step explanation

1. The function receives the user and task.
2. An administrator is allowed to update the task.
3. The task owner is also allowed to update it.
4. Everyone else receives `false`.

The business rule is immediately visible. If permission rules later become numerous or configurable, the design can evolve at that time.

---

## 9. Advantages

- Improves readability and maintainability.
- Reduces development time and cognitive load.
- Makes testing and debugging easier.
- Helps new developers understand the system.
- Reduces bugs caused by unnecessary complexity.

---

## 10. Limitations and Trade-offs

- The simplest implementation may need to change when requirements grow.
- A very simple algorithm may not be efficient at a much larger scale.
- Removing every abstraction can create duplication and tight coupling.
- Security, correctness, reliability, and scalability must not be ignored in the name of simplicity.

KISS does not mean choosing a careless shortcut. A solution must first be correct and safe. Among the correct solutions, prefer the clearest one that meets the known requirements.

---

## 11. Practical KISS Checklist

Before completing a design or pull request, ask:

- Does this solve a current requirement?
- Can a new developer understand it easily?
- Is every layer or abstraction necessary?
- Can the control flow be made clearer?
- Are the names descriptive?
- Is the function doing one focused job?
- Am I preparing for an imagined requirement?
- Do measurements justify this optimization?
- Can the solution be safely simplified?

---

## 12. Key Takeaways

- KISS means choosing the simplest **correct and maintainable** solution.
- Simple does not mean careless, incomplete, or unscalable.
- Avoid abstractions and optimizations without a real need.
- Add complexity only when requirements or evidence justify it.
- Write code for developers to understand first and computers to execute second.

> Good backend code is not impressive because it looks complicated. It is impressive because it solves a complicated problem clearly.