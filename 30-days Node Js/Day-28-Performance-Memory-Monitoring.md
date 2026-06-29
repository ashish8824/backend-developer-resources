# Day 28 — Performance: Memory Leaks, Profiling, Garbage Collection, Monitoring

> **Why this day matters:** This closes out Week 4's technical content with the topic that most reliably separates "has shipped features" from "has been on-call and debugged a real production incident." Memory leaks and performance degradation are exactly the kind of problems that don't show up in code review or local testing — they show up at 2am, three days after a deploy, when memory usage has slowly crept up and the process finally crashes. Today builds the diagnostic instincts for that scenario.

---

## 1. Background: a practical mental model of memory in Node

Recall Day 5's mention that Buffers live OUTSIDE the V8 heap. The fuller picture: a Node process's memory is roughly divided into:

- **The V8 heap** — where your JavaScript objects, strings, closures, etc. live. This is what **garbage collection** manages.
- **Off-heap memory** — Buffers, things managed directly by Node/C++ bindings, not subject to V8's garbage collector in the same way.
- **The call stack** — local to currently executing function calls (Day 2).

**What garbage collection actually does, at a practical level (not a deep V8 internals dive, but enough to reason about real bugs):** V8 periodically scans the heap, identifies which objects are still **reachable** (referenced, directly or indirectly, from somewhere your code can still access — global variables, active closures, objects still referenced by something alive), and **frees the memory of anything that's NOT reachable anymore**. The key insight: **a memory leak in JavaScript isn't about forgetting to "free" memory manually (there's no `malloc`/`free` to forget) — it's about accidentally keeping a reference to something alive longer than you intended**, preventing the garbage collector from ever being able to reclaim it.

---

## 2. Common Memory Leak Patterns in Node.js Apps — recognize these by name

### Pattern 1: Forgotten EventEmitter listeners (Day 4 callback, directly relevant again)

```javascript
// THE LEAK: every time this function runs, it adds ANOTHER listener to
// 'emitter', and NONE of them are ever removed. Over the life of a
// long-running server, if this function is called repeatedly (e.g.,
// once per incoming request, by mistake), listeners accumulate
// forever, each one holding references to whatever it closed over.
function handleRequest(req, res) {
  emitter.on('data', (data) => {
    res.json(data); // this closure holds a reference to 'res' for THIS
                      // specific request, but the LISTENER itself never
                      // gets removed, so 'res' (and everything it
                      // references) can never be garbage collected,
                      // even long after this request has finished
  });
}

// THE FIX: either register listeners ONCE outside the per-request
// function entirely, or explicitly remove a listener once it's no
// longer needed
function handleRequestFixed(req, res) {
  const listener = (data) => res.json(data);
  emitter.once('data', listener); // .once() auto-removes itself after firing
}
```

### Pattern 2: Unbounded caches/maps

```javascript
// THE LEAK: a cache that grows forever, with no eviction policy at all --
// every unique key ever seen stays in memory PERMANENTLY, for the
// entire life of the process.
const cache = new Map();

function getCachedData(key, fetchFn) {
  if (cache.has(key)) return cache.get(key);
  const data = fetchFn();
  cache.set(key, data); // nothing EVER removes old entries
  return data;
}

// THE FIX: bound the cache size (an LRU -- Least Recently Used --
// eviction strategy is the standard approach), or better yet for most
// real backend use cases, use Redis (Day 17) with a TTL instead of an
// in-memory structure, since Redis-based expiration solves this
// entirely AND solves the Day 15 multi-instance sharing problem at the
// same time.
```

### Pattern 3: Closures unintentionally capturing large objects

```javascript
// THE LEAK: this function captures the ENTIRE 'largeDataset' in its
// closure, even though it only actually needs ONE specific value from
// it. If this returned function is stored somewhere long-lived (e.g.,
// registered as a callback that persists for the app's lifetime), the
// WHOLE largeDataset stays alive in memory too, even though only one
// small piece of it is actually used.
function createHandler(largeDataset) {
  const specificValue = largeDataset.importantField;

  return function handler() {
    console.log(specificValue);
    // 'largeDataset' is STILL reachable here too, even though we only
    // use 'specificValue' -- JS closures capture by REFERENCE to the
    // enclosing scope, not just the specific variables you end up using
  };
}

// THE FIX: extract ONLY what you need before creating the closure,
// and let the large object itself become unreachable/eligible for GC
function createHandlerFixed(largeDataset) {
  const specificValue = largeDataset.importantField;
  largeDataset = null; // explicitly drop the reference (or simply let
                         // it go out of scope naturally if structured
                         // so the closure never captures it)
  return function handler() {
    console.log(specificValue);
  };
}
```

### Pattern 4: Timers that are never cleared

```javascript
// THE LEAK: every call to this function starts a NEW interval that
// runs FOREVER, and nothing ever stops the previous ones -- a classic
// source of slowly accumulating background work and memory
function startPolling() {
  setInterval(() => {
    checkForUpdates(); // this callback, and anything it references, stays
                         // alive for as long as this interval keeps running
  }, 5000);
}

// THE FIX: always keep a reference to the timer ID, and explicitly
// clear it when it's no longer needed (e.g., on cleanup, on a
// connection close, on graceful shutdown -- Day 27's topic!)
function startPollingFixed() {
  const intervalId = setInterval(() => checkForUpdates(), 5000);
  return () => clearInterval(intervalId); // returns a cleanup function
}
```

---

## 3. Profiling Tools — how to actually FIND a leak, not just recognize patterns in theory

### `process.memoryUsage()` — the simplest first check

```javascript
// A quick, built-in way to log memory usage over time -- often the
// FIRST thing to add when investigating a suspected leak, since it
// requires no extra tooling at all
setInterval(() => {
  const usage = process.memoryUsage();
  console.log({
    rss: `${Math.round(usage.rss / 1024 / 1024)} MB`,         // total memory allocated to the process
    heapTotal: `${Math.round(usage.heapTotal / 1024 / 1024)} MB`, // total size of the V8 heap
    heapUsed: `${Math.round(usage.heapUsed / 1024 / 1024)} MB`,   // actually USED heap memory -- the most useful number to watch
    external: `${Math.round(usage.external / 1024 / 1024)} MB`,  // memory used by C++ objects bound to JS (e.g., Buffers)
  });
}, 5000);
```

**What to actually look for:** if `heapUsed` keeps climbing steadily over time, even during periods of STABLE, repeated traffic (not increasing load), and never comes back down even after garbage collection would be expected to run — that's the classic signature of a real memory leak, as opposed to normal memory fluctuation under varying load.

### `--inspect` and Chrome DevTools — for a deeper, visual investigation

```bash
node --inspect app.js
# Then open chrome://inspect in Chrome, and connect to your running
# Node process -- this gives you access to a full debugger AND a
# memory profiler with the SAME interface used for debugging
# client-side JavaScript in the browser
```

**What you can do once connected:** take **heap snapshots** at two different points in time (e.g., before and after running a suspicious operation many times), and compare them — DevTools will show you exactly WHAT KINDS of objects grew in number/size between the two snapshots, which is often enough to point directly at the leaking code (e.g., "10,000 more `EventEmitter` listener objects exist in snapshot 2 than snapshot 1" — pointing straight at Pattern 1 above).

### clinic.js — a more guided, Node-specific profiling toolkit

```bash
npm install -g clinic

# clinic doctor runs your app under various scenarios and produces a
# visual report flagging WHICH KIND of performance issue you likely
# have (I/O bound, CPU bound, event-loop blocking, memory issue) --
# useful as a STARTING point when you don't yet know what's wrong
clinic doctor -- node app.js

# clinic flame produces a FLAME GRAPH -- a visual representation of
# where your CPU time is actually being spent, function by function --
# extremely useful for finding a SPECIFIC slow/blocking function
# (connects directly back to Day 2's event-loop-blocking concern)
clinic flame -- node app.js
```

**Interview tip:** mentioning clinic.js or Chrome DevTools heap snapshots BY NAME when asked "how would you debug a memory leak in production" signals real, hands-on debugging experience — far stronger than a vague "I'd look at memory usage" answer.

---

## 4. Garbage Collection — a practical, not academic, understanding

### The two GC strategies worth knowing the NAMES of (without needing deep internals)

**Background:** V8's garbage collector treats objects differently based on how long they've survived, because most objects in a typical program are short-lived (created, used briefly, then discarded), while a smaller number live much longer.

- **Minor GC (Scavenge)** — runs frequently, on the "young generation" (recently created objects) — fast, since most short-lived objects are cleaned up here quickly.
- **Major GC (Mark-Sweep / Mark-Compact)** — runs less frequently, on the "old generation" (objects that have survived multiple minor GC cycles) — slower, more thorough, and historically could even cause noticeable pauses ("stop-the-world" pauses) in older V8 versions, though modern V8 has made major strides in making this more incremental/concurrent to reduce pause times.

**Why this matters practically, even without deep internals knowledge:** if an object is SHORT-LIVED (created and discarded quickly), it's cheap for V8 to handle. If you accidentally keep objects alive for a LONG time unnecessarily (any of the leak patterns above), you're forcing more objects into the "old generation," which is more expensive to garbage collect — meaning leaks don't just consume more memory, they can also make garbage collection ITSELF slower and more disruptive over time.

### Manually triggering GC (rarely needed, but worth knowing it exists)

```bash
node --expose-gc app.js
```
```javascript
if (global.gc) {
  global.gc(); // manually triggers garbage collection -- almost NEVER
                 // appropriate in actual production code, but genuinely
                 // useful when manually profiling/debugging a leak
                 // (e.g., forcing a GC right before taking a heap
                 // snapshot, to get a clean baseline)
}
```

---

## 5. What to Actually Monitor in Production — beyond just "is it up"

### Background: connecting back to Day 23's liveness/readiness distinction

A health check tells you the app is RUNNING. It does NOT tell you whether the app is slowly degrading, leaking memory, or about to hit a resource limit. Real production monitoring goes further:

```javascript
// A more informative health/metrics endpoint, beyond Day 23's basic version
app.get('/metrics', (req, res) => {
  const usage = process.memoryUsage();
  res.json({
    uptime: process.uptime(),
    memory: {
      heapUsedMB: Math.round(usage.heapUsed / 1024 / 1024),
      rssMB: Math.round(usage.rss / 1024 / 1024),
    },
    // Event loop lag -- a DIRECT, practical measurement connecting back
    // to Day 2's entire event loop lesson: if something is blocking the
    // event loop, this number will visibly increase, even before users
    // start reporting slowness
    eventLoopDelay: getEventLoopDelay(),
  });
});

// A simple way to measure event loop lag: schedule a callback for
// "right now" via setImmediate, and measure how long it ACTUALLY took
// to fire -- if the event loop is busy/blocked, this measured delay
// will be noticeably higher than expected
function getEventLoopDelay() {
  const start = process.hrtime.bigint();
  return new Promise((resolve) => {
    setImmediate(() => {
      const delayNs = process.hrtime.bigint() - start;
      resolve(Number(delayNs) / 1e6); // convert to milliseconds
    });
  });
}
```

**Key production metrics worth naming in an interview, beyond raw memory:**
- **Request latency (p50/p95/p99)** — average latency hides outliers; the 95th/99th percentile reveals how bad the WORST experienced requests are, which often matters more for user experience than the average.
- **Error rate** — the percentage of requests resulting in 4xx/5xx (Day 9), tracked over time, ideally broken down by endpoint.
- **Event loop lag** — as shown above, a direct signal of event-loop-blocking issues (Day 2).
- **CPU and memory usage trends over time** — not just a snapshot, but the TREND, since a slow memory leak is invisible in a single point-in-time check but obvious on a graph over hours/days.
- **Active database connections / connection pool saturation** — a common, real bottleneck that's easy to overlook (Day 11's connection management).

**Real tools used in production for this (worth name-dropping if relevant):** Datadog, New Relic, Prometheus + Grafana, and Sentry specifically for error tracking/alerting (tying back to Day 13's error handling — these tools are often where your `isOperational` vs. unexpected-bug distinction actually gets USED, surfacing unexpected errors to the team automatically).

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Your production server's memory usage climbs steadily over several days and eventually crashes — walk me through how you'd investigate."** → Start with `process.memoryUsage()` logging/monitoring to confirm the trend, then use heap snapshots (Chrome DevTools via `--inspect`) at two points in time to identify WHAT kind of object is accumulating, then map that back to a likely pattern (forgotten listeners, unbounded cache, uncleaned timers).
- **"How would you tell the difference between 'the app needs more memory because of legitimately higher traffic' and 'the app has a real memory leak'?"** → A leak shows STEADILY INCREASING memory even during STABLE or repeated load (or even after traffic drops back down) and never returns to a baseline; legitimate load-driven memory use should roughly track traffic and come back down as load decreases.
- **"What would you add to a health-check/metrics endpoint beyond just 'returns 200 OK'?"** → Memory usage, event loop lag, uptime, and ideally connectivity checks to critical dependencies (Day 23's readiness check concept) — turning a binary up/down signal into something that reveals gradual degradation before a full outage occurs.
- **"Why might major garbage collection pauses become MORE frequent or longer over the life of a process with a memory leak?"** → Leaked objects accumulate in the "old generation," which major GC has to scan and clean more thoroughly — a growing amount of long-lived (leaked) data makes each major GC cycle more expensive over time, compounding the original problem.

---

## 7. Hands-on practice for today

```javascript
// practice-day28-leak-demo.js -- a DELIBERATE, observable memory leak,
// for the purpose of watching it happen and confirming you can spot it
const express = require('express');
const app = express();

const leakyArray = []; // intentionally never cleared -- the leak itself

app.get('/leak', (req, res) => {
  // Each request adds a sizable chunk of data that's NEVER removed --
  // simulating Pattern 2 (unbounded cache) from section 2
  leakyArray.push(new Array(100000).fill('leaked-data'));
  res.json({ totalLeakedChunks: leakyArray.length });
});

app.get('/memory', (req, res) => {
  const usage = process.memoryUsage();
  res.json({ heapUsedMB: Math.round(usage.heapUsed / 1024 / 1024) });
});

app.listen(3000, () => console.log('Running on http://localhost:3000'));
```

Run it, then hit `/leak` repeatedly (`for i in {1..50}; do curl http://localhost:3000/leak; done`), checking `/memory` before and after — confirm `heapUsedMB` climbs steadily and doesn't come back down. Then, as a deliberate exercise, run the SAME app with `node --inspect practice-day28-leak-demo.js`, connect via `chrome://inspect`, take a heap snapshot, run the leak-triggering loop again, take a SECOND snapshot, and use DevTools' comparison view to see `leakyArray`'s growth directly — turning today's theory into a real, hands-on diagnostic exercise.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What is a "memory leak" in JavaScript, precisely, given there's no manual memory management?**
   A: It's not about forgetting to free memory — it's about accidentally keeping a reference to an object alive (via a forgotten listener, an unbounded cache, a closure, or an uncleared timer) longer than intended, preventing the garbage collector from ever reclaiming memory that's no longer actually needed.

2. **Q: Name two common causes of memory leaks in a long-running Node.js server.**
   A: Forgotten EventEmitter listeners that accumulate without being removed, and unbounded in-memory caches/Maps that grow forever with no eviction policy.

3. **Q: How would you confirm a suspected memory leak, beyond just guessing?**
   A: Monitor `process.memoryUsage().heapUsed` over time under stable/repeated load — a real leak shows a steady upward trend that never returns to baseline, distinct from normal memory fluctuation that tracks actual traffic levels.

4. **Q: How do heap snapshots help you find the SOURCE of a leak, not just confirm one exists?**
   A: Taking two snapshots at different points in time (e.g., before and after triggering suspicious behavior repeatedly) and comparing them in Chrome DevTools reveals exactly which types of objects grew in count/size between the two, often pointing directly at the leaking code pattern.

5. **Q: What's the practical difference between minor and major garbage collection?**
   A: Minor GC runs frequently and cheaply on recently created, typically short-lived objects. Major GC runs less often but more thoroughly on longer-lived objects that have survived multiple minor GC cycles — leaked objects end up here, making major GC progressively more expensive as a leak grows.

6. **Q: Beyond uptime, what metrics would you want visibility into for a production Node.js API?**
   A: Request latency percentiles (especially p95/p99, not just average), error rate by endpoint, event loop lag (a direct signal of event-loop-blocking issues), memory usage trends over time, and database connection pool saturation.

---

## Day 29-30 Preview
Week 4 is complete — you've now covered rate limiting algorithms, load balancing, multi-layer caching, microservices, Docker, CI/CD, and performance/monitoring, on top of Weeks 1-3's fundamentals through advanced Node internals. The final two days shift entirely into **interview execution**: Day 29 covers system design question frameworks (how to structure an answer to "design a URL shortener," "design a rate limiter," "design a notification system") plus a curated set of the highest-frequency Node-specific interview questions across all 28 days, and Day 30 is a full mock interview combining a live coding round with behavioral/experience-based questions, plus a final targeted review of whichever areas still feel shaky.
