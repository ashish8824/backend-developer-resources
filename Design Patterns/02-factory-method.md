# 02 — Factory Method Pattern

> **Category:** GoF Creational  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Imagine you work at a restaurant. When a customer orders food, they say "I want pasta." They don't go into the kitchen, gather ingredients, and cook it themselves. They just say what they want, and the kitchen figures out HOW to make it.

The **Factory Method pattern** is exactly this: instead of your code saying `new EmailNotification()` or `new SMSNotification()` directly, you ask a **factory** to create the right object for you. The factory decides which class to instantiate.

**In one sentence:** A factory is a function or class whose job is to create and return objects.

---

## The Problem It Solves

Without Factory Method:

```javascript
// ❌ Your business logic is tightly coupled to concrete classes
function sendAlert(type, message) {
  if (type === 'email') {
    const notifier = new EmailNotifier();      // tied to EmailNotifier
    notifier.send(message);
  } else if (type === 'sms') {
    const notifier = new SMSNotifier();        // tied to SMSNotifier
    notifier.send(message);
  } else if (type === 'push') {
    const notifier = new PushNotifier();       // tied to PushNotifier
    notifier.send(message);
  }
  // Adding WhatsApp? Edit this function again!
}
```

Problems:
1. Every time you add a new notification type, you edit this function
2. This function now knows about ALL notification classes
3. Hard to test — you can't easily swap a fake notifier

With Factory Method — you add a new type by adding a new class, not editing existing code (this is the **Open/Closed Principle**).

---

## Visual Diagram

```
                    ┌─────────────────────┐
                    │  NotificationFactory │
                    │  ─────────────────── │
                    │  create(type)        │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │EmailNotifier │  │ SMSNotifier  │  │ PushNotifier │
   │ ──────────── │  │ ──────────── │  │ ──────────── │
   │ send(msg)    │  │ send(msg)    │  │ send(msg)    │
   └──────────────┘  └──────────────┘  └──────────────┘
         │                  │                  │
         └──────────────────┴──────────────────┘
                            │
                   All implement the same
                   interface: send(message)


Client code only knows about the Factory and the interface.
It never imports EmailNotifier, SMSNotifier, or PushNotifier directly.
```

---

## How It Works (Step by Step)

1. Define a **common interface** (or base class) that all products share
2. Create **concrete classes** that implement that interface
3. Create a **factory** that takes some input and returns the right concrete class
4. Client code calls the factory — it never uses `new ConcreteClass()` directly

---

## Node.js Implementation

### Step 1: Simple Factory (Most Common, Not Strict GoF)

```javascript
// notifiers/EmailNotifier.js
class EmailNotifier {
  constructor(config = {}) {
    this.fromAddress = config.from || 'noreply@app.com';
    this.smtpHost = config.smtpHost || 'smtp.gmail.com';
  }

  async send(message) {
    // In real life: use nodemailer
    console.log(`📧 EMAIL sent to ${message.to}`);
    console.log(`   From: ${this.fromAddress}`);
    console.log(`   Subject: ${message.subject}`);
    console.log(`   Body: ${message.body}`);
    return { success: true, channel: 'email', to: message.to };
  }

  getChannel() { return 'email'; }
}

// notifiers/SMSNotifier.js
class SMSNotifier {
  constructor(config = {}) {
    this.accountSid = config.twilioSid || process.env.TWILIO_SID;
    this.authToken = config.twilioToken || process.env.TWILIO_TOKEN;
    this.fromNumber = config.from || '+1234567890';
  }

  async send(message) {
    // In real life: use twilio SDK
    console.log(`📱 SMS sent to ${message.to}`);
    console.log(`   From: ${this.fromNumber}`);
    console.log(`   Body: ${message.body}`);
    return { success: true, channel: 'sms', to: message.to };
  }

  getChannel() { return 'sms'; }
}

// notifiers/PushNotifier.js
class PushNotifier {
  constructor(config = {}) {
    this.fcmKey = config.fcmKey || process.env.FCM_SERVER_KEY;
  }

  async send(message) {
    // In real life: use firebase-admin
    console.log(`🔔 PUSH sent to device: ${message.to}`);
    console.log(`   Title: ${message.subject}`);
    console.log(`   Body: ${message.body}`);
    return { success: true, channel: 'push', to: message.to };
  }

  getChannel() { return 'push'; }
}

// notifiers/WhatsAppNotifier.js — new one added later, NO existing code changes
class WhatsAppNotifier {
  async send(message) {
    console.log(`💬 WhatsApp sent to ${message.to}`);
    return { success: true, channel: 'whatsapp', to: message.to };
  }
  getChannel() { return 'whatsapp'; }
}
```

```javascript
// NotificationFactory.js
const EmailNotifier = require('./notifiers/EmailNotifier');
const SMSNotifier   = require('./notifiers/SMSNotifier');
const PushNotifier  = require('./notifiers/PushNotifier');
const WhatsAppNotifier = require('./notifiers/WhatsAppNotifier');

class NotificationFactory {
  // Registry approach — easy to add new types
  static #registry = {
    email:    EmailNotifier,
    sms:      SMSNotifier,
    push:     PushNotifier,
    whatsapp: WhatsAppNotifier,
  };

  static create(type, config = {}) {
    const NotifierClass = NotificationFactory.#registry[type];

    if (!NotifierClass) {
      const available = Object.keys(NotificationFactory.#registry).join(', ');
      throw new Error(
        `Unknown notifier type: "${type}". Available: ${available}`
      );
    }

    return new NotifierClass(config);
  }

  // Register a new type at runtime (plugin-style)
  static register(type, NotifierClass) {
    if (NotificationFactory.#registry[type]) {
      throw new Error(`Notifier type "${type}" is already registered`);
    }
    NotificationFactory.#registry[type] = NotifierClass;
    console.log(`✅ Registered new notifier: ${type}`);
  }

  static getAvailableTypes() {
    return Object.keys(NotificationFactory.#registry);
  }
}

module.exports = NotificationFactory;
```

```javascript
// Usage
const NotificationFactory = require('./NotificationFactory');

// Create different notifiers — client doesn't know about concrete classes
const emailNotifier = NotificationFactory.create('email', { from: 'hello@myapp.com' });
const smsNotifier   = NotificationFactory.create('sms');
const pushNotifier  = NotificationFactory.create('push');

const message = {
  to: 'user@example.com',
  subject: 'Your order has shipped!',
  body: 'Your order #1234 is on its way.',
};

await emailNotifier.send(message);
// 📧 EMAIL sent to user@example.com

await smsNotifier.send({ to: '+919876543210', body: 'Your order shipped!' });
// 📱 SMS sent to +919876543210

// Adding a custom notifier at runtime:
class SlackNotifier {
  async send(message) {
    console.log(`💼 Slack message sent to ${message.to}`);
    return { success: true, channel: 'slack' };
  }
  getChannel() { return 'slack'; }
}

NotificationFactory.register('slack', SlackNotifier);
const slackNotifier = NotificationFactory.create('slack');
await slackNotifier.send({ to: '#alerts', body: 'Server is down!' });
```

---

### Step 2: Real-World — NotificationService Using the Factory

```javascript
// NotificationService.js
const NotificationFactory = require('./NotificationFactory');

class NotificationService {
  constructor() {
    this.defaultChannels = ['email'];
    this.retryAttempts = 3;
  }

  // Send via one channel
  async send(type, message) {
    const notifier = NotificationFactory.create(type);
    return await this.#sendWithRetry(notifier, message);
  }

  // Send via multiple channels at once
  async sendMultiChannel(types, message) {
    const results = await Promise.allSettled(
      types.map(type => this.send(type, message))
    );

    return results.map((result, index) => ({
      channel: types[index],
      status: result.status,
      value: result.value,
      error: result.reason?.message,
    }));
  }

  // Send based on user preferences (from DB)
  async sendToUser(user, message) {
    const channels = user.notificationPreferences || this.defaultChannels;
    console.log(`Sending to user ${user.id} via: ${channels.join(', ')}`);
    return this.sendMultiChannel(channels, { ...message, to: user.email });
  }

  async #sendWithRetry(notifier, message, attempt = 1) {
    try {
      return await notifier.send(message);
    } catch (error) {
      if (attempt < this.retryAttempts) {
        console.warn(`Attempt ${attempt} failed, retrying...`);
        await new Promise(r => setTimeout(r, 1000 * attempt)); // exponential backoff
        return this.#sendWithRetry(notifier, message, attempt + 1);
      }
      throw error;
    }
  }
}

// Usage
const notificationService = new NotificationService();

// Send to a specific user based on their preferences
const user = {
  id: 42,
  email: 'alice@example.com',
  phone: '+919876543210',
  deviceToken: 'abc123',
  notificationPreferences: ['email', 'push'],
};

const results = await notificationService.sendToUser(user, {
  subject: 'Welcome to our app!',
  body: 'Thanks for signing up.',
});

console.log(results);
// [
//   { channel: 'email', status: 'fulfilled', value: { success: true } },
//   { channel: 'push',  status: 'fulfilled', value: { success: true } }
// ]
```

---

### Step 3: Abstract Factory (Factory of Factories)

When you need families of related objects:

```javascript
// Payment processors — each has a ChargeStrategy AND a RefundStrategy

// Stripe implementations
class StripeChargeStrategy {
  async charge(amount, currency, source) {
    console.log(`💳 Stripe charging ${currency} ${amount}`);
    return { transactionId: `stripe_${Date.now()}`, status: 'success' };
  }
}

class StripeRefundStrategy {
  async refund(transactionId, amount) {
    console.log(`↩️  Stripe refunding ${amount} for ${transactionId}`);
    return { refundId: `stripe_refund_${Date.now()}` };
  }
}

// PayPal implementations
class PayPalChargeStrategy {
  async charge(amount, currency, source) {
    console.log(`🅿️  PayPal charging ${currency} ${amount}`);
    return { transactionId: `paypal_${Date.now()}`, status: 'success' };
  }
}

class PayPalRefundStrategy {
  async refund(transactionId, amount) {
    console.log(`↩️  PayPal refunding ${amount} for ${transactionId}`);
    return { refundId: `paypal_refund_${Date.now()}` };
  }
}

// Abstract Factory — creates related objects together
class PaymentProcessorFactory {
  static create(provider) {
    const factories = {
      stripe: {
        createChargeStrategy: () => new StripeChargeStrategy(),
        createRefundStrategy: () => new StripeRefundStrategy(),
      },
      paypal: {
        createChargeStrategy: () => new PayPalChargeStrategy(),
        createRefundStrategy: () => new PayPalRefundStrategy(),
      },
    };

    const factory = factories[provider];
    if (!factory) throw new Error(`Unknown payment provider: ${provider}`);
    return factory;
  }
}

// Usage — client code doesn't know which provider it's using
const provider = process.env.PAYMENT_PROVIDER || 'stripe'; // from config
const paymentFactory = PaymentProcessorFactory.create(provider);

const charger = paymentFactory.createChargeStrategy();
const refunder = paymentFactory.createRefundStrategy();

const result = await charger.charge(99.99, 'USD', 'card_tok_123');
// 💳 Stripe charging USD 99.99

const refund = await refunder.refund(result.transactionId, 99.99);
// ↩️  Stripe refunding 99.99 for stripe_1234567890
```

---

## Real-World Examples in Node.js Ecosystem

### Express.js — Router as a factory
```javascript
// Express's Router is a factory for route handlers
const router = express.Router(); // factory creates a router instance
router.get('/users', handler);
```

### Mongoose — Model as a factory
```javascript
// mongoose.model() is a factory — creates Model classes
const User = mongoose.model('User', userSchema); // factory!
const Order = mongoose.model('Order', orderSchema); // factory!

const user = new User({ name: 'Alice' }); // instances from the factory-made class
```

### Sequelize — ORM factories
```javascript
const User = sequelize.define('User', { /* schema */ }); // factory
```

---

## When to Use Factory Method

| Situation | Use? | Reason |
|-----------|------|--------|
| Multiple interchangeable implementations | ✅ | Core use case |
| Adding new types without changing existing code | ✅ | Open/Closed Principle |
| The type to create is determined at runtime | ✅ | Dynamic object creation |
| Need to centralize object creation logic | ✅ | Single place to add validation |
| Only one type, unlikely to change | ❌ | Over-engineering |
| Simple `if/else` with 2 types | ❌ | Too simple, factory overkill |

---

## Interview Questions & Answers

**Q: What is the Factory Method pattern?**
> It defines an interface for creating objects but lets the factory function/class decide which concrete class to instantiate. Client code is decoupled from concrete implementations.

**Q: How is Factory Method different from Abstract Factory?**
> Factory Method creates ONE product. Abstract Factory creates FAMILIES of related products. Example: Factory Method creates "a notifier". Abstract Factory creates "a charge strategy + a refund strategy" that belong together.

**Q: How is Factory different from just using `new`?**
> `new ConcreteClass()` couples you to that specific class. Factory decouples — client just says "give me a notifier for email", and the factory decides which class to use. You can change the implementation without changing the client.

**Q: Where is Factory Method used in Node.js ecosystem?**
> `mongoose.model()`, `express.Router()`, database driver connection factories, `http.createServer()`, `crypto.createHash('sha256')`.

**Q: What's the Registry pattern in Factory?**
> Storing notifier types in a map (`{ email: EmailNotifier, sms: SMSNotifier }`) allows adding new types without modifying the factory's `create()` method — just add to the registry.

---

## Summary

```
Factory Method Pattern
├── What: Delegate object creation to a factory function/class
├── Why: Decouple client from concrete implementations
├── How: Common interface + factory that maps type → class
├── Node.js: Registry map, module-level factories, dynamic requires
└── Watch out: Don't over-engineer simple 2-option if/else cases
```
