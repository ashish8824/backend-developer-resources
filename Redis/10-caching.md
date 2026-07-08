# Module 10: Caching (Full Deep Dive)

**Level:** Intermediate

Caching is the #1 real-world use case for Redis. This module goes deep: patterns, expiry strategies, and the hardest problem in caching — invalidation.

---

## 1. Recap: The Cache-Aside Pattern

We already saw this pattern in action in Module 9. Let's formalize it:

```
1. Request comes in
2. Check Redis → HIT?  → return cached data (fast path)
3.               MISS? → query database
4.                        → store result in Redis
                          → return data to user
```

```js
async function getProduct(id) {
  const cacheKey = `product:${id}`;

  // Step 1: check the cache
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Step 2: cache miss — go to the database
  const product = await db.query("SELECT * FROM products WHERE id = ?", [id]);

  // Step 3: populate the cache for next time
  await redis.set(cacheKey, JSON.stringify(product), "EX", 600); // 10 minutes

  return product;
}
```

This is called "cache-aside" because the **application code** is responsible for managing the cache — Redis itself doesn't know anything about your database.

---

## 2. Other Caching Patterns (Good to Know)

**Write-Through Cache:** Every time you write to the database, you *immediately* also update the cache — so the cache is never stale.

```js
async function updateProduct(id, newData) {
  await db.query("UPDATE products SET ... WHERE id = ?", [id]);   // 1. update DB
  await redis.set(`product:${id}`, JSON.stringify(newData), "EX", 600); // 2. update cache immediately
}
```

**Write-Behind (Write-Back) Cache:** You write to the cache first (fast), and update the database *later*, asynchronously (e.g., via a queue). Riskier — if the app crashes before the database write happens, you lose data — but very fast for write-heavy workloads. Less common for beginners; mentioned here so you recognize the term.

**Read-Through Cache:** Similar to cache-aside, but the *cache layer itself* (not your app code) is responsible for fetching from the database on a miss. This usually requires a specialized caching library/framework. Less common in typical Node.js setups, where cache-aside (manual) is simpler and more explicit.

---

## 3. Choosing the Right TTL (Expiry Time)

There's no single "correct" TTL — it depends on how often the data changes and how costly staleness is.

| Data Type | Suggested TTL | Reasoning |
|---|---|---|
| Static reference data (country list, categories) | Hours to days | Rarely changes |
| Product details | Minutes (5–30) | Changes occasionally (price, stock) |
| User profile | Minutes | Changes occasionally |
| Trending / leaderboard data | Seconds to a few minutes | Changes frequently, staleness is usually acceptable |
| Real-time stock prices | Seconds or less, or don't cache at all | Staleness is unacceptable |

```js
// Example: different TTLs for different data "temperatures"
await redis.set("categories:all", data, "EX", 60 * 60 * 24);  // 24 hours — rarely changes
await redis.set(`product:${id}`, data, "EX", 60 * 10);         // 10 minutes — changes sometimes
await redis.set("trending:posts", data, "EX", 30);              // 30 seconds — changes often
```

---

## 4. The Hardest Problem: Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." — a famous programmer joke, and it's funny because it's true.

**The problem:** What happens when the underlying data changes *before* the cache naturally expires? Users might see stale (outdated) data.

**Solution 1 — Just wait for TTL to expire (simplest).** Acceptable when brief staleness is fine (e.g., a product page showing slightly outdated stock count for a few minutes).

**Solution 2 — Actively delete/update the cache when data changes.**

```js
async function updateProductPrice(id, newPrice) {
  // 1. Update the source of truth
  await db.query("UPDATE products SET price = ? WHERE id = ?", [newPrice, id]);

  // 2. Invalidate (delete) the stale cache entry immediately
  await redis.del(`product:${id}`);

  // Next request for this product will be a cache MISS,
  // which will re-fetch fresh data from the database and re-cache it.
}
```

**Solution 3 — Update the cache directly instead of deleting it** (avoids a temporary "cold" miss right after the update):

```js
async function updateProductPrice(id, newPrice) {
  const updatedProduct = await db.query(
    "UPDATE products SET price = ? WHERE id = ? RETURNING *",
    [newPrice, id]
  );

  // Refresh the cache with the new value directly
  await redis.set(`product:${id}`, JSON.stringify(updatedProduct), "EX", 600);
}
```

---

## 5. Avoiding the "Cache Stampede" Problem

**The problem:** Imagine a very popular cache key expires, and suddenly 10,000 simultaneous requests all get a cache MISS at the exact same moment. All 10,000 requests then hit your database simultaneously, trying to regenerate the same value — potentially overwhelming your database right when it's most vulnerable.

```
Cache expires
     │
     ▼
10,000 requests ALL miss cache at once
     │
     ▼
10,000 requests ALL hit the database at once  💥
```

**Solution: A simple "lock" so only ONE request regenerates the cache, while others wait or get slightly stale data.**

```js
async function getProductWithStampedeProtection(id) {
  const cacheKey = `product:${id}`;
  const lockKey = `lock:product:${id}`;

  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  // Try to acquire a short-lived lock — "NX" means "only set if it doesn't already exist"
  const gotLock = await redis.set(lockKey, "1", "EX", 5, "NX");

  if (!gotLock) {
    // Someone else is already regenerating the cache — wait briefly and retry
    await new Promise((r) => setTimeout(r, 100));
    return getProductWithStampedeProtection(id); // simple retry (in production, add a retry limit)
  }

  // We got the lock — we're responsible for regenerating the cache
  const product = await db.query("SELECT * FROM products WHERE id = ?", [id]);
  await redis.set(cacheKey, JSON.stringify(product), "EX", 600);
  await redis.del(lockKey); // release the lock

  return product;
}
```

We'll cover the `SET ... NX` locking pattern properly, with more nuance, in Module 14 (Distributed Locks).

---

## 6. Caching Whole API Responses (Middleware Pattern)

Instead of caching inside each route manually, you can build a reusable Express middleware:

```js
// middleware/cache.js

const redis = require("../config/redisClient");

// A reusable middleware factory — pass in a TTL in seconds
function cacheMiddleware(ttlSeconds) {
  return async (req, res, next) => {
    // Build a cache key based on the full URL (includes query params)
    const cacheKey = `route:${req.originalUrl}`;

    try {
      const cached = await redis.get(cacheKey);
      if (cached) {
        console.log(`⚡ Cache HIT for ${cacheKey}`);
        return res.json(JSON.parse(cached));
      }
    } catch (err) {
      console.error("Redis read failed, continuing without cache:", err.message);
    }

    // Monkey-patch res.json so that whenever the route eventually calls
    // res.json(data), we ALSO save that data into Redis before sending it.
    const originalJson = res.json.bind(res);
    res.json = (data) => {
      redis.set(cacheKey, JSON.stringify(data), "EX", ttlSeconds).catch((err) => {
        console.error("Redis write failed:", err.message);
      });
      return originalJson(data);
    };

    next(); // no cache found — continue to the actual route handler
  };
}

module.exports = cacheMiddleware;
```

**Usage:**

```js
const cacheMiddleware = require("../middleware/cache");

// Cache this entire route's response for 5 minutes
router.get("/products", cacheMiddleware(300), async (req, res) => {
  const products = await db.query("SELECT * FROM products");
  res.json(products); // this call gets intercepted and cached automatically
});
```

---

## 7. Interview Question: "How do you keep a cache from serving stale data?"

**Answer:** There's no single perfect solution — it's a trade-off. Common strategies: (1) set a sensible TTL matched to how tolerant the app is to staleness, (2) actively invalidate (delete or update) the cache entry whenever the underlying data changes, and (3) for very popular keys, use a short-lived lock to prevent "cache stampedes" where many simultaneous misses hammer the database at once.

---

## 8. Summary

- **Cache-Aside** is the standard pattern: check cache → miss → query DB → populate cache → return.
- Choose TTLs based on how often data changes and how costly staleness is.
- Actively invalidate (delete/update) cache entries when the underlying data changes, for lower staleness.
- Guard against **cache stampedes** on popular keys using a short lock.
- A reusable Express middleware can cache entire API responses with minimal code duplication.

---

## 9. Homework

1. Implement the cache-aside pattern for a `/orders/:id` endpoint of your own design.
2. Add active invalidation: when an order is updated, delete its cache entry.
3. Explain, in your own words, what a "cache stampede" is and how the lock-based solution prevents it.

---

**Next up:** [Module 11 — Session Storage](./11-session-storage.md)
