# Module 4: Redis Data Types

**Level:** Beginner

This is one of the most important modules in the whole course. A lot of people think Redis is "just a key-value store where values are strings." That's wrong! Redis supports **rich data structures**, and picking the right one for the job is what separates beginners from senior engineers.

---

## 1. The Big Picture

Every single thing in Redis is stored as a **key → value** pair. What changes is the **type** of the value. Think of it like this:

```js
// JavaScript has different data types too:
let a = "hello";        // String
let b = [1, 2, 3];      // Array
let c = { x: 1 };       // Object
let d = new Set([1,2]); // Set

// Redis is similar — the VALUE behind a key can be one of several types:
// String, List, Hash, Set, Sorted Set, and more (Streams, Bitmaps, HyperLogLog, Geo)
```

We'll cover the 5 core types every Node.js developer must know, then briefly touch on advanced ones.

---

## 2. Strings

The simplest type. A key maps to a single string value (which can also hold numbers or JSON text).

**Real use cases:** caching a single value, counters, feature flags, simple tokens.

```bash
# Set a string value
SET username "ashish"

# Get it back
GET username
# "ashish"

# Numbers work too, and Redis gives you atomic increment/decrement
SET views 10
INCR views      # views is now 11 — atomic, safe even with many simultaneous requests
DECR views      # views is now 10 again
INCRBY views 5  # views is now 15

# Set a value that automatically expires after 60 seconds (great for OTPs!)
SETEX otp:9876543210 60 "482913"
# key = "otp:9876543210", expires in 60 seconds, value = "482913"
```

```js
// Same operations from Node.js using ioredis
await redis.set("username", "ashish");          // store a string
const name = await redis.get("username");        // "ashish"

await redis.set("views", 10);
await redis.incr("views");                        // atomically +1
await redis.incrby("views", 5);                    // atomically +5

// SETEX = "SET" + "EX"(expiry) → stores value AND sets a TTL (time-to-live) in one command
await redis.setex("otp:9876543210", 60, "482913"); // expires automatically in 60s
```

---

## 3. Lists

An ordered collection of strings — think of it like a JavaScript array, but one you can efficiently push/pop from **both ends**.

**Real use cases:** activity feeds, recent notifications, simple job queues, "latest N items."

```bash
# Push items onto the LEFT (front) of a list
LPUSH recent_orders "order:101"
LPUSH recent_orders "order:102"
# list is now: [ "order:102", "order:101" ]

# Push onto the RIGHT (end) of a list
RPUSH recent_orders "order:103"
# list is now: [ "order:102", "order:101", "order:103" ]

# Get a range of items (0 to -1 means "everything")
LRANGE recent_orders 0 -1
# 1) "order:102"
# 2) "order:101"
# 3) "order:103"

# Trim the list to only keep the latest 5 items (very common pattern!)
LTRIM recent_orders 0 4
```

```js
// Node.js equivalent
await redis.lpush("recent_orders", "order:101"); // add to the front
await redis.rpush("recent_orders", "order:103"); // add to the end

const orders = await redis.lrange("recent_orders", 0, -1); // get all items
console.log(orders); // [ 'order:102', 'order:101', 'order:103' ]

await redis.ltrim("recent_orders", 0, 4); // keep only the latest 5 items
```

> We'll use Lists again in Module 15 (Queues).

---

## 4. Hashes

A hash is like storing a **JavaScript object** as the value — a key maps to multiple field-value pairs. Perfect for representing a single "record" (like a user profile) without needing to `JSON.stringify` and `JSON.parse` every time.

**Real use cases:** user profiles, product details, settings objects.

```bash
# HSET key field value [field value ...]
HSET user:15 name "Ashish" age 24 city "Bangalore"

# Get a single field
HGET user:15 name
# "Ashish"

# Get ALL fields and values
HGETALL user:15
# 1) "name"
# 2) "Ashish"
# 3) "age"
# 4) "24"
# 5) "city"
# 6) "Bangalore"

# Increment a numeric field directly (no need to read, modify, write yourself!)
HINCRBY user:15 age 1
# age is now 25
```

```js
// Node.js equivalent
await redis.hset("user:15", {
  name: "Ashish",
  age: 24,
  city: "Bangalore",
});

const name = await redis.hget("user:15", "name"); // "Ashish"

const user = await redis.hgetall("user:15");
// { name: 'Ashish', age: '24', city: 'Bangalore' }
// NOTE: Redis always returns strings — you'll need to parseInt() numbers yourself.

await redis.hincrby("user:15", "age", 1); // age becomes 25
```

**Why use a Hash instead of `JSON.stringify()` in a String?**

| Approach | Pros | Cons |
|---|---|---|
| String + JSON | Simple, works everywhere | Must fetch/rewrite the WHOLE object to change one field |
| Hash | Can update a single field directly and atomically (e.g., just `age`) | Slightly more commands to learn |

---

## 5. Sets

An **unordered collection of unique values** — just like a JavaScript `Set`. Duplicate values are automatically ignored.

**Real use cases:** unique visitors, tags, "users who liked this post," checking membership quickly.

```bash
# Add members to a set
SADD post:55:likes "user:1" "user:2" "user:3"

# Add the same user again — it won't duplicate
SADD post:55:likes "user:1"

# Check how many unique members
SCARD post:55:likes
# (integer) 3

# Check if a specific user liked the post — O(1) fast!
SISMEMBER post:55:likes "user:2"
# (integer) 1  (means true)

# Get all members
SMEMBERS post:55:likes
```

```js
// Node.js equivalent
await redis.sadd("post:55:likes", "user:1", "user:2", "user:3");

const count = await redis.scard("post:55:likes"); // 3 (unique likes)

const liked = await redis.sismember("post:55:likes", "user:2"); // 1 (true)

const allLikers = await redis.smembers("post:55:likes");
```

Sets also support powerful set-math operations directly in Redis:

```bash
SADD friends:ashish "priya" "raj" "meera"
SADD friends:priya  "raj" "meera" "sam"

# Common friends between Ashish and Priya
SINTER friends:ashish friends:priya
# 1) "raj"
# 2) "meera"
```

---

## 6. Sorted Sets (ZSets)

Like a Set, but every member also has a **score** attached, and Redis automatically keeps members sorted by that score. This is one of Redis's most powerful and unique data structures.

**Real use cases:** leaderboards, ranking systems, "top N" lists, priority queues.

```bash
# ZADD key score member
ZADD leaderboard 100 "ashish"
ZADD leaderboard 250 "priya"
ZADD leaderboard 180 "raj"

# Get everyone, sorted lowest → highest score
ZRANGE leaderboard 0 -1 WITHSCORES
# 1) "ashish"  2) "100"
# 3) "raj"     4) "180"
# 5) "priya"   6) "250"

# Get everyone, HIGHEST score first (typical leaderboard order)
ZREVRANGE leaderboard 0 -1 WITHSCORES

# Get a specific user's rank (0-indexed, based on ascending score)
ZRANK leaderboard "raj"
# (integer) 1

# Increase someone's score (e.g., they scored more points)
ZINCRBY leaderboard 50 "ashish"
# ashish's score is now 150
```

```js
// Node.js equivalent
await redis.zadd("leaderboard", 100, "ashish");
await redis.zadd("leaderboard", 250, "priya");
await redis.zadd("leaderboard", 180, "raj");

// Get top players, highest score first, with scores
const top = await redis.zrevrange("leaderboard", 0, -1, "WITHSCORES");
// [ 'priya', '250', 'raj', '180', 'ashish', '100' ]

await redis.zincrby("leaderboard", 50, "ashish"); // ashish now has 150
```

---

## 7. Quick Decision Table: Which Type Should I Use?

| I need to... | Use this type |
|---|---|
| Store a single simple value (token, flag, counter) | **String** |
| Store an ordered list of recent items / build a simple queue | **List** |
| Store an object-like record (user profile, product) | **Hash** |
| Track unique items, check membership quickly, do set math | **Set** |
| Build a leaderboard or anything that needs ranking by score | **Sorted Set (ZSet)** |

---

## 8. A Peek at Advanced Types (We'll use these later)

- **Streams** — an append-only log, great for event pipelines (similar to Kafka, but simpler).
- **Bitmaps** — extremely memory-efficient way to track yes/no flags for millions of users (e.g., "did user X log in today?").
- **HyperLogLog** — approximate counting of unique items using very little memory (e.g., "roughly how many unique visitors today?").
- **Geo** — store and query locations (e.g., "find all drivers within 5km").

We won't go deep into these in this course since they're less commonly needed day-to-day, but it's good to know they exist.

---

## 9. Interview Question: "What's the difference between a Set and a Sorted Set?"

**Answer:** A Set stores unique unordered members with O(1) membership checks. A Sorted Set also stores unique members, but each member has an associated numeric score, and Redis automatically maintains the members in sorted order by that score — which makes range queries (like "give me the top 10") extremely fast (O(log N)).

---

## 10. Summary

You now know Redis's 5 core data types:

- **String** — single values, counters, tokens
- **List** — ordered collections, feeds, simple queues
- **Hash** — object-like records
- **Set** — unique unordered collections, membership checks
- **Sorted Set** — ranked/scored collections, leaderboards

Choosing the right type for the job is a key skill — it directly affects performance and how clean your code is.

---

## 11. Homework

1. Using `redis-cli`, create a Hash representing your own profile (`name`, `age`, `city`) and fetch it with `HGETALL`.
2. Create a Sorted Set called `game:scores` with 3 players and use `ZREVRANGE` to print them highest-score-first.
3. In your own words, explain why you'd choose a Hash over storing a JSON string for a user profile.

---

**Next up:** [Module 5 — Redis Commands (Full Reference)](./05-redis-commands.md)
