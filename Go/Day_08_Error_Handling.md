# Day 8: Error Handling

> **Goal for today:** Understand Go's `error` type and why Go deliberately avoids try/catch, how to create and check custom errors (`errors.New`, `fmt.Errorf`, `errors.Is`, `errors.As`), and a deep dive into `panic`, `recover`, and `defer`. This is one of the most "Go-specific" and heavily interviewed topics.

---

## 1. Go's Philosophy on Errors (Concept First)

If you've looked at Java, Python, or JavaScript, you've probably seen `try/catch` blocks for handling errors:
```java
// Java (NOT Go):
try {
    doSomething();
} catch (Exception e) {
    System.out.println("Error occurred");
}
```

**Go deliberately does NOT have `try/catch`.** This is one of the most debated (and most frequently asked about) design decisions in Go. Instead, Go treats errors as **ordinary values** — just like an `int` or a `string` — that get returned from functions and explicitly checked by the calling code.

### Why did Go's designers choose this approach?

1. **Explicitness** — with try/catch, it's easy to forget to handle an exception, or to accidentally let it "bubble up" silently through many layers of code without anyone noticing. In Go, since errors are just regular return values, you're forced (by convention and often necessity) to explicitly decide what to do with them, right at the point they occur.
2. **No hidden control flow** — with exceptions, a single line of code could secretly jump execution to some `catch` block far away, making code harder to trace. In Go, code flows top-to-bottom, predictably — an error just becomes another value you check with a normal `if` statement.
3. **Errors are expected, not "exceptional"** — Go's philosophy is that things like "file not found" or "network timeout" are NORMAL, expected possibilities in a program, not rare emergency events that deserve a whole separate control-flow mechanism.

**Analogy:** Try/catch is like an alarm system that suddenly interrupts whatever you're doing when something goes wrong, forcing you to a different room (the catch block) to deal with it. Go's approach is like a form where every question has a mandatory "did this go wrong?" checkbox right next to the answer — you handle it right where it happens, not in some separate emergency room.

---

## 2. The `error` Type

`error` is a built-in interface type in Go (remember interfaces from Day 7!). It's defined internally as:
```go
type error interface {
    Error() string
}
```
This means: **anything that has an `Error() string` method automatically counts as an `error`** — consistent with everything we learned about implicit interface implementation yesterday.

### The Standard Go Pattern: Returning `(result, error)`

You've already seen a preview of this back on Day 3. Now let's understand it fully.

```go
package main

import (
    "errors"
    "fmt"
)

func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, errors.New("cannot divide by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 2)
    if err != nil {
        fmt.Println("Error:", err)
        return
    }
    fmt.Println("Result:", result)

    result, err = divide(10, 0)
    if err != nil {
        fmt.Println("Error:", err) // Output: Error: cannot divide by zero
        return
    }
    fmt.Println("Result:", result)
}
```

**Breakdown — line by line:**
- `import ("errors" "fmt")` — we now import TWO packages. `errors` is the standard library package specifically for creating and working with error values.
- `func divide(a, b int) (int, error)` — this function returns TWO values: the calculated result (`int`), AND an `error`. This is the exact "multiple return values" feature from Day 3, now applied to Go's most common real-world use case.
- `errors.New("cannot divide by zero")` — creates a new error value, holding the given message. `errors.New` is the simplest way to create an error.
- `return 0, errors.New(...)` — when something goes wrong, we return a "zero-ish" result (`0` here, since there's no meaningful answer) ALONGSIDE the actual error explaining what went wrong.
- `return a / b, nil` — when everything succeeds, we return the real result, and `nil` for the error (meaning "no error occurred"). **`nil` is the zero value for the `error` interface type**, similar to how `nil` was the zero value for pointers on Day 6.
- `if err != nil {` — THIS is the standard Go idiom you will see in almost EVERY Go codebase: immediately after calling a function that might fail, check if `err` is not `nil`. If it's not `nil`, something went wrong, and you handle it right there (print it, return early, etc.).

### Visual: The `(result, error)` Pattern

```
   Function call:  result, err := divide(10, 0)
                             │
                             v
              ┌───────────────────────────┐
              │  Did something go wrong?     │
              └───────────┬───────────────┘
                     yes   │   no
              ┌────────────┘   └────────────┐
              v                              v
    err is NOT nil                    err IS nil
    result is meaningless             result holds the real answer
    → handle the error                → safely use result
```

---

## 3. Creating Custom Errors

### Method 1: `errors.New()` (simple, static message)

```go
err := errors.New("something went wrong")
```
Best for simple, fixed error messages with no dynamic data.

### Method 2: `fmt.Errorf()` (formatted, dynamic message)

```go
package main

import "fmt"

func validateAge(age int) error {
    if age < 0 {
        return fmt.Errorf("invalid age: %d, age cannot be negative", age)
    }
    return nil
}

func main() {
    err := validateAge(-5)
    if err != nil {
        fmt.Println(err) // Output: invalid age: -5, age cannot be negative
    }
}
```
`fmt.Errorf` works just like `fmt.Printf` (using format verbs like `%d`, `%s` from Day 1), but instead of printing the text, it returns it wrapped as an `error` value. This is extremely useful when your error message needs to include dynamic details (like the actual invalid value).

### Method 3: Custom Error Types (for more structured/detailed errors)

Sometimes a simple text message isn't enough — you want to attach extra structured data to an error (like an error code, or which field failed validation). You can do this by creating your own struct that implements the `error` interface (remember: it just needs an `Error() string` method).

```go
package main

import "fmt"

type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed on field '%s': %s", e.Field, e.Message)
}

func validateAge(age int) error {
    if age < 0 {
        return &ValidationError{Field: "age", Message: "cannot be negative"}
    }
    return nil
}

func main() {
    err := validateAge(-5)
    if err != nil {
        fmt.Println(err)
        // Output: validation failed on field 'age': cannot be negative
    }
}
```

**Breakdown:**
- `ValidationError` is a normal struct, with `Field` and `Message` fields, giving us STRUCTURED information about what went wrong (not just a plain text blob).
- `func (e *ValidationError) Error() string` — because this struct has an `Error() string` method (using a pointer receiver, as discussed on Day 5, since it's common practice for error types), it automatically satisfies the `error` interface.
- We return `&ValidationError{...}` — a POINTER to our custom error struct (pointer receivers require this, as we learned on Day 6).
- `fmt.Sprintf` is like `fmt.Printf`, but instead of printing directly to the screen, it returns the formatted text as a `string` — useful when building a string you'll use elsewhere (like inside `Error()` here).

This pattern is used constantly in production Go code, especially when different parts of the program need to check specifically WHICH kind of error occurred, not just read a text message.

---

## 4. `errors.Is` and `errors.As` (Modern Error Checking)

As Go codebases grew larger, developers needed better tools for checking error types and "wrapped" errors (an error containing another error inside it, adding more context as it travels up through function calls). The `errors` package provides two key functions for this.

### Error Wrapping with `%w`

```go
package main

import (
    "errors"
    "fmt"
)

var ErrNotFound = errors.New("item not found")

func findItem(id int) error {
    if id != 1 {
        return fmt.Errorf("findItem failed: %w", ErrNotFound)
    }
    return nil
}

func main() {
    err := findItem(5)

    if errors.Is(err, ErrNotFound) {
        fmt.Println("The item was not found!")
    }

    fmt.Println(err) // Output: findItem failed: item not found
}
```

**Breakdown:**
- `var ErrNotFound = errors.New("item not found")` — we define a package-level "sentinel error" (a pre-defined error VALUE that other code can check against later). This is a common Go pattern.
- `fmt.Errorf("findItem failed: %w", ErrNotFound)` — the special `%w` verb (instead of `%s` or `%v`) WRAPS the original error inside a new one, adding extra context ("findItem failed:") while still preserving a link back to the original error underneath.
- `errors.Is(err, ErrNotFound)` — checks: "does `err` (possibly wrapped multiple layers deep) match `ErrNotFound` somewhere inside it?" This returns `true` even though `err`'s actual text message is now different ("findItem failed: item not found"), because `errors.Is` looks THROUGH the wrapping to find the original error.

**Why is this useful?** Imagine a database function returns "item not found," and that error travels up through 3-4 layers of your application, each layer adding more context ("failed to process order: failed to find item: item not found"). Without wrapping, you'd lose track of what the ORIGINAL underlying problem was. `errors.Is` lets you check "was this ultimately caused by ErrNotFound?" no matter how many layers of context were added on top.

### `errors.As` — Extracting a Specific Custom Error Type

While `errors.Is` checks if an error MATCHES a specific error VALUE, `errors.As` checks if an error matches a specific error TYPE, and gives you access to it (including any extra structured data it might contain).

```go
var validationErr *ValidationError

if errors.As(err, &validationErr) {
    fmt.Println("Field with issue:", validationErr.Field)
}
```
This says: "if `err` is (or wraps) a `*ValidationError` somewhere, extract it into `validationErr` so I can access its `Field` and `Message` data directly."

**Quick distinction (a common interview question):**
| Function | Purpose |
|---|---|
| `errors.Is(err, targetErr)` | "Does this error match this SPECIFIC error VALUE (even if wrapped)?" |
| `errors.As(err, &targetType)` | "Does this error match this specific error TYPE, and can I extract it to access its extra data?" |

---

## 5. `panic` and `recover`

While `error` handles EXPECTED, recoverable problems, Go also has `panic` for truly EXCEPTIONAL, unexpected situations — things so severe the program genuinely cannot continue safely (like a nil pointer dereference, which we saw on Day 6, or accessing an array index that doesn't exist).

### `panic` — Immediately Stops Normal Execution

```go
package main

import "fmt"

func riskyOperation() {
    panic("something catastrophic happened!")
}

func main() {
    fmt.Println("Before risky operation")
    riskyOperation()
    fmt.Println("After risky operation") // this line NEVER runs
}
```
Output:
```
Before risky operation
panic: something catastrophic happened!

goroutine 1 [running]:
...
exit status 2
```
When `panic` is called, the program immediately stops executing normal code, starts "unwinding" (backing out of) all the function calls currently in progress, and eventually crashes the program entirely — UNLESS something catches it using `recover`.

### `recover` — Catching a Panic (Go's Closest Thing to try/catch)

```go
package main

import "fmt"

func riskyOperation() {
    defer func() {
        if r := recover(); r != nil {
            fmt.Println("Recovered from panic:", r)
        }
    }()

    panic("something catastrophic happened!")
}

func main() {
    fmt.Println("Before risky operation")
    riskyOperation()
    fmt.Println("After risky operation") // this DOES run now!
}
```
Output:
```
Before risky operation
Recovered from panic: something catastrophic happened!
After risky operation
```

**Breakdown:**
- `recover()` is a built-in function that stops a panic from crashing the whole program — but it ONLY works when called directly inside a `defer`red function (we'll explain `defer` fully in the next section). This is a strict Go rule.
- If `recover()` is called while a panic IS happening, it returns the value passed to `panic()` (here, the string message), and normal execution resumes right after the function where the panic was recovered.
- If there's NO panic happening, `recover()` just returns `nil` and does nothing.

**Important guidance:** In idiomatic Go, `panic`/`recover` is used SPARINGLY — mainly for truly unexpected, unrecoverable situations (like a critical programming bug), or at the top level of a program/goroutine to prevent a single failure from crashing an entire application (common in web servers, so one bad request doesn't take down the whole server). For NORMAL, expected error conditions (file not found, invalid input, network failure), always prefer returning a regular `error` value instead. **Interviewers often specifically ask: "should you use panic for normal error handling?" — the correct answer is NO.**

---

## 6. `defer` — Deferring Execution Until the Function Returns

We used `defer` above without fully explaining it — let's do that now, since it's essential and used constantly in Go.

`defer` schedules a function call to run LATER — specifically, right before the surrounding function returns/finishes, regardless of HOW it finishes (whether it returns normally, or even if a panic occurs).

```go
package main

import "fmt"

func main() {
    fmt.Println("Start")
    defer fmt.Println("This runs LAST, even though it's written here")
    fmt.Println("Middle")
    fmt.Println("End")
}
```
Output:
```
Start
Middle
End
This runs LAST, even though it's written here
```

**Breakdown:** Even though the `defer` line appears second in the code, its execution is postponed until JUST BEFORE `main()` actually finishes — so it runs LAST, after everything else in the function body has completed.

### Why is `defer` so useful? — Resource Cleanup

The most common real-world use of `defer` is ensuring cleanup code runs, no matter what happens (success, early return, or even a panic) — like closing a file after you're done reading it, or closing a database connection.

```go
package main

import (
    "fmt"
    "os"
)

func readFile(filename string) {
    file, err := os.Open(filename)
    if err != nil {
        fmt.Println("Error opening file:", err)
        return
    }
    defer file.Close() // guaranteed to run when readFile() finishes, no matter how

    fmt.Println("File opened successfully, reading...")
    // ... reading logic would go here ...
}
```

**Why this matters:** Without `defer`, you'd have to remember to manually call `file.Close()` at EVERY possible exit point of the function (every `return` statement, every error path) — easy to forget one, causing a "resource leak" (a file or connection staying open longer than it should, eventually causing performance problems). `defer file.Close()` guarantees it happens exactly once, right when the function is about to exit, no matter which path led there.

### Multiple `defer` calls — LIFO Order (Last In, First Out)

```go
func main() {
    defer fmt.Println("1")
    defer fmt.Println("2")
    defer fmt.Println("3")
    fmt.Println("Main logic running")
}
```
Output:
```
Main logic running
3
2
1
```
**Important interview point:** If you have multiple `defer` statements, they execute in REVERSE order — the LAST one you `defer` is the FIRST one to actually run. This is called **LIFO order** (Last In, First Out) — think of it like a stack of plates: the last plate you put ON TOP is the first one you take back OFF.

---

## 7. Putting It All Together: `panic`, `recover`, and `defer`

```go
package main

import "fmt"

func safeDivide(a, b int) (result int, err error) {
    defer func() {
        if r := recover(); r != nil {
            err = fmt.Errorf("recovered from panic: %v", r)
        }
    }()

    result = a / b // this will PANIC if b is 0 (division by zero)
    return result, nil
}

func main() {
    result, err := safeDivide(10, 0)
    if err != nil {
        fmt.Println("Error:", err)
    } else {
        fmt.Println("Result:", result)
    }
}
```
Output:
```
Error: recovered from panic: runtime error: integer divide by zero
```

**Breakdown:** This combines several Day 8 concepts elegantly:
- `(result int, err error)` — named return values (from Day 3).
- Dividing by zero with `int` types causes a genuine runtime PANIC in Go (unlike floating point division by zero, which produces `+Inf`).
- Our `defer`red function calls `recover()`. If a panic occurred, it converts that panic into a normal `error`, assigning it to the named return value `err`.
- The caller (`main`) never sees a crash — it just sees a normal returned `error`, exactly like any other Go error-handling situation. This is a genuinely useful, idiomatic pattern for "containing" panics at a function boundary and converting them into ordinary error values.

---

## 8. Common Beginner Mistakes

1. **Forgetting to check `err != nil` after a function call** — this is the #1 Go mistake. Always check errors immediately after receiving them.
2. **Using `panic` for normal, expected errors** — reserve `panic` for truly exceptional, unrecoverable situations. Use `error` returns for everything else.
3. **Calling `recover()` outside of a `defer`red function** — it only works when called directly inside a deferred function; otherwise it does nothing.
4. **Assuming `defer` runs immediately** — it doesn't; it always waits until the surrounding function is about to return.
5. **Forgetting the LIFO order of multiple `defer` calls** — the most recently deferred call runs first.
6. **Comparing errors with `==` instead of `errors.Is`** when dealing with wrapped errors — direct `==` comparison won't "see through" wrapping, while `errors.Is` will.

---

## 9. Day 8 Practice Exercise

Write a program that:
1. Defines a custom error type `NegativeNumberError` with a `Number` field, implementing the `error` interface.
2. Writes a function `squareRoot(n float64) (float64, error)` that returns a `NegativeNumberError` if `n` is negative, otherwise returns the square root (use `math.Sqrt` from the standard `math` package).
3. Uses `errors.As` in `main()` to check specifically if the returned error is a `NegativeNumberError`, and if so, print the offending number.
4. Adds a `defer` statement in `main()` that prints "Program finished" right before the program ends.

<details>
<summary>Click to see solution</summary>

```go
package main

import (
    "errors"
    "fmt"
    "math"
)

type NegativeNumberError struct {
    Number float64
}

func (e *NegativeNumberError) Error() string {
    return fmt.Sprintf("cannot take square root of negative number: %.2f", e.Number)
}

func squareRoot(n float64) (float64, error) {
    if n < 0 {
        return 0, &NegativeNumberError{Number: n}
    }
    return math.Sqrt(n), nil
}

func main() {
    defer fmt.Println("Program finished")

    result, err := squareRoot(-9)
    if err != nil {
        var negErr *NegativeNumberError
        if errors.As(err, &negErr) {
            fmt.Println("Negative number error, value was:", negErr.Number)
        }
        return
    }
    fmt.Println("Result:", result)
}
```
</details>

---

## 10. Day 8 Interview Questions (Quick Prep)

1. **Q: Why doesn't Go have try/catch?**
   A: Go treats errors as ordinary values returned from functions, promoting explicit handling right where they occur, avoiding hidden control flow, and treating expected failures as normal, not "exceptional."

2. **Q: What's the difference between `error` and `panic`?**
   A: `error` is used for expected, recoverable failure conditions and returned as a normal value. `panic` is used for truly severe, unexpected situations that halt normal execution, and should be used sparingly.

3. **Q: What does `recover()` do, and where must it be called?**
   A: It stops a panic from crashing the program and returns the panic's value. It must be called directly inside a `defer`red function to have any effect.

4. **Q: What is `defer`, and what order do multiple defers run in?**
   A: `defer` schedules a function call to run right before the surrounding function returns. Multiple defers run in LIFO order (last deferred, first executed).

5. **Q: What's the difference between `errors.Is` and `errors.As`?**
   A: `errors.Is` checks whether an error matches a specific error VALUE (seeing through any wrapping). `errors.As` checks whether an error matches a specific error TYPE, extracting it so you can access its fields.

6. **Q: What is the most common real-world use of `defer`?**
   A: Resource cleanup — like closing files, database connections, or network connections — guaranteeing it happens regardless of how the function exits.

---

## Summary of Day 8

- Go treats errors as ordinary return values (no try/catch), following the `(result, error)` pattern.
- `error` is an interface requiring an `Error() string` method; `nil` means "no error."
- Create errors with `errors.New()` (simple), `fmt.Errorf()` (formatted/dynamic), or custom structs implementing `Error() string` (structured data).
- `%w` in `fmt.Errorf` wraps an error, preserving its identity for `errors.Is`/`errors.As` checks even through multiple layers of added context.
- `panic` halts normal execution for severe, unexpected situations; `recover` (only inside `defer`) can catch a panic and prevent a crash.
- `defer` schedules a function call to run just before the surrounding function returns, commonly used for guaranteed resource cleanup; multiple defers run in LIFO order.

**Tomorrow (Day 9):** We start Concurrency — one of Go's most famous strengths. We'll learn Goroutines: what they are, how they differ from traditional threads, the `go` keyword, and WaitGroups for coordinating multiple goroutines.
