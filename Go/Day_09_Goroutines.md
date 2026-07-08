# Day 9: Goroutines (Concurrency Part 1)

> **Goal for today:** Understand what goroutines are, how they differ from traditional threads, how to launch them with the `go` keyword, and how to coordinate multiple goroutines using WaitGroups. This is Go's most famous strength and a heavily tested interview topic.

---

## 1. What is Concurrency? (Concept First)

Before we touch Go-specific syntax, let's build a clear mental model of what "concurrency" even means, since it's often confused with a related word: "parallelism."

**Concurrency** means DEALING with multiple tasks at once — structuring your program so multiple things are IN PROGRESS at overlapping times. It doesn't necessarily mean they're literally happening at the exact same instant.

**Parallelism** means actually EXECUTING multiple tasks at the exact same physical instant, typically requiring multiple CPU cores.

**Analogy:** Imagine a single chef in a kitchen cooking three dishes. The chef works on Dish A for a bit, then switches to stir Dish B, then chops vegetables for Dish C, then goes back to Dish A — juggling all three by switching between them rapidly. This is **concurrency** — one chef, handling multiple tasks by interleaving them, even though only one thing is happening at any exact instant.

Now imagine THREE chefs, each cooking one dish simultaneously, at the exact same time. This is **parallelism** — genuinely simultaneous execution, requiring multiple "workers" (or CPU cores).

```
CONCURRENCY (one worker, juggling tasks):
Time:  [--A--][--B--][--A--][--C--][--B--][--A--]
        (interleaved, but never truly simultaneous)

PARALLELISM (multiple workers, truly simultaneous):
Worker 1: [------A------]
Worker 2: [------B------]
Worker 3: [------C------]
        (all happening at the exact same time)
```

Go is designed to make concurrency easy to WRITE, and if your computer has multiple CPU cores, Go can automatically take advantage of them to also achieve real parallelism — but the CODE you write for concurrency looks the same either way. This is a famous line from Rob Pike (one of Go's creators): *"Concurrency is not parallelism"* — concurrency is about STRUCTURE (how you organize independent tasks); parallelism is about EXECUTION (whether they run simultaneously). This distinction is a genuinely common interview question.

---

## 2. What is a Goroutine?

A **goroutine** is a lightweight, independently-executing function managed by the Go runtime. It's Go's core building block for concurrency.

**Analogy:** If a normal function call is like doing a task yourself, right now, and waiting for it to finish before moving on — a goroutine is like handing that task off to an assistant and saying "go do this, I won't wait around, I'll keep doing other things." The assistant works on it independently, in the background.

### Goroutines vs Traditional OS Threads (Major Interview Topic)

In many other languages, achieving concurrency means creating an **OS thread** (a unit of execution managed directly by your computer's operating system). Threads are powerful, but they're relatively "heavy" — each thread typically requires a good chunk of memory (often 1-2 MB) and has noticeable overhead for the OS to create and manage.

**Goroutines are much lighter.** They're managed by the Go RUNTIME (not directly by the OS), starting with a very small stack (often just a few KB), which can grow as needed. This means you can comfortably run THOUSANDS, even hundreds of thousands, of goroutines in a single program — something that would be impractical or impossible with traditional OS threads.

| Feature | OS Thread | Goroutine |
|---|---|---|
| Managed by | Operating System | Go runtime |
| Typical starting memory | ~1-2 MB | ~2 KB (grows as needed) |
| Creation cost | Relatively expensive | Very cheap |
| Typical count in a program | Dozens to hundreds | Thousands to millions |
| Switching between them | Handled by OS (heavier) | Handled by Go runtime (lighter, faster) |

**How does this work under the hood?** The Go runtime uses something called an **M:N scheduler** — it maps many goroutines (M) onto a smaller number of actual OS threads (N), intelligently switching goroutines in and out of those threads as needed. You, as the developer, don't need to manage this yourself — Go handles all the complex scheduling behind the scenes. This is precisely why Go earned its reputation for making concurrency dramatically simpler than languages that require you to manually manage threads.

### Visual: Goroutines vs Threads

```
   Traditional Threading Model:
   Thread 1 (heavy) ──┐
   Thread 2 (heavy) ──┼── managed directly by OS
   Thread 3 (heavy) ──┘   (expensive to create many)

   Go's Goroutine Model:
   Goroutine 1 ─┐
   Goroutine 2 ─┤
   Goroutine 3 ─┼──> Go Runtime Scheduler ──> maps onto a few OS Threads
   Goroutine 4 ─┤       (lightweight,             (efficiently shared)
   ... (1000s) ─┘        cheap to create)
```

---

## 3. Launching a Goroutine with the `go` Keyword

Starting a goroutine is remarkably simple — just put the keyword `go` before a function call.

```go
package main

import (
    "fmt"
    "time"
)

func sayHello() {
    fmt.Println("Hello from a goroutine!")
}

func main() {
    go sayHello() // launches sayHello as a goroutine, runs independently

    fmt.Println("Hello from main!")
    time.Sleep(1 * time.Second) // wait a bit, so the program doesn't exit too fast
}
```

Possible Output:
```
Hello from main!
Hello from a goroutine!
```

**Breakdown:**
- `go sayHello()` — this doesn't call `sayHello` and wait for it to finish, like a normal function call would. Instead, it says "start `sayHello` running INDEPENDENTLY, in the background, and immediately continue with the NEXT line of code without waiting."
- Because `main()` continues immediately after launching the goroutine, `"Hello from main!"` will typically print FIRST (since it doesn't have to wait), and `"Hello from a goroutine!"` prints whenever the Go scheduler gets around to actually running that goroutine — which could be a tiny fraction of a second later.
- `time.Sleep(1 * time.Second)` — pauses the `main` function for 1 second. This is here for an IMPORTANT reason explained next.

### The Critical Rule: When `main()` Ends, ALL Goroutines Die Immediately

This is one of the most important, most commonly misunderstood facts about goroutines, and a frequent source of beginner bugs (and interview questions).

```go
package main

import "fmt"

func sayHello() {
    fmt.Println("Hello from a goroutine!")
}

func main() {
    go sayHello()
    fmt.Println("Hello from main!")
    // main() ends here immediately!
}
```
With this version (no `time.Sleep`), there's a very good chance the output is JUST:
```
Hello from main!
```
The goroutine `sayHello()` might NEVER get a chance to print anything at all! **The moment `main()` finishes, the ENTIRE PROGRAM exits immediately — it does NOT wait for any goroutines still running in the background.** This is why we added `time.Sleep` above — but that's actually a bad, unreliable solution (how do you know exactly how long to wait?). The PROPER solution is **WaitGroups**, which we'll cover next.

**Analogy:** Think of `main()` finishing like the office building closing for the night and the lights turning off — any employees (goroutines) still working when that happens get sent home immediately, mid-task, whether they were done or not.

---

## 4. WaitGroups — Properly Waiting for Goroutines to Finish

A **WaitGroup** (from the `sync` package) is a counter-based tool that lets your main program WAIT until a specific number of goroutines have all finished their work, before continuing.

```go
package main

import (
    "fmt"
    "sync"
)

func sayHello(wg *sync.WaitGroup) {
    defer wg.Done() // signals "I'm finished" when this function exits
    fmt.Println("Hello from a goroutine!")
}

func main() {
    var wg sync.WaitGroup

    wg.Add(1) // "I'm about to launch 1 goroutine, please track it"
    go sayHello(&wg)

    wg.Wait() // "pause here until all tracked goroutines call Done()"
    fmt.Println("All goroutines finished!")
}
```
Output (now guaranteed, reliable order):
```
Hello from a goroutine!
All goroutines finished!
```

**Breakdown — this is the standard Go pattern you'll use CONSTANTLY with goroutines:**
- `var wg sync.WaitGroup` — creates a new WaitGroup, which internally keeps an integer counter (starting at 0).
- `wg.Add(1)` — increases the internal counter by 1. You call this BEFORE launching each goroutine, essentially saying "add one more task I need to wait for."
- `go sayHello(&wg)` — we pass a POINTER to the WaitGroup (`&wg`) into the goroutine, so it can properly signal back when it's done (remember pointers from Day 6 — this is a real, practical use case!).
- `defer wg.Done()` — inside `sayHello`, this decreases the counter by 1, and importantly, we use `defer` (from Day 8!) so this happens automatically right when the function finishes, no matter how it finishes.
- `wg.Wait()` — this BLOCKS (pauses) `main()`'s execution until the WaitGroup's internal counter reaches exactly 0 — meaning every goroutine that was `Add`ed has called `Done()`.

### Launching MULTIPLE Goroutines with a WaitGroup

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Printf("Worker %d is working\n", id)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go worker(i, &wg)
    }

    wg.Wait()
    fmt.Println("All workers finished!")
}
```
Possible Output (notice the ORDER is unpredictable!):
```
Worker 3 is working
Worker 1 is working
Worker 5 is working
Worker 2 is working
Worker 4 is working
All workers finished!
```

**Crucial interview point: the order in which goroutines actually run and print is NOT guaranteed or predictable.** The Go scheduler decides when each goroutine gets a chance to run, and this can vary between runs of the SAME program. `wg.Wait()` guarantees ALL of them finish before "All workers finished!" prints, but it says NOTHING about the ORDER they ran in relative to each other.

### Visual: WaitGroup Counter

```
   wg.Add(1)  →  counter: 1
   wg.Add(1)  →  counter: 2
   wg.Add(1)  →  counter: 3
                     │
        (goroutines run independently, in any order)
                     │
   worker finishes → wg.Done() → counter: 2
   worker finishes → wg.Done() → counter: 1
   worker finishes → wg.Done() → counter: 0
                     │
                     v
        wg.Wait() unblocks, since counter reached 0
```

---

## 5. Race Conditions — A Critical Danger with Goroutines

Since multiple goroutines can run concurrently and potentially access the SAME shared data, you can run into a **race condition** — a bug where the final result depends unpredictably on the exact timing/order in which goroutines happen to run, often producing incorrect or inconsistent results.

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup
    counter := 0

    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++ // DANGEROUS: multiple goroutines modifying the same variable at once!
        }()
    }

    wg.Wait()
    fmt.Println("Final counter:", counter) // often NOT 1000! Unpredictable.
}
```

**Why does this happen?** `counter++` looks like a single, simple operation, but internally it actually involves THREE steps: (1) read the current value of `counter`, (2) add 1 to it, (3) write the new value back. If TWO goroutines happen to read the SAME value at nearly the same time, both might add 1 to that same starting value and write back the same result — effectively losing one of the increments. With 1000 goroutines all racing to update the same variable, many of these increments can get "lost" this way, so the final count often ends up LESS than 1000, and the exact number varies unpredictably each time you run it.

**We'll cover the proper FIX for this (using Mutexes, and Channels) in more depth on Day 10** — for now, the key takeaway is: **be very careful when multiple goroutines access/modify the SAME shared variable — this is a classic source of bugs, and a favorite interview topic ("what is a race condition, and how would you detect/prevent one?").**

### Detecting Race Conditions: The `-race` Flag

Go actually has a built-in tool to help you catch these bugs during development:
```bash
go run -race main.go
```
This runs your program with a special "race detector" that monitors memory access patterns and warns you if it detects a genuine race condition — an extremely useful tool to know about and mention in interviews.

---

## 6. Anonymous Goroutines (Launching Inline Functions)

Just like we saw with anonymous functions on Day 3, you can launch a goroutine using an inline function directly, without giving it a separate name first:

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    var wg sync.WaitGroup

    wg.Add(1)
    go func() {
        defer wg.Done()
        fmt.Println("Running inside an anonymous goroutine!")
    }()

    wg.Wait()
}
```
This pattern — `go func() { ... }()` — is EXTREMELY common in real Go code, especially for quick, one-off concurrent tasks that don't need to be reused elsewhere as a named function.

**A subtle but important trap with loop variables (a classic Go gotcha, fixed in newer Go versions but still commonly asked about):**

```go
for i := 1; i <= 3; i++ {
    go func() {
        fmt.Println(i) // in older Go versions (before 1.22), this could print unexpected values!
    }()
}
```
In Go versions BEFORE 1.22, the loop variable `i` was SHARED across all goroutine closures, meaning by the time the goroutines actually ran, `i` might have already changed to its final value — leading to surprising, incorrect output (like printing `4, 4, 4` instead of `1, 2, 3`, or `3, 3, 3`). **Since Go 1.22, each loop iteration gets its OWN copy of the loop variable**, fixing this issue automatically. Still, it's a well-known historical gotcha that interviewers sometimes ask about, so it's worth knowing — and the old, safe workaround (passing the loop variable explicitly as a parameter) is still good practice to recognize:
```go
for i := 1; i <= 3; i++ {
    go func(n int) { // n is now a genuinely separate copy for each goroutine
        fmt.Println(n)
    }(i) // pass i in as an argument, capturing its CURRENT value at that point
}
```

---

## 7. Common Beginner Mistakes

1. **Forgetting that `main()` doesn't wait for goroutines** — the program can exit before background goroutines finish, unless you explicitly wait (with a WaitGroup, or other synchronization tools we'll see on Day 10).
2. **Using `time.Sleep` instead of a proper WaitGroup** to "wait" for goroutines — unreliable and not idiomatic; always prefer `sync.WaitGroup` (or channels) for genuine synchronization.
3. **Forgetting `wg.Add(1)` before launching a goroutine**, or calling it AFTER `go func...` instead of before — this can cause `wg.Wait()` to return too early, before all goroutines have actually started.
4. **Not passing the WaitGroup as a POINTER** (`*sync.WaitGroup`) — if you accidentally pass it by value (a copy), the goroutine's `Done()` call won't affect the original WaitGroup that `main()` is watching.
5. **Race conditions from multiple goroutines modifying shared data** without proper synchronization — always be cautious when goroutines touch the same variables.
6. **Expecting a predictable execution order among goroutines** — the Go scheduler doesn't guarantee any particular order; only `wg.Wait()` guarantees they've ALL finished by that point.

---

## 8. Day 9 Practice Exercise

Write a program that:
1. Defines a function `printSquare(n int, wg *sync.WaitGroup)` that prints the square of `n`, and properly signals `wg.Done()` when finished.
2. In `main()`, launches 5 goroutines (for numbers 1 through 5) calling `printSquare`, using a `sync.WaitGroup` to properly wait for all of them to finish before printing "All done!"
3. Run it a couple of times and observe that the ORDER of printed squares is not guaranteed to be the same each time.

<details>
<summary>Click to see solution</summary>

```go
package main

import (
    "fmt"
    "sync"
)

func printSquare(n int, wg *sync.WaitGroup) {
    defer wg.Done()
    fmt.Println(n, "squared is", n*n)
}

func main() {
    var wg sync.WaitGroup

    for i := 1; i <= 5; i++ {
        wg.Add(1)
        go printSquare(i, &wg)
    }

    wg.Wait()
    fmt.Println("All done!")
}
```
</details>

---

## 9. Day 9 Interview Questions (Quick Prep)

1. **Q: What is a goroutine, and how is it different from an OS thread?**
   A: A goroutine is a lightweight, independently-executing function managed by the Go runtime (not the OS). It starts with a very small stack (a few KB, growing as needed), making it far cheaper than an OS thread, so a program can run thousands or millions of goroutines simultaneously.

2. **Q: What's the difference between concurrency and parallelism?**
   A: Concurrency is about STRUCTURING a program to handle multiple tasks by interleaving them (not necessarily simultaneous). Parallelism is about actually EXECUTING multiple tasks at the exact same instant, usually needing multiple CPU cores.

3. **Q: What happens to running goroutines when `main()` finishes?**
   A: They are immediately terminated — Go does NOT wait for background goroutines to finish when `main()` returns.

4. **Q: What is a `sync.WaitGroup`, and how do you use it?**
   A: A counter-based synchronization tool. You call `Add(n)` before launching goroutines to track how many to wait for, each goroutine calls `Done()` when finished (often via `defer`), and `Wait()` blocks until the counter reaches zero.

5. **Q: What is a race condition? How can you detect one in Go?**
   A: A bug that occurs when multiple goroutines access/modify shared data concurrently without proper synchronization, causing unpredictable results. Go provides a built-in race detector via `go run -race main.go`.

6. **Q: Why must you pass a `*sync.WaitGroup` (pointer) rather than a plain `sync.WaitGroup` value into a goroutine?**
   A: Because Go is pass-by-value by default (Day 6) — passing by value would give the goroutine a separate COPY of the WaitGroup, so its `Done()` calls wouldn't affect the original one that `main()` is watching with `Wait()`.

---

## Summary of Day 9

- Concurrency is about structuring independent tasks; parallelism is about truly simultaneous execution — Go's model supports both, using the same code.
- A goroutine is a lightweight function managed by the Go runtime, launched with the `go` keyword; far cheaper than a traditional OS thread.
- `main()` does NOT wait for goroutines to finish when it exits — all goroutines are terminated immediately when `main()` returns.
- `sync.WaitGroup` (`Add`, `Done`, `Wait`) is the standard tool for properly waiting for a known number of goroutines to complete.
- Race conditions occur when multiple goroutines access shared data without synchronization — use `go run -race` to detect them.
- Goroutine execution order is never guaranteed; only synchronization tools like WaitGroups guarantee completion, not ordering.

**Tomorrow (Day 10):** We'll cover Channels — Go's built-in mechanism for goroutines to safely communicate and share data with each other, buffered vs unbuffered channels, the `select` statement, common concurrency patterns like worker pools, and how to avoid deadlocks.
