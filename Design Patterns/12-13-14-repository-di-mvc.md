# 12 — Repository Pattern

> **Category:** Application Design  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Imagine a library. You don't go into the shelves and search for a book yourself. You go to the **librarian** and say: "I want the book called 'Clean Code'." The librarian knows where it is, how to find it, and brings it to you. You don't care if it's shelf 3 or in a digital system.

The **Repository pattern** is that librarian for your data. Your business logic says "give me user with id 5" and the Repository knows whether to look in PostgreSQL, MongoDB, Redis, or a flat file — your business code doesn't care.

**In one sentence:** Repository is an abstraction layer between your business logic and your data access layer — it looks like a collection of objects but hides all database details.

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      Repository Pattern                          │
│                                                                 │
│  Business Logic           Repository           Data Source      │
│  ─────────────            ──────────           ───────────      │
│                                                                 │
│  userService           UserRepository                           │
│  .getAdmins()  ──────> .findByRole('admin') ──> PostgreSQL      │
│                                                                 │
│  orderService          OrderRepository                          │
│  .getHistory() ──────> .findByUserId(id) ───> MongoDB           │
│                                                                 │
│  ↑ Business logic          ↑ Knows HOW          ↑ Actual DB    │
│    knows WHAT it wants       to get data                        │
│    but NOT how to get it                                        │
│                                                                 │
│  Swap PostgreSQL → MongoDB? Only change the Repository.         │
│  Business logic stays IDENTICAL.                                │
└─────────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Step 1: Define the Repository Interface

```javascript
// repositories/IUserRepository.js
// This is the "contract" — what any user repository must provide

class IUserRepository {
  async findById(id)                    { throw new Error('Not implemented'); }
  async findByEmail(email)              { throw new Error('Not implemented'); }
  async findAll(filters, options)       { throw new Error('Not implemented'); }
  async findByRole(role)                { throw new Error('Not implemented'); }
  async save(user)                      { throw new Error('Not implemented'); } // create or update
  async delete(id)                      { throw new Error('Not implemented'); }
  async count(filters)                  { throw new Error('Not implemented'); }
  async existsByEmail(email)            { throw new Error('Not implemented'); }
}

module.exports = IUserRepository;
```

---

### Step 2: PostgreSQL Implementation

```javascript
// repositories/PostgresUserRepository.js
const IUserRepository = require('./IUserRepository');

class PostgresUserRepository extends IUserRepository {
  #pool;

  constructor(pool) {
    super();
    this.#pool = pool;
  }

  async findById(id) {
    const result = await this.#pool.query(
      'SELECT * FROM users WHERE id = $1 AND deleted_at IS NULL',
      [id]
    );
    return result.rows[0] || null;
  }

  async findByEmail(email) {
    const result = await this.#pool.query(
      'SELECT * FROM users WHERE email = $1 AND deleted_at IS NULL',
      [email.toLowerCase()]
    );
    return result.rows[0] || null;
  }

  async findAll({ role, isActive, search } = {}, { page = 1, limit = 20, sortBy = 'created_at', sortDir = 'DESC' } = {}) {
    const conditions = ['deleted_at IS NULL'];
    const params = [];

    if (role) {
      params.push(role);
      conditions.push(`role = $${params.length}`);
    }
    if (isActive !== undefined) {
      params.push(isActive);
      conditions.push(`is_active = $${params.length}`);
    }
    if (search) {
      params.push(`%${search}%`);
      conditions.push(`(name ILIKE $${params.length} OR email ILIKE $${params.length})`);
    }

    const offset = (page - 1) * limit;
    params.push(limit, offset);

    const sql = `
      SELECT * FROM users
      WHERE ${conditions.join(' AND ')}
      ORDER BY ${sortBy} ${sortDir}
      LIMIT $${params.length - 1}
      OFFSET $${params.length}
    `;

    const result = await this.#pool.query(sql, params);
    return result.rows;
  }

  async findByRole(role) {
    return this.findAll({ role });
  }

  async save(userData) {
    if (userData.id) {
      // Update
      const result = await this.#pool.query(
        `UPDATE users SET name=$1, email=$2, role=$3, is_active=$4, updated_at=NOW()
         WHERE id=$5 RETURNING *`,
        [userData.name, userData.email, userData.role, userData.isActive, userData.id]
      );
      return result.rows[0];
    } else {
      // Create
      const result = await this.#pool.query(
        `INSERT INTO users (name, email, password_hash, role, is_active)
         VALUES ($1, $2, $3, $4, $5) RETURNING *`,
        [userData.name, userData.email, userData.passwordHash, userData.role || 'user', userData.isActive ?? true]
      );
      return result.rows[0];
    }
  }

  async delete(id) {
    // Soft delete
    await this.#pool.query(
      'UPDATE users SET deleted_at = NOW() WHERE id = $1',
      [id]
    );
    return true;
  }

  async count(filters = {}) {
    const result = await this.#pool.query(
      'SELECT COUNT(*) FROM users WHERE deleted_at IS NULL'
    );
    return parseInt(result.rows[0].count);
  }

  async existsByEmail(email) {
    const result = await this.#pool.query(
      'SELECT 1 FROM users WHERE email = $1 AND deleted_at IS NULL LIMIT 1',
      [email.toLowerCase()]
    );
    return result.rows.length > 0;
  }
}

module.exports = PostgresUserRepository;
```

---

### Step 3: In-Memory Implementation (for Testing!)

This is the HUGE advantage of Repository:

```javascript
// repositories/InMemoryUserRepository.js
const IUserRepository = require('./IUserRepository');

class InMemoryUserRepository extends IUserRepository {
  #users = new Map();
  #nextId = 1;

  async findById(id) {
    return this.#users.get(String(id)) || null;
  }

  async findByEmail(email) {
    for (const user of this.#users.values()) {
      if (user.email === email.toLowerCase()) return user;
    }
    return null;
  }

  async findAll({ role, isActive, search } = {}, { page = 1, limit = 20 } = {}) {
    let users = [...this.#users.values()];
    if (role)       users = users.filter(u => u.role === role);
    if (isActive !== undefined) users = users.filter(u => u.isActive === isActive);
    if (search)     users = users.filter(u =>
      u.name.includes(search) || u.email.includes(search)
    );
    return users.slice((page - 1) * limit, page * limit);
  }

  async findByRole(role) { return this.findAll({ role }); }

  async save(userData) {
    if (userData.id) {
      const existing = this.#users.get(String(userData.id));
      const updated = { ...existing, ...userData, updatedAt: new Date() };
      this.#users.set(String(userData.id), updated);
      return updated;
    } else {
      const id = String(this.#nextId++);
      const user = { ...userData, id, createdAt: new Date(), updatedAt: new Date() };
      this.#users.set(id, user);
      return user;
    }
  }

  async delete(id) { this.#users.delete(String(id)); return true; }
  async count() { return this.#users.size; }
  async existsByEmail(email) {
    for (const user of this.#users.values()) {
      if (user.email === email.toLowerCase()) return true;
    }
    return false;
  }

  // Test helper — inspect internal state
  _getAll() { return [...this.#users.values()]; }
  _clear()  { this.#users.clear(); this.#nextId = 1; }
  _seed(users) { users.forEach(u => this.#users.set(String(u.id), u)); }
}

module.exports = InMemoryUserRepository;
```

---

### Step 4: Business Logic (UserService) — Uses Repository, Never Touches DB

```javascript
// services/UserService.js
const bcrypt = require('bcrypt');

class UserService {
  #userRepo;

  constructor(userRepository) {
    this.#userRepo = userRepository; // injected — could be Postgres, Mongo, InMemory
  }

  async register({ name, email, password }) {
    // 1. Validate
    if (!name || name.length < 2) throw new Error('Name too short');
    if (!email || !email.includes('@')) throw new Error('Invalid email');
    if (!password || password.length < 8) throw new Error('Password too short');

    // 2. Check uniqueness
    const exists = await this.#userRepo.existsByEmail(email);
    if (exists) throw new Error('Email already registered');

    // 3. Hash password
    const passwordHash = await bcrypt.hash(password, 12);

    // 4. Save
    const user = await this.#userRepo.save({ name, email, passwordHash, role: 'user' });

    // Return safe user (no passwordHash)
    const { passwordHash: _, ...safeUser } = user;
    return safeUser;
  }

  async getUser(id) {
    const user = await this.#userRepo.findById(id);
    if (!user) throw new Error(`User ${id} not found`);
    const { passwordHash: _, ...safeUser } = user;
    return safeUser;
  }

  async listAdmins() {
    return this.#userRepo.findByRole('admin');
  }

  async searchUsers(query, page = 1) {
    return this.#userRepo.findAll({ search: query }, { page, limit: 20 });
  }

  async deactivateUser(id, requesterId) {
    if (id === requesterId) throw new Error('Cannot deactivate yourself');
    const user = await this.#userRepo.findById(id);
    if (!user) throw new Error('User not found');
    return this.#userRepo.save({ ...user, isActive: false });
  }
}

module.exports = UserService;
```

---

### Step 5: Tests — No Database Needed!

```javascript
// tests/UserService.test.js
const { describe, it, beforeEach, expect } = require('@jest/globals');
const UserService = require('../services/UserService');
const InMemoryUserRepository = require('../repositories/InMemoryUserRepository');

describe('UserService', () => {
  let userService;
  let userRepo;

  beforeEach(() => {
    userRepo = new InMemoryUserRepository(); // No DB! Fast tests!
    userService = new UserService(userRepo);
  });

  it('registers a new user', async () => {
    const user = await userService.register({
      name: 'Alice',
      email: 'alice@example.com',
      password: 'Password123',
    });

    expect(user.name).toBe('Alice');
    expect(user.email).toBe('alice@example.com');
    expect(user.passwordHash).toBeUndefined(); // not exposed
    expect(await userRepo.count()).toBe(1);
  });

  it('rejects duplicate emails', async () => {
    await userService.register({ name: 'Alice', email: 'alice@example.com', password: 'Pass1234' });
    await expect(
      userService.register({ name: 'Alice2', email: 'alice@example.com', password: 'Pass1234' })
    ).rejects.toThrow('Email already registered');
  });

  it('throws when user not found', async () => {
    await expect(userService.getUser(999)).rejects.toThrow('User 999 not found');
  });
});
```

---

## Repository vs DAO (Data Access Object)

| | Repository | DAO |
|--|------------|-----|
| **Focus** | Domain objects (business concepts) | Database records (tables/rows) |
| **Interface** | `findActiveUsers()`, `findByRole()` | `select()`, `insert()`, `update()` |
| **Abstraction** | Higher — hides ALL data logic | Lower — maps to DB operations |
| **Testing** | Easy to swap with in-memory | Harder to mock |

---

## Interview Q&A

**Q: What is the Repository pattern?**
> An abstraction layer between business logic and data storage. Business code calls repository methods (`findById`, `save`, `delete`) without knowing or caring whether data is in PostgreSQL, MongoDB, Redis, or memory.

**Q: Repository vs DAO?**
> DAO is a thin wrapper around database operations. Repository is a higher-level abstraction around domain objects. Repository hides all persistence concerns; DAO just simplifies DB calls.

**Q: Why use Repository?**
> Testability (swap in-memory for tests), database independence (swap Postgres→Mongo without touching business logic), single responsibility (data access in one place), cleaner business logic.

---

---

# 13 — Dependency Injection

> **Category:** Application Design  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

When you make coffee at home, you go to the store to buy coffee beans yourself (you CREATE your dependency). But in a restaurant, the barista receives coffee beans delivered by a supplier — they don't go shopping (they RECEIVE their dependency).

**Dependency Injection (DI):** Instead of a class creating its own dependencies with `new`, they are **given** (injected) to it from outside.

**In one sentence:** Give a class its dependencies instead of letting it create them — this makes it testable, flexible, and loosely coupled.

---

## Visual Diagram

```
WITHOUT DI (tight coupling):           WITH DI (loose coupling):
────────────────────────               ─────────────────────────
OrderService                           OrderService
  constructor() {                        constructor(userRepo, mailer) {
    this.userRepo = new               ←    this.userRepo = userRepo;
      PostgresUserRepo();  ← hard          this.mailer   = mailer;
    this.mailer = new          coded      }
      SmtpMailer();        ← hard
  }                          coded    // Whoever creates OrderService
                                      // decides what to inject:
// Can never swap implementations!   const svc = new OrderService(
// Tests need a real DB!                new PostgresUserRepo(pool),
                                        new SmtpMailer(config)
                                      );
                                      // OR in tests:
                                      const svc = new OrderService(
                                        new InMemoryUserRepo(),
                                        new FakeMailer()
                                      );
```

---

## Node.js Implementation

### Version 1: Constructor Injection (Best Practice)

```javascript
// ── Dependencies (all interchangeable via interface) ──

class SmtpMailer {
  async sendEmail(to, subject, body) {
    console.log(`[SMTP] Sending "${subject}" to ${to}`);
    // nodemailer.sendMail(...)
    return { messageId: `smtp_${Date.now()}` };
  }
}

class FakeMailer { // for tests
  constructor() { this.sentEmails = []; }
  async sendEmail(to, subject, body) {
    this.sentEmails.push({ to, subject, body });
    console.log(`[FAKE] Recorded email to ${to}`);
    return { messageId: `fake_${Date.now()}` };
  }
}

class PostgresOrderRepo {
  constructor(pool) { this.pool = pool; }
  async save(order) {
    const res = await this.pool.query('INSERT INTO orders ... RETURNING *', [...]);
    return res.rows[0];
  }
  async findById(id) { /* DB query */ }
}

class InMemoryOrderRepo { // for tests
  #orders = new Map();
  async save(order) {
    const id = order.id || String(Date.now());
    const saved = { ...order, id };
    this.#orders.set(id, saved);
    return saved;
  }
  async findById(id) { return this.#orders.get(id) || null; }
  _getAll() { return [...this.#orders.values()]; }
}

// ── Service — receives dependencies, doesn't create them ──

class OrderService {
  #orderRepo;
  #mailer;
  #logger;

  // Constructor injection — most common and recommended
  constructor(orderRepo, mailer, logger = console) {
    this.#orderRepo = orderRepo;
    this.#mailer    = mailer;
    this.#logger    = logger;
  }

  async createOrder(userId, items, total) {
    this.#logger.info(`Creating order for user ${userId}`);

    const order = await this.#orderRepo.save({
      userId, items, total, status: 'pending', createdAt: new Date()
    });

    await this.#mailer.sendEmail(
      `user_${userId}@example.com`,
      'Order Confirmed!',
      `Your order ${order.id} for ₹${total} has been placed.`
    );

    return order;
  }

  async getOrder(id) {
    const order = await this.#orderRepo.findById(id);
    if (!order) throw new Error(`Order ${id} not found`);
    return order;
  }
}

// ── Composition Root — where you wire everything together ──
// This is the ONLY place that knows about concrete classes

// Production:
const { Pool } = require('pg');
const pool = new Pool(process.env.DATABASE_URL);
const orderService = new OrderService(
  new PostgresOrderRepo(pool),
  new SmtpMailer({ host: process.env.SMTP_HOST }),
  require('./logger'),
);

// Tests:
const testRepo   = new InMemoryOrderRepo();
const testMailer = new FakeMailer();
const testService = new OrderService(testRepo, testMailer);

// Test without any real infrastructure!
const order = await testService.createOrder(1, [{ sku: 'A', qty: 2 }], 999);
console.log(testMailer.sentEmails); // inspect what was "sent"
```

---

### Version 2: DI Container (Advanced)

```javascript
// Simple DI Container — auto-resolves dependencies

class Container {
  #registrations = new Map();
  #instances = new Map();

  // Register a factory function
  register(name, factory, { singleton = false } = {}) {
    this.#registrations.set(name, { factory, singleton });
    return this;
  }

  // Register a value directly (for config, constants)
  registerValue(name, value) {
    this.#instances.set(name, value);
    return this;
  }

  // Resolve a dependency by name
  resolve(name) {
    // Check if already instantiated (singleton cache)
    if (this.#instances.has(name)) {
      return this.#instances.get(name);
    }

    const registration = this.#registrations.get(name);
    if (!registration) throw new Error(`No registration for: "${name}"`);

    const instance = registration.factory(this); // pass container for sub-dependencies

    if (registration.singleton) {
      this.#instances.set(name, instance); // cache it
    }

    return instance;
  }
}

// ── Wire up the container ──

const container = new Container();

// Values / config
container.registerValue('dbConfig', {
  host: process.env.DB_HOST, port: 5432, database: 'myapp'
});

container.registerValue('smtpConfig', {
  host: process.env.SMTP_HOST, port: 587
});

// Infrastructure — singletons (one instance for the whole app)
container.register('db', (c) => {
  const { Pool } = require('pg');
  return new Pool(c.resolve('dbConfig'));
}, { singleton: true });

container.register('mailer', (c) => {
  return new SmtpMailer(c.resolve('smtpConfig'));
}, { singleton: true });

container.register('logger', () => require('./logger'), { singleton: true });

// Repositories
container.register('userRepository', (c) => {
  return new PostgresUserRepo(c.resolve('db'));
}, { singleton: true });

container.register('orderRepository', (c) => {
  return new PostgresOrderRepo(c.resolve('db'));
}, { singleton: true });

// Services — auto-resolve their dependencies
container.register('userService', (c) => {
  return new UserService(
    c.resolve('userRepository'),
  );
}, { singleton: true });

container.register('orderService', (c) => {
  return new OrderService(
    c.resolve('orderRepository'),
    c.resolve('mailer'),
    c.resolve('logger'),
  );
}, { singleton: true });

// ── Usage in Express routes ──
const orderService = container.resolve('orderService');

app.post('/orders', async (req, res) => {
  const order = await orderService.createOrder(req.user.id, req.body.items, req.body.total);
  res.json(order);
});
```

---

## Three Types of DI

```javascript
// 1. Constructor Injection ✅ RECOMMENDED
class Service {
  constructor(dep) { this.dep = dep; } // injected at creation
}

// 2. Property/Setter Injection ⚠️ OK for optional deps
class Service {
  setLogger(logger) { this.logger = logger; } // injected after creation
}

// 3. Interface Injection (rare in JS) — dep injects itself into class
```

---

## Interview Q&A

**Q: What is Dependency Injection?**
> A technique where a class receives its dependencies from the outside (via constructor, setter, or container) rather than creating them itself. Promotes testability and loose coupling.

**Q: Why use DI?**
> Testability (inject mocks/fakes), flexibility (swap implementations), single responsibility (class doesn't manage its own lifecycle dependencies), easier maintenance.

**Q: Constructor vs setter injection?**
> Constructor injection: dependency required at creation, immutable, clear contract. Setter injection: dependency optional/changeable after creation. Constructor injection is preferred.

**Q: What is an IoC container?**
> Inversion of Control container — a registry that knows how to create and wire up all the objects in your app. You ask it for a service; it resolves all its dependencies automatically.

---

---

# 14 — MVC Pattern

> **Category:** Application Design  
> **Difficulty:** ⭐ Beginner  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it?

**Model-View-Controller** splits your app into three parts:
- **Model:** Data and business logic (users, orders, database)
- **View:** What the user sees (HTML, JSON response, template)
- **Controller:** Receives requests, calls Model, returns View

In backend APIs (Express.js), the "View" is usually a **JSON response** — there's no HTML template.

---

## Visual Diagram

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────────────────┐
│                    Controller                        │
│  - Receives HTTP request                            │
│  - Validates input (req.body, req.params)           │
│  - Calls the right Service/Model                    │
│  - Returns JSON response                            │
└─────────────────┬──────────────────────┬────────────┘
                  │                      │
                  ▼                      ▼
         ┌──────────────┐      ┌──────────────────┐
         │    Model     │      │      View        │
         │  (Service +  │      │  (JSON response) │
         │  Repository) │      │                  │
         │              │      │  res.json({...}) │
         │  Business    │      │  res.status(201) │
         │  logic here  │      │  .json({...})    │
         └──────────────┘      └──────────────────┘
```

---

## Full Express.js MVC Example

```javascript
// ── Model (data + business logic) ──
// models/User.js
class User {
  constructor({ id, name, email, role, createdAt }) {
    this.id        = id;
    this.name      = name;
    this.email     = email;
    this.role      = role;
    this.createdAt = createdAt;
  }

  isAdmin() { return this.role === 'admin'; }

  toJSON() {
    return { id: this.id, name: this.name, email: this.email, role: this.role };
  }
}

// services/UserService.js — business logic (part of Model layer)
class UserService {
  constructor(userRepo) { this.userRepo = userRepo; }

  async getUser(id) {
    const row = await this.userRepo.findById(id);
    if (!row) throw new Object.assign(new Error('User not found'), { statusCode: 404 });
    return new User(row);
  }

  async createUser({ name, email, password }) {
    const exists = await this.userRepo.existsByEmail(email);
    if (exists) throw Object.assign(new Error('Email taken'), { statusCode: 409 });
    const hash = await bcrypt.hash(password, 12);
    const row  = await this.userRepo.save({ name, email, passwordHash: hash });
    return new User(row);
  }

  async listUsers({ page = 1, limit = 20 } = {}) {
    const rows = await this.userRepo.findAll({}, { page, limit });
    return rows.map(r => new User(r));
  }
}

// ── Controller — glue between HTTP and Service ──
// controllers/UserController.js
class UserController {
  #userService;

  constructor(userService) { this.#userService = userService; }

  // GET /users/:id
  getUser = async (req, res, next) => {
    try {
      const user = await this.#userService.getUser(req.params.id);
      res.json({ success: true, data: user });
    } catch (error) {
      next(error); // pass to error middleware
    }
  };

  // POST /users
  createUser = async (req, res, next) => {
    try {
      const { name, email, password } = req.body;

      // Input validation (controller responsibility)
      if (!name || !email || !password) {
        return res.status(400).json({
          success: false,
          error: 'name, email, and password are required',
        });
      }

      const user = await this.#userService.createUser({ name, email, password });
      res.status(201).json({ success: true, data: user });
    } catch (error) {
      next(error);
    }
  };

  // GET /users
  listUsers = async (req, res, next) => {
    try {
      const page  = parseInt(req.query.page)  || 1;
      const limit = parseInt(req.query.limit) || 20;
      const users = await this.#userService.listUsers({ page, limit });
      res.json({ success: true, data: users, page, limit });
    } catch (error) {
      next(error);
    }
  };

  // DELETE /users/:id
  deleteUser = async (req, res, next) => {
    try {
      await this.#userService.deleteUser(req.params.id);
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  };
}

// ── View — JSON response (built into Express, no template needed for APIs) ──

// ── Router — maps URLs to Controller methods ──
// routes/userRoutes.js
const express = require('express');

function createUserRouter(userController) {
  const router = express.Router();

  router.get('/',    userController.listUsers);
  router.post('/',   userController.createUser);
  router.get('/:id', userController.getUser);
  router.delete('/:id', userController.deleteUser);

  return router;
}

// ── app.js — wire everything together ──
const { Pool } = require('pg');
const express = require('express');

const app = express();
app.use(express.json());

// Composition root
const pool       = new Pool({ connectionString: process.env.DATABASE_URL });
const userRepo   = new PostgresUserRepository(pool);
const userService = new UserService(userRepo);
const userCtrl   = new UserController(userService);

app.use('/api/users', createUserRouter(userCtrl));

// Error handling middleware (catches errors from next(error))
app.use((err, req, res, next) => {
  const status = err.statusCode || 500;
  res.status(status).json({
    success: false,
    error:   err.message || 'Internal server error',
  });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## Folder Structure for MVC in Node.js

```
src/
├── controllers/          ← Handle HTTP: parse input, call service, return JSON
│   ├── UserController.js
│   └── OrderController.js
├── services/             ← Business logic: validation, orchestration
│   ├── UserService.js
│   └── OrderService.js
├── repositories/         ← Data access: queries, CRUD
│   ├── UserRepository.js
│   └── OrderRepository.js
├── models/               ← Domain objects: User, Order classes
│   ├── User.js
│   └── Order.js
├── routes/               ← URL → Controller mappings
│   ├── userRoutes.js
│   └── orderRoutes.js
├── middleware/           ← Cross-cutting: auth, logging, error handling
│   ├── auth.js
│   └── errorHandler.js
└── app.js                ← Composition root: wire everything up
```

---

## Interview Q&A

**Q: What is MVC?**
> Architectural pattern separating: Model (data + business logic), View (presentation), Controller (handles requests, coordinates M and V). In REST APIs, View = JSON response.

**Q: Where does validation go?**
> Basic input validation (is field present? correct format?) → Controller. Business rule validation (is email unique? is user active?) → Service/Model.

**Q: Why MVC?**
> Separation of concerns — each layer has one job. Testable — test Services without HTTP. Maintainable — change how data is stored without touching controllers.

---

## Summary

```
Repository: Librarian for data — hides DB details from business logic
  → Interface + Postgres impl + InMemory impl for tests

Dependency Injection: Give classes their dependencies, don't let them create them
  → Constructor injection preferred; DI container for large apps

MVC: Controller handles HTTP, Service has business logic, Repository has data access
  → In Node.js REST APIs: View = JSON response
  → Folder: controllers / services / repositories / models / routes
```
