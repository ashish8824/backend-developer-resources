# Day 2 — The Event Loop (Deep Dive)

> **Why this day matters more than any other day in this course:**
> If you have 3 years of experience and you can't explain the event loop clearly, an interviewer will doubt everything else you said. This is the #1 most-asked Node.js conceptual question at the 2-3 YOE level, asked in almost every interview, sometimes 2-3 different ways in the same interview to see if you really understand it or just memorized a definition. So today we go SLOW and DEEP.

---

## 1. First, the background you need before "event loop" makes sense

Before learning the event loop, you need to understand the **problem it's solving**. Let's build it from scratch with a story.

### The old way: blocking, synchronous servers

Imagine a single waiter in a restaurant. A blocking model works like this:
- Waiter takes Table 1's order
- Waiter walks to kitchen, **stands there and waits** until the food is fully cooked
- Waiter brings food to Table 1
- ONLY THEN does the waiter go to Table 2

If the kitchen takes 10 minutes to cook, every other table starves for 10 minutes, even though the waiter is doing nothing during that time except standing around. This is what a **blocking I/O** server does. In languages/runtimes that use one thread per request and block on I/O, this either wastes huge amounts of thread/memory (if you spawn a thread per request) or creates this exact starvation problem (if single-threaded and blocking).

### The Node.js way: non-blocking, event-driven

Now imagine the same waiter, but smarter:
- Waiter takes Table 1's order, hands it to the kitchen, and **immediately walks away** to take Table 2's order — doesn't wait around
- Waiter takes Table 2's order, hands it to kitchen, walks to Table 3
- ...this repeats for all tables...
- Whenever the kitchen **finishes** an order, it rings a bell. The waiter, when free, checks "did any bell ring?" and delivers that food.

This is Node.js. **One thread (the waiter)** handles many concurrent requests by **never blocking** — it delegates the slow work (the "cooking" — disk reads, network calls, DB queries) elsewhere and just keeps checking back for completions. The "kitchen" here is the **OS kernel and libuv's thread pool**, and the "bell + the waiter checking for it" is literally **the event loop**.

This is why Node.js is described as:
> **Single-threaded, non-blocking, event-driven, asynchronous I/O model.**

Every one of those four words now means something concrete to you, not just jargon.

---

## 2. The actual pieces involved (so you're not confusing terms)

| Term | What it actually is |
|---|---|
| **Call stack** | Where your currently-running JS function calls live. Single-threaded — one stack, one thing executing at a time. |
| **Heap** | Where objects/variables are stored in memory. |
| **libuv** | A C library Node uses under the hood. Provides the event loop itself, plus a thread pool for certain blocking work. |
| **Thread pool** | A small set of background OS threads (default **4**, configurable via `UV_THREADPOOL_SIZE`) that libuv uses for things like file system operations, DNS lookups (`dns.lookup`), and some crypto functions. |
| **Event loop** | A loop, running continuously in the main thread, that checks: "is the call stack empty? if yes, is there a completed callback waiting? if yes, push it onto the call stack and run it." |
| **Callback queue / task queues** | Where completed async operations' callback functions wait until the event loop picks them up. |

**Critical clarification interviewers want to hear:** the event loop itself does NOT execute your async I/O — the **OS kernel** (for networking, mostly via epoll/kqueue/IOCP depending on OS) or **libuv's thread pool** (for fs, DNS, crypto) does the actual work. The event loop's only job is **coordination** — checking when things are done and scheduling your callback to run on the main thread.

---

## 3. The Event Loop Phases (this is the part to memorize and be able to draw)

The event loop isn't one queue — it's **six phases**, run in a fixed order, in a continuous cycle:

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval callbacks whose time has elapsed
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks      │  I/O callbacks deferred from previous cycle (some system errors)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare        │  internal use only, rarely relevant to app code
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │            poll            │  retrieves new I/O events (fs reads, network data); executes their callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │            check           │  setImmediate() callbacks run here, right after poll phase
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       close callbacks      │  e.g. socket.on('close', ...)
│  └─────────────┬─────────────┘
└────────────────┘
```

You don't need to memorize all six phase names perfectly for most interviews — but you **do** need to deeply understand 3 of them, because they're the ones that show up in actual code: **timers**, **poll**, and **check**. And you need to understand 2 special queues that are NOT part of these 6 phases at all (this trips up almost everyone): **microtasks** and **`process.nextTick`**.

---

## 4. Microtasks vs Macrotasks — the part everyone gets wrong

This is the single most misunderstood part of the event loop, so let's be very precise.

### Macrotasks (a.k.a. "the 6 phases above")
- `setTimeout`, `setInterval` → timers phase
- I/O callbacks (`fs.readFile`, network requests) → poll phase
- `setImmediate` → check phase

### Microtasks (a special, higher-priority queue)
- **Promise callbacks** (`.then()`, `.catch()`, `.finally()`, and code after `await`)
- `queueMicrotask()`

### `process.nextTick` queue (Node-specific, even HIGHER priority than microtasks)
- This is **not technically part of the event loop at all** — it runs at the end of the **current operation**, before the event loop continues to the next phase.

### THE RULE (memorize this exact order):

> After **every single callback finishes executing** (not after every phase — after every individual callback), Node:
> 1. First drains the entire `process.nextTick` queue
> 2. Then drains the entire microtask queue (Promises)
> 3. THEN moves on to the next callback in the macrotask queue / next phase

This happens **constantly**, between every callback — not just once per loop cycle.

### Let's trace actual code, line by line, like an interviewer will ask you to

```javascript
console.log('1: start'); // synchronous, runs immediately

setTimeout(() => {
  console.log('2: setTimeout (macrotask - timers phase)');
}, 0);

setImmediate(() => {
  console.log('3: setImmediate (macrotask - check phase)');
});

Promise.resolve().then(() => {
  console.log('4: promise.then (microtask)');
});

process.nextTick(() => {
  console.log('5: process.nextTick (highest priority queue)');
});

console.log('6: end'); // synchronous, runs immediately
```

**Output (in this exact order):**
```
1: start
6: end
5: process.nextTick (highest priority queue)
4: promise.then (microtask)
2: setTimeout (macrotask - timers phase)
3: setImmediate (macrotask - check phase)
```

**Why, step by step:**
1. `console.log('1: start')` and `console.log('6: end')` are **synchronous** — they run immediately as the script is read top to bottom, before anything async gets a chance. This is why "1" and "6" print before everything else even though "6" appears after the async calls in the source code.
2. Once the synchronous code (the "main script") finishes, the call stack is empty. Before touching the event loop's phases at all, Node **drains `process.nextTick` queue completely** → prints "5".
3. Then Node **drains the microtask (Promise) queue completely** → prints "4".
4. NOW the event loop actually starts cycling through phases. It enters the **timers phase** first → the `setTimeout(..., 0)` callback fires → prints "2".
5. It continues to **poll phase** (nothing here in this example), then **check phase** → `setImmediate` callback fires → prints "3".

**Important nuance interviewers love to probe:** `setTimeout(fn, 0)` vs `setImmediate(fn)` — which runs first is **NOT guaranteed** if called from the **main module** (top-level code), because it depends on process performance and timing precision. But if both are called **inside an I/O callback** (e.g., inside `fs.readFile`'s callback), `setImmediate` is **always guaranteed to run before** `setTimeout(fn, 0)`. This is because after an I/O callback finishes (poll phase), the very next phase is check (`setImmediate`'s home), and the loop has to go all the way back around to timers for `setTimeout`.

```javascript
const fs = require('fs');

fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);
  setImmediate(() => console.log('immediate'));
});

// Guaranteed output here:
// immediate
// timeout
```

This exact snippet is a classic interview "predict the output" question. Practice explaining WHY, not just memorizing the output.

---

## 5. Why does `process.nextTick` exist, and why is it dangerous?

**What it's for:** to let you defer a callback to run right after the current operation, before I/O events or timers get a chance — useful for things like ensuring an error event is emitted after the current synchronous code finishes (so listeners have a chance to attach first), or for letting a function always behave asynchronously even in edge cases (consistency).

**Why it's dangerous:** because `process.nextTick` callbacks are drained **completely** before the event loop moves to the next phase, if you keep calling `process.nextTick` from within a `process.nextTick` callback recursively, you create an **infinite loop that starves the event loop** — I/O, timers, everything else, never gets a chance to run. This is called **"I/O starvation"** and is a real production bug pattern interviewers ask about.

```javascript
// DANGER: this will hang your entire Node process — I/O never runs
function recursive() {
  process.nextTick(recursive);
}
recursive();
```

---

## 6. How this connects to real backend work (3 YOE framing)

When an interviewer at the 3-year level asks about the event loop, they aren't just testing trivia — they're checking if you understand **why certain code patterns hurt production performance**. Be ready to connect event loop knowledge to real scenarios:

- **"Why did our API's response times spike under load?"** → Likely a synchronous, CPU-heavy operation (e.g., a large JSON.parse, a synchronous bcrypt hash, a heavy loop) blocked the event loop, so every other concurrent request had to wait — even ones unrelated to that operation.
- **"Why use `bcrypt.hash()` async version instead of `bcryptjs` sync version in a high-traffic API?"** → The sync version blocks the single thread; under concurrent traffic, every request queues up behind it.
- **"What does `Promise.all` actually let us do in terms of the event loop?"** → It lets multiple async operations (e.g., 3 independent DB queries) get kicked off without waiting for each one sequentially — they all get dispatched, and the event loop resumes your code only once all promises settle, instead of blocking serially.

---

## 7. Hands-on practice for today

Run this and predict the output BEFORE running it. Then check yourself.

```javascript
// event-loop-practice.js

console.log('A');

setTimeout(() => console.log('B'), 0);

Promise.resolve()
  .then(() => console.log('C'))
  .then(() => console.log('D'));

process.nextTick(() => console.log('E'));

setImmediate(() => console.log('F'));

console.log('G');

// Try to predict the order: A, G print first (sync).
// Then nextTick queue: E.
// Then microtask queue fully drains: C, then D (because D is chained after C, 
//   it gets added to the microtask queue only after C resolves, but it's still 
//   drained in the SAME microtask-draining phase since the queue keeps getting 
//   checked until empty)
// Then macrotasks: B (timers phase), then F (check phase) — order between B and F
//   is not strictly guaranteed at top level, but typically follows timer/check order.
```

Run it for real:
```bash
node event-loop-practice.js
```

Then modify it: wrap the `setTimeout` and `setImmediate` inside an `fs.readFile` callback (like the snippet in section 4) and observe how the guaranteed ordering changes.

---

## 8. Interview Q&A Recap (say these out loud, don't just read them)

1. **Q: Explain the event loop in your own words.**
   A: Node runs JS on a single thread. When it hits an async operation (file read, network call, timer), it hands that work off to the OS or libuv's thread pool instead of waiting, and keeps executing other code. The event loop is the mechanism that continuously checks, in a fixed sequence of phases (timers → pending callbacks → poll → check → close), whether any of that delegated work has completed, and if so, schedules its callback to run on the main thread once the call stack is empty.

2. **Q: What's the difference between microtasks and macrotasks?**
   A: Macrotasks are the standard phase-based callbacks — timers, I/O, setImmediate. Microtasks are Promise callbacks (`.then`), and they have higher priority — the entire microtask queue is drained after every single callback, before the event loop proceeds to the next macrotask.

3. **Q: Where does `process.nextTick` fit in?**
   A: It's even higher priority than microtasks — it's not technically part of the event loop phases at all. It runs immediately after the current operation finishes, before microtasks and before the loop proceeds. Overusing it recursively can starve the event loop entirely.

4. **Q: setTimeout(fn, 0) vs setImmediate(fn) — which runs first?**
   A: At the top level / main module, it's not strictly guaranteed (timing dependent). But inside an I/O callback, `setImmediate` is always guaranteed to run first, because the poll phase (where I/O callbacks run) is immediately followed by the check phase (setImmediate's home), whereas timers requires the loop to cycle all the way back around.

5. **Q: Give a real production example of "blocking the event loop" and how you'd fix it.**
   A: A synchronous CPU-heavy task (e.g., parsing a huge CSV synchronously, a large synchronous crypto hash, an unoptimized recursive function) running inside a request handler blocks ALL other requests on that single thread. Fix: offload to a Worker Thread or a separate microservice/child process, or use the async version of the library if available, or move heavy work to a background job queue (Redis + BullMQ, covered in Week 3) and respond to the client immediately.

---

## Tomorrow (Day 3) Preview
We move to **async patterns in practice**: callbacks → "callback hell" → Promises → `async/await`, plus **proper error handling patterns** in async code (try/catch with await, `.catch()` chains, unhandled promise rejections, and why uncaught errors can crash a Node process). This builds directly on today's mental model.
