# 01 — Singleton Pattern

> **Category:** GoF Creational  
> **Difficulty:** ⭐ Beginner  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Imagine your application needs a database connection. Every time some part of your code needs to talk to the database, should it create a brand new connection? No — that's wasteful and slow. You want **one shared connection** that everybody reuses.

That's the Singleton pattern: **a class that can only ever have one instance, ever, anywhere in your app.**

Think of it like the President of a country. There's only one. Everyone who needs to talk to "the President" talks to the same person — not a new President created each time.

---

## The Problem It Solves

Without Singleton, imagine this happening:

```
Request 1 arrives → creates DB connection #1
Request 2 arrives → creates DB connection #2
Request 3 arrives → creates DB connection #3
... 1000 requests → 1000 DB connections!
```

Database servers have connection limits. This crashes your app. You also waste memory, CPU, and time creating/destroying objects repeatedly.

**Singleton ensures:**
- Only ONE instance is ever created
- That same instance is shared across the whole application
- It's created lazily (only when first needed)

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    Your Application                      │
│                                                         │
│  ┌──────────────┐    getInstance()   ┌───────────────┐  │
│  │  UserService │ ──────────────────>│               │  │
│  └──────────────┘                   │   Database    │  │
│                                     │   Connection  │  │
│  ┌──────────────┐    getInstance()   │  (Singleton)  │  │
│  │ OrderService │ ──────────────────>│               │  │
│  └──────────────┘                   │  ✓ Same object│  │
│                                     │  ✓ One time   │  │
│  ┌──────────────┐    getInstance()   │    created    │  │
│  │  AuthService │ ──────────────────>│               │  │
│  └──────────────┘                   └───────────────┘  │
│                                                         │
│         All three get the EXACT SAME instance           │
└─────────────────────────────────────────────────────────┘
```

---

## How It Works (Step by Step)

1. Make the constructor **private** (so nobody can do `new DatabaseConnection()`)
2. Store the single instance in a **static variable** on the class itself
3. Provide a **static method** `getInstance()` that:
   - If instance doesn't exist → create it, store it, return it
   - If instance already exists → just return it

---

## Node.js Implementation

### Version 1: Classic Class-Based Singleton

```javascript
class DatabaseConnection {
  // Static variable to hold the single instance
  static #instance = null;

  // Private constructor — nobody outside can call new DatabaseConnection()
  #connection = null;

  constructor() {
    // Simulate connecting to a database
    this.#connection = {
      host: 'localhost',
      port: 5432,
      database: 'myapp',
      connectedAt: new Date().toISOString(),
    };
    console.log('🔌 Database connection established at', this.#connection.connectedAt);
  }

  // The ONLY way to get the instance
  static getInstance() {
    if (!DatabaseConnection.#instance) {
      // First time: create it
      DatabaseConnection.#instance = new DatabaseConnection();
    }
    // Every time: return the same one
    return DatabaseConnection.#instance;
  }

  // Methods on the singleton
  query(sql) {
    console.log(`Executing query: ${sql}`);
    return { rows: [], sql };
  }

  getStatus() {
    return {
      connected: true,
      host: this.#connection.host,
      connectedAt: this.#connection.connectedAt,
    };
  }
}

// --- How to use it ---

const db1 = DatabaseConnection.getInstance();
const db2 = DatabaseConnection.getInstance();
const db3 = DatabaseConnection.getInstance();

console.log(db1 === db2); // true  — same object!
console.log(db2 === db3); // true  — same object!

// Output:
// 🔌 Database connection established at 2024-01-15T10:00:00.000Z
// (Only printed ONCE — constructor ran only once)
// true
// true

db1.query('SELECT * FROM users');
// Executing query: SELECT * FROM users
```

---

### Version 2: The Node.js Module Way (Most Common in Practice)

Here's a secret: **Node.js modules ARE singletons by default.**

When you `require()` or `import` a module, Node.js caches it. Every `require()` of the same file returns the same cached export.

```javascript
// db.js — This file IS the singleton
const { Pool } = require('pg'); // PostgreSQL client

class Database {
  constructor() {
    this.pool = new Pool({
      host: process.env.DB_HOST || 'localhost',
      port: process.env.DB_PORT || 5432,
      database: process.env.DB_NAME || 'myapp',
      user: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASSWORD,
      max: 20,              // max 20 connections in pool
      idleTimeoutMillis: 30000,
    });

    // Handle pool errors
    this.pool.on('error', (err) => {
      console.error('Unexpected DB pool error:', err);
    });

    console.log('✅ Database pool created');
  }

  async query(text, params) {
    const start = Date.now();
    try {
      const result = await this.pool.query(text, params);
      const duration = Date.now() - start;
      console.log(`Query executed in ${duration}ms | rows: ${result.rowCount}`);
      return result;
    } catch (error) {
      console.error('Query error:', error);
      throw error;
    }
  }

  async getClient() {
    return this.pool.connect();
  }

  async close() {
    await this.pool.end();
    console.log('Database pool closed');
  }
}

// Export a single instance — this IS the singleton
module.exports = new Database();
```

```javascript
// userService.js
const db = require('./db'); // Gets the same instance every time

async function getUserById(id) {
  const result = await db.query('SELECT * FROM users WHERE id = $1', [id]);
  return result.rows[0];
}

module.exports = { getUserById };
```

```javascript
// orderService.js
const db = require('./db'); // Same db instance as userService!

async function getOrdersByUser(userId) {
  const result = await db.query(
    'SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC',
    [userId]
  );
  return result.rows;
}

module.exports = { getOrdersByUser };
```

```javascript
// app.js
const db1 = require('./db');
const db2 = require('./db');

console.log(db1 === db2); // true — Node.js module cache at work
// "✅ Database pool created" — printed only ONCE
```

---

### Version 3: Configuration Singleton (Real-World Example)

```javascript
// config.js
const fs = require('fs');
const path = require('path');

class Config {
  static #instance = null;
  #settings = {};

  constructor() {
    this.#loadConfig();
  }

  #loadConfig() {
    // Load from environment variables
    this.#settings = {
      server: {
        port: parseInt(process.env.PORT) || 3000,
        host: process.env.HOST || 'localhost',
        env: process.env.NODE_ENV || 'development',
      },
      database: {
        url: process.env.DATABASE_URL,
        maxConnections: parseInt(process.env.DB_MAX_CONN) || 10,
      },
      auth: {
        jwtSecret: process.env.JWT_SECRET || 'dev-secret',
        jwtExpiry: process.env.JWT_EXPIRY || '7d',
      },
      redis: {
        url: process.env.REDIS_URL || 'redis://localhost:6379',
      },
    };

    console.log(`⚙️  Config loaded for environment: ${this.#settings.server.env}`);
  }

  static getInstance() {
    if (!Config.#instance) {
      Config.#instance = new Config();
    }
    return Config.#instance;
  }

  get(key) {
    // Support dot notation: config.get('server.port')
    return key.split('.').reduce((obj, k) => obj?.[k], this.#settings);
  }

  getAll() {
    return { ...this.#settings }; // Return a copy, not the original
  }

  isDevelopment() {
    return this.#settings.server.env === 'development';
  }

  isProduction() {
    return this.#settings.server.env === 'production';
  }
}

// Usage
const config = Config.getInstance();

console.log(config.get('server.port'));    // 3000
console.log(config.get('auth.jwtSecret')); // 'dev-secret'
console.log(config.isDevelopment());       // true

module.exports = Config.getInstance();
// OR: module.exports = Config; — and let consumers call getInstance()
```

---

### Version 4: Logger Singleton

```javascript
// logger.js
const { createWriteStream } = require('fs');
const path = require('path');

class Logger {
  static #instance = null;
  #logStream;
  #logLevel;

  constructor() {
    this.#logLevel = process.env.LOG_LEVEL || 'info';

    // Single log file stream for the whole app
    if (process.env.NODE_ENV === 'production') {
      this.#logStream = createWriteStream(
        path.join(process.cwd(), 'app.log'),
        { flags: 'a' } // append mode
      );
    }
  }

  static getInstance() {
    if (!Logger.#instance) {
      Logger.#instance = new Logger();
    }
    return Logger.#instance;
  }

  #formatMessage(level, message, meta = {}) {
    return JSON.stringify({
      timestamp: new Date().toISOString(),
      level,
      message,
      ...meta,
    });
  }

  #write(level, message, meta) {
    const formatted = this.#formatMessage(level, message, meta);

    if (this.#logStream) {
      this.#logStream.write(formatted + '\n');
    } else {
      // Development: colorful console output
      const colors = { error: '\x1b[31m', warn: '\x1b[33m', info: '\x1b[36m', debug: '\x1b[90m' };
      const reset = '\x1b[0m';
      console.log(`${colors[level] || ''}${formatted}${reset}`);
    }
  }

  info(message, meta)  { this.#write('info',  message, meta); }
  warn(message, meta)  { this.#write('warn',  message, meta); }
  error(message, meta) { this.#write('error', message, meta); }
  debug(message, meta) { this.#write('debug', message, meta); }
}

module.exports = Logger.getInstance();
```

```javascript
// Anywhere in your app:
const logger = require('./logger');

logger.info('Server started', { port: 3000 });
logger.error('DB connection failed', { host: 'localhost', attempt: 3 });
```

---

## When to Use Singleton

| Situation | Use Singleton? | Why |
|-----------|---------------|-----|
| Database connection pool | ✅ Yes | Expensive to create, should be shared |
| App configuration | ✅ Yes | Read once, use everywhere |
| Logger | ✅ Yes | One stream/file for the whole app |
| Cache (in-memory) | ✅ Yes | Must be shared or it's useless |
| HTTP client (axios instance) | ✅ Yes | Reuse connection pooling |
| User object | ❌ No | Each user is different |
| Request handler | ❌ No | Each request is separate |
| Service with no state | ⚠️ Maybe | Module exports work fine |

---

## Common Mistakes & Pitfalls

### Mistake 1: Not handling async initialization

```javascript
// ❌ BAD — constructor can't be async
class BadDB {
  constructor() {
    this.connection = await connect(); // SyntaxError!
  }
}

// ✅ GOOD — use a factory method or lazy init
class GoodDB {
  static #instance = null;
  #connection = null;

  static async getInstance() {
    if (!GoodDB.#instance) {
      GoodDB.#instance = new GoodDB();
      await GoodDB.#instance.#init();
    }
    return GoodDB.#instance;
  }

  async #init() {
    this.#connection = await connect();
  }
}

// Usage
const db = await GoodDB.getInstance();
```

### Mistake 2: Singleton in tests (hardest problem)

```javascript
// ❌ BAD — tests share state, cause flaky tests
test('user query', async () => {
  const db = DatabaseConnection.getInstance();
  // This is the REAL db from another test!
});

// ✅ GOOD — dependency injection makes testing easier
class UserService {
  constructor(db) { // inject the dependency
    this.db = db;
  }

  async getUser(id) {
    return this.db.query('SELECT * FROM users WHERE id = $1', [id]);
  }
}

// In tests:
const mockDb = { query: jest.fn().mockResolvedValue({ rows: [{ id: 1, name: 'Alice' }] }) };
const service = new UserService(mockDb);
```

### Mistake 3: Race conditions (Node.js is usually safe, but…)

```javascript
// In a single Node.js process, this is fine because JS is single-threaded
// But if you use Worker Threads, each thread gets its own singleton!

const { Worker, isMainThread } = require('worker_threads');

// ⚠️ Each Worker thread has its own module cache = its own singleton
```

---

## Interview Questions & Answers

**Q: What is the Singleton pattern?**
> A class that ensures only one instance exists throughout the application lifetime and provides a global access point to it.

**Q: Is the Singleton pattern good or bad?**
> It's a trade-off. Good for truly shared resources (DB pool, config, logger). Bad when overused — it creates hidden global state, makes testing harder, and tight coupling.

**Q: How does Node.js module caching relate to Singleton?**
> Node.js caches modules after first `require()`. So `module.exports = new MyClass()` is effectively a Singleton — same instance returned every time.

**Q: When would you NOT use Singleton?**
> - When you need multiple instances (e.g., connecting to two different databases)
> - When it makes testing harder (use DI instead)
> - When you need per-request state (use DI with request scope)

**Q: How do you make a Singleton thread-safe?**
> In Node.js, single-threaded JS means it's automatically safe. In Java, you'd use `synchronized` or the "initialization-on-demand holder" pattern. In Node.js with Worker Threads, use a shared `SharedArrayBuffer` or IPC.

---

## Real-World Usage in Popular Libraries

- **Mongoose** (`mongoose.connect()`) — single connection shared across models
- **Winston logger** — default logger is a singleton
- **Express app** — `app` object is typically a singleton passed around
- **Redis client** — one client instance for the whole app
- **dotenv** — `require('dotenv').config()` loads once, cached

---

## Summary

```
Singleton Pattern
├── What: One instance, globally accessible
├── Why: Expensive resources, shared state, coordination point
├── How: Static getInstance() + private constructor
├── Node.js way: module.exports = new MyClass() (free Singleton!)
└── Watch out: Testing difficulty, hidden globals, Worker Threads
```
