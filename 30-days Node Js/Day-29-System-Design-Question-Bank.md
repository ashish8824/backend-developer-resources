# Day 29 — System Design Frameworks + The High-Frequency Question Bank

> **Why this day matters:** You now have the knowledge. Today is about **retrieval under pressure** — having a repeatable framework so you never freeze on a vague system design prompt, and having rapid-fire familiarity with the questions that come up again and again across real Node.js backend interviews. This is a reference day: less new theory, more structured practice.

---

## 1. A Repeatable Framework for "Design X" Questions

System design questions feel open-ended and scary specifically because candidates don't have a structure to fall back on. Use this exact sequence every time — it works for "design a rate limiter," "design a URL shortener," "design a notification system," or almost any variant.

### Step 1: Clarify Requirements (always start here — never jump to a solution)

Ask 2-3 sharp questions before designing anything. This alone signals seniority — junior candidates often start drawing boxes immediately.

- **Functional requirements:** "What are the core things this system must DO?" (e.g., for a URL shortener: create a short URL, redirect to the original, maybe track click counts)
- **Scale:** "Roughly what traffic/data volume are we talking about?" (100 requests/day vs. 100,000/second leads to very different designs)
- **Non-functional priorities:** "Is this more read-heavy or write-heavy? Is strong consistency required, or is eventual consistency acceptable?"

### Step 2: High-Level Architecture (the big boxes, before any detail)

Sketch the major components and how data flows between them — client, API layer, database, cache, any background processing — without committing to deep technical choices yet.

### Step 3: Deep Dive on the Interesting Parts

Pick the 1-2 components that actually matter for THIS specific problem and go deep — this is where your Week 3/4 knowledge (Redis, queues, rate limiting algorithms, caching, load balancing) gets applied.

### Step 4: Identify Bottlenecks and Tradeoffs

Proactively name what would break first under scale, and what you'd do about it — this is the step that most distinguishes a strong answer, since it shows you're thinking past "it works" toward "it works reliably at scale."

### Step 5: Wrap Up By Naming What You'd Do With More Time

Briefly mention what you'd explore further (monitoring, specific edge cases, a more detailed data model) — shows awareness without needing to solve everything in the room.

---

## 2. Worked Example: "Design a URL Shortener"

**Step 1 — Clarify:** "Should short URLs be permanent, or can they expire? Do we need click analytics? What's the expected scale — are we talking thousands or billions of URLs?" (Assume: permanent URLs, basic click tracking, expect significant scale.)

**Step 2 — High-level architecture:**
```
Client ──> Load Balancer (Day 23) ──> API servers (Node, clustered -- Day 15)
                                              │
                          ┌───────────────────┼───────────────────┐
                          ▼                                       ▼
                   Redis Cache (Day 17)                    Database (Day 11)
                   (hot URL lookups)                  (short_code -> long_url mapping)
```

**Step 3 — Deep dive (the interesting part: generating short codes):**
- Option A: hash the long URL (e.g., MD5) and take the first 7 characters — risk of collisions, needs a collision-check-and-retry strategy.
- Option B: a counter-based approach (auto-incrementing ID, converted to base62 for compactness) — guarantees uniqueness without collision checking, but requires a centrally coordinated counter (a potential bottleneck/single point of contention at extreme scale, solvable with techniques like pre-allocated ID ranges per server).
- Mention: redirects (`GET /:shortCode`) are overwhelmingly READ-heavy compared to creation — this directly justifies aggressive caching (Day 17's cache-aside pattern) of short-code-to-URL lookups in Redis, since the same popular short URLs get hit repeatedly.

**Step 4 — Bottlenecks:** the database read path for redirects is the highest-traffic component — caching (Day 17/24) absorbs most of this. The ID-generation step under extreme write concurrency could become a bottleneck — discuss pre-allocated ID ranges or a distributed ID generation approach (e.g., Twitter's Snowflake algorithm, worth knowing the NAME even without full implementation detail).

**Step 5 — Wrap up:** "With more time I'd dig into click-analytics data modeling, and whether we need geographic redirection/CDN-level caching (Day 24) for global low-latency redirects."

---

## 3. Worked Example: "Design a Rate Limiter" (your Day 22 knowledge, applied through this framework)

**Step 1 — Clarify:** "Is this for a single service or a public-facing API with multiple tiers? Multiple server instances?" (Assume: public API, multiple tiers, horizontally scaled.)

**Step 2 — High-level architecture:** API Gateway/middleware layer checking against a shared Redis store (Day 15/18's multi-instance problem, resolved by Redis) before forwarding to backend services.

**Step 3 — Deep dive:** walk through the algorithm choice (token bucket, with reasoning — Day 22), the atomicity concern under concurrency (Lua scripts/transactions), and identifying clients (API key + IP fallback).

**Step 4 — Bottlenecks:** Redis itself becoming a bottleneck or single point of failure at extreme scale (discuss Redis Cluster/replication briefly, or the fail-open/fail-closed tradeoff from Day 22 if Redis has an outage).

**Step 5 — Wrap up:** mention monitoring 429 rates and tuning limits based on real observed usage patterns.

---

## 4. Worked Example: "Design a Notification System" (a common variant worth a quick pass)

**Step 1 — Clarify:** "What channels — email, push, SMS, in-app? Real-time or can there be delay? What's the volume?"

**Step 2 — High-level architecture:**
```
Event source (e.g., order placed) ──> Message Queue (Day 19 — BullMQ/RabbitMQ)
                                              │
                       ┌──────────────────────┼──────────────────────┐
                       ▼                       ▼                       ▼
                Email Worker            Push Worker              SMS Worker
                (each consumes from its own queue/topic, scales independently)
```

**Step 3 — Deep dive:** why a queue instead of synchronous sending (Day 19 — decoupling, retry/persistence, not blocking the triggering request), idempotency for retried notification jobs (don't double-send), and — if real-time in-app notifications are needed — WebSockets/Socket.io (Day 20) with the Redis Adapter for cross-instance delivery.

**Step 4 — Bottlenecks:** a slow/down third-party email provider shouldn't block push/SMS sending — separate queues per channel allow independent scaling and isolate failures (this is a direct microservices-style decoupling argument from Day 25, even within a single system).

**Step 5 — Wrap up:** mention user notification preferences/opt-outs as a data modeling consideration worth exploring further.

---

## 5. The High-Frequency Question Bank — Rapid Recall Across All 28 Days

Use this as a speed drill: read each question, answer OUT LOUD in under 30 seconds, then check yourself against the day it came from if unsure.

**Fundamentals (Days 1-7)**
1. Is Node.js single-threaded? (Day 2)
2. Explain the event loop's priority order: nextTick, microtasks, macrotasks. (Day 2)
3. CommonJS vs ES Modules — key differences? (Day 1)
4. What's callback hell, and how do Promises fix it? (Day 3)
5. What's the difference between `Promise.all` and `Promise.allSettled`? (Day 3)
6. What are the four types of streams? (Day 5)
7. What is backpressure? (Day 5)
8. What does Express give you over the raw `http` module? (Day 6)

**Express, REST, Databases, Auth (Days 8-14)**
9. Why does middleware order matter in Express? (Day 8)
10. 401 vs 403 — what's the difference? (Day 9)
11. Offset vs cursor pagination — tradeoffs? (Day 9)
12. Is MongoDB schema-less? Does Mongoose change that? (Day 10)
13. What's the N+1 query problem? (Day 11)
14. What is a database transaction, and what does Atomicity mean? (Day 11)
15. Is a JWT encrypted? (Day 12)
16. Sessions vs JWT — core tradeoff? (Day 12)
17. What's the difference between an operational error and a programmer bug? (Day 13)
18. Is CORS a server-side security feature? (Day 13)

**Advanced Node, Redis, Queues, WebSockets, Testing (Days 15-21)**
19. Difference between `cluster` and Worker Threads? (Day 15)
20. Do cluster workers share memory? (Day 15)
21. Difference between `spawn` and `exec`? (Day 16)
22. Why is Redis fast? (Day 17)
23. Explain the cache-aside pattern. (Day 17)
24. Why is Pub/Sub unreliable for critical messages? (Day 18)
25. Why can't in-memory rate limiting work correctly across multiple instances? (Day 18)
26. What is idempotency, and why does it matter for job queues? (Day 19)
27. How do you scale WebSocket broadcasting across multiple server instances? (Day 20)
28. Unit vs integration tests — what's actually different? (Day 21)

**System Design & DevOps (Days 22-28)**
29. Compare token bucket and fixed window rate limiting. (Day 22)
30. Round robin vs least connections load balancing? (Day 23)
31. Why might more caching layers make invalidation HARDER? (Day 24)
32. What's a "distributed monolith"? (Day 25)
33. Container vs virtual machine — precise difference? (Day 26)
34. Why does Dockerfile instruction order matter? (Day 26)
35. How does `pm2 reload` achieve zero downtime? (Day 27)
36. What happens to in-flight requests without `SIGTERM` handling? (Day 27)
37. What's a memory leak in JS, given there's no manual memory management? (Day 28)

**If any of these made you pause for more than a few seconds, that's your signal — go back to that specific day's file and re-read just that section, then come back and try the question again.**

---

## 6. A Note on Connecting Topics Across Days — interviewers reward this

Notice how many answers above reference EACH OTHER (Redis solving Day 15's clustering problem, Day 18's rate limiting AND Day 20's WebSockets AND Day 23's sessions all hitting the same multi-instance wall). **Explicitly drawing these connections in an interview is a strong signal of real understanding versus memorized facts.** If asked "why is Redis so commonly paired with Node.js backends," a strong answer doesn't just define Redis — it says something like: *"It solves the recurring problem of needing shared state across multiple processes or server instances — caching, sessions, rate limiting counters, and cross-instance WebSocket broadcasting all hit the same fundamental issue: in-memory state in one Node process isn't visible to another, and Redis is the common, simple fix for that across all of those use cases."* That single answer demonstrates command of FOUR different topics at once.

---

## 7. Self-Assessment Before Tomorrow's Mock Interview

Rate yourself honestly (1-5) on each area below. Anything below a 3 is worth a focused re-read tonight, BEFORE Day 30's mock interview:

- [ ] Event loop internals (Day 2)
- [ ] Async patterns and error handling (Day 3)
- [ ] Express middleware and REST design (Days 8-9)
- [ ] Database tradeoffs — SQL vs NoSQL, N+1, transactions (Days 10-11)
- [ ] Authentication internals — JWT, sessions, RBAC (Day 12)
- [ ] Cluster, Worker Threads, child_process distinctions (Days 15-16)
- [ ] Redis use cases — caching, pub/sub, rate limiting (Days 17-18)
- [ ] Message queues and idempotency (Day 19)
- [ ] WebSocket scaling (Day 20)
- [ ] Testing strategy and mocking (Day 21)
- [ ] Rate limiting algorithms (Day 22)
- [ ] Load balancing and caching at scale (Days 23-24)
- [ ] Microservices tradeoffs (Day 25)
- [ ] Docker fundamentals (Day 26)
- [ ] CI/CD and graceful shutdown (Day 27)
- [ ] Memory leaks and monitoring (Day 28)

---

## Tomorrow (Day 30) Preview
The final day: a full mock interview combining a live-coding-style exercise, a system-design walkthrough using today's framework, and behavioral/experience-based questions ("tell me about a time you debugged a production issue," "tell me about a tradeoff you had to make") — plus space to do a final targeted review of whichever boxes above you didn't check off.
