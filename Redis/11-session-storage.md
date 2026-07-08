# Module 11: Session Storage

**Level:** Intermediate

Sessions are one of Redis's most common real-world jobs. Let's understand why, and build a real login-session system.

---

## 1. Why Do We Need "Sessions" At All?

HTTP is **stateless** — meaning each request is independent, and the server doesn't automatically "remember" who you are between requests. But apps need to know "is this user logged in, and who are they?" on every request.

**Two common approaches:**

1. **Server-side sessions** — the server stores session data (who's logged in, etc.) in some storage, and gives the browser a small random "session ID" (usually in a cookie). On each request, the browser sends the ID back, and the server looks up the full session data.
2. **JWT (stateless tokens)** — the server encodes user info directly into a signed token, and the browser sends the whole token back each time. No server-side storage needed (but harder to instantly revoke — we'll discuss this in Module 12's cousin topic, refresh tokens, later).

This module focuses on **approach 1**, using Redis as the session store — extremely common in real production Express apps.

---

## 2. Why Redis Specifically for Sessions?

- Sessions need to be read on **almost every single request** — this demands speed. Reading from a SQL database on every request would add noticeable latency to your entire app.
- Sessions are inherently **temporary** — Redis's built-in `EXPIRE` maps perfectly to "log the user out after N minutes of inactivity."
- Sessions are simple key-value-ish data — a great fit for Redis's Hash or String types.

```
Browser (has session ID cookie)
      │
      ▼
Node.js/Express  ──lookup──▶  Redis (session:abc123 → { userId: 15, role: "admin" })
      │
      ▼
   Response
```

---

## 3. Building It: `express-session` + Redis

The most common real-world approach uses the `express-session` middleware with a Redis-backed session store adapter.

```bash
npm install express-session connect-redis
```

```js
// app.js

const express = require("express");
const session = require("express-session");
const RedisStore = require("connect-redis").default; // adapter connecting express-session to Redis
const redis = require("./config/redisClient");

const app = express();

app.use(
  session({
    store: new RedisStore({ client: redis, prefix: "session:" }), // store sessions in Redis, keys like "session:abc123"
    secret: process.env.SESSION_SECRET || "change-this-in-production", // used to sign the session ID cookie
    resave: false,            // don't re-save session if nothing changed
    saveUninitialized: false, // don't create a session until something is actually stored in it
    cookie: {
      httpOnly: true,          // cookie can't be accessed via JavaScript (protects against XSS stealing it)
      secure: process.env.NODE_ENV === "production", // only send cookie over HTTPS in production
      maxAge: 1000 * 60 * 60, // cookie (and session) expires after 1 hour
    },
  })
);

app.use(express.json());
```

**Login route — creating a session:**

```js
app.post("/login", async (req, res) => {
  const { username, password } = req.body;

  // (In a real app: verify credentials against your database with hashed passwords!)
  const user = await verifyUserCredentials(username, password);

  if (!user) {
    return res.status(401).json({ error: "Invalid credentials" });
  }

  // Store whatever data you need on req.session —
  // express-session + connect-redis automatically saves this into Redis for you.
  req.session.userId = user.id;
  req.session.role = user.role;

  res.json({ message: "Logged in successfully" });
});
```

**Protected route — reading the session:**

```js
function requireLogin(req, res, next) {
  if (!req.session.userId) {
    return res.status(401).json({ error: "Please log in first" });
  }
  next();
}

app.get("/dashboard", requireLogin, (req, res) => {
  res.json({ message: `Welcome back, user #${req.session.userId}` });
});
```

**Logout route — destroying the session:**

```js
app.post("/logout", (req, res) => {
  // This tells connect-redis to DELETE the corresponding key in Redis
  req.session.destroy((err) => {
    if (err) return res.status(500).json({ error: "Could not log out" });
    res.clearCookie("connect.sid"); // clear the cookie on the browser too
    res.json({ message: "Logged out successfully" });
  });
});
```

---

## 4. What Actually Gets Stored in Redis?

If you peek into Redis after logging in:

```bash
redis-cli
127.0.0.1:6379> KEYS session:*
1) "session:abc123xyz"

127.0.0.1:6379> GET session:abc123xyz
"{\"cookie\":{...},\"userId\":15,\"role\":\"admin\"}"

127.0.0.1:6379> TTL session:abc123xyz
(integer) 3542   # seconds remaining, matches our maxAge of 1 hour
```

`connect-redis` automatically handles setting the TTL to match your cookie's `maxAge`, and automatically refreshes it (if `resave` behavior is configured that way) so active users don't get logged out mid-session.

---

## 5. Building a Manual Session System (Without a Library) — Understanding the Concept

Sometimes it's valuable to see this without the "magic" of a library, so you truly understand what's happening:

```js
const crypto = require("crypto");

// Generate a secure random session ID
function generateSessionId() {
  return crypto.randomBytes(32).toString("hex");
}

app.post("/login-manual", async (req, res) => {
  const user = await verifyUserCredentials(req.body.username, req.body.password);
  if (!user) return res.status(401).json({ error: "Invalid credentials" });

  const sessionId = generateSessionId();
  const sessionKey = `session:${sessionId}`;

  // Store session data as a Hash — clean, structured, and lets us update individual fields later
  await redis.hset(sessionKey, {
    userId: user.id,
    role: user.role,
  });

  // Sessions should always expire — 1 hour here
  await redis.expire(sessionKey, 3600);

  // Send the session ID to the browser as an httpOnly cookie
  res.cookie("sessionId", sessionId, {
    httpOnly: true,
    maxAge: 1000 * 60 * 60,
  });

  res.json({ message: "Logged in" });
});

async function requireLoginManual(req, res, next) {
  const sessionId = req.cookies.sessionId;
  if (!sessionId) return res.status(401).json({ error: "Not logged in" });

  const session = await redis.hgetall(`session:${sessionId}`);
  if (!session || !session.userId) {
    return res.status(401).json({ error: "Session expired or invalid" });
  }

  req.user = session; // attach to the request for use in the route handler
  next();
}
```

This is essentially what `express-session` + `connect-redis` do for you automatically, just with more polish (signing, security defaults, etc.).

---

## 6. "Log Out From All Devices" — A Common Real Feature

Because sessions are stored as individual Redis keys per login, you can easily support "log out everywhere":

```js
async function logoutAllSessions(userId) {
  // In a real system, you'd maintain a Set of session IDs per user, e.g.:
  // SADD user:15:sessions "sessionIdA" "sessionIdB"
  const sessionIds = await redis.smembers(`user:${userId}:sessions`);

  // Delete every session key for this user
  for (const sessionId of sessionIds) {
    await redis.del(`session:${sessionId}`);
  }

  await redis.del(`user:${userId}:sessions`); // clear the tracking set too
}
```

---

## 7. Interview Question: "Why store sessions in Redis instead of just using JWTs?"

**Answer:** JWTs are stateless (no server-side storage needed) but that's actually a double-edged sword: once issued, a JWT is valid until it expires — you can't easily force-invalidate a single JWT early (e.g., after logout or a stolen device) without extra infrastructure like a blacklist (which is itself... often stored in Redis!). Server-side sessions in Redis give you instant, precise control: delete the key, and that session is immediately dead — great for "log out now," banning a user, or reacting to a security incident.

---

## 8. Summary

- HTTP is stateless — sessions let the server "remember" a logged-in user across requests.
- Redis is ideal for session storage: fast reads on every request, and built-in expiry matches session timeouts naturally.
- `express-session` + `connect-redis` is the standard, batteries-included approach.
- Understanding the manual version (session ID + Hash + TTL + cookie) helps you see exactly what's happening under the hood.
- Redis-backed sessions make features like "log out everywhere" straightforward.

---

## 9. Homework

1. Build the `/login`, `/dashboard`, and `/logout` routes using `express-session` + `connect-redis`.
2. Inspect Redis directly with `redis-cli` after logging in — find your session key and check its TTL.
3. Implement a `/logout-all` route using the Set-based pattern shown above.

---

**Next up:** [Module 12 — Pub/Sub](./12-pubsub.md)
