# 05 — Adapter Pattern

> **Category:** GoF Structural  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

You buy a laptop in India, then travel to the UK. The power sockets are different shapes. Do you buy a new laptop? No — you use a **power adapter** that fits the UK socket on one side and your Indian plug on the other.

The **Adapter pattern** does the same thing in code: it's a wrapper that **translates one interface into another** so incompatible pieces of code can work together.

**In one sentence:** An Adapter wraps an incompatible class/object and makes it look like the interface your code expects.

---

## The Problem It Solves

You have:
- Your code that expects an interface (e.g., `paymentGateway.charge(amount, currency)`)
- A third-party library with a different interface (e.g., `stripeClient.charges.create({ amount, currency, source })`)

You can't change the library. You can't change your interface without breaking everything. You create an Adapter in the middle.

```
Your Code ──────> [Adapter] ──────> Third-Party Library
(expects your      translates        (has its own
 interface)        the call)          weird interface)
```

---

## Visual Diagram

```
┌───────────────────────────────────────────────────────────────┐
│                      Adapter Pattern                           │
│                                                               │
│  Your App            Adapter               External Library   │
│  ────────            ───────               ────────────────   │
│                                                               │
│  paymentGateway  ──> StripeAdapter      ──> stripe.charges   │
│  .charge(         │  .charge(amount,    │   .create({        │
│    amount,        │    currency) {      │     amount: amt*100,│
│    currency       │    stripe.charges   │     currency,      │
│  )                │    .create({...})   │     source: token  │
│                   │  }                  │   })               │
│                   └─────────────────────┘                    │
│                                                               │
│  Your app never changes.                                      │
│  Swap Stripe → PayPal? Write PayPalAdapter, done.            │
└───────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Version 1: Payment Gateway Adapter

```javascript
// The interface YOUR code expects
// (This is the "Target Interface" — what your app talks to)

// paymentGateway.js — defines what a payment gateway should look like
class PaymentGateway {
  async charge(amount, currency, customerId, description) {
    throw new Error('charge() must be implemented');
  }

  async refund(transactionId, amount) {
    throw new Error('refund() must be implemented');
  }

  async getTransaction(transactionId) {
    throw new Error('getTransaction() must be implemented');
  }
}

// ── The "Adaptees" — third-party SDKs with their OWN interfaces ──

// Fake Stripe SDK (pretend this is the npm package)
class StripeSDK {
  constructor(apiKey) { this.apiKey = apiKey; }

  async charges_create({ amount, currency, customer, description, source }) {
    // Stripe wants amount in CENTS (paise for INR)
    console.log(`[Stripe] Creating charge: ${currency} ${amount} paise`);
    return {
      id: `ch_stripe_${Date.now()}`,
      amount,
      currency,
      status: 'succeeded',
      customer,
      created: Math.floor(Date.now() / 1000),
    };
  }

  async refunds_create({ charge, amount }) {
    console.log(`[Stripe] Refunding charge: ${charge}`);
    return { id: `re_stripe_${Date.now()}`, charge, amount, status: 'succeeded' };
  }

  async charges_retrieve(chargeId) {
    console.log(`[Stripe] Retrieving charge: ${chargeId}`);
    return { id: chargeId, status: 'succeeded', amount: 9900 };
  }
}

// Fake Razorpay SDK
class RazorpaySDK {
  constructor({ key_id, key_secret }) {
    this.keyId = key_id;
    this.keySecret = key_secret;
  }

  async orders_create({ amount, currency, receipt, notes }) {
    // Razorpay also uses paise
    console.log(`[Razorpay] Creating order: ${currency} ${amount} paise`);
    return {
      id: `order_razorpay_${Date.now()}`,
      amount,
      currency,
      receipt,
      status: 'created',
    };
  }

  async payments_fetch(paymentId) {
    console.log(`[Razorpay] Fetching payment: ${paymentId}`);
    return { id: paymentId, status: 'captured', amount: 9900 };
  }

  async payments_refund(paymentId, { amount, notes }) {
    console.log(`[Razorpay] Refunding payment: ${paymentId}`);
    return { id: `rfnd_${Date.now()}`, payment_id: paymentId, amount };
  }
}

// ── Now the ADAPTERS ──

// StripeAdapter — wraps Stripe SDK, exposes YOUR interface
class StripeAdapter extends PaymentGateway {
  #stripe;

  constructor(apiKey) {
    super();
    this.#stripe = new StripeSDK(apiKey);
  }

  async charge(amount, currency, customerId, description) {
    // YOUR interface uses rupees; Stripe wants paise (multiply by 100)
    const amountInPaise = Math.round(amount * 100);

    const charge = await this.#stripe.charges_create({
      amount: amountInPaise,
      currency: currency.toLowerCase(),
      customer: customerId,
      description,
      source: 'tok_visa',  // Stripe-specific field
    });

    // Translate Stripe response → YOUR format
    return {
      transactionId: charge.id,
      amount,
      currency,
      status: charge.status === 'succeeded' ? 'success' : 'failed',
      provider: 'stripe',
      createdAt: new Date(charge.created * 1000).toISOString(),
    };
  }

  async refund(transactionId, amount) {
    const amountInPaise = Math.round(amount * 100);
    const refund = await this.#stripe.refunds_create({
      charge: transactionId,
      amount: amountInPaise,
    });

    return {
      refundId: refund.id,
      originalTransactionId: transactionId,
      amount,
      status: refund.status === 'succeeded' ? 'success' : 'failed',
    };
  }

  async getTransaction(transactionId) {
    const charge = await this.#stripe.charges_retrieve(transactionId);
    return {
      transactionId: charge.id,
      amount: charge.amount / 100,  // convert paise → rupees
      status: charge.status,
    };
  }
}

// RazorpayAdapter — wraps Razorpay SDK, exposes YOUR interface
class RazorpayAdapter extends PaymentGateway {
  #razorpay;

  constructor(keyId, keySecret) {
    super();
    this.#razorpay = new RazorpaySDK({ key_id: keyId, key_secret: keySecret });
  }

  async charge(amount, currency, customerId, description) {
    const amountInPaise = Math.round(amount * 100);

    const order = await this.#razorpay.orders_create({
      amount: amountInPaise,
      currency: currency.toUpperCase(),
      receipt: `receipt_${Date.now()}`,
      notes: { customerId, description },
    });

    return {
      transactionId: order.id,
      amount,
      currency,
      status: order.status === 'created' ? 'success' : 'failed',
      provider: 'razorpay',
      createdAt: new Date().toISOString(),
    };
  }

  async refund(transactionId, amount) {
    const amountInPaise = Math.round(amount * 100);
    const refund = await this.#razorpay.payments_refund(transactionId, {
      amount: amountInPaise,
      notes: { reason: 'Customer request' },
    });

    return {
      refundId: refund.id,
      originalTransactionId: transactionId,
      amount,
      status: 'success',
    };
  }

  async getTransaction(transactionId) {
    const payment = await this.#razorpay.payments_fetch(transactionId);
    return {
      transactionId: payment.id,
      amount: payment.amount / 100,
      status: payment.status,
    };
  }
}

// ── Your application code — never changes ──

class OrderService {
  #paymentGateway; // expects PaymentGateway interface

  constructor(paymentGateway) {
    this.#paymentGateway = paymentGateway; // inject whichever adapter
  }

  async processOrder(order) {
    console.log(`Processing order for ₹${order.total}`);

    const result = await this.#paymentGateway.charge(
      order.total,
      'INR',
      order.customerId,
      `Order #${order.id}`
    );

    if (result.status === 'success') {
      console.log(`✅ Payment successful! Transaction: ${result.transactionId}`);
    }
    return result;
  }

  async cancelOrder(transactionId, amount) {
    return this.#paymentGateway.refund(transactionId, amount);
  }
}

// ── Switch payment providers with ONE line change ──

// Using Stripe
const stripeGateway = new StripeAdapter(process.env.STRIPE_SECRET_KEY);
const orderService = new OrderService(stripeGateway);

// Switch to Razorpay? Change just this:
// const razorpayGateway = new RazorpayAdapter(process.env.RAZORPAY_KEY_ID, process.env.RAZORPAY_KEY_SECRET);
// const orderService = new OrderService(razorpayGateway);

await orderService.processOrder({ id: '1001', total: 999, customerId: 'cust_abc' });
// [Stripe] Creating charge: INR 99900 paise
// ✅ Payment successful! Transaction: ch_stripe_1234567890
```

---

### Version 2: Legacy System Adapter

```javascript
// Adapting an old XML-based internal API to work with your JSON-based code

const xml2js = require('xml2js'); // npm package to parse XML

// ── Old legacy inventory system (pretend you can't change it) ──
class LegacyInventorySystem {
  // Returns XML (old-school!)
  async getStock(productCode) {
    return `
      <inventory>
        <product>
          <code>${productCode}</code>
          <name>Laptop Pro 15</name>
          <quantity>42</quantity>
          <warehouse>MAIN</warehouse>
          <lastUpdated>2024-01-15</lastUpdated>
        </product>
      </inventory>
    `;
  }

  // Expects XML input
  async updateStock(xmlPayload) {
    console.log('[Legacy] Updating stock with XML:', xmlPayload.substring(0, 50) + '...');
    return `<result><status>OK</status><updatedAt>2024-01-15</updatedAt></result>`;
  }
}

// ── Adapter — makes legacy system look like a modern JSON API ──
class InventoryAdapter {
  #legacySystem;
  #parser;
  #builder;

  constructor() {
    this.#legacySystem = new LegacyInventorySystem();
    this.#parser = new xml2js.Parser({ explicitArray: false });
    this.#builder = new xml2js.Builder();
  }

  async getStock(productCode) {
    // Call legacy system
    const xmlResponse = await this.#legacySystem.getStock(productCode);

    // Parse XML → JSON
    const parsed = await this.#parser.parseStringPromise(xmlResponse);
    const product = parsed.inventory.product;

    // Return clean JSON
    return {
      productCode: product.code,
      name: product.name,
      quantity: parseInt(product.quantity),
      warehouse: product.warehouse,
      lastUpdated: new Date(product.lastUpdated),
      available: parseInt(product.quantity) > 0,
    };
  }

  async updateStock(productCode, quantity) {
    // Convert JSON → XML for legacy system
    const xmlPayload = this.#builder.buildObject({
      stockUpdate: {
        productCode,
        quantity,
        updatedBy: 'api',
        timestamp: new Date().toISOString(),
      },
    });

    const xmlResponse = await this.#legacySystem.updateStock(xmlPayload);

    // Parse XML response → JSON
    const parsed = await this.#parser.parseStringPromise(xmlResponse);
    return {
      success: parsed.result.status === 'OK',
      updatedAt: new Date(parsed.result.updatedAt),
    };
  }
}

// Your modern code — only knows JSON
const inventory = new InventoryAdapter();

const stock = await inventory.getStock('LAPTOP-PRO-15');
console.log(stock);
// { productCode: 'LAPTOP-PRO-15', name: 'Laptop Pro 15', quantity: 42, available: true, ... }

const updateResult = await inventory.updateStock('LAPTOP-PRO-15', 40);
console.log(updateResult); // { success: true, updatedAt: Date }
```

---

### Version 3: Logger Adapter (Standardizing Multiple Logging Libraries)

```javascript
// Suppose your app must support winston, pino, or bunyan depending on environment
// You don't want your code to care which one is installed

// Fake Winston
class WinstonLogger {
  log(level, message, meta) {
    console.log(`[Winston][${level.toUpperCase()}] ${message}`, meta || '');
  }
}

// Fake Pino
class PinoLogger {
  info(obj, msg) { console.log(`[Pino][INFO] ${msg}`, obj || ''); }
  warn(obj, msg) { console.log(`[Pino][WARN] ${msg}`, obj || ''); }
  error(obj, msg) { console.log(`[Pino][ERROR] ${msg}`, obj || ''); }
  debug(obj, msg) { console.log(`[Pino][DEBUG] ${msg}`, obj || ''); }
}

// YOUR standard logger interface
class Logger {
  info(message, meta)  { throw new Error('Not implemented'); }
  warn(message, meta)  { throw new Error('Not implemented'); }
  error(message, meta) { throw new Error('Not implemented'); }
  debug(message, meta) { throw new Error('Not implemented'); }
}

// Winston Adapter
class WinstonAdapter extends Logger {
  #winston;
  constructor() { super(); this.#winston = new WinstonLogger(); }

  info(message, meta)  { this.#winston.log('info', message, meta); }
  warn(message, meta)  { this.#winston.log('warn', message, meta); }
  error(message, meta) { this.#winston.log('error', message, meta); }
  debug(message, meta) { this.#winston.log('debug', message, meta); }
}

// Pino Adapter — different call signature!
class PinoAdapter extends Logger {
  #pino;
  constructor() { super(); this.#pino = new PinoLogger(); }

  // Pino puts meta FIRST, message SECOND — adapter flips them
  info(message, meta)  { this.#pino.info(meta || {}, message); }
  warn(message, meta)  { this.#pino.warn(meta || {}, message); }
  error(message, meta) { this.#pino.error(meta || {}, message); }
  debug(message, meta) { this.#pino.debug(meta || {}, message); }
}

// Your application code — always the same interface
function createLogger() {
  const logLibrary = process.env.LOG_LIBRARY || 'winston';
  const adapters = { winston: WinstonAdapter, pino: PinoAdapter };
  const Adapter = adapters[logLibrary] || WinstonAdapter;
  return new Adapter();
}

const logger = createLogger();
logger.info('Server started', { port: 3000 });
logger.error('Connection failed', { host: 'localhost', retries: 3 });
// Works identically regardless of which library is underneath
```

---

## Adapter vs Facade — Know the Difference

This is a common interview question:

| | Adapter | Facade |
|--|---------|--------|
| **Purpose** | Make incompatible interface compatible | Simplify a complex interface |
| **Wraps** | One class/object | Multiple classes/subsystems |
| **Interface change** | Yes — translates to expected interface | Yes — hides complexity |
| **Analogy** | Power plug adapter | Hotel concierge (handles everything for you) |
| **When** | Third-party libraries, legacy systems | Complex subsystems, multiple services |

---

## When to Use Adapter

| Situation | Use? |
|-----------|------|
| Integrating third-party library with different interface | ✅ Yes |
| Working with legacy code you can't modify | ✅ Yes |
| Switching providers (Stripe → Razorpay) without changing app | ✅ Yes |
| Standardizing multiple library interfaces | ✅ Yes |
| Classes already share the same interface | ❌ No |

---

## Interview Questions & Answers

**Q: What is the Adapter pattern?**
> A structural pattern that converts the interface of a class into another interface clients expect. It lets incompatible classes work together without modifying their source code.

**Q: What are real-world examples in Node.js?**
> Payment gateway wrappers (Stripe/Razorpay), database driver adapters (switching from MySQL to PostgreSQL), logging library adapters (Winston/Pino), third-party API wrappers.

**Q: Adapter vs Facade?**
> Adapter makes two incompatible interfaces work together (1:1 translation). Facade simplifies a complex set of interfaces into one simple interface (many:1 simplification).

**Q: Adapter vs Decorator?**
> Adapter changes the interface. Decorator keeps the same interface but adds behavior. You use Adapter when interfaces don't match; you use Decorator when you want to add features.

---

## Summary

```
Adapter Pattern
├── What: Translate one interface into another
├── Why: Third-party libs, legacy code, provider switching
├── How: Wrapper class that calls the adaptee, maps fields, converts types
├── Node.js: Payment gateways, API clients, logging libraries
└── Vs Facade: Adapter = interface translation | Facade = complexity hiding
```
