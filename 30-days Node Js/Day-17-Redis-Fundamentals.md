# Day 17 — Redis Part 1: Data Structures and Caching

> **Why this day matters:** Redis is one of the most commonly asked-about tools in backend interviews at the 3 YOE level, specifically because it shows up in real production systems for caching, session storage, rate limiting, and pub/sub — and it directly solves the "cluster workers don't share memory" problem from Day 15. Today builds the foundation (what Redis actually is, its data structures, and caching) before Day 18 covers pub/sub, session storage, and rate limiting specifically.

---

## 1. Background: what is Redis, precisely, and why is it so fast?

**Redis (REmote DIctionary Server)** is an **in-memory data structure store** — it keeps all of its data in **RAM**, not on disk (though it CAN persist to disk for durability, covered below). It's most commonly used as a **cache** and a **shared data store** for things multiple processes/servers need to agree on.

**Why is it so fast, precisely — this is a real interview question, not just trivia:**
1. **All data lives in RAM**, and RAM access is orders of magnitude faster than disk access (reading from RAM takes nanoseconds; reading from even a fast SSD takes microseconds — a 1000x+ difference). This is the single biggest reason for Redis's speed.
2. Redis is **single-threaded** for command execution (similar conceptually to Node's JS execution model!) — this might sound like it would be a bottleneck, but it actually avoids the overhead and complexity of locking/synchronizing data across multiple threads, and since Redis operations are typically very fast and simple, a single thread can process an enormous number of operations per second without needing parallelism for most workloads.
3. Its data structures (covered below) are specifically designed for **O(1) or O(log n)** time complexity on common operations — Redis isn't just "a key-value store," it's a toolkit of purpose-built, efficient structures.

**Connecting this directly back to Day 15:** remember the problem — cluster workers (or separate server instances) each have their OWN separate memory, so they can't share an in-memory JS variable. Redis solves this by being an **external, separate, shared service** that ALL your Node processes (regardless of which machine or which cluster worker) can connect to and read/write the SAME data.

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Worker 1    │     │  Worker 2    │     │  Worker 3    │
│ (process A)  │     │ (process B)  │     │ (process C)  │
└──────┬───────┘     └──────┬───────┘     └──────┬───────┘
       │                    │                    │
       └────────────────────┼────────────────────┘
                             ▼
                    ┌─────────────────┐
                    │      Redis       │   <- ONE shared source of truth,
                    │  (in-memory data) │      regardless of which worker
                    └─────────────────┘      a request lands on
```

---

## 2. Connecting to Redis from Node.js

```javascript
const { createClient } = require('redis'); // the official 'redis' npm package

const client = createClient({
  url: 'redis://localhost:6379', // Redis's default port is 6379
});

client.on('error', (err) => console.error('Redis Client Error:', err));

async function connect() {
  await client.connect();
  console.log('Connected to Redis');
}

connect();
```

---

## 3. Redis Data Structures — know what each one is FOR, not just its name

Redis isn't just "a giant hash map." It provides several distinct data structures, each suited to different real-world problems. Understanding WHEN to use which one is the actual skill being tested in interviews.

### Strings — the simplest, most common type

```javascript
// SET/GET — the most basic operation, storing a single value under a key
await client.set('user:1:name', 'Alice');
const name = await client.get('user:1:name'); // 'Alice'

// SETEX / EX option — set a value WITH an automatic expiration time (in
// seconds). This single feature is THE foundation of Redis-based caching
// (section 4) -- after the TTL (Time To Live) expires, Redis automatically
// deletes the key, no manual cleanup needed.
await client.set('session:abc123', 'userId:1', { EX: 3600 }); // expires in 1 hour

// INCR / DECR — atomic increment/decrement, commonly used for counters
// (e.g., page views, rate limiting -- Day 18 will use this exact operation)
await client.incr('page:home:views'); // atomically adds 1, even under concurrent access
```

**Why `INCR` being atomic matters — a real concurrency point:** if you instead did `GET` the current value, added 1 in your JS code, then `SET` the new value, TWO concurrent requests could both read the same starting value, both add 1, and both write back the same incremented result — losing one of the increments (a classic **race condition**). `INCR` performs the read-and-write as ONE atomic operation inside Redis itself, eliminating this race condition entirely. This is a genuinely important distinction to be able to explain.

### Hashes — storing an object-like structure under one key

```javascript
// A Hash lets you store MULTIPLE field-value pairs under ONE key --
// conceptually similar to storing a small JS object, without needing to
// JSON.stringify/parse a String type for simple structured data.
await client.hSet('user:1', {
  name: 'Alice',
  email: 'alice@test.com',
  age: '28', // Redis hash values are always strings -- you'd parse numbers on the app side
});

const name = await client.hGet('user:1', 'name'); // get ONE field
const allFields = await client.hGetAll('user:1');  // get ALL fields as an object
```

**When to use a Hash instead of just storing a JSON string:** if you need to update or read INDIVIDUAL fields frequently without touching the whole object, a Hash is more efficient — you'd otherwise need to fetch the whole JSON string, parse it, modify one field, re-stringify it, and write the whole thing back, which is wasteful for large objects with frequent partial updates.

### Lists — ordered collections, great for queues

```javascript
// Lists maintain INSERTION ORDER and support efficient push/pop from
// EITHER end -- this is exactly what makes them suitable as a basic
// queue or stack structure.
await client.lPush('recent:logins', 'user:42');  // push to the LEFT (front)
await client.rPush('task:queue', 'process-order-123'); // push to the RIGHT (back)

const recentLogins = await client.lRange('recent:logins', 0, 9); // get the first 10 items
const nextTask = await client.lPop('task:queue'); // pop from the front -- classic FIFO queue pattern
```

**Real use case:** a basic job queue (though Day 19's BullMQ builds a much more robust system ON TOP of Redis Lists and other structures specifically for this purpose — worth knowing Lists are part of what makes that possible underneath).

### Sets — unordered, UNIQUE collections

```javascript
// Sets automatically enforce uniqueness -- adding the same value twice
// has no effect, which makes them perfect for tracking distinct items
// without manual duplicate-checking logic.
await client.sAdd('post:123:likedBy', 'user:1');
await client.sAdd('post:123:likedBy', 'user:2');
await client.sAdd('post:123:likedBy', 'user:1'); // no-op, user:1 already in the set

const likeCount = await client.sCard('post:123:likedBy'); // count of unique members -- 2, not 3
const hasLiked = await client.sIsMember('post:123:likedBy', 'user:1'); // true -- O(1) lookup!

// Set OPERATIONS -- intersection, union, difference -- are genuinely
// powerful, real-world-useful built-in operations
const mutualFriends = await client.sInter(['user:1:friends', 'user:2:friends']);
```

**Real use case:** tracking unique likes/views, mutual friends/followers calculations, tag systems, deduplication.

### Sorted Sets — like Sets, but with a "score" for ranking

```javascript
// Sorted Sets combine uniqueness (like a Set) with an associated
// numeric SCORE per member, automatically keeping members sorted by
// that score -- this is THE structure for building leaderboards.
await client.zAdd('game:leaderboard', [
  { score: 1500, value: 'player1' },
  { score: 2300, value: 'player2' },
  { score: 1800, value: 'player3' },
]);

// Get the top 3 scores, descending
const topPlayers = await client.zRange('game:leaderboard', 0, 2, { REV: true });

// Get a specific player's RANK (position) -- a genuinely useful,
// non-obvious built-in capability
const rank = await client.zRevRank('game:leaderboard', 'player2'); // 0 (1st place)
```

**Real use case:** leaderboards, "top N trending items" type features, anything requiring ranked, sorted membership with efficient range queries.

---

## 4. Caching — the most common real-world Redis use case

### Background: why cache at all?

If a particular piece of data is **expensive to compute or fetch** (a complex database query, an aggregation across millions of rows, a call to a slow third-party API) and is **requested frequently with the same result**, recomputing/refetching it every single time wastes time and resources. Caching stores the RESULT of that expensive operation somewhere fast (Redis) so subsequent requests for the same data can be served almost instantly, skipping the expensive step entirely.

### The Cache-Aside Pattern (the most common caching strategy — know this by name)

```javascript
// THE PATTERN, explained step by step:
// 1. Check the cache (Redis) first
// 2. If found ("cache hit") -- return it immediately, skip the database
// 3. If NOT found ("cache miss") -- fetch from the database, THEN store
//    the result in the cache for next time, THEN return it
async function getUserById(userId) {
  const cacheKey = `user:${userId}`;

  // STEP 1: check cache first
  const cached = await client.get(cacheKey);
  if (cached) {
    console.log('Cache HIT'); // fast path -- no database query needed at all
    return JSON.parse(cached);
  }

  // STEP 2: cache miss -- fall back to the actual database
  console.log('Cache MISS');
  const user = await User.findByPk(userId); // Day 11's Sequelize, for example
  if (!user) return null;

  // STEP 3: populate the cache for NEXT time, with an expiration (TTL)
  // so stale data doesn't live forever if the underlying data changes
  await client.set(cacheKey, JSON.stringify(user), { EX: 300 }); // cache for 5 minutes

  return user;
}
```

### Cache Invalidation — "the hardest problem in computer science" (a real, famous saying, worth quoting)

**Background: what goes wrong if you DON'T think about this?** If a user updates their profile, but the OLD version is still sitting in the cache with a 5-minute TTL, anyone reading from the cache sees **stale data** for up to 5 minutes. Depending on the use case, this might be perfectly fine (eventual consistency is acceptable) or genuinely problematic (e.g., caching account balance or permission data).

```javascript
// THE FIX for write operations: invalidate (delete) the cache entry
// IMMEDIATELY whenever the underlying data changes, so the NEXT read
// is forced to be a cache miss, fetching fresh data and re-populating
// the cache correctly.
async function updateUser(userId, updates) {
  const user = await User.findByPk(userId);
  await user.update(updates);

  // Without this line, the cache would continue serving STALE data
  // for up to the remaining TTL after this update.
  await client.del(`user:${userId}`);

  return user;
}
```

**Strategies worth knowing the names of (a common interview ask: "how do you handle cache invalidation"):**
- **TTL-based expiration** (shown above) — simplest, accepts some staleness window, good when slight staleness is tolerable.
- **Write-through invalidation** (shown above, `del` on update) — actively clears/updates the cache the moment underlying data changes, minimizing staleness window.
- **Cache stampede consideration:** if a popular cache key expires and MANY concurrent requests all experience a cache miss at the SAME moment, they could all simultaneously hit the database with the same expensive query at once — a real production issue, sometimes mitigated with techniques like locking the recomputation or staggered/jittered TTLs.

---

## 5. Redis Persistence — briefly, a fact worth knowing

**Background: if Redis stores data in RAM, what happens if the server restarts?** By default, all in-memory data would be LOST on restart — for pure caching use cases, this is often acceptable (the cache just rebuilds itself via cache misses). But Redis also supports optional **persistence** mechanisms if you need data to survive a restart:
- **RDB (Redis Database)** — periodic snapshots of the dataset saved to disk.
- **AOF (Append Only File)** — logs every write operation, replayed on restart to rebuild state.

**Interview-relevant nuance:** if asked "is Redis durable?" — the honest, precise answer is: *"By default, Redis is in-memory and data can be lost on restart, but it offers optional RDB snapshotting and/or AOF logging for persistence if durability is required. Many use cases (caching, ephemeral session data) don't need persistence at all, while others (using Redis as a primary data store for certain features) would configure one of these mechanisms."*

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Your API's response times improved dramatically after adding Redis caching, but occasionally users see outdated data — what's happening, and is it acceptable?"** → This is the inherent caching/staleness tradeoff — discuss TTL length choices and whether write-through invalidation is needed for THIS specific data's consistency requirements.
- **"How would you implement a real-time leaderboard for a game with millions of players?"** → Redis Sorted Sets — `ZADD` for score updates, `ZREVRANGE` for top-N queries, `ZREVRANK` for a specific player's rank — all efficient even at large scale.
- **"Why use Redis's `INCR` instead of reading a value, incrementing in your application code, and writing it back?"** → Atomicity — prevents race conditions where concurrent requests could read the same stale value and lose an increment, since `INCR` performs the read-modify-write as one indivisible operation inside Redis.
- **"What's the cache-aside pattern, and what's a downside to be aware of?"** → Check cache first, fall back to the source of truth on a miss, then populate the cache — downside: the cache stampede risk when a hot key expires under heavy concurrent load, plus the general challenge of choosing the right invalidation strategy and TTL for each kind of data.

---

## 7. Hands-on practice for today

```javascript
// practice-day17.js (requires a local Redis server running, or a free
// cloud Redis instance like Upstash/Redis Cloud for quick testing)
const { createClient } = require('redis');

async function main() {
  const client = createClient();
  await client.connect();

  // Strings + TTL
  await client.set('greeting', 'Hello Redis', { EX: 10 });
  console.log(await client.get('greeting'));

  // Hash
  await client.hSet('product:1', { name: 'Laptop', price: '1200' });
  console.log(await client.hGetAll('product:1'));

  // Set
  await client.sAdd('tags:article:1', ['nodejs', 'redis', 'backend']);
  console.log(await client.sMembers('tags:article:1'));

  // Sorted Set (leaderboard simulation)
  await client.zAdd('leaderboard', [
    { score: 100, value: 'alice' },
    { score: 250, value: 'bob' },
    { score: 175, value: 'carol' },
  ]);
  const top2 = await client.zRange('leaderboard', 0, 1, { REV: true });
  console.log('Top 2:', top2);

  // Cache-aside simulation with a fake "slow" data fetch
  async function getExpensiveData() {
    const cacheKey = 'expensive:data';
    const cached = await client.get(cacheKey);
    if (cached) {
      console.log('Cache HIT');
      return JSON.parse(cached);
    }
    console.log('Cache MISS -- simulating slow computation...');
    await new Promise((r) => setTimeout(r, 1000)); // pretend this is slow
    const data = { result: 42, computedAt: new Date().toISOString() };
    await client.set(cacheKey, JSON.stringify(data), { EX: 30 });
    return data;
  }

  console.log(await getExpensiveData()); // MISS, slow
  console.log(await getExpensiveData()); // HIT, instant

  await client.quit();
}

main();
```

Run this with a local Redis instance (`docker run -p 6379:6379 redis` is the fastest way to get one running if you don't have Redis installed) and observe the cache hit/miss timing difference directly.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: Why is Redis so much faster than a typical relational database for simple lookups?**
   A: It stores all data in RAM rather than on disk, and RAM access is orders of magnitude faster than disk access — combined with simple, purpose-built data structures designed for O(1)/O(log n) operations.

2. **Q: Name three Redis data structures and a real use case for each.**
   A: Strings (caching, counters, TTL-based sessions), Hashes (storing object-like structured data efficiently with partial field access), Sorted Sets (leaderboards, ranked data with efficient range queries).

3. **Q: Why is `INCR` important for things like view counters or rate limiting?**
   A: It performs the read-increment-write as ONE atomic operation inside Redis, preventing race conditions that would occur if you manually read a value, incremented it in application code, and wrote it back — which could lose increments under concurrent access.

4. **Q: Explain the cache-aside pattern.**
   A: Check the cache first; on a hit, return immediately. On a miss, fetch from the actual data source (database/API), store the result in the cache with an appropriate TTL, then return it — so subsequent requests for the same data are served from the fast cache instead of repeating the expensive operation.

5. **Q: What's a downside or risk of caching you should be aware of?**
   A: Stale data during the cache's TTL window (requiring an invalidation strategy on writes if staleness isn't acceptable for that data), and "cache stampede" risk, where a popular expired key causes many concurrent requests to simultaneously hit the underlying data source at once.

6. **Q: Is data stored in Redis durable across a server restart?**
   A: Not by default — Redis is in-memory and can lose data on restart, but supports optional persistence via RDB snapshots and/or AOF logging if durability is required for a given use case.

---

## Tomorrow (Day 18) Preview
We continue Redis with the topics that come up MOST often in interviews: **Pub/Sub messaging**, using Redis as a **session store** for horizontally-scaled apps, and — critically — **implementing rate limiting with Redis** (token bucket and sliding window algorithms), directly setting up Day 22's deeper rate limiting discussion.
