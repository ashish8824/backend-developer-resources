# Rate Limiter

## What is a Rate Limiter?

A Rate Limiter controls how many requests a user/client can make in a given period.

**Example:** 100 requests / minute

```
Request 1   → Allowed
Request 2   → Allowed
...
Request 100 → Allowed
Request 101 → Rejected (HTTP 429)
```

## Why do we need it?

### 1. Prevent Abuse

A malicious user may send 10,000 requests/sec. Without rate limiting:

```
Server CPU ↑
Memory ↑
Database Load ↑
Application Crash
```

### 2. Fair Usage

Suppose 100 users are using an API and one user sends 5000 requests/minute.
A rate limiter ensures everyone gets a fair share of shared resources (DB connections, thread pools, bandwidth).

### 3. Security

Protects against:

- Brute Force Login Attacks
- OTP Spam
- API Abuse / Scraping
- DDoS-like traffic

### 4. Cost Control

If you're paying per downstream call (e.g. a third-party SMS/email API, a metered cloud service), an uncapped client can blow up your bill. Rate limiting caps worst-case spend.

## Real-World Examples

| API           | Limit                       |
|---------------|------------------------------|
| Login API     | 5 login attempts / minute    |
| OTP API       | 3 OTP requests / 10 minutes  |
| Payment API   | 20 transactions / minute     |
| Public API    | 100 requests / minute        |

## Where Should Rate Limiting Live?

| Layer | Pros | Cons |
|---|---|---|
| Client-side throttling | Saves network round trips entirely | Easily bypassed by a malicious/buggy client — never trust the client alone |
| API Gateway / Load Balancer (e.g. Nginx, Kong, AWS API Gateway) | Centralized, blocks traffic before it reaches app servers, no app code changes | Less business-context aware (harder to do "5 login attempts per *user*" if the gateway only sees IPs) |
| Application layer | Full access to business context (user ID, plan tier, endpoint semantics) | Adds load to app servers; bad traffic still reaches them before being rejected |
| Database / downstream layer | Last line of defense | Too late — by now you've already paid the cost of the request reaching here |

In practice, production systems layer these: a coarse IP-based limit at the gateway, and a finer user/endpoint-based limit in the application (often backed by Redis either way).

## Common Algorithms

### 1. Fixed Window

Store: `User -> Request Count`

**Example:** Window = 1 minute, Limit = 5

```
Time: 10:00:00
Requests:
1 -> Allowed
2 -> Allowed
3 -> Allowed
4 -> Allowed
5 -> Allowed
6 -> Blocked

At 10:01:00 -> Counter resets.
```

**Problem — the Boundary Problem:**

```
10:00:59 -> 5 requests
10:01:01 -> 5 requests
```

That's 10 requests in 2 seconds, but the rate limiter still allows them because they fall in two different windows.

- **Memory:** O(1) per user — just a counter and a timestamp.
- **Pros:** Trivial to implement, very cheap.
- **Cons:** Allows up to 2x the limit at window boundaries.

### 2. Sliding Window Log

Instead of storing counts, store the timestamp of every request.

**Example:** Limit = 5 requests, Window = 60 sec

```
Current Time: 10:01:00
Window = 60 sec, so remove all requests older than 10:00:00
Now count the remaining (active) timestamps.
If count < limit -> Allow, else -> Reject
```

- **Memory:** O(limit) per user — you must store every timestamp in the window. Expensive at high limits/high traffic.
- **Pros:** Perfectly accurate, no boundary problem.
- **Cons:** Memory-heavy; pruning old timestamps on every request adds overhead.

### 3. Sliding Window Counter (Hybrid)

A practical compromise between Fixed Window (cheap, inaccurate) and Sliding Window Log (accurate, expensive). Keep counts for the *current* and *previous* fixed windows, and weight the previous window's count by how much it overlaps the current sliding window.

**Example:** Limit = 5/min, previous window (10:00–10:01) had 5 requests, current window (10:01–10:02) has 2 requests so far, and we're 30% into the current window (10:01:18).

```
Weighted count = (previous_window_count * (1 - elapsed_fraction)) + current_window_count
              = (5 * (1 - 0.30)) + 2
              = 3.5 + 2
              = 5.5  -> exceeds limit of 5 -> Reject
```

- **Memory:** O(1) per user — just two counters.
- **Pros:** Near-accurate, cheap to store. This is what most production systems (e.g. Cloudflare) actually use.
- **Cons:** Slightly approximate (the weighting assumes uniform request distribution within the previous window, which isn't always true).

### 4. Token Bucket (Most Popular)

Imagine a bucket with Capacity = 10 Tokens. Every request consumes 1 Token. Tokens refill continuously at a fixed rate.

**Example:** Refill rate = 2 Tokens/sec, Initial = 10 Tokens

```
User sends 10 Requests -> All allowed. Tokens: 0
Wait 3 sec -> New Tokens: 6 -> Allows 6 more requests
```

This is used widely because it allows short bursts (up to bucket capacity) while still enforcing a long-term average rate (the refill rate).

- **Memory:** O(1) per user — token count + last refill timestamp.
- **Pros:** Burst-friendly, smooth average enforcement, simple math.
- **Cons:** A full bucket lets a client fire a large burst instantly, which can momentarily spike downstream load.

### 5. Leaky Bucket

Conceptually the inverse of Token Bucket. Requests enter a queue (the "bucket"); the queue is processed ("leaks") at a fixed, constant output rate. If the queue is full, new requests are dropped.

```
Incoming requests -> [ Queue, capacity = N ] -> processed at fixed rate R/sec -> Server
```

- **Pros:** Output rate is perfectly smooth — great when the downstream system (e.g. a legacy DB) cannot tolerate *any* bursts.
- **Cons:** Bursty clients get queued/delayed rather than served immediately, even if capacity is technically available — less burst-friendly than Token Bucket.

### Algorithm Comparison

| Algorithm | Accuracy | Memory | Allows Bursts? | Typical Use |
|---|---|---|---|---|
| Fixed Window | Low (boundary spikes) | O(1) | Yes (uncontrolled, at boundary) | Simple internal APIs |
| Sliding Window Log | High | O(limit) | Controlled | Low-traffic, high-accuracy needs |
| Sliding Window Counter | High (approx.) | O(1) | Controlled | Most production systems |
| Token Bucket | High | O(1) | Yes (controlled, up to capacity) | Public APIs, AWS, Stripe |
| Leaky Bucket | High | O(queue size) | No (smoothed output) | Traffic shaping to fragile downstreams |

## HTTP Semantics for Rejected Requests

A well-behaved rate limiter doesn't just return a bare `429` — it tells the client how to behave:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1750250460
```

- `Retry-After`: seconds the client should wait before retrying.
- `X-RateLimit-Limit` / `Remaining` / `Reset`: lets well-behaved clients self-throttle *before* hitting the limit.

## Interview Question: Design a Login Rate Limiter

**Requirement:** 5 login attempts/minute/user

**Store:**

```
Map<UserId, Count>
Map<UserId, WindowStartTime>
```

**Logic:**

```
Window expired?
    YES -> reset count
    NO  -> increment count

Count > 5 ?
    Reject
Else
    Allow
```

## Java Implementation (Fixed Window, Single Server)

```java
import java.util.HashMap;
import java.util.Map;

public class RateLimiter {

    private final int limit;
    private final long windowSize;

    private final Map<String, Integer> requestCount =
            new HashMap<>();

    private final Map<String, Long> windowStart =
            new HashMap<>();

    public RateLimiter(int limit, long windowSizeSeconds) {
        this.limit = limit;
        this.windowSize = windowSizeSeconds * 1000;
    }

    public synchronized boolean allowRequest(String userId) {

        long now = System.currentTimeMillis();

        // First request
        if (!windowStart.containsKey(userId)) {
            windowStart.put(userId, now);
            requestCount.put(userId, 1);
            return true;
        }

        long start = windowStart.get(userId);

        // Window expired
        if (now - start >= windowSize) {
            windowStart.put(userId, now);
            requestCount.put(userId, 1);
            return true;
        }

        int count = requestCount.get(userId);

        // Limit exceeded
        if (count >= limit) {
            return false;
        }

        requestCount.put(userId, count + 1);
        return true;
    }
}
```

## Distributed Rate Limiting with Redis

A naive distributed implementation does this in *two separate* Redis calls:

```
INCR  user:123:count
EXPIRE user:123:count 60
```

**Problem:** between the `INCR` and the `EXPIRE`, another request could read the count, or the process could crash after `INCR` but before `EXPIRE`, leaving a key that never expires. This is a classic **race condition** — two operations that need to be atomic, but aren't.

**Fix: do it in a single atomic Lua script** (Redis executes Lua scripts atomically):

```lua
-- rate_limiter.lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])

local current = redis.call("INCR", key)

if current == 1 then
    redis.call("EXPIRE", key, window)
end

if current > limit then
    return 0   -- reject
else
    return 1   -- allow
end
```

Called from Java (via Jedis):

```java
String script = "..."; // the Lua script above
Object result = jedis.eval(script, 1, "user:123:count", "5", "60");
boolean allowed = result.equals(1L);
```

This guarantees the increment, the TTL-set, and the limit-check all happen as one indivisible operation — no race conditions even with thousands of concurrent requests across multiple app servers.

For an accurate **sliding window** in Redis, a common pattern uses a Sorted Set per user, where the score is the request timestamp:

```
ZADD user:123:requests <timestamp> <timestamp>
ZREMRANGEBYSCORE user:123:requests 0 <timestamp - window>
ZCARD user:123:requests
```

`ZREMRANGEBYSCORE` prunes anything outside the window, and `ZCARD` gives the current count — again, this should be wrapped in a Lua script to keep it atomic.

## Production Architecture

```
Client
  |
Load Balancer
  |
Node1  Node2  Node3
      |
     Redis
```

Redis stores request counts so all servers share the same limit.

## Interview Questions

**Q1: Why not use HashMap in production?**

- Application restart → data lost
- Multiple servers → counts become inconsistent (each server has its own copy)

**Q2: Why Redis?**

- Fast, in-memory
- Atomic operations (e.g. `INCR`, or a Lua script for multi-step atomicity)
- TTL support (auto-expiring windows)
- Distributed — shared state across all app servers

**Q3: Which algorithm is best?**

Generally **Token Bucket** or **Sliding Window Counter**, because they:

- Support bursts in a controlled way (Token Bucket) or stay highly accurate at O(1) memory (Sliding Window Counter)
- Are computationally efficient
- Are widely adopted in production systems (AWS, Stripe, Cloudflare)

**Q4: How do you avoid race conditions in a distributed rate limiter backed by Redis?**

Don't issue `INCR` and `EXPIRE` as two separate round trips — wrap the whole check-and-increment in a single atomic Lua script (or use Redis's `MULTI`/`EXEC` transaction), so concurrent requests from different app servers can't interleave and corrupt the count.

**Q5: How is Token Bucket different from Leaky Bucket?**

Token Bucket controls the *rate of acceptance* and allows bursts up to bucket capacity (tokens accumulate while idle). Leaky Bucket controls the *rate of processing/output* — requests queue up and drain at a constant rate, smoothing bursts rather than allowing them through.

**Q6: What would you rate-limit by — IP, user ID, or API key?**

Depends on the threat model: IP-based limiting stops anonymous abuse but punishes everyone behind a shared NAT/proxy; user-ID-based limiting is fair per-account but requires authentication to have already happened; API-key-based limiting is standard for B2B APIs where you want to enforce per-customer quotas/tiers. Many systems combine layers (a loose IP limit + a strict per-user limit).
