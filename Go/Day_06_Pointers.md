# Day 6: Pointers

> **Goal for today:** Deeply understand what pointers are, what `&` and `*` actually mean, the real difference between pass-by-value and pass-by-reference, and practical guidance on when to use pointers in Go. This topic connects directly to what you learned on Day 5 (value vs pointer receivers) — today we go underneath that to understand WHY it works that way.

---

## 1. What is a Pointer? (Concept First — Start with Memory)

To understand pointers, we first need a simple mental model of how computer memory works.

Every piece of data in your program (every variable) is stored somewhere in the computer's **memory (RAM)**. Each spot in memory has a unique **address** — like a house number on a street.

**Analogy:** Imagine every variable is a house, and it has:
- A **name** (the variable name you gave it, like `age`) — this is just a label for YOUR convenience, so you don't have to remember the actual address.
- An **address** (a specific location in memory, like `0xc0000140a0` — a hexadecimal number) — this is where the value is ACTUALLY stored.
- **Contents** (the actual value inside, like `25`).

```
  Variable name:  age
  Memory address: 0xc0000140a0   (like a house number)
  Contents:       25              (what's actually inside)
```

A **pointer** is simply a variable that stores an ADDRESS instead of storing a regular value directly. Instead of holding `25`, a pointer holds "the address where 25 lives."

**Analogy continued:** If a normal variable is a house containing furniture (the value), a pointer is like a **piece of paper with the house's address written on it**. The paper itself isn't the house — it just tells you WHERE the house is, so you can go there.

---

## 2. The Two Key Symbols: `&` and `*`

This is where most beginners get confused, because the SAME symbol (`*`) is used in two different contexts. Let's separate them clearly.

### `&` — The "Address Of" Operator

`&` is placed BEFORE a variable name to get its memory address.

```go
package main

import "fmt"

func main() {
    age := 25
    fmt.Println(age)   // Output: 25              (the value itself)
    fmt.Println(&age)  // Output: 0xc0000140a0     (the memory address of age)
}
```

Think of `&age` as literally asking: *"What is the address of the `age` variable?"*

### `*` — Has TWO different meanings depending on context (this is the confusing part)

**Meaning 1: Declaring a pointer TYPE**
```go
var p *int // p is a variable that will hold the ADDRESS of an int
```
Here, `*int` means "a pointer to an int" — i.e., "this variable will store the address of a location that holds an int value." This is used when DECLARING the type.

**Meaning 2: Dereferencing (following the pointer to get the actual value)**
```go
fmt.Println(*p) // gives you the VALUE stored at the address p is pointing to
```
Here, `*p` means "go to the address stored in `p`, and give me the actual value sitting there." This is called **dereferencing** — "following the pointer" to reach the real data.

### Full Example Putting It Together

```go
package main

import "fmt"

func main() {
    age := 25       // a normal variable holding 25
    p := &age       // p is a pointer, now holding the ADDRESS of age

    fmt.Println("Value of age:", age)     // 25
    fmt.Println("Address of age:", &age)  // e.g., 0xc0000140a0
    fmt.Println("Value of p (an address):", p) // e.g., 0xc0000140a0 (same address!)
    fmt.Println("Value at the address p points to:", *p) // 25 (dereferencing)
}
```

### Visual Diagram

```
   Memory:
   ┌───────────────────────┐
   │ Address: 0xc0000140a0  │
   │ Variable name: age      │
   │ Value: 25                │
   └───────────────────────┘
              ^
              │  p holds this address (points to it)
              │
   ┌───────────────────────┐
   │ Address: 0xc0000150b0  │
   │ Variable name: p         │
   │ Value: 0xc0000140a0      │  <- p's "value" IS an address
   └───────────────────────┘

   &age   → gives you: 0xc0000140a0  (the address of age)
   p      → holds:      0xc0000140a0  (same address, stored inside p)
   *p     → gives you:  25            (dereferences p, fetches the actual value at that address)
```

### Quick Memory Trick to Remember `&` vs `*`

- **`&`** = "**A**ddress of" (both start with sounds you can associate — think "&" gives you the **A**ddress).
- **`*`** = "the value **at** this address" when used on a variable that's already a pointer (dereferencing). When used in a type declaration like `*int`, it just means "this is a pointer type."

---

## 3. Why Do Pointers Matter? Pass-by-Value vs Pass-by-Reference

This is the PRACTICAL reason pointers exist, and it directly explains behavior you saw on Day 5.

### Go is "Pass-by-Value" by Default

In Go, when you pass a variable into a function, Go creates a COPY of it. The function works on that copy — changes made inside the function do NOT affect the original variable back in the caller.

```go
package main

import "fmt"

func changeValue(x int) {
    x = 100 // only changes the local copy
}

func main() {
    num := 5
    changeValue(num)
    fmt.Println(num) // Output: 5  <- unchanged! The function only modified its own copy.
}
```

**Visual:**
```
main()                         changeValue(x int)
┌─────────┐                    ┌─────────┐
│ num = 5  │  -- copies 5 -->  │ x = 5    │
└─────────┘                    └─────────┘
                                    │
                                    │ x = 100 (only this local copy changes)
                                    v
                                ┌─────────┐
                                │ x = 100  │
                                └─────────┘
    num is STILL 5 back here, completely untouched
```

### Using a Pointer to Actually Modify the Original

```go
package main

import "fmt"

func changeValue(x *int) {
    *x = 100 // dereference: go to the address, change the value stored THERE
}

func main() {
    num := 5
    changeValue(&num) // pass the ADDRESS of num, not num itself
    fmt.Println(num)  // Output: 100  <- actually changed!
}
```

**Breakdown:**
- `changeValue(x *int)` — the parameter `x` is now a POINTER to an int, not an int directly.
- `changeValue(&num)` — we pass `&num` (the address of `num`), not `num` itself.
- Inside the function, `*x = 100` means "go to the address stored in `x` (which is `num`'s address), and change the value stored there to 100." Since this is the SAME memory location as `num`, the original `num` genuinely changes.

**Visual:**
```
main()                              changeValue(x *int)
┌───────────────────┐               ┌───────────────────┐
│ num = 5             │  <--------  │ x = address of num  │
│ (address: 0xABC)    │   points    └───────────────────┘
└───────────────────┘   to it              │
        ^                                   │ *x = 100
        │                                   │ (follows pointer, changes
        └───────────────────────────────────┘  the value AT that address)

    num is now 100, because we modified the actual memory location it lives at
```

**This is exactly what happened with pointer receivers on Day 5!** A pointer receiver method works the same way — Go passes the memory address of the struct, so changes inside the method reach the real, original struct.

---

## 4. Connecting This Back to Day 5 (Value vs Pointer Receivers)

Now that you understand pointers properly, Day 5's behavior should make complete sense:

```go
type Person struct {
    Name string
}

// Value receiver — Person is COPIED, just like passing an int by value
func (p Person) ChangeName(newName string) {
    p.Name = newName // only changes the copy
}

// Pointer receiver — a POINTER to Person is passed, just like &num
func (p *Person) ChangeName(newName string) {
    p.Name = newName // changes the ORIGINAL, because p holds the real address
}
```

It's the exact same underlying mechanism you just learned with plain `int` values — structs (and receivers) follow identical rules.

---

## 5. The Zero Value of a Pointer: `nil`

If you declare a pointer but don't point it at anything yet, its default (zero) value is `nil` — meaning "points to nothing."

```go
var p *int
fmt.Println(p) // Output: <nil>
```

**IMPORTANT — a very common runtime crash for beginners:** If you try to DEREFERENCE a `nil` pointer (try to access `*p` when `p` is `nil`), your program will crash with a runtime error called a **nil pointer dereference** — this is one of the most common bugs in Go (and many other languages with pointers).

```go
var p *int
fmt.Println(*p) // PANIC: runtime error: invalid memory address or nil pointer dereference
```

**Always check if a pointer is `nil` before dereferencing it, if there's any chance it might not have been assigned:**
```go
if p != nil {
    fmt.Println(*p)
} else {
    fmt.Println("p is nil, nothing to show")
}
```

---

## 6. `new()` — Another Way to Create a Pointer

Go has a built-in function `new()` that allocates memory for a given type and returns a pointer to it (with the zero value already set).

```go
p := new(int) // p is a *int, pointing to a newly allocated int, currently holding 0
fmt.Println(*p) // Output: 0

*p = 42
fmt.Println(*p) // Output: 42
```

In practice, you'll use `&variable` (getting the address of an existing variable) far more often than `new()` in everyday Go code — but it's good to recognize `new()` when you see it, since it does appear in some codebases and interview questions.

---

## 7. Pointers with Slices and Maps — A Special Case

Here's an important nuance: **slices and maps already behave somewhat like references internally**, even without you explicitly using `&` and `*`. This is because a slice internally contains a pointer to its underlying array (as we saw on Day 4), and a map is implemented internally using a pointer to its underlying data structure too.

```go
func modifySlice(s []int) {
    s[0] = 999 // this DOES affect the original slice's underlying array!
}

func main() {
    numbers := []int{1, 2, 3}
    modifySlice(numbers)
    fmt.Println(numbers) // Output: [999 2 3]  <- changed, even without using a pointer!
}
```

**Why does this happen without explicit pointers?** Because when you pass a slice to a function, Go copies the slice's internal "header" (pointer + length + capacity) — but that copied pointer still points to the SAME underlying array as the original. So changing an ELEMENT through the copied slice still reaches the same shared data.

**BUT — appending doesn't always behave this way:**
```go
func addElement(s []int) {
    s = append(s, 100) // may or may not affect the original, depending on capacity!
}

func main() {
    numbers := []int{1, 2, 3}
    addElement(numbers)
    fmt.Println(numbers) // Output: [1 2 3]  <- unchanged! append created a new underlying array
}
```
This is a well-known Go "gotcha" and a common interview trap. If you want a function to reliably modify a slice by appending to it, you should pass a POINTER to the slice, or simply return the modified slice and reassign it in the caller:
```go
func addElement(s []int) []int {
    return append(s, 100)
}

func main() {
    numbers := []int{1, 2, 3}
    numbers = addElement(numbers) // reassigning, the safe and idiomatic way
    fmt.Println(numbers) // Output: [1 2 3 100]
}
```

---

## 8. When Should You Actually Use Pointers in Go?

Here's practical, real-world guidance (also commonly asked in interviews):

| Use a pointer when... | Use a plain value when... |
|---|---|
| You need a function/method to MODIFY the original data | You just need to READ data, no modification needed |
| The data structure (like a large struct) is expensive to copy repeatedly | The data is small (like a single int, bool, or small struct) |
| You want to represent "no value" using `nil` (pointers can be nil; plain values like `int` cannot) | You always want a guaranteed, non-nil value |
| Working with things like linked lists, trees (data structures that inherently need to reference other nodes) | Working with simple, independent, small values |

**Simple guiding question to ask yourself:** *"Does this function/method need to change the original data, or is the data too large to copy efficiently? If yes to either, use a pointer."*

---

## 9. Common Beginner Mistakes

1. **Confusing `&` and `*`** — remember: `&` GETS an address (from a value), `*` FOLLOWS an address (to get back to a value), OR declares a pointer type.
2. **Dereferencing a `nil` pointer** — causes a runtime panic. Always check `if p != nil` when there's uncertainty.
3. **Expecting `append` inside a function to always modify the original slice** — it might not, if the underlying array needs to grow. Prefer returning and reassigning the slice.
4. **Overusing pointers unnecessarily** — for small, simple values (like a single `int` or small struct used only for reading), plain values are simpler and often more efficient than pointers. Don't reach for pointers "just in case" — use them with clear purpose.
5. **Trying to take the address of a literal value directly in some contexts** — e.g., `&5` isn't valid because `5` isn't a variable with an address; you'd need `x := 5; p := &x` first (though Go does allow `&SomeStruct{}` for struct literals specifically, which is a handy shortcut).

---

## 10. Day 6 Practice Exercise

Write a program that:
1. Declares an int variable `score := 50`.
2. Writes a function `doubleScore(s *int)` that doubles the value at the given pointer.
3. Calls `doubleScore(&score)` and prints `score` afterward to confirm it changed.
4. Writes a function `safeDivide(a, b int, result *float64) bool` that:
   - Returns `false` and does nothing to `result` if `b == 0`.
   - Otherwise, stores `a / b` (as a float64) into `*result` and returns `true`.
5. Demonstrate calling `safeDivide` with valid and invalid (zero) divisors.

<details>
<summary>Click to see solution</summary>

```go
package main

import "fmt"

func doubleScore(s *int) {
    *s = *s * 2
}

func safeDivide(a, b int, result *float64) bool {
    if b == 0 {
        return false
    }
    *result = float64(a) / float64(b)
    return true
}

func main() {
    score := 50
    doubleScore(&score)
    fmt.Println("Doubled score:", score) // 100

    var result float64
    ok := safeDivide(10, 2, &result)
    if ok {
        fmt.Println("Division result:", result) // 5
    }

    ok = safeDivide(10, 0, &result)
    if !ok {
        fmt.Println("Cannot divide by zero!")
    }
}
```
</details>

---

## 11. Day 6 Interview Questions (Quick Prep)

1. **Q: What does the `&` operator do?**
   A: It returns the memory address of a variable.

2. **Q: What does the `*` operator do?**
   A: Depends on context — in a type declaration (`*int`), it declares a pointer type. When applied to a pointer variable (`*p`), it dereferences the pointer, retrieving the value stored at that address.

3. **Q: Is Go pass-by-value or pass-by-reference by default?**
   A: Pass-by-value. Function arguments (including structs) are copied by default, unless you explicitly pass a pointer.

4. **Q: What is a nil pointer, and what happens if you dereference one?**
   A: A nil pointer points to nothing (the zero value for pointer types). Dereferencing it causes a runtime panic: "nil pointer dereference."

5. **Q: Why do slices sometimes appear to be passed "by reference" even without explicit pointers?**
   A: Because a slice internally contains a pointer to its underlying array. Modifying an existing element through a copied slice affects the shared underlying array. However, operations like `append` can allocate a NEW underlying array, breaking that shared connection — so append-based modifications aren't reliably visible outside the function unless the slice is returned/reassigned or a pointer to the slice is used.

6. **Q: When would you choose a pointer over a value in Go?**
   A: When the function/method needs to modify the original data, when the data structure is large and copying would be costly, or when you need to represent the possibility of "no value" using `nil`.

---

## Summary of Day 6

- A pointer is a variable that stores a memory address rather than a direct value.
- `&variable` gets the address of a variable; `*pointer` dereferences a pointer to get the value at that address (or declares a pointer type, depending on context).
- Go is pass-by-value by default — functions get copies of arguments unless you explicitly pass a pointer.
- Pointer receivers (from Day 5) work by passing the struct's address, allowing methods to modify the original data — this is the same mechanism as passing `&variable` to a plain function.
- `nil` is the zero value for pointers; dereferencing a nil pointer causes a runtime panic.
- Slices and maps have pointer-like behavior internally, but `append` can break this because it may allocate a new underlying array.
- Use pointers when you need to modify original data, avoid costly copying of large structures, or need the possibility of a "no value" (`nil`) state.

**Tomorrow (Day 7):** We'll cover Interfaces — one of Go's most powerful and distinctive features. You'll learn how Go achieves polymorphism WITHOUT an `implements` keyword (implicit implementation), the empty interface (`any`), type assertions, and type switches.
