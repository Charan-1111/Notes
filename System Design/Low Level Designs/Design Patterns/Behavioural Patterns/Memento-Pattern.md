# Memento Design Pattern

The **Memento pattern** is a behavioral design pattern that captures an object’s state so it can be restored later.

The object creates and restores its own snapshots. A separate history manager stores those snapshots without needing to understand their contents.

**Simple example:**

Before editing text, save the editor’s current state.

If the user selects Undo, restore that saved state.

**Command vs Memento:**

- **Command:** Represents an action to execute.
- **Memento:** Represents a state to restore.

They can work together: a command can save a memento before changing an object.

## When to Use

- When implementing undo functionality.
- When an object needs checkpoints that can be restored later.
- When history management should remain separate from state-saving and restoration logic.
- When exposing internal state to the history manager would create unnecessary coupling.

**Example:**

A diagram editor saves the positions and properties of shapes before a change. Undo restores the previous snapshot.

Restoring application state does not automatically reverse external effects such as sent emails or completed payments.

## Components

| Component | Responsibility | Editor example |
|---|---|---|
| **Originator** | Creates snapshots and restores its state from them. | `Editor`. |
| **Memento** | Holds a snapshot that is treated as immutable. | `EditorMemento`. |
| **Caretaker** | Stores and retrieves snapshots without interpreting their contents. | `History`. |

The originator understands the snapshot’s contents. The caretaker manages its lifetime and ordering.

In Go, unexported fields are private to a **package**, not to a particular type. Types in the same package can technically access them, so the caretaker must respect the design boundary.

## Working

For an undo implementation that saves state before each edit:

1. The originator creates a snapshot of its current state.
2. The caretaker stores that snapshot.
3. The edit changes the originator.
4. Undo removes the latest snapshot from history.
5. The originator restores that snapshot.

**Example:**

| Action | Editor content | Saved history, oldest first |
|---|---|---|
| Start | `""` | Empty |
| Save, then type `"Hello"` | `"Hello"` | `""` |
| Save, then type `" World!"` | `"Hello World!"` | `""`, `"Hello"` |
| Undo | `"Hello"` | `""` |
| Undo again | `""` | Empty |

Saving after every edit is also possible, but undo must then skip or remove the current-state snapshot.

In this example, saving **before each edit** keeps the logic simple.

## Pros

- **Preserves encapsulation:** The caretaker does not need to interpret the originator’s state.
- **Simplifies restoration:** The originator controls how its state is saved and restored.
- **Supports multiple checkpoints:** History can retain snapshots from different moments.
- **Separates responsibilities:** The editor manages state, history manages snapshots, and mementos hold saved data.
- **Supports undo/redo designs:** Snapshots provide the state needed for restoration.

Undo/redo still requires explicit history management; snapshots alone do not implement it.

## Cons

- **Memory usage:** Many large snapshots can consume substantial memory.
- **Copying costs:** Capturing and restoring complex state can be expensive.
- **History management:** The caretaker must decide how many snapshots to retain.
- **Mutable-state risks:** A shallow copy may still share data with the live object.
- **Persistence compatibility:** Snapshots stored across application versions may need schema versioning or migration.

## Sample Implementation in Golang

This example saves a snapshot before each edit and handles attempts to undo an empty history.

```go
package main

import "fmt"

// Memento: saved editor state.
// Treat it as immutable after creation.
type EditorMemento struct {
	content string
}

// Originator: owns the current state.
type Editor struct {
	content string
}

func (e *Editor) Type(words string) {
	e.content += words
}

func (e *Editor) Save() EditorMemento {
	return EditorMemento{
		content: e.content,
	}
}

func (e *Editor) Restore(m EditorMemento) {
	e.content = m.content
}

func (e *Editor) GetContent() string {
	return e.content
}

// Caretaker: stores snapshots without inspecting their state.
type History struct {
	snapshots []EditorMemento
}

func (h *History) Push(m EditorMemento) {
	h.snapshots = append(h.snapshots, m)
}

func (h *History) Pop() (EditorMemento, bool) {
	if len(h.snapshots) == 0 {
		return EditorMemento{}, false
	}

	lastIndex := len(h.snapshots) - 1
	snapshot := h.snapshots[lastIndex]

	// Release references held by the removed element.
	h.snapshots[lastIndex] = EditorMemento{}
	h.snapshots = h.snapshots[:lastIndex]

	return snapshot, true
}

// Coordinates undo without inspecting snapshot contents.
func Undo(editor *Editor, history *History) bool {
	snapshot, ok := history.Pop()
	if !ok {
		return false
	}

	editor.Restore(snapshot)
	return true
}

// Client.
func main() {
	editor := &Editor{}
	history := &History{}

	// Save the empty state before the first edit.
	history.Push(editor.Save())
	editor.Type("Hello")

	// Save "Hello" before the second edit.
	history.Push(editor.Save())
	editor.Type(" World!")

	fmt.Printf("Current: %q\n", editor.GetContent())

	if Undo(editor, history) {
		fmt.Printf("After undo: %q\n", editor.GetContent())
	}

	if Undo(editor, history) {
		fmt.Printf("After second undo: %q\n", editor.GetContent())
	}

	if !Undo(editor, history) {
		fmt.Println("Nothing to undo")
	}
}
```

**Output:**

```text
Current: "Hello World!"
After undo: "Hello"
After second undo: ""
Nothing to undo
```

### Step 1: Define the Memento

```go
type EditorMemento struct {
	content string
}
```

The memento stores the editor’s saved content.

It has no methods for changing that content. This implementation treats it as immutable.

The lowercase field prevents direct access from other packages, but code in the same package can still access it.

### Step 2: Let the Originator Save Its State

```go
func (e *Editor) Save() EditorMemento {
	return EditorMemento{
		content: e.content,
	}
}
```

The editor creates a snapshot of its current content.

Strings are immutable in normal Go usage. Later assigning new content to the editor does not change the saved string.

The snapshot therefore preserves the earlier value without needing a deep copy of the string’s bytes.

### Step 3: Let the Originator Restore Its State

```go
func (e *Editor) Restore(m EditorMemento) {
	e.content = m.content
}
```

The editor knows how to interpret and restore its snapshot.

The caretaker does not need to read the `content` field.

This example assumes snapshots belong to the appropriate editor. Systems with multiple originators may need ownership checks.

### Step 4: Store Snapshots in History

```go
func (h *History) Push(m EditorMemento) {
	h.snapshots = append(h.snapshots, m)
}
```

History uses a stack: the most recently saved snapshot is retrieved first.

This matches undo behavior, which reverses the most recent edit first.

### Step 5: Handle Empty History Explicitly

```go
snapshot, ok := history.Pop()
```

The boolean distinguishes:

- A valid snapshot, including one containing empty text.
- An empty history with no snapshot to restore.

Returning a value and boolean avoids passing a nil snapshot to the editor.

Inside `Pop()`, clearing the removed slice element also releases its reference to saved content. The slice’s backing storage may still remain allocated.

### Step 6: Save Before Changing State

```go
history.Push(editor.Save())
editor.Type(" World!")
```

The snapshot captures `"Hello"` before the edit produces `"Hello World!"`.

Undo therefore restores `"Hello"` immediately.

In your original example, the latest snapshot already contained the current text, so the first undo restored the same content.

### Step 7: Coordinate Undo

```go
func Undo(editor *Editor, history *History) bool {
	snapshot, ok := history.Pop()
	if !ok {
		return false
	}

	editor.Restore(snapshot)
	return true
}
```

The function retrieves a snapshot and asks the editor to restore it.

It never inspects the snapshot’s internal state.

### Step 8: Understand Complex Snapshots

Strings keep this example simple.

If the editor stored slices, maps, or pointers, copying the struct could leave the snapshot sharing mutable data with the editor.

For an independent snapshot, copy all mutable state that must be preserved. Depending on the data structure, this may require a deep copy.

### Step 9: Extend to Redo When Needed

A common undo/redo design uses two stacks:

- **Undo stack:** Previous states.
- **Redo stack:** States that were undone.

Before undoing, save the current state to the redo stack. Before redoing, save the current state to the undo stack.

A new edit normally clears the redo stack because it starts a different history path.

The sample implements undo only.
