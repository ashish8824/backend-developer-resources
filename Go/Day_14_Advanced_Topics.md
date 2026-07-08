# Day 14: Advanced Topics

> **Goal for today:** Learn Generics (Go 1.18+), the `context` package for timeouts and cancellation, memory management/garbage collection basics, and common Go idioms and best practices. These are modern, frequently-asked interview topics that show depth of Go knowledge.

---

## 1. Generics (Go 1.18+)

### The Problem Generics Solve (Concept First)

Imagine you want to write a function that finds the maximum of two numbers. Without generics, you'd need a SEPARATE function for each type:

```go
func MaxInt(a, b int) int {
    if a > b {
        return a
    }
    return b
}

func MaxFloat(a, b float64) float64 {
    if a > b {
        return a
    }
    return b
}
```

This is repetitive — the LOGIC is identical, only the TYPE differs. Before Go 1.18 (released in 2022), Go had no clean way to write ONE function that worked across multiple types while still keeping full type safety (the alternative was using `interface{}`/`any`, but that loses compile-time type checking and requires manual type assertions, as we saw on Day 7 — messy and error-prone).

**Generics** solve this: they let you write a function or type that works with MULTIPLE types, specified as a PARAMETER, while still getting full compile-time type safety.

**Analogy:** Think of a generic function like a **vending machine template** that can be configured to dispense ANY product (chips, soda, candy) — the MACHINERY (how to insert coins, how to dispense) stays the same, but you specify WHAT it dispenses when you set it up. Without generics, you'd need a completely separate machine built from scratch for each product type, even though they work identically.

### Writing a Generic Function

```go
package main

import "fmt"

func Max[T int | float64](a, b T) T {
    if a > b {
        return a
    }
    return b
}

func main() {
    fmt.Println(Max(3, 7))         // 7 (works with int)
    fmt.Println(Max(3.5, 2.1))     // 3.5 (works with float64)
}
```

**Breakdown — this is genuinely new syntax, let's go slow:**
- `func Max[T int | float64](a, b T) T {` — the square brackets `[T int | float64]` right after the function name declare a **type parameter**. `T` is a placeholder name we're inventing (you can call it anything, `T` is just a common convention, standing for "Type") — it represents "whatever specific type is used when this function is actually called."
- `int | float64` — this is a **type constraint**, restricting `T` to ONLY be `int` OR `float64` (using `|` to mean "or," similar to how we've used `|` for OR conditions before). This means you CANNOT call `Max("hello", "world")` with strings — Go's compiler would reject it, since `string` isn't in the allowed list.
- `(a, b T) T` — now `a` and `b` are both of type `T` (whatever specific type gets used), and the function returns a `T` too.
- When we CALL `Max(3, 7)`, Go automatically figures out that `T` should be `int` this time, based on the arguments provided (this automatic figuring-out is called **type inference** — you usually don't need to explicitly specify the type yourself).

### Built-in Constraint: `comparable`

Go provides some built-in constraints for common needs. `comparable` restricts `T` to any type that supports `==` and `!=` comparisons:

```go
func Contains[T comparable](slice []T, target T) bool {
    for _, item := range slice {
        if item == target {
            return true
        }
    }
    return false
}

func main() {
    numbers := []int{1, 2, 3, 4}
    fmt.Println(Contains(numbers, 3)) // true

    names := []string{"Amit", "Priya"}
    fmt.Println(Contains(names, "Zara")) // false
}
```
This ONE function now works for a slice of `int`s, a slice of `string`s, or ANY other comparable type — without writing separate versions, and WITHOUT losing type safety (unlike using `interface{}`).

### Generic Types (Not Just Functions)

Generics also work with STRUCTS, letting you build reusable data structures:

```go
package main

import "fmt"

type Stack[T any] struct {
    items []T
}

func (s *Stack[T]) Push(item T) {
    s.items = append(s.items, item)
}

func (s *Stack[T]) Pop() (T, bool) {
    var zero T
    if len(s.items) == 0 {
        return zero, false
    }
    lastIndex := len(s.items) - 1
    item := s.items[lastIndex]
    s.items = s.items[:lastIndex]
    return item, true
}

func main() {
    intStack := &Stack[int]{}
    intStack.Push(1)
    intStack.Push(2)
    intStack.Push(3)

    value, ok := intStack.Pop()
    fmt.Println(value, ok) // 3 true

    stringStack := &Stack[string]{}
    stringStack.Push("hello")
    stringStack.Push("world")

    strValue, _ := stringStack.Pop()
    fmt.Println(strValue) // world
}
```

**Breakdown:**
- `type Stack[T any] struct { items []T }` — defines a generic struct. `Stack[int]` becomes a stack that only holds `int`s; `Stack[string]` becomes a stack that only holds `string`s — same underlying code, specialized per type.
- `func (s *Stack[T]) Push(item T)` — methods on a generic struct also need to reference the type parameter `[T]` in the receiver.
- `var zero T` — this declares a variable of type `T` without an initial value, which gives us the ZERO VALUE for whatever `T` happens to be (`0` for int, `""` for string, etc.) — useful here as a placeholder to return when the stack is empty (paired with `ok = false` so the caller knows not to trust this value, exactly like the "comma ok" pattern from Days 4/7/10!).

**Why does this matter for interviews?** Generics are relatively NEW to Go (added in 2022) and represent one of the language's biggest evolutions. Interviewers often ask "what are generics, and why were they added to Go?" — a good answer highlights: avoiding code duplication across types, while keeping full compile-time type safety (unlike the old `interface{}`-based workarounds).

---

## 2. The `context` Package

The `context` package is used for controlling the LIFETIME of operations — particularly useful for setting timeouts, deadlines, and cancellation signals across goroutines and API calls (very commonly used in real backend Go code, especially web servers).

### Why is `context` needed?

Imagine your server calls a database, which takes too long, or a client disconnects before your server finishes processing their request. Without a way to SIGNAL "stop working on this, it's no longer needed," your program might waste resources continuing pointless work. `context` provides a standard, consistent way to propagate these "please stop" signals through your program, including across goroutines.

### `context.Background()` and `context.WithTimeout()`

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func slowOperation(ctx context.Context) error {
    select {
    case <-time.After(3 * time.Second): // simulates a slow task taking 3 seconds
        fmt.Println("Operation completed!")
        return nil
    case <-ctx.Done(): // triggered if the context's deadline/cancellation fires first
        return ctx.Err()
    }
}

func main() {
    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel() // always call cancel to release resources, even if not timed out

    err := slowOperation(ctx)
    if err != nil {
        fmt.Println("Error:", err) // Output: Error: context deadline exceeded
    }
}
```

**Breakdown — combining several days' concepts:**
- `context.Background()` — creates a base, empty context — the typical starting point when you don't have an existing context to build on (like at the very start of `main()` or handling a new incoming request).
- `context.WithTimeout(context.Background(), 2*time.Second)` — creates a NEW context that will automatically "expire" (signal cancellation) after 2 seconds. It returns the new context AND a `cancel` function.
- `defer cancel()` — the returned `cancel` function should ALWAYS be called eventually (even if the timeout never triggers naturally), to properly release resources associated with the context — a textbook `defer` use case from Day 8.
- Inside `slowOperation`, we use `select` (Day 10!) to race between the actual work finishing (`time.After(3 * time.Second)`) and the context's deadline being reached (`ctx.Done()`). Since our operation takes 3 seconds but our timeout is only 2 seconds, `ctx.Done()` fires FIRST, and we return `ctx.Err()` — which produces the message "context deadline exceeded."

### `context.WithCancel()` — Manual Cancellation

```go
ctx, cancel := context.WithCancel(context.Background())

go func() {
    time.Sleep(1 * time.Second)
    cancel() // manually trigger cancellation after 1 second
}()

<-ctx.Done()
fmt.Println("Context was cancelled:", ctx.Err())
```
This lets you manually decide WHEN to cancel an operation (rather than a fixed timeout), useful for things like "cancel this operation if the user clicks a cancel button."

### Practical Use Case: HTTP Requests with Context

```go
req, _ := http.NewRequestWithContext(ctx, "GET", "https://example.com", nil)
resp, err := http.DefaultClient.Do(req)
```
This attaches our timeout/cancellation context DIRECTLY to an HTTP request — if the context expires before the request completes, the request is automatically aborted. This is an extremely common real-world pattern in production Go services, to avoid requests hanging indefinitely.

**Interview tip:** A common question is *"why would you use `context` in a web server?"* — the key answer: to propagate cancellation/timeout signals across an entire request's lifecycle (including any downstream calls it makes, like to a database or another API), ensuring resources aren't wasted on work that's no longer needed (e.g., because the client already disconnected, or a reasonable time limit was exceeded).

---

## 3. Memory Management & Garbage Collection Basics

### Stack vs Heap (Quick Conceptual Overview)

Programs use two main areas of memory:
- **Stack** — fast, automatically managed memory used for local variables within a function call. When a function returns, its stack memory is automatically reclaimed immediately.
- **Heap** — memory used for data that needs to outlive a single function call (e.g., something referenced by a pointer that's returned or stored elsewhere). Heap memory isn't automatically reclaimed the instant a function returns — it needs a separate cleanup process.

### Garbage Collection (GC)

Go has a built-in **garbage collector** — an automatic background process that identifies HEAP memory that's no longer being used/referenced by anything in your program, and reclaims it, freeing up memory for reuse. This means, unlike languages like C, **you never need to manually free memory in Go** — the garbage collector handles it for you automatically.

**Analogy:** Think of the garbage collector like an automatic housekeeping service that periodically walks through your house, identifies items nobody is using anymore (nothing is "holding onto" them), and clears them out — without you needing to remember to throw anything away yourself.

### Escape Analysis — Does a Variable Go on the Stack or Heap?

Go's compiler automatically decides whether a variable should live on the stack or heap, through a process called **escape analysis**. If the compiler determines a variable's lifetime could extend beyond its function's execution (e.g., because a pointer to it is returned or stored somewhere external), it "escapes" to the heap. Otherwise, it stays safely and efficiently on the stack.

```go
func createPerson() *Person {
    p := Person{Name: "Rahul"} // this ESCAPES to the heap!
    return &p                    // because we're returning a pointer to it
}
```
Here, even though `p` is declared as a normal local variable, since we return `&p` (a pointer to it), Go's compiler recognizes that `p` needs to survive beyond `createPerson()`'s own execution — so it allocates `p` on the HEAP instead of the stack, ensuring it remains valid after the function returns (this would actually be dangerous/impossible in languages like C without garbage collection, but Go handles it safely and automatically).

**Interview-level takeaway:** You generally DON'T need to manually manage stack vs heap allocation in Go — the compiler handles it via escape analysis, and the garbage collector reclaims unused heap memory automatically. It's still valuable to UNDERSTAND this happens, since excessive unnecessary heap allocations (e.g., unnecessarily using pointers for very small, short-lived data) can impact performance — a topic that comes up in more advanced performance-focused interviews.

---

## 4. Common Go Idioms & Best Practices

Let's consolidate several well-known "the Go way" principles, some of which we've already touched on, gathered together as a practical checklist — genuinely useful both for interviews and for writing better Go code.

### 1. "Accept interfaces, return structs" (Day 7)
Write flexible function parameters (interfaces), but return concrete, specific types.

### 2. Handle errors immediately, don't ignore them
```go
// Good:
result, err := doSomething()
if err != nil {
    return err
}

// Bad (never do this):
result, _ := doSomething() // silently ignoring a potential error
```

### 3. Keep functions small and focused
Each function should ideally do ONE clear thing well — easier to test, read, and reuse.

### 4. Use meaningful, short variable names
Go favors short, clear names, especially for small scopes (like `i` for a loop counter, `err` for errors) — but favors descriptive names for things with wider scope/importance.

### 5. Avoid unnecessary use of `interface{}`/`any`
Prefer specific types or generics where possible, for better compile-time safety (Day 7 and today's generics section).

### 6. Use `gofmt`
Go includes a built-in code formatter, `gofmt` (or the newer `go fmt`), which automatically formats your code to a single, consistent style. The ENTIRE Go community uses this same tool, which means virtually all Go code looks stylistically consistent — reducing pointless debates about formatting/style. Run it with:
```bash
go fmt ./...
```

### 7. Use `go vet` to catch common mistakes
```bash
go vet ./...
```
This built-in tool analyzes your code for suspicious constructs that might be genuine bugs (like a `Printf` call with mismatched format verbs), even if the code technically compiles fine.

### 8. Prefer composition over inheritance (Day 5)
Use struct embedding rather than trying to force a class-hierarchy-style design.

### 9. Don't use `panic` for ordinary error handling (Day 8)
Reserve `panic` for truly exceptional, unrecoverable situations.

### 10. Close resources with `defer` immediately after opening them
```go
file, err := os.Open("data.txt")
if err != nil {
    return err
}
defer file.Close() // write this IMMEDIATELY after confirming the open succeeded
```

### 11. Use table-driven tests (Day 12)
The idiomatic way to test multiple scenarios cleanly.

### 12. Name packages clearly and keep them focused
A package name should hint at what it provides (like `mathutils`, `httpclient`), avoiding overly generic names like `utils` or `common` when possible, since these tend to become disorganized "dumping grounds" over time.

---

## 5. Common Beginner Mistakes

1. **Overusing generics when a simple interface or concrete type would do** — generics are powerful, but not always necessary; use them when you genuinely need type-safe reuse across multiple types.
2. **Forgetting `defer cancel()` when creating a context** — this can lead to resources not being released properly.
3. **Assuming you need to manually manage memory in Go** — the garbage collector handles this; manual memory management isn't part of everyday Go programming.
4. **Ignoring `go vet` and `gofmt`** — both are quick, free ways to catch mistakes and maintain consistent style; skipping them is a missed, easy win.
5. **Confusing type constraints syntax** — remember, `[T int | float64]` restricts `T` to specific listed types; `[T any]` allows literally any type.

---

## 6. Day 14 Practice Exercise

Write a program that:
1. Writes a generic function `Sum[T int | float64](numbers []T) T` that sums a slice of numbers of either type.
2. Writes a generic struct `Queue[T any]` with `Enqueue(item T)` and `Dequeue() (T, bool)` methods (similar structure to the `Stack` example, but removing from the FRONT instead of the back).
3. Uses `context.WithTimeout` to simulate a task that sometimes finishes in time, and sometimes times out (vary the sleep duration to demonstrate both outcomes).

<details>
<summary>Click to see solution</summary>

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func Sum[T int | float64](numbers []T) T {
    var total T
    for _, n := range numbers {
        total += n
    }
    return total
}

type Queue[T any] struct {
    items []T
}

func (q *Queue[T]) Enqueue(item T) {
    q.items = append(q.items, item)
}

func (q *Queue[T]) Dequeue() (T, bool) {
    var zero T
    if len(q.items) == 0 {
        return zero, false
    }
    item := q.items[0]
    q.items = q.items[1:]
    return item, true
}

func simulateTask(ctx context.Context, duration time.Duration) error {
    select {
    case <-time.After(duration):
        return nil
    case <-ctx.Done():
        return ctx.Err()
    }
}

func main() {
    ints := []int{1, 2, 3, 4}
    fmt.Println("Sum of ints:", Sum(ints))

    floats := []float64{1.5, 2.5, 3.0}
    fmt.Println("Sum of floats:", Sum(floats))

    q := &Queue[string]{}
    q.Enqueue("first")
    q.Enqueue("second")
    val, _ := q.Dequeue()
    fmt.Println("Dequeued:", val)

    ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
    defer cancel()

    if err := simulateTask(ctx, 1*time.Second); err != nil {
        fmt.Println("Task 1 failed:", err)
    } else {
        fmt.Println("Task 1 completed in time!")
    }

    ctx2, cancel2 := context.WithTimeout(context.Background(), 1*time.Second)
    defer cancel2()

    if err := simulateTask(ctx2, 3*time.Second); err != nil {
        fmt.Println("Task 2 failed:", err) // this one times out
    } else {
        fmt.Println("Task 2 completed in time!")
    }
}
```
</details>

---

## 7. Day 14 Interview Questions (Quick Prep)

1. **Q: What problem do generics solve in Go?**
   A: They let you write functions/types that work across multiple types with full compile-time type safety, avoiding code duplication that would otherwise require separate versions per type (or the loss of type safety from using `interface{}`).

2. **Q: What is a type constraint in generics?**
   A: A restriction on what types a type parameter can accept, e.g., `[T int | float64]` limits `T` to those specific types; `comparable` and `any` are common built-in constraints.

3. **Q: What is the `context` package used for?**
   A: Propagating timeouts, deadlines, and cancellation signals across function calls and goroutines, commonly used in web servers and API calls to avoid wasting resources on work that's no longer needed.

4. **Q: Does Go require manual memory management?**
   A: No — Go has an automatic garbage collector that reclaims unused heap memory; you never need to manually free memory.

5. **Q: What is escape analysis?**
   A: The compiler's process of determining whether a variable can safely stay on the stack, or must "escape" to the heap because its lifetime extends beyond its function's execution (e.g., a pointer to it is returned).

6. **Q: Name a few idiomatic Go best practices.**
   A: Accept interfaces/return structs, handle errors immediately, keep functions small, use `gofmt`/`go vet`, prefer composition over inheritance, avoid `panic` for normal errors, use `defer` for cleanup, and use table-driven tests.

---

## Summary of Day 14

- Generics (`[T constraint]`) let you write reusable, type-safe functions and structs that work across multiple types without duplicating code or sacrificing compile-time safety.
- `context.Context` propagates timeouts, deadlines, and cancellation signals across function calls and goroutines — essential for real-world backend services.
- Go automatically manages memory via garbage collection; the compiler's escape analysis decides stack vs heap allocation without developer intervention.
- Idiomatic Go favors: accepting interfaces, returning structs, immediate error handling, small focused functions, `gofmt`/`go vet`, composition over inheritance, sparing use of `panic`, and disciplined `defer`-based cleanup.

**Tomorrow (Day 15 — Final Day!):** We'll do a full Interview Prep session — the most commonly asked Go interview questions with answers, common coding problems, system design considerations where Go shines, and tips for explaining these concepts clearly to others (since you want to teach this too!).
