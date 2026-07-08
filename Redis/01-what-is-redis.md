# Module 1: What is Redis?

**Level:** Beginner

---

## 1. The Problem We're Trying to Solve

Imagine your website has **1 million users**, and every user does this:

```
GET /profile
```

Every time this happens, your server does this:

```
User → Node.js → PostgreSQL → Node.js → User
```

With 10 users, this is fine.
With 10,000 users, your database starts to slow down.
With 100,000 users, your database can start to crumble.

**Why?** Because every single request is hitting the database, and databases read from **disk**, which is slow compared to memory.

---

## 2. Disk vs RAM (The Core Idea Behind Redis)

Think of it like a library.

- **Disk (Database)** = A cupboard full of books. To answer a question, you have to walk to the cupboard, search for the right book, open it, find the page, and read the answer.
- **RAM (Redis)** = A notebook already open on your desk. You just glance at it. Instant answer.

Roughly speaking:

| Storage | Speed |
|---|---|
| Disk (Database) | Milliseconds |
| RAM (Redis) | Microseconds |

That's about **100–1000x faster**, depending on the workload.

---

## 3. So, What Exactly is Redis?

**Redis** = **RE**mote **DI**ctionary **S**erver

It is:

- An **in-memory data store** (data lives in RAM, not on disk)
- A **cache**
- A **message broker** (for Pub/Sub messaging)
- A **key-value store**

Most companies use it primarily as a **cache** — a fast, temporary layer sitting in front of a slower database.

---

## 4. What Does "Key-Value" Mean?

It's the simplest possible way to store data — just like a JavaScript object or a `Map`.

```js
// A plain JS object is a key-value store
const user = {
  name: "Ashish",
  city: "Bangalore"
};

// Redis works the same way, just outside your app, in its own memory space:
// KEY           VALUE
// "user:name"   "Ashish"
// "user:city"   "Bangalore"
```

Instead of running a SQL query like:

```sql
SELECT * FROM users WHERE id = 15;
```

Redis just does a direct lookup:

```
KEY:   user:15
VALUE: { "id": 15, "name": "Ashish", "email": "ashish@example.com" }
```

There's no searching, no scanning, no joins — Redis jumps straight to the value using the key. This is why it's so fast (mostly **O(1)**, meaning constant time, regardless of how much data you have).

---

## 5. Why Not Store Everything in Redis?

Because **RAM is expensive**, and disk is cheap.

| Storage | Cost | Size |
|---|---|---|
| Hard Disk | Cheap | 1 TB+ easily |
| RAM | Expensive | Usually much smaller, e.g. 16–64 GB |

So Redis is best used for:

- Frequently accessed data (things read over and over)
- Temporary data (things that expire, like OTPs)
- Session data
- Cached query results
- Tokens (JWT, refresh tokens)
- Leaderboards
- Rate-limiting counters

**Permanent, "source of truth" business data** (orders, user accounts, transactions) still belongs in a real database like PostgreSQL or MongoDB.

---

## 6. Real World Example: Instagram Followers

Millions of people check "How many followers does this account have?" every second.

Follower counts don't change every millisecond. If Instagram queried the database on every single view, that would mean **millions of unnecessary SQL queries** per second.

Instead, the flow looks like this:

```
Database (source of truth)
      │
      ▼
   Redis (fast copy)
      │
      ▼
    User (sees the number instantly)
```

---

## 7. Restaurant Analogy

**Without caching:**

```
Customer → Chef → Kitchen → Cook from scratch → Serve
```

Every order takes full preparation time.

**With caching:**

```
Customer → Counter → Already-prepared food → Serve
```

The food is already made, so the wait is nearly zero. This is exactly what Redis does for data.

---

## 8. Where Redis Fits in a Node.js App

```
        Client
          │
     HTTP Request
          │
      Node.js API
          │
   ┌─────────────┐
   │ Check Redis │
   └──────┬──────┘
          │
   Found? ┼── Yes → Return cached data → Done (fast!)
          │
          └── No  → Query PostgreSQL
                        │
                        ▼
                 Store result in Redis
                        │
                        ▼
                 Return response to user
```

This pattern — check Redis first, fall back to the database, then save the result back to Redis — is called the **Cache-Aside Pattern**. It's the most common caching strategy in the industry, and we'll build it with real code in Module 10.

---

## 9. Who Actually Uses Redis?

Almost every large tech company uses Redis somewhere in their stack, including companies like Netflix, Uber, Airbnb, Discord, and GitHub. Common uses:

- Caching
- Session storage
- Rate limiting
- Pub/Sub messaging (real-time features)
- Background job queues

---

## 10. Where You'll Use Redis in a Node.js/Express App

```
Express API
     │
Authentication
     │
   JWT
     │
  Redis
     │
   User
```

Typical real use cases we'll build in this course:

- OTP storage
- Email verification tokens
- Password reset tokens
- Refresh token blacklists (logging users out securely)
- User sessions
- API response caching
- Rate limiting middleware
- Job queues (using BullMQ, which is built on Redis)
- Real-time notifications (Pub/Sub)
- Distributed locks (preventing race conditions across multiple servers)

---

## 11. Redis vs PostgreSQL — Side by Side

| | PostgreSQL | Redis |
|---|---|---|
| Storage | Disk | Mainly RAM |
| Speed | Slower | Extremely fast |
| Purpose | Permanent storage | Temporary / cached storage (can also persist) |
| Query style | SQL | Simple key-based operations |
| Relationships | Complex joins | No joins |
| Role | Source of truth | Fast-access layer |

Think of PostgreSQL as the **master copy** and Redis as the **fast copy**:

```
PostgreSQL (Master Copy)
        │
        ▼
     Redis (Fast Copy)
```

**Important rule:** Redis should almost never be your only/primary database for critical business data — treat it as an accelerator sitting in front of your real database.

---

## 12. Interview Question: "Why is Redis so fast?"

A strong answer covers these points:

1. Data lives in **RAM**, not on disk — RAM access is orders of magnitude faster.
2. Most Redis operations are **O(1)** (constant time) — the operation takes the same time whether you have 10 keys or 10 million.
3. Redis is written in **C** and uses highly optimized, purpose-built data structures internally.
4. It uses a lightweight, efficient event-driven model for handling requests, avoiding unnecessary overhead (we'll dig into this more in Module 6).

---

## 13. Summary

In this module, you learned:

- What Redis is (an in-memory key-value data store)
- Why Redis exists (databases are slow under heavy read load)
- The difference between RAM and disk
- What "key-value" storage means
- Where Redis fits inside a typical Node.js/Express architecture
- Why Redis is so fast
- Real production use cases

---

## 14. Homework

Before moving to Module 2, think through these (no need to write essays, just make sure you can explain them out loud):

1. Why is RAM faster than disk?
2. Why shouldn't we store *all* application data in Redis?
3. What is a key-value database, in your own words?
4. Where does Redis sit in a typical Node.js request flow?
5. Name three types of data that are good candidates for caching.

---

**Next up:** [Module 2 — Why Redis Exists (Deep Dive)](./02-why-redis-exists.md)
