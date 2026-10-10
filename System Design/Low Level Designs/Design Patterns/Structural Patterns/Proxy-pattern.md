# Proxy Design Pattern

- A **structural design pattern** that provides a substitute for another object to control access to it.
- The client calls the proxy, and the proxy decides whether and when to call the real object.
- The proxy can add access checks, lazy initialization, caching, or logging without changing the real object's implementation.

**Example:** A security guard controls access to a building. A visitor approaches the guard, who checks permission before allowing entry. Similarly, a proxy can check permission before forwarding a request.

**Key idea:** The proxy and real object implement the same interface, so the client can use either through that interface.

## When to Use

| Use case | What the proxy does | Example |
|---|---|---|
| Access control | Checks permissions before forwarding a request | Allow only authorized users to read a document |
| Lazy initialization | Creates an expensive object when it is first needed | Load an image only when it is displayed |
| Remote proxy | Represents a remote service through a local interface | A client object forwards calls to a service using gRPC |
| Caching | Reuses stored results instead of repeating expensive work | Return a cached product lookup |
| Logging/monitoring | Records activity around an operation | Measure how long a service call takes |

A proxy is useful when you need to **control access or manage the lifecycle of another object**.

## Components

Using an image-loading example:

| Component | Responsibility | Example |
|---|---|---|
| **Subject** | Defines the interface used by the client | `Image` |
| **Real Subject** | Performs the actual operation | `RealImage` |
| **Proxy** | Controls access to the real subject | `ProxyImage` |
| **Client** | Calls operations through the subject interface | `main()` |

Both `RealImage` and `ProxyImage` implement `Image`.

The client only needs to know that it can call `Display()`. It does not need to manage image loading.

## Working

1. The client calls a method on the proxy through the subject interface.
2. The proxy performs any necessary checks or preparation.
3. If needed, the proxy forwards the call to the real object.
4. The real object performs the operation.

The proxy does **not always forward the request**:
- An access-control proxy may reject it.
- A caching proxy may return an existing result.
- A lazy-initialization proxy may create the real object before forwarding it.

**For image loading:**
- First `Display()` call: create the real image, then display it.
- Later calls: reuse the real image and display it again.

## Pros

- **Access control:** Checks permissions before allowing an operation.
- **Lazy initialization:** Avoids creating expensive objects that may never be used.
- **Potential performance improvement:** Caching can reduce repeated work.
- **Additional behavior:** Adds logging or monitoring without modifying the real object.
- **Remote access:** Provides a local interface for calling a remote service.
- **Consistent client code:** The client uses the same interface for the proxy and real object.

These benefits depend on the type of proxy. For example, a logging proxy does not automatically provide caching.

## Cons

- **Increased complexity:** Adds another object and another layer of behavior.
- **Possible overhead:** Permission checks, logging, or other work can increase execution time.
- **Harder debugging:** Understanding an operation may require checking both the proxy and real object.
- **Unnecessary abstraction:** A proxy may be excessive when direct access is sufficient.
- **Network limitations:** Remote proxies must handle timeouts, connection failures, and other network errors.
- **Concurrency concerns:** Shared proxy state may require synchronization.
- **Cache consistency:** A caching proxy must decide when stored results become outdated.

## Sample Implementation in Golang

This example demonstrates a **lazy-initialization proxy**.

Loading is simulated with `fmt.Println`; the code does not read an actual image file.

```go
package main

import "fmt"

// Subject: both the real image and proxy implement this interface.
type Image interface {
	Display()
}

// Real Subject: represents the loaded image.
type RealImage struct {
	fileName string
}

func NewRealImage(fileName string) *RealImage {
	fmt.Println("Loading image from disk:", fileName)

	return &RealImage{
		fileName: fileName,
	}
}

func (r *RealImage) Display() {
	fmt.Println("Displaying image:", r.fileName)
}

// Proxy: delays creating the real image until it is needed.
type ProxyImage struct {
	fileName  string
	realImage *RealImage
}

func NewProxyImage(fileName string) *ProxyImage {
	return &ProxyImage{
		fileName: fileName,
	}
}

func (p *ProxyImage) Display() {
	if p.realImage == nil {
		p.realImage = NewRealImage(p.fileName)
	}

	p.realImage.Display()
}

func main() {
	var img Image = NewProxyImage("photo.png")

	fmt.Println("Proxy created; image is not loaded yet.")

	img.Display() // Creates the real image, then displays it.
	img.Display() // Reuses the real image and displays it again.
}
```

### Expected Output

```text
Proxy created; image is not loaded yet.
Loading image from disk: photo.png
Displaying image: photo.png
Displaying image: photo.png
```

### Step-by-Step Code Explanation

**1. Define the common interface**

```go
type Image interface {
	Display()
}
```

The client works with `Image`.

In Go, interface implementation is implicit. Both `*RealImage` and `*ProxyImage` implement `Image` because they provide `Display()`.

**2. Implement the real object**

```go
func NewRealImage(fileName string) *RealImage {
	fmt.Println("Loading image from disk:", fileName)

	return &RealImage{
		fileName: fileName,
	}
}
```

Creating `RealImage` represents the expensive loading operation.

Its `Display()` method represents displaying the loaded image.

**3. Create the proxy without loading the image**

```go
var img Image = NewProxyImage("photo.png")
```

The proxy stores the file name, but its `realImage` field is initially `nil`.

No real image has been created yet.

**4. Initialize the real image on the first call**

```go
if p.realImage == nil {
	p.realImage = NewRealImage(p.fileName)
}
```

The first `Display()` call finds that the real image does not exist, so the proxy creates it.

This is **lazy initialization**: delay creation until the object is needed.

**5. Delegate the operation**

```go
p.realImage.Display()
```

Once the real image exists, the proxy forwards the display operation to it.

**6. Reuse the object on later calls**

On the second call, `realImage` is already set. The proxy skips initialization and calls `Display()` again.

This reuses the loaded object; it does **not** cache the result of `Display()`.

**7. Keep the client dependent on the interface**

The client can use either implementation:

```go
var img Image = NewProxyImage("photo.png")
```

Or:

```go
var img Image = NewRealImage("photo.png")
```

The client still calls `img.Display()`. The difference is that creating `RealImage` loads immediately, while creating `ProxyImage` delays loading.

### Go-Specific Considerations

#### Concurrent Initialization

The basic example is intended for sequential use.

If multiple goroutines call `Display()` concurrently, reading and writing `realImage` without synchronization creates a **data race**. More than one goroutine could also attempt initialization.

For initialization that should happen exactly once, use `sync.Once`:

```go
type ProxyImage struct {
	fileName  string
	realImage *RealImage
	once      sync.Once
}

func (p *ProxyImage) Display() {
	p.once.Do(func() {
		p.realImage = NewRealImage(p.fileName)
	})

	p.realImage.Display()
}
```

Add `"sync"` to the imports when using this version.

Step by step:
1. Every call enters `once.Do`.
2. Only one call executes the initialization function.
3. Other concurrent calls wait for that initialization to finish.
4. Each call then delegates to `realImage.Display()`.

`sync.Once` protects initialization only. If the real object's methods modify shared state, those methods may need their own synchronization.

#### Loading Can Fail

Real file loading can fail because a file is missing, unreadable, or invalid.

A production interface could return an error:

```go
type Image interface {
	Display() error
}
```

Both the proxy and real object would implement this signature. The proxy would return any loading or display error to the client.

If failed initialization should be retried, choose the synchronization approach carefully: `sync.Once` does not rerun its initialization function after it returns.

#### Proxy vs. Decorator

Both patterns can wrap an object and implement the same interface.

- **Proxy:** Primarily controls access, such as checking permission or delaying creation.
- **Decorator:** Primarily adds behavior, such as adding compression to a writer.

Logging can fit either pattern depending on the design's purpose. The distinction is the **intent**, not just the code structure.