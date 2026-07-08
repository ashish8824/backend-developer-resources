# Day 5: Structs & Methods

> **Goal for today:** Learn how Go groups related data together using structs (a blueprint for custom data types), how to attach behavior to structs using methods, the crucial difference between value receivers and pointer receivers (a top interview question), and struct embedding — Go's alternative to traditional class inheritance.

---

## 1. What is a Struct? (Concept First)

So far, we've stored single pieces of data in variables (`name string`, `age int`). But real-world things usually have MULTIPLE related properties together. A "Person" isn't just a name — it's a name AND an age AND an email, all bundled together as one unit.

A **struct** (short for "structure") is a custom data type that groups multiple related fields together under one name.

**Analogy:** Think of a struct like a **form** you fill out — a job application form has labeled fields: Name, Age, Email, Phone Number. The struct is the blueprint (the empty form template), and each time you fill it out for a specific person, that's called an **instance** of the struct.

```
 Struct "Person" (the blueprint/form):
 ┌─────────────────────────┐
 │ Name:    ___________     │
 │ Age:     ___________     │
 │ Email:   ___________     │
 └─────────────────────────┘

 An instance (a filled-out form):
 ┌─────────────────────────┐
 │ Name:    Rahul           │
 │ Age:     25               │
 │ Email:   rahul@mail.com  │
 └─────────────────────────┘
```

If you're coming from Java in your mind at all: a Go `struct` is somewhat similar to a Java class, but WITHOUT built-in inheritance, and WITHOUT constructors in the traditional sense. Go handles these differently, as we'll see.

---

## 2. Defining and Using a Struct

### Step 1: Define the struct (the blueprint)

```go
package main

import "fmt"

type Person struct {
    Name  string
    Age   int
    Email string
}

func main() {
    var p1 Person
    p1.Name = "Rahul"
    p1.Age = 25
    p1.Email = "rahul@mail.com"

    fmt.Println(p1) // Output: {Rahul 25 rahul@mail.com}
}
```

**Breakdown:**
- `type Person struct { ... }` — `type` is a keyword used to define a new custom type. `Person` is the name we're giving this new type. `struct { ... }` declares that this type is a struct, with the fields listed inside the braces.
- `Name string`, `Age int`, `Email string` — these are the **fields** of the struct, each with its own name and type, just like declaring separate variables, but grouped under one struct.
- `var p1 Person` — declares a variable `p1` of type `Person`. At this point, all its fields hold their zero values (`""` for string, `0` for int).
- `p1.Name = "Rahul"` — the dot `.` is used to access (or set) a specific field of the struct instance.

### Step 2: Creating a struct with values directly (struct literal)

```go
p2 := Person{
    Name:  "Priya",
    Age:   28,
    Email: "priya@mail.com",
}

fmt.Println(p2)
```

You can also skip the field names if you provide values in the EXACT order the fields were declared (though this is considered less readable and more error-prone, so most Go developers prefer naming the fields explicitly):
```go
p3 := Person{"Amit", 30, "amit@mail.com"}
```

### Accessing individual fields

```go
fmt.Println(p2.Name)  // Priya
fmt.Println(p2.Age)   // 28
```

---

## 3. Nested Structs

A struct can contain another struct as one of its fields — this lets you model more complex, real-world relationships.

```go
type Address struct {
    City    string
    Pincode string
}

type Person struct {
    Name    string
    Age     int
    Address Address // a struct field, holding another struct type
}

func main() {
    p := Person{
        Name: "Rahul",
        Age:  25,
        Address: Address{
            City:    "Bangalore",
            Pincode: "560001",
        },
    }

    fmt.Println(p.Address.City) // Output: Bangalore
}
```
Notice `p.Address.City` — you chain dots `.` to reach fields nested inside fields.

---

## 4. Methods: Attaching Behavior to Structs

So far, our `Person` struct just holds data — it doesn't DO anything. A **method** is a function that's attached to a specific type (usually a struct), giving that type its own behavior.

### Defining a Method

```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

func (p Person) Greet() {
    fmt.Println("Hi, my name is", p.Name)
}

func main() {
    p1 := Person{Name: "Rahul", Age: 25}
    p1.Greet() // Output: Hi, my name is Rahul
}
```

**Breakdown — this is the key new syntax to understand:**
```go
func (p Person) Greet() {
```
- `func` — as usual, declares a function.
- `(p Person)` — this is called the **receiver**. This is what makes this an ordinary function into a METHOD — it's now specifically attached to the `Person` type. `p` is just a variable name (like a parameter) representing "whichever Person instance called this method," and `Person` is the type it's attached to.
- `Greet()` — the method name.
- Inside the method body, `p.Name` accesses the `Name` field of whichever `Person` instance called this method.

**Calling it:** `p1.Greet()` — this looks exactly like accessing a struct field, but since `Greet` is a method (not a field), Go knows to run the function, automatically passing `p1` in as the receiver `p`.

**Analogy:** Think of a receiver like saying "this action belongs to this type of object." A `Person` can `Greet()`. A `Car` might `StartEngine()`. The receiver `(p Person)` is like saying "this ability is specifically something a Person object can do."

---

## 5. Value Receivers vs Pointer Receivers (MAJOR Interview Topic)

This is genuinely one of the most frequently asked Go interview questions, so let's build a rock-solid understanding.

### Value Receiver (what we just saw)

```go
func (p Person) Greet() {
    fmt.Println("Hi, my name is", p.Name)
}
```

With a **value receiver**, Go makes a **COPY** of the struct when the method is called. Any changes made to `p` inside the method only affect that copy — the ORIGINAL struct outside the method remains untouched.

```go
func (p Person) ChangeName(newName string) {
    p.Name = newName // this only changes the COPY, not the original!
}

func main() {
    person := Person{Name: "Rahul"}
    person.ChangeName("Amit")
    fmt.Println(person.Name) // Output: Rahul  <- unchanged! surprising to beginners
}
```

### Pointer Receiver (using `*`)

```go
func (p *Person) ChangeName(newName string) {
    p.Name = newName // this changes the ORIGINAL struct
}

func main() {
    person := Person{Name: "Rahul"}
    person.ChangeName("Amit")
    fmt.Println(person.Name) // Output: Amit  <- actually changed!
}
```

**Breakdown:**
- `(p *Person)` — the `*` before `Person` means this receiver is a **pointer** to a `Person`, not a copy of it. (We'll deeply cover pointers on Day 6 — for now, just know: a pointer holds the actual MEMORY ADDRESS of the original struct, so changes made through it affect the real, original data, not a copy.)
- Go conveniently lets you write `p.Name = newName` even though `p` is technically a pointer — Go automatically handles the "dereferencing" for you behind the scenes, so you don't need extra special syntax here.

### Visual: Value Receiver vs Pointer Receiver

```
VALUE RECEIVER (copy):
   Original Person (Name: "Rahul")
              │
              │  copies data
              v
   Method's local copy (Name: "Rahul")
              │
              │  method changes the COPY only
              v
   Copy now has Name: "Amit"  ← but original is untouched!


POINTER RECEIVER (reference):
   Original Person (Name: "Rahul")
              ^
              │  pointer points DIRECTLY at the original
              │
   Method modifies through the pointer
              │
              v
   Original Person is now: Name: "Amit"  ← actually changed!
```

### When should you use a pointer receiver vs a value receiver?

This is the exact interview question you'll likely face — here are the practical rules Go developers follow:

| Use a **pointer receiver** when... | Use a **value receiver** when... |
|---|---|
| The method needs to MODIFY the struct's fields | The method only READS data, doesn't change anything |
| The struct is large (copying it repeatedly would be wasteful/slow) | The struct is small (like just an int or two fields) |
| You want consistency — if ANY method on a type uses a pointer receiver, it's conventional to make ALL methods on that type use pointer receivers too | The struct represents an immutable value conceptually (e.g., a simple `Point{X, Y}`) |

**Simple rule of thumb to remember:** *"If the method changes the struct, or the struct is large, use a pointer receiver. If it's just reading small data, a value receiver is fine."*

### A subtlety: calling pointer receiver methods

Go is smart about this — even if you have a regular (non-pointer) variable, Go automatically takes its address for you when calling a pointer receiver method:
```go
person := Person{Name: "Rahul"}
person.ChangeName("Amit") // Go automatically converts this to (&person).ChangeName("Amit")
```
You don't need to manually write `&person.ChangeName(...)` — Go handles this conversion automatically for convenience, AS LONG AS `person` is an addressable variable (a regular variable you can point to, which is normally the case).

---

## 6. Struct Embedding (Go's Alternative to Inheritance)

Go does **NOT** have traditional class inheritance like Java (`class Dog extends Animal`). Instead, Go uses a concept called **embedding**, which achieves code reuse in a different, simpler way — often described as "composition over inheritance."

### Example

```go
package main

import "fmt"

type Animal struct {
    Name string
    Age  int
}

func (a Animal) Describe() {
    fmt.Println(a.Name, "is", a.Age, "years old")
}

type Dog struct {
    Animal // embedded struct — notice: no field name given, just the type
    Breed  string
}

func main() {
    d := Dog{
        Animal: Animal{Name: "Buddy", Age: 3},
        Breed:  "Labrador",
    }

    d.Describe()             // Output: Buddy is 3 years old  <- inherited from Animal!
    fmt.Println(d.Name)      // Output: Buddy  <- accessing Animal's field directly through Dog
    fmt.Println(d.Breed)     // Output: Labrador
}
```

**Breakdown — this is the key idea:**
- Inside `Dog`, we wrote just `Animal` (the type name), with NO field name in front of it. This is called an **embedded field** (or "anonymous field").
- Because `Animal` is embedded (not named), Go automatically "promotes" all of `Animal`'s fields (`Name`, `Age`) and methods (`Describe()`) up to `Dog`. That's why we can call `d.Describe()` and `d.Name` directly, even though those actually belong to `Animal`.
- This gives Go a form of code reuse similar to inheritance, WITHOUT the complexity of a full class hierarchy system. It's called **composition** — a `Dog` "has an" `Animal` inside it, rather than "is an" `Animal` in the strict class-hierarchy sense (even though it behaves conveniently like it "is one" due to promotion).

### Visual: Embedding

```
   Dog struct:
   ┌─────────────────────────────┐
   │  Animal (embedded)           │
   │  ┌─────────────────────┐    │
   │  │ Name: "Buddy"         │   │
   │  │ Age:  3                │   │
   │  └─────────────────────┘    │
   │  Breed: "Labrador"            │
   └─────────────────────────────┘

   d.Name  → looks inside Dog, doesn't find "Name" directly,
             but finds it inside the embedded Animal → returns "Buddy"
```

### Overriding an embedded method

If `Dog` defines its own method with the SAME name as one in `Animal`, `Dog`'s own version takes priority (this is similar to "overriding" in traditional OOP):

```go
func (d Dog) Describe() {
    fmt.Println(d.Name, "is a good dog, aged", d.Age)
}
```
Now `d.Describe()` calls `Dog`'s version, not `Animal`'s, because Go looks for the method on the exact type first before checking embedded types.

---

## 7. Structs and Functions (Passing Structs Around)

By default, when you pass a struct to a function, Go passes a **COPY** of it (just like the value receiver behavior we saw earlier):

```go
func updateAge(p Person) {
    p.Age = 100 // only changes the local copy
}

func main() {
    person := Person{Name: "Rahul", Age: 25}
    updateAge(person)
    fmt.Println(person.Age) // Output: 25  <- unchanged
}
```

To actually modify the original struct through a function, pass a **pointer** instead:
```go
func updateAge(p *Person) {
    p.Age = 100 // modifies the ORIGINAL
}

func main() {
    person := Person{Name: "Rahul", Age: 25}
    updateAge(&person) // & gets the memory address of person
    fmt.Println(person.Age) // Output: 100  <- changed!
}
```
This exact same value-vs-pointer logic applies to structs passed to regular functions, not just methods. We'll go MUCH deeper into pointers and the `&`/`*` symbols tomorrow (Day 6), so don't worry if this feels slightly new right now — today's goal was just to see it in the context of structs.

---

## 8. Common Beginner Mistakes

1. **Forgetting that value receivers/parameters work on COPIES** — expecting a change inside a method/function to reflect in the original struct, when actually a value receiver was used.
2. **Inconsistent receiver types** — mixing value receivers and pointer receivers across different methods of the SAME struct type. Best practice: pick one style (usually pointer, if ANY method needs to modify) and stick with it for all methods of that type.
3. **Confusing embedding with inheritance** — embedding is composition, not a strict "is-a" class hierarchy. There's no polymorphism in the traditional OOP sense (a `Dog` isn't automatically usable everywhere an `Animal` is expected — that requires interfaces, which we cover on Day 7).
4. **Forgetting to name fields in a struct literal** when the order might not match the declared field order — always prefer named fields (`Person{Name: "X", Age: 1}`) over positional (`Person{"X", 1}`) for clarity and safety.

---

## 9. Day 5 Practice Exercise

Write a program that:
1. Defines a struct `Book` with fields `Title`, `Author`, and `Pages` (int).
2. Defines a method `Summary()` (value receiver) that prints something like: `"War and Peace by Leo Tolstoy (1225 pages)"`.
3. Defines a method `AddPages(extra int)` (pointer receiver) that increases the `Pages` field by `extra`.
4. In `main()`, create a `Book` instance, call `Summary()`, then call `AddPages(50)` and call `Summary()` again to confirm the page count changed.

<details>
<summary>Click to see solution</summary>

```go
package main

import "fmt"

type Book struct {
    Title  string
    Author string
    Pages  int
}

func (b Book) Summary() {
    fmt.Printf("%s by %s (%d pages)\n", b.Title, b.Author, b.Pages)
}

func (b *Book) AddPages(extra int) {
    b.Pages += extra
}

func main() {
    book := Book{
        Title:  "War and Peace",
        Author: "Leo Tolstoy",
        Pages:  1225,
    }

    book.Summary()
    book.AddPages(50)
    book.Summary()
}
```
Output:
```
War and Peace by Leo Tolstoy (1225 pages)
War and Peace by Leo Tolstoy (1275 pages)
```
</details>

---

## 10. Day 5 Interview Questions (Quick Prep)

1. **Q: What is a struct in Go?**
   A: A custom data type that groups multiple related fields together, similar to a lightweight class without inheritance or constructors.

2. **Q: What's the difference between a value receiver and a pointer receiver?**
   A: A value receiver gets a COPY of the struct — changes inside the method don't affect the original. A pointer receiver gets a reference to the ORIGINAL struct — changes inside the method DO affect the original.

3. **Q: When should you use a pointer receiver over a value receiver?**
   A: When the method needs to modify the struct's fields, when the struct is large (to avoid expensive copying), or for consistency if other methods on the same type already use pointer receivers.

4. **Q: Does Go support class inheritance like Java?**
   A: No. Go uses struct embedding (composition) instead, where one struct can embed another, automatically promoting its fields and methods.

5. **Q: If a struct is passed to a regular function (not a method), is it passed by value or by reference by default?**
   A: By value (a copy), unless you explicitly pass a pointer to it using `&`.

6. **Q: What happens if both an embedded struct and the outer struct define a method with the same name?**
   A: The outer struct's own method takes priority (similar to overriding).

---

## Summary of Day 5

- Structs group related fields together into a custom data type, defined with `type Name struct { ... }`.
- Methods attach behavior to a type using a receiver: `func (receiverName Type) MethodName() { ... }`.
- Value receivers work on a COPY of the struct (changes don't persist to the original); pointer receivers (`*Type`) work on the ORIGINAL struct (changes DO persist).
- Rule of thumb: use pointer receivers when modifying data or working with large structs; value receivers for small, read-only operations.
- Struct embedding lets one struct include another, automatically promoting its fields/methods — Go's composition-based alternative to inheritance.
- Structs passed to functions are copied by default, unless a pointer is passed explicitly.

**Tomorrow (Day 6):** We'll do a focused, deep dive into Pointers — what `&` and `*` really mean, pass-by-value vs pass-by-reference in full detail, and practical guidance on when and why to use pointers in Go.
