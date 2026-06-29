# Node.js Event Loop & Non-Blocking I/O

## What is the Node.js Event Loop?

The Event Loop is the mechanism that allows Node.js to perform non-blocking I/O operations despite JavaScript being single-threaded. It continuously checks for pending tasks, I/O operations, and timers — processing them one at a time.

```
Node.js is single-threaded — only ONE thing runs at a time in JS.
But it can handle thousands of concurrent connections because:
  - I/O (network, disk) is handed off to the OS
  - While OS handles I/O, Node.js processes other requests
  - When I/O completes, callback is queued and executed
```

## Blocking vs Non-Blocking I/O

### Blocking (Synchronous) — Traditional

```
Thread 1: Request 1 -> Read file -> WAITING... -> Process -> Respond
Thread 2: Request 2 -> Read DB   -> WAITING... -> Process -> Respond
Thread 3: Request 3 -> Read file -> WAITING... -> Process -> Respond

1000 concurrent requests = 1000 threads (expensive: 1-8MB RAM each = 1-8GB RAM)
```

### Non-Blocking (Asynchronous) — Node.js

```
Single Thread: 
  Request 1 arrives -> initiate file read -> (OS handles it, Node.js moves on)
  Request 2 arrives -> initiate DB query  -> (OS handles it, Node.js moves on)
  Request 3 arrives -> initiate file read -> (OS handles it, Node.js moves on)
  ...
  File read complete (callback) -> process Request 1 -> respond
  DB query complete (callback)  -> process Request 2 -> respond

1000 concurrent requests = 1 thread + 1000 OS-level async operations
```

## Event Loop Architecture

```
┌─────────────────────────────────────────┐
│              Node.js Process            │
│                                         │
│   ┌─────────────────────────────────┐   │
│   │         Event Loop              │   │
│   │  ┌──────────────────────────┐   │   │
│   │  │ 1. timers               │   │   │
│   │  │    (setTimeout,          │   │   │
│   │  │     setInterval)         │   │   │
│   │  ├──────────────────────────┤   │   │
│   │  │ 2. pending callbacks    │   │   │
│   │  │    (I/O errors)         │   │   │
│   │  ├──────────────────────────┤   │   │
│   │  │ 3. idle, prepare        │   │   │
│   │  ├──────────────────────────┤   │   │
│   │  │ 4. poll                 │   │   │
│   │  │    (new I/O events)     │   │   │
│   │  ├──────────────────────────┤   │   │
│   │  │ 5. check                │   │   │
│   │  │    (setImmediate)        │   │   │
│   │  ├──────────────────────────┤   │   │
│   │  │ 6. close callbacks      │   │   │
│   │  └──────────────────────────┘   │   │
│   └─────────────────────────────────┘   │
│                   │                     │
│     ┌─────────────▼───────────────┐     │
│     │    libuv Thread Pool        │     │
│     │  (file I/O, DNS, crypto)    │     │
│     │    4 threads (default)      │     │
│     └─────────────────────────────┘     │
└─────────────────────────────────────────┘
```

## Event Loop Phases (In Detail)

### Phase 1: Timers

Executes callbacks scheduled by `setTimeout()` and `setInterval()` whose delay has elapsed.

```javascript
setTimeout(() => console.log('Timer 1'), 0);
setTimeout(() => console.log('Timer 2'), 0);
// Both execute in Timers phase
```

### Phase 2: Pending Callbacks

Executes I/O callbacks that were deferred from the previous loop iteration (e.g. TCP errors).

### Phase 3: Idle, Prepare

Internal use only.

### Phase 4: Poll (Most Time Spent Here)

- Executes I/O callbacks (file reads, network responses, DB queries)
- Blocks here waiting for new events if there are no timers or setImmediate callbacks pending

```javascript
fs.readFile('/etc/hosts', (err, data) => {
    console.log('File read complete');  // Executes in Poll phase
});
```

### Phase 5: Check

Executes `setImmediate()` callbacks.

```javascript
setImmediate(() => console.log('setImmediate'));
// Executes in Check phase — AFTER current I/O callbacks
```

### Phase 6: Close Callbacks

Executes close event callbacks (e.g. `socket.on('close', ...)`).

## Microtasks vs Macrotasks

Between each phase (and after each macrotask callback), Node.js processes the microtask queue completely before moving to the next phase.

**Microtasks (highest priority):**
- `process.nextTick()` — executes before any I/O callbacks, even before `Promise.then`
- `Promise.then()` / `async/await`

**Macrotasks (lower priority):**
- `setTimeout()`, `setInterval()`
- `setImmediate()`
- I/O callbacks

```javascript
console.log('1: Start');

setTimeout(() => console.log('4: setTimeout'), 0);

setImmediate(() => console.log('5: setImmediate'));

Promise.resolve()
    .then(() => console.log('3: Promise.then'));

process.nextTick(() => console.log('2: nextTick'));

console.log('6: End');

// Output order:
// 1: Start
// 6: End
// 2: nextTick          <- microtask, highest priority
// 3: Promise.then      <- microtask
// 4: setTimeout        <- macrotask
// 5: setImmediate      <- macrotask (check phase)
```

## The libuv Thread Pool

Node.js is single-threaded for JavaScript, but some operations CAN'T be async at the OS level:

- File system operations (on some OSes)
- DNS lookup (`dns.lookup`)
- Crypto operations (`crypto.pbkdf2`, `crypto.randomBytes`)
- `zlib` compression

These are offloaded to the **libuv thread pool** (default: 4 threads). The thread pool threads do the blocking work and callback the event loop when done.

```javascript
const crypto = require('crypto');

// This uses a thread pool thread (blocking crypto operation)
crypto.pbkdf2('password', 'salt', 100000, 64, 'sha512', (err, key) => {
    console.log('Hash computed');  // callback on event loop when done
});

console.log('Non-blocking: continues here immediately');
```

```bash
# Increase thread pool size for CPU-intensive tasks
UV_THREADPOOL_SIZE=8 node server.js
```

## Blocking the Event Loop (What NOT to Do)

Since Node.js is single-threaded, any synchronous CPU-intensive code blocks the entire event loop — no requests can be processed during this time.

```javascript
// BAD — blocks event loop for 5 seconds!
app.get('/compute', (req, res) => {
    let sum = 0;
    for (let i = 0; i < 10_000_000_000; i++) {
        sum += i;
    }
    res.json({ sum });
});

// During this computation, ALL other incoming requests are queued and waiting
// Server appears frozen to all other users!
```

**What blocks the event loop:**

```javascript
// Synchronous I/O (never in production)
const data = fs.readFileSync('/large-file.json');  // blocks!

// Heavy JSON parsing
const parsed = JSON.parse(hugeJsonString);  // blocks if very large

// Synchronous crypto
const hash = crypto.pbkdf2Sync('password', 'salt', 100000, 64, 'sha512');  // blocks!

// Complex regex on large strings
const match = /^(a+)+$/.exec(largeString);  // catastrophic backtracking

// Tight loops
while (true) { /* ... */ }
```

**Solutions:**

```javascript
// 1. Use async versions (most cases)
const data = await fs.promises.readFile('/file.json');  // non-blocking

// 2. Worker Threads for CPU-intensive work
const { Worker } = require('worker_threads');

app.get('/compute', (req, res) => {
    const worker = new Worker('./compute-worker.js', {
        workerData: { input: req.query.n }
    });
    worker.on('message', (result) => res.json({ result }));
    worker.on('error', (err) => res.status(500).json({ error: err.message }));
});

// compute-worker.js
const { workerData, parentPort } = require('worker_threads');
let sum = 0;
for (let i = 0; i < workerData.input; i++) sum += i;
parentPort.postMessage(sum);
// Runs in separate thread — doesn't block event loop

// 3. Child Process for separate CPU core
const { fork } = require('child_process');

app.get('/compute', (req, res) => {
    const child = fork('./compute.js');
    child.send({ n: req.query.n });
    child.on('message', (result) => {
        res.json({ result });
        child.kill();
    });
});
```

## Async Patterns in Node.js

### Callbacks (Old Style)

```javascript
fs.readFile('/data.json', 'utf8', (err, data) => {
    if (err) return handleError(err);
    const parsed = JSON.parse(data);
    db.query('SELECT ...', (err, rows) => {  // callback hell begins
        if (err) return handleError(err);
        // deeper nesting...
    });
});
```

### Promises

```javascript
fs.promises.readFile('/data.json', 'utf8')
    .then(data => JSON.parse(data))
    .then(parsed => db.query('SELECT ...', [parsed.id]))
    .then(rows => res.json(rows))
    .catch(err => res.status(500).json({ error: err.message }));
```

### Async/Await (Modern, Preferred)

```javascript
app.get('/data', async (req, res) => {
    try {
        const data = await fs.promises.readFile('/data.json', 'utf8');
        const parsed = JSON.parse(data);
        const rows = await db.query('SELECT * FROM users WHERE id = $1', [parsed.id]);
        res.json(rows);
    } catch (err) {
        res.status(500).json({ error: err.message });
    }
});
```

### Parallel Async Operations

```javascript
// Sequential (slow — waits for each before next)
const user    = await userService.getUser(userId);
const orders  = await orderService.getOrders(userId);
const rewards = await rewardService.getRewards(userId);
// Total: sum of all three wait times

// Parallel (fast — all run simultaneously)
const [user, orders, rewards] = await Promise.all([
    userService.getUser(userId),
    orderService.getOrders(userId),
    rewardService.getRewards(userId)
]);
// Total: max of the three wait times

// Parallel with error handling
const results = await Promise.allSettled([
    userService.getUser(userId),
    orderService.getOrders(userId)
]);
// results[0].status = 'fulfilled' or 'rejected'
// Doesn't fail-fast — gets all results even if some fail
```

## Worker Threads (Node.js 12+)

Worker threads run JavaScript in parallel threads — sharing memory (unlike child processes).

```javascript
// main.js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');

if (isMainThread) {
    // Main thread: create workers for CPU tasks
    const worker = new Worker(__filename, {
        workerData: { number: 1000000 }
    });

    worker.on('message', (result) => {
        console.log('Result:', result);
    });
} else {
    // Worker thread: do CPU work
    let sum = 0;
    for (let i = 0; i < workerData.number; i++) sum += i;
    parentPort.postMessage(sum);
}
```

### Worker Thread Pool

```javascript
const { Worker } = require('worker_threads');

class WorkerPool {
    constructor(workerFile, numWorkers) {
        this.workers = Array.from({ length: numWorkers },
            () => ({ worker: new Worker(workerFile), busy: false }));
    }

    async run(data) {
        const available = this.workers.find(w => !w.busy);
        if (!available) throw new Error('No workers available');

        available.busy = true;
        return new Promise((resolve, reject) => {
            available.worker.postMessage(data);
            available.worker.once('message', (result) => {
                available.busy = false;
                resolve(result);
            });
            available.worker.once('error', (err) => {
                available.busy = false;
                reject(err);
            });
        });
    }
}

const pool = new WorkerPool('./cpu-worker.js', 4);

app.get('/compute', async (req, res) => {
    const result = await pool.run({ n: req.query.n });
    res.json({ result });
});
```

## Node.js Streams

Streams process data in chunks — without loading everything into memory.

```javascript
// Without streams: loads entire file into memory (bad for large files)
app.get('/download', async (req, res) => {
    const data = await fs.promises.readFile('/huge-file.csv');  // 2GB in memory!
    res.send(data);
});

// With streams: pipes chunks directly to response (memory efficient)
app.get('/download', (req, res) => {
    const readStream = fs.createReadStream('/huge-file.csv');
    res.setHeader('Content-Type', 'text/csv');
    readStream.pipe(res);  // data flows chunk by chunk, never all in memory
});

// Transform stream (read -> transform -> write)
const { Transform } = require('stream');

const upperCaseTransform = new Transform({
    transform(chunk, encoding, callback) {
        this.push(chunk.toString().toUpperCase());
        callback();
    }
});

fs.createReadStream('/input.txt')
    .pipe(upperCaseTransform)
    .pipe(fs.createWriteStream('/output.txt'));
```

## Event Emitter Pattern

Node.js uses EventEmitter as the foundation of its async model.

```javascript
const EventEmitter = require('events');

class OrderService extends EventEmitter {
    placeOrder(orderData) {
        const order = this.saveOrder(orderData);

        // Emit event — all listeners called synchronously
        this.emit('orderPlaced', order);

        return order;
    }
}

const orderService = new OrderService();

// Register listeners
orderService.on('orderPlaced', (order) => {
    emailService.sendConfirmation(order.userEmail);
});

orderService.on('orderPlaced', (order) => {
    inventoryService.reserve(order.items);
});

orderService.on('error', (err) => {
    console.error('Order service error:', err);
});
```

## Interview Questions

**Q1: What is the Node.js Event Loop and why does it matter?**

The Event Loop is the mechanism that enables Node.js to handle concurrent I/O without multiple threads. It runs continuously, checking for completed I/O operations, timers, and other callbacks. When an async operation (file read, HTTP request) completes, its callback is queued and the event loop executes it. This allows one thread to serve thousands of concurrent connections — as long as no single operation blocks the thread.

**Q2: What is the difference between process.nextTick and setImmediate?**

`process.nextTick` callbacks execute before the event loop proceeds to the next phase — even before Promise callbacks. `setImmediate` executes in the "check" phase of the current event loop iteration — after I/O callbacks. `nextTick` has higher priority than Promises; `setImmediate` fires after the current poll phase completes. Use `nextTick` to defer work within the current operation; `setImmediate` to defer to after I/O.

**Q3: What blocks the Node.js Event Loop and how do you prevent it?**

Synchronous CPU-intensive operations block the single JS thread — heavy loops, synchronous file reads, large JSON.parse, complex regex, synchronous crypto. During this time no other requests can be processed. Prevent it by: using async APIs (`fs.promises`), offloading CPU work to Worker Threads, using streaming for large data, or running CPU-intensive services as separate processes/services.

**Q4: What is the libuv thread pool and what does it handle?**

libuv is the async I/O library underlying Node.js. Its thread pool (4 threads by default) handles operations that can't be done asynchronously at the OS level: file system operations, DNS lookup, crypto (`pbkdf2`, `randomBytes`), and zlib. These run on separate threads; when complete, their callbacks are queued on the event loop. Set `UV_THREADPOOL_SIZE` to increase for crypto/file-heavy workloads.

**Q5: What are Worker Threads and when should you use them?**

Worker Threads run JavaScript in parallel OS threads, sharing memory via SharedArrayBuffer. Use them for CPU-intensive operations (image processing, complex calculations, ML inference) that would block the event loop if run on the main thread. Unlike child processes, workers share memory (no serialization overhead) but they increase memory usage and add coordination complexity. For I/O-bound work, async/await is sufficient; Worker Threads are specifically for CPU-bound parallelism.
