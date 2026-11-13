# MVC Design Pattern

**Model–View–Controller (MVC)** is an architectural pattern that separates an application into three parts:

- **Model:** Owns the application’s data and business rules.
- **View:** Presents that data to the user.
- **Controller:** Interprets user input and coordinates what happens next.

**Example:** When someone edits a user profile, the controller receives the proposed name and age, the model decides whether the change is valid, and the view displays the resulting profile.

## When to Use

- When an application has business rules you want to test independently of its UI.
- When input handling, data rules, and presentation change for different reasons.
- When the same data needs different presentations, such as a profile page and an admin view.

For a small script that reads input and prints a result, a few functions may be enough. MVC is useful when separating the responsibilities makes changes easier to make and test.

## Components

| Component | Responsibility | User profile example |
|---|---|---|
| **Model** | Holds data and enforces rules when it changes. | `User` rejects blank names and negative ages. |
| **View** | Formats data for display. It can contain presentation logic, such as labels or date formatting. | `UserView` prints the profile. |
| **Controller** | Receives input, calls the model, and chooses what to render or return. | `UserController` sends edits to `User` and asks `UserView` to display it. |

The model can hold data in memory, as it does here. A database is a way to persist data; it is not the model itself.

## Working

1. The caller sends a proposed profile update to the controller.
2. The controller passes the name and age to the model.
3. The model validates both values before changing either field. If validation fails, the previous profile remains intact.
4. The caller asks the controller to render the current profile, and the controller passes it to the view.

**In a server-rendered Go application**, the browser submits a form to an HTTP handler acting as the controller. The handler parses the form, calls the model, and then renders a response or redirects to the profile page. Parsing `"30"` into an integer is input handling; deciding whether a negative age is allowed is a model rule.

Some interactive MVC applications use Observer-style notifications to refresh views when the model changes. That is optional. This console example renders explicitly by calling `Render()`.

## Pros

- **Easier testing:** Test the model’s validation and update rules without rendering a UI.
- **Easier presentation changes:** Switch from console output to HTML without rewriting the profile rules.
- **Reusable business rules:** Different controllers can call the same model operations.
- **Clear ownership:** There is a clear place to put input handling, business rules, and presentation.

## Cons

- **Extra structure:** Separate components can be unnecessary for small programs.
- **More code to trace:** Following one action may take you through a controller, a model, and a view.
- **Risk of oversized controllers:** Putting business rules in handlers makes them harder to reuse and easier to duplicate.
- **Shared contracts still matter:** Changing the shape of the data can require updates in both the controller and the view.

## Sample Implementation in Golang

The example displays Alice’s profile, updates it to Bob’s, and then attempts an invalid update. The last render shows that the invalid update did not change either field.

```go
package main

import (
	"errors"
	"fmt"
	"strings"
)

// Model: owns profile data and enforces update rules.
type User struct {
	name string
	age  int
}

func (u *User) UpdateProfile(name string, age int) error {
	name = strings.TrimSpace(name)

	if name == "" {
		return errors.New("name must not be blank")
	}

	if age < 0 {
		return errors.New("age must not be negative")
	}

	// Apply the update only after both values pass validation.
	u.name = name
	u.age = age
	return nil
}

// View: formats the profile for display.
type UserView struct{}

func (UserView) Display(user User) {
	fmt.Printf("User: %s, Age: %d\n", user.name, user.age)
}

// Controller: coordinates updates and rendering.
type UserController struct {
	model *User
	view  UserView
}

func (c *UserController) SetUser(name string, age int) error {
	return c.model.UpdateProfile(name, age)
}

func (c *UserController) Render() {
	c.view.Display(*c.model)
}

func main() {
	controller := UserController{
		model: &User{name: "Alice", age: 25},
		view:  UserView{},
	}

	controller.Render()

	if err := controller.SetUser("Bob", 30); err != nil {
		fmt.Println("Update failed:", err)
	}
	controller.Render()

	if err := controller.SetUser("Charlie", -1); err != nil {
		fmt.Println("Update failed:", err)
	}
	controller.Render()
}
```

**Expected output:**

```text
User: Alice, Age: 25
User: Bob, Age: 30
Update failed: age must not be negative
User: Bob, Age: 30
```

### Step-by-Step Explanation

1. **Put the rules in the model.** `UpdateProfile()` trims the name and validates both inputs before assigning either field. When `SetUser("Charlie", -1)` runs, the age check fails before the name can change to Charlie.

2. **Keep formatting in the view.** `UserView.Display()` decides how to print the name and age. It receives a copy of `User` in this example, so displaying the profile does not require changing the stored data.

3. **Let the controller coordinate.** `SetUser()` passes the update to the model and returns any error. `Render()` passes the current profile to the view. The controller holds a `*User`, so both operations work with the same model instance.

4. **Keep rendering explicit.** `main()` calls `Render()` after each attempt. A successful edit displays the new profile; a rejected edit displays the previous one. No notification mechanism is needed for this flow.

All three components are in one Go package for clarity. MVC is about who owns each responsibility; it does not require three separate packages.