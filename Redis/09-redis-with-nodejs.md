# Module 9: Redis with Node.js (Full Integration)

**Level:** Intermediate

Now we bring everything together and build a real Express.js app that talks to Redis properly — with a clean, reusable structure you can use in real projects.

---

## 1. Project Structure

```
redis-node-course/
├── config/
│   └── redisClient.js     # single shared Redis connection
├── routes/
│   └── userRoutes.js      # example routes using Redis
├── app.js                 # Express app setup
└── package.json
```

**Why a single shared client file?** You should create **one** Redis connection and reuse it across your whole app — never create a new connection per request. Connections are relatively expensive to set up, and Redis has a limit on how many simultaneous connections it will accept.

---

## 2. The Redis Client (Production-Ready Version)

```js
// config/redisClient.js

const Redis = require("ioredis");

// Create ONE Redis client for the entire application.
// In real projects, pull these values from environment variables, never hardcode them.
const redis = new Redis({
  host: process.env.REDIS_HOST || "127.0.0.1",
  port: process.env.REDIS_PORT || 6379,
  password: process.env.REDIS_PASSWORD || undefined,

  // Automatically retry connecting if Redis is temporarily unreachable.
  // This function is called every time a connection attempt fails.
  retryStrategy(times) {
    // "times" = number of failed attempts so far.
    // Wait a bit longer each time, but never more than 2 seconds.
    const delay = Math.min(times * 50, 2000);
    return delay; // ioredis will wait this many ms before retrying
  },

  // Reconnect automatically if the connection drops for certain errors
  reconnectOnError(err) {
    const targetError = "READONLY";
    if (err.message.includes(targetError)) {
      // Common during Redis failover — reconnect to find the new primary
      return true;
    }
    return false;
  },
});

redis.on("connect", () => console.log("✅ Redis connected"));
redis.on("ready", () => console.log("✅ Redis ready to accept commands"));
redis.on("error", (err) => console.error("❌ Redis error:", err.message));
redis.on("close", () => console.warn("⚠️ Redis connection closed"));
redis.on("reconnecting", () => console.log("🔄 Redis reconnecting..."));

module.exports = redis;
```

---

## 3. Setting Up Express

```js
// app.js

const express = require("express");
const userRoutes = require("./routes/userRoutes");

const app = express();

app.use(express.json()); // parse incoming JSON request bodies

app.use("/users", userRoutes);

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`🚀 Server running on http://localhost:${PORT}`);
});
```

---

## 4. A Real Example Route: Cache-Aside Pattern in Action

This is a practical preview of what we'll formalize in Module 10 — here we just want to see Redis and Express working together end-to-end.

```js
// routes/userRoutes.js

const express = require("express");
const router = express.Router();
const redis = require("../config/redisClient");

// Pretend this is a slow database call (simulated with a delay)
async function getUserFromDatabase(id) {
  console.log("🐢 Querying the (slow) database...");
  await new Promise((resolve) => setTimeout(resolve, 1000)); // simulate 1 second delay
  return { id, name: "Ashish", email: "ashish@example.com" };
}

router.get("/:id", async (req, res) => {
  const userId = req.params.id;
  const cacheKey = `user:${userId}`; // consistent naming convention: "type:id"

  try {
    // 1. Try Redis first
    const cachedUser = await redis.get(cacheKey);

    if (cachedUser) {
      console.log("⚡ Cache HIT");
      // Redis stores strings, so we parse the JSON back into an object
      return res.json({ source: "cache", data: JSON.parse(cachedUser) });
    }

    console.log("❌ Cache MISS");

    // 2. Not in Redis — fetch from the "database"
    const user = await getUserFromDatabase(userId);

    // 3. Store the result in Redis for next time, with a 1-hour expiry
    await redis.set(cacheKey, JSON.stringify(user), "EX", 3600);

    // 4. Return the fresh data
    res.json({ source: "database", data: user });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Something went wrong" });
  }
});

module.exports = router;
```

**Try it:**

```bash
# First request — takes ~1 second (cache miss, hits "database")
curl http://localhost:3000/users/15

# Second request — instant (cache hit!)
curl http://localhost:3000/users/15
```

---

## 5. Handling Redis Being Down Gracefully

A critical production lesson: **Redis being unavailable should not crash your entire app.** Since Redis is usually an accelerator (not your source of truth), your app should be able to fall back to the database if Redis has a hiccup.

```js
router.get("/:id", async (req, res) => {
  const userId = req.params.id;
  const cacheKey = `user:${userId}`;
  let cachedUser = null;

  try {
    cachedUser = await redis.get(cacheKey);
  } catch (err) {
    // Log it, but DON'T crash — just treat this like a cache miss
    console.error("⚠️ Redis unavailable, falling back to database:", err.message);
  }

  if (cachedUser) {
    return res.json({ source: "cache", data: JSON.parse(cachedUser) });
  }

  const user = await getUserFromDatabase(userId);

  // Also wrap the cache WRITE in a try/catch — a failed cache write
  // shouldn't prevent us from returning a correct response to the user
  try {
    await redis.set(cacheKey, JSON.stringify(user), "EX", 3600);
  } catch (err) {
    console.error("⚠️ Could not write to Redis cache:", err.message);
  }

  res.json({ source: "database", data: user });
});
```

This pattern — **"Redis is a nice-to-have, never a must-have for correctness"** — is one of the most important production lessons in this entire course.

---

## 6. Naming Convention for Keys (Very Important Habit)

Adopt a consistent naming pattern early. The industry-standard convention uses colons as separators, like a namespace:

```
type:id            → user:15
type:id:subtype    → user:15:sessions
feature:identifier → otp:9876543210
cache:route:param  → cache:products:page1
```

This keeps things organized and makes it easy to use `SCAN` patterns later (e.g., `SCAN 0 MATCH user:*` to find all user-related keys).

```js
// ✅ Good, consistent, self-documenting
const cacheKey = `user:${userId}`;
const otpKey = `otp:${phoneNumber}`;
const sessionKey = `session:${sessionId}`;
```

---

## 7. Interview Question: "Why create a single shared Redis connection instead of one per request?"

**Answer:** Establishing a TCP connection has overhead (handshake, authentication). Creating a new connection on every single request would be slow and would quickly exhaust Redis's max connection limit under real traffic. A single, long-lived, shared connection (or a small connection pool) reused across all requests is the standard, efficient pattern — this mirrors how you'd handle a database connection pool too.

---

## 8. Summary

- Create **one** shared Redis client and reuse it across your whole Express app.
- Use environment variables for connection details, never hardcode them.
- Build in reconnect/retry logic — Redis connections can drop, and your app should recover gracefully.
- Always wrap Redis calls in `try/catch` and fall back to the database on failure — Redis should never be a single point of failure for correctness.
- Use a consistent `type:id` key naming convention from day one.

---

## 9. Homework

1. Build the Express route above and confirm the second request to `/users/15` is noticeably faster than the first.
2. Temporarily stop your Redis server while your Express app is running, and confirm your app still responds correctly (just slower) instead of crashing.
3. Design (on paper) a key naming scheme for a blog app: posts, comments, and authors.

---

**Next up:** [Module 10 — Caching (Full Deep Dive)](./10-caching.md)
