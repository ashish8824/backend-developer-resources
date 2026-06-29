# 07 — Facade Pattern

> **Category:** GoF Structural  
> **Difficulty:** ⭐ Beginner  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

When you check into a hotel, you don't deal with housekeeping, the kitchen, the parking team, and the laundry service separately. You call the **concierge** and say "I need my room cleaned and dinner at 7." The concierge handles ALL of that behind the scenes.

A **Facade** is a concierge for your code: a single simple interface that hides a complex system of many moving parts underneath.

**In one sentence:** A Facade provides one clean, simple interface to a complicated subsystem.

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Facade Pattern                            │
│                                                                 │
│  Client                                                         │
│   │                                                             │
│   └──> OrderFacade.placeOrder(item, userId)                     │
│              │                                                  │
│              ├──> InventoryService.reserve(item)                │
│              ├──> PaymentService.charge(userId, amount)         │
│              ├──> ShippingService.createLabel(item, userId)     │
│              ├──> NotificationService.sendConfirmation(userId)  │
│              └──> AuditService.log('order.placed', ...)         │
│                                                                 │
│  Client only calls ONE method.                                  │
│  Facade orchestrates FIVE services.                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

```javascript
// ── Complex subsystems (each does one thing) ──

class InventoryService {
  async checkAvailability(productId, qty) {
    console.log(`[Inventory] Checking ${qty}x product ${productId}`);
    return { available: true, stockLeft: 50 };
  }
  async reserve(productId, qty, orderId) {
    console.log(`[Inventory] Reserved ${qty}x product ${productId} for order ${orderId}`);
    return { reservationId: `res_${Date.now()}` };
  }
  async release(reservationId) {
    console.log(`[Inventory] Released reservation ${reservationId}`);
  }
}

class PaymentService {
  async charge(userId, amount, currency = 'INR') {
    console.log(`[Payment] Charging ₹${amount} to user ${userId}`);
    return { transactionId: `txn_${Date.now()}`, status: 'success' };
  }
  async refund(transactionId) {
    console.log(`[Payment] Refunding transaction ${transactionId}`);
    return { refundId: `ref_${Date.now()}` };
  }
}

class ShippingService {
  async calculateCost(productId, userAddress) {
    console.log(`[Shipping] Calculating cost for product ${productId}`);
    return { cost: 49, estimatedDays: 3 };
  }
  async createLabel(orderId, productId, address) {
    console.log(`[Shipping] Creating label for order ${orderId}`);
    return { trackingId: `TRK${Date.now()}`, carrier: 'FedEx' };
  }
}

class NotificationService {
  async sendOrderConfirmation(userId, orderId, trackingId) {
    console.log(`[Notification] Sending confirmation to user ${userId} for order ${orderId}`);
    return { emailSent: true, smsSent: true };
  }
  async sendOrderCancellation(userId, orderId) {
    console.log(`[Notification] Sending cancellation to user ${userId} for order ${orderId}`);
  }
}

class AuditService {
  async log(event, data) {
    console.log(`[Audit] ${event}:`, JSON.stringify(data));
  }
}

// ── The Facade ──

class OrderFacade {
  #inventory;
  #payment;
  #shipping;
  #notification;
  #audit;

  constructor() {
    // Facade owns and manages all subsystems
    this.#inventory    = new InventoryService();
    this.#payment      = new PaymentService();
    this.#shipping     = new ShippingService();
    this.#notification = new NotificationService();
    this.#audit        = new AuditService();
  }

  // ONE simple method for client to call
  async placeOrder({ productId, qty, userId, address, amount }) {
    const orderId = `ord_${Date.now()}`;
    let reservation = null;
    let transaction = null;

    try {
      // Step 1: Check and reserve inventory
      const stock = await this.#inventory.checkAvailability(productId, qty);
      if (!stock.available) throw new Error('Product out of stock');

      reservation = await this.#inventory.reserve(productId, qty, orderId);

      // Step 2: Charge payment
      transaction = await this.#payment.charge(userId, amount);
      if (transaction.status !== 'success') throw new Error('Payment failed');

      // Step 3: Create shipping label
      const shipping = await this.#shipping.createLabel(orderId, productId, address);

      // Step 4: Notify user
      await this.#notification.sendOrderConfirmation(userId, orderId, shipping.trackingId);

      // Step 5: Audit log
      await this.#audit.log('order.placed', { orderId, userId, productId, amount });

      console.log(`✅ Order ${orderId} placed successfully!`);
      return { orderId, trackingId: shipping.trackingId, transactionId: transaction.transactionId };

    } catch (error) {
      // Rollback on failure
      console.error(`❌ Order failed: ${error.message}. Rolling back...`);
      if (reservation) await this.#inventory.release(reservation.reservationId);
      if (transaction) await this.#payment.refund(transaction.transactionId);
      await this.#audit.log('order.failed', { orderId, userId, error: error.message });
      throw error;
    }
  }

  async cancelOrder(orderId, userId) {
    // Another complex operation simplified to one call
    console.log(`Cancelling order ${orderId}`);
    await this.#payment.refund(`txn_for_${orderId}`);
    await this.#notification.sendOrderCancellation(userId, orderId);
    await this.#audit.log('order.cancelled', { orderId, userId });
    return { cancelled: true };
  }
}

// ── Client code — beautifully simple ──

const orderFacade = new OrderFacade();

const result = await orderFacade.placeOrder({
  productId: 'LAPTOP-001',
  qty: 1,
  userId: 'user_42',
  address: { city: 'Mumbai', pin: '400001' },
  amount: 79999,
});

console.log(result);
// { orderId: 'ord_...', trackingId: 'TRK...', transactionId: 'txn_...' }
```

---

## Facade vs Adapter

| | Facade | Adapter |
|--|--------|---------|
| **Wraps** | Multiple subsystems | One object |
| **Purpose** | Simplify complexity | Translate interface |
| **Interface** | Creates a new simpler one | Converts to expected one |
| **Analogy** | Hotel concierge | Power plug adapter |

---

## Interview Q&A

**Q: Facade vs Adapter?** Facade hides complexity of MANY classes. Adapter makes ONE class match an expected interface.

**Q: Real-world examples?** AWS SDK (one call triggers many internal AWS services), Mongoose (hides MongoDB driver complexity), Express app setup.

---

---

# 08 — Proxy Pattern

> **Category:** GoF Structural  
> **Difficulty:** ⭐⭐⭐ Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

A **Proxy** is a stand-in or gatekeeper for another object. It has the **same interface** as the real object, so clients don't know they're talking to a proxy. But the proxy can intercept calls and add behavior: access control, caching, lazy loading, logging.

Real-life example: A **credit card** is a proxy for your bank account. You use it like cash (same interface), but the card company can intercept the transaction, add fraud detection, track spending, and decide whether to allow it.

**In one sentence:** A proxy sits in front of a real object, intercepts calls, and optionally adds behavior like caching, auth checks, or lazy loading.

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                       Proxy Pattern                          │
│                                                             │
│  Client                                                     │
│    │                                                        │
│    │  same interface as real object                         │
│    ▼                                                        │
│  ┌─────────────────────┐                                    │
│  │      Proxy          │                                    │
│  │  ────────────────── │                                    │
│  │  - intercept call   │                                    │
│  │  - check cache?     │──────> ┌────────────────────┐     │
│  │  - check auth?      │        │   Real Object      │     │
│  │  - log?             │<────── │   (Service/DB/API) │     │
│  │  - transform?       │        └────────────────────┘     │
│  └─────────────────────┘                                    │
│                                                             │
│  Types of Proxy:                                            │
│  ● Virtual Proxy    — lazy loading (create real obj late)   │
│  ● Protection Proxy — access control                        │
│  ● Cache Proxy      — cache results                         │
│  ● Logging Proxy    — audit trail                           │
│  ● Remote Proxy     — represent object in another server    │
└─────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Virtual Proxy (Lazy Loading)

```javascript
// Heavy object — expensive to create
class HeavyReportGenerator {
  constructor() {
    console.log('📊 Loading report generator (expensive: connecting to analytics DB...)');
    this.analyticsDB = 'connected'; // imagine real DB connection setup
  }

  async generateSalesReport(month, year) {
    console.log(`Generating sales report for ${month}/${year}`);
    return { month, year, revenue: 1250000, orders: 842 };
  }

  async generateUserReport(userId) {
    return { userId, totalOrders: 12, totalSpent: 45000 };
  }
}

// Virtual Proxy — only creates HeavyReportGenerator when first needed
class ReportGeneratorProxy {
  #realGenerator = null; // not created yet

  #getReal() {
    if (!this.#realGenerator) {
      console.log('[PROXY] Creating real report generator for the first time...');
      this.#realGenerator = new HeavyReportGenerator();
    }
    return this.#realGenerator;
  }

  async generateSalesReport(month, year) {
    return this.#getReal().generateSalesReport(month, year);
  }

  async generateUserReport(userId) {
    return this.#getReal().generateUserReport(userId);
  }
}

const reportProxy = new ReportGeneratorProxy();
// 👆 HeavyReportGenerator NOT created yet — proxy just exists

// Only when we actually need a report:
const report = await reportProxy.generateSalesReport('Jan', 2024);
// [PROXY] Creating real report generator for the first time...
// 📊 Loading report generator (expensive: connecting to analytics DB...)
// Generating sales report for Jan/2024
```

---

### Cache Proxy

```javascript
class WeatherAPI {
  async getWeather(city) {
    console.log(`🌐 [API CALL] Fetching weather for ${city}...`);
    // Simulate API call
    await new Promise(r => setTimeout(r, 200));
    return { city, temp: 28, condition: 'Sunny', humidity: 65 };
  }
}

class CachingWeatherProxy {
  #api;
  #cache = new Map();
  #ttlMs;

  constructor(ttlSeconds = 300) {
    this.#api = new WeatherAPI();
    this.#ttlMs = ttlSeconds * 1000;
  }

  async getWeather(city) {
    const key = city.toLowerCase();
    const cached = this.#cache.get(key);

    if (cached && Date.now() < cached.expiresAt) {
      console.log(`✅ [CACHE HIT] Returning cached weather for ${city}`);
      return cached.data;
    }

    const weather = await this.#api.getWeather(city);
    this.#cache.set(key, { data: weather, expiresAt: Date.now() + this.#ttlMs });
    console.log(`💾 [CACHE SET] Cached weather for ${city}`);
    return weather;
  }

  clearCache() { this.#cache.clear(); }
  getCacheSize() { return this.#cache.size; }
}

const weather = new CachingWeatherProxy(60); // cache for 60 seconds

await weather.getWeather('Mumbai');  // 🌐 API CALL
await weather.getWeather('Mumbai');  // ✅ CACHE HIT
await weather.getWeather('Delhi');   // 🌐 API CALL
await weather.getWeather('Mumbai');  // ✅ CACHE HIT
```

---

### JavaScript's Built-in Proxy Object

```javascript
// JavaScript has a NATIVE Proxy object — extremely powerful

const user = { name: 'Alice', age: 25, _password: 'secret123' };

const safeUser = new Proxy(user, {
  // Intercept property reads
  get(target, prop) {
    if (prop.startsWith('_')) {
      throw new Error(`Access denied: "${prop}" is private`);
    }
    console.log(`[PROXY] Reading: ${prop}`);
    return target[prop];
  },

  // Intercept property writes
  set(target, prop, value) {
    if (prop === 'age' && (typeof value !== 'number' || value < 0 || value > 150)) {
      throw new Error(`Invalid age: ${value}`);
    }
    if (prop.startsWith('_')) {
      throw new Error(`Cannot set private property: ${prop}`);
    }
    console.log(`[PROXY] Setting: ${prop} = ${value}`);
    target[prop] = value;
    return true;
  },

  // Intercept property existence checks
  has(target, prop) {
    if (prop.startsWith('_')) return false; // hide private props
    return prop in target;
  },
});

console.log(safeUser.name);  // [PROXY] Reading: name → 'Alice'
safeUser.age = 26;           // [PROXY] Setting: age = 26

try { console.log(safeUser._password); } // Error: Access denied
catch(e) { console.error(e.message); }

try { safeUser.age = -5; }  // Error: Invalid age: -5
catch(e) { console.error(e.message); }

console.log('_password' in safeUser); // false (hidden by has trap)
```

---

## Proxy vs Decorator

| | Proxy | Decorator |
|--|-------|-----------|
| **Intent** | Control access/lifecycle | Add behavior/features |
| **Client knows?** | Usually NO (transparent) | Sometimes yes |
| **Examples** | Auth, cache, lazy load, logging | Logging, retry, validation |

---

## Interview Q&A

**Q: Types of Proxy?** Virtual (lazy load), Protection (auth), Cache (memoize), Logging (audit), Remote (represent remote object).

**Q: JavaScript's `Proxy` object?** A native language feature that lets you intercept property access, assignment, function calls, etc. on any object using traps (`get`, `set`, `has`, `apply`).

---

---

# 09 — Observer Pattern

> **Category:** GoF Behavioral  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

You subscribe to a YouTube channel. When the creator uploads a video, **all subscribers** automatically get notified — without the creator needing to know who you are or contact you directly.

The **Observer pattern** works exactly like this:
- **Subject** (YouTube channel) — maintains a list of observers and notifies them when something changes
- **Observers** (subscribers) — register their interest and react when notified

**In one sentence:** Define a one-to-many dependency between objects so that when one object changes state, all its dependents are automatically notified.

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Observer Pattern                         │
│                                                             │
│   EventEmitter / Subject                                    │
│   ┌──────────────────────────┐                             │
│   │  subscribers = [         │                             │
│   │    emailHandler,         │                             │
│   │    smsHandler,           │                             │
│   │    analyticsHandler      │                             │
│   │  ]                       │                             │
│   │                          │                             │
│   │  emit('order.placed') ───┼──> emailHandler(data)       │
│   │                          ├──> smsHandler(data)          │
│   │                          └──> analyticsHandler(data)   │
│   └──────────────────────────┘                             │
│                                                             │
│   Subject doesn't know or care what handlers do.           │
│   Handlers don't know about each other.                     │
│   Completely decoupled!                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Version 1: Build Your Own EventEmitter

```javascript
// EventEmitter from scratch
class EventEmitter {
  #listeners = new Map(); // event name → array of handlers

  on(event, handler) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, []);
    }
    this.#listeners.get(event).push(handler);
    return this; // chainable
  }

  off(event, handler) {
    const handlers = this.#listeners.get(event) || [];
    this.#listeners.set(event, handlers.filter(h => h !== handler));
    return this;
  }

  once(event, handler) {
    // Auto-removes after first call
    const wrapper = (...args) => {
      handler(...args);
      this.off(event, wrapper);
    };
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    const handlers = this.#listeners.get(event) || [];
    handlers.forEach(handler => {
      try {
        handler(...args);
      } catch (error) {
        console.error(`Error in handler for "${event}":`, error);
      }
    });
    return this;
  }

  listenerCount(event) {
    return (this.#listeners.get(event) || []).length;
  }
}

// ── Usage ──

const orderEvents = new EventEmitter();

// Observers register themselves
orderEvents.on('order.placed', (order) => {
  console.log(`📧 [EmailService] Sending confirmation to ${order.userEmail}`);
});

orderEvents.on('order.placed', (order) => {
  console.log(`📱 [SMSService] Sending SMS to ${order.userPhone}`);
});

orderEvents.on('order.placed', (order) => {
  console.log(`📊 [Analytics] Recording order ${order.id} worth ₹${order.total}`);
});

orderEvents.on('order.placed', async (order) => {
  console.log(`📦 [Inventory] Reserving items for order ${order.id}`);
  // async observers work too
});

orderEvents.once('order.placed', (order) => {
  console.log(`🎁 [Promo] First order detected! Sending welcome coupon.`);
  // This handler fires ONCE then unregisters itself
});

// Subject emits — notifies ALL observers
orderEvents.emit('order.placed', {
  id: 'ORD-1001',
  userEmail: 'alice@example.com',
  userPhone: '+919876543210',
  total: 2499,
  items: [{ sku: 'BOOK-001', qty: 2 }],
});
// All 5 observers fire!

orderEvents.emit('order.placed', {
  id: 'ORD-1002',
  userEmail: 'bob@example.com',
  userPhone: '+919876543211',
  total: 5999,
  items: [{ sku: 'LAPTOP-001', qty: 1 }],
});
// 4 observers fire (the 'once' handler is gone)
```

---

### Version 2: Node.js Built-in EventEmitter (Use This!)

```javascript
// Node.js has EventEmitter built in — always use this, not your own
const EventEmitter = require('events');

class OrderService extends EventEmitter {
  #db;

  constructor(db) {
    super();
    this.#db = db;
  }

  async placeOrder(orderData) {
    // 1. Save to DB
    const order = await this.#db.orders.create(orderData);

    // 2. Emit event — observers handle the rest
    this.emit('placed', order);

    return order;
  }

  async cancelOrder(orderId) {
    const order = await this.#db.orders.findById(orderId);
    await this.#db.orders.update(orderId, { status: 'cancelled' });
    this.emit('cancelled', order);
    return order;
  }

  async shipOrder(orderId, trackingId) {
    const order = await this.#db.orders.update(orderId, { status: 'shipped', trackingId });
    this.emit('shipped', { ...order, trackingId });
    return order;
  }
}

// ── Set up observers elsewhere — clean separation ──

// notifications.js
function setupOrderNotifications(orderService, notificationService) {
  orderService.on('placed', async (order) => {
    await notificationService.sendEmail(order.userEmail, 'order-confirm', order);
    await notificationService.sendSMS(order.userPhone, `Order ${order.id} confirmed!`);
  });

  orderService.on('shipped', async (order) => {
    await notificationService.sendEmail(order.userEmail, 'order-shipped', order);
  });

  orderService.on('cancelled', async (order) => {
    await notificationService.sendEmail(order.userEmail, 'order-cancelled', order);
  });
}

// analytics.js
function setupOrderAnalytics(orderService, analyticsService) {
  orderService.on('placed',    (order) => analyticsService.track('order_placed', order));
  orderService.on('shipped',   (order) => analyticsService.track('order_shipped', order));
  orderService.on('cancelled', (order) => analyticsService.track('order_cancelled', order));
}

// inventory.js
function setupInventoryUpdates(orderService, inventoryService) {
  orderService.on('placed', async (order) => {
    for (const item of order.items) {
      await inventoryService.reserve(item.sku, item.qty, order.id);
    }
  });
  orderService.on('cancelled', async (order) => {
    for (const item of order.items) {
      await inventoryService.release(item.sku, item.qty, order.id);
    }
  });
}

// ── Wire everything up in app.js ──
const orderService = new OrderService(db);
setupOrderNotifications(orderService, notificationService);
setupOrderAnalytics(orderService, analyticsService);
setupInventoryUpdates(orderService, inventoryService);

// Now placing an order automatically triggers all observers:
await orderService.placeOrder({ userId: 1, items: [...], total: 1999 });
// Email sent ✓, SMS sent ✓, Analytics recorded ✓, Inventory reserved ✓
```

---

### Observer vs Pub/Sub (Important Interview Topic)

```
Observer Pattern:                  Pub/Sub Pattern:
────────────────                   ────────────────
Subject knows observers            Publisher does NOT know subscribers
Direct coupling (same process)     Decoupled via message broker (Redis, Kafka)
Synchronous (usually)              Asynchronous (always)
Same service                       Across services / microservices

EventEmitter = Observer
Kafka / RabbitMQ / Redis Pub/Sub = Pub/Sub
```

---

## Interview Q&A

**Q: What is the Observer pattern?**
> One-to-many dependency: when a subject changes state, all registered observers are automatically notified. Decouples the subject from its dependents.

**Q: Observer vs Pub/Sub?**
> Observer: subject knows its observers, same process, direct call. Pub/Sub: publisher doesn't know subscribers, a broker (Kafka, Redis) sits in between, works across services.

**Q: Node.js EventEmitter?**
> Node's built-in implementation of the Observer pattern. `on()` = subscribe, `emit()` = notify, `off()` = unsubscribe, `once()` = subscribe for one event.

**Q: What's the risk of too many observers?**
> Memory leaks if observers aren't removed (Node warns about this: "MaxListenersExceededWarning"). Always `.off()` or `.removeAllListeners()` when done.

---

## Summary (All Three Patterns)

```
Facade: "One door to many rooms"
  → Simplify complex subsystems with a single clean interface
  → OrderFacade.placeOrder() orchestrates 5 services

Proxy: "A gatekeeper with the same face"
  → Control access: cache, lazy load, auth, logging
  → Same interface as real object, client doesn't know

Observer: "Subscribe and get notified"
  → One-to-many: subject emits, all observers react
  → Node.js EventEmitter is the built-in implementation
  → Decouples emitter from handlers completely
```
