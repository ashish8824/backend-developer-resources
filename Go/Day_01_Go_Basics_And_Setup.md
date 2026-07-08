# Day 1: Go Basics & Setup

> **Goal for today:** By the end of Day 1, you will understand what Go is, why it exists, how to install it, how to write and run your first Go program, and how variables, constants, and data types work. Everything is explained assuming you have **zero prior programming background** (not even Java), so we go slow and explain every symbol.

---

## 1. What is Go (Golang)?

Go is a programming language created by **Google** in 2009 by three engineers: Robert Griesemer, Rob Pike, and Ken Thompson.

Think of Go like a **new, simpler toolbox** built because older toolboxes (like C++ and Java) had become too complicated for the kind of large-scale software Google was building.

### Why was Go created?
Google had huge codebases with thousands of engineers. They faced problems like:
- Code took too long to **compile** (compiling = converting human-readable code into a program the computer can run).
- Languages like C++ were too complex, with lots of confusing features.
- They needed a language that made it **easy to write programs that do many things at once** (this is called *concurrency* — we'll cover this deeply on Day 9 and 10).

So Go was designed with three main promises:
1. **Simple** — small language, easy to learn, few keywords (only 25!).
2. **Fast** — compiles to machine code directly (no middleman like a virtual machine slowing it down).
3. **Great at concurrency** — doing multiple tasks at once is a first-class feature, not an afterthought.

### Where is Go used in the real world?
| Company/Project | What they use Go for |
|---|---|
| Docker | Container technology is built in Go |
| Kubernetes | The most popular container orchestration tool, written in Go |
| Uber | Backend microservices |
| Netflix | Backend tools |
| PayPal | Payment processing services |

**Simple analogy:** If C++ is like a fully-loaded Swiss Army knife with 50 tools (powerful but heavy and complex), Go is like a clean, sharp kitchen knife — it does fewer things, but does them very well and is easy to hold.

---

## 2. Compiled vs Interpreted Languages (Important Foundation Concept)

Since you're new to programming, let's clear this up first because it explains *why* Go behaves the way it does.

- **Interpreted languages** (like Python, JavaScript): Code is read and executed line-by-line by another program (an interpreter) *while it runs*. Slower, but flexible.
- **Compiled languages** (like Go, C, C++): Code is first fully translated into **machine code** (binary instructions the computer's processor understands) by a **compiler**. This creates an executable file. Then you run that file directly. Faster, because there's no translation happening while the program runs.

```
Your Go Code (.go file)
        |
        |  (go build compiles it)
        v
  Machine Code / Executable File
        |
        |  (you run this file)
        v
     Program Output
```

Go is a **compiled language**. This is why Go programs run fast — similar in speed to C/C++, but much easier to write.

---

## 3. Installing Go

### Step 1: Download Go
Go to the official website: **https://go.dev/dl/**
Download the installer for your operating system (Windows/Mac/Linux).

### Step 2: Install
- **Windows:** Run the `.msi` installer, follow the on-screen steps (Next → Next → Finish).
- **Mac:** Run the `.pkg` installer, or if you use Homebrew: `brew install go`
- **Linux:** Download the `.tar.gz` file and extract it, or use your package manager: `sudo apt install golang-go`

### Step 3: Verify Installation
Open your terminal (Command Prompt / Terminal / PowerShell) and type:

```bash
go version
```

If installed correctly, you'll see something like:
```
go version go1.22.0 linux/amd64
```

This confirms Go is installed and tells you the version.

### Step 4: Check your environment
```bash
go env
```
This shows configuration details like where Go is installed, where your packages get downloaded, etc. You don't need to memorize this — just know this command exists for troubleshooting.

---

## 4. Go Workspace: GOPATH vs Go Modules

This confuses a lot of beginners, so let's clear it up with simple language.

### The Old Way: GOPATH
In early Go versions, ALL your Go code had to live inside one specific folder called `GOPATH` (like `C:\Users\YourName\go`). Every project had to follow a strict folder structure inside this one location. This was rigid and annoying — like being told you can only ever cook in one specific kitchen in your house, no matter what you're cooking.

### The Modern Way: Go Modules (what you'll use)
Since Go 1.11+, we use **Go Modules**. This lets you create a Go project **anywhere** on your computer, and Go tracks the project's dependencies (external code libraries your project uses) using a file called `go.mod`.

Think of `go.mod` like a **recipe ingredients list** — it tells Go: "this project needs these specific external packages, at these specific versions."

**You will use Go Modules for everything in this course.** We'll create our first module in the next section.

---

## 5. Your First Go Program

### Step 1: Create a project folder
```bash
mkdir hello-go
cd hello-go
```
- `mkdir` = "make directory" → creates a new folder
- `cd` = "change directory" → moves you into that folder

### Step 2: Initialize a Go module
```bash
go mod init hello-go
```
This creates a `go.mod` file — this officially tells Go "this folder is a Go project named hello-go."

### Step 3: Create your first file
Create a file named `main.go` and write this code:

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Go!")
}
```

### Step 4: Run it
```bash
go run main.go
```

Output:
```
Hello, Go!
```

### Now let's break down EVERY LINE of this code (since you're new to programming):

```go
package main
```
- **What it means:** Every Go file must belong to a "package" (a way of grouping related code together — like a folder label). This tells Go: "the code in this file belongs to the `main` package."
- **Why `main` specifically?** The package named `main` is special — it tells Go "this is a program that can be run directly" (as opposed to a package that's just a reusable library meant to be imported by other programs).

```go
import "fmt"
```
- **What it means:** `import` brings in code from another package so you can use it.
- **`fmt`** stands for "format" — it's part of Go's **standard library** (pre-written code that comes bundled with Go, so you don't have to write everything from scratch). The `fmt` package has functions for printing text to the screen, reading input, formatting strings, etc.
- **Analogy:** Think of `import` like borrowing a specific toolbox from a shared tool shed before you start work. You only borrow the tools (`fmt`) you actually need.

```go
func main() {
    ...
}
```
- **`func`** is the keyword to declare a function (a reusable block of code that does something).
- **`main`** is again a special name — this is the **entry point** of your program. When you run a Go program, execution always starts inside the `main()` function. It's like the "front door" of your house — the very first place you walk in.
- The `{ }` curly braces mark the **beginning and end** of the function's code block — everything inside them is what the function does.

```go
fmt.Println("Hello, Go!")
```
- `fmt.Println` — this is calling the `Println` function that lives inside the `fmt` package (that's what the dot `.` means — "go inside this package and use this specific function").
- `Println` = "Print Line" → prints the text and moves to a new line afterward.
- `"Hello, Go!"` is a **string** (text data) — text must always be wrapped in double quotes in Go.

### Visual Summary of Program Flow

```
 ┌─────────────────────────────┐
 │  package main                │  <- declares this file's package
 ├─────────────────────────────┤
 │  import "fmt"                │  <- loads the fmt toolbox
 ├─────────────────────────────┤
 │  func main() {                │  <- program starts executing HERE
 │      fmt.Println("Hello, Go!")│  <- runs this line
 │  }                             │  <- program ends here
 └─────────────────────────────┘
```

### `go run` vs `go build` (Important distinction, common interview question)

| Command | What it does |
|---|---|
| `go run main.go` | Compiles AND runs the program immediately, but does NOT save a permanent executable file. Great for quick testing during development. |
| `go build main.go` | Compiles the program into a permanent executable file (e.g., `main.exe` on Windows, or `main` on Mac/Linux) that you can run anytime later, even without Go installed on that machine. |

```bash
go build main.go   # creates an executable file
./main              # runs the executable (Mac/Linux)
main.exe            # runs the executable (Windows)
```

**Simple analogy:** `go run` is like microwaving food and eating it immediately (quick, but nothing is saved). `go build` is like cooking a meal and storing it in a container (takes a moment longer, but now you have a ready-to-eat file you can run anytime).

---

## 6. Variables in Go

A **variable** is a named container that stores a value in the computer's memory so you can use and change it later.

**Analogy:** Think of a variable like a labeled box. You write a name on the box (like `age`) and put something inside it (like the number `25`). Later, you can look inside the box using its label, or replace what's inside.

### Declaring variables — Method 1: using `var`

```go
var age int = 25
var name string = "Rahul"
var isStudent bool = true
```

Breaking this down:
- `var` = keyword to declare a variable
- `age` = the name you're giving this variable
- `int` = the **type** of data it will hold (a whole number, no decimals)
- `= 25` = assigns the value 25 to it

You can also declare without assigning a value immediately:
```go
var age int
age = 25
```
Here, `age` starts as `0` (Go automatically gives numbers a default value of `0` until you assign something) and then we set it to 25 on the next line.

### Declaring variables — Method 2: Short variable declaration `:=`

This is the way you'll use MOST OFTEN in Go (it's the idiomatic/preferred style):

```go
age := 25
name := "Rahul"
isStudent := true
```

- `:=` automatically figures out (infers) the type based on the value you give it. So `age := 25` automatically makes `age` an `int`, because 25 is a whole number.
- **Important rule:** `:=` can ONLY be used inside functions (not for variables declared outside of any function, at the top "package level"). We'll see this distinction as we progress.

### Multiple variables at once
```go
var x, y int = 10, 20
name, age := "Rahul", 25
```

### Why does Java-comfort matter here?
Since you mentioned you're not very comfortable with Java — good news: Go is actually **simpler** than Java in this area. Java requires you to always specify the type explicitly (`int age = 25;`) and always end lines with semicolons. Go lets you skip the type (using `:=`) and does **not** require semicolons at the end of lines — Go's compiler adds them automatically behind the scenes.

---

## 7. Constants

A **constant** is like a variable, but its value can **never change** once set. Use `const` when you know a value should stay fixed throughout the program (like the value of Pi, or a fixed tax rate).

```go
const pi = 3.14159
const appName = "MyGoApp"
```

If you try to change a constant later in the code, Go will give you a **compile-time error** (an error caught before the program even runs) — this protects you from accidentally changing something that should never change.

**Analogy:** A variable is like a whiteboard (write, erase, rewrite). A constant is like something engraved in stone — permanent.

---

## 8. Basic Data Types in Go

A **data type** tells the computer what *kind* of value a variable holds, and how much memory to set aside for it.

### Numeric Types

| Type | Description | Example |
|---|---|---|
| `int` | Whole numbers (no decimals). Size depends on your system (usually 64-bit) | `10`, `-5`, `2024` |
| `int8`, `int16`, `int32`, `int64` | Whole numbers with a FIXED size in bits (used when you need precise control over memory) | `int8` can hold -128 to 127 |
| `uint` | Unsigned integer — only positive whole numbers (no negative) | `10`, `2024` |
| `float32`, `float64` | Numbers with decimal points | `3.14`, `-0.5` |

### Text Type

| Type | Description | Example |
|---|---|---|
| `string` | A sequence of characters (text), always in double quotes | `"Hello"`, `"Go123"` |

### Boolean Type

| Type | Description | Example |
|---|---|---|
| `bool` | Only two possible values: `true` or `false`. Used for yes/no, on/off type logic | `true`, `false` |

### Visual: Memory Box Analogy

```
 int  age     →  [ 25 ]        (a box that holds whole numbers)
 string name  →  [ "Rahul" ]   (a box that holds text)
 bool isOn    →  [ true ]      (a box that holds true/false only)
 float64 gpa  →  [ 8.75 ]      (a box that holds decimal numbers)
```

### Checking a variable's type (for learning purposes)
```go
package main

import "fmt"

func main() {
    age := 25
    fmt.Printf("Type of age: %T\n", age)
}
```
Output: `Type of age: int`

- `Printf` = "Print Formatted" — lets you insert variable values into a string using special placeholders (called **format verbs**).
- `%T` is a format verb meaning "print the Type of this variable."
- `\n` means "new line" (move the cursor to the next line after printing).

---

## 9. Type Conversion

Go is **strongly typed**, meaning it will NOT automatically convert one type into another for you (unlike some languages). You must convert manually. This prevents accidental bugs.

```go
var age int = 25
var ageFloat float64 = float64(age)  // manually converting int to float64

var price float64 = 99.99
var priceInt int = int(price)  // manually converting float64 to int (this will CUT OFF the decimal, becoming 99, not rounded)
```

**Common beginner mistake:** Trying to do this:
```go
var age int = 25
var name string = "Age: " + age  // ERROR! Cannot combine string and int directly
```
This will fail to compile because Go doesn't automatically convert `int` to `string`. You must convert explicitly:
```go
import "strconv"
var name string = "Age: " + strconv.Itoa(age)  // Itoa = "Integer to ASCII" (converts int to string)
```

---

## 10. Comments in Go

Comments are notes in your code that the computer ignores — they're for humans to read and understand the code.

```go
// This is a single-line comment

/*
This is a
multi-line comment
*/
```

Good practice: use comments to explain **why** you did something, not just **what** the code does (the code itself already shows what it does).

---

## 11. Putting It All Together — A Complete Example

```go
package main

import "fmt"

func main() {
    // Declaring variables
    var name string = "Priya"
    age := 24
    const country = "India"
    isEmployed := true
    gpa := 8.9

    // Printing values using fmt.Println (simple print)
    fmt.Println("Name:", name)
    fmt.Println("Age:", age)
    fmt.Println("Country:", country)
    fmt.Println("Employed:", isEmployed)
    fmt.Println("GPA:", gpa)

    // Printing using fmt.Printf (formatted print with placeholders)
    fmt.Printf("%s is %d years old and lives in %s.\n", name, age, country)
}
```

**Output:**
```
Name: Priya
Age: 24
Country: India
Employed: true
GPA: 8.9
Priya is 24 years old and lives in India.
```

### Format verbs used in `Printf` (quick reference table — you'll use these a lot)

| Verb | Meaning | Used for |
|---|---|---|
| `%s` | string | text values |
| `%d` | decimal (integer) | whole numbers |
| `%f` | float | decimal numbers |
| `%t` | boolean | true/false |
| `%v` | default format | any value (works for anything, useful when unsure) |
| `%T` | type | shows the data type |

---

## 12. Common Beginner Mistakes (so you avoid them)

1. **Declaring a variable and never using it.** Go will refuse to compile if you declare a variable with `:=` or `var` and never use it anywhere. This is intentional — Go enforces clean code.
   ```go
   age := 25  // ERROR if 'age' is never used later: "declared and not used"
   ```
2. **Forgetting `package main` or `func main()`** — without these, Go won't know where your program starts.
3. **Using `:=` outside a function** — short declaration only works inside functions.
4. **Mismatched types** — trying to combine a `string` and `int` directly without converting.

---

## 13. Day 1 Practice Exercise

Try writing a program that:
1. Declares your name (string), age (int), and whether you're currently learning Go (bool).
2. Prints all three using `fmt.Println`.
3. Prints a sentence using `fmt.Printf` like: `"Hi, my name is X and I am Y years old."`

Try it yourself before checking the solution below.

<details>
<summary>Click to see solution</summary>

```go
package main

import "fmt"

func main() {
    myName := "Your Name"
    myAge := 22
    learningGo := true

    fmt.Println(myName)
    fmt.Println(myAge)
    fmt.Println(learningGo)

    fmt.Printf("Hi, my name is %s and I am %d years old.\n", myName, myAge)
}
```
</details>

---

## 14. Day 1 Interview Questions (Quick Prep)

1. **Q: What is Go and why was it created?**
   A: Go is a statically typed, compiled language created by Google to solve problems with slow builds, complexity in large codebases, and to make concurrent programming easier.

2. **Q: Difference between `go run` and `go build`?**
   A: `go run` compiles and runs code immediately without saving an executable. `go build` compiles and creates a permanent executable file.

3. **Q: Difference between `var` and `:=`?**
   A: `var` can be used anywhere (inside or outside functions) and lets you optionally specify a type explicitly. `:=` is short-hand, only usable inside functions, and infers the type automatically.

4. **Q: What happens if you declare a variable and never use it?**
   A: Go throws a compile-time error: "declared and not used." This is intentional to keep code clean.

5. **Q: Is Go a compiled or interpreted language?**
   A: Compiled — Go code is translated directly into machine code before running, which makes it fast.

6. **Q: What is `go.mod`?**
   A: A file that defines a Go module — it tracks the module's name and its dependencies (external packages it needs).

---

## Summary of Day 1

- Go is a simple, fast, compiled language built by Google, great for concurrency.
- We installed Go and verified it with `go version`.
- We use Go Modules (`go mod init`) to manage projects instead of the old GOPATH system.
- Every Go program starts execution in `func main()` inside `package main`.
- Variables can be declared with `var` (explicit) or `:=` (short, type-inferred).
- Constants (`const`) hold values that never change.
- Go has basic types: `int`, `float64`, `string`, `bool`.
- Go requires manual type conversion — it won't do it automatically.
- `fmt` package is used for printing (`Println`, `Printf`).

**Tomorrow (Day 2):** We'll learn Control Flow — `if/else`, Go's unique `switch` statement, and the single `for` loop that Go uses for every kind of looping (while loops, infinite loops, and traditional counting loops).
