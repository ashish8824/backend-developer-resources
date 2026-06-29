# Day 22 — Rate Limiting Algorithms: The System Design Deep Dive

> **Why this day matters:** Day 18 implemented rate limiting with Redis as part of learning Redis. Today revisits the SAME topic from a different angle — as a **system design interview question** in its own right, the kind asked as "design a rate limiter" with 30-45 minutes on a whiteboard or doc, expecting you to discuss tradeoffs, distributed considerations, and where to place the limiter in your architecture — not just write the code. This is one of the most commonly asked system design questions specifically for backend roles, and a focused second pass cements it for that exact interview format.

---

## 1. Background: reframing rate limiting as a system design problem

When Day 18 covered this, the framing was "how do I implement this in my Express app." Today's framing is broader: **"design a rate limiter"** as a standalone system design question expects you to think about:
- WHERE in the architecture the rate limiter lives (application code? a dedicated middleware layer? the load balancer/API gateway?)
- WHAT identifies a "client" being limited (IP address? API key? user ID? a combination?)
- HOW the limit is communicated back to the client
- HOW the system behaves under distributed/multi-instance conditions (directly building on Day 15 and Day 18)
- WHAT happens when the rate limiting STORE itself (e.g., Redis) is slow or down

This broader framing is exactly what separates "I implemented rate limiting once" from "I can design a rate limiting system" — the latter is what's actually being tested.

---

## 2. The Four Algorithms — A Clean, Comparative Pass

This is a deliberate re-explanation from Day 18, framed for clean whiteboard/verbal delivery rather than code-first. If asked to explain these in an interview without immediately reaching for a keyboard, this is how to structure the answer.

### Fixed Window Counter

**The idea:** divide time into fixed-size windows (e.g., every distinct 60-second block aligned to the clock), count requests within the current window, reject once the count exceeds the limit, reset to zero at each new window boundary.

**Pros:** simplest to implement and reason about; very low memory (one counter per client).
**Cons:** the boundary burst problem (Day 18) — up to 2x the intended limit can pass through right at a window boundary.

### Sliding Window Log

**The idea:** store the exact timestamp of every request; on each new request, discard timestamps older than the window size, then count what remains.

**Pros:** perfectly accurate — no boundary burst issue at all, since the window is always "the last N seconds from right now," continuously moving.
**Cons:** memory scales with request volume (one stored entry per request within the window) — expensive for high-traffic clients/endpoints.

### Sliding Window Counter (a hybrid worth knowing — wasn't fully covered Day 18)

**The idea:** a middle ground between fixed window's cheapness and sliding window log's accuracy. Keep TWO fixed-window counters (the current window and the previous one), and estimate the "true" sliding count using a weighted combination — e.g., if you're 30% of the way through the current window, count = (current window's count) + (70% of the previous window's count). This approximates a true sliding window without storing every individual timestamp.

**Pros:** much cheaper than sliding window log (just two counters, not N timestamps), while still substantially reducing the boundary burst problem compared to plain fixed window.
**Cons:** an approximation, not perfectly precise — acceptable for the vast majority of real-world use cases, which is exactly why it's a popular production choice (Cloudflare's public rate limiting documentation describes using this exact approach).

### Token Bucket

**The idea (restated cleanly):** a bucket holds up to `capacity` tokens; tokens refill at a steady `rate` over time; each request consumes one token; requests are rejected when the bucket is empty.

**Pros:** naturally allows bursts (a client that's been idle can suddenly use up its full saved capacity at once) while still enforcing a steady average rate over time — this matches REAL traffic patterns better than a rigid window, since legitimate clients are naturally bursty (e.g., a user might load a page that fires 8 API calls at once, then go idle for a while).
**Cons:** slightly more complex to implement and reason about than fixed window.

### Leaky Bucket (a related, sometimes-confused alternative worth distinguishing)

**The idea:** similar bucket metaphor, but inverted in spirit — requests are added to a queue (the "bucket"), and they're processed ("leaked out") at a STEADY, constant rate, regardless of how fast they arrived. If the queue/bucket overflows (too many requests arrive faster than they can leak out), new requests are dropped.

**Key distinction from token bucket:** token bucket controls the RATE of ALLOWING requests through (with burst tolerance); leaky bucket controls the RATE of PROCESSING requests, smoothing out bursts into a steady, constant outflow rather than allowing burst-through. Token bucket is far more common for API rate limiting specifically; leaky bucket shows up more in network traffic shaping contexts. Knowing this distinction exists, even briefly, signals broader awareness.

### The Comparison Table (the one to have memorized cold for this exact interview question)

| Algorithm | Accuracy | Memory cost | Allows bursts? | Common real-world use |
|---|---|---|---|---|
| Fixed Window | Low (boundary burst) | Very low | Yes (at boundaries, unintentionally) | Simple internal tools, low-stakes limits |
| Sliding Window Log | Perfect | High (scales with traffic) | No | Low-traffic, high-precision needs |
| Sliding Window Counter | Good approximation | Low | Slightly, bounded | Cloudflare and similar large-scale production systems |
| Token Bucket | Good | Low | Yes, by design (intentional, controlled) | AWS API Gateway, Stripe API, most modern public APIs |
| Leaky Bucket | Good | Low | No (smooths everything to a constant rate) | Network/traffic shaping, less common for HTTP APIs specifically |

---

## 3. Where Does the Rate Limiter Live? — the architecture placement question

**This is the part many engineers skip, but it's exactly what a system design interview wants you to discuss.**

```
Client ──> [API Gateway / Load Balancer] ──> [Your Node.js App] ──> Database
              ▲
              │
        Rate limiting could live HERE (gateway level)
        OR inside your app as middleware (application level)
        OR both (defense in depth)
```

**Option 1: Application-level (Express middleware, as built on Day 18)**
- Pros: full access to application context (e.g., can rate limit by authenticated `userId`, not just IP; can apply different limits per route based on business logic).
- Cons: the request has already consumed some server resources (connection handling, routing) before being rejected; doesn't protect against truly massive volumetric attacks aimed at simply overwhelming your servers' network capacity.

**Option 2: API Gateway / Load Balancer level (e.g., AWS API Gateway, Kong, Nginx, Cloudflare)**
- Pros: rejects excessive traffic BEFORE it even reaches your application servers, protecting backend capacity more effectively; often has DDoS-aware infrastructure behind it.
- Cons: less access to fine-grained application context (may only have IP/API key, not deep business logic like "this specific user's subscription tier").

**The strong, senior-level answer when asked "where would you implement rate limiting":** *"Often both, serving different purposes — a coarse-grained limit at the gateway/load balancer level to protect overall infrastructure capacity from abuse or attacks, and a finer-grained, business-logic-aware limit at the application level (e.g., different limits per subscription tier, or per specific endpoint based on its cost to compute)."* This "defense in depth" framing — multiple layers, each doing a job suited to its position — is a recurring theme in good system design answers generally, not just for rate limiting.

---

## 4. Identifying "Who" to Rate Limit

```javascript
// By IP address -- simplest, but has real weaknesses worth naming:
// many users can share ONE public IP (corporate networks, NAT, mobile
// carriers), unfairly limiting all of them together as if they were one
// client; conversely, a single attacker can rotate through many IPs to
// evade IP-based limits entirely.
const identifier = req.ip;

// By API key / authenticated user ID -- much more precise and fair,
// but only works for AUTHENTICATED requests -- you still need a
// fallback (often IP-based) for unauthenticated/public endpoints.
const identifier = req.user?.userId || req.ip;

// By a COMBINATION (often the most robust real-world approach) --
// e.g., limit by API key AND by IP, catching both "one compromised key
// being abused from many machines" and "one IP hammering many endpoints"
```

**Interview tip:** if asked "how do you decide what to rate limit by," the precise answer should acknowledge there's no single universally correct identifier — it depends on what you're protecting against (abuse from one account vs. volumetric attacks from one source) and whether the endpoint requires authentication at all.

---

## 5. Communicating Limits Back to the Client — REST conventions (Day 9 callback)

```javascript
// Standard, widely-adopted headers (used by GitHub, Stripe, and many
// other major APIs) that tell the client their current rate limit
// status PROACTIVELY, so well-behaved clients can self-throttle BEFORE
// hitting the limit and getting rejected.
res.setHeader('X-RateLimit-Limit', 100);       // total allowed in this window
res.setHeader('X-RateLimit-Remaining', 42);     // how many are left
res.setHeader('X-RateLimit-Reset', 1719240000); // when the window resets (Unix timestamp)

// When actually rejecting a request:
res.status(429).json({ error: 'Too many requests' });
res.setHeader('Retry-After', 30); // tells the client exactly how many
                                    // seconds to wait before retrying --
                                    // a well-behaved client should respect this
```

**Interview tip:** mentioning the `Retry-After` header specifically is a small but real signal of API design maturity — it shows you're thinking about how a GOOD client should behave in response to your rate limiting, not just how your server enforces it.

---

## 6. The Distributed System Edge Cases — what a strong candidate brings up unprompted

### What happens if Redis (the shared rate-limit store) goes down or becomes slow?

This is a genuinely important question a strong candidate raises proactively: **should your rate limiter FAIL OPEN (allow all requests through if Redis is unreachable) or FAIL CLOSED (reject all requests if Redis is unreachable)?** There's no universally correct answer — it's a real tradeoff:
- **Fail open:** prioritizes availability — your API stays usable even if the rate limiter's backing store is down, at the cost of temporarily having NO rate limiting protection during that outage.
- **Fail closed:** prioritizes protection — but now a Redis outage takes down your ENTIRE API's availability too, even though the actual application logic/database might be perfectly healthy.

```javascript
async function rateLimitMiddleware(req, res, next) {
  try {
    const allowed = await checkRateLimit(req.ip); // involves a Redis call
    if (!allowed) return res.status(429).json({ error: 'Too many requests' });
    next();
  } catch (err) {
    // Redis is down/unreachable -- a DELIBERATE choice point:
    console.error('Rate limiter store unavailable:', err);
    next(); // FAIL OPEN: prioritizing availability over rate limit enforcement here
    // (the alternative: `return res.status(503).json({ error: 'Service temporarily unavailable' });`
    //  would be FAILING CLOSED instead)
  }
}
```

**Most real production systems lean toward failing open for rate limiters specifically** (since the rate limiter's job is to prevent overload, not to BE a cause of an outage), but explicitly naming this tradeoff and that it's a deliberate engineering decision — rather than an accident of how the code happens to be written — is exactly the kind of thinking system design interviews are probing for.

### A subtlety with the token bucket algorithm specifically, at scale

If you're running the token bucket logic across multiple Node instances all hitting the SAME Redis-backed bucket, you need the refill calculation + token consumption to happen as a single ATOMIC operation (recall Day 17's `INCR` atomicity lesson) — otherwise two concurrent requests could both read the bucket as having 1 token available and both consume it, allowing one extra request through. In real implementations, this is often handled with a **Redis Lua script** (allowing multiple Redis operations to execute as one atomic unit, since Redis guarantees a Lua script runs without interruption from other commands) rather than separate GET/SET calls from your application code.

```javascript
// A simplified illustration of WHY this matters (not a full Lua script
// implementation, since writing real Lua is beyond today's scope, but
// understanding WHY it's needed is the actual interview-relevant point)

// PROBLEM: separate read-then-write calls from Node have a race window
const bucket = await redis.hGetAll(key); // READ
// ... time passes, ANOTHER request could run its own read here too ...
await redis.hSet(key, { tokens: newTokenCount }); // WRITE -- might overwrite
// concurrent work based on stale data read by another request

// THE FIX (conceptually): a Lua script bundles the read-calculate-write
// into ONE atomic step that Redis executes without any other command
// being able to interleave in the middle of it.
```

**Interview tip:** you don't need to write real Lua code in most interviews — being able to articulate the RACE CONDITION risk and that "a Lua script or Redis transaction (`MULTI`/`EXEC`) would be needed to make this atomic at scale" is the level of depth expected, not full implementation.

---

## 7. How this connects to real backend work (3 YOE framing)

- **"Design a rate limiter for a public API"** → Walk through: identify by API key (with IP fallback), choose token bucket (or sliding window counter) for the algorithm with a clear reason why, place it at both the gateway and application level, communicate limits via standard headers, and proactively raise the fail-open/fail-closed tradeoff for when the backing store is unavailable.
- **"Your rate limiter works fine with low traffic but seems to let through MORE requests than the configured limit under high concurrent load" → why?"** → Likely a non-atomic read-modify-write pattern in the rate limiting logic (the token bucket race condition above), fixable with a Redis Lua script or transaction.
- **"How would you give certain premium users a higher rate limit than free users?"** → Identify by authenticated `userId` rather than IP, and look up the user's tier (cached, ideally, to avoid a DB call on every single rate-limit check) to determine which limit configuration applies.
- **"What would you monitor to know if your rate limiting is working correctly in production?"** → Rate of 429 responses over time (sudden spikes might indicate an attack OR a misconfigured limit hurting legitimate users), and correlating rate-limited IPs/users with other signals (is this a legitimate power user being throttled too aggressively, or an actual abuse pattern).

---

## 8. Mock System Design Round — Practice Explaining This Out Loud

Set a timer for 5 minutes and explain your answer to this out loud, as if to an interviewer, before checking the discussion points above:

> "Design a rate limiter for an API that needs to support both free-tier and paid-tier users, where paid users get a higher limit. The system runs on multiple server instances behind a load balancer."

A strong answer touches: identifying the user (auth-based, with tier lookup), the algorithm choice (token bucket, with reasoning), the NEED for a shared store (Redis, citing the Day 15/18 multi-instance problem explicitly), the atomicity concern under concurrency, where in the architecture this lives, how limits are communicated to the client, and the fail-open/fail-closed tradeoff if Redis has an outage.

---

## 9. Interview Q&A Recap (say these out loud)

1. **Q: Compare token bucket and leaky bucket.**
   A: Token bucket controls how many requests are ALLOWED through, naturally permitting bursts up to the bucket's capacity while maintaining a steady average refill rate. Leaky bucket controls the rate requests are PROCESSED/output, smoothing everything into a constant outflow rate and dropping overflow — token bucket is far more common for API rate limiting specifically.

2. **Q: What is the sliding window counter, and why might a large-scale system prefer it over a sliding window log?**
   A: It approximates a true sliding window using just two fixed-window counters (current and previous) with a weighted estimate, avoiding the high per-request memory cost of storing every individual timestamp (as sliding window log does), while still substantially reducing the boundary burst problem of plain fixed windows.

3. **Q: Where in your system's architecture would you place rate limiting, and why might you use more than one layer?**
   A: At the gateway/load balancer level for coarse-grained protection of overall infrastructure capacity, and at the application level for finer-grained, business-logic-aware limits (e.g., per subscription tier or per endpoint cost) — using both provides defense in depth, each layer suited to what it can see and protect.

4. **Q: What's the fail-open vs fail-closed tradeoff for a rate limiter, and which is more common in practice?**
   A: Fail-open allows all requests through if the rate limiter's backing store (e.g., Redis) becomes unreachable, prioritizing API availability over enforcement during that outage. Fail-closed rejects all requests in that scenario, prioritizing protection but making the rate limiter itself a single point of failure for the whole API. Most production systems lean toward failing open for rate limiters specifically.

5. **Q: Why can token bucket logic break under high concurrency without extra care, and how do you fix it?**
   A: A naive read-then-write implementation has a race window where two concurrent requests could both read the bucket as having enough tokens and both succeed, over-allowing requests. Fixing this requires making the read-calculate-write sequence atomic, typically via a Redis Lua script or a Redis transaction (`MULTI`/`EXEC`).

6. **Q: What does the `Retry-After` header communicate, and why include it?**
   A: It tells a rate-limited client exactly how many seconds to wait before retrying, allowing well-behaved clients to back off appropriately instead of immediately retrying and potentially making the situation worse.

---

## Tomorrow (Day 23) Preview
We cover **load balancing**: the core concepts (horizontal vs vertical scaling), common load balancing algorithms (round robin, least connections, IP hash), using Nginx as a reverse proxy/load balancer in front of multiple Node instances, health checks, and the sticky sessions concept that was foreshadowed back on Day 20's WebSocket scaling discussion.
