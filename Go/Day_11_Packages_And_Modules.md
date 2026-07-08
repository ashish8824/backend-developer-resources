# Day 11: Packages & Modules

> **Goal for today:** Learn how to properly organize Go code into packages, understand `go.mod`/`go.sum`, the exported vs unexported rule (capital letter convention), and take a practical tour of essential standard library packages you'll use constantly (`fmt`, `strings`, `strconv`, `time`, `os`).

---

## 1. What is a Package? (Recap and Deep Dive)

Back on Day 1, we briefly mentioned `package main`. Now let's fully understand packages, since organizing code well is essential for any real project (and a common interview/practical topic).

A **package** is simply a way of grouping related Go code files together. Every `.go` file belongs to exactly ONE package, declared at the very top with `package packageName`.

**Analogy:** Think of a package like a labeled drawer in a filing cabinet. Instead of throwing every document loose into one giant pile (one massive file with everything), you group related documents into labeled drawers — a "Finance" drawer, an "HR" drawer, etc. Each drawer (package) can be opened (imported) by whoever needs what's inside.

### Why organize code into packages?
- **Reusability** — code in a package can be imported and reused by other parts of your program, or even by OTHER programs entirely.
- **Organization** — related functionality stays together, making large codebases easier to navigate.
- **Encapsulation** — packages can hide certain implementation details, exposing only what's meant to be used externally (we'll cover this with the exported/unexported rule shortly).

---

## 2. `package main` vs Other Packages

We've used `package main` in every example so far. Let's clarify what makes it special.

- **`package main`** — this is a special, reserved package name. A `main` package MUST contain a `func main()`, and it tells Go "this is a runnable program," not just a reusable library. When you `go build` or `go run` a `main` package, Go produces an executable program.
- **Any other package name** (like `package mathutils`, `package models`) — these are LIBRARY packages, meant to be imported and used by OTHER code, but cannot be run directly on their own (no `func main()` needed or expected).

### Typical Project Structure

```
myproject/
├── go.mod
├── main.go              (package main)
└── mathutils/
    └── calculator.go     (package mathutils)
```

**Important Go convention:** The FOLDER name typically matches the package name declared inside the `.go` files within it. So the folder `mathutils/` would contain files starting with `package mathutils`.

---

## 3. Creating and Using Your Own Package

Let's build a small real example.

### Step 1: Create the project

```bash
mkdir myproject
cd myproject
go mod init myproject
```

### Step 2: Create a package folder with a file inside it

Create `mathutils/calculator.go`:
```go
package mathutils

func Add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}
```

### Step 3: Use this package from `main.go`

Create `main.go` in the project's root folder:
```go
package main

import (
    "fmt"
    "myproject/mathutils"
)

func main() {
    result := mathutils.Add(5, 3)
    fmt.Println("Sum:", result)
}
```

**Breakdown:**
- `import "myproject/mathutils"` — the import path is built from your module's name (declared in `go.mod`, here `myproject`) plus the folder path to the package (`mathutils`).
- `mathutils.Add(5, 3)` — we access the `Add` function using dot notation, just like we accessed functions from `fmt` (`fmt.Println`) since Day 1. That's literally what's been happening this whole time — `fmt` is just ANOTHER package, one that comes bundled with Go itself (the "standard library").

### Run it
```bash
go run main.go
```
Output:
```
Sum: 8
```

**But wait — why couldn't we call `mathutils.multiply(...)` from `main.go`?** This brings us to a crucial Go rule.

---

## 4. Exported vs Unexported — The Capital Letter Rule (Very Important!)

This is one of Go's simplest but most distinctive rules, and a frequent interview question.

**In Go, whether a function, variable, type, or struct field is accessible OUTSIDE its own package is determined ENTIRELY by whether its name starts with an UPPERCASE or lowercase letter.**

- **Uppercase first letter** (e.g., `Add`, `Person`, `Name`) → **EXPORTED** — visible and usable from OTHER packages that import this one.
- **Lowercase first letter** (e.g., `multiply`, `person`, `name`) → **UNEXPORTED** — only visible and usable WITHIN the same package; completely hidden from any other package that imports it.

```go
package mathutils

func Add(a, b int) int {       // EXPORTED - usable from other packages
    return a + b
}

func multiply(a, b int) int {  // unexported - ONLY usable within mathutils itself
    return a * b
}
```

If you tried `mathutils.multiply(2, 3)` from `main.go`, you'd get a compile error: `multiply is not exported by package mathutils` (or similar) — Go enforces this strictly at compile time, not just as a suggestion.

**Why does Go do this instead of using keywords like `public`/`private` (as in Java)?** It's a deliberate simplicity choice — no extra keywords needed, just a consistent naming convention that's immediately visible just by looking at a name, without needing to check separate access-modifier keywords.

**Analogy:** Think of it like a building with two types of doors: doors with a shiny brass nameplate (capital letter — "Reception," anyone from outside can walk right in) vs plain unmarked doors (lowercase — "Staff Only," only people already working inside that specific department can use them).

### This rule applies EVERYWHERE — not just functions

```go
package models

type Person struct {
    Name string // exported field - accessible from other packages
    age  int    // unexported field - hidden from other packages
}

func NewPerson(name string, age int) Person { // exported constructor-style function
    return Person{Name: name, age: age}
}
```

Here, even though `Person` itself is exported (so other packages CAN create/use a `Person`), the `age` field is unexported — other packages CANNOT directly read or set `p.age`. This is a common, deliberate technique for **encapsulation** — hiding internal details while providing controlled access through exported functions/methods (like a `GetAge()` method, if you wanted to allow reading it, but not directly modifying it).

### Quick Reference Table

| Name Style | Example | Accessible from other packages? |
|---|---|---|
| Starts with UPPERCASE | `Add`, `Person`, `Name`, `MaxRetries` | Yes (exported) |
| Starts with lowercase | `multiply`, `person`, `name`, `maxRetries` | No (unexported, package-private) |

---

## 5. `go.mod` — Understanding the Module File

We first saw `go.mod` on Day 1, created with `go mod init`. Let's understand its contents properly now.

```
module myproject

go 1.22

require (
    github.com/some/package v1.2.3
)
```

**Breakdown:**
- `module myproject` — declares the module's name (this is the base import path used by everything inside your project, as we saw with `myproject/mathutils`).
- `go 1.22` — declares the minimum Go version this module requires/was written for.
- `require (...)` — lists external dependencies (packages written by OTHER people, downloaded from the internet) that your project needs, along with their specific versions.

### Adding an External Dependency

```bash
go get github.com/some/package
```
This downloads the package, adds it to `go.mod` automatically, and makes it available to import in your code.

### `go.sum` — What is it?

Alongside `go.mod`, Go also generates a `go.sum` file. This file contains cryptographic checksums (like unique fingerprints) of every dependency's exact contents. Its purpose is **security and reproducibility** — it ensures that if you (or a teammate, or a deployment server) download the SAME dependencies again later, you get EXACTLY the same code, unmodified, byte-for-byte, matching what was originally verified. You typically never edit `go.sum` by hand — Go manages it automatically.

**Quick interview distinction:**
| File | Purpose |
|---|---|
| `go.mod` | Declares the module name, Go version, and list of dependencies (with version numbers) |
| `go.sum` | Contains checksums verifying the exact, untampered contents of each dependency |

---

## 6. A Practical Tour of Essential Standard Library Packages

Go comes bundled with a rich **standard library** — pre-written, well-tested packages covering common needs, so you don't have to write everything from scratch or rely on third-party code for basics. Let's tour the most commonly used ones.

### `fmt` — Formatting and Printing (You already know this one!)

| Function | Purpose |
|---|---|
| `fmt.Println(...)` | Prints values, adds a newline at the end |
| `fmt.Printf(...)` | Prints formatted text using format verbs (`%s`, `%d`, etc.) |
| `fmt.Sprintf(...)` | Same as Printf, but RETURNS the string instead of printing it |
| `fmt.Scan(&var)` | Reads user input from the keyboard into a variable |

```go
var name string
fmt.Print("Enter your name: ")
fmt.Scan(&name)
fmt.Println("Hello,", name)
```
`fmt.Scan(&name)` — notice the `&` (from Day 6!) — `Scan` needs a POINTER to know WHERE to store the value it reads in, since it needs to modify the actual variable, not a copy.

### `strings` — Working with Text

```go
package main

import (
    "fmt"
    "strings"
)

func main() {
    s := "Hello, Go Programming!"

    fmt.Println(strings.ToUpper(s))            // HELLO, GO PROGRAMMING!
    fmt.Println(strings.ToLower(s))             // hello, go programming!
    fmt.Println(strings.Contains(s, "Go"))       // true
    fmt.Println(strings.Replace(s, "Go", "Java", 1)) // Hello, Java Programming!
    fmt.Println(strings.Split(s, " "))           // [Hello, Go Programming!]
    fmt.Println(strings.TrimSpace("   hi   "))   // "hi" (removes leading/trailing spaces)
    fmt.Println(strings.HasPrefix(s, "Hello"))   // true
}
```
The `strings` package is used CONSTANTLY in real Go code — searching, replacing, splitting, joining, and cleaning up text.

### `strconv` — Converting Between Strings and Numbers

Remember on Day 1, we mentioned you can't directly combine a `string` and an `int`? `strconv` (short for "string conversion") is the package for exactly this kind of conversion.

```go
package main

import (
    "fmt"
    "strconv"
)

func main() {
    // String to int
    num, err := strconv.Atoi("42") // "Atoi" = ASCII to Integer
    if err != nil {
        fmt.Println("Conversion error:", err)
    }
    fmt.Println(num + 8) // 50

    // Int to string
    text := strconv.Itoa(100) // "Itoa" = Integer to ASCII
    fmt.Println("Number is: " + text)

    // String to float
    f, _ := strconv.ParseFloat("3.14", 64)
    fmt.Println(f)
}
```
**Important:** `strconv.Atoi` returns TWO values (`int, error`) — exactly the pattern from Day 8! If the string isn't a valid number (like trying to convert `"hello"` to an int), it returns a non-nil `error` instead of crashing your program.

### `time` — Working with Dates and Durations

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    now := time.Now()
    fmt.Println("Current time:", now)

    later := now.Add(2 * time.Hour) // adding 2 hours to the current time
    fmt.Println("2 hours later:", later)

    fmt.Println("Formatted:", now.Format("2006-01-02 15:04:05"))
}
```

**A genuinely strange but important Go quirk:** Go's date formatting doesn't use symbols like `YYYY-MM-DD` (common in many other languages). Instead, it uses a SPECIFIC REFERENCE DATE: `Mon Jan 2 15:04:05 MST 2006` (memorable because if you read the numbers as 01/02 03:04:05PM '06, they go in ascending order: 1, 2, 3, 4, 5, 6). You format dates by writing this exact reference date in the pattern you want, and Go substitutes in the actual values.
```go
now.Format("2006-01-02") // produces something like "2026-07-01"
now.Format("02-Jan-2006") // produces something like "01-Jul-2026"
```
This trips up EVERY Go beginner at least once, and is a genuinely well-known, commonly-discussed quirk — good to know both for interviews and for avoiding real confusion later.

We also saw `time.Sleep()` back on Day 9, used to pause execution for a specified duration.

### `os` — Interacting with the Operating System

```go
package main

import (
    "fmt"
    "os"
)

func main() {
    args := os.Args // command-line arguments passed to the program
    fmt.Println(args)

    // Reading an environment variable
    home := os.Getenv("HOME")
    fmt.Println("Home directory:", home)

    // Exiting the program with a specific status code
    if len(args) < 2 {
        fmt.Println("Not enough arguments")
        os.Exit(1) // non-zero means "something went wrong"
    }
}
```
`os` is used for things like reading command-line arguments, reading/writing files (more on this on Day 13), reading environment variables, and controlling program exit behavior.

---

## 7. `init()` Function — A Special Package-Level Function

Go has a special function you can define in ANY package (yes, even `main`), called `init()`. It runs AUTOMATICALLY, before `main()`, whenever the package is loaded — you never call it yourself.

```go
package main

import "fmt"

func init() {
    fmt.Println("This runs first, automatically!")
}

func main() {
    fmt.Println("This runs second.")
}
```
Output:
```
This runs first, automatically!
This runs second.
```

**Use case:** `init()` is typically used for one-time setup tasks needed before the rest of the program runs — like initializing configuration values, validating environment setup, or registering something with another package. It's used sparingly in idiomatic Go, since overusing it can make program startup behavior harder to trace (since it runs "invisibly," without an explicit call anywhere in your visible code flow).

---

## 8. Common Beginner Mistakes

1. **Forgetting the capital letter rule** — trying to access a lowercase (unexported) name from another package, which Go simply refuses to compile.
2. **Mismatching folder name and package name** — while technically allowed to differ, it's strongly conventional (and much less confusing) to keep them matching.
3. **Manually editing `go.sum`** — this file is machine-generated; let Go manage it automatically via `go get`/`go mod tidy`.
4. **Ignoring the returned `error` from `strconv` conversions** — always check it, since invalid input (like a non-numeric string) will produce a genuine error, not a crash, but only if you actually check for it.
5. **Being confused by Go's unusual time formatting reference date** — remember it's always based on `Mon Jan 2 15:04:05 MST 2006`, not symbolic placeholders like `YYYY-MM-DD`.

---

## 9. Day 11 Practice Exercise

Write a program that:
1. Creates a package `stringutils` with an exported function `Reverse(s string) string` that reverses a given string, and an unexported helper function it uses internally.
2. In `main.go`, imports `stringutils` and uses `Reverse` on a sample string.
3. Uses `strconv` to convert a user-provided (or hardcoded) string number into an int, safely handling any conversion error.
4. Uses `time.Now().Format(...)` to print the current date in `DD-Mon-YYYY` format.

<details>
<summary>Click to see solution</summary>

**stringutils/reverse.go:**
```go
package stringutils

func Reverse(s string) string {
    runes := toRuneSlice(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}

func toRuneSlice(s string) []rune {
    return []rune(s)
}
```

**main.go:**
```go
package main

import (
    "fmt"
    "strconv"
    "time"

    "myproject/stringutils"
)

func main() {
    reversed := stringutils.Reverse("Golang")
    fmt.Println("Reversed:", reversed)

    numStr := "123"
    num, err := strconv.Atoi(numStr)
    if err != nil {
        fmt.Println("Invalid number:", err)
    } else {
        fmt.Println("Converted number:", num)
    }

    fmt.Println("Today's date:", time.Now().Format("02-Jan-2006"))
}
```
*(Note: `[]rune(s)` converts a string into a slice of "runes" — Go's way of representing individual characters, which handles non-English characters correctly, unlike simply indexing bytes. This is a small preview of a more advanced string-handling detail.)*
</details>

---

## 10. Day 11 Interview Questions (Quick Prep)

1. **Q: How does Go determine if a function/variable is accessible from another package?**
   A: By its name's first letter — uppercase means exported (accessible externally), lowercase means unexported (package-private).

2. **Q: What's the difference between `go.mod` and `go.sum`?**
   A: `go.mod` declares the module name, Go version, and dependency list with versions. `go.sum` contains checksums verifying the exact, unmodified contents of each dependency, for security and reproducibility.

3. **Q: What is the `init()` function used for?**
   A: One-time setup logic that runs automatically before `main()`, without being explicitly called — commonly used for configuration or initialization tasks.

4. **Q: How do you convert a string to an int in Go, and what should you watch out for?**
   A: Using `strconv.Atoi(s)`, which returns `(int, error)` — you should always check the returned error, since invalid input won't crash the program but will return a non-nil error.

5. **Q: Why does Go use a specific reference date (`Mon Jan 2 15:04:05 MST 2006`) for time formatting instead of symbols like `YYYY-MM-DD`?**
   A: It's a distinctive Go design choice — you format a date by writing this exact reference date in your desired output pattern, and Go substitutes the real values accordingly.

---

## Summary of Day 11

- A package groups related Go code; `package main` is special and required for runnable programs (must include `func main()`).
- Exported (capitalized) names are accessible from other packages; unexported (lowercase) names are package-private — Go's core mechanism for encapsulation, enforced by the compiler.
- `go.mod` declares your module and its dependencies; `go.sum` verifies dependency integrity via checksums.
- The standard library provides essential, ready-to-use packages: `fmt` (formatting/printing), `strings` (text manipulation), `strconv` (string/number conversion), `time` (dates and durations, with Go's unique reference-date formatting), and `os` (OS interaction, args, environment variables).
- `init()` runs automatically before `main()`, useful for one-time setup, but should be used sparingly.

**Tomorrow (Day 12):** We'll cover Testing in Go — writing tests with the `testing` package, the idiomatic table-driven test pattern, benchmarks, and the basics of mocking for testing code that depends on interfaces.
