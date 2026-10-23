# Memento Design Pattern
- Behavioural design pattern that lets us capture and store an objects internal state so it can be restored later, without exposing its internal implementation details, and without violating encapsulation.
- It is commonly used for undo/redo functionality. where an object's state can be reverted to a previous state safely and transparently.

- ## When to Use
	- When we need to implement undo/redo functionality.
	- When we want to support checkpointing or versioning of an object's state.
	- When we want to separate the concerns of state storage from state management logic.

- ## Components
	- **Originator** - The object whose state needs to be saved and restored. It creates a memento encapsulating its current state and can restore its state from a memento.
	- **Memento** - An immutable object that stores the snapshot of an originator's internal state. It restricts access to this state to maintain encapsulation.
	- **Caretaker** - Manages and keeps track of memento objects, often storing them in a stack. It request saving or restoring the originator's state but does not modify or inspect the mementos

- ## Working
	1. Originator creates a memento that represents a snapshot of its current state.
	2. Caretaker keeps the memento as a history checkpoint.
	3. When needed, the caretaker can request the originator to restore its state from a previously saved memento.
	4. This mechanism allows undo functionality by rolling back to earlier states without exposing the internal details of the originator.

- ## Pros
	- Preserves encapsulation - Internal state is stored without exposing the originator's private fields.
	- Supports undo/redo functionality - Makes it straight forward to rollback or restore previous states.
	- Simplifies state restoration - The originator knows how to save and restore itself, so the caretaker doesn't need to understood the details.
	- Multiple Checkpoints possible - We can keep several mementos for different points in time and restore any of them.
	- Separation of Concerns - Caretaker manages the history, Originator manages the state, and memento just holds the snapshot.

- ## Cons
- High memory usage - Storing many snapshots ( especially with large states ) can consume a lot of memory.
- Performance Overhead - Creating and restoring mementos repeatedly may slowdown the system.
- Caretaker responsibility - Caretaker must manage the lifecycle of mementos.
	- Eg :- When to discard old ones to free memory.
- Serialization issues - If the originator's structure changes frequently, keeping mementos compatible becomes difficult.

- ## Sample implementation in Golang

```go
// Text editor with Undo

package main

import "fmt"

// Memento
type EditorMemento struct {
	content string
}

// Originator
type Editor struct {
	content string
}

func (e *Editor) Type(words string) {
	e.content += words
}

func (e *Editor) Save() *EditorMemento {
	return &EditorMemento{content: e.content}
}

func (e *Editor) Restore(m *EditorMemento) {
	e.content = m.content
}

func (e *Editor) GetContent() string {
	return e.content
}

// Care Taker
type History struct {
	snapshots []*EditorMemento
}

func (h *History) Push(m *EditorMemento) {
	h.snapshots = append(h.snapshots, m)
}

func (h *History) Pop() *EditorMemento {
	if len(h.snapshots) == 0 {
		return nil
	}

	last := h.snapshots[len(h.snapshots)-1]
	h.snapshots = h.snapshots[:len(h.snapshots)-1]

	return last
}

// Client
func main() {
	editor := &Editor{}
	history := &History{}

	editor.Type("Hello")
	history.Push(editor.Save())

	editor.Type("World!")
	history.Push(editor.Save())

	fmt.Println("Current : ", editor.GetContent())

	// Undo
	editor.Restore(history.Pop())
	fmt.Println("After Undo : ", editor.GetContent())

	// Undo Again
	editor.Restore(history.Pop())
	fmt.Println("After second undo : ", editor.GetContent())
}
```
