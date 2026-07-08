# Module 13: Rate Limiting

**Level:** Advanced

Rate limiting controls how many requests a user (or IP address) can make in a given time window — critical for protecting your API from abuse, brute-force attacks, and accidental overload.

---

## 1. Why Rate Limiting Matters

Without limits, a single misbehaving client (or attacker) could:

- Hammer your login endpoint trying thousands of password guesses (brute force).
- Scrape your entire API extremely fast.
- Accidentally overload your server with a buggy retry loop.

**Rate limiting says:** "You may only make N requests per T seconds. After that, you get blocked (usually with an HTTP `429 Too Many Requests`) until the window resets."

Redis is perfect for this because rate limiting needs to be **extremely fast** (checked on every request) and needs **atomic counters** that work correctly even under massive concurrency.

---

## 2. Approach 1: Fixed Window Counter (Simplest)

**Idea:** Count requests per user within a fixed time window (e.g., "this minute"). Reset the counter when the window ends.

```js
async function fixedWindowRateLimit(userId, limit = 10, windowSeconds = 60) {
  // Key includes a time bucket so it naturally resets — e.g. "ratelimit:15:2026-07-04-14-32"
  const currentMinute = Math.floor(Date.now() / 1000 / windowSeconds);
  const key = `ratelimit:${userId}:${currentMinute}`;

  // INCR atomically increments the counter (creates it at 1 if it doesn't exist)
  const count = await redis.incr(key);

  if (count === 1) {
    // First request in this window — set the key to expire automatically
    // so we don't accumulate rate-limit keys forever
    await redis.expire(key, windowSeconds);
  }

  if (count > limit) {
    return { allowed: false, remaining: 0 };
  }

  return { allowed: true, remaining: limit - count };
}
```

**Express middleware using it:**

```js
async function rateLimitMiddleware(req, res, next) {
  const userId = req.ip; // or req.user.id if authenticated

  const { allowed, remaining } = await fixedWindowRateLimit(userId, 10, 60);

  res.setHeader("X-RateLimit-Remaining", remaining);

  if (!allowed) {
    return res.status(429).json({ error: "Too many requests. Please slow down." });
  }

  next();
}

app.use("/api", rateLimitMiddleware);
```

**Problem with fixed windows — the "boundary burst" issue:**

```
Window 1: [10:00:00 - 10:00:59]   Window 2: [10:01:00 - 10:01:59]
                    │
User sends 10 requests at 10:00:59  ──┐
User sends 10 requests at 10:01:00  ──┴─▶ 20 requests in ~1 second!
```

Because the counter resets exactly at the window boundary, a clever/unlucky client could send a burst right at the edge of two windows and effectively double their allowed rate for a brief moment.

---

## 3. Approach 2: Sliding Window Log (More Accurate)

**Idea:** Instead of a fixed bucket, track the **exact timestamp** of every request in a Sorted Set, and only count requests within the last N seconds *relative to right now* — not tied to a fixed clock boundary.

```js
async function slidingWindowRateLimit(userId, limit = 10, windowSeconds = 60) {
  const key = `ratelimit:sliding:${userId}`;
  const now = Date.now();
  const windowStart = now - windowSeconds * 1000;

  // 1. Remove any timestamps older than our window (cleanup)
  await redis.zremrangebyscore(key, 0, windowStart);

  // 2. Count how many requests remain within the window
  const requestCount = await redis.zcard(key);

  if (requestCount >= limit) {
    return { allowed: false };
  }

  // 3. Record this request (score = timestamp, member = a unique id for this request)
  await redis.zadd(key, now, `${now}-${Math.random()}`);

  // 4. Make sure the key eventually cleans itself up if the user goes inactive
  await redis.expire(key, windowSeconds);

  return { allowed: true };
}
```

This is more accurate (no boundary burst issue) but uses slightly more memory (storing one entry per request within the window, instead of just a single counter).

---

## 4. Approach 3: Token Bucket (Smooth, Allows Bursts Intentionally)

**Idea:** Imagine a bucket that holds up to N tokens. Each request consumes 1 token. Tokens refill gradually over time (e.g., 1 token every 6 seconds). If the bucket is empty, the request is denied.

This is great for APIs that want to allow occasional short bursts, while still enforcing a steady average rate.

```js
async function tokenBucketRateLimit(userId, maxTokens = 10, refillRatePerSecond = 1) {
  const key = `ratelimit:bucket:${userId}`;
  const now = Date.now() / 1000; // current time in seconds

  // We store the bucket state as a Hash: { tokens, lastRefillTimestamp }
  const bucket = await redis.hgetall(key);

  let tokens = bucket.tokens ? parseFloat(bucket.tokens) : maxTokens;
  let lastRefill = bucket.lastRefill ? parseFloat(bucket.lastRefill) : now;

  // Calculate how many tokens should have regenerated since the last check
  const elapsedSeconds = now - lastRefill;
  const refilledTokens = elapsedSeconds * refillRatePerSecond;
  tokens = Math.min(maxTokens, tokens + refilledTokens); // never exceed the max capacity

  if (tokens < 1) {
    // Not enough tokens — deny the request, but still save the updated refill state
    await redis.hset(key, { tokens, lastRefill: now });
    return { allowed: false };
  }

  // Consume 1 token for this request
  tokens -= 1;
  await redis.hset(key, { tokens, lastRefill: now });

  return { allowed: true, tokensRemaining: tokens };
}
```

> Note: in a real high-concurrency production system, you'd wrap this logic in a Lua script (see below) to make the whole read-modify-write sequence atomic — otherwise two simultaneous requests could both read the same token count before either writes back, letting more requests through than intended.

---

## 5. Making It Truly Atomic with a Lua Script (Production-Grade)

Redis lets you run a small script directly on the server, executed as a **single atomic operation** — no other command can interleave in the middle of it. This solves the "read-then-write" race condition mentioned above.

```js
// A Lua script for a simple atomic fixed-window rate limiter
const rateLimitScript = `
  local key = KEYS[1]
  local limit = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])

  local current = redis.call("INCR", key)

  if current == 1 then
    redis.call("EXPIRE", key, window)
  end

  if current > limit then
    return 0  -- not allowed
  else
    return 1  -- allowed
  end
`;

async function atomicRateLimit(userId, limit = 10, windowSeconds = 60) {
  const key = `ratelimit:${userId}`;

  // eval runs the Lua script atomically on the Redis server itself
  const result = await redis.eval(rateLimitScript, 1, key, limit, windowSeconds);

  return result === 1;
}
```

**Why this matters:** Without Lua, `INCR` + `EXPIRE` are two separate round-trips, but `INCR` alone is already atomic — the real risk shows up in more complex logic (like the token bucket) where you read a value, do math on it, then write it back. A Lua script guarantees the *entire* sequence happens as one uninterruptible unit, which is essential at high concurrency.

---

## 6. Comparing the Approaches

| Approach | Accuracy | Memory Use | Complexity | Good For |
|---|---|---|---|---|
| Fixed Window | Low-ish (boundary bursts) | Very low | Very simple | Quick, simple protection |
| Sliding Window Log | High | Higher (stores each request) | Medium | APIs needing precise limits |
| Token Bucket | High, allows controlled bursts | Low | Medium-high | APIs that want smooth but flexible limits |

---

## 7. Interview Question: "How would you rate-limit a login endpoint to prevent brute-force attacks?"

**Answer:** Use a Redis counter keyed by IP address (and/or username) with a fixed or sliding window — e.g., "max 5 failed login attempts per 15 minutes." On each failed login, `INCR` the counter and set an expiry on first attempt. If the counter exceeds the limit, reject further attempts (ideally with an increasing lockout period) and return a `429` status. You'd typically combine this with logging/alerting for repeated abuse from the same IP.

---

## 8. Summary

- Rate limiting protects APIs from abuse and overload — Redis's atomic counters and expiry make it a natural fit.
- **Fixed window** = simplest, but can allow bursts at window boundaries.
- **Sliding window log** = more accurate, slightly more memory.
- **Token bucket** = allows controlled bursts while enforcing a steady average rate.
- For anything beyond a simple `INCR`, use a **Lua script** to guarantee atomicity under concurrency.

---

## 9. Homework

1. Implement the fixed-window rate limiter as Express middleware and test it with rapid `curl` requests.
2. Explain, in your own words, the "boundary burst" problem with fixed windows.
3. Try running the Lua-based atomic rate limiter and confirm it behaves correctly under a quick burst of requests (e.g., using a small script that fires 20 requests in a loop).

---

**Next up:** [Module 14 — Distributed Locks](./14-distributed-locks.md)
