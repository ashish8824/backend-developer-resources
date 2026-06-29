# 15–20 — Architecture Patterns

---

# 15 — Microservices Architecture

> **Category:** Architecture  
> **Difficulty:** ⭐⭐⭐ Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

A monolith is like a Swiss Army knife — one tool that does everything. Useful, but if the corkscrew breaks, the whole knife is in repair.

**Microservices** is like having separate, specialized tools — a dedicated knife, a dedicated corkscrew, dedicated scissors. Each can be replaced, upgraded, or scaled independently.

**In one sentence:** Microservices structures an application as a collection of small, independently deployable services, each responsible for exactly one business capability.

---

## Monolith vs Microservices

```
MONOLITH                          MICROSERVICES
────────                          ─────────────

┌─────────────────────┐           ┌───────────┐  ┌──────────┐
│      MyApp          │           │  User     │  │  Order   │
│  ─────────────────  │           │  Service  │  │  Service │
│  UserController     │           │  :3001    │  │  :3002   │
│  OrderController    │           └───────────┘  └──────────┘
│  PaymentController  │
│  UserService        │           ┌───────────┐  ┌──────────┐
│  OrderService       │           │ Payment   │  │Notifica- │
│  PaymentService     │           │ Service   │  │tion Svc  │
│  UserRepo           │           │  :3003    │  │  :3004   │
│  OrderRepo          │           └───────────┘  └──────────┘
│  PostgreSQL         │
│  Redis              │           Each has its OWN:
└─────────────────────┘           - Codebase
                                  - Database
One deploy for everything         - Deployment pipeline
All or nothing scaling            - Tech stack (!)
```

---

## Core Principles

```
1. Single Responsibility   Each service does ONE business function
2. Independent Deploy      Deploy UserService without touching OrderService
3. Own Data Store          Each service has its own database
4. Communicate via API     HTTP/REST or message queues
5. Failure Isolation       One service crashing doesn't bring down others
```

---

## Node.js Implementation

### User Service

```javascript
// user-service/src/app.js
const express = require('express');
const app = express();
app.use(express.json());

// User service ONLY knows about users
app.get('/users/:id', async (req, res) => {
  const user = await userRepo.findById(req.params.id);
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

app.post('/users', async (req, res) => {
  const user = await userService.createUser(req.body);
  res.status(201).json(user);
});

app.listen(3001, () => console.log('User Service on port 3001'));
```

### Order Service (calls User Service via HTTP)

```javascript
// order-service/src/app.js
const express = require('express');
const axios   = require('axios');

const USER_SERVICE_URL = process.env.USER_SERVICE_URL || 'http://user-service:3001';

app.post('/orders', async (req, res) => {
  try {
    // Call User Service to validate user exists
    const userResponse = await axios.get(`${USER_SERVICE_URL}/users/${req.body.userId}`);
    const user = userResponse.data;

    // Create order
    const order = await orderService.createOrder({
      userId: user.id,
      items:  req.body.items,
      total:  req.body.total,
    });

    res.status(201).json(order);
  } catch (error) {
    if (error.response?.status === 404) {
      return res.status(400).json({ error: 'User not found' });
    }
    res.status(500).json({ error: 'Failed to create order' });
  }
});

app.listen(3002, () => console.log('Order Service on port 3002'));
```

### Docker Compose (run all services together)

```yaml
# docker-compose.yml
version: '3.8'
services:
  user-service:
    build: ./user-service
    ports: ['3001:3001']
    environment:
      - DATABASE_URL=postgresql://postgres:pass@user-db:5432/users
    depends_on: [user-db]

  order-service:
    build: ./order-service
    ports: ['3002:3002']
    environment:
      - DATABASE_URL=postgresql://postgres:pass@order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:3001
    depends_on: [order-db, user-service]

  user-db:
    image: postgres:15
    environment:
      POSTGRES_DB: users
      POSTGRES_PASSWORD: pass

  order-db:
    image: postgres:15
    environment:
      POSTGRES_DB: orders
      POSTGRES_PASSWORD: pass
```

---

## When to Use Microservices

| Choose Microservices | Choose Monolith |
|---------------------|-----------------|
| Large team (multiple squads) | Small team (< 10 devs) |
| Different scaling needs per service | Uniform scaling |
| Different tech stacks needed | Single tech stack fine |
| Independent deployment required | Coordinated deploys OK |
| Product is mature, boundaries clear | Early-stage, still figuring out domain |

> **Start with a monolith. Extract to microservices when pain is clear.**

---

## Interview Q&A

**Q: What are the benefits of microservices?**
> Independent deployment, independent scaling, technology flexibility, fault isolation, team autonomy.

**Q: What are the drawbacks?**
> Distributed system complexity, network latency, data consistency across services, need for service discovery, more DevOps overhead.

**Q: When would you NOT use microservices?**
> Early-stage startups, small teams, unclear domain boundaries. Start with monolith, migrate later.

---

---

# 16 — API Gateway Pattern

> **Category:** Architecture  
> **Difficulty:** ⭐⭐⭐ Intermediate

---

## What is it?

Without an API Gateway, every client must know the address of every microservice. With an API Gateway, there's **one front door** for all clients. It routes requests, handles auth, rate limits, and can aggregate responses.

```
Without Gateway:                 With API Gateway:
────────────────                 ────────────────

Mobile App ──────> User Svc     Mobile App ─────>
Web App    ──────> Order Svc                      API Gateway
IoT Device ──────> Payment Svc  Web App    ──────>    │
                 > Notif Svc    IoT Device ──────>    ├──> User Svc
                                                      ├──> Order Svc
Each client knows all services!                       ├──> Payment Svc
                                                      └──> Notif Svc

                                 One entry point. Clients know ONE address.
```

---

## What Gateway Does

```
1. Routing          /api/users  → User Service
                    /api/orders → Order Service

2. Authentication   Verify JWT token ONCE (services trust gateway)

3. Rate Limiting    100 requests/min per IP

4. SSL Termination  HTTPS on gateway; HTTP internally

5. Load Balancing   Round-robin across multiple instances

6. Response Agg.   Fetch from User + Order + Payment, return one response

7. Caching          Cache GET /products for 5 minutes

8. Logging          Log ALL requests in one place
```

---

## Node.js Implementation (Express as Gateway)

```javascript
// api-gateway/src/app.js
const express    = require('express');
const httpProxy  = require('http-proxy-middleware');
const rateLimit  = require('express-rate-limit');
const jwt        = require('jsonwebtoken');

const app = express();
app.use(express.json());

// ── Service Registry (in production: use Consul, etcd, or env vars) ──
const services = {
  user:    process.env.USER_SERVICE_URL    || 'http://user-service:3001',
  order:   process.env.ORDER_SERVICE_URL   || 'http://order-service:3002',
  payment: process.env.PAYMENT_SERVICE_URL || 'http://payment-service:3003',
  product: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3004',
};

// ── Rate Limiting ──
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100,
  message: { error: 'Too many requests, please try again later' },
  standardHeaders: true,
});
app.use('/api/', limiter);

// ── Authentication Middleware ──
function authenticate(req, res, next) {
  // Public routes — skip auth
  const publicRoutes = [
    { method: 'POST', path: '/api/auth/login' },
    { method: 'POST', path: '/api/users' },           // register
    { method: 'GET',  path: '/api/products' },        // public catalog
  ];

  const isPublic = publicRoutes.some(
    r => req.method === r.method && req.path.startsWith(r.path)
  );
  if (isPublic) return next();

  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Authentication required' });

  try {
    req.user = jwt.verify(token, process.env.JWT_SECRET);
    // Add user context to headers for downstream services
    req.headers['X-User-Id']   = req.user.sub;
    req.headers['X-User-Role'] = req.user.role;
    next();
  } catch {
    res.status(401).json({ error: 'Invalid or expired token' });
  }
}
app.use(authenticate);

// ── Request Logging ──
app.use((req, res, next) => {
  const start = Date.now();
  res.on('finish', () => {
    console.log(`${req.method} ${req.path} ${res.statusCode} ${Date.now() - start}ms | user:${req.user?.sub || 'anon'}`);
  });
  next();
});

// ── Proxy Routes ──
app.use('/api/users',    httpProxy.createProxyMiddleware({ target: services.user,    changeOrigin: true, pathRewrite: { '^/api/users': '/users' } }));
app.use('/api/orders',   httpProxy.createProxyMiddleware({ target: services.order,   changeOrigin: true, pathRewrite: { '^/api/orders': '/orders' } }));
app.use('/api/payments', httpProxy.createProxyMiddleware({ target: services.payment, changeOrigin: true, pathRewrite: { '^/api/payments': '/payments' } }));
app.use('/api/products', httpProxy.createProxyMiddleware({ target: services.product, changeOrigin: true, pathRewrite: { '^/api/products': '/products' } }));

// ── Response Aggregation — fetch from multiple services in one call ──
app.get('/api/dashboard/:userId', async (req, res) => {
  const userId = req.params.userId;

  // Authorization check
  if (req.user.sub !== userId && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Forbidden' });
  }

  try {
    // Parallel calls to multiple services
    const [userRes, ordersRes, statsRes] = await Promise.all([
      axios.get(`${services.user}/users/${userId}`,        { headers: { 'X-Internal': 'gateway' } }),
      axios.get(`${services.order}/orders?userId=${userId}&limit=5`, { headers: { 'X-Internal': 'gateway' } }),
      axios.get(`${services.order}/orders/stats/${userId}`, { headers: { 'X-Internal': 'gateway' } }),
    ]);

    // Aggregate into one response
    res.json({
      user:         userRes.data,
      recentOrders: ordersRes.data,
      stats:        statsRes.data,
    });
  } catch (error) {
    res.status(500).json({ error: 'Failed to load dashboard' });
  }
});

// ── Health Check ──
app.get('/health', async (req, res) => {
  const checks = await Promise.allSettled(
    Object.entries(services).map(async ([name, url]) => {
      const r = await axios.get(`${url}/health`, { timeout: 2000 });
      return { name, status: 'up' };
    })
  );

  const health = checks.map((c, i) => ({
    service: Object.keys(services)[i],
    status:  c.status === 'fulfilled' ? 'up' : 'down',
  }));

  const allHealthy = health.every(h => h.status === 'up');
  res.status(allHealthy ? 200 : 503).json({ status: allHealthy ? 'ok' : 'degraded', services: health });
});

app.listen(3000, () => console.log('API Gateway on port 3000'));
```

---

## Interview Q&A

**Q: What does an API Gateway do?**
> Single entry point for all clients. Handles routing, authentication, rate limiting, SSL termination, response aggregation, load balancing, and logging in one place.

**Q: Gateway vs Load Balancer?**
> Load balancer distributes traffic across identical service instances. Gateway routes to DIFFERENT services based on path/method and adds cross-cutting concerns.

**Q: Popular API Gateway tools?**
> Kong, AWS API Gateway, nginx, Traefik, Spring Cloud Gateway, APISIX.

---

---

# 17 — Circuit Breaker Pattern

> **Category:** Architecture (Resilience)  
> **Difficulty:** ⭐⭐⭐ Intermediate

---

## What is it? (The Simple Idea)

Your house has electrical circuit breakers. If there's a power surge, the breaker **trips** (opens) to protect your appliances. Once the danger passes, you reset it.

The **Circuit Breaker pattern** does the same for service calls: if a downstream service keeps failing, stop calling it. This prevents cascade failures and gives the failing service time to recover.

---

## Three States

```
┌─────────────────────────────────────────────────────────────┐
│                   Circuit Breaker States                     │
│                                                             │
│   CLOSED ──────────────────────────────────────────────>   │
│   (Normal)  failures < threshold    requests pass through   │
│       │                                                     │
│       │ failures >= threshold                               │
│       ▼                                                     │
│   OPEN ────────────────────────────────────────────────>   │
│   (Failing) FAST FAIL! Don't even try the service          │
│       │     Return cached/fallback response immediately     │
│       │                                                     │
│       │ after cooldown period (e.g., 30 seconds)            │
│       ▼                                                     │
│   HALF-OPEN ──────────────────────────────────────────>   │
│   (Testing) Let ONE request through as a probe              │
│       │                                                     │
│       ├── success? ──> back to CLOSED ✅                    │
│       └── failure? ──> back to OPEN ❌ (reset timer)        │
└─────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

```javascript
// CircuitBreaker.js

class CircuitBreaker {
  #state = 'CLOSED';   // CLOSED | OPEN | HALF_OPEN
  #failureCount = 0;
  #successCount = 0;
  #lastFailureTime = null;
  #name;
  #options;

  constructor(name, options = {}) {
    this.#name    = name;
    this.#options = {
      failureThreshold:  options.failureThreshold  || 5,    // trips after 5 failures
      successThreshold:  options.successThreshold  || 2,    // closes after 2 successes
      cooldownPeriodMs:  options.cooldownPeriodMs  || 30000, // 30 second cooldown
      requestTimeoutMs:  options.requestTimeoutMs  || 5000, // 5 second timeout
      fallback:          options.fallback          || null, // fallback function
    };
  }

  get state() { return this.#state; }

  async call(fn) {
    // ── OPEN state: fast fail ──
    if (this.#state === 'OPEN') {
      const timeSinceLastFailure = Date.now() - this.#lastFailureTime;

      if (timeSinceLastFailure < this.#options.cooldownPeriodMs) {
        console.log(`🔴 [${this.#name}] Circuit OPEN — fast failing. Cooldown: ${Math.round((this.#options.cooldownPeriodMs - timeSinceLastFailure) / 1000)}s left`);

        if (this.#options.fallback) return this.#options.fallback();
        throw new Error(`Circuit breaker OPEN for ${this.#name}`);
      }

      // Cooldown expired — move to HALF_OPEN for testing
      this.#transitionTo('HALF_OPEN');
    }

    // ── HALF_OPEN: let one request through ──
    if (this.#state === 'HALF_OPEN') {
      console.log(`🟡 [${this.#name}] Circuit HALF_OPEN — testing with one request`);
    }

    // ── Execute the function with timeout ──
    try {
      const result = await this.#executeWithTimeout(fn);
      this.#onSuccess();
      return result;
    } catch (error) {
      this.#onFailure(error);
      throw error;
    }
  }

  async #executeWithTimeout(fn) {
    return new Promise((resolve, reject) => {
      const timer = setTimeout(() => {
        reject(new Error(`Request timeout after ${this.#options.requestTimeoutMs}ms`));
      }, this.#options.requestTimeoutMs);

      fn().then(
        (result) => { clearTimeout(timer); resolve(result); },
        (error)  => { clearTimeout(timer); reject(error); }
      );
    });
  }

  #onSuccess() {
    this.#failureCount = 0;

    if (this.#state === 'HALF_OPEN') {
      this.#successCount++;
      if (this.#successCount >= this.#options.successThreshold) {
        this.#transitionTo('CLOSED');
      }
    }
  }

  #onFailure(error) {
    this.#failureCount++;
    this.#successCount = 0;
    this.#lastFailureTime = Date.now();

    console.warn(`⚠️  [${this.#name}] Failure #${this.#failureCount}: ${error.message}`);

    if (this.#state === 'HALF_OPEN') {
      this.#transitionTo('OPEN'); // back to OPEN on any failure in HALF_OPEN
    } else if (this.#failureCount >= this.#options.failureThreshold) {
      this.#transitionTo('OPEN');
    }
  }

  #transitionTo(newState) {
    console.log(`🔄 [${this.#name}] Circuit: ${this.#state} → ${newState}`);
    this.#state = newState;
    if (newState === 'HALF_OPEN') this.#successCount = 0;
  }

  getStats() {
    return {
      name: this.#name,
      state: this.#state,
      failureCount: this.#failureCount,
      lastFailureTime: this.#lastFailureTime,
    };
  }
}

// ── Usage ──

const paymentCircuit = new CircuitBreaker('payment-service', {
  failureThreshold: 3,          // trip after 3 failures
  successThreshold: 2,          // close after 2 successes
  cooldownPeriodMs: 15000,      // try again after 15 seconds
  fallback: async () => ({
    success: false,
    message: 'Payment service unavailable. Please try again in a few minutes.',
    shouldRetry: true,
  }),
});

// ── In your service ──
async function chargePayment(userId, amount) {
  return paymentCircuit.call(async () => {
    // This is the actual service call
    const response = await axios.post(`${PAYMENT_SERVICE_URL}/charge`, {
      userId, amount
    }, { timeout: 5000 });
    return response.data;
  });
}

// Simulate failures
for (let i = 0; i < 5; i++) {
  try {
    await chargePayment('user_1', 999);
  } catch (e) {
    console.log(`Attempt ${i+1}: ${e.message}`);
  }
}
// After 3 failures: 🔄 payment-service: CLOSED → OPEN
// Attempts 4 & 5: 🔴 Circuit OPEN — fast failing (no actual call made)
```

---

## Interview Q&A

**Q: What is the Circuit Breaker pattern?**
> Prevents cascade failures in distributed systems. Monitors failures and when threshold is reached, "opens" the circuit — fast-failing all requests instead of waiting for timeouts.

**Q: What are the three states?**
> CLOSED (normal, requests pass through), OPEN (failing, fast fail all requests), HALF-OPEN (testing, let one request through to see if service recovered).

**Q: Libraries for Circuit Breaker in Node.js?**
> `opossum` (most popular), `cockatiel`, `brakes`. In Java: Resilience4j, Hystrix.

---

---

# 18 — Saga Pattern

> **Category:** Architecture (Distributed Transactions)  
> **Difficulty:** ⭐⭐⭐⭐ Advanced

---

## What is it? (The Simple Idea)

In a monolith, you can wrap multiple operations in a database transaction: if one fails, everything rolls back. In microservices, each service has its own database — there's no single transaction.

The **Saga pattern** manages distributed transactions by breaking them into a sequence of local transactions. If one fails, compensating transactions run to undo the work already done.

**Real-life analogy:** Booking a holiday package — flight, hotel, car rental. If car rental fails, you automatically cancel the hotel booking and the flight. Each step has a "reverse step."

---

## Two Types of Saga

```
CHOREOGRAPHY (Event-Based)         ORCHESTRATION (Central Coordinator)
──────────────────────────         ────────────────────────────────────

OrderSvc ──publishes──>             Saga Orchestrator
  order.placed                           │
     │                              calls each service
     ▼                                   │
PaymentSvc hears it ──publishes──>       ├──> PaymentSvc.charge()
  payment.charged                        │       success/fail?
     │                                   │
     ▼                              if fail:
InventorySvc hears it                    └──> PaymentSvc.refund()
  inventory.reserved                          (compensating tx)
     │
     ▼
ShippingSvc hears it

No central coordinator.            Central orchestrator tells each
Services react to events.          service what to do.
Pros: loose coupling               Pros: easy to track, debug
Cons: hard to track, debug         Cons: orchestrator can be bottleneck
```

---

## Node.js Implementation (Orchestration Style)

```javascript
// saga/OrderSaga.js

class OrderSaga {
  constructor(services) {
    this.orderService     = services.order;
    this.paymentService   = services.payment;
    this.inventoryService = services.inventory;
    this.shippingService  = services.shipping;
    this.notifyService    = services.notification;
  }

  async execute(orderData) {
    const context = {
      orderId:       null,
      transactionId: null,
      reservationId: null,
      shippingId:    null,
      compensations: [], // stack of rollback operations
    };

    try {
      // Step 1: Create Order
      console.log('Step 1: Creating order...');
      const order = await this.orderService.create(orderData);
      context.orderId = order.id;
      context.compensations.push(async () => {
        console.log('↩️  Compensating: Cancelling order');
        await this.orderService.cancel(context.orderId);
      });

      // Step 2: Reserve Inventory
      console.log('Step 2: Reserving inventory...');
      const reservation = await this.inventoryService.reserve(
        orderData.items,
        order.id
      );
      context.reservationId = reservation.id;
      context.compensations.push(async () => {
        console.log('↩️  Compensating: Releasing inventory');
        await this.inventoryService.release(context.reservationId);
      });

      // Step 3: Charge Payment
      console.log('Step 3: Charging payment...');
      const payment = await this.paymentService.charge(
        orderData.userId,
        orderData.total,
        order.id
      );
      context.transactionId = payment.transactionId;
      context.compensations.push(async () => {
        console.log('↩️  Compensating: Refunding payment');
        await this.paymentService.refund(context.transactionId);
      });

      // Step 4: Create Shipping Label
      console.log('Step 4: Creating shipping label...');
      const shipping = await this.shippingService.createLabel(
        order.id,
        orderData.deliveryAddress
      );
      context.shippingId = shipping.id;
      context.compensations.push(async () => {
        console.log('↩️  Compensating: Cancelling shipment');
        await this.shippingService.cancel(context.shippingId);
      });

      // Step 5: Send Confirmation (no compensation needed — email already sent)
      console.log('Step 5: Sending confirmation...');
      await this.notifyService.sendOrderConfirmation(orderData.userId, order.id, shipping.trackingId);

      // Update order to completed
      await this.orderService.updateStatus(order.id, 'confirmed');

      console.log(`✅ Saga completed successfully! Order: ${order.id}`);
      return { success: true, orderId: order.id, trackingId: shipping.trackingId };

    } catch (error) {
      console.error(`❌ Saga failed at step: ${error.message}. Running compensations...`);

      // Run compensations in REVERSE order
      const compensationsToRun = [...context.compensations].reverse();
      for (const compensate of compensationsToRun) {
        try {
          await compensate();
        } catch (compensationError) {
          // Log and continue — compensations should be idempotent
          console.error('Compensation failed:', compensationError.message);
        }
      }

      throw new Error(`Order failed: ${error.message}`);
    }
  }
}

// ── Usage ──

const saga = new OrderSaga({ order, payment, inventory, shipping, notification });

try {
  const result = await saga.execute({
    userId: 'user_42',
    items: [{ sku: 'LAPTOP-001', qty: 1 }],
    total: 79999,
    deliveryAddress: { city: 'Mumbai', pin: '400001' },
  });
  console.log('Order placed:', result);
} catch (error) {
  console.error('Order failed, all steps rolled back:', error.message);
}
```

---

## Interview Q&A

**Q: What is the Saga pattern?**
> Manages distributed transactions across microservices using a sequence of local transactions, each with a compensating transaction for rollback. Replaces ACID transactions in distributed systems.

**Q: Choreography vs Orchestration?**
> Choreography: services react to each other's events — loose coupling but hard to track. Orchestration: central saga coordinator tells each service what to do — easier to debug but coordinator is a dependency.

**Q: What are compensating transactions?**
> The "undo" operations for each step. If Step 3 fails, compensation for Steps 2 and 1 run in reverse order to roll back the saga to its original state.

---

---

# 19 — CQRS Pattern

> **Category:** Architecture  
> **Difficulty:** ⭐⭐⭐⭐ Advanced

---

## What is it? (The Simple Idea)

**CQRS = Command Query Responsibility Segregation**

In most apps, you use the same model to read and write data. CQRS says: **separate them completely**.

- **Commands:** Write operations (create order, update user, delete product) — optimize for consistency
- **Queries:** Read operations (get dashboard, search products, list orders) — optimize for speed

```
WITHOUT CQRS:                     WITH CQRS:
─────────────                     ──────────

GET /orders   ─────>              GET /orders  ──────> Read Model
POST /orders  ─────>  Same        POST /orders ──────> Write Model
PUT /orders   ─────>  Model         (Query)              (Command)
                                      │                      │
                                      ▼                      ▼
                                  Read DB               Write DB
                                (denormalized,         (normalized,
                                 fast queries)          consistent)
```

---

## Why Separate Them?

```
Read patterns:                    Write patterns:
─────────────                     ───────────────
Complex JOIN queries               Simple inserts/updates
Needs aggregation (SUM, COUNT)     Needs transactions
High volume (100x more reads)      Low volume
Can be slightly stale (OK)         Must be consistent
Wants specific projections         Works on full entity
```

---

## Node.js Implementation

```javascript
// ── Write Side: Commands ──

class CreateOrderCommand {
  constructor({ userId, items, total, shippingAddress }) {
    this.userId          = userId;
    this.items           = items;
    this.total           = total;
    this.shippingAddress = shippingAddress;
    this.timestamp       = new Date();
  }
}

class CreateOrderCommandHandler {
  constructor(writeDb, eventBus) {
    this.writeDb  = writeDb;  // normalized PostgreSQL
    this.eventBus = eventBus; // Kafka/Redis to sync read model
  }

  async handle(command) {
    // Validation
    if (!command.userId)    throw new Error('userId required');
    if (!command.items?.length) throw new Error('items required');
    if (command.total <= 0) throw new Error('total must be positive');

    // Write to write DB (simple, normalized)
    const order = await this.writeDb.query(`
      INSERT INTO orders (user_id, total, status, shipping_address, created_at)
      VALUES ($1, $2, 'pending', $3, NOW())
      RETURNING *
    `, [command.userId, command.total, JSON.stringify(command.shippingAddress)]);

    // Insert items
    for (const item of command.items) {
      await this.writeDb.query(`
        INSERT INTO order_items (order_id, sku, qty, price)
        VALUES ($1, $2, $3, $4)
      `, [order.rows[0].id, item.sku, item.qty, item.price]);
    }

    // Publish event to update read model asynchronously
    await this.eventBus.publish('order.created', {
      orderId:   order.rows[0].id,
      userId:    command.userId,
      total:     command.total,
      itemCount: command.items.length,
      createdAt: order.rows[0].created_at,
    });

    return order.rows[0];
  }
}

// ── Read Side: Queries ──

class GetOrderQuery {
  constructor(orderId) { this.orderId = orderId; }
}

class GetUserOrdersQuery {
  constructor(userId, { page = 1, limit = 20, status } = {}) {
    this.userId = userId;
    this.page   = page;
    this.limit  = limit;
    this.status = status;
  }
}

class GetOrderQueryHandler {
  constructor(readDb) {
    // Read DB is DENORMALIZED — all data pre-joined for fast reads
    // Could be a separate PostgreSQL read replica, MongoDB, Elasticsearch
    this.readDb = readDb;
  }

  async handle(query) {
    if (query instanceof GetOrderQuery) {
      // Simple lookup from denormalized view
      const result = await this.readDb.query(`
        SELECT
          o.id, o.status, o.total, o.created_at,
          u.name AS user_name, u.email AS user_email,
          json_agg(json_build_object(
            'sku', oi.sku, 'qty', oi.qty, 'price', oi.price, 'name', p.name
          )) AS items
        FROM orders o
        JOIN users u ON o.user_id = u.id
        JOIN order_items oi ON oi.order_id = o.id
        JOIN products p ON p.sku = oi.sku
        WHERE o.id = $1
        GROUP BY o.id, u.name, u.email
      `, [query.orderId]);

      return result.rows[0] || null;
    }

    if (query instanceof GetUserOrdersQuery) {
      const conditions = ['o.user_id = $1'];
      const params = [query.userId];

      if (query.status) {
        params.push(query.status);
        conditions.push(`o.status = $${params.length}`);
      }

      params.push(query.limit, (query.page - 1) * query.limit);
      const result = await this.readDb.query(`
        SELECT o.id, o.status, o.total, o.created_at, COUNT(oi.id) AS item_count
        FROM orders o
        JOIN order_items oi ON oi.order_id = o.id
        WHERE ${conditions.join(' AND ')}
        GROUP BY o.id
        ORDER BY o.created_at DESC
        LIMIT $${params.length - 1} OFFSET $${params.length}
      `, params);

      return result.rows;
    }
  }
}

// ── Command Bus + Query Bus ──

class CommandBus {
  #handlers = new Map();

  register(CommandClass, handler) {
    this.#handlers.set(CommandClass.name, handler);
  }

  async dispatch(command) {
    const handler = this.#handlers.get(command.constructor.name);
    if (!handler) throw new Error(`No handler for ${command.constructor.name}`);
    return handler.handle(command);
  }
}

class QueryBus {
  #handler; // single handler in this simplified example
  constructor(handler) { this.#handler = handler; }
  async ask(query) { return this.#handler.handle(query); }
}

// ── Wire up ──

const commandBus = new CommandBus();
commandBus.register(CreateOrderCommand, new CreateOrderCommandHandler(writeDb, eventBus));

const queryBus = new QueryBus(new GetOrderQueryHandler(readDb));

// ── Express routes using CQRS ──

// Write endpoint — dispatch command
app.post('/orders', async (req, res) => {
  try {
    const command = new CreateOrderCommand(req.body);
    const result  = await commandBus.dispatch(command);
    res.status(201).json({ orderId: result.id });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

// Read endpoint — dispatch query
app.get('/orders/:id', async (req, res) => {
  const order = await queryBus.ask(new GetOrderQuery(req.params.id));
  if (!order) return res.status(404).json({ error: 'Not found' });
  res.json(order);
});

app.get('/users/:userId/orders', async (req, res) => {
  const orders = await queryBus.ask(
    new GetUserOrdersQuery(req.params.userId, req.query)
  );
  res.json(orders);
});
```

---

## Interview Q&A

**Q: What is CQRS?**
> Command Query Responsibility Segregation — separates the read model (optimized for queries) from the write model (optimized for commands/writes). Different models, potentially different databases.

**Q: When to use CQRS?**
> High read/write imbalance, complex reporting queries that don't fit the write model, audit trail requirements, event sourcing. NOT for simple CRUD apps — it adds significant complexity.

**Q: CQRS vs Event Sourcing?**
> They often go together but are separate. CQRS separates read/write models. Event Sourcing stores state as a sequence of events (not current state). You can use CQRS without Event Sourcing.

---

---

# 20 — Event-Driven Architecture

> **Category:** Architecture  
> **Difficulty:** ⭐⭐⭐ Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

In a traditional "request/response" world, Service A calls Service B and waits for a reply. Like a phone call.

In **Event-Driven Architecture**, Service A publishes an event ("user registered") and goes back to its work. Any service interested in that event subscribes and reacts — on its own time. Like sending a tweet: you post it, and whoever follows you sees it when they check.

**In one sentence:** Services communicate by producing and consuming events via a message broker — completely decoupled from each other.

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                 Event-Driven Architecture                        │
│                                                                 │
│  Producers                 Message Broker           Consumers   │
│  ─────────                 ──────────────           ─────────   │
│                                                                 │
│  UserService ──publishes──> [user.registered] ──> EmailService │
│                             Topic/Queue        ──> AnalyticsSvc│
│                                                ──> RewardsSvc  │
│                                                                 │
│  OrderService ─publishes──> [order.placed]    ──> Inventory    │
│                             Topic/Queue        ──> Payment      │
│                                                ──> Shipping     │
│                                                ──> Analytics    │
│                                                                 │
│  Producers and consumers NEVER talk directly.                   │
│  The broker buffers, routes, and persists events.               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation with Redis Pub/Sub

```javascript
// eventBus/RedisBus.js
const Redis = require('ioredis');

class RedisBus {
  constructor(redisUrl) {
    this.publisher  = new Redis(redisUrl);  // publishes
    this.subscriber = new Redis(redisUrl);  // subscribes (separate connection!)
    this.handlers   = new Map();

    this.subscriber.on('message', (channel, message) => {
      const handlers = this.handlers.get(channel) || [];
      const data = JSON.parse(message);
      handlers.forEach(h => {
        h(data).catch(err => console.error(`Handler error on ${channel}:`, err));
      });
    });
  }

  async publish(event, data) {
    const message = JSON.stringify({
      event,
      data,
      timestamp: new Date().toISOString(),
      id: `evt_${Date.now()}_${Math.random().toString(36).slice(2)}`,
    });
    await this.publisher.publish(event, message);
    console.log(`📤 Published: ${event}`);
  }

  async subscribe(event, handler) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
      await this.subscriber.subscribe(event);
      console.log(`📥 Subscribed to: ${event}`);
    }
    this.handlers.get(event).push(handler);
  }
}

module.exports = new RedisBus(process.env.REDIS_URL);
```

```javascript
// user-service/app.js
const eventBus = require('../eventBus/RedisBus');

class UserService {
  async register(userData) {
    // Create user
    const user = await userRepo.save(userData);

    // Publish event — don't care who handles it
    await eventBus.publish('user.registered', {
      userId: user.id,
      email:  user.email,
      name:   user.name,
      plan:   user.plan,
    });

    return user;
  }
}

// order-service/app.js
class OrderService {
  async createOrder(orderData) {
    const order = await orderRepo.save(orderData);

    await eventBus.publish('order.placed', {
      orderId:    order.id,
      userId:     order.userId,
      items:      order.items,
      total:      order.total,
      userEmail:  orderData.userEmail,
    });

    return order;
  }
}
```

```javascript
// email-service/app.js — Consumer
const eventBus = require('../eventBus/RedisBus');

// Subscribe to events
eventBus.subscribe('user.registered', async ({ data }) => {
  console.log(`📧 Sending welcome email to ${data.email}`);
  await mailer.sendEmail(data.email, 'Welcome!', `Hello ${data.name}!`);
});

eventBus.subscribe('order.placed', async ({ data }) => {
  console.log(`📧 Sending order confirmation to ${data.userEmail}`);
  await mailer.sendEmail(data.userEmail, `Order ${data.orderId} Confirmed`, `Your order is confirmed!`);
});

// analytics-service/app.js — Also a consumer
eventBus.subscribe('user.registered', async ({ data }) => {
  await analyticsDb.track('user_registered', { userId: data.userId, plan: data.plan });
});

eventBus.subscribe('order.placed', async ({ data }) => {
  await analyticsDb.track('order_placed', { orderId: data.orderId, total: data.total });
});

// inventory-service/app.js — Also a consumer
eventBus.subscribe('order.placed', async ({ data }) => {
  for (const item of data.items) {
    await inventoryRepo.decrement(item.sku, item.qty);
  }
});
```

---

## Production: Kafka (for High Volume)

```javascript
// eventBus/KafkaBus.js
const { Kafka } = require('kafkajs');

class KafkaBus {
  constructor() {
    const kafka = new Kafka({
      clientId: process.env.SERVICE_NAME,
      brokers: (process.env.KAFKA_BROKERS || 'localhost:9092').split(','),
    });
    this.producer = kafka.producer();
    this.consumer = kafka.consumer({ groupId: process.env.SERVICE_NAME });
  }

  async connect() {
    await this.producer.connect();
    await this.consumer.connect();
  }

  async publish(topic, data) {
    await this.producer.send({
      topic,
      messages: [{
        key: data.id || String(Date.now()),
        value: JSON.stringify({ ...data, timestamp: new Date().toISOString() }),
        headers: { source: process.env.SERVICE_NAME },
      }],
    });
  }

  async subscribe(topic, handler) {
    await this.consumer.subscribe({ topic, fromBeginning: false });
    await this.consumer.run({
      eachMessage: async ({ message }) => {
        const data = JSON.parse(message.value.toString());
        try {
          await handler(data);
        } catch (error) {
          console.error(`Failed to process message from ${topic}:`, error);
          // In production: send to dead letter queue
        }
      },
    });
  }
}
```

---

## Key Concepts

```
Message Broker:  Kafka (high volume), RabbitMQ (routing), Redis (simple)

Topic:           A named channel for events (e.g., 'order.placed')

Consumer Group:  Multiple instances of a service, each consuming partitions
                 (Kafka guarantees each message goes to ONE consumer in a group)

Dead Letter Queue (DLQ):
                 Where failed messages go after max retries

Idempotency:    Consumers must handle duplicate events gracefully
                (message might be delivered twice — always use unique IDs)

Event Schema:   Define clear structure (use JSON Schema or Protobuf)
```

---

## Event-Driven vs Observer Pattern vs Pub/Sub

| | Observer | Pub/Sub (in-process) | Event-Driven Architecture |
|--|---------|---------------------|--------------------------|
| **Scope** | Same process | Same process | Cross-service |
| **Broker** | None | None | Kafka/RabbitMQ/Redis |
| **Subject knows observers?** | Yes | No | No |
| **Persistence** | No | No | Yes (Kafka retains messages) |
| **Example** | EventEmitter | Node EventEmitter | Kafka, RabbitMQ |

---

## Interview Q&A

**Q: What is Event-Driven Architecture?**
> Services communicate asynchronously by publishing and consuming events through a message broker. Producers don't know consumers; consumers don't know producers — fully decoupled.

**Q: Benefits of EDA?**
> Decoupling, scalability (consumers scale independently), resilience (broker buffers if consumer is down), flexibility (add new consumers without touching producer).

**Q: How do you handle duplicate events?**
> Idempotent consumers — each event has a unique ID, consumer checks if it's already processed this ID before acting. Store processed event IDs in Redis or DB.

**Q: How do you handle event ordering?**
> Kafka maintains order within a partition. Use the same partition key (e.g., `userId`) for events that must be ordered relative to each other.

**Q: What is a Dead Letter Queue?**
> Where messages go after failing processing N times. Used for debugging, manual inspection, and retry later.

---

## Final Summary — All Architecture Patterns

```
15. Microservices:     Small, independent services with own databases and deployments
    → One service per business domain; communicate via HTTP or events

16. API Gateway:       Single front door — routing, auth, rate limiting, aggregation
    → All client traffic goes through the gateway

17. Circuit Breaker:   Stop calling a failing service; fast fail; recover gracefully
    → 3 states: CLOSED (normal) → OPEN (failing) → HALF_OPEN (testing)

18. Saga:              Distributed transactions via compensating steps
    → Each step has an undo; run undos in reverse on failure

19. CQRS:              Separate read model from write model
    → Commands (writes) optimized for consistency
    → Queries (reads) optimized for speed

20. Event-Driven:      Services talk via events through a message broker
    → Producers publish; consumers subscribe; broker decouples them
    → Kafka for high volume; RabbitMQ for routing; Redis for simplicity
```
