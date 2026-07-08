# Module 14: Distributed Locks

**Level:** Advanced

When multiple servers (or multiple processes) might try to do the same critical operation at the same time, you need a way to ensure only **one** of them succeeds. That's what a distributed lock is for.

---

## 1. The Problem: Race Conditions Across Multiple Servers

Imagine you run 3 instances of your Node.js app (for scalability) behind a load balancer. Now imagine a scheduled job — "send the daily report email" — running on **all 3 instances at once**, because they all have the same cron schedule.

```
Server 1 ──▶ "Time to send the daily report!" ──▶ sends email
Server 2 ──▶ "Time to send the daily report!" ──▶ sends email  (duplicate!)
Server 3 ──▶ "Time to send the daily report!" ──▶ sends email  (duplicate!)
```

Result: your users get the same email 3 times. Not great.

A **single-process lock** (like a JavaScript variable or a mutex library) won't help here, because each server is a *separate process on a separate machine* — they don't share memory. You need a lock that all 3 servers can see and respect: a **distributed lock**, stored somewhere all of them can reach — Redis is a perfect fit because it's fast, shared, and supports atomic operations.

---

## 2. The Basic Idea: `SET key value NX EX seconds`

Recall from Module 5: `SET key value NX` only succeeds if the key **doesn't already exist**. This single atomic command is the foundation of a Redis lock:

```js
async function acquireLock(lockName, ttlSeconds = 10) {
  const lockKey = `lock:${lockName}`;

  // NX = only set if it doesn't exist ("I'm claiming this lock")
  // EX = automatically expire after ttlSeconds (a safety net — see below)
  const result = await redis.set(lockKey, "locked", "NX", "EX", ttlSeconds);

  // result is "OK" if we successfully acquired the lock, or null if someone else already holds it
  return result === "OK";
}

async function releaseLock(lockName) {
  const lockKey = `lock:${lockName}`;
  await redis.del(lockKey);
}
```

**Usage:**

```js
async function sendDailyReportSafely() {
  const gotLock = await acquireLock("daily-report-email", 30); // hold lock for max 30 seconds

  if (!gotLock) {
    console.log("Another server already sending the report. Skipping.");
    return;
  }

  try {
    await sendDailyReportEmail(); // only ONE server will ever reach this line
  } finally {
    await releaseLock("daily-report-email"); // always release, even if an error occurs
  }
}
```

Because `SET ... NX` is a single atomic Redis command, even if all 3 servers call `acquireLock` at the **exact same millisecond**, Redis guarantees only one of them gets `"OK"` back — the others get `null`.

---

## 3. Why the TTL (Expiry) is Critical — "Deadlock Prevention"

**What if the server holding the lock crashes before calling `releaseLock`?** Without a TTL, that lock would remain forever, and no server could ever acquire it again — a permanent deadlock.

```js
// ❌ DANGEROUS: no expiry — if this process crashes, the lock is stuck forever
await redis.set(lockKey, "locked", "NX");

// ✅ SAFE: expires automatically after 30s even if we crash before releasing
await redis.set(lockKey, "locked", "NX", "EX", 30);
```

**Rule of thumb:** Always set a TTL slightly longer than the maximum time you expect the protected operation to realistically take.

---

## 4. The Problem With a Simple Lock: Releasing Someone Else's Lock

Here's a subtle bug: what if Server A's operation takes *longer* than the TTL? The lock **expires automatically** while Server A is still working, and Server B could then acquire the "same" lock and start working too — now they're both running at once. Then, when Server A finally finishes, it calls `releaseLock`, which deletes the key — but that key now actually belongs to **Server B**! Server A just deleted Server B's lock.

```
Server A acquires lock (TTL=10s)
Server A's operation takes 15s (longer than TTL!)
   ...at 10s, lock auto-expires...
Server B acquires the "same" lock at 10s  ⚠️ now both are running
Server A finishes at 15s, calls releaseLock() → deletes Server B's active lock!  ⚠️ bug
```

**Fix: Use a unique random value per lock holder, and only delete if the value matches yours.**

```js
const crypto = require("crypto");

async function acquireLockSafely(lockName, ttlSeconds = 10) {
  const lockKey = `lock:${lockName}`;
  const uniqueToken = crypto.randomBytes(16).toString("hex"); // identifies THIS specific lock attempt

  const result = await redis.set(lockKey, uniqueToken, "NX", "EX", ttlSeconds);

  return result === "OK" ? uniqueToken : null; // return the token so we can safely release later
}

// A Lua script guarantees the "check, then delete" is a single atomic step —
// this prevents accidentally deleting someone ELSE's lock
const releaseLockScript = `
  if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  else
    return 0
  end
`;

async function releaseLockSafely(lockName, uniqueToken) {
  const lockKey = `lock:${lockName}`;
  // Only deletes the key if its value STILL matches our unique token —
  // meaning we still actually own this lock
  return redis.eval(releaseLockScript, 1, lockKey, uniqueToken);
}
```

**Usage:**

```js
async function sendDailyReportSafely() {
  const token = await acquireLockSafely("daily-report-email", 30);
  if (!token) {
    console.log("Lock already held elsewhere. Skipping.");
    return;
  }

  try {
    await sendDailyReportEmail();
  } finally {
    await releaseLockSafely("daily-report-email", token); // safe: only releases OUR lock
  }
}
```

---

## 5. Redlock: The "Enterprise-Grade" Version (Good to Know)

For a *single* Redis instance, the pattern above is generally solid for most use cases. But if you're running Redis in a highly available cluster setup (Module 16) and need extremely strong guarantees even during Redis failovers, there's a more advanced algorithm called **Redlock**, which acquires the lock across *multiple independent* Redis instances and only considers the lock "acquired" if a majority of them agree.

You won't need to implement Redlock by hand — libraries like `redlock` (npm) implement it for you. Just know the term for interviews: **Redlock is Redis's official algorithm for distributed locking across multiple Redis nodes, designed for extra safety in failover scenarios.**

---

## 6. Interview Question: "Why do you need a unique token in a distributed lock, not just a simple flag?"

**Answer:** Without a unique token, if a lock expires (TTL) while its original holder is still working, and another process acquires it, the *original* holder could later call a naive `DEL` and accidentally delete the *new* holder's lock — letting two processes believe they safely hold the lock at the same time, defeating the whole purpose. A unique per-holder token, checked atomically (via a Lua script) before deletion, ensures you only ever release a lock that is *still actually yours*.

---

## 7. Summary

- Distributed locks coordinate access to a shared resource across multiple servers/processes.
- The foundation is `SET key value NX EX seconds` — atomic "claim if unclaimed."
- Always set a TTL to prevent permanent deadlocks if a holder crashes.
- Use a unique token per lock attempt + a Lua script on release, to avoid accidentally releasing someone else's lock.
- For extra-strong guarantees across a Redis cluster, look into the Redlock algorithm (via the `redlock` npm library).

---

## 8. Homework

1. Simulate the "duplicate email" scenario without a lock (run the same function twice quickly) — notice both executions succeed.
2. Add the lock and confirm only one execution proceeds while the other is skipped.
3. Explain, in your own words, why a plain `DEL` on release is unsafe without a unique token check.

---

**Next up:** [Module 15 — Queues](./15-queues.md)
