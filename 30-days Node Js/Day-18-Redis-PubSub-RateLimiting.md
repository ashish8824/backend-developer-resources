# Day 18 — Redis Part 2: Pub/Sub, Session Storage, and Rate Limiting

> **Why this day matters:** This is arguably the single highest-yield interview day in this entire course. Rate limiting with Redis is one of the most commonly asked **practical coding exercises** at the 3 YOE level — interviewers love it because it touches algorithms, concurrency, and real production thinking all at once. Pub/Sub and session storage round out the picture of "why does almost every real Node backend have a Redis instance sitting next to it."

---

## 1. Redis Pub/Sub — background and mental model

### Background: how is this different from the EventEmitter pattern from Day 4?

Recall Day 4's EventEmitter: a broadcaster emits, listeners react — but that ENTIRELY happens **within one Node process's memory**. Redis Pub/Sub applies the exact same broadcaster/listener mental model, but **across process and machine boundaries** — a publisher running in one Node process (or even a different server entirely) can notify subscribers running in completely different processes/servers, because Redis itself sits in the middle relaying messages.

```
┌────────────┐                                    ┌────────────┐
│ Publisher  │── PUBLISH 'order:created' {...} ──>│            │
│ (Server A) │                                    │   Redis    │
└────────────┘                                    │            │
                                                    │  (relays   │──> Subscriber 1 (Server B)
┌────────────┐                                    │  messages) │──> Subscriber 2 (Server C)
│ Publisher  │── PUBLISH 'order:created' {...} ──>│            │──> Subscriber 3 (same Server A!)
│ (Server A) │                                    └────────────┘
└────────────┘
```

```javascript
// subscriber.js
const { createClient } = require('redis');

async function subscribe() {
  const subscriber = createClient();
  await subscriber.connect();

  // Subscribing to a CHANNEL -- conceptually identical to EventEmitter's
  // .on('eventName', callback), but this listener can be in a COMPLETELY
  // SEPARATE process or server from whoever eventually publishes to it.
  await subscriber.subscribe('order:created', (message) => {
    const order = JSON.parse(message);
    console.log('New order received:', order);
    // e.g., send a notification email, update analytics, etc.
  });

  console.log('Listening for order:created events...');
}

subscribe();
```

```javascript
// publisher.js -- this could be running on a COMPLETELY different
// server/process than the subscriber above, and it would still work
const { createClient } = require('redis');

async function publishOrder() {
  const publisher = createClient();
  await publisher.connect();

  await publisher.publish('order:created', JSON.stringify({
    orderId: 123,
    total: 99.99,
  }));

  await publisher.quit();
}

publishOrder();
```

### A critical limitation interviewers expect you to know: Pub/Sub messages are NOT persisted

**This is the single most important fact about Redis Pub/Sub, and it's frequently tested:** if a subscriber is **not actively connected and listening** at the exact moment a message is published, that subscriber **misses the message permanently** — there is no replay, no queue, no "catch up on missed messages" capability built into basic Pub/Sub. This makes it fundamentally different from a proper **message queue** (Day 19's BullMQ, or RabbitMQ/Kafka), which DO persist messages until they're explicitly consumed/acknowledged.

**Interview tip — the precise distinction to state if asked "Pub/Sub vs a message queue":** *"Pub/Sub is fire-and-forget — great for real-time notifications where missing a message occasionally is acceptable (e.g., live chat presence updates, cache invalidation broadcasts across servers). A message queue persists messages until they're processed, with acknowledgment and retry mechanics, making it appropriate for tasks that absolutely must be processed eventually, even if a consumer was temporarily down."* Knowing this distinction, and choosing the right tool based on it, is a strong senior-level signal.

**A real, practical use case for Pub/Sub specifically:** invalidating an in-memory cache across MULTIPLE server instances simultaneously. If Server A updates some data and clears ITS local in-memory cache, Servers B and C have no idea — they'd keep serving stale local cache. Publishing an `'invalidate:user:42'` message lets every server instance (each subscribed) clear its own local cache the moment any one of them makes a change.

---

## 2. Redis as a Session Store — connecting Day 12's auth to Day 15's clustering problem

### Background: why does this matter again, specifically?

Recall Day 12's session-based auth: the server stores session data (often in memory by default with `express-session`) and a cookie just holds a pointer/ID. Recall Day 15's cluster problem: in-memory data isn't shared across multiple worker processes/server instances. **Put these two together, and you get a real, common production bug:** a user logs in (session created on Worker 1's in-memory store), but their NEXT request lands on Worker 2 (thanks to round-robin distribution) — which has never heard of that session and treats them as logged out. This is precisely why production apps using session-based auth almost always pair `express-session` with a **shared, external session store** — Redis being the overwhelmingly common choice.

```javascript
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient();
redisClient.connect();

app.use(session({
  store: new RedisStore({ client: redisClient }), // THE key change from Day 12's example --
                                                     // session data now lives in Redis, not
                                                     // in this specific worker process's memory
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { secure: true, httpOnly: true, maxAge: 24 * 60 * 60 * 1000 },
}));
```

**Now, no matter which cluster worker, or even which entirely separate server machine, handles a given request, the session lookup checks the SAME shared Redis store** — solving the exact problem described above. This is a genuinely important, frequently-tested piece of real-world architecture knowledge: **"how would you handle sessions in a horizontally scaled application?"** is close to a guaranteed question if session-based auth comes up at all in an interview.

---

## 3. Rate Limiting with Redis — the highest-value section of today

### Background: why rate limit at all?

Rate limiting protects your API from being overwhelmed — whether by a buggy client retrying too aggressively, a malicious actor attempting a brute-force attack or scraping your data, or simply more traffic than your infrastructure can currently handle. It's also how API providers (Stripe, GitHub, Twitter/X) enforce usage tiers (e.g., "free tier: 100 requests per hour").

### Why does rate limiting NEED a shared store like Redis (this connects directly to Day 15 again)?

If you tracked request counts in a plain in-memory JS object/Map, and you're running multiple cluster workers or server instances, **each one would have its own separate count** — a client could send 100 requests, have them distributed round-robin across 4 workers, and each worker would only see ~25 requests, never tripping any individual worker's limit even though the CLIENT's true total was over the intended threshold. Redis, being one shared store all workers/instances connect to, lets you track the TRUE global count for a given client correctly, regardless of which server instance handles any individual request.

### Algorithm 1: Fixed Window Counter (simplest)

```javascript
// Background: divide time into fixed windows (e.g., "this calendar
// minute"), count requests within the CURRENT window, reset to zero
// at the start of every new window.
async function fixedWindowRateLimit(client, identifier, limit, windowSeconds) {
  const key = `ratelimit:${identifier}:${Math.floor(Date.now() / (windowSeconds * 1000))};`
  // ^ the key itself CHANGES every windowSeconds, naturally "resetting" the counter

  const count = await client.incr(key);

  if (count === 1) {
    // Only the FIRST request in this window needs to set an expiration --
    // this ensures the key (and its counter) is automatically cleaned up
    // by Redis once the window passes, with no manual cleanup needed.
    await client.expire(key, windowSeconds);
  }

  return count <= limit; // true = request allowed, false = rate limited
}

// Usage in Express middleware
async function rateLimitMiddleware(req, res, next) {
  const identifier = req.ip; // or req.user.userId for authenticated rate limits
  const allowed = await fixedWindowRateLimit(redisClient, identifier, 100, 60); // 100 req/min

  if (!allowed) {
    return res.status(429).json({ error: 'Too many requests, please try again later' });
  }
  next();
}
```

**The real weakness of fixed windows — a frequently-asked follow-up:** a client could send 100 requests in the LAST second of one window, then immediately send another 100 requests in the FIRST second of the next window — that's 200 requests within roughly 1-2 seconds, despite the limit being "100 per minute." This is called the **"boundary burst" problem**, and it's exactly why more sophisticated algorithms exist.

### Algorithm 2: Sliding Window Log (more accurate, more expensive)

```javascript
// Background: instead of a fixed window, track the EXACT TIMESTAMP of
// every individual request, and only count requests within the last
// N seconds FROM RIGHT NOW (a continuously moving window, not a fixed
// calendar-aligned one) -- this eliminates the boundary burst problem
// entirely, at the cost of storing more data (one entry per request,
// not just one counter).
async function slidingWindowRateLimit(client, identifier, limit, windowSeconds) {
  const key = `ratelimit:sliding:${identifier}`;
  const now = Date.now();
  const windowStart = now - windowSeconds * 1000;

  // Using a Sorted Set (Day 17!) with the TIMESTAMP as the score --
  // this lets us efficiently remove old entries and count current ones.
  await client.zRemRangeByScore(key, 0, windowStart); // remove anything OUTSIDE the current window
  const currentCount = await client.zCard(key);        // count what's left (i.e., within the window)

  if (currentCount < limit) {
    await client.zAdd(key, [{ score: now, value: `${now}-${Math.random()}` }]); // record THIS request
    await client.expire(key, windowSeconds); // cleanup safety net
    return true;
  }

  return false; // limit exceeded
}
```

**Why this is more accurate, explained precisely:** because it always looks at "the last N seconds from THIS EXACT moment," not "the current fixed calendar-aligned window," there's no boundary to burst across — the count is always a true reflection of recent activity. The tradeoff: it stores one entry PER REQUEST (within the window) rather than just a single number, using more memory for high-traffic identifiers.

### Algorithm 3: Token Bucket (a very common, more nuanced approach worth knowing by name)

**Background, conceptually (without full implementation, since it's more complex):** imagine a bucket that holds a maximum number of "tokens" (e.g., 10). Tokens are added back at a steady rate (e.g., 1 per second) up to the bucket's maximum capacity. Each request consumes ONE token; if the bucket is empty, the request is rejected. This naturally allows brief BURSTS of traffic (using up multiple saved-up tokens at once) while still enforcing a steady average rate over time — a more realistic model of how real traffic actually behaves (bursty, not perfectly smooth) compared to fixed or sliding windows.

```javascript
// A simplified token bucket using Redis -- storing the current token
// count and the last refill timestamp, recalculating refills lazily
// each time a request comes in (rather than running a separate
// continuous background process to add tokens every second).
async function tokenBucketRateLimit(client, identifier, capacity, refillRatePerSecond) {
  const key = `ratelimit:bucket:${identifier}`;
  const now = Date.now();

  const bucket = await client.hGetAll(key);
  let tokens = bucket.tokens ? parseFloat(bucket.tokens) : capacity;
  let lastRefill = bucket.lastRefill ? parseFloat(bucket.lastRefill) : now;

  // Calculate how many tokens should have been added since the last check
  const elapsedSeconds = (now - lastRefill) / 1000;
  tokens = Math.min(capacity, tokens + elapsedSeconds * refillRatePerSecond);

  if (tokens >= 1) {
    tokens -= 1; // consume one token for this request
    await client.hSet(key, { tokens: tokens.toString(), lastRefill: now.toString() });
    return true;
  }

  await client.hSet(key, { tokens: tokens.toString(), lastRefill: now.toString() });
  return false; // no tokens available -- rate limited
}
```

**Interview tip — knowing all three algorithm NAMES and their core tradeoff is often enough**, even without writing the token bucket implementation perfectly from memory under pressure: *"Fixed window is simplest but allows boundary bursts. Sliding window log is precise but uses more memory. Token bucket allows controlled burstiness while maintaining a steady average rate, and is what many real-world APIs (including AWS and Stripe) conceptually use."* This framing alone demonstrates strong comparative understanding.

### Using a library instead of hand-rolling this in production

```javascript
// In real production code, you'd typically use a battle-tested library
// rather than hand-rolling rate limiting logic -- but understanding the
// algorithm underneath (as built above) is exactly what interviews probe,
// since libraries hide these details.
const { RateLimiterRedis } = require('rate-limiter-flexible');

const rateLimiter = new RateLimiterRedis({
  storeClient: redisClient,
  points: 100,     // max requests
  duration: 60,    // per 60 seconds
});

async function rateLimitMiddleware(req, res, next) {
  try {
    await rateLimiter.consume(req.ip);
    next();
  } catch (rejRes) {
    res.status(429).json({ error: 'Too many requests' });
  }
}
```

---

## 4. How this connects to real backend work (3 YOE framing)

- **"How would you implement rate limiting for a public API with multiple server instances behind a load balancer?"** → Must use a shared store (Redis), not in-memory counters, exactly because of the Day 15 multi-instance problem — then pick an algorithm based on precision needs (fixed window for simplicity, sliding window/token bucket for more accurate burst handling).
- **"You're scaling your app horizontally and users keep getting logged out randomly — what's the likely cause?"** → Session data stored in-memory per-instance instead of in a shared Redis store, causing requests landing on a different instance to not recognize the existing session.
- **"How would you notify all running instances of your app to clear a specific cached value the moment it changes?"** → Redis Pub/Sub — publish an invalidation event that every instance subscribes to and reacts to independently.
- **"What's the tradeoff between Redis Pub/Sub and a proper message queue for processing background jobs?"** → Pub/Sub is fire-and-forget with no persistence or retry — a missed message is lost forever. A message queue (Day 19) persists jobs until acknowledged, making it the correct choice for anything that must reliably be processed.

---

## 5. Hands-on practice for today

```javascript
// practice-day18-ratelimit.js
const { createClient } = require('redis');
const express = require('express');

async function main() {
  const client = createClient();
  await client.connect();

  const app = express();

  async function rateLimitMiddleware(req, res, next) {
    const identifier = req.ip;
    const limit = 5;
    const windowSeconds = 30;
    const key = `ratelimit:${identifier}:${Math.floor(Date.now() / (windowSeconds * 1000))}`;

    const count = await client.incr(key);
    if (count === 1) await client.expire(key, windowSeconds);

    res.setHeader('X-RateLimit-Limit', limit);
    res.setHeader('X-RateLimit-Remaining', Math.max(0, limit - count));

    if (count > limit) {
      return res.status(429).json({ error: 'Too many requests, try again shortly' });
    }
    next();
  }

  app.get('/api/data', rateLimitMiddleware, (req, res) => {
    res.json({ message: 'Here is your data', timestamp: Date.now() });
  });

  app.listen(3000, () => console.log('Running on http://localhost:3000'));
}

main();
```

Test it by hitting `/api/data` more than 5 times within 30 seconds (`for i in {1..7}; do curl http://localhost:3000/api/data; echo; done`) and watch the 6th and 7th requests get a 429 — then wait 30 seconds and confirm it resets.

---

## 6. Interview Q&A Recap (say these out loud)

1. **Q: How is Redis Pub/Sub different from Node's EventEmitter?**
   A: EventEmitter operates entirely within one Node process's memory. Redis Pub/Sub broadcasts messages across process and machine boundaries through a shared Redis instance, letting publishers and subscribers be entirely separate processes/servers.

2. **Q: What happens to a Pub/Sub message if no subscriber is listening at the moment it's published?**
   A: It's lost permanently — basic Redis Pub/Sub doesn't persist or queue messages, unlike a proper message queue which retains messages until they're acknowledged/processed.

3. **Q: Why does session-based authentication need a shared store like Redis in a horizontally scaled app?**
   A: Without it, session data created on one server instance/cluster worker isn't visible to other instances, so a user's subsequent requests landing on a different instance would appear unauthenticated — a shared Redis-backed session store fixes this by giving every instance the same source of truth.

4. **Q: Why can't you implement accurate rate limiting using a simple in-memory counter in a multi-instance deployment?**
   A: Each instance/worker would maintain its own separate count, so a client's true total request volume across all instances could exceed the intended limit without any single instance's local counter ever reaching it — Redis provides the necessary shared, accurate count.

5. **Q: What's the weakness of fixed-window rate limiting, and what fixes it?**
   A: It allows a "boundary burst" — a client can send the full limit right before a window resets and again right after, effectively doubling allowed traffic briefly. Sliding window or token bucket algorithms address this by not resetting abruptly at fixed calendar-aligned boundaries.

6. **Q: Briefly describe the token bucket algorithm.**
   A: A bucket holds a maximum number of tokens, refilled at a steady rate over time; each request consumes one token, and requests are rejected when the bucket is empty — this allows controlled bursts of traffic while maintaining a steady average rate over time.

---

## Tomorrow (Day 19) Preview
We cover **message queues** properly: BullMQ (Redis-based job queues), and a conceptual introduction to RabbitMQ/Kafka — building directly on today's Pub/Sub limitations by introducing tools designed specifically for reliable, persistent, retry-capable background job processing.
