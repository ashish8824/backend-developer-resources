# Day 12: Testing in Go

> **Goal for today:** Learn how to write tests using Go's built-in `testing` package, master the idiomatic table-driven test pattern, write benchmarks, and understand the basics of mocking for testing code that depends on interfaces. Testing is a heavily emphasized part of professional Go development and comes up often in interviews.

---

## 1. Why Testing Matters (Concept First)

A **test** is code that automatically checks whether OTHER code behaves the way you expect. Instead of manually running your program and eyeballing the output every time you make a change, you write a test ONCE, and then you can re-run it instantly, as many times as you want, to confirm your code still works correctly — especially useful after making changes.

**Analogy:** Think of testing like a **quality control checklist** at a factory. Instead of a human inspecting every single product by hand each time (slow, error-prone, inconsistent), you set up an automated checking machine ONCE that quickly verifies "does this product meet the required specifications?" Every new batch runs through the SAME checklist automatically.

### Why is Go's testing story particularly notable?

Unlike many languages that need an external testing framework/library, **Go includes testing capability built directly into the language and standard library** (the `testing` package), along with a built-in command (`go test`) to run them. This is intentional — Go's designers wanted testing to be a natural, frictionless, first-class part of everyday development, not an afterthought requiring extra setup.

---

## 2. Writing Your First Test

Go has strict, specific conventions for how test files and functions must be named and structured — the tooling relies on these conventions to automatically find and run your tests.

### The Rules
1. Test files must end with `_test.go` (e.g., `calculator_test.go`).
2. Test functions must start with `Test`, followed by a capitalized word (e.g., `TestAdd`).
3. Test functions must accept exactly one parameter: `t *testing.T`.

### Example: Testing a Simple Function

**calculator.go:**
```go
package calculator

func Add(a, b int) int {
    return a + b
}
```

**calculator_test.go:**
```go
package calculator

import "testing"

func TestAdd(t *testing.T) {
    result := Add(2, 3)
    expected := 5

    if result != expected {
        t.Errorf("Add(2, 3) = %d; expected %d", result, expected)
    }
}
```

**Breakdown:**
- `import "testing"` — brings in Go's built-in testing package.
- `func TestAdd(t *testing.T) {` — following the naming rule (`Test` + capitalized name), and accepting `t *testing.T`, the special testing object Go provides you.
- We call the actual function being tested (`Add(2, 3)`), and compare the result to what we EXPECT (`5`).
- `t.Errorf(...)` — this is how you report a test FAILURE. `Errorf` works just like `fmt.Printf` (Day 1) — it accepts a formatted message — but specifically marks THIS test as failed, and prints the given message as an explanation of what went wrong. Importantly, `Errorf` does NOT stop the test function immediately — it just marks a failure and continues (useful if you want to check multiple things in one test and see ALL the failures, not just the first one).

### Running Tests

```bash
go test
```
Output (if passing):
```
PASS
ok      myproject/calculator    0.002s
```
Output (if failing):
```
--- FAIL: TestAdd (0.00s)
    calculator_test.go:8: Add(2, 3) = 4; expected 5
FAIL
```

For more detail on which specific tests ran:
```bash
go test -v
```
Output:
```
=== RUN   TestAdd
--- PASS: TestAdd (0.00s)
PASS
ok      myproject/calculator    0.002s
```

---

## 3. `t.Errorf` vs `t.Fatalf` — An Important Distinction

| Function | Behavior |
|---|---|
| `t.Errorf(...)` | Marks the test as FAILED, but CONTINUES running the rest of the test function |
| `t.Fatalf(...)` | Marks the test as FAILED, and IMMEDIATELY STOPS the rest of the test function |

```go
func TestSomething(t *testing.T) {
    result, err := riskyOperation()
    if err != nil {
        t.Fatalf("unexpected error: %v", err) // stop here, no point continuing
    }

    if result != 10 {
        t.Errorf("expected 10, got %d", result) // report and continue checking more
    }
}
```

**When to use which?** Use `t.Fatalf` when continuing the test would be POINTLESS or could cause a crash (like trying to use a value that failed to be created properly, e.g., a nil pointer). Use `t.Errorf` when you want to report a problem but it's still safe/useful to keep checking other things in the same test.

---

## 4. Table-Driven Tests (The Idiomatic Go Pattern — VERY Important!)

This is one of the most distinctly "Go" patterns, and interviewers frequently ask about it or expect to see it in code samples. Instead of writing a SEPARATE test function for every possible input/output scenario, you define a "table" (a slice of test cases) and loop through it, running the SAME test logic against each one.

```go
package calculator

import "testing"

func TestAddTableDriven(t *testing.T) {
    tests := []struct {
        name     string
        a, b     int
        expected int
    }{
        {"positive numbers", 2, 3, 5},
        {"negative numbers", -2, -3, -5},
        {"mixed numbers", -2, 3, 1},
        {"zeros", 0, 0, 0},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := Add(tt.a, tt.b)
            if result != tt.expected {
                t.Errorf("Add(%d, %d) = %d; expected %d", tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```

**Breakdown — this combines MANY concepts from previous days:**
- `tests := []struct { ... }{ ... }` — an anonymous struct (Day 5 concept, structs) inside a slice (Day 4 concept) — each entry represents ONE test case, with a descriptive `name`, input values (`a`, `b`), and the `expected` result.
- `for _, tt := range tests {` — a standard for-range loop (Day 2/4) over our test cases.
- `t.Run(tt.name, func(t *testing.T) { ... })` — this runs a NAMED SUB-TEST for each table entry. `t.Run` accepts a name (for identifying this specific sub-test in output) and an anonymous function (Day 3 concept, closures) containing the actual test logic.

### Why is this pattern so valued in Go?

1. **DRY (Don't Repeat Yourself)** — you write the testing LOGIC once, and just add new rows to the table to cover more scenarios, instead of duplicating near-identical test functions.
2. **Clear, readable output** — since each case is a named sub-test, when running with `-v`, you get a clean breakdown of EXACTLY which specific case passed or failed:
```
=== RUN   TestAddTableDriven
=== RUN   TestAddTableDriven/positive_numbers
=== RUN   TestAddTableDriven/negative_numbers
=== RUN   TestAddTableDriven/mixed_numbers
=== RUN   TestAddTableDriven/zeros
--- PASS: TestAddTableDriven (0.00s)
    --- PASS: TestAddTableDriven/positive_numbers (0.00s)
    --- PASS: TestAddTableDriven/negative_numbers (0.00s)
    --- PASS: TestAddTableDriven/mixed_numbers (0.00s)
    --- PASS: TestAddTableDriven/zeros (0.00s)
```
3. **Easy to extend** — adding a new test case is as simple as adding one more line to the table, rather than writing a whole new function.

**Interview tip:** If asked "how would you write tests in Go for a function with many different input scenarios," confidently describe the table-driven test pattern — it's considered THE idiomatic Go answer.

---

## 5. Testing Functions That Return Errors

Since Go's error handling (Day 8) is central to the language, testing error-returning functions is extremely common:

```go
package calculator

import "testing"

func TestDivide(t *testing.T) {
    tests := []struct {
        name        string
        a, b        int
        expected    int
        expectError bool
    }{
        {"valid division", 10, 2, 5, false},
        {"division by zero", 10, 0, 0, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result, err := Divide(tt.a, tt.b)

            if tt.expectError {
                if err == nil {
                    t.Errorf("expected an error, but got none")
                }
                return
            }

            if err != nil {
                t.Errorf("unexpected error: %v", err)
            }
            if result != tt.expected {
                t.Errorf("Divide(%d, %d) = %d; expected %d", tt.a, tt.b, result, tt.expected)
            }
        })
    }
}
```
This pattern checks BOTH the success case (correct result, no error) AND the failure case (an error was genuinely returned when expected), combining the table-driven approach with Day 8's error-checking conventions.

---

## 6. `go test` Common Flags

| Command | Purpose |
|---|---|
| `go test` | Run all tests in the current package |
| `go test ./...` | Run all tests in the current directory AND all subdirectories (whole project) |
| `go test -v` | Verbose output — shows every individual test run |
| `go test -run TestAdd` | Run only tests matching the given name pattern |
| `go test -cover` | Shows test coverage percentage (how much of your code is actually exercised by tests) |

---

## 7. Benchmarks — Measuring Performance

Go's `testing` package also supports **benchmarks** — tests specifically designed to measure how FAST a piece of code runs, useful for comparing different implementations or catching performance regressions.

```go
package calculator

import "testing"

func BenchmarkAdd(b *testing.B) {
    for i := 0; i < b.N; i++ {
        Add(2, 3)
    }
}
```

**Breakdown:**
- Benchmark functions must start with `Benchmark` (instead of `Test`), and accept `b *testing.B` instead of `t *testing.T`.
- `b.N` — Go automatically determines an appropriate large number of iterations to run the loop, adjusting it to get a stable, statistically meaningful timing measurement. You don't set this number yourself — the testing framework figures out how many repetitions produce a reliable result.
- Inside the loop, you call the code you want to measure.

### Running Benchmarks

```bash
go test -bench=.
```
Output:
```
BenchmarkAdd-8       1000000000    0.25 ns/op
```
This tells you: the benchmark ran on 8 CPU cores (`-8`), completed roughly 1 billion iterations, and each individual call to `Add` took about 0.25 nanoseconds on average (`ns/op` = "nanoseconds per operation").

**Practical use case:** If you have two different ways to implement the same function, you can write a benchmark for each and directly compare their `ns/op` values to decide which is actually faster in practice — removing guesswork from performance decisions.

---

## 8. Mocking Basics — Testing Code That Depends on Interfaces

Recall Day 7: interfaces let you write flexible code that works with ANY type satisfying certain methods. This becomes extremely useful for TESTING — you can create a FAKE ("mock") version of a dependency, specifically for use in tests, replacing a real, complicated dependency (like a database or external API) with a simple, predictable, controllable stand-in.

### Example Scenario: Testing Code That Depends on an Interface

```go
package notifier

type MessageSender interface {
    Send(message string) error
}

type OrderService struct {
    sender MessageSender
}

func NewOrderService(sender MessageSender) *OrderService {
    return &OrderService{sender: sender}
}

func (o *OrderService) PlaceOrder(item string) error {
    return o.sender.Send("Order placed: " + item)
}
```

**Breakdown:**
- `OrderService` depends on a `MessageSender` INTERFACE, not a specific concrete implementation (like a real email or SMS system). This is exactly the "accept interfaces" principle from Day 7.
- In REAL production use, you'd pass in a genuine `EmailSender` or `SMSSender` (like we built on Day 7).
- But for TESTING, we don't want to actually send real emails/texts every time we run our tests! Instead, we create a fake, mock implementation.

### Creating a Mock for Testing

```go
package notifier

import "testing"

type MockSender struct {
    SentMessages []string
    ShouldFail   bool
}

func (m *MockSender) Send(message string) error {
    if m.ShouldFail {
        return errors.New("mock send failure")
    }
    m.SentMessages = append(m.SentMessages, message)
    return nil
}

func TestPlaceOrder(t *testing.T) {
    mock := &MockSender{}
    service := NewOrderService(mock)

    err := service.PlaceOrder("Laptop")
    if err != nil {
        t.Errorf("unexpected error: %v", err)
    }

    if len(mock.SentMessages) != 1 {
        t.Errorf("expected 1 message sent, got %d", len(mock.SentMessages))
    }

    if mock.SentMessages[0] != "Order placed: Laptop" {
        t.Errorf("unexpected message: %s", mock.SentMessages[0])
    }
}
```

**Breakdown:**
- `MockSender` is a struct we created purely for testing purposes. It implements the `Send(message string) error` method (satisfying the `MessageSender` interface implicitly, exactly as Day 7 taught us), but instead of ACTUALLY sending anything, it just records the message into a slice (`SentMessages`) so our test can inspect what WOULD have been sent.
- `ShouldFail` lets us simulate a failure scenario on demand, by setting it to `true` in a different test case, to verify our `OrderService` correctly handles errors from its dependency too.
- Since `NewOrderService` accepts anything satisfying `MessageSender`, we can pass in our `MockSender` instead of a real sender — this is ONLY possible because `OrderService` was designed around an INTERFACE rather than a specific concrete type. This is exactly why Day 7's "accept interfaces" principle matters so much for writing genuinely testable Go code.

**This is the foundation of mocking in Go** — there ARE popular third-party libraries (like `testify/mock`, `gomock`) that provide more advanced/automated mocking tools, but understanding this manual, interface-based approach is essential, since it's the underlying mechanism those libraries build upon.

---

## 9. Common Beginner Mistakes

1. **Forgetting the `_test.go` filename suffix** — Go's tooling won't recognize the file as containing tests without it.
2. **Forgetting the `Test` prefix on test function names**, or not capitalizing the word after it (e.g., `Testadd` instead of `TestAdd` — Go requires the letter right after `Test` to be uppercase, following the exported-name convention from Day 11).
3. **Using `t.Fatalf` when you actually wanted to continue checking other things** — remember, `Fatalf` stops the test immediately; `Errorf` lets it continue.
4. **Writing many nearly-identical test functions** instead of using the table-driven pattern — a missed opportunity for cleaner, more maintainable tests.
5. **Designing code around concrete types instead of interfaces**, making it hard or impossible to substitute mocks during testing — a good reason to keep Day 7's "accept interfaces" principle in mind even when just writing regular application code, not just tests.

---

## 10. Day 12 Practice Exercise

Write a program that:
1. Creates a package `stringutils` with a function `IsPalindrome(s string) bool` that checks if a string reads the same forwards and backwards (ignore case).
2. Writes a table-driven test `TestIsPalindrome` covering at least 4 cases: a true palindrome, a non-palindrome, an empty string, and a single character.
3. BONUS: Writes a benchmark `BenchmarkIsPalindrome` for this function.

<details>
<summary>Click to see solution</summary>

**stringutils/palindrome.go:**
```go
package stringutils

import "strings"

func IsPalindrome(s string) bool {
    s = strings.ToLower(s)
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}
```

**stringutils/palindrome_test.go:**
```go
package stringutils

import "testing"

func TestIsPalindrome(t *testing.T) {
    tests := []struct {
        name     string
        input    string
        expected bool
    }{
        {"palindrome word", "Racecar", true},
        {"non-palindrome word", "Hello", false},
        {"empty string", "", true},
        {"single character", "a", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := IsPalindrome(tt.input)
            if result != tt.expected {
                t.Errorf("IsPalindrome(%q) = %v; expected %v", tt.input, result, tt.expected)
            }
        })
    }
}

func BenchmarkIsPalindrome(b *testing.B) {
    for i := 0; i < b.N; i++ {
        IsPalindrome("Racecar")
    }
}
```
</details>

---

## 11. Day 12 Interview Questions (Quick Prep)

1. **Q: What naming conventions must Go test files and functions follow?**
   A: Test files must end in `_test.go`; test functions must start with `Test` followed by a capitalized word, and accept `t *testing.T`.

2. **Q: What's the difference between `t.Errorf` and `t.Fatalf`?**
   A: `t.Errorf` marks the test as failed but continues running; `t.Fatalf` marks it as failed and immediately stops the test function.

3. **Q: What is a table-driven test, and why is it considered idiomatic in Go?**
   A: A pattern where multiple test cases are defined as a slice of structs (a "table"), then looped through with a single shared test logic block (often using `t.Run` for named sub-tests). It's idiomatic because it avoids duplicating test logic and produces clear, per-case output.

4. **Q: How do you run a benchmark in Go, and what does `b.N` represent?**
   A: Using `go test -bench=.`; `b.N` is automatically determined by the testing framework to run enough iterations for a statistically stable timing measurement.

5. **Q: How does Go's interface system help with mocking in tests?**
   A: Since code can depend on an interface rather than a concrete type, you can substitute a fake/mock implementation (satisfying the same interface) during tests, avoiding real, complex, or slow dependencies like databases or external services.

6. **Q: What command shows test coverage in Go?**
   A: `go test -cover`.

---

## Summary of Day 12

- Go has built-in testing support via the `testing` package and `go test` command — no external framework required for basics.
- Test files end in `_test.go`; test functions start with `Test`, accepting `t *testing.T`.
- `t.Errorf` reports a failure and continues; `t.Fatalf` reports a failure and stops immediately.
- Table-driven tests (a slice of struct test cases, looped with `t.Run`) are the idiomatic Go pattern for covering multiple scenarios cleanly.
- Benchmarks (`Benchmark` prefix, `b *testing.B`, `b.N` iterations) measure code performance via `go test -bench=.`.
- Interfaces (Day 7) enable mocking — substituting fake implementations of dependencies during tests, keeping tests fast, predictable, and independent of real external systems.

**Tomorrow (Day 13):** We'll cover File Handling, JSON, and building a simple REST API — reading/writing files, encoding/decoding JSON with struct tags, and using `net/http` to build and call HTTP endpoints.
