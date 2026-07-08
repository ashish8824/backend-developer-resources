# Module 5: Redis Commands (Practical Reference)

**Level:** Beginner

You've already seen many commands in Module 4. This module organizes them into a clean reference you'll come back to again and again, plus a few important commands we haven't covered yet (keys, expiry, and inspecting your data).

---

## 1. Key Management Commands

These work on **any** key, regardless of its type (String, List, Hash, etc.).

```bash
# Does a key exist?
EXISTS username
# (integer) 1  → yes, 0 → no

# Delete a key (works for any type)
DEL username

# Rename a key
RENAME username user_name

# Check what TYPE a key is
TYPE user_name
# "string" / "list" / "hash" / "set" / "zset"

# Find all keys matching a pattern (⚠️ avoid in production — see warning below)
KEYS user:*

# Get the count of all keys in the current database
DBSIZE

# Wipe EVERYTHING in the current Redis database (⚠️ dangerous, dev/testing only!)
FLUSHDB
```

> ⚠️ **Warning about `KEYS`:** `KEYS` scans the *entire* keyspace, which can freeze Redis for a noticeable moment on large datasets, because Redis processes commands one at a time (Module 6 explains why). In production, use `SCAN` instead (a safer, incremental version) — we'll touch on this in Module 17.

```js
// Node.js equivalents
const exists = await redis.exists("username");     // 1 or 0
await redis.del("username");
await redis.rename("username", "user_name");
const type = await redis.type("user_name");         // "string"
const keys = await redis.keys("user:*");            // ⚠️ avoid in production, use scanStream instead
const count = await redis.dbsize();
```

---

## 2. Expiry Commands (Very Important!)

Redis lets you set a "time to live" (TTL) on any key. Once the TTL runs out, Redis automatically deletes the key for you. This is huge for things like OTPs, sessions, and temporary caches.

```bash
# Set a value with no expiry
SET session:abc "user_15_data"

# Add a TTL of 3600 seconds (1 hour) to an existing key
EXPIRE session:abc 3600

# Check how many seconds are left before expiry
TTL session:abc
# (integer) 3599

# -1 means the key exists but has NO expiry set
# -2 means the key doesn't exist at all

# Set a value AND its expiry in a single atomic command (preferred!)
SETEX session:abc 3600 "user_15_data"

# Remove an expiry (make the key permanent again)
PERSIST session:abc
```

```js
// Node.js equivalents
await redis.set("session:abc", "user_15_data");
await redis.expire("session:abc", 3600);          // TTL in seconds

const ttl = await redis.ttl("session:abc");        // seconds remaining

await redis.setex("session:abc", 3600, "user_15_data"); // set + expire in one call

await redis.persist("session:abc"); // remove expiry
```

**Why is this "atomic" thing important?** If you do `SET` then a separate `EXPIRE` call, there's a tiny window between the two commands where the key exists *without* an expiry. If your server crashes in that exact window, that key would live forever by mistake. `SETEX` (or `SET key value EX seconds`) does both in one indivisible step — eliminating that risk entirely.

---

## 3. String Commands (Recap + New Ones)

```bash
SET key value
GET key
SET key value EX 60        # set with expiry (60 seconds) in one line — same as SETEX
SET key value NX           # only set IF the key does NOT already exist (great for locks! Module 14)
SET key value XX           # only set IF the key ALREADY exists
MSET key1 val1 key2 val2   # set multiple keys at once
MGET key1 key2             # get multiple keys at once
STRLEN key                 # length of the string value
APPEND key "more text"     # append to an existing string
```

```js
await redis.set("key", "value");
await redis.set("key", "value", "EX", 60);   // expires in 60s
await redis.set("key", "value", "NX");        // only if not already set
await redis.mset("k1", "v1", "k2", "v2");
const values = await redis.mget("k1", "k2");  // ['v1', 'v2']
```

---

## 4. List, Hash, Set, and Sorted Set Commands (Quick Reference)

We covered the core versions of these in Module 4. Here's a compact cheat-sheet of the most-used ones:

```bash
# LIST
LPUSH key value      # add to front
RPUSH key value      # add to end
LPOP key             # remove & return from front
RPOP key             # remove & return from end
LLEN key             # length of list
LRANGE key 0 -1      # get all items

# HASH
HSET key field value
HGET key field
HGETALL key
HDEL key field
HEXISTS key field
HLEN key

# SET
SADD key member
SREM key member
SISMEMBER key member
SMEMBERS key
SCARD key

# SORTED SET
ZADD key score member
ZSCORE key member
ZRANK key member
ZRANGE key 0 -1 WITHSCORES
ZREM key member
```

---

## 5. Running Multiple Commands Together: Pipelines

Every command you send to Redis is a network round-trip. If you need to run 100 commands, doing them one at a time means 100 round-trips — slow! A **pipeline** batches multiple commands into a single network round-trip.

```js
// ❌ Slow: 3 separate network round-trips
await redis.set("a", 1);
await redis.set("b", 2);
await redis.set("c", 3);

// ✅ Fast: all 3 commands sent together in ONE round-trip
const pipeline = redis.pipeline();
pipeline.set("a", 1);
pipeline.set("b", 2);
pipeline.set("c", 3);
const results = await pipeline.exec(); // executes all queued commands at once
```

We'll use pipelines again for performance optimization in Module 17.

---

## 6. Interview Question: "How would you safely delete thousands of keys matching a pattern in production?"

**Bad answer:** `redis-cli KEYS "cache:*" | xargs redis-cli DEL` — using `KEYS` can block Redis on large datasets.

**Good answer:** Use `SCAN` with a `MATCH` pattern and a small `COUNT`, iterating in small batches, so Redis never has to pause to build one giant result set:

```bash
SCAN 0 MATCH cache:* COUNT 100
```

This returns a small batch of matching keys plus a "cursor" — you keep calling `SCAN` with the returned cursor until it comes back as `0`, meaning you've scanned everything. Because it works in small incremental steps, it doesn't block other clients like `KEYS` can.

---

## 7. Summary

- Key commands (`EXISTS`, `DEL`, `TYPE`, `RENAME`) work across all data types.
- Expiry commands (`EXPIRE`, `TTL`, `SETEX`, `PERSIST`) let Redis clean up temporary data automatically.
- `SET ... NX` (only set if not exists) is the foundation of distributed locks (Module 14).
- Avoid `KEYS` in production — use `SCAN` instead.
- Pipelines batch multiple commands into a single network round-trip for performance.

---

## 8. Homework

1. Set a key with a 10-second expiry, then use `TTL` to watch the countdown.
2. Try `SET mykey "hello" NX` twice in a row — notice the second call does nothing because the key already exists.
3. Write a small Node.js script that uses a pipeline to set 5 keys at once.

---

**Next up:** [Module 6 — How Redis Works Internally](./06-how-redis-works-internally.md)
