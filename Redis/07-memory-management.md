# Module 7: Memory Management

**Level:** Intermediate

Since Redis stores data in RAM, and RAM is limited and expensive, understanding how Redis manages memory is essential for running it safely in production.

---

## 1. The Core Problem

RAM is finite. If you keep adding data to Redis forever without limits, eventually you'll run out of memory, and Redis (or your whole server) can crash or start rejecting writes.

You need answers to:

1. How much memory is Redis currently using?
2. What happens when Redis runs low on memory?
3. How do I prevent unbounded growth in the first place?

---

## 2. Checking Memory Usage

```bash
# Overview of memory usage
INFO memory

# Key fields to look at:
# used_memory_human:   e.g. "45.20M"  → human-readable current usage
# maxmemory_human:     the configured memory limit (0 = unlimited)
# maxmemory_policy:    what Redis does when the limit is hit (see below)
```

```bash
# Memory used by a SPECIFIC key (very useful for debugging "why is memory so high?")
MEMORY USAGE user:15
# (integer) 128   ← size in bytes
```

```js
// Node.js
const info = await redis.info("memory");
console.log(info); // raw text output, parse the fields you need

const bytes = await redis.memory("USAGE", "user:15");
console.log(`user:15 uses ${bytes} bytes`);
```

---

## 3. Setting a Memory Limit

By default, Redis will try to use as much memory as your machine allows — which is dangerous in production, because if Redis (and everything else on the server) exhausts RAM, the whole machine can slow to a crawl or crash.

```bash
# Set a hard memory limit of 256 megabytes (in redis.conf, or live via CONFIG SET)
CONFIG SET maxmemory 256mb

# Check current setting
CONFIG GET maxmemory
```

**Always set `maxmemory` explicitly in production.** Leaving it unbounded is one of the most common real-world Redis mistakes.

---

## 4. What Happens When Redis Hits the Memory Limit? (Eviction Policies)

Once `maxmemory` is reached, Redis needs a strategy for what to do next. This is controlled by `maxmemory-policy`. Here are the most important options:

| Policy | Behavior |
|---|---|
| `noeviction` (default) | Reject new writes with an error, but allow reads. Safest for "Redis as source-of-truth" use cases, but can break your app if you're not ready to handle write errors. |
| `allkeys-lru` | Evict the **L**east **R**ecently **U**sed key across the whole dataset, regardless of expiry. Great general-purpose cache policy. |
| `allkeys-lfu` | Evict the **L**east **F**requently **U**sed key. Better than LRU when some keys are accessed rarely but recently (avoids evicting "hot" keys that just happened to not be touched in the last few seconds). |
| `volatile-lru` | Only evict keys that **have an expiry set**, using LRU order. Keys without a TTL are never evicted — safer if you mix permanent and temporary data in the same Redis instance. |
| `volatile-ttl` | Evict keys with the **soonest expiry** first, among keys that have a TTL. |
| `volatile-random` / `allkeys-random` | Evict a random key (rarely used, but available). |

```bash
# Example: use Redis purely as a cache — evict least-recently-used keys when full
CONFIG SET maxmemory-policy allkeys-lru
```

**Rule of thumb:**
- If Redis is **purely a cache** (all data can be safely regenerated from your database) → use `allkeys-lru` or `allkeys-lfu`.
- If Redis holds a **mix** of permanent-ish data and temporary/cache data → use `volatile-lru`, and make sure only your *cache* keys have a TTL set.
- If Redis must **never silently lose data** (e.g., you're using it as a lightweight primary store for something) → use `noeviction`, and make sure your app gracefully handles write failures.

---

## 5. LRU vs LFU — A Simple Analogy

Imagine a small bookshelf (limited memory) in a shared office:

- **LRU (Least Recently Used):** "Which book hasn't been touched in the longest time? Remove that one."
- **LFU (Least Frequently Used):** "Which book has been picked up the fewest total times, historically? Remove that one."

LFU is smarter for data with a "hot and cold" pattern — e.g., a book that's read constantly every single day, but simply hasn't been touched in the last hour, shouldn't be evicted just because it wasn't "recently" used. LFU tracks a longer memory of *frequency*, not just *recency*.

---

## 6. Memory Optimization Tips (Practical)

**1. Always set expiry on cache data.**

```js
// ❌ Bad: this key lives forever, silently consuming memory
await redis.set("cache:product:55", JSON.stringify(product));

// ✅ Good: automatically cleaned up after 1 hour
await redis.set("cache:product:55", JSON.stringify(product), "EX", 3600);
```

**2. Use Hashes instead of many separate String keys, when possible.**

Redis has internal overhead *per key* (metadata Redis stores about each key). Storing 1,000 separate string keys is less memory-efficient than storing one Hash with 1,000 fields, especially for small values.

```js
// ❌ Less memory-efficient: 3 separate keys, each with overhead
await redis.set("user:15:name", "Ashish");
await redis.set("user:15:age", "24");
await redis.set("user:15:city", "Bangalore");

// ✅ More memory-efficient: 1 key (a Hash) holding all 3 fields
await redis.hset("user:15", { name: "Ashish", age: 24, city: "Bangalore" });
```

**3. Avoid storing huge values in a single key** (e.g., a 50MB JSON blob). Break large datasets into smaller pieces where practical — huge single values can cause latency spikes when read/written/evicted.

**4. Monitor regularly** — memory issues tend to creep up slowly, so set up alerts on `used_memory` in production (Module 17 covers monitoring).

---

## 7. Interview Question: "What happens if Redis runs out of memory?"

**Answer:** It depends on the configured `maxmemory-policy`. With the default `noeviction`, Redis will reject further write commands with an "OOM" error while still allowing reads, until memory is freed up. With an eviction policy like `allkeys-lru`, Redis will automatically remove keys (based on the policy's rule) to make room for new writes — meaning some data could be silently lost, which is acceptable for a pure cache but dangerous if that data was important and unrecoverable.

---

## 8. Summary

- RAM is limited — always configure `maxmemory` explicitly in production.
- `maxmemory-policy` decides what happens when the limit is hit: reject writes (`noeviction`) or evict keys (`allkeys-lru`, `allkeys-lfu`, `volatile-lru`, etc.).
- Choose eviction policy based on whether Redis holds pure cache data, a mix, or critical data.
- Practical tips: always set TTLs on cache data, prefer Hashes over many small String keys, avoid oversized values, and monitor memory over time.

---

## 9. Homework

1. Run `CONFIG GET maxmemory-policy` on your local Redis and note the current setting.
2. Set `maxmemory` to `50mb` and `maxmemory-policy` to `allkeys-lru`, then explain in your own words what will happen once you exceed that limit.
3. Rewrite this snippet to be more memory-efficient using a Hash: storing `product:9:name`, `product:9:price`, and `product:9:stock` as three separate String keys.

---

**Next up:** [Module 8 — Persistence (RDB & AOF)](./08-persistence.md)
