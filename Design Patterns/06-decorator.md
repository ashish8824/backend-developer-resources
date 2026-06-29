# 06 — Decorator Pattern

> **Category:** GoF Structural  
> **Difficulty:** ⭐⭐⭐ Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Think of a plain coffee. You can add milk → it becomes latte. Add caramel → caramel latte. Add whipped cream → caramel latte with cream. Each addition **wraps** the previous coffee and adds something new, but at the end you still have a coffee with a `cost()` and a `description()`.

The **Decorator pattern** lets you add new behavior to an object **at runtime** by wrapping it — without changing the original class, and without creating a new subclass for every combination.

**In one sentence:** Wrap an object inside another object that has the same interface but adds extra behavior before/after the core operation.

---

## The Problem It Solves

Imagine a `UserService` with a `getUser()` method. Now you want to:
- Log every call
- Cache results
- Check if the user is authorized
- Measure how long it takes

Without Decorator, you'd stuff ALL of this into `UserService`. Or you'd create:
- `LoggedUserService`
- `CachedUserService`
- `AuthorizedUserService`
- `LoggedAndCachedUserService`
- `LoggedCachedAndAuthorizedUserService`
- … 💥 class explosion

With Decorator, you stack wrappers:

```javascript
const service = new AuthDecorator(
  new CacheDecorator(
    new LoggingDecorator(
      new UserService()   // core
    )
  )
);
```

Same interface. Behavior composed at runtime. Core class stays clean.

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      Decorator Pattern                           │
│                                                                 │
│  Outer (Auth Check)                                             │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Middle (Cache)                                        │    │
│  │  ┌─────────────────────────────────────────────────┐  │    │
│  │  │  Inner (Logger)                                 │  │    │
│  │  │  ┌──────────────────────────────────────────┐  │  │    │
│  │  │  │  Core: UserService                       │  │  │    │
│  │  │  │  getUser(id) { ...real DB query... }     │  │  │    │
│  │  │  └──────────────────────────────────────────┘  │  │    │
│  │  │  + logs before/after                           │  │    │
│  │  └─────────────────────────────────────────────────┘  │    │
│  │  + checks cache before calling inner                  │    │
│  └────────────────────────────────────────────────────────┘    │
│  + checks auth before calling middle                           │
│                                                                 │
│  getUser(id) call flow:                                         │
│  Auth → Cache → Logger → UserService → Logger → Cache → Auth   │
│  (check)  (hit?)  (log)    (DB query)   (log)   (store)        │
└─────────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Version 1: Service Decorator Stack

```javascript
// ── Core Service (stays clean, just does its job) ──

class UserService {
  constructor(db) {
    this.db = db;
  }

  async getUser(id) {
    // Just the core business logic
    const user = await this.db.query('SELECT * FROM users WHERE id = $1', [id]);
    if (!user.rows.length) throw new Error(`User ${id} not found`);
    return user.rows[0];
  }

  async updateUser(id, data) {
    const result = await this.db.query(
      'UPDATE users SET name=$1, email=$2 WHERE id=$3 RETURNING *',
      [data.name, data.email, id]
    );
    return result.rows[0];
  }

  async deleteUser(id) {
    await this.db.query('DELETE FROM users WHERE id = $1', [id]);
    return { success: true };
  }
}

// ── Base Decorator — wraps any UserService-compatible object ──

class UserServiceDecorator {
  constructor(service) {
    this._service = service; // the wrapped service
  }

  // By default, pass through to the wrapped service
  async getUser(id)            { return this._service.getUser(id); }
  async updateUser(id, data)   { return this._service.updateUser(id, data); }
  async deleteUser(id)         { return this._service.deleteUser(id); }
}

// ── Decorator 1: Logging ──

class LoggingDecorator extends UserServiceDecorator {
  constructor(service, logger = console) {
    super(service);
    this.logger = logger;
  }

  async getUser(id) {
    this.logger.info(`[LOG] getUser called with id=${id}`);
    const start = Date.now();
    try {
      const result = await this._service.getUser(id); // call the inner service
      this.logger.info(`[LOG] getUser success | id=${id} | ${Date.now() - start}ms`);
      return result;
    } catch (error) {
      this.logger.error(`[LOG] getUser failed | id=${id} | error=${error.message}`);
      throw error;
    }
  }

  async updateUser(id, data) {
    this.logger.info(`[LOG] updateUser called | id=${id}`, { data });
    const result = await this._service.updateUser(id, data);
    this.logger.info(`[LOG] updateUser success | id=${id}`);
    return result;
  }

  async deleteUser(id) {
    this.logger.warn(`[LOG] deleteUser called | id=${id}`); // warn for deletes!
    const result = await this._service.deleteUser(id);
    this.logger.warn(`[LOG] deleteUser success | id=${id}`);
    return result;
  }
}

// ── Decorator 2: Caching ──

class CachingDecorator extends UserServiceDecorator {
  constructor(service, cache) {
    super(service);
    this.cache = cache; // e.g., a Redis client or in-memory Map
    this.ttl = 60; // seconds
  }

  #cacheKey(id) { return `user:${id}`; }

  async getUser(id) {
    const key = this.#cacheKey(id);

    // Try cache first
    const cached = await this.cache.get(key);
    if (cached) {
      console.log(`[CACHE] HIT for ${key}`);
      return JSON.parse(cached);
    }

    // Cache miss — call inner service
    console.log(`[CACHE] MISS for ${key}`);
    const user = await this._service.getUser(id);

    // Store in cache
    await this.cache.set(key, JSON.stringify(user), this.ttl);
    return user;
  }

  async updateUser(id, data) {
    const result = await this._service.updateUser(id, data);
    // Invalidate cache after update
    await this.cache.del(this.#cacheKey(id));
    console.log(`[CACHE] Invalidated cache for user:${id}`);
    return result;
  }

  async deleteUser(id) {
    const result = await this._service.deleteUser(id);
    await this.cache.del(this.#cacheKey(id));
    console.log(`[CACHE] Invalidated cache for user:${id}`);
    return result;
  }
}

// ── Decorator 3: Authorization ──

class AuthDecorator extends UserServiceDecorator {
  constructor(service, authContext) {
    super(service);
    this.authContext = authContext; // { currentUser, permissions }
  }

  #requirePermission(action, resourceId) {
    const { currentUser, permissions } = this.authContext;
    const allowed = permissions.includes(action) || currentUser.id === resourceId;
    if (!allowed) {
      throw new Error(`Unauthorized: user ${currentUser.id} cannot ${action} resource ${resourceId}`);
    }
  }

  async getUser(id) {
    this.#requirePermission('read:users', id);
    return this._service.getUser(id);
  }

  async updateUser(id, data) {
    this.#requirePermission('write:users', id);
    return this._service.updateUser(id, data);
  }

  async deleteUser(id) {
    this.#requirePermission('delete:users', id);
    return this._service.deleteUser(id);
  }
}

// ── Decorator 4: Validation ──

class ValidationDecorator extends UserServiceDecorator {
  async updateUser(id, data) {
    // Validate before calling inner service
    if (!data.name || data.name.trim().length < 2) {
      throw new Error('Name must be at least 2 characters');
    }
    if (!data.email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(data.email)) {
      throw new Error('Invalid email address');
    }
    return this._service.updateUser(id, data);
  }
}

// ── Compose them all together ──

const db = { query: async () => ({ rows: [{ id: 1, name: 'Alice', email: 'alice@test.com' }] }) };
const cache = new Map(); // simple in-memory cache
const cacheAdapter = {
  get: async (k) => cache.get(k) || null,
  set: async (k, v) => cache.set(k, v),
  del: async (k) => cache.delete(k),
};
const authContext = { currentUser: { id: 1 }, permissions: ['read:users', 'write:users'] };

const userService = new AuthDecorator(
  new CachingDecorator(
    new LoggingDecorator(
      new ValidationDecorator(
        new UserService(db)
      )
    ),
    cacheAdapter
  ),
  authContext
);

// Now every call goes through: Auth → Cache → Logger → Validation → UserService
const user = await userService.getUser(1);
// [LOG] getUser called with id=1
// [CACHE] MISS for user:1
// [LOG] getUser success | id=1 | 2ms

const user2 = await userService.getUser(1);
// [LOG] getUser called with id=1
// [CACHE] HIT for user:1
// [LOG] getUser success | id=1 | 0ms
```

---

### Version 2: Express Middleware IS the Decorator Pattern

This is the most important thing to understand: **Express middleware is the Decorator pattern in disguise.**

```javascript
const express = require('express');
const app = express();

// Each middleware WRAPS the request/response with extra behavior
// and calls next() to pass to the inner handler

// Decorator 1: Request logging
function requestLogger(req, res, next) {
  const start = Date.now();
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.path}`);
  res.on('finish', () => {
    console.log(`[${new Date().toISOString()}] ${req.method} ${req.path} ${res.statusCode} ${Date.now() - start}ms`);
  });
  next(); // call the next "inner" handler
}

// Decorator 2: Authentication
function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'No token provided' });

  try {
    req.user = verifyJWT(token); // attach user to request
    next();
  } catch {
    res.status(401).json({ error: 'Invalid token' });
  }
}

// Decorator 3: Rate limiting
function rateLimit(maxPerMinute) {
  const counts = new Map();
  return (req, res, next) => {
    const key = req.ip;
    const count = (counts.get(key) || 0) + 1;
    counts.set(key, count);
    setTimeout(() => counts.set(key, Math.max(0, counts.get(key) - 1)), 60000);
    if (count > maxPerMinute) return res.status(429).json({ error: 'Too many requests' });
    next();
  };
}

// Decorator 4: Response caching
function cacheResponse(ttlSeconds) {
  const cache = new Map();
  return (req, res, next) => {
    const key = req.path;
    if (cache.has(key)) {
      console.log(`[CACHE] Serving cached response for ${key}`);
      return res.json(cache.get(key));
    }
    const originalJson = res.json.bind(res); // save original
    res.json = (data) => {                   // wrap res.json
      cache.set(key, data);
      setTimeout(() => cache.delete(key), ttlSeconds * 1000);
      originalJson(data);
    };
    next();
  };
}

// Stack them — each wraps the next
app.use(requestLogger);         // outermost decorator
app.use(rateLimit(60));         // 60 requests per minute
app.get('/users/:id',
  authenticate,                 // auth decorator (route-level)
  cacheResponse(30),            // cache for 30 seconds
  async (req, res) => {         // the actual handler (core)
    const user = await db.findUser(req.params.id);
    res.json(user);
  }
);
```

---

### Version 3: Function Decorator (Functional Style)

```javascript
// Higher-order functions are decorators!

// ── Core function ──
async function fetchUserFromDB(id) {
  // expensive DB call
  return { id, name: 'Alice', email: 'alice@example.com' };
}

// ── Decorator factory: adds logging ──
function withLogging(fn, fnName = fn.name) {
  return async function(...args) {
    console.log(`[LOG] Calling ${fnName}(${args.join(', ')})`);
    const start = Date.now();
    try {
      const result = await fn(...args);
      console.log(`[LOG] ${fnName} completed in ${Date.now() - start}ms`);
      return result;
    } catch (error) {
      console.error(`[LOG] ${fnName} failed: ${error.message}`);
      throw error;
    }
  };
}

// ── Decorator factory: adds caching ──
function withCache(fn, ttlMs = 60000) {
  const cache = new Map();
  return async function(...args) {
    const key = JSON.stringify(args);
    if (cache.has(key)) {
      const { value, expiresAt } = cache.get(key);
      if (Date.now() < expiresAt) {
        console.log(`[CACHE] Hit for key: ${key}`);
        return value;
      }
      cache.delete(key);
    }
    const result = await fn(...args);
    cache.set(key, { value: result, expiresAt: Date.now() + ttlMs });
    return result;
  };
}

// ── Decorator factory: adds retry ──
function withRetry(fn, maxRetries = 3, delayMs = 1000) {
  return async function(...args) {
    let lastError;
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await fn(...args);
      } catch (error) {
        lastError = error;
        if (attempt < maxRetries) {
          console.warn(`[RETRY] Attempt ${attempt} failed. Retrying in ${delayMs}ms...`);
          await new Promise(r => setTimeout(r, delayMs * attempt));
        }
      }
    }
    throw lastError;
  };
}

// ── Compose decorators (right to left) ──
const fetchUser = withLogging(
  withCache(
    withRetry(fetchUserFromDB, 3),
    30000  // cache 30 seconds
  )
);

// Call it — works exactly like fetchUserFromDB but with logging, caching, retry
const user = await fetchUser(1);
// [LOG] Calling fetchUserFromDB(1)
// [LOG] fetchUserFromDB completed in 5ms

const user2 = await fetchUser(1);
// [LOG] Calling fetchUserFromDB(1)
// [CACHE] Hit for key: [1]
// [LOG] fetchUserFromDB completed in 0ms

// ── Utility: compose multiple decorators cleanly ──
function compose(...decorators) {
  return (fn) => decorators.reduceRight((acc, decorator) => decorator(acc), fn);
}

const enhance = compose(
  (fn) => withLogging(fn),
  (fn) => withCache(fn, 60000),
  (fn) => withRetry(fn, 3)
);

const enhancedFetchUser = enhance(fetchUserFromDB);
```

---

## When to Use Decorator

| Situation | Use? | Why |
|-----------|------|-----|
| Adding cross-cutting concerns (logging, caching, auth) | ✅ Yes | Core use case |
| Want to add behavior without modifying original class | ✅ Yes | Open/Closed Principle |
| Need different combinations of behaviors | ✅ Yes | Avoids class explosion |
| Express middleware | ✅ Yes | It IS the Decorator pattern |
| Simple one-time behavior addition | ❌ Maybe | Just extend the class |

---

## Decorator vs Other Patterns

| Pattern | What it does |
|---------|-------------|
| **Decorator** | Same interface, adds behavior around the core |
| **Adapter** | Different interface, translates calls |
| **Proxy** | Same interface, controls ACCESS (auth, lazy load, logging) |
| **Facade** | Simplifies multiple complex interfaces into one |

> Decorator and Proxy look very similar. The difference is intent: Proxy controls access/lifecycle; Decorator adds responsibilities.

---

## Interview Questions & Answers

**Q: What is the Decorator pattern?**
> Attaches additional behavior to an object dynamically by wrapping it in decorator objects that share the same interface. Alternatives to subclassing for extending functionality.

**Q: How is Express middleware related to Decorator?**
> Each middleware wraps the request/response pipeline and calls `next()` to invoke the inner handler — exactly like a decorator calling the wrapped object's method. Middleware IS the Decorator pattern applied to HTTP handling.

**Q: What's the difference between Decorator and inheritance?**
> Inheritance is static (compile-time). Decorator is dynamic (runtime). With Decorator you compose behaviors at runtime; with inheritance, you bake combinations into class names at design time, causing class explosion.

**Q: What does "same interface" mean in Decorator?**
> Every decorator implements the same interface as the object it wraps. So client code can't tell if it's talking to the real object or a stack of decorators — they all look the same from outside.

---

## Summary

```
Decorator Pattern
├── What: Wrap objects to add behavior, keeping the same interface
├── Why: Cross-cutting concerns without polluting core logic
├── How: Decorator class wraps inner service, calls it, adds before/after behavior
├── Node.js: Express middleware, function wrappers (withLogging, withCache, withRetry)
└── Key: Stack decorators like Russian dolls — outermost runs first
```
