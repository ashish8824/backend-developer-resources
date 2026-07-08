# Day 3: Functions in Go

> **Goal for today:** Understand what functions are, how to define and call them, Go's special ability to return multiple values, named return values, variadic functions, and closures. Functions are the building blocks of every Go program, so we go deep today.

---

## 1. What is a Function? (Concept First)

A **function** is a named, reusable block of code that performs a specific task. Instead of writing the same code again and again, you write it once inside a function, and then just "call" (use) that function whenever you need it.

**Analogy:** Think of a function like a **coffee machine**. You don't rebuild the machine every time you want coffee — you just press a button (call the function), maybe select some options (give it inputs), and it gives you coffee (a result/output).

```
   INPUT (ingredients)  →   [ FUNCTION: makeCoffee ]   →   OUTPUT (a cup of coffee)
   e.g., "sugar level"                                     e.g., the finished coffee
```

You've actually already used a function since Day 1: `fmt.Println()` is a function — someone at Google already wrote the code to print text, and you just "call" it whenever you need that behavior, without knowing (or caring) how it works internally.

---

## 2. Defining and Calling a Function

### Basic Syntax

```go
package main

import "fmt"

func greet() {
    fmt.Println("Hello there!")
}

func main() {
    greet() // calling the function
}
```

**Breakdown:**
- `func` — keyword that says "I'm defining a function."
- `greet` — the name we're giving this function (you can name it almost anything, following Go naming rules).
- `()` — empty parentheses mean this function takes no inputs.
- `{ }` — the function body — the code that runs when this function is called.
- `greet()` inside `main()` — this is how you **call** (execute) the function. Without calling it, the code inside `greet` never runs, no matter how it's defined.

**Important:** Just like `main()`, every function needs its own `{ }` braces marking where it starts and ends.

---

## 3. Functions with Parameters (Inputs)

A **parameter** is a placeholder for a value the function expects to receive when called. This lets your function behave differently based on the input given.

```go
package main

import "fmt"

func greet(name string) {
    fmt.Println("Hello,", name)
}

func main() {
    greet("Rahul")
    greet("Priya")
}
```
Output:
```
Hello, Rahul
Hello, Priya
```

**Breakdown:**
- `name string` inside the parentheses — this declares a parameter named `name`, of type `string`. This means: "when someone calls this function, they MUST provide a text value, and inside the function, we'll refer to that value using the name `name`."
- `greet("Rahul")` — here, `"Rahul"` is called an **argument** (the actual value passed in). Note the small distinction: **parameter** = the placeholder in the function definition; **argument** = the actual value you pass when calling it.

### Multiple Parameters

```go
func addNumbers(a int, b int) {
    sum := a + b
    fmt.Println("Sum is:", sum)
}

func main() {
    addNumbers(5, 10) // Output: Sum is: 15
}
```

**Shortcut:** If consecutive parameters share the same type, you can write the type only once at the end:
```go
func addNumbers(a, b int) {
    // same as (a int, b int)
}
```

---

## 4. Return Values (Output)

A function can **return** (send back) a value to whoever called it, using the `return` keyword.

```go
package main

import "fmt"

func add(a int, b int) int {
    return a + b
}

func main() {
    result := add(5, 10)
    fmt.Println("Result:", result) // Output: Result: 15
}
```

**Breakdown:**
- `func add(a int, b int) int {` — the `int` written AFTER the parentheses (before the opening brace) declares the **return type** — it tells Go "this function will send back a value of type `int`."
- `return a + b` — this stops the function immediately and sends the calculated value (`a + b`) back to wherever the function was called.
- `result := add(5, 10)` — here, we're capturing the returned value into a new variable called `result`.

**Important rule:** If a function declares a return type (like `int`), it MUST return a value of that exact type somewhere in every possible path through the code. Go's compiler checks this strictly and won't let you forget it.

---

## 5. Multiple Return Values (Go's Signature Feature — Very Important for Interviews!)

Here's something that makes Go stand out from many languages (including Java, which typically only lets a method return ONE value): **Go functions can return multiple values at once.**

```go
package main

import "fmt"

func divide(a int, b int) (int, int) {
    quotient := a / b
    remainder := a % b
    return quotient, remainder
}

func main() {
    q, r := divide(17, 5)
    fmt.Println("Quotient:", q)
    fmt.Println("Remainder:", r)
}
```
Output:
```
Quotient: 3
Remainder: 2
```

**Breakdown:**
- `(int, int)` after the parameters — declares that this function returns TWO values, both of type `int`.
- `return quotient, remainder` — returns both values at once, separated by a comma.
- `q, r := divide(17, 5)` — captures both returned values into two separate variables in one line.

### Why does this matter so much in real Go code?

This feature is used EVERYWHERE in Go, especially for **error handling** (which we'll cover in-depth on Day 8). The standard Go pattern is: a function returns its actual result AND an error value together, like this:

```go
func divide(a, b int) (int, error) {
    if b == 0 {
        return 0, fmt.Errorf("cannot divide by zero")
    }
    return a / b, nil
}

func main() {
    result, err := divide(10, 0)
    if err != nil {
        fmt.Println("Error occurred:", err)
        return
    }
    fmt.Println("Result:", result)
}
```
Don't worry about fully understanding `error` and `nil` yet — that's Day 8's focus in detail. Just recognize this pattern now, because you WILL see `(value, err)` returned from functions constantly in real Go code, and it's built entirely on this "multiple return values" feature.

**Interview tip:** This is one of the most commonly asked "what makes Go different" questions. Confidently mention: *"Go allows functions to return multiple values, which is heavily used for the idiomatic error-handling pattern of returning a result alongside an error."*

### Ignoring a Return Value with `_`
Sometimes you don't care about one of the returned values. Use the blank identifier `_` (we saw this in Day 2 with `for-range`):
```go
q, _ := divide(17, 5) // ignoring the remainder, only keeping the quotient
```

---

## 6. Named Return Values

Go allows you to give names to your return values right in the function signature. This can make the code more self-documenting (readable) and lets you use a "naked" `return` (return with no values explicitly listed).

```go
func divide(a, b int) (quotient int, remainder int) {
    quotient = a / b
    remainder = a % b
    return // "naked return" — automatically returns quotient and remainder
}
```

**Breakdown:**
- `(quotient int, remainder int)` — this declares TWO variables (`quotient` and `remainder`) right in the function signature, both automatically initialized to `0` (their default value for `int`).
- Inside the function, we just assign values to them normally (`quotient = a / b`), no need to declare them again with `:=`.
- `return` alone (no values listed) — Go automatically knows to return whatever `quotient` and `remainder` currently hold.

**When to use named returns?** They're great for short, simple functions where the return values benefit from having clear names (improves readability, acts like built-in documentation). For longer/complex functions, most Go developers avoid naked returns because it can make code harder to follow — you'd have to scroll up to remember what's being returned.

---

## 7. Variadic Functions (Functions that accept ANY number of arguments)

Sometimes you don't know in advance how many arguments will be passed. A **variadic function** allows a variable (changeable) number of arguments of the same type.

```go
package main

import "fmt"

func sum(numbers ...int) int {
    total := 0
    for _, num := range numbers {
        total += num
    }
    return total
}

func main() {
    fmt.Println(sum(1, 2, 3))          // Output: 6
    fmt.Println(sum(10, 20, 30, 40))   // Output: 100
    fmt.Println(sum())                 // Output: 0 (no arguments at all is also valid)
}
```

**Breakdown:**
- `numbers ...int` — the `...` before the type `int` means "accept zero or more `int` values here." Inside the function, `numbers` behaves like a **slice** (a list — we'll cover slices deeply on Day 4) of integers.
- `for _, num := range numbers` — loops through every number passed in, adding each to `total`.
- `total += num` — shorthand for `total = total + num`.

You've actually already used a variadic function without realizing it: `fmt.Println()` itself is variadic — that's why you can pass it one value, or five values, and it just works: `fmt.Println("a", "b", "c")`.

**Important rule:** A variadic parameter must be the LAST parameter in the function, and there can only be one:
```go
func example(name string, scores ...int) {
    // valid: fixed parameter first, variadic parameter last
}
```

### Passing a slice into a variadic function
If you already have a slice of numbers and want to pass them all in, use `...` after the slice name to "spread" it out:
```go
nums := []int{1, 2, 3, 4}
fmt.Println(sum(nums...))
```

---

## 8. Functions as Values & Anonymous Functions

In Go, functions are treated as "first-class citizens" — meaning you can store a function inside a variable, pass a function as an argument to another function, or even return a function from another function. This might feel unusual at first, so let's build it up slowly.

### Storing a function in a variable

```go
package main

import "fmt"

func main() {
    var greet = func(name string) {
        fmt.Println("Hi,", name)
    }

    greet("Amit") // Output: Hi, Amit
}
```

Here, `func(name string) { fmt.Println("Hi,", name) }` is called an **anonymous function** — a function without a name, defined right where it's used, and stored inside the variable `greet`. You then call it just like any named function: `greet("Amit")`.

**Analogy:** Normally, you name a function like giving a name tag to a worker so you can call them by name later ("greet, come do your job"). An anonymous function is like hiring someone for a quick one-time task without bothering to give them a name tag at all — you just hand them the task directly.

### Immediately Invoked Function (calling an anonymous function right away)

```go
func main() {
    result := func(a, b int) int {
        return a + b
    }(5, 10) // <- notice the () right after the function body, calling it immediately

    fmt.Println(result) // Output: 15
}
```

### Passing a function as an argument to another function

```go
package main

import "fmt"

func operate(a int, b int, operation func(int, int) int) int {
    return operation(a, b)
}

func add(a, b int) int {
    return a + b
}

func multiply(a, b int) int {
    return a * b
}

func main() {
    fmt.Println(operate(5, 3, add))      // Output: 8
    fmt.Println(operate(5, 3, multiply)) // Output: 15
}
```

**Breakdown:**
- `operation func(int, int) int` — this parameter itself is a function! It says: "give me a function that takes two `int`s and returns an `int`."
- Inside `operate`, we call whatever function was passed in: `operation(a, b)`.
- In `main()`, we pass in `add` or `multiply` (without calling them — no `()` after the name) as the actual "operation" to use.

This pattern is very powerful — it lets you write flexible, reusable code where the exact behavior can be customized by whoever calls the function.

---

## 9. Closures (Functions that "Remember" Their Surrounding Variables)

A **closure** is an anonymous function that "captures" (remembers) variables from the place it was created, even after that outer function has finished running. This is a genuinely tricky concept for beginners, so let's build intuition slowly.

```go
package main

import "fmt"

func counter() func() int {
    count := 0

    return func() int {
        count++
        return count
    }
}

func main() {
    increment := counter()

    fmt.Println(increment()) // Output: 1
    fmt.Println(increment()) // Output: 2
    fmt.Println(increment()) // Output: 3
}
```

**Let's break this down VERY carefully, step by step:**

1. `func counter() func() int {` — `counter` is a function that RETURNS another function (that returned function itself takes no arguments and returns an `int`).
2. Inside `counter`, we declare `count := 0`. This is a normal local variable.
3. We then `return func() int { count++; return count }` — this returns a brand new anonymous function. Crucially, this inner function **uses** the `count` variable from its surrounding environment.
4. In `main()`, `increment := counter()` calls `counter()` ONCE. This runs the code inside `counter`, sets up `count = 0`, and returns the inner anonymous function, which gets stored in `increment`.
5. Now here's the magic: even though `counter()` has already finished running, the returned function `increment` still "remembers" and has continued access to that specific `count` variable. It doesn't reset back to 0 every time.
6. Every time you call `increment()`, it increases the SAME `count` variable by 1 and returns it. That's why we get 1, then 2, then 3.

**Analogy:** Think of `counter()` as a factory that builds a personal ticket-counting machine and hands it to you. That machine has its own private ticket counter sealed inside it that only IT can access and update. Every time you press the button (call `increment()`), its internal number goes up by one — and it remembers this number between button presses, because the counter is "enclosed" (hence the name "closure") inside that specific machine.

### Why does this matter?

Closures are used heavily in real Go code for things like:
- Creating function generators (like our counter example)
- Maintaining state between function calls without using global variables
- Callback functions (functions passed to be called later, e.g., in web servers, event handlers)

---

## 10. Common Beginner Mistakes

1. **Forgetting the return type in the function signature** when the function actually returns something. Go will show a compile error if you try to `return` a value from a function that has no declared return type.
2. **Mismatching multiple return value counts** — if a function returns 2 values, you must capture BOTH (or explicitly ignore one with `_`), you can't just capture one directly.
   ```go
   result := divide(10, 2) // ERROR: divide returns TWO values, but only one variable given
   ```
3. **Confusing parameters and arguments** in conversation (not a code error, but a common terminology mix-up in interviews).
4. **Forgetting `...` is only valid for the LAST parameter** in a variadic function.
5. **Not understanding that closures keep the variable alive** — beginners sometimes expect the variable to "reset" each time, but it doesn't; it's shared and persists.

---

## 11. Day 3 Practice Exercise

Write a program that:
1. Defines a function `calculateArea` that takes `length` and `width` (both `float64`) and returns the area.
2. Defines a variadic function `average` that accepts any number of `float64` values and returns their average.
3. Write a closure-based function `makeMultiplier(factor int) func(int) int` that returns a function which multiplies any given number by `factor`.

<details>
<summary>Click to see solution</summary>

```go
package main

import "fmt"

func calculateArea(length, width float64) float64 {
    return length * width
}

func average(numbers ...float64) float64 {
    total := 0.0
    for _, n := range numbers {
        total += n
    }
    return total / float64(len(numbers))
}

func makeMultiplier(factor int) func(int) int {
    return func(n int) int {
        return n * factor
    }
}

func main() {
    fmt.Println("Area:", calculateArea(5.5, 2.0))
    fmt.Println("Average:", average(10, 20, 30))

    double := makeMultiplier(2)
    triple := makeMultiplier(3)

    fmt.Println("Double of 5:", double(5)) // 10
    fmt.Println("Triple of 5:", triple(5)) // 15
}
```
**Note:** `len(numbers)` returns the number of items in the `numbers` list, and we convert it with `float64(len(numbers))` because `len()` returns an `int`, but we need a `float64` for accurate decimal division.
</details>

---

## 12. Day 3 Interview Questions (Quick Prep)

1. **Q: Can a Go function return more than one value? Give a real-world use case.**
   A: Yes — this is a defining feature of Go. It's most commonly used for the `(result, error)` return pattern used throughout Go's error handling.

2. **Q: What is a variadic function?**
   A: A function that accepts a variable number of arguments of the same type, declared using `...type` for the last parameter, e.g. `func sum(nums ...int) int`.

3. **Q: What is a closure? Give an example use case.**
   A: A closure is a function that captures and remembers variables from its surrounding scope, even after the outer function has returned. Commonly used for generating stateful functions like counters, or for callbacks.

4. **Q: What's the difference between a parameter and an argument?**
   A: A parameter is the placeholder variable defined in the function signature; an argument is the actual value passed in when the function is called.

5. **Q: What is a named return value in Go?**
   A: A return value that's given a name directly in the function signature, allowing a "naked return" statement without explicitly listing the values.

6. **Q: Are functions "first-class citizens" in Go? What does that mean?**
   A: Yes — functions can be assigned to variables, passed as arguments, and returned from other functions, just like any other value type.

---

## Summary of Day 3

- Functions are reusable blocks of code defined with `func`.
- Parameters are inputs; return types declare outputs.
- Go functions can return **multiple values** — a defining, heavily-used feature (especially for error handling).
- Named return values let you label outputs directly in the function signature and use "naked returns."
- Variadic functions (`...type`) accept a flexible number of arguments of the same type.
- Functions are first-class citizens — they can be stored in variables, passed around, and returned from other functions.
- Closures are functions that "remember" variables from their surrounding scope, even after the outer function has finished executing.

**Tomorrow (Day 4):** We'll dive into Arrays, Slices, and Maps — Go's core data structures for storing collections of data. This includes a deep look at slice internals (length vs capacity), which is one of the most frequently asked Go interview topics.
