# Day 3 — Async Patterns: Callbacks → Promises → Async/Await + Error Handling

> **Why this day matters:** Yesterday you learned the *engine* (the event loop). Today you learn the *syntax and patterns* developers actually write on top of that engine. An interviewer with 3 YOE expectations won't just ask "what is a Promise" — they'll ask you to **refactor callback hell into async/await**, explain **why unhandled promise rejections crash production apps**, and **trace error propagation** through async code. This is the day that turns yesterday's theory into things you type every day at work.

---

## 1. Background: why did callbacks even exist in the first place?

Go back to Day 2's core idea: Node doesn't wait for slow operations (file reads, DB calls, network requests). But if it doesn't wait... **how does your code get the result once it's ready?**

The answer, historically, was: **you pass a function (a callback) that Node will call later, once the result is ready.** This was the *only* pattern available in early Node.js (pre-2015), so it's baked deep into core Node APIs (`fs.readFile`, `http` callbacks, etc.) and a lot of older production codebases still use it.

```javascript
const fs = require('fs');

// "Here's a function — call it WHEN you're done reading the file."
// Node does NOT wait here. It registers this callback and moves on
// to execute the next lines of your script immediately.
fs.readFile('data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Something went wrong:', err);
    return;
  }
  console.log('File contents:', data);
});

console.log('This line runs BEFORE the file contents are printed!');
```

**Node.js convention you must know: "error-first callbacks."**
Notice the callback signature: `(err, data) => {}`. This is a hard convention across nearly all of Node's core APIs and most npm packages of that era: **the first argument is always reserved for an error** (or `null` if there's no error), and the second argument onward is the actual result. Interviewers sometimes literally ask: *"What is the error-first callback convention in Node.js?"* — now you can answer it precisely.

---

## 2. The problem: "Callback Hell" (a.k.a. the Pyramid of Doom)

**What it is:** when you need to perform several async operations **in sequence**, where each one depends on the previous one's result, you end up nesting callbacks inside callbacks inside callbacks.

```javascript
// Imagine: get a user, then get their orders, then get the order's payment details
getUser(userId, (err, user) => {
  if (err) return handleError(err);
  getOrders(user.id, (err, orders) => {
    if (err) return handleError(err);
    getPaymentDetails(orders[0].id, (err, payment) => {
      if (err) return handleError(err);
      console.log(payment);
      // ...and if there were 2 more steps, this would nest even deeper
    });
  });
});
```

**Why this is genuinely a problem (not just "ugly code"):**
- **Readability collapses** — code grows sideways instead of top-to-bottom, harder to follow the actual sequence of logic.
- **Error handling is repetitive and easy to forget** — you must remember `if (err) return handleError(err)` at EVERY level. Miss one, and an error silently disappears or crashes unpredictably.
- **Hard to compose** — running things in parallel (e.g., "fetch user AND fetch settings at the same time") is awkward to express with raw callbacks; you'd have to manually track how many callbacks have fired.

This exact pain point is **why Promises were introduced** — it's important to know the "why," not just the "what," because interviewers ask "why did JavaScript move away from callbacks?" to test real understanding versus rote memorization.

---

## 3. Promises — what they actually are under the hood

**What it is, precisely:** A Promise is an object representing the **eventual result** of an async operation. It's not the result itself — it's a *container* that will eventually hold either a successful value or an error, and lets you attach handlers for either outcome.

### The 3 states of a Promise (memorize this — asked very often)

1. **Pending** — initial state, operation not yet complete.
2. **Fulfilled** — operation completed successfully, has a resulting value.
3. **Rejected** — operation failed, has a reason/error.

**Critical detail:** once a Promise settles (fulfilled or rejected), it is **permanently locked** in that state — it can never go back to pending or switch from fulfilled to rejected or vice versa. This is what makes Promises predictable and chainable.

```javascript
// Creating a Promise manually (you rarely do this in real backend work —
// most libraries already return Promises — but you MUST understand this
// to explain how Promises work under the hood)
function delay(ms) {
  // The Promise constructor takes a function with two parameters:
  // 'resolve' (call this on success) and 'reject' (call this on failure)
  return new Promise((resolve, reject) => {
    if (ms < 0) {
      reject(new Error('Delay cannot be negative'));
      return;
    }
    setTimeout(() => {
      resolve(`Waited ${ms}ms`); // this value becomes available in .then()
    }, ms);
  });
}

delay(1000)
  .then((result) => console.log(result))   // runs if resolved
  .catch((err) => console.error(err));      // runs if rejected
```

### Rewriting callback hell with Promise chaining

```javascript
// Assume getUser, getOrders, getPaymentDetails now RETURN Promises
// (this is how most modern libraries, like Mongoose or Axios, are designed)
getUser(userId)
  .then((user) => getOrders(user.id))
  .then((orders) => getPaymentDetails(orders[0].id))
  .then((payment) => console.log(payment))
  .catch((err) => handleError(err)); // ONE catch handles errors from ANY step above
```

Notice: this is **flat**, not nested — and there's only **one** error handler for the entire chain, because an error at any step automatically skips all remaining `.then()` calls and jumps straight to the nearest `.catch()`. This is a core behavior to understand: **Promise rejection propagates down the chain until it finds a `.catch()` (or an `async/await` function's `try/catch`)**.

### `Promise.all`, `Promise.allSettled`, `Promise.race`, `Promise.any` — all 4 get asked

```javascript
// Promise.all — runs all in PARALLEL, waits for ALL to succeed.
// If even ONE rejects, the whole thing rejects immediately (fail-fast).
// Use when: you need ALL results and can't proceed without any one of them.
const [user, settings, permissions] = await Promise.all([
  getUser(id),
  getSettings(id),
  getPermissions(id),
]);

// Promise.allSettled — runs all in PARALLEL, waits for ALL to finish
// (success or failure), and never rejects itself. You get an array of
// { status: 'fulfilled', value } or { status: 'rejected', reason } for each.
// Use when: you want results from everything regardless of individual failures
// (e.g., sending notifications to 5 services — one failing shouldn't stop the rest).
const results = await Promise.allSettled([
  notifySlack(msg),
  notifyEmail(msg),
  notifySMS(msg),
]);

// Promise.race — resolves/rejects as soon as the FIRST promise settles
// (whichever finishes first, success or failure).
// Use when: implementing timeouts — race the real operation against a timer.
const result = await Promise.race([
  fetchDataFromAPI(),
  new Promise((_, reject) => setTimeout(() => reject(new Error('Timeout')), 5000)),
]);

// Promise.any — resolves as soon as the FIRST one succeeds; only rejects if
// ALL of them fail (with an AggregateError).
// Use when: trying multiple redundant sources and you just need ONE to work
// (e.g., hitting 3 mirrored API endpoints, first success wins).
const firstSuccess = await Promise.any([
  fetchFromMirror1(),
  fetchFromMirror2(),
  fetchFromMirror3(),
]);
```

**This table is a great thing to have memorized for interviews:**

| Method | Waits for | Fails when | Common real use case |
|---|---|---|---|
| `Promise.all` | All to settle | Any one rejects (fail-fast) | Fetch user + orders + settings in parallel, all required |
| `Promise.allSettled` | All to settle | Never rejects | Send notifications to multiple channels, don't care if one fails |
| `Promise.race` | First to settle | If the fastest one rejects | Implement a timeout pattern |
| `Promise.any` | First to fulfill | All reject | Try multiple redundant data sources |

---

## 4. Async/Await — "syntactic sugar," but you need to know what's under the sugar

**What it is:** `async/await` is built **on top of** Promises — it's not a separate mechanism. `async` functions always return a Promise, and `await` pauses execution **within that function** (not the whole program) until the awaited Promise settles.

```javascript
// The exact same logic as the .then() chain above, written with async/await
async function getPaymentInfo(userId) {
  try {
    const user = await getUser(userId);
    const orders = await getOrders(user.id);
    const payment = await getPaymentDetails(orders[0].id);
    return payment;
  } catch (err) {
    // try/catch here catches errors from ANY of the awaited calls above —
    // exactly like .catch() does for a Promise chain, because under the
    // hood, a rejected awaited Promise THROWS inside the async function.
    handleError(err);
    throw err; // re-throw so the CALLER of this function also knows it failed
  }
}
```

**Why this is genuinely better for readability (not just preference):** the code reads top-to-bottom like synchronous code, while still being fully non-blocking underneath. This is the single biggest reason async/await replaced raw `.then()` chains in most modern codebases.

**Critical gotcha #1 — sequential vs parallel awaits (asked CONSTANTLY in interviews):**

```javascript
// WRONG (if these are independent): this runs SEQUENTIALLY —
// getOrders doesn't even START until getUser fully finishes.
// Total time = time(getUser) + time(getOrders) -- SLOWER
async function slow() {
  const user = await getUser(1);
  const orders = await getOrders(2); // waits for getUser to finish first, unnecessarily
  return { user, orders };
}

// RIGHT (when independent): kick off both at the same time using Promise.all
// Total time = max(time(getUser), time(getOrders)) -- FASTER
async function fast() {
  const [user, orders] = await Promise.all([getUser(1), getOrders(2)]);
  return { user, orders };
}
```

This is a real performance bug seen in production code, and a favorite "spot the issue" interview question: junior/mid-level engineers often `await` things sequentially out of habit even when the operations don't depend on each other, silently doubling response time.

**Critical gotcha #2 — forgetting `await` in a loop:**

```javascript
// WRONG: forEach does NOT wait for async callbacks — it fires all of them
// and moves on immediately, ignoring the returned promises entirely.
// This can cause unhandled rejections and unpredictable completion order.
async function processAllWrong(ids) {
  ids.forEach(async (id) => {
    await processItem(id); // this await is USELESS here — forEach doesn't await it
  });
  console.log('Done!'); // this prints IMMEDIATELY, before any item is actually processed
}

// RIGHT (sequential, one at a time): use a plain for...of loop
async function processAllSequential(ids) {
  for (const id of ids) {
    await processItem(id); // properly waits before moving to next iteration
  }
  console.log('Done!'); // prints only after ALL items are processed
}

// RIGHT (parallel, all at once): use Promise.all with map
async function processAllParallel(ids) {
  await Promise.all(ids.map((id) => processItem(id)));
  console.log('Done!');
}
```

---

## 5. Error Handling Deep Dive — where most production incidents trace back to

### Unhandled Promise Rejections — a real production killer

**What happens if you DON'T catch a rejected promise:**

```javascript
async function riskyOperation() {
  throw new Error('Something broke');
}

riskyOperation(); // NOTICE: no .catch() and not awaited inside a try/catch
```

In modern Node.js (since v15), an **unhandled promise rejection will crash your entire process** by default (it used to just print a warning in older versions). This is a HUGE deal in production — one missed `.catch()` on a non-critical background task can take down your entire API server.

**Why Node made this change:** an unhandled rejection usually means a real bug went unnoticed, and silently swallowing it (old behavior) let bugs hide in production for a long time before anyone caught them. Crashing loudly forces you to fix it — paired with a process manager (PM2, Kubernetes restarts — Week 4) that restarts the process.

**How to globally catch what slipped through (a safety net, not a substitute for proper handling):**

```javascript
// This should exist in EVERY production Node.js app as a last line of defense.
// It does NOT fix bugs — it lets you LOG them and shut down gracefully
// instead of crashing in an uncontrolled, silent way.
process.on('unhandledRejection', (reason, promise) => {
  console.error('Unhandled Rejection at:', promise, 'reason:', reason);
  // In production: log to your monitoring tool (e.g., Sentry, Datadog), 
  // then often gracefully shut down so a process manager can restart cleanly.
});

process.on('uncaughtException', (err) => {
  console.error('Uncaught Exception:', err);
  // Same idea — but this catches SYNCHRONOUS errors that escaped a try/catch.
  // After logging, it's standard practice to exit the process — the app is 
  // in an unknown state at this point and continuing could cause worse issues.
  process.exit(1);
});
```

### Error handling patterns at the Express route level (forward-reference to Week 2, but good to see now)

```javascript
// A common real-world mistake: forgetting try/catch in an async Express route
// handler. If getUser() rejects, Express does NOT automatically catch it in
// older Express versions (pre-5) — this can crash the server or hang the request.
app.get('/user/:id', async (req, res) => {
  try {
    const user = await getUser(req.params.id);
    res.json(user);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// A cleaner production pattern: a wrapper that auto-catches async errors
// and forwards them to Express's centralized error-handling middleware,
// so you don't repeat try/catch in every single route handler.
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

app.get('/user/:id', asyncHandler(async (req, res) => {
  const user = await getUser(req.params.id);
  res.json(user);
}));
```

---

## 6. How this connects to real backend work (3 YOE framing)

Be ready to discuss these scenarios — they come up as "tell me about a time..." or "how would you handle..." questions:

- **"How would you fetch data from 3 independent microservices efficiently?"** → `Promise.all` (or `allSettled` if partial failure is acceptable) instead of sequential awaits.
- **"Our server crashed randomly in production with no clear error in our route handlers — how would you investigate?"** → Check for unhandled promise rejections from background tasks (cron jobs, event listeners, fire-and-forget async calls) not wrapped in try/catch, and check if `process.on('unhandledRejection')` logging was in place to capture it.
- **"How do you avoid duplicating try/catch in every Express route?"** → The `asyncHandler` wrapper pattern above, or use a library like `express-async-errors`.
- **"What's wrong with using `.forEach()` for async operations?"** → It doesn't await the callbacks, so it doesn't guarantee completion order or even completion at all before continuing.

---

## 7. Hands-on practice for today

```javascript
// async-practice.js
// Task: refactor this callback-based code into async/await with proper error handling

function getUserCB(id, cb) {
  setTimeout(() => {
    if (id <= 0) return cb(new Error('Invalid id'));
    cb(null, { id, name: 'Test User' });
  }, 500);
}

function getOrdersCB(userId, cb) {
  setTimeout(() => {
    cb(null, [{ orderId: 101, total: 250 }]);
  }, 500);
}

// STEP 1: Wrap these in Promises (this pattern is called "promisifying" —
// Node even ships a built-in helper for this: util.promisify)
const util = require('util');
const getUser = util.promisify(getUserCB);
const getOrders = util.promisify(getOrdersCB);

// STEP 2: Write an async function using try/catch that calls both,
// IN PARALLEL if they don't depend on each other, or sequentially if they do.
// (Here getOrders needs userId from getUser, so this must be sequential.)
async function main() {
  try {
    const user = await getUser(1);
    const orders = await getOrders(user.id);
    console.log({ user, orders });
  } catch (err) {
    console.error('Failed:', err.message);
  }
}

main();
```

Run it, then break it intentionally (`getUser(-1)`) to see your `catch` block handle it gracefully — and try removing the `try/catch` entirely to watch Node throw an unhandled rejection warning/crash.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What is "callback hell" and why is it a problem beyond just looking ugly?**
   A: Deeply nested callbacks from sequential dependent async calls. Beyond readability, it makes error handling repetitive and error-prone (forgetting `if (err)` at any level silently swallows errors), and makes parallel composition awkward.

2. **Q: What are the three states of a Promise, and can a Promise change state more than once?**
   A: Pending, fulfilled, rejected. No — once settled (fulfilled or rejected), it's permanently locked in that state.

3. **Q: Difference between `Promise.all` and `Promise.allSettled`?**
   A: `Promise.all` fails fast — if any promise rejects, the whole thing rejects immediately, even if others would have succeeded. `Promise.allSettled` always resolves once all promises are done, giving you a status/value or status/reason for each — useful when partial failure is acceptable.

4. **Q: What happens if a Promise rejection is never caught, in a modern Node.js version?**
   A: It crashes the entire process by default (since Node v15) — this is a deliberate change from earlier versions that just logged a warning, specifically to surface real bugs instead of letting them silently fail in production.

5. **Q: What's wrong with `array.forEach(async (item) => { await doSomething(item) })`?**
   A: `forEach` doesn't wait for the async callbacks to resolve — it fires them all without waiting, so code after the loop runs before the async work completes, and rejected promises inside become unhandled. Use a `for...of` loop for sequential awaiting, or `Promise.all` with `.map()` for parallel.

6. **Q: How would you implement a timeout for a slow API call?**
   A: `Promise.race()` between the actual API call and a Promise that rejects after a `setTimeout` — whichever settles first wins.

---

## Tomorrow (Day 4) Preview
We cover Node's **core built-in modules** used daily in real backend work: `fs` (file system, sync vs async vs promises API), `path`, `os`, and a deep dive into `EventEmitter` (the pattern underlying Node's entire I/O model, and the basis for building your own event-driven features — e.g., emitting a "user registered" event that multiple parts of your app react to).
