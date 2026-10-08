# Composite Design Pattern

The **Composite pattern** is a structural design pattern that lets clients perform the same operation on individual objects and groups of objects.

It is useful for **part–whole hierarchies**: structures where a group contains individual items or other groups.

**Simple example:**

- A file is an individual object.
- A directory is a group containing files or other directories.
- Both provide a `Show()` method.

The client can call `Show()` without checking whether the object is a file or a directory.

In Go, we implement this using a **shared interface** and **composition**.

## When to Use

- When representing tree structures such as directories, menus, or UI components.
- When an operation makes sense for both individual items and groups.
- When clients should use the same operation without checking each object's concrete type.

**Example:**

In a UI:

- A button is a leaf.
- A panel contains buttons and other panels.
- Calling `Render()` on a panel renders its children.

Composite is useful when the shared operation has meaningful behavior for both kinds of objects.

## Components

| Component | Responsibility | File system example |
|---|---|---|
| **Component** | Defines operations shared by leaves and composites. | `FileSystemNode`. |
| **Leaf** | Performs the operation directly and has no children. | `File`. |
| **Composite** | Holds child components and coordinates their operations. | `Directory`. |
| **Client** | Uses the shared interface to perform operations. | `DisplayTree()`. |

A composite can contain both leaves and other composites because its children use the shared interface.

## Working

1. The client builds a hierarchy by adding components to directories.
2. The client calls `Show()` on a component.
3. A file displays its own name.
4. A directory displays its name and calls `Show()` on each child.
5. Nested directories repeat this process recursively.

The client does not need file-versus-directory checks during display. Each object's method provides the appropriate behavior.

**Shared operations do not require identical capabilities.**

Both files and directories support `Show()`, but only directories support `Add()`.

## Pros

- **Uniform operations:** Uses the same interface for individual objects and groups.
- **Simpler client code:** Avoids concrete-type checks when performing shared operations.
- **Natural hierarchy representation:** Models nested structures directly.
- **Extensibility:** Adds new component types without changing clients that use the shared interface.
- **Flexible composition:** Builds and extends hierarchies at runtime.

## Cons

- **Harder to enforce child restrictions:** A broad component interface may allow combinations that violate application rules.
- **More difficult tracing:** Deep hierarchies can make recursive operations harder to debug.
- **Traversal cost:** Operations on a group may visit many descendants.
- **Requires structural discipline:** Cycles and unintended shared children can break tree assumptions.

Go interfaces preserve type safety. The challenge is enforcing domain rules—for example, allowing only certain component types inside a particular group.

## Sample Implementation in Golang

This example creates an **in-memory hierarchy**. It does not access actual files or directories.

```go
package main

import "fmt"

// Component: shared operation for files and directories.
type FileSystemNode interface {
	Show(indentation string)
}

// Leaf: has no children.
type File struct {
	name string
}

func (f *File) Show(indentation string) {
	fmt.Println(indentation + f.name)
}

// Composite: can contain files and other directories.
type Directory struct {
	name     string
	children []FileSystemNode
}

func (d *Directory) Add(node FileSystemNode) {
	d.children = append(d.children, node)
}

func (d *Directory) Show(indentation string) {
	fmt.Println(indentation + d.name + "/")

	for _, child := range d.children {
		child.Show(indentation + "  ")
	}
}

// Client operation: depends only on the shared interface.
func DisplayTree(node FileSystemNode) {
	node.Show("")
}

func main() {
	// Create leaves.
	file1 := &File{name: "file1.txt"}
	file2 := &File{name: "file2.txt"}
	file3 := &File{name: "file3.txt"}

	// Create composites.
	documents := &Directory{name: "Documents"}
	pictures := &Directory{name: "Pictures"}
	root := &Directory{name: "Root"}

	// Build the hierarchy.
	documents.Add(file1)
	documents.Add(file2)
	pictures.Add(file3)

	root.Add(documents)
	root.Add(pictures)

	// Display a group through the shared interface.
	DisplayTree(root)

	// The same client operation also accepts a leaf.
	fmt.Println("\nIndividual file:")
	DisplayTree(file1)
}
```

**Output:**

```text
Root/
  Documents/
    file1.txt
    file2.txt
  Pictures/
    file3.txt

Individual file:
file1.txt
```

### Step 1: Define the Component Interface

```go
type FileSystemNode interface {
	Show(indentation string)
}
```

This defines the operation both files and directories support.

Go satisfies interfaces implicitly. Because their methods use pointer receivers, `*File` and `*Directory` satisfy `FileSystemNode`.

### Step 2: Implement the Leaf

```go
type File struct {
	name string
}

func (f *File) Show(indentation string) {
	fmt.Println(indentation + f.name)
}
```

A file displays its name and stops.

It has no children and does not need an `Add()` method.

### Step 3: Store Children Through the Interface

```go
type Directory struct {
	name     string
	children []FileSystemNode
}
```

The slice can contain files and directories.

A directory can therefore contain another directory, allowing arbitrary nesting.

### Step 4: Add Children

```go
func (d *Directory) Add(node FileSystemNode) {
	d.children = append(d.children, node)
}
```

`Add()` appends a component to the directory.

A nil slice supports `append()`, so the directory does not need to initialize `children` explicitly.

Tree construction uses directory-specific methods. Uniform treatment applies when performing the shared operation.

### Step 5: Delegate Recursively

```go
func (d *Directory) Show(indentation string) {
	fmt.Println(indentation + d.name + "/")

	for _, child := range d.children {
		child.Show(indentation + "  ")
	}
}
```

The directory first displays its own name.

It then calls `Show()` on every child, adding two spaces for the next level.

- A file prints its name.
- A directory prints its name and visits its children.

The interface selects the appropriate implementation without an explicit type check.

### Step 6: Keep the Client Independent of Concrete Types

```go
func DisplayTree(node FileSystemNode) {
	node.Show("")
}
```

The client accepts the shared interface.

Both calls are valid:

```go
DisplayTree(root)
DisplayTree(file1)
```

The client performs the same operation whether it receives a group or an individual item.

### Step 7: Preserve the Tree Structure

The example assumes valid, non-nil children and an acyclic hierarchy.

Avoid operations such as:

```go
root.Add(root)
```

This creates a cycle. Calling `Show()` would repeatedly visit the same directory until execution fails.

Indirect cycles are also possible—for example, a child directory containing its parent.

Production code should enforce the structural rules it needs. If each node must have only one parent, that restriction also requires explicit handling.

### Step 8: Understand Traversal Cost

For a valid tree with `n` nodes and height `h`:

- **Node traversal:** O(n), because each node is visited once.
- **Recursive call stack:** O(h), because nested directories add call depth.

These bounds exclude formatting, output, and indentation-string allocation costs.

Very deep hierarchies may benefit from an iterative traversal using an explicit stack.
