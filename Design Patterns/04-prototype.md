# 04 — Prototype Pattern

> **Category:** GoF Creational  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Imagine you've spent 10 minutes configuring a perfect document in Word — fonts, margins, headers, styles. Now you need 5 more similar documents. Do you start from scratch each time? No — you **copy** the first one and tweak only what's different.

That's the Prototype pattern: **create new objects by cloning an existing object** instead of building from scratch.

The existing object is the "prototype" (the template). Cloning it is fast, cheap, and preserves all its configuration.

**In one sentence:** Instead of `new MyClass()`, you do `existingObject.clone()` to get a new object with the same starting state.

---

## The Problem It Solves

Some objects are **expensive to create**:
- Require a DB lookup
- Need complex computation to initialize
- Have deep nested configuration
- Must connect to external services

Creating them fresh every time wastes resources. Cloning a pre-built instance is much faster.

Also, sometimes you want **slight variations** of the same object — clone it, then change just one or two properties.

```
Without Prototype:
  new GameCharacter() → load textures → parse animations → connect to physics → 200ms

With Prototype:
  masterCharacter.clone() → copy memory → 2ms ✅
```

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     Prototype Pattern                        │
│                                                             │
│   Registry (Cache)          clone()                         │
│   ─────────────────                                         │
│   'warrior' ──────────> ┌──────────────┐                   │
│   'mage'    ──────────> │  Prototype   │──── clone() ────> │
│   'archer'  ──────────> │  (Original)  │                   │
│                         └──────────────┘    ┌───────────┐  │
│                                             │   Copy 1  │  │
│                                             │ (modified)│  │
│                                             └───────────┘  │
│                                                             │
│                                             ┌───────────┐  │
│                                             │   Copy 2  │  │
│                                             │ (modified)│  │
│                                             └───────────┘  │
│                                                             │
│   ⚠️  Shallow Clone vs Deep Clone:                          │
│                                                             │
│   Original: { name: 'A', stats: { hp: 100 } }              │
│                                                             │
│   Shallow:  clone.stats === original.stats  ← SAME ref!    │
│   Deep:     clone.stats !== original.stats  ← SAFE copy ✅ │
└─────────────────────────────────────────────────────────────┘
```

---

## The Most Important Concept: Shallow vs Deep Copy

This is what trips everyone up. You MUST understand this before using Prototype.

```javascript
// Original object
const original = {
  name: 'Warrior',
  stats: { hp: 100, mp: 50 },   // ← nested object
  skills: ['slash', 'block'],   // ← array (also an object)
};

// ──── SHALLOW COPY ────
const shallow = { ...original };
// OR: const shallow = Object.assign({}, original);

shallow.name = 'Clone Warrior'; // ✅ Safe — primitive, own copy
shallow.stats.hp = 999;         // ❌ DANGER — modifies original.stats.hp too!
shallow.skills.push('fire');    // ❌ DANGER — modifies original.skills too!

console.log(original.stats.hp);   // 999 ← original was changed!
console.log(original.skills);     // ['slash', 'block', 'fire'] ← changed!

// ──── DEEP COPY ────
const deep = JSON.parse(JSON.stringify(original));
// OR: structuredClone(original)  ← modern Node.js 17+

deep.stats.hp = 999;           // ✅ Safe — original untouched
deep.skills.push('fire');      // ✅ Safe — original untouched

console.log(original.stats.hp);  // 100 ← unchanged ✅
console.log(original.skills);    // ['slash', 'block'] ← unchanged ✅
```

---

## Node.js Implementation

### Version 1: Basic Prototype with clone()

```javascript
// GameCharacter.js

class GameCharacter {
  constructor(name, type, stats, skills, equipment) {
    this.name      = name;
    this.type      = type;           // 'warrior', 'mage', etc.
    this.stats     = stats;          // { hp, mp, attack, defense }
    this.skills    = skills;         // ['skill1', 'skill2']
    this.equipment = equipment;      // { weapon, armor, accessory }
    this.level     = 1;
    this.experience = 0;
    this.createdAt = new Date().toISOString();

    // Simulate expensive init (texture loading, physics setup, etc.)
    console.log(`⚙️  Building character from scratch: ${name} (expensive!)`);
  }

  // ── The Prototype method ──
  clone() {
    // Deep copy using structuredClone (Node.js 17+)
    const cloned = structuredClone(this);

    // structuredClone copies all nested objects safely
    // But we want the clone to be its own character, so reset some fields
    cloned.name      = `${this.name} (clone)`;
    cloned.level     = 1;
    cloned.experience = 0;
    cloned.createdAt = new Date().toISOString();

    console.log(`⚡ Cloned character: ${cloned.name} (fast!)`);
    return cloned;
  }

  // Give the clone a new name
  withName(name) {
    this.name = name;
    return this; // for chaining
  }

  levelUp() {
    this.level++;
    this.stats.hp     += 10;
    this.stats.attack += 2;
    return this;
  }

  addSkill(skill) {
    this.skills.push(skill);
    return this;
  }

  toString() {
    return `${this.name} (Lv${this.level}) | HP:${this.stats.hp} ATK:${this.stats.attack}`;
  }
}

// ── Usage ──

// Create ONE expensive master character
const masterWarrior = new GameCharacter(
  'Master Warrior',
  'warrior',
  { hp: 200, mp: 30, attack: 45, defense: 60 },
  ['slash', 'block', 'charge'],
  { weapon: 'Iron Sword', armor: 'Steel Plate', accessory: 'Shield' }
);
// ⚙️  Building character from scratch: Master Warrior (expensive!)

// Now clone it many times — fast!
const warrior1 = masterWarrior.clone();
warrior1.withName('Aragorn').levelUp().levelUp();

const warrior2 = masterWarrior.clone();
warrior2.withName('Boromir').addSkill('parry');

const warrior3 = masterWarrior.clone();
warrior3.withName('Faramir');

console.log(warrior1.toString());  // Aragorn (Lv3) | HP:220 ATK:49
console.log(warrior2.toString());  // Boromir (Lv1) | HP:200 ATK:45
console.log(warrior3.toString());  // Faramir (Lv1) | HP:200 ATK:45

// Prove they're independent:
warrior1.stats.hp = 999;
console.log(warrior2.stats.hp); // 200 — NOT affected ✅
```

---

### Version 2: Prototype Registry (Most Useful Pattern)

A central store of "template" objects you can clone by name:

```javascript
// PrototypeRegistry.js

class PrototypeRegistry {
  #registry = new Map();

  // Register a prototype object under a name
  register(name, prototype) {
    if (this.#registry.has(name)) {
      throw new Error(`Prototype "${name}" already registered`);
    }
    this.#registry.set(name, prototype);
    console.log(`📦 Registered prototype: ${name}`);
  }

  // Get a CLONE of the registered prototype
  clone(name, overrides = {}) {
    const prototype = this.#registry.get(name);
    if (!prototype) {
      throw new Error(`No prototype registered as "${name}"`);
    }

    // Deep clone and apply overrides
    const cloned = structuredClone(prototype);
    Object.assign(cloned, overrides);
    return cloned;
  }

  list() {
    return [...this.#registry.keys()];
  }
}

// ── Example: Email Template Registry ──

const emailRegistry = new PrototypeRegistry();

// Register base email templates
emailRegistry.register('welcome', {
  from: 'noreply@myapp.com',
  subject: 'Welcome to MyApp!',
  html: '<h1>Welcome!</h1><p>Thanks for joining.</p>',
  headers: { 'X-Email-Type': 'transactional' },
  priority: 'normal',
  trackOpens: true,
  trackClicks: true,
  unsubscribeLink: true,
});

emailRegistry.register('password-reset', {
  from: 'security@myapp.com',
  subject: 'Reset your password',
  html: '<h1>Password Reset</h1><p>Click the link below.</p>',
  headers: { 'X-Email-Type': 'security', 'X-Priority': '1' },
  priority: 'high',
  trackOpens: false,   // don't track security emails
  trackClicks: false,
  unsubscribeLink: false,
});

emailRegistry.register('order-confirm', {
  from: 'orders@myapp.com',
  subject: 'Your order is confirmed!',
  html: '<h1>Order Confirmed</h1>',
  headers: { 'X-Email-Type': 'transactional' },
  priority: 'normal',
  trackOpens: true,
  trackClicks: true,
  unsubscribeLink: false,
});

// ── Now use them ──

// Clone and customize welcome email for a specific user
const aliceWelcome = emailRegistry.clone('welcome', {
  to: 'alice@example.com',
  subject: 'Welcome to MyApp, Alice!',  // override subject
  html: '<h1>Welcome, Alice!</h1><p>We are so glad you joined!</p>',
});

const bobWelcome = emailRegistry.clone('welcome', {
  to: 'bob@example.com',
  subject: 'Welcome to MyApp, Bob!',
  html: '<h1>Welcome, Bob!</h1><p>Get started here.</p>',
});

// Security email — different base config
const resetEmail = emailRegistry.clone('password-reset', {
  to: 'alice@example.com',
  html: `<h1>Reset Password</h1>
         <p><a href="https://myapp.com/reset/tok_abc123">Click here</a></p>
         <p>Expires in 1 hour.</p>`,
});

console.log(aliceWelcome.trackOpens);  // true (from template)
console.log(resetEmail.trackOpens);    // false (from template)
console.log(resetEmail.priority);      // 'high' (from template)

// They're independent — modifying one doesn't affect the template
aliceWelcome.subject = 'CHANGED';
console.log(emailRegistry.clone('welcome').subject); // 'Welcome to MyApp!' — unchanged ✅
```

---

### Version 3: Config Prototype for Different Environments

```javascript
// ConfigPrototype.js

const baseConfig = {
  server: {
    host: 'localhost',
    port: 3000,
    cors: { origin: '*', methods: ['GET', 'POST', 'PUT', 'DELETE'] },
    rateLimit: { windowMs: 15 * 60 * 1000, max: 100 },
  },
  database: {
    host: 'localhost',
    port: 5432,
    name: 'myapp',
    pool: { min: 2, max: 10 },
    ssl: false,
  },
  cache: {
    host: 'localhost',
    port: 6379,
    ttl: 3600,
  },
  logging: {
    level: 'debug',
    format: 'pretty',
    destination: 'console',
  },
};

function createConfig(environment, overrides = {}) {
  // Deep clone the base config
  const config = structuredClone(baseConfig);

  // Apply environment-specific settings
  const envSettings = {
    development: {
      // use defaults mostly
    },
    staging: {
      server: { port: 8080, rateLimit: { max: 50 } },
      database: { host: 'staging-db.internal', ssl: true },
      logging: { level: 'info', format: 'json' },
    },
    production: {
      server: { port: 443, cors: { origin: 'https://myapp.com' }, rateLimit: { max: 20 } },
      database: { host: 'prod-db.internal', pool: { min: 5, max: 50 }, ssl: true },
      cache: { host: 'prod-redis.internal', ttl: 86400 },
      logging: { level: 'warn', format: 'json', destination: 'file' },
    },
  };

  // Deep merge environment settings into the clone
  deepMerge(config, envSettings[environment] || {});
  deepMerge(config, overrides); // then apply any extra overrides

  return config;
}

function deepMerge(target, source) {
  for (const key of Object.keys(source)) {
    if (source[key] && typeof source[key] === 'object' && !Array.isArray(source[key])) {
      target[key] = target[key] || {};
      deepMerge(target[key], source[key]);
    } else {
      target[key] = source[key];
    }
  }
}

// Usage
const devConfig  = createConfig('development');
const stagConfig = createConfig('staging');
const prodConfig = createConfig('production', {
  database: { name: 'myapp_prod' } // extra override
});

console.log(devConfig.logging.level);   // 'debug'
console.log(stagConfig.logging.level);  // 'info'
console.log(prodConfig.logging.level);  // 'warn'
console.log(prodConfig.database.ssl);   // true
console.log(devConfig.database.ssl);    // false — independent! ✅
```

---

### Version 4: Deep Clone Techniques Compared

```javascript
// Clone method comparison — know this for interviews!

const original = {
  name: 'Alice',
  address: { city: 'Mumbai', zip: '400001' },
  tags: ['admin', 'user'],
  createdAt: new Date(),
  greet: function() { return `Hello, ${this.name}`; },
  secret: undefined,
};

// ── Method 1: Spread (Shallow) ──
const spread = { ...original };
// ✅ Fast, simple
// ❌ Shallow — nested objects are references
// ❌ Functions NOT copied (they're references, but same ref)

// ── Method 2: Object.assign (Shallow) ──
const assigned = Object.assign({}, original);
// Same as spread, essentially

// ── Method 3: JSON round-trip ──
const jsonClone = JSON.parse(JSON.stringify(original));
// ✅ Deep copy of plain data
// ❌ Loses: functions, undefined, Date (becomes string), RegExp, Map, Set
console.log(jsonClone.createdAt); // "2024-01-15T..." ← string, not Date!
console.log(jsonClone.greet);     // undefined ← function lost!

// ── Method 4: structuredClone (Best for most cases) ──
const structured = structuredClone(original);
// ✅ Deep copy
// ✅ Handles Date, RegExp, Map, Set, ArrayBuffer
// ❌ Cannot clone functions
// ✅ Available: Node.js 17+, all modern browsers
console.log(structured.createdAt instanceof Date); // true ✅
console.log(structured.address === original.address); // false — truly separate ✅

// ── Method 5: Custom clone() method ──
class Product {
  constructor(id, name, pricing, tags) {
    this.id      = id;
    this.name    = name;
    this.pricing = pricing; // { base, discount, currency }
    this.tags    = tags;
    this.createdAt = new Date();
  }

  clone() {
    return new Product(
      this.id,
      this.name,
      { ...this.pricing },    // shallow copy of pricing (it's one level deep)
      [...this.tags],         // copy array
    );
  }
}

const laptop = new Product('p1', 'Laptop', { base: 80000, discount: 0.1, currency: 'INR' }, ['electronics', 'computers']);
const laptopClone = laptop.clone();

laptopClone.pricing.discount = 0.2;
console.log(laptop.pricing.discount);      // 0.1 — unchanged ✅
console.log(laptopClone instanceof Product); // true ✅ — proper type preserved
```

---

## Real-World Usage in Node.js

```javascript
// 1. Mongoose documents — .toObject() + spread
const userDoc = await User.findById(id);
const plainCopy = { ...userDoc.toObject() }; // detached from Mongoose

// 2. Jest mocking — object spread for test data
const baseUser = { id: 1, name: 'Alice', role: 'user', isActive: true };
const adminUser = { ...baseUser, role: 'admin' };         // override role
const bannedUser = { ...baseUser, isActive: false };      // override isActive

// 3. Redux / state management — immutable updates
const state = { users: [...], config: { theme: 'dark' } };
const newState = { ...state, config: { ...state.config, theme: 'light' } };

// 4. Express request cloning (for middleware testing)
const mockReq = { ...realReq, user: { id: 99, role: 'admin' } };
```

---

## When to Use Prototype

| Situation | Use? | Why |
|-----------|------|-----|
| Object creation is expensive | ✅ Yes | Clone is faster than `new` |
| Need many similar objects with slight variations | ✅ Yes | Clone + tweak |
| Template objects (email, config, game entity) | ✅ Yes | Natural fit |
| Creating test fixtures | ✅ Yes | Base object + overrides |
| Simple objects with no nesting | ❌ No | Spread/assign is enough |

---

## Interview Questions & Answers

**Q: What is the Prototype pattern?**
> Creating new objects by copying (cloning) an existing prototype object instead of using `new ClassName()`. Useful when object creation is expensive or when you need many similar objects.

**Q: What is the difference between shallow copy and deep copy?**
> Shallow copy duplicates only the top-level properties; nested objects/arrays are still shared references. Deep copy recursively duplicates everything, so all levels are independent.

**Q: What are the ways to deep clone in JavaScript/Node.js?**
> `structuredClone()` (Node 17+, best general-purpose), `JSON.parse(JSON.stringify())` (simple but loses Dates/functions), custom `clone()` methods (full control), or libraries like `lodash.cloneDeep`.

**Q: When would Prototype be better than Factory?**
> When the object is expensive to initialize from scratch. Factory always calls `new`; Prototype copies a pre-built instance. Example: a complex game character with pre-loaded assets — clone is 100x faster than re-initializing.

**Q: How does JavaScript's prototype chain relate to the Prototype pattern?**
> They share a name but are different concepts. JS's prototype chain is about inheritance (`Object.prototype`). The GoF Prototype pattern is about cloning instances. In JS, every object has a prototype chain, but the GoF pattern is specifically about `clone()` methods for object creation.

---

## Summary

```
Prototype Pattern
├── What: Create objects by cloning an existing "prototype" object
├── Why: Expensive init, many similar objects, template-based creation
├── How: .clone() method that returns a deep copy with optional tweaks
├── Key concept: Shallow vs Deep copy — always deep copy nested objects
├── Node.js tools: structuredClone(), JSON round-trip, custom clone()
└── Watch out: Shared references in shallow copies cause subtle bugs
```
