# Module 18: Interview Questions (Full Compilation)

**Level:** Advanced

You've now covered the whole course. This module compiles the interview questions from every module, plus additional common ones, all in one place for revision — organized by topic, with concise model answers.

---

## 1. Fundamentals

**Q: What is Redis, and why is it used?**
A: Redis is an in-memory, key-value data store that also supports message brokering (Pub/Sub). It's used because RAM access is dramatically faster than disk access, making it ideal as a caching layer, session store, and for other latency-sensitive tasks in front of a slower primary database.

**Q: Why is Redis so fast?**
A: Data lives in RAM, most operations are O(1) or O(log N), it's implemented in C with purpose-built data structures per type, and it uses an efficient, non-blocking, single-threaded event loop that avoids locking overhead.

**Q: When would you NOT use Redis?**
A: When data must be immediately durable and strongly consistent (e.g., financial ledgers — use an ACID database), when the dataset is too large to affordably fit in RAM, or when you need complex relational queries/joins.

---

## 2. Data Types

**Q: What data types does Redis support, and when would you use each?**
A: String (simple values, counters, tokens), List (ordered collections, feeds, simple queues), Hash (object-like records), Set (unique unordered collections, membership checks), Sorted Set (ranked/scored data like leaderboards). Also more specialized types: Streams (event logs), Bitmaps (compact flags), HyperLogLog (approximate counting), Geo (location queries).

**Q: What's the difference between a Set and a Sorted Set?**
A: A Set stores unique unordered members with fast O(1) membership checks. A Sorted Set attaches a numeric score to each member and keeps them automatically sorted by that score, enabling fast ranked range queries (e.g., "top 10").

**Q: Why use a Hash instead of storing a JSON string in a regular key?**
A: A Hash lets you read/update individual fields directly and atomically (e.g., just increment `age`) without fetching and rewriting the entire object, and is generally more memory-efficient for many small related fields than many separate top-level keys.

---

## 3. Internals & Architecture

**Q: Is Redis single-threaded? How does it handle many clients at once then?**
A: Yes, core command execution is single-threaded, which avoids locking overhead since commands can't interleave. It handles many simultaneous clients using a non-blocking, event-driven I/O model (similar to Node.js's event loop), efficiently monitoring many connections without a thread per connection.

**Q: Why is running `KEYS *` dangerous in production?**
A: It forces Redis's single command-processing thread to scan the entire keyspace in one go, blocking all other clients until it finishes — which can take a noticeable amount of time on large datasets. `SCAN` should be used instead, since it works incrementally without blocking.

---

## 4. Memory & Persistence

**Q: What happens when Redis runs out of memory?**
A: Depends on `maxmemory-policy`. With `noeviction` (default), further writes are rejected with an OOM error while reads still work. With an eviction policy like `allkeys-lru`, Redis automatically removes keys according to that policy's rule to free up space for new writes.

**Q: What's the difference between RDB and AOF persistence?**
A: RDB takes periodic full snapshots of the dataset — compact and fast to restore, but can lose data since the last snapshot. AOF logs every write command as it happens — much safer (as little as ~1 second of data loss with `appendfsync everysec`), but produces larger files (though periodically compacted). Production systems often use both together.

**Q: How would you choose an eviction policy?**
A: If Redis is a pure, fully-regenerable cache, use `allkeys-lru` or `allkeys-lfu`. If Redis mixes permanent-ish and cache data, use `volatile-lru` and ensure only cache keys have a TTL. If losing data is unacceptable, use `noeviction` and handle write failures gracefully in the app.

---

## 5. Application Integration

**Q: Why use a single shared Redis connection instead of creating one per request?**
A: Establishing connections has real overhead (handshake, auth), and creating one per request would be slow and could exhaust Redis's connection limits under real traffic — a single, long-lived, reused connection (or small pool) is the standard efficient pattern, just like with database connections.

**Q: How should your app behave if Redis becomes unavailable?**
A: It should degrade gracefully, not crash. Wrap Redis calls in try/catch, fall back to querying the database directly (accepting reduced performance), and use client-level retry/reconnect logic — Redis should be treated as an accelerator, not a single point of failure for correctness.

---

## 6. Caching

**Q: Explain the Cache-Aside pattern.**
A: On a request, check Redis first. If found (cache hit), return it directly. If not found (cache miss), query the database, store the result in Redis with a TTL, then return it. The application code manages the cache; Redis itself is unaware of the database.

**Q: How do you handle cache invalidation when underlying data changes?**
A: Either let the TTL naturally expire the stale entry (fine when brief staleness is acceptable), or actively delete/update the cache entry as part of the same operation that updates the source-of-truth database, so subsequent reads get fresh data immediately.

**Q: What is a cache stampede, and how do you prevent it?**
A: When a popular cache key expires and many simultaneous requests all miss at once, hammering the database simultaneously to regenerate the same value. Prevented using a short-lived lock (`SET ... NX EX`) so only one request regenerates the cache while others wait briefly or receive slightly stale data.

---

## 7. Sessions

**Q: Why store sessions in Redis instead of just using JWTs?**
A: JWTs are stateless and can't be easily invalidated early once issued (without extra infrastructure like a blacklist). Server-side sessions in Redis give precise, instant control — deleting the key immediately kills that session, which is valuable for logout, banning users, or responding to security incidents.

---

## 8. Pub/Sub

**Q: What's the difference between Redis Pub/Sub and a message queue?**
A: Pub/Sub broadcasts messages live to whoever is currently subscribed — messages aren't stored, and are lost if nobody's listening at that instant. A message queue persists jobs until a worker processes them, supports retries, and guarantees eventual delivery even if workers are temporarily offline.

---

## 9. Rate Limiting & Locks

**Q: How would you rate-limit a login endpoint to prevent brute-force attacks?**
A: Use a Redis counter (via `INCR`) keyed by IP/username with a time window (fixed or sliding). Set an expiry on first attempt, reject further attempts past the limit with a `429`, and consider increasing lockout durations plus logging/alerting for repeated abuse.

**Q: Why does a distributed lock need a unique token, not just a simple flag?**
A: Without a unique token, if a lock's TTL expires while its original holder is still working and a new holder acquires it, the original holder's eventual release could delete the *new* holder's active lock — breaking the mutual exclusion guarantee entirely. A unique token, checked atomically via a Lua script before deletion, ensures you only ever release a lock you still actually own.

---

## 10. Queues

**Q: Why use Redis-backed queues instead of just running background work inline or with `setTimeout`?**
A: Inline execution blocks the response and ties work to a single request/process — in-memory timers vanish on restart. A Redis-backed queue persists jobs independently, supports retries/delays/priorities, and lets you scale workers horizontally, independent of your web server's scaling.

---

## 11. Scaling & High Availability

**Q: What's the difference between Redis replication and Redis Cluster?**
A: Replication provides redundancy — every replica holds a full copy of the data, mainly for read scaling and failover. Redis Cluster shards data across multiple independent master nodes so no single machine holds/serves all the data, enabling true horizontal scaling. Production systems typically combine both: a cluster of shards, each internally protected by replication.

**Q: What does Redis Sentinel do?**
A: It monitors a master/replica setup and automatically promotes a replica to master if the current master fails, without manual intervention, and notifies connected clients of the new master's address.

---

## 12. Production Readiness

**Q: What would you check before deploying Redis to production?**
A: Authentication/ACLs enabled, not exposed to the public internet, `maxmemory` and eviction policy explicitly set, dangerous commands restricted (`FLUSHALL`, `KEYS`), a deliberately chosen persistence strategy matched to data criticality, monitoring/alerting on memory and latency, graceful application-level fallback on Redis outages, and a tested failover plan.

---

## 13. Rapid-Fire Round (Good for Quick Warmups)

| Question | Short Answer |
|---|---|
| Default Redis port? | 6379 |
| Command to check if Redis is alive? | `PING` → `PONG` |
| Command to set a key with expiry in one step? | `SETEX` (or `SET key val EX seconds`) |
| Command for atomic increment? | `INCR` |
| What does TTL of `-1` mean? | Key exists, no expiry set |
| What does TTL of `-2` mean? | Key doesn't exist |
| What protocol does Redis use? | RESP (REdis Serialization Protocol) |
| What's `BLPOP` used for? | Blocking pop from a List — used for building simple queues |
| Safer alternative to `KEYS`? | `SCAN` |
| Algorithm for locking across multiple Redis nodes? | Redlock |

---

## 14. Final Summary of the Whole Course

You've gone from "What is Redis?" all the way to production architecture: data types, internals, memory management, persistence, real Node.js/Express integration, caching strategies, sessions, Pub/Sub, rate limiting, distributed locks, queues, clustering/replication, and production best practices. That's genuinely the full toolkit senior backend engineers use Redis for in real systems — you're well equipped to both build with it and speak confidently about it in interviews.

---

## 15. Final Homework / Self-Check

Without looking back at the modules, try to answer these from memory:

1. Explain Redis's core value proposition in 2 sentences.
2. Name all 5 core data types and one real use case for each.
3. Explain the Cache-Aside pattern and one problem it can run into.
4. Explain the difference between replication and clustering.
5. List 3 things you'd check before trusting Redis in production.

If you can answer all 5 confidently, you've genuinely internalized this course. 🎉
