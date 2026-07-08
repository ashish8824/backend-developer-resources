# Day 15: Interview Prep (Final Day!)

> **Goal for today:** Consolidate everything from the past 14 days into interview-ready form. We'll cover the most commonly asked Go interview questions with model answers, classic coding problems, system design considerations where Go shines, a mock interview flow, and tips for explaining these concepts clearly when YOU teach others. Congratulations on making it to Day 15!

---

## 1. How to Use This Day

Don't just READ through this passively — for each question, try to answer it OUT LOUD or in writing FIRST, in your own words, before checking the model answer. This is exactly how real interviews work, and it's also excellent practice for your goal of teaching others (if you can explain it without notes, you truly understand it).

---

## 2. Top Go Interview Questions by Category

### Category A: Language Fundamentals

**Q1: What makes Go different from languages like Java or Python?**
> Go is a statically-typed, compiled language (Day 1) with built-in concurrency primitives (goroutines/channels, Days 9-10), no traditional class inheritance (uses composition/embedding instead, Day 5), explicit error handling instead of exceptions (Day 8), and a strong emphasis on simplicity — a small language spec, one loop keyword (Day 2), automatic code formatting (`gofmt`), and fast compilation.

**Q2: Explain the difference between `var` and `:=`.**
> `var` can declare variables anywhere (inside or outside functions) and optionally specify an explicit type. `:=` is shorthand, only usable inside functions, and infers the type from the assigned value. (Day 1)

**Q3: What is the zero value in Go? Give examples.**
> Every type has a default "zero value" assigned automatically when a variable is declared without an explicit value: `0` for numeric types, `""` for strings, `false` for booleans, `nil` for pointers/slices/maps/channels/interfaces/functions. (Days 1, 4, 6)

**Q4: Why does Go's `switch` not fall through by default, unlike C/Java?**
> It's a deliberate design choice to avoid a common source of bugs (forgetting `break` in other languages). Go automatically stops after a matching case; `fallthrough` is available if you explicitly need that behavior. (Day 2)

### Category B: Data Structures

**Q5: Explain the difference between arrays and slices, and slice length vs capacity.**
> Arrays have a fixed size that's part of their type; slices are dynamic, resizable views over an underlying array. A slice tracks a pointer to that array, a length (elements currently in use), and a capacity (total space before reallocation is needed). `append` may trigger a new, larger underlying array when capacity is exceeded. (Day 4)

**Q6: What happens if you modify a sub-slice? Does it affect the original?**
> Yes — a sub-slice shares the same underlying array as its parent, so modifying an element through one is visible through the other. Use `copy()` for a genuinely independent copy. (Day 4)

**Q7: How do you safely check if a key exists in a map?**
> The "comma ok" idiom: `value, exists := myMap[key]`. Directly accessing a missing key returns the zero value silently, without indicating whether it was truly present. (Day 4)

### Category C: Structs, Pointers, Interfaces

**Q8: Value receiver vs pointer receiver — when do you use each?**
> Value receivers operate on a COPY (changes don't persist to the original); pointer receivers operate on the ORIGINAL (changes DO persist). Use pointer receivers when a method needs to modify state, when the struct is large (avoiding copy overhead), or for consistency if other methods on the type already use pointers. (Day 5)

**Q9: What does `&` and `*` mean in Go?**
> `&` returns the memory address of a variable ("address of"). `*` either declares a pointer type (`*int`) or dereferences a pointer to access the value at that address, depending on context. (Day 6)

**Q10: How does Go achieve polymorphism without class inheritance?**
> Through interfaces (implicit implementation — any type with the required methods automatically satisfies an interface, no `implements` keyword) and struct embedding (composition, promoting fields/methods from an embedded type). (Days 5, 7)

**Q11: What's the difference between a plain type assertion and the "comma ok" form?**
> A plain assertion (`x.(Type)`) panics at runtime if wrong. The "comma ok" form (`v, ok := x.(Type)`) returns `ok = false` instead of panicking — always prefer this when uncertain of the underlying type. (Day 7)

### Category D: Error Handling

**Q12: Why doesn't Go have try/catch?**
> Go treats errors as ordinary return values, forcing explicit handling right where they occur, avoiding hidden control flow, and treating expected failures as a normal part of programming, not "exceptional" events requiring separate machinery. (Day 8)

**Q13: `panic`/`recover` vs regular `error` — when should each be used?**
> `error` is for expected, recoverable conditions (file not found, invalid input) and should be the default. `panic` is reserved for truly severe, unrecoverable situations (or historically, unrecoverable programmer bugs like nil pointer dereferences); `recover` (only usable inside a `defer`) can catch a panic to prevent a full crash, often used at a boundary (like a web server request handler) to contain failures. (Day 8)

**Q14: What's the difference between `errors.Is` and `errors.As`?**
> `errors.Is` checks if an error matches a specific error VALUE (seeing through wrapping via `%w`). `errors.As` checks if an error matches a specific error TYPE, extracting it to access its fields. (Day 8)

### Category E: Concurrency (Almost Always Asked!)

**Q15: What is a goroutine, and how does it differ from an OS thread?**
> A lightweight, independently-executing function managed by the Go runtime (not the OS), starting with a tiny stack (~2KB, growing as needed) — far cheaper than an OS thread (~1-2MB), allowing thousands to millions of goroutines in a single program. (Day 9)

**Q16: What happens to goroutines when `main()` returns?**
> They're terminated immediately — Go does NOT wait for background goroutines to finish. Use `sync.WaitGroup` or channels to properly synchronize. (Day 9)

**Q17: What is a race condition, and how do you detect one?**
> A bug from multiple goroutines accessing/modifying shared data concurrently without synchronization, causing unpredictable results. Detect with `go run -race`. (Day 9)

**Q18: Explain the difference between buffered and unbuffered channels.**
> Unbuffered channels require sender and receiver to be ready simultaneously (a blocking "handshake"). Buffered channels (`make(chan T, capacity)`) allow sends to proceed without an immediate receiver, up to the buffer's capacity, only blocking once full. (Day 10)

**Q19: What is a worker pool, and why use one?**
> A pattern where a fixed number of goroutines pull tasks from a shared channel, achieving controlled, bounded concurrency instead of launching unlimited goroutines that could overwhelm system resources. (Day 10)

**Q20: What causes a deadlock, and how does Go handle it?**
> Goroutines permanently stuck waiting on each other (e.g., sending on an unbuffered channel with no receiver). Go's runtime can detect certain simple deadlocks and crash with a clear error rather than hanging silently. (Day 10)

### Category F: Modern Go & Best Practices

**Q21: What problem do generics solve?**
> Writing reusable functions/types across multiple types with full compile-time type safety, avoiding both code duplication (separate functions per type) and the loss of type safety from using `interface{}`. (Day 14)

**Q22: What is the `context` package used for?**
> Propagating timeouts, deadlines, and cancellation signals across function calls and goroutines — essential in web servers/APIs to avoid wasting resources on work that's no longer needed. (Day 14)

**Q23: Does Go require manual memory management?**
> No — Go has automatic garbage collection; the compiler's escape analysis decides whether variables live on the stack or heap, without developer intervention. (Day 14)

**Q24: What is a table-driven test, and why is it idiomatic?**
> A pattern defining multiple test cases as a slice of structs, looped through with shared test logic (often via `t.Run` for named sub-tests) — avoids duplicating test code and produces clear, per-case output. (Day 12)

---

## 3. Common Coding Problems (Practice These!)

### Problem 1: FizzBuzz (warm-up, seen on Day 2)
Print numbers 1-100, but "Fizz" for multiples of 3, "Buzz" for multiples of 5, "FizzBuzz" for both.

### Problem 2: Reverse a String
```go
func reverseString(s string) string {
    runes := []rune(s)
    for i, j := 0, len(runes)-1; i < j; i, j = i+1, j-1 {
        runes[i], runes[j] = runes[j], runes[i]
    }
    return string(runes)
}
```

### Problem 3: Check for a Palindrome
```go
func isPalindrome(s string) bool {
    s = strings.ToLower(s)
    for i, j := 0, len(s)-1; i < j; i, j = i+1, j-1 {
        if s[i] != s[j] {
            return false
        }
    }
    return true
}
```

### Problem 4: Find Duplicates in a Slice (uses Maps, Day 4)
```go
func findDuplicates(nums []int) []int {
    seen := make(map[int]bool)
    var duplicates []int

    for _, n := range nums {
        if seen[n] {
            duplicates = append(duplicates, n)
        } else {
            seen[n] = true
        }
    }
    return duplicates
}
```

### Problem 5: Count Word Frequency (Maps + Strings, Days 4, 11)
```go
func wordFrequency(text string) map[string]int {
    words := strings.Fields(text) // splits by whitespace
    freq := make(map[string]int)

    for _, word := range words {
        word = strings.ToLower(word)
        freq[word]++
    }
    return freq
}
```

### Problem 6: Concurrent Sum Using Goroutines and Channels (Days 9-10)
Sum a large slice of numbers by splitting the work across multiple goroutines.
```go
func concurrentSum(numbers []int, numWorkers int) int {
    chunkSize := (len(numbers) + numWorkers - 1) / numWorkers
    results := make(chan int, numWorkers)
    var wg sync.WaitGroup

    for i := 0; i < len(numbers); i += chunkSize {
        end := i + chunkSize
        if end > len(numbers) {
            end = len(numbers)
        }

        wg.Add(1)
        go func(chunk []int) {
            defer wg.Done()
            sum := 0
            for _, n := range chunk {
                sum += n
            }
            results <- sum
        }(numbers[i:end])
    }

    go func() {
        wg.Wait()
        close(results)
    }()

    total := 0
    for partial := range results {
        total += partial
    }
    return total
}
```
This is a genuinely GREAT problem to be able to explain confidently — it combines slicing (Day 4), goroutines/WaitGroups (Day 9), and channels (Day 10) into a realistic concurrent pattern.

### Problem 7: Implement a Simple LRU-style Cache Check (Struct + Map combo, Days 4-5)
A simplified version — check if you can explain how you'd track "least recently used" conceptually, even if you don't write the full eviction logic live. This tests structural thinking, common in mid-to-senior interviews.

---

## 4. System Design Questions Where Go Shines

Interviewers sometimes ask higher-level design questions to see if you understand WHY you'd reach for Go for a particular problem. Practice explaining your reasoning for these:

**"Design a rate limiter for an API."**
> Discuss using channels or a buffered channel as a "token bucket," goroutines to refill tokens periodically, and `context` for handling request timeouts.

**"Design a system to process a large batch of files concurrently, but not overwhelm the system."**
> This is a direct application of the worker pool pattern (Day 10) — a fixed number of goroutines pulling file paths from a channel, bounding concurrency.

**"How would you handle a slow downstream API call in your Go service, so it doesn't hang your server?"**
> Discuss `context.WithTimeout` (Day 14) to bound how long you'll wait, and proper error handling if it times out.

**"Why might you choose Go over Python/Java for a high-throughput microservice?"**
> Discuss goroutines being far cheaper than threads for high-concurrency workloads (Day 9), Go's fast compilation and startup time, and static binaries (no separate runtime/VM needed for deployment) making it simple to containerize (relevant to why Docker/Kubernetes themselves are written in Go).

---

## 5. Mock Interview — Try Answering These Yourself First!

Cover the answers below and genuinely attempt each one, ideally speaking your answer out loud as if in a real interview.

1. Walk me through what happens when you run `go run main.go`.
2. What's the output of this code, and why?
   ```go
   func main() {
       defer fmt.Println("1")
       defer fmt.Println("2")
       fmt.Println("3")
   }
   ```
   <details><summary>Answer</summary>Prints "3", then "2", then "1" — defers run in LIFO order, after the surrounding function's other code completes. (Day 8)</details>

3. What's wrong with this code?
   ```go
   func modify(s []int) {
       s = append(s, 100)
   }
   func main() {
       nums := []int{1, 2, 3}
       modify(nums)
       fmt.Println(nums)
   }
   ```
   <details><summary>Answer</summary>Prints `[1 2 3]`, NOT `[1 2 3 100]` — since `append` may allocate a new underlying array, the change isn't visible back in `main()`. The function would need to return the modified slice, and the caller would need to reassign it. (Days 4, 6)</details>

4. How would you make this code thread-safe?
   ```go
   var counter int
   func increment() {
       counter++
   }
   ```
   <details><summary>Answer</summary>Use a `sync.Mutex` to lock around the increment, or better, use channels to coordinate access, following Go's "share memory by communicating" philosophy. (Days 9, 10)</details>

5. Explain, in your own words, why "accept interfaces, return structs" is good advice.
   <details><summary>Answer</summary>Accepting interfaces makes your functions flexible and testable (can substitute mocks, Day 12) and decoupled from specific implementations. Returning concrete structs gives callers clarity about exactly what they're getting, avoiding ambiguity. (Day 7)</details>

---

## 6. Tips for Explaining Go Concepts to Others (Since You Want to Teach!)

Since your original goal was to learn Go well enough to TEACH it, here are practical tips based on how this course was structured:

1. **Always start with WHY, not just HOW.** Before showing syntax, explain the PROBLEM it solves (e.g., "we need multiple return values because Go doesn't have exceptions, so errors need to travel alongside results").

2. **Use analogies for abstract concepts.** Pointers as "addresses," interfaces as "job descriptions," channels as "pneumatic tubes" — concrete, physical comparisons make abstract computer science concepts click much faster for beginners.

3. **Show broken code, not just working code.** Some of the most valuable teaching moments in this course were showing what goes WRONG (nil pointer dereference, race conditions, append not persisting) — understanding failure modes deepens understanding far more than only seeing success cases.

4. **Connect new concepts back to old ones explicitly.** Notice how, throughout this course, we kept saying "remember the comma-ok pattern from Day 4/7/10" or "this uses defer, from Day 8" — pointing out these RECURRING patterns helps learners see Go as a coherent, consistent language, not a pile of disconnected syntax rules.

5. **Have learners predict output BEFORE running code.** Asking "what do you think this prints, and why?" (like the defer LIFO example above) engages active thinking rather than passive reading.

6. **Encourage building small real things early.** A REST API (Day 13), even a tiny one, is far more motivating and memorable than only ever seeing isolated snippets.

7. **Be honest about genuinely confusing parts.** Things like `&`/`*`, slice capacity growth, or closures ARE tricky for virtually everyone at first — normalizing that struggle (rather than implying it should be obvious) builds learner confidence rather than discouragement.

---

## 7. Final Review Checklist — Can You Confidently Explain All of These?

Go through this list and rate yourself honestly (✅ confident / ⚠️ shaky / ❌ need to review):

- [ ] Compiled vs interpreted languages, `go run` vs `go build`
- [ ] `var` vs `:=`, zero values
- [ ] `if/else`, `switch` (no fallthrough), the one `for` loop
- [ ] Functions: multiple returns, named returns, variadic, closures
- [ ] Arrays vs slices, length vs capacity, sub-slice sharing
- [ ] Maps and the "comma ok" idiom
- [ ] Structs, methods, value vs pointer receivers, embedding
- [ ] Pointers: `&`, `*`, pass-by-value vs pass-by-reference
- [ ] Interfaces: implicit implementation, empty interface, type assertion/switch
- [ ] Errors: `(result, error)` pattern, custom errors, `errors.Is`/`As`, `panic`/`recover`/`defer`
- [ ] Goroutines, WaitGroups, race conditions
- [ ] Channels, buffered vs unbuffered, `select`, worker pools, deadlocks
- [ ] Packages, exported vs unexported, `go.mod`/`go.sum`
- [ ] Testing: table-driven tests, benchmarks, mocking via interfaces
- [ ] JSON encoding/decoding, struct tags, building a REST API with `net/http`
- [ ] Generics, `context`, garbage collection basics

If most of these are ✅, you're genuinely well-prepared for a Go interview at a solid junior-to-mid level, and you have a strong foundation to teach others confidently. Anything ⚠️ or ❌ — go back to that specific day's file and re-read the relevant section; you now have all 15 days saved as reference material.

---

## 8. What's Next After This 15-Day Course?

To keep growing beyond this roadmap:
- **Build a real project** — a CLI tool, a small REST API with a database, or a concurrent web scraper. Nothing cements knowledge like building something real end-to-end.
- **Read Go's official documentation and the "Effective Go" guide** — Go's own team maintains excellent, concise style guidance.
- **Explore popular Go frameworks/libraries** if relevant to your goals (e.g., `gin` or `echo` for web frameworks, `gorm` for databases) — though everything you learned here is framework-agnostic, standard-library-first knowledge, which is exactly the right foundation.
- **Practice explaining these concepts out loud to a friend, or write your own blog posts/notes** — since teaching is genuinely one of the best ways to deepen your own understanding, and it directly supports your original goal.

---

## Congratulations!

You've completed a full 15-day journey through Go — from `fmt.Println("Hello, Go!")` on Day 1 all the way to generics, context, and full interview readiness on Day 15. You now have:
- 15 detailed reference documents covering every core Go concept
- Practice exercises with solutions for each topic
- A full bank of interview questions and model answers
- Coding problems commonly asked in real interviews
- Teaching tips to help you pass this knowledge on to others

Good luck with your interviews — and enjoy teaching Go to others! 🎉
