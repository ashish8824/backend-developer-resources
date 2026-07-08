# Day 7: Interfaces

> **Goal for today:** Understand one of Go's most powerful and distinctive features — Interfaces. You'll learn what interfaces are, Go's unique "implicit implementation" (no `implements` keyword needed), the empty interface (`any`), type assertions, and type switches. This is a heavily tested interview topic.

---

## 1. What is an Interface? (Concept First)

So far, we've worked with CONCRETE types — `int`, `string`, `Person` struct, etc. — types that describe exactly WHAT data looks like.

An **interface** is different: it doesn't describe data at all. It describes **BEHAVIOR** — a set of methods that a type must have. It's a "contract" that says: "any type that has these specific methods can be used here, I don't care what the type actually is underneath."

**Analogy:** Think of an interface like a **job description**, not a specific person. A job posting for "Driver" might say: "must be able to `Drive()` and `ParkVehicle()`." It doesn't care if the applicant is a taxi driver, a truck driver, or a delivery person — ANYONE who can perform those two actions qualifies for the job. The interface is the job description (the required abilities); the actual person filling that role could be any type at all, as long as they meet the requirements.

```
  Interface "Driver" (the job description / contract):
  ┌─────────────────────────────┐
  │  Must be able to:             │
  │    - Drive()                    │
  │    - ParkVehicle()              │
  └─────────────────────────────┘

  Anyone who can do BOTH of these things automatically qualifies as a "Driver"
  — whether they're a TaxiDriver, TruckDriver, or DeliveryPerson struct.
```

---

## 2. Defining and Using an Interface

### Step 1: Define the interface

```go
package main

import "fmt"

type Shape interface {
    Area() float64
}
```

**Breakdown:**
- `type Shape interface { ... }` — defines a new interface type named `Shape`.
- `Area() float64` — this says: "any type that wants to count as a `Shape` MUST have a method called `Area()` that returns a `float64`." Notice there's no function body here — an interface only declares WHAT methods must exist, not HOW they work. The actual implementation is left to whichever type uses it.

### Step 2: Create types that satisfy the interface

```go
type Circle struct {
    Radius float64
}

func (c Circle) Area() float64 {
    return 3.14159 * c.Radius * c.Radius
}

type Rectangle struct {
    Width  float64
    Height float64
}

func (r Rectangle) Area() float64 {
    return r.Width * r.Height
}
```

Both `Circle` and `Rectangle` have an `Area()` method that returns a `float64`. That means **both of them automatically satisfy the `Shape` interface** — even though we never wrote anything like `Circle implements Shape` anywhere!

### THE Big Difference from Java/Other Languages: Implicit Implementation

This is genuinely one of the most important and most frequently asked Go interview points.

In Java, if you want a class to implement an interface, you must EXPLICITLY declare it:
```java
// Java (NOT Go):
class Circle implements Shape {
    ...
}
```

**In Go, there is NO `implements` keyword at all.** A type automatically satisfies an interface simply by having all the required methods, with matching signatures. This is called **implicit (or "structural") implementation** — Go checks the SHAPE (the structure of methods) of a type, not any explicit declaration.

**Analogy:** In Java, it's like you need to sign an official paper saying "I am qualified for this job." In Go, nobody checks a paper — if you can genuinely DO the job (you have the right methods), you're automatically considered qualified, no paperwork needed.

**Why does Go do this?** It makes code more flexible and decoupled. You can define an interface AFTER a type already exists, or in a completely different package, and existing types will automatically satisfy it if they happen to have the right methods — no need to go back and modify the original type's code.

### Step 3: Using the interface

```go
package main

import "fmt"

func printArea(s Shape) {
    fmt.Println("Area:", s.Area())
}

func main() {
    c := Circle{Radius: 5}
    r := Rectangle{Width: 4, Height: 6}

    printArea(c) // Area: 78.53975
    printArea(r) // Area: 24
}
```

**Breakdown:**
- `func printArea(s Shape)` — this function accepts ANY type that satisfies the `Shape` interface (i.e., has an `Area() float64` method). It doesn't know or care whether it's a `Circle` or a `Rectangle` — it only cares that whatever is passed in CAN calculate its own area.
- We can pass `c` (a `Circle`) or `r` (a `Rectangle`) into the SAME function, and it works for both. This is a form of **polymorphism** — "many shapes, one function that handles them all uniformly."

### Visual: How Interfaces Work

```
   Interface "Shape" says: "must have Area() float64"

         ┌───────────┐        ┌───────────┐
         │  Circle    │        │ Rectangle  │
         │  Area()✓   │        │  Area()✓   │
         └───────────┘        └───────────┘
                │                    │
                └─────────┬──────────┘
                          v
              Both automatically satisfy
              the "Shape" interface
                          │
                          v
            func printArea(s Shape) accepts BOTH
```

---

## 3. Why Interfaces Matter — Go's Design Philosophy

Interfaces are central to how Go code is designed. A famous Go proverb (a well-known guiding principle among Go developers) says:

*"Accept interfaces, return structs."*

This means: when WRITING a function, prefer accepting an interface as a parameter (so it works with ANY type that satisfies the needed behavior), but RETURN a concrete, specific type (a struct) from your functions, so callers know exactly what they're getting back.

This makes code:
- **Flexible** — your function works with types you haven't even thought of yet, as long as they satisfy the interface.
- **Testable** — you can substitute a "fake"/"mock" version of a type (a common technique in testing, which we'll explore on Day 12) as long as it satisfies the same interface.
- **Decoupled** — code doesn't need to know the exact concrete type it's working with, only the behavior it needs.

---

## 4. Interfaces with Multiple Methods

An interface can require more than one method:

```go
type Vehicle interface {
    Start() string
    Stop() string
}

type Car struct {
    Model string
}

func (c Car) Start() string {
    return c.Model + " is starting"
}

func (c Car) Stop() string {
    return c.Model + " is stopping"
}
```

`Car` satisfies `Vehicle` only because it implements BOTH `Start()` AND `Stop()`. If it were missing even one of these methods, it would NOT satisfy the interface, and passing a `Car` where a `Vehicle` is expected would cause a compile error.

---

## 5. The Empty Interface: `interface{}` and `any`

An **empty interface** has NO methods required at all:
```go
var x interface{}
```
Since it requires zero methods, EVERY single type in Go automatically satisfies it — meaning a variable of type `interface{}` can hold a value of ANY type whatsoever.

```go
package main

import "fmt"

func printAnything(val interface{}) {
    fmt.Println(val)
}

func main() {
    printAnything(42)
    printAnything("hello")
    printAnything(3.14)
    printAnything(true)
    printAnything(Circle{Radius: 5})
}
```
All of these work, because `interface{}` places no restrictions on what type is passed.

### The Modern Shortcut: `any`

Since Go 1.18, there's a built-in alias `any` that means exactly the same thing as `interface{}`, just shorter and more readable:

```go
func printAnything(val any) { // identical meaning to interface{}
    fmt.Println(val)
}
```
**In modern Go code, you'll almost always see `any` used instead of `interface{}`** — they're 100% interchangeable, but `any` is now the preferred, idiomatic style.

**Caution / interview point:** While `interface{}`/`any` is flexible, overusing it defeats the purpose of Go's strong typing — you lose compile-time safety checks, since Go can no longer verify what methods/operations are valid on a value of unknown type. Use it sparingly, mainly when you genuinely need to handle arbitrary/unknown types (like in generic-style utility functions before Go had true Generics — which we'll cover on Day 14).

---

## 6. Type Assertion

If you have a value stored as an interface (like `any`), and you need to get back its SPECIFIC underlying type to actually use it meaningfully, you use a **type assertion**.

```go
package main

import "fmt"

func main() {
    var val interface{} = "Hello, Go!"

    str := val.(string) // type assertion: "I assert that val actually holds a string"
    fmt.Println(str)
}
```

**Breakdown:**
- `val.(string)` — this says "I'm confident `val` actually contains a `string` underneath; give it back to me as an actual, usable `string`."
- If `val` genuinely does hold a `string`, this works fine, and `str` becomes a normal `string` you can use.

### The DANGER of a plain type assertion

If your assertion is WRONG (the value doesn't actually hold the type you claimed), your program will **panic** (crash) at runtime:
```go
var val interface{} = 42
str := val.(string) // PANIC: interface conversion: interface {} is int, not string
```

### The SAFE way — "comma ok" idiom (you've seen this pattern before, on Day 4!)

```go
val := interface{}(42)

str, ok := val.(string)
if ok {
    fmt.Println("It's a string:", str)
} else {
    fmt.Println("val is NOT a string")
}
```
- `ok` will be `true` if the assertion succeeds, `false` if it fails — and crucially, **your program will NOT panic** using this form, even if the assertion is wrong. `str` will just hold the zero value (`""` for string) if `ok` is `false`.

**Always prefer the "comma ok" form when you're not 100% certain of the underlying type**, to avoid crashing your program.

---

## 7. Type Switch (Checking Multiple Possible Types)

If you need to check a value against SEVERAL possible types, writing repeated type assertions would be clunky. Go provides a cleaner tool: the **type switch** — combining the `switch` statement you learned on Day 2 with type checking.

```go
package main

import "fmt"

func describe(val interface{}) {
    switch v := val.(type) {
    case int:
        fmt.Println("It's an int:", v)
    case string:
        fmt.Println("It's a string:", v)
    case bool:
        fmt.Println("It's a bool:", v)
    default:
        fmt.Println("Unknown type")
    }
}

func main() {
    describe(42)
    describe("hello")
    describe(true)
    describe(3.14)
}
```
Output:
```
It's an int: 42
It's a string: hello
It's a bool: true
Unknown type
```

**Breakdown:**
- `switch v := val.(type) {` — this special syntax (`.(type)`, using the literal word `type`) checks the ACTUAL underlying type of `val` against each `case`.
- In each `case`, `v` automatically becomes that SPECIFIC type. For example, inside `case int:`, `v` is treated as a genuine `int`, so you can use it just like any normal int variable.
- `default:` catches anything that doesn't match any listed case — here, `3.14` is a `float64`, which we didn't check for, so it falls into `default`.

**This exact pattern is extremely common in real Go code** whenever you're working with data of unknown/mixed types (e.g., parsing JSON with unpredictable structure, which we'll touch on Day 13).

---

## 8. Interfaces as Function Parameters — A Full Practical Example

```go
package main

import "fmt"

type Notifier interface {
    Notify(message string) string
}

type EmailNotifier struct {
    Email string
}

func (e EmailNotifier) Notify(message string) string {
    return "Emailing " + e.Email + ": " + message
}

type SMSNotifier struct {
    Phone string
}

func (s SMSNotifier) Notify(message string) string {
    return "Texting " + s.Phone + ": " + message
}

func sendAlert(n Notifier, message string) {
    result := n.Notify(message)
    fmt.Println(result)
}

func main() {
    email := EmailNotifier{Email: "user@mail.com"}
    sms := SMSNotifier{Phone: "9876543210"}

    sendAlert(email, "Server is down!")
    sendAlert(sms, "Server is down!")
}
```
Output:
```
Emailing user@mail.com: Server is down!
Texting 9876543210: Server is down!
```

This demonstrates the real power of interfaces: `sendAlert` doesn't need to know or care HOW notification actually happens (email vs SMS) — it just calls `Notify()`, and whichever concrete type was passed in handles the details its own way. If tomorrow you add a `PushNotifier` or `SlackNotifier`, `sendAlert` doesn't need to change AT ALL, as long as the new type also has a `Notify(message string) string` method.

---

## 9. Common Beginner Mistakes

1. **Trying to explicitly declare "implements"** — Go has no such keyword; implementation is automatic/implicit, based purely on having the right methods.
2. **Forgetting a method signature must match EXACTLY** (same parameter types AND same return types) for a type to satisfy an interface — even a small mismatch (like returning `int` instead of `float64`) means the type does NOT satisfy the interface.
3. **Using plain type assertions without "comma ok"** in situations where the type isn't guaranteed — this risks a runtime panic. Prefer `value, ok := x.(SomeType)`.
4. **Overusing `any`/`interface{}`** everywhere, losing the benefits of Go's compile-time type checking. Use interfaces with clearly defined methods when you know what behavior you need.
5. **Confusing an interface with a struct** — an interface defines behavior (a contract, no implementation); a struct defines data (with optional methods providing actual implementation).

---

## 10. Day 7 Practice Exercise

Write a program that:
1. Defines an interface `Animal` with a method `Sound() string`.
2. Defines two structs, `Dog` and `Cat`, both implementing `Sound()` (e.g., Dog returns "Woof", Cat returns "Meow").
3. Writes a function `makeSound(a Animal)` that prints the sound.
4. Calls `makeSound` with both a `Dog` and a `Cat`.
5. BONUS: Write a function `identify(val interface{})` that uses a type switch to print whether the given value is an `int`, `string`, or something else.

<details>
<summary>Click to see solution</summary>

```go
package main

import "fmt"

type Animal interface {
    Sound() string
}

type Dog struct{}

func (d Dog) Sound() string {
    return "Woof"
}

type Cat struct{}

func (c Cat) Sound() string {
    return "Meow"
}

func makeSound(a Animal) {
    fmt.Println(a.Sound())
}

func identify(val interface{}) {
    switch v := val.(type) {
    case int:
        fmt.Println("This is an int:", v)
    case string:
        fmt.Println("This is a string:", v)
    default:
        fmt.Println("Unknown type")
    }
}

func main() {
    d := Dog{}
    c := Cat{}

    makeSound(d) // Woof
    makeSound(c) // Meow

    identify(10)
    identify("Go")
    identify(3.5)
}
```
</details>

---

## 11. Day 7 Interview Questions (Quick Prep)

1. **Q: What is an interface in Go?**
   A: A type that defines a set of method signatures (a contract for behavior), without specifying how those methods are implemented.

2. **Q: How does a type "implement" an interface in Go?**
   A: Implicitly — a type automatically satisfies an interface simply by having ALL the required methods with matching signatures. There's no `implements` keyword.

3. **Q: What is the empty interface, and what does `any` mean?**
   A: `interface{}` (or its modern alias `any`, since Go 1.18) has no required methods, so every type satisfies it — it can hold a value of any type.

4. **Q: What's the difference between a plain type assertion and the "comma ok" form?**
   A: A plain assertion (`x.(Type)`) panics at runtime if the assertion is wrong. The "comma ok" form (`v, ok := x.(Type)`) returns `ok = false` instead of panicking, making it the safer choice when unsure of the type.

5. **Q: What is a type switch, and when would you use it?**
   A: A construct (`switch v := x.(type) { case Type1: ... }`) used to check a value's underlying type against multiple possibilities — useful when handling values of unknown or mixed types.

6. **Q: What's the Go proverb related to interfaces, and what does it mean?**
   A: "Accept interfaces, return structs" — functions should accept flexible interface parameters (to work with many types), but return concrete, specific types so callers know exactly what they're getting.

---

## Summary of Day 7

- An interface defines a contract of required method signatures — pure behavior, no implementation.
- Go uses IMPLICIT implementation — any type with the right methods automatically satisfies an interface, no `implements` keyword needed.
- The empty interface (`interface{}`, or `any` since Go 1.18) can hold a value of any type, since it requires zero methods.
- Type assertion (`x.(Type)`) extracts a concrete type from an interface value; prefer the safe "comma ok" form (`v, ok := x.(Type)`) to avoid runtime panics.
- A type switch (`switch v := x.(type) { case ... }`) lets you branch logic based on a value's actual underlying type.
- Interfaces enable polymorphism and flexible, decoupled, testable code — a core part of idiomatic Go design.

**Tomorrow (Day 8):** We'll cover Error Handling — Go's `error` type and why Go deliberately avoids try/catch, creating custom errors, `errors.Is`/`errors.As`, and a deep dive into `panic`, `recover`, and `defer`.
