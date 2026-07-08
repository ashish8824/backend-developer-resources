# Day 10: Channels (Concurrency Part 2)

> **Goal for today:** Learn Channels — Go's built-in mechanism for goroutines to safely communicate and share data with each other. We'll cover buffered vs unbuffered channels, the `select` statement, common concurrency patterns (worker pools, fan-in/fan-out), and how to avoid deadlocks. This completes Go's concurrency story, alongside yesterday's goroutines.

---

## 1. Why Do We Need Channels? (Concept First)

On Day 9, we saw that when multiple goroutines access the SAME shared variable (like our `counter++` example), we can get **race conditions** — unpredictable, incorrect results.

Go has a famous design philosophy for solving this problem, expressed as a proverb:

> *"Do not communicate by sharing memory; instead, share memory by communicating."*

This means: instead of having multiple goroutines directly poke at the SAME shared variable (which is risky and requires careful locking), Go encourages goroutines to safely PASS data between each other through a dedicated communication pipe — a **channel**. Only one goroutine touches the data at any given moment, safely passed along like a baton in a relay race.

**Analogy:** Imagine two people trying to share a single notebook, both scribbling in it at the same time — chaotic, and things get overwritten or corrupted (this is like the race condition from Day 9). Now imagine instead they use a **pneumatic tube system** (like the ones banks used to send documents between counters) — one person puts a note in the tube, and it safely arrives at the other person, who takes it out. Only one person touches the note at any given moment. This tube is a **channel**.

```
Goroutine A                    Channel                    Goroutine B
┌───────────┐              ┌───────────┐              ┌───────────┐
│  sends      │ ─────────> │   pipe/tube   │ ─────────> │  receives   │
│  data        │              └───────────┘              │  data        │
└───────────┘                                            └───────────┘
```

---

## 2. Creating and Using a Channel

### Declaring a Channel

```go
ch := make(chan int) // creates a channel that can carry int values
```
`chan int` means "a channel that transports `int` values." Just like slices and maps, channels are created using the `make()` function.

### Sending and Receiving

```go
package main

import "fmt"

func main() {
    ch := make(chan string)

    go func() {
        ch <- "Hello from the goroutine!" // SEND a value into the channel
    }()

    message := <-ch // RECEIVE a value from the channel
    fmt.Println(message)
}
```
Output:
```
Hello from the goroutine!
```

**Breakdown — the key new syntax is the arrow `<-`:**
- `ch <- "Hello..."` — the arrow points INTO the channel, meaning "send this value into the channel." Read it as: "send this value TO the channel."
- `message := <-ch` — the arrow points AWAY from the channel, meaning "receive a value FROM the channel." Read it as: "take a value FROM the channel."

**Memory trick:** The arrow `<-` always shows the DIRECTION data is flowing. `ch <- value` (arrow going INTO `ch`) = sending. `value := <-ch` (arrow coming OUT of `ch`) = receiving.

---

## 3. Unbuffered Channels — Synchronization Built In

The channel we just created (`make(chan string)`) is called an **unbuffered channel**. This has a crucial property: **a send operation BLOCKS (pauses) until another goroutine is ready to receive it, and vice versa.** The sender and receiver must "meet" at the same moment for the data to pass through — this is called a **synchronous** handoff.

**Why did our example above work without explicitly using a WaitGroup?** Because the unbuffered channel itself provided the synchronization! `main()`'s line `message := <-ch` BLOCKS, waiting patiently until the goroutine actually sends something. This naturally solves the same "main() doesn't wait for goroutines" problem from Day 9, but through data communication instead of an explicit WaitGroup.

### Visual: Unbuffered Channel (Rendezvous / Handshake)

```
   Goroutine (sender)              main() (receiver)
   ┌─────────────┐                 ┌─────────────┐
   │ ch <- "Hi"    │  ---- BLOCKS ----> waiting...   │
   │  (waits here)  │                 │  <-ch          │
   └─────────────┘                 └─────────────┘
        Both sides "meet" at the same moment — data passes through,
        then BOTH sides are free to continue.
```

---

## 4. Buffered Channels — Some Breathing Room

A **buffered channel** has a fixed CAPACITY — it can hold a certain number of values WITHOUT requiring an immediate receiver. Only once the buffer is FULL does a send operation block.

```go
ch := make(chan int, 3) // buffered channel with capacity 3

ch <- 1 // doesn't block — buffer has room
ch <- 2 // doesn't block — buffer has room
ch <- 3 // doesn't block — buffer is now FULL (capacity reached)

fmt.Println(<-ch) // 1  (receiving frees up space)
fmt.Println(<-ch) // 2
fmt.Println(<-ch) // 3
```

**Breakdown:**
- `make(chan int, 3)` — the second argument (`3`) specifies the buffer capacity, similar syntax to `make([]int, len, cap)` from Day 4.
- The first 3 sends succeed immediately without blocking, because there's room in the buffer.
- If we tried a 4th send (`ch <- 4`) WITHOUT anything receiving first, THAT send would block, since the buffer is full.

### Unbuffered vs Buffered — Quick Comparison Table

| Feature | Unbuffered Channel | Buffered Channel |
|---|---|---|
| Created with | `make(chan int)` | `make(chan int, capacity)` |
| Send blocks until... | A receiver is ready RIGHT NOW | The buffer becomes full |
| Guarantees synchronization? | Yes — sender and receiver "meet" | No — sender can "drop off" and move on, as long as there's room |
| Use case | Strict coordination between goroutines | Decoupling sender/receiver timing, allowing some flexibility |

**Interview tip:** A common question is *"what's the difference between buffered and unbuffered channels?"* — the key answer is about WHEN the send operation blocks: unbuffered blocks until someone's actively receiving; buffered only blocks once the buffer is completely full.

---

## 5. Closing a Channel

When a sender is completely done sending values, it can `close()` the channel to signal "no more data is coming."

```go
package main

import "fmt"

func main() {
    ch := make(chan int, 3)

    ch <- 1
    ch <- 2
    ch <- 3
    close(ch) // no more values will be sent

    for value := range ch {
        fmt.Println(value)
    }
}
```
Output:
```
1
2
3
```

**Breakdown:**
- `close(ch)` — marks the channel as closed. Importantly, you can STILL receive any remaining buffered values after closing — closing just means no MORE values can be SENT.
- `for value := range ch` — a channel can be used with `range` (just like slices/maps from Day 4)! This loop automatically receives values one by one until the channel is BOTH empty AND closed, at which point the loop ends automatically.

### Checking if a Channel is Closed — The "Comma OK" Idiom Again!

Just like we saw with maps (Day 4) and type assertions (Day 7), channels use the SAME "comma ok" pattern to check if a channel is closed:

```go
value, ok := <-ch
if !ok {
    fmt.Println("Channel is closed, no more values")
} else {
    fmt.Println("Received:", value)
}
```
`ok` will be `false` once the channel is closed AND drained (empty) — this is how you can distinguish "I received a genuine zero value" from "the channel is actually closed."

**Important rule (common interview trap):** Only the SENDER should close a channel, never the receiver. Also, sending on an ALREADY-closed channel causes a PANIC. Closing an already-closed channel also panics. Channels don't need to always be explicitly closed — it's only necessary when a receiver genuinely needs to know "no more values are coming" (like with `range`).

---

## 6. The `select` Statement

`select` lets a goroutine wait on MULTIPLE channel operations at once, proceeding with whichever one becomes ready FIRST. Think of it like a `switch` statement (Day 2), but for channels instead of values.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    ch1 := make(chan string)
    ch2 := make(chan string)

    go func() {
        time.Sleep(1 * time.Second)
        ch1 <- "Message from ch1"
    }()

    go func() {
        time.Sleep(2 * time.Second)
        ch2 <- "Message from ch2"
    }()

    for i := 0; i < 2; i++ {
        select {
        case msg1 := <-ch1:
            fmt.Println(msg1)
        case msg2 := <-ch2:
            fmt.Println(msg2)
        }
    }
}
```
Output:
```
Message from ch1
Message from ch2
```

**Breakdown:**
- `select { case ... case ... }` — Go checks all the listed channel operations, and whichever one is ready FIRST gets executed. Here, since `ch1` receives its message after only 1 second (faster than `ch2`'s 2 seconds), `case msg1 := <-ch1` fires first.
- The loop runs twice, so both messages eventually get printed, in the order they actually became ready — `select` naturally lets you react to whichever event happens first, without wasting time manually checking one channel, then the other, in sequence.

### `select` with a `default` case (Non-blocking Channel Operations)

```go
select {
case msg := <-ch:
    fmt.Println("Received:", msg)
default:
    fmt.Println("No message ready, moving on")
}
```
If NO channel is ready immediately, `default` runs instead of blocking/waiting — useful when you want to "check" a channel without getting stuck waiting for it.

---

## 7. Common Concurrency Pattern: Worker Pools

A **worker pool** is a very common, practical pattern: you have a fixed number of "worker" goroutines that pull tasks from a shared channel and process them, allowing you to control how much concurrent work happens at once (rather than launching an unbounded, potentially huge number of goroutines all at once).

```go
package main

import (
    "fmt"
    "sync"
)

func worker(id int, jobs <-chan int, results chan<- int, wg *sync.WaitGroup) {
    defer wg.Done()
    for job := range jobs {
        fmt.Printf("Worker %d processing job %d\n", id, job)
        results <- job * 2 // simulate some work: doubling the number
    }
}

func main() {
    jobs := make(chan int, 5)
    results := make(chan int, 5)
    var wg sync.WaitGroup

    // Launch 3 worker goroutines
    for w := 1; w <= 3; w++ {
        wg.Add(1)
        go worker(w, jobs, results, &wg)
    }

    // Send 5 jobs into the jobs channel
    for j := 1; j <= 5; j++ {
        jobs <- j
    }
    close(jobs) // signal: no more jobs coming

    wg.Wait()      // wait for all workers to finish
    close(results) // now safe to close results, since no more will be sent

    for result := range results {
        fmt.Println("Result:", result)
    }
}
```

**Breakdown of new concepts here:**
- `jobs <-chan int` — this is a **directional channel type**. `<-chan int` means "a channel that can ONLY be received from" (read-only, from this function's perspective). This is a nice safety feature — the `worker` function explicitly can't accidentally SEND into `jobs`, only receive from it.
- `results chan<- int` — similarly, `chan<- int` means "a channel that can ONLY be sent to" (write-only, from this function's perspective).
- `for job := range jobs` — the worker keeps pulling jobs from the channel until `jobs` is closed AND drained.
- We launch 3 workers, but send 5 jobs — so workers naturally pick up new jobs as they finish previous ones, achieving controlled, bounded concurrency (only 3 things happening at once, no matter how many total jobs there are).

**Why is this pattern so useful in real-world code?** Imagine processing thousands of image uploads, or handling thousands of API requests. Launching thousands of unrestrained goroutines all at once could overwhelm your system's memory or an external resource (like a database). A worker pool lets you cap the concurrency to a sensible, controlled number (like 3, 10, or 100 workers), while still processing everything eventually.

### Visual: Worker Pool

```
   jobs channel:  [1][2][3][4][5]
                       │
         ┌────────────┼────────────┐
         v             v             v
     Worker 1      Worker 2      Worker 3
    (picks jobs   (picks jobs   (picks jobs
     as available)  as available)  as available)
         │             │             │
         └────────────┼────────────┘
                       v
              results channel
```

---

## 8. Deadlocks — A Common Danger

A **deadlock** happens when goroutines get stuck waiting on each other FOREVER, with no way to proceed. Go's runtime is actually smart enough to DETECT certain simple deadlocks and crash the program with a clear error, rather than hanging forever silently.

### Classic Deadlock Example

```go
package main

func main() {
    ch := make(chan int) // unbuffered
    ch <- 1 // BLOCKS forever — nobody is receiving, and we're the only goroutine!
}
```
Output:
```
fatal error: all goroutines are asleep - deadlock!
```

**Why does this deadlock?** `ch` is unbuffered, so `ch <- 1` needs a receiver ready at the SAME moment. But there's no other goroutine running to receive it — `main()` itself is the only goroutine, and it's stuck waiting on this very line. Nothing can ever unblock it, so Go detects this and crashes with a helpful error rather than hanging silently forever.

### Common Causes of Deadlocks

1. **Sending on an unbuffered channel with no goroutine ready to receive.**
2. **Forgetting to close a channel** that a `range` loop is waiting on — the loop will wait forever for more values or a close signal that never comes.
3. **A WaitGroup counter that never reaches zero** — e.g., calling `wg.Add(2)` but only having one goroutine call `wg.Done()`.
4. **Two goroutines each waiting on a channel the OTHER one is supposed to send to first** — a circular waiting dependency.

**Practical debugging tip:** If your Go program just "hangs" forever with no output and no crash, suspect a deadlock or a goroutine stuck waiting on a channel/WaitGroup that will never be satisfied. Carefully trace through: is every channel eventually closed or received from? Does every `wg.Add()` have a matching `wg.Done()`?

---

## 9. Channels vs WaitGroups vs Mutexes — When to Use What

Now that you know goroutines (Day 9), WaitGroups (Day 9), and channels (today), here's practical, interview-friendly guidance on choosing between them:

| Tool | Best used for... |
|---|---|
| `sync.WaitGroup` | Simply waiting for a known number of goroutines to FINISH, when you don't need them to exchange data with each other. |
| Channels | Goroutines need to SEND/SHARE actual data with each other, or coordinate based on events/timing (like `select`). |
| `sync.Mutex` (briefly: a "mutual exclusion lock" that only lets ONE goroutine access a shared variable at a time) | When you specifically need direct, low-level protection of a shared variable and channels feel like overkill for a very simple case. (We won't go deep into Mutex today, but it's worth knowing it exists as Go's more "traditional" locking tool, an alternative to the channel-based "share memory by communicating" approach.) |

---

## 10. Common Beginner Mistakes

1. **Forgetting the direction of the arrow (`<-`)** — `ch <- value` sends, `value := <-ch` receives. Easy to mix up when first learning.
2. **Sending on an unbuffered channel with no active receiver** — causes the sender to block forever (deadlock), unless something else is set up to receive.
3. **Closing a channel more than once, or sending on a closed channel** — both cause a runtime panic.
4. **Forgetting to close a channel that a `range` loop depends on** — the loop will hang forever waiting for a close signal.
5. **Using an unbuffered channel when a buffered one would avoid unnecessary blocking** (or vice versa) — think carefully about whether you truly need strict synchronization (unbuffered) or just some flexible slack (buffered).
6. **Launching unlimited goroutines for unlimited work** without a worker-pool-style limit — can overwhelm system resources; consider capping concurrency for large workloads.

---

## 11. Day 10 Practice Exercise

Write a program that:
1. Creates a buffered channel `numbers` with capacity 5.
2. Launches a goroutine that sends the numbers 1 through 5 into the channel, then closes it.
3. In `main()`, uses a `for range` loop to receive and print each number as it comes in.
4. BONUS: Implement a simple worker pool with 2 workers that each square the numbers received (instead of just printing them), sending results to a `results` channel, then print all the squared results.

<details>
<summary>Click to see solution</summary>

```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    numbers := make(chan int, 5)

    go func() {
        for i := 1; i <= 5; i++ {
            numbers <- i
        }
        close(numbers)
    }()

    results := make(chan int, 5)
    var wg sync.WaitGroup

    // Worker pool with 2 workers
    for w := 1; w <= 2; w++ {
        wg.Add(1)
        go func(id int) {
            defer wg.Done()
            for n := range numbers {
                fmt.Printf("Worker %d squaring %d\n", id, n)
                results <- n * n
            }
        }(w)
    }

    wg.Wait()
    close(results)

    for r := range results {
        fmt.Println("Squared result:", r)
    }
}
```
</details>

---

## 12. Day 10 Interview Questions (Quick Prep)

1. **Q: What is a channel, and why does Go use it for concurrency?**
   A: A channel is a typed pipe for sending and receiving values between goroutines, allowing safe communication without directly sharing memory — following Go's philosophy of "share memory by communicating."

2. **Q: What's the difference between a buffered and an unbuffered channel?**
   A: An unbuffered channel requires a sender and receiver to be ready at the same moment (blocks until both meet). A buffered channel allows sends to proceed without an immediate receiver, up to its fixed capacity, only blocking once full.

3. **Q: What does the `select` statement do?**
   A: It lets a goroutine wait on multiple channel operations simultaneously, proceeding with whichever becomes ready first; a `default` case makes it non-blocking.

4. **Q: What causes a deadlock in Go, and how does Go handle it?**
   A: A deadlock happens when goroutines are permanently stuck waiting on each other (e.g., sending on an unbuffered channel with no receiver). Go's runtime can detect certain simple deadlocks and crash with a clear "all goroutines are asleep - deadlock!" error.

5. **Q: What is a worker pool, and why would you use one?**
   A: A pattern where a fixed number of goroutines pull tasks from a shared channel, allowing controlled, bounded concurrency instead of launching unlimited unrestrained goroutines.

6. **Q: What happens if you send a value on an already-closed channel?**
   A: The program panics at runtime.

---

## Summary of Day 10

- Channels let goroutines communicate safely, following Go's "share memory by communicating" philosophy — avoiding the race conditions from Day 9.
- `ch <- value` sends; `value := <-ch` receives — the arrow direction indicates data flow.
- Unbuffered channels block until sender and receiver are both ready (a synchronous handshake); buffered channels (`make(chan T, capacity)`) allow some slack before blocking.
- `close(ch)` signals no more values will be sent; `range` over a channel automatically stops once it's closed and drained; the "comma ok" idiom (`v, ok := <-ch`) checks if a channel is closed.
- `select` lets a goroutine react to whichever of multiple channel operations becomes ready first; `default` makes it non-blocking.
- Worker pools use a fixed number of goroutines pulling from a shared channel to achieve controlled, bounded concurrency — a very common real-world pattern.
- Deadlocks occur when goroutines wait on each other forever; Go can detect and report simple cases automatically.

**Tomorrow (Day 11):** We'll cover Packages & Modules — how to properly organize Go code into packages, understanding `go.mod`/`go.sum`, exported vs unexported identifiers (the capital letter rule), and a tour of essential standard library packages (`fmt`, `strings`, `strconv`, `time`, `os`).
