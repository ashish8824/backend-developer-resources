# Module 6: How Redis Works Internally

**Level:** Intermediate

This module answers the question interviewers love to ask: **"Why is Redis so fast, and how does it actually work under the hood?"**

---

## 1. Redis is Single-Threaded (For Commands)

This surprises a lot of people. Unlike Node.js's event loop handling many things concurrently through async I/O, or a database using multiple threads to handle many queries in parallel, **Redis processes one command at a time**, on a single thread, for the actual command execution.

Sounds like it would be *slow*, right? It's actually the opposite, and here's why:

- **No lock contention.** Multiple threads modifying the same data need locks to avoid corrupting it, and managing those locks has real overhead (waiting, context-switching). Single-threaded execution means zero lock overhead — a command either fully completes or hasn't started yet.
- **Every command is simple and fast.** Most Redis operations are O(1) or O(log N) — they finish in microseconds, so there's very little to "wait" for.
- **No context-switching costs.** Switching between threads has CPU overhead. A single thread avoids this entirely.

```
Traditional multi-threaded DB:
Thread 1 ──lock──▶ modify data ──unlock──▶
Thread 2   (waits for lock)  ──lock──▶ modify data ──unlock──▶
Thread 3   (waits...)                    ──lock──▶ ...

Redis (single-threaded command execution):
Command 1 ──▶ done (instant, no locking needed)
Command 2 ──▶ done
Command 3 ──▶ done
```

> Note: Modern Redis *does* use background threads for some things like slow I/O operations and lazy deletion of large keys — but the core command execution (reading/writing your data) is single-threaded.

---

## 2. The Event Loop (Sound Familiar?)

If you're a Node.js developer, this next part will feel very familiar, because **Redis uses an event loop, just like Node.js!**

```
   ┌────────────────────────────┐
   │        Event Loop          │
   │                             │
   │  1. Accept new connections  │
   │  2. Read incoming commands  │
   │  3. Execute command         │
   │  4. Send response           │
   │  5. Repeat                  │
   └────────────────────────────┘
```

Redis uses a mechanism (based on `epoll`/`kqueue`, similar to what Node's `libuv` uses) to efficiently watch thousands of client connections **without needing a thread per connection**. This is exactly why both Node.js and Redis can handle huge numbers of concurrent connections with modest hardware — they share the same core philosophy: **don't block, don't waste threads waiting around.**

This is also *why Redis and Node.js pair so well together* — they think about concurrency the same way.

---

## 3. What Happens When You Run a Command?

Let's trace through `SET name "Ashish"` step by step:

```
1. Client (your Node.js app) sends: SET name "Ashish"
        │
2. Redis's event loop picks up the incoming data on the socket
        │
3. Redis parses the command (using the RESP protocol — see below)
        │
4. The single command-processing thread executes it:
       - Look up (or create) the key "name" in Redis's internal hash table
       - Store the value "Ashish"
        │
5. Redis sends back: +OK
        │
6. Your Node.js client receives "OK" and your `await redis.set(...)` resolves
```

This entire round trip usually takes well under a millisecond on a local network.

---

## 4. RESP: The Protocol Redis Speaks

RESP (**RE**dis **S**erialization **P**rotocol) is the simple text-based protocol Redis and its clients use to talk to each other. It's intentionally simple — easy to parse extremely fast, which is part of why Redis has so little overhead per command.

You'll never write RESP by hand (`ioredis` handles this for you), but it helps to see what's actually flowing over the network:

```
Client sends:      *3\r\n$3\r\nSET\r\n$4\r\nname\r\n$6\r\nAshish\r\n
                    (this just means: "an array of 3 items: SET, name, Ashish")

Server responds:    +OK\r\n
                    (this just means: "a simple string: OK")
```

---

## 5. In-Memory Data Structures

Internally, Redis stores your keys in a giant **hash table** (similar conceptually to a JavaScript `Map`), which is what gives `GET`/`SET` their O(1) speed — Redis can jump directly to a key's memory location instead of scanning through data.

For the more advanced types:

- **Lists** are implemented using efficient linked-list-like structures (`quicklist`) so pushing/popping from either end is fast.
- **Hashes** use compact, memory-efficient encodings when small, and switch to full hash tables when they grow large.
- **Sorted Sets** use a **skip list** — a clever structure that allows fast (O(log N)) insertion, deletion, and range queries while keeping elements sorted.

You don't need to memorize these internals, but knowing that Redis picks smart, purpose-built structures per data type (rather than one generic structure for everything) is a great interview talking point.

---

## 6. Why "In-Memory" Doesn't Mean "Data Is Always Lost"

A common misconception: "If Redis stores everything in RAM, doesn't all my data disappear if the server restarts?"

**Not necessarily** — Redis has two optional persistence mechanisms (RDB snapshots and AOF logs) that write data to disk in the background, so it can be reloaded after a restart. We'll cover this in full detail in Module 8.

---

## 7. Node.js Event Loop vs Redis Event Loop — A Nice Parallel

| | Node.js | Redis |
|---|---|---|
| Concurrency model | Single-threaded event loop | Single-threaded event loop (for commands) |
| Handles many connections? | Yes, via non-blocking I/O | Yes, via non-blocking I/O |
| Heavy CPU work | Can block the event loop (bad!) | Commands are designed to be fast (O(1)/O(log N)) to avoid blocking |
| Background work | Worker threads / libuv thread pool | Background threads for things like slow deletes, persistence |

This is exactly why a slow Redis command (like a giant `KEYS *` on millions of keys, from Module 5) is dangerous — it blocks the *single* thread, and every other client waiting on Redis has to wait too, just like a blocking `for` loop would freeze Node's event loop.

```js
// ❌ In Node.js: a long synchronous loop blocks EVERYONE
for (let i = 0; i < 10_000_000_000; i++) {} // freezes your whole server

// ❌ In Redis: a huge KEYS * command blocks EVERYONE the same way
// KEYS *   (on a database with millions of keys)
```

---

## 8. Interview Question: "Since Redis is single-threaded, how does it handle thousands of clients at once?"

**Answer:** Redis uses a non-blocking, event-driven I/O model (similar to Node.js's event loop). It doesn't dedicate a thread per client connection — instead, it uses OS-level mechanisms like `epoll` to efficiently monitor many sockets at once, only doing work when there's actual data to process. Combined with the fact that individual commands are extremely fast (mostly O(1) or O(log N)), a single thread can serve an enormous number of requests per second.

---

## 9. Summary

- Redis's core command execution is single-threaded — this avoids locking overhead and keeps things simple and predictable.
- It uses an event-loop model very similar to Node.js, which is part of why they pair so naturally together.
- Commands communicate over the lightweight RESP protocol.
- Internally, Redis uses specialized, efficient data structures per type (hash tables, quicklists, skip lists).
- A single slow command can block all other clients — always avoid heavy, unbounded operations like `KEYS *` in production.

---

## 10. Homework

1. In your own words, explain why single-threaded Redis can still be extremely fast.
2. Compare Node.js's event loop to Redis's event loop — what's similar, what's different?
3. Why is running `KEYS *` on a large production database dangerous?

---

**Next up:** [Module 7 — Memory Management](./07-memory-management.md)
