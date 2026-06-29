# Caching (Redis)

## What is Caching?

Caching stores frequently accessed data in a fast-access layer (usually memory) so you don't hit the slower source (database, disk, network) every time.

```
Request: GET /user/123
Without cache: Query DB every time
With cache:    Check Redis first, fall back to DB only on miss
```

## Why do we need it?

### 1. Reduce Database Load

Suppose 10,000 requests/sec hit the same product page.

```
Without caching: DB hit 10,000 times/sec -> DB CPU ↑, latency ↑, connections exhausted
With caching:    DB hit only on cache miss, most requests served from memory
```

### 2. Reduce Latency

```
DB query:      ~50ms
Redis lookup:  ~1ms
```

### 3. Improve Scalability

One Redis instance can serve millions of reads/sec, far more than a relational DB. Read-heavy systems (e.g. a news feed, a product catalog) often have read:write ratios of 100:1 or higher — caching is what makes that ratio survivable.

### 4. Reduce Cost

Many managed DBs charge by IOPS/read capacity. Offloading reads to an in-memory cache directly cuts the bill.

## Real-World Examples

| Use Case            | Approach                                  |
|----------------------|--------------------------------------------|
| Product Catalog      | Cache product details, TTL = 10 min         |
| Session Storage       | Cache user session, TTL = 30 min            |
| Leaderboard           | Redis Sorted Set, real-time ranking          |
| Rate Limiter counters | Redis (see Rate Limiter notes)               |
| Feed/Timeline         | Pre-computed cached feed per user, invalidated on new post |
| Config/Feature Flags  | Rarely-changing data, long TTL, cached app-wide |

## Caching Strategies

### 1. Cache-Aside (Lazy Loading) — Most Common

**Read flow:**

```
Check Redis
  HIT  -> return value
  MISS -> query DB -> store in Redis -> return value
```

**Write flow:**

```
Write to DB
(cache updated only on next read, or invalidated)
```

- **Pros:** Simple, only caches what's actually requested (no wasted memory on unused data).
- **Cons:** First request after expiry is always slow (cache miss penalty).

### 2. Write-Through

**Write flow:**

```
Write to Redis AND DB at the same time
```

- **Pros:** Cache always consistent with DB; reads are always fast since the cache is always warm for written data.
- **Cons:** Write latency increases (two writes, often synchronous).

### 3. Write-Behind (Write-Back)

**Write flow:**

```
Write to Redis immediately
Asynchronously flush to DB later
```

- **Pros:** Very fast writes — great for write-heavy workloads (e.g. view counters, analytics events).
- **Cons:** Risk of data loss if Redis crashes before the flush completes; more complex to implement correctly (batching, retry, ordering).

### 4. Read-Through

Similar to Cache-Aside, but the cache layer itself is responsible for loading from the DB on a miss — usually handled by a library/framework (e.g. Spring Cache, a caching ORM layer) rather than application code explicitly checking and populating the cache.

### 5. Write-Around

**Write flow:**

```
Write goes directly to the DB, bypassing the cache entirely
```

The cache only gets populated later, on a subsequent read (cache-aside style). Useful when written data is unlikely to be read again soon — avoids polluting the cache with data nobody will request (e.g. bulk import jobs, write-once logs).

### Strategy Comparison

| Strategy | Write Speed | Read Speed (after write) | Consistency Risk | Best For |
|---|---|---|---|---|
| Cache-Aside | Fast (DB only) | Slow on first read | Stale cache if invalidation is missed | General-purpose, default choice |
| Write-Through | Slower (2 writes) | Always fast | Strong | Read-heavy, write-light data |
| Write-Behind | Very fast | Always fast | Risk of loss on crash | Write-heavy, loss-tolerant data |
| Read-Through | N/A (delegates to cache-aside under the hood) | Slow on first read | Same as cache-aside | Framework-managed caching |
| Write-Around | Fast (DB only) | Slow until next read | Low pollution risk | Rarely-re-read writes |

## Common Cache Failure Modes (Important Interview Topic)

These terms get confused with each other constantly — know the distinctions.

### Cache Penetration

A query for a key that **doesn't exist** in either the cache or the DB (e.g. an attacker probing random/non-existent user IDs). Since it's never found, it's never cached, so *every* such request bypasses the cache and hits the DB directly — repeatedly.

**Fix:**
- Cache the "not found" result too (a short-TTL null/sentinel value), so repeated lookups of the same missing key don't reach the DB.
- Use a **Bloom Filter** in front of the cache to cheaply check "could this key possibly exist?" before even querying the DB.

### Cache Breakdown (a.k.a. Hotspot Invalidation)

A **single, very popular key** expires, and a flood of concurrent requests for that *one* key all miss the cache at the same instant and all hit the DB simultaneously — even though the rest of the cache is fine.

**Fix:**
- Mutex/lock: only the first request that misses rebuilds the value; others wait or get a slightly stale value.
- Never let hot keys expire naturally — refresh them proactively in the background before TTL runs out (a.k.a. "refresh-ahead").

### Cache Avalanche

**Many keys** expire at (or close to) the same time — e.g. you warmed 100,000 keys all with `TTL = 600s` at the same moment, so they all expire together — causing a sudden, large-scale wave of DB queries all at once.

**Fix:**
- Add random "jitter" to TTLs (e.g. `600s ± random(0, 60s)`) so expirations are spread out instead of synchronized.
- Have a fallback/circuit-breaker so the DB doesn't get fully overwhelmed if a large wave does occur (e.g. serve stale data temporarily, or queue requests).

### Cache Stampede (a.k.a. Thundering Herd)

The general term for *any* scenario above where many requests pile onto the DB simultaneously after a cache miss — Cache Breakdown is really the single-hot-key flavor of a stampede, and Cache Avalanche is the many-keys flavor. The fixes (locking, jitter, refresh-ahead) overlap.

## Eviction Policies

When the cache is full, something has to go.

| Policy | Behavior |
|---|---|
| LRU (Least Recently Used) | Evict the item not accessed for the longest time |
| LFU (Least Frequently Used) | Evict the item accessed least often |
| FIFO | Evict the oldest inserted item |
| TTL-based | Evict on expiry, regardless of usage |

Redis defaults to `allkeys-lru`, but supports several configurable eviction policies (e.g. `volatile-lru` only evicts keys that have a TTL set, leaving persistent keys untouched).

## Why Redis specifically?

- In-memory → sub-millisecond reads
- Built-in TTL support per key
- Atomic operations (e.g. `INCR` — useful for counters/rate limiters)
- Rich data structures and their typical uses:
  - **String** — simple key/value cache (e.g. serialized JSON of a product)
  - **Hash** — store an object's fields without re-serializing the whole thing (e.g. user profile fields)
  - **List** — recent activity feeds, simple queues
  - **Set** — unique membership checks (e.g. "has this user already liked this post?")
  - **Sorted Set** — leaderboards, anything needing ranked/range queries by score
- Supports clustering for horizontal scale
- Persistence options (RDB snapshots, AOF logs) if durability is needed

## Scaling Redis

### Redis Cluster (Sharding)

Splits data across multiple nodes using **hash slots** (16,384 fixed slots, each key is hashed to a slot, each node owns a range of slots — conceptually similar to consistent hashing). This lets you scale beyond a single machine's memory and handle more throughput by spreading load.

### Redis Sentinel (High Availability)

A separate mechanism for **failover**, not sharding: Sentinel processes monitor a primary/replica setup, and automatically promote a replica to primary if the primary goes down. Often used for smaller deployments that need HA but not horizontal sharding; Redis Cluster has its own built-in failover too for larger deployments.

## The Hot Key Problem

Even with Cluster sharding, if one key (e.g. a viral post, a celebrity's profile) gets a disproportionate share of traffic, it can overload the *single node* that owns that key's hash slot — sharding doesn't fix this since it's about one key, not data volume.

**Mitigations:**
- Add a local in-process cache (e.g. a small Caffeine/Guava cache) in front of Redis on each app server for the hottest keys, so most reads never even reach Redis.
- Replicate the hot key under several key variants (e.g. `product:42:0`, `product:42:1`, ...) and randomly pick one per request, spreading load across nodes/replicas.

## Multi-Level Caching

Real systems often layer multiple cache tiers, each trading capacity for speed:

```
Client
  |
CDN (edge cache, e.g. CloudFront/Cloudflare) -- static/public content
  |
Local in-process cache (per app server, e.g. Caffeine) -- nanosecond access, smallest capacity
  |
Redis (shared, distributed) -- sub-millisecond, shared across all servers
  |
Database -- source of truth
```

Each layer absorbs traffic so the layer below it sees progressively less load.

## Java Implementation (Cache-Aside with Redis)

```java
import redis.clients.jedis.Jedis;

public class ProductCacheService {

    private final Jedis redis;
    private final ProductRepository dbRepo;
    private final int ttlSeconds = 600; // 10 minutes

    public ProductCacheService(Jedis redis, ProductRepository dbRepo) {
        this.redis = redis;
        this.dbRepo = dbRepo;
    }

    public String getProduct(String productId) {

        String key = "product:" + productId;

        // Check cache
        String cached = redis.get(key);
        if (cached != null) {
            return cached.equals("NULL") ? null : cached; // cache hit (incl. cached negative result)
        }

        // Cache miss -> fetch from DB
        String dbValue = dbRepo.findById(productId);

        if (dbValue != null) {
            redis.setex(key, ttlSeconds, dbValue);
        } else {
            // Negative caching: prevents cache penetration from repeated lookups of missing IDs
            redis.setex(key, 60, "NULL");
        }

        return dbValue;
    }

    public void updateProduct(String productId, String newValue) {
        dbRepo.save(productId, newValue);

        // Invalidate cache instead of updating it
        redis.del("product:" + productId);
    }
}
```

`updateProduct` deletes the key rather than overwriting it — this is the standard cache-aside write pattern. It avoids race conditions where a slow DB write could overwrite a fresher cache value. The `"NULL"` sentinel with a short TTL is a simple defense against cache penetration.

## Production Architecture

```
Client
  |
Load Balancer
  |
Node1  Node2  Node3
      |
   Redis Cluster
      |
   Database
```

All app servers share the same Redis cluster so cache state is consistent regardless of which node handles the request.

## Interview Questions

**Q1: What is Cache Stampede (Thundering Herd)?**

A popular key expires, and thousands of requests hit at the same moment, all miss the cache simultaneously, and all hammer the DB at once.

**Fix:**
- Lock-based regeneration (only one request rebuilds the value, others wait)
- Probabilistic early expiry / refresh-ahead (refresh slightly before TTL ends)

**Q2: How do you keep cache and DB consistent?**

You generally can't guarantee perfect real-time consistency without write-through. Standard approach:
- Cache-aside + delete-on-write
- Short TTLs as a safety net for missed invalidations

**Q3: What's the difference between LRU and LFU?**

- LRU: time-based (when was it last touched)
- LFU: frequency-based (how often was it touched)

LFU is better when some items are accessed in repeated bursts; LRU is simpler and works well for general recency-based access patterns.

**Q4: Why use Redis instead of just adding more application-server memory (local cache)?**

A local cache (e.g. an in-process HashMap) isn't shared across servers, so each node ends up with different stale data and you waste memory duplicating the same entries. Redis gives one shared source of truth that all nodes read from.

**Q5: What's the difference between Cache Penetration, Cache Breakdown, and Cache Avalanche?**

- **Penetration:** repeated queries for a key that *doesn't exist anywhere* (cache or DB) — fixed with negative caching or a Bloom filter.
- **Breakdown:** one hot key expires and many concurrent requests for *that single key* hit the DB at once — fixed with locking or refresh-ahead.
- **Avalanche:** many keys expire around the same time, causing a broad wave of DB load — fixed with TTL jitter.

**Q6: How would you cache data for a viral/hot key without overloading a single Redis node?**

Add a local in-process cache layer in front of Redis for very hot keys, and/or shard the hot key into multiple key variants distributed across nodes, randomly picking one per request.
