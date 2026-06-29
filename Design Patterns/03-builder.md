# 03 — Builder Pattern

> **Category:** GoF Creational  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Think about ordering a custom burger at a restaurant. You don't hand them one giant "burger object" all at once. You say: "I want a whole wheat bun. Add the patty. Add lettuce. No onions. Add extra cheese. Done."

You build it step by step.

The **Builder Pattern** lets you construct complex objects step by step. Instead of a constructor with 10 parameters (which is confusing), you chain methods to set only the properties you care about — and call `.build()` at the end to get the finished object.

**In one sentence:** Builder separates the construction of a complex object from its representation so the same construction process can create different representations.

---

## The Problem It Solves

### The "Telescoping Constructor" problem:

```javascript
// ❌ BAD — What does each argument mean?? You have to count positions.
const user = new User(
  'Alice',           // name
  'alice@email.com', // email
  null,              // phone (not set)
  null,              // address (not set)
  true,              // isAdmin
  false,             // isBanned
  '2024-01-01',      // createdAt
  null,              // avatarUrl (not set)
  'en',              // language
  'USD'              // currency
);
// Confusion! What was the 6th argument again?
```

### With Builder:

```javascript
// ✅ GOOD — clear, readable, only set what you need
const user = new UserBuilder()
  .setName('Alice')
  .setEmail('alice@email.com')
  .setAdmin(true)
  .setLanguage('en')
  .setCurrency('USD')
  .build();
```

---

## Visual Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                         Builder Pattern                          │
│                                                                 │
│  Client                Builder               Product            │
│  ──────                ───────               ───────            │
│                                                                 │
│  UserBuilder()         #name = null                             │
│       │                #email = null         ┌─────────────┐   │
│       │                #role = 'user'        │    User     │   │
│       ▼                #isActive = true      │ ─────────── │   │
│  .setName('Alice') ──> name = 'Alice'        │ name        │   │
│       │                                      │ email       │   │
│       ▼                                      │ role        │   │
│  .setEmail(...) ────> email = '...'          │ isActive    │   │
│       │                                      │ createdAt   │   │
│       ▼                                      │ ...         │   │
│  .setRole('admin')──> role = 'admin'         └─────────────┘   │
│       │                                             ▲           │
│       ▼                                             │           │
│  .build() ──────────────────────────────────────── ┘           │
│                    validates + constructs                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Version 1: User Builder

```javascript
// UserBuilder.js

class User {
  constructor(props) {
    // User is a plain object — built by the builder
    Object.assign(this, props);
    Object.freeze(this); // Immutable once created
  }

  toString() {
    return `User(${this.name}, ${this.email}, role=${this.role})`;
  }
}

class UserBuilder {
  constructor() {
    // Default values
    this.#props = {
      name: null,
      email: null,
      phone: null,
      role: 'user',          // default
      isActive: true,        // default
      isBanned: false,       // default
      language: 'en',        // default
      currency: 'USD',       // default
      avatarUrl: null,
      address: null,
      createdAt: new Date().toISOString(),
      updatedAt: new Date().toISOString(),
    };
  }

  #props;

  // Each setter returns 'this' — enables method chaining
  setName(name) {
    if (!name || typeof name !== 'string') throw new Error('Name must be a non-empty string');
    this.#props.name = name.trim();
    return this; // ← key for chaining
  }

  setEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    if (!emailRegex.test(email)) throw new Error(`Invalid email: ${email}`);
    this.#props.email = email.toLowerCase();
    return this;
  }

  setPhone(phone) {
    this.#props.phone = phone;
    return this;
  }

  setRole(role) {
    const validRoles = ['user', 'admin', 'moderator', 'superadmin'];
    if (!validRoles.includes(role)) {
      throw new Error(`Invalid role: ${role}. Must be one of: ${validRoles.join(', ')}`);
    }
    this.#props.role = role;
    return this;
  }

  setActive(isActive) {
    this.#props.isActive = Boolean(isActive);
    return this;
  }

  setBanned(isBanned) {
    this.#props.isBanned = Boolean(isBanned);
    return this;
  }

  setLanguage(lang) {
    this.#props.language = lang;
    return this;
  }

  setCurrency(currency) {
    this.#props.currency = currency;
    return this;
  }

  setAddress(address) {
    this.#props.address = address;
    return this;
  }

  setAvatarUrl(url) {
    this.#props.avatarUrl = url;
    return this;
  }

  // Final validation before building
  #validate() {
    const errors = [];
    if (!this.#props.name)  errors.push('name is required');
    if (!this.#props.email) errors.push('email is required');
    if (this.#props.isBanned && this.#props.role === 'admin') {
      errors.push('Admins cannot be banned');
    }
    if (errors.length > 0) {
      throw new Error(`Cannot build User: ${errors.join(', ')}`);
    }
  }

  build() {
    this.#validate();
    return new User({ ...this.#props });
  }
}

// --- Usage ---

const alice = new UserBuilder()
  .setName('Alice Johnson')
  .setEmail('alice@example.com')
  .setPhone('+919876543210')
  .setRole('admin')
  .setLanguage('en')
  .setCurrency('INR')
  .build();

console.log(alice);
// User { name: 'Alice Johnson', email: 'alice@example.com', role: 'admin', ... }

// Minimal user (uses defaults)
const bob = new UserBuilder()
  .setName('Bob')
  .setEmail('bob@example.com')
  .build();

console.log(bob.role);     // 'user' (default)
console.log(bob.isActive); // true (default)
console.log(bob.language); // 'en' (default)

// Validation catches errors
try {
  const badUser = new UserBuilder()
    .setName('Charlie')
    // forgot email!
    .build();
} catch (err) {
  console.error(err.message); // "Cannot build User: email is required"
}
```

---

### Version 2: Query Builder (Super Common in Node.js!)

This is THE most common real-world use of Builder in backend dev:

```javascript
// QueryBuilder.js
class QueryBuilder {
  constructor(tableName) {
    this.#table = tableName;
    this.#conditions = [];
    this.#columns = ['*'];
    this.#joins = [];
    this.#orderBys = [];
    this.#params = [];
    this.#paramCount = 0;
    this.#limitVal = null;
    this.#offsetVal = null;
    this.#operation = 'SELECT';
  }

  #table;
  #conditions;
  #columns;
  #joins;
  #orderBys;
  #params;
  #paramCount;
  #limitVal;
  #offsetVal;
  #operation;

  select(...columns) {
    this.#columns = columns.length > 0 ? columns : ['*'];
    return this;
  }

  where(column, operator, value) {
    this.#paramCount++;
    this.#conditions.push(`${column} ${operator} $${this.#paramCount}`);
    this.#params.push(value);
    return this;
  }

  whereIn(column, values) {
    const placeholders = values.map(() => {
      this.#paramCount++;
      return `$${this.#paramCount}`;
    }).join(', ');
    this.#conditions.push(`${column} IN (${placeholders})`);
    this.#params.push(...values);
    return this;
  }

  whereNull(column) {
    this.#conditions.push(`${column} IS NULL`);
    return this;
  }

  whereNotNull(column) {
    this.#conditions.push(`${column} IS NOT NULL`);
    return this;
  }

  join(table, localKey, operator, foreignKey) {
    this.#joins.push(`JOIN ${table} ON ${localKey} ${operator} ${foreignKey}`);
    return this;
  }

  leftJoin(table, localKey, operator, foreignKey) {
    this.#joins.push(`LEFT JOIN ${table} ON ${localKey} ${operator} ${foreignKey}`);
    return this;
  }

  orderBy(column, direction = 'ASC') {
    this.#orderBys.push(`${column} ${direction.toUpperCase()}`);
    return this;
  }

  limit(n) {
    this.#limitVal = parseInt(n);
    return this;
  }

  offset(n) {
    this.#offsetVal = parseInt(n);
    return this;
  }

  // Pagination shorthand
  paginate(page, perPage = 10) {
    this.#limitVal = perPage;
    this.#offsetVal = (page - 1) * perPage;
    return this;
  }

  build() {
    const parts = [
      `SELECT ${this.#columns.join(', ')}`,
      `FROM ${this.#table}`,
    ];

    if (this.#joins.length) parts.push(this.#joins.join('\n'));
    if (this.#conditions.length) parts.push(`WHERE ${this.#conditions.join(' AND ')}`);
    if (this.#orderBys.length) parts.push(`ORDER BY ${this.#orderBys.join(', ')}`);
    if (this.#limitVal !== null) parts.push(`LIMIT ${this.#limitVal}`);
    if (this.#offsetVal !== null) parts.push(`OFFSET ${this.#offsetVal}`);

    return {
      sql: parts.join('\n'),
      params: this.#params,
    };
  }
}

// --- Usage ---

// Simple query
const { sql, params } = new QueryBuilder('users')
  .select('id', 'name', 'email')
  .where('is_active', '=', true)
  .where('role', '=', 'admin')
  .orderBy('created_at', 'DESC')
  .limit(10)
  .build();

console.log(sql);
// SELECT id, name, email
// FROM users
// WHERE is_active = $1 AND role = $2
// ORDER BY created_at DESC
// LIMIT 10

console.log(params); // [true, 'admin']

// Complex query with joins and pagination
const { sql: sql2, params: params2 } = new QueryBuilder('orders')
  .select('orders.id', 'orders.total', 'users.name AS user_name', 'users.email')
  .leftJoin('users', 'orders.user_id', '=', 'users.id')
  .where('orders.status', '=', 'pending')
  .where('orders.total', '>', 100)
  .whereNotNull('orders.shipped_at')
  .whereIn('orders.payment_method', ['card', 'upi'])
  .orderBy('orders.created_at', 'DESC')
  .paginate(2, 20) // page 2, 20 per page
  .build();

console.log(sql2);
// SELECT orders.id, orders.total, users.name AS user_name, users.email
// FROM orders
// LEFT JOIN users ON orders.user_id = users.id
// WHERE orders.status = $1 AND orders.total > $2 AND ...
// ORDER BY orders.created_at DESC
// LIMIT 20
// OFFSET 20

// Use with actual DB
const { Pool } = require('pg');
const pool = new Pool({ /* config */ });

async function findPendingOrders(minTotal, page) {
  const { sql, params } = new QueryBuilder('orders')
    .select('id', 'total', 'user_id', 'status')
    .where('status', '=', 'pending')
    .where('total', '>=', minTotal)
    .orderBy('created_at', 'DESC')
    .paginate(page, 10)
    .build();

  const result = await pool.query(sql, params);
  return result.rows;
}
```

---

### Version 3: HTTP Request Builder

```javascript
// HttpRequestBuilder.js
const https = require('https');

class HttpRequestBuilder {
  constructor() {
    this.#config = {
      method: 'GET',
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
      timeout: 30000,
      retries: 0,
      params: {},
      body: null,
      url: null,
    };
  }

  #config;

  url(url) {
    this.#config.url = url;
    return this;
  }

  method(method) {
    this.#config.method = method.toUpperCase();
    return this;
  }

  get(url)    { return this.url(url).method('GET'); }
  post(url)   { return this.url(url).method('POST'); }
  put(url)    { return this.url(url).method('PUT'); }
  delete(url) { return this.url(url).method('DELETE'); }
  patch(url)  { return this.url(url).method('PATCH'); }

  header(key, value) {
    this.#config.headers[key] = value;
    return this;
  }

  bearerToken(token) {
    return this.header('Authorization', `Bearer ${token}`);
  }

  apiKey(key, headerName = 'X-API-Key') {
    return this.header(headerName, key);
  }

  body(data) {
    this.#config.body = typeof data === 'string' ? data : JSON.stringify(data);
    return this;
  }

  queryParam(key, value) {
    this.#config.params[key] = value;
    return this;
  }

  timeout(ms) {
    this.#config.timeout = ms;
    return this;
  }

  retry(times) {
    this.#config.retries = times;
    return this;
  }

  build() {
    if (!this.#config.url) throw new Error('URL is required');
    return { ...this.#config };
  }
}

// Usage
const requestConfig = new HttpRequestBuilder()
  .post('https://api.payment.com/charge')
  .bearerToken(process.env.PAYMENT_API_TOKEN)
  .header('X-Idempotency-Key', 'unique-request-id-123')
  .body({ amount: 999, currency: 'INR', customerId: 'cust_abc' })
  .timeout(10000)
  .retry(3)
  .build();

console.log(requestConfig);
// {
//   method: 'POST',
//   url: 'https://api.payment.com/charge',
//   headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer ...', 'X-Idempotency-Key': '...' },
//   body: '{"amount":999,...}',
//   timeout: 10000,
//   retries: 3
// }
```

---

### Version 4: Email Builder (Real-World with Nodemailer)

```javascript
// EmailBuilder.js
const nodemailer = require('nodemailer');

class Email {
  constructor(props) {
    this.from    = props.from;
    this.to      = props.to;
    this.cc      = props.cc;
    this.bcc     = props.bcc;
    this.subject = props.subject;
    this.html    = props.html;
    this.text    = props.text;
    this.attachments = props.attachments;
  }
}

class EmailBuilder {
  #props = {
    from: 'noreply@myapp.com',
    to: [],
    cc: [],
    bcc: [],
    subject: '',
    html: '',
    text: '',
    attachments: [],
  };

  from(address) {
    this.#props.from = address;
    return this;
  }

  to(...addresses) {
    this.#props.to.push(...addresses.flat());
    return this;
  }

  cc(...addresses) {
    this.#props.cc.push(...addresses.flat());
    return this;
  }

  bcc(...addresses) {
    this.#props.bcc.push(...addresses.flat());
    return this;
  }

  subject(subject) {
    this.#props.subject = subject;
    return this;
  }

  htmlBody(html) {
    this.#props.html = html;
    return this;
  }

  textBody(text) {
    this.#props.text = text;
    return this;
  }

  // Use a template
  template(name, variables = {}) {
    const templates = {
      welcome: (v) => `
        <h1>Welcome, ${v.name}!</h1>
        <p>Thanks for joining us. <a href="${v.link}">Verify your email</a></p>
      `,
      passwordReset: (v) => `
        <h1>Reset your password</h1>
        <p>Click <a href="${v.link}">here</a> to reset. Link expires in 1 hour.</p>
      `,
      orderConfirm: (v) => `
        <h1>Order Confirmed!</h1>
        <p>Order #${v.orderId} total: ₹${v.total}</p>
      `,
    };

    const templateFn = templates[name];
    if (!templateFn) throw new Error(`Unknown template: ${name}`);
    this.#props.html = templateFn(variables);
    return this;
  }

  attachment(filename, content, mimeType = 'application/octet-stream') {
    this.#props.attachments.push({ filename, content, contentType: mimeType });
    return this;
  }

  build() {
    if (!this.#props.to.length) throw new Error('At least one recipient required');
    if (!this.#props.subject) throw new Error('Subject is required');
    if (!this.#props.html && !this.#props.text) throw new Error('Email body required');
    return new Email(this.#props);
  }
}

// Usage
const welcomeEmail = new EmailBuilder()
  .to('alice@example.com')
  .subject('Welcome to MyApp!')
  .template('welcome', { name: 'Alice', link: 'https://myapp.com/verify/abc123' })
  .build();

const orderEmail = new EmailBuilder()
  .from('orders@myapp.com')
  .to('alice@example.com')
  .cc('support@myapp.com')
  .subject('Order #1234 Confirmed')
  .template('orderConfirm', { orderId: '1234', total: '1,299' })
  .attachment('invoice.pdf', invoicePdfBuffer, 'application/pdf')
  .build();

// Send
const transporter = nodemailer.createTransport({ /* smtp config */ });
await transporter.sendMail(welcomeEmail);
await transporter.sendMail(orderEmail);
```

---

## When to Use Builder

| Situation | Use Builder? | Why |
|-----------|-------------|-----|
| Object with 4+ optional parameters | ✅ Yes | Avoid telescoping constructors |
| Need to validate before creating object | ✅ Yes | .build() is the validation gate |
| Building SQL queries dynamically | ✅ Yes | Classic use case |
| Constructing HTTP requests | ✅ Yes | Readable configuration |
| Simple 2-field object | ❌ No | Overkill |
| Object with all required fields | ❌ Maybe | Constructor is fine |

---

## Interview Questions & Answers

**Q: What is the Builder pattern?**
> Separates the construction of a complex object into discrete steps. A builder class accumulates configuration through method chaining, then a `build()` method creates the final validated object.

**Q: What problem does Builder solve that a constructor doesn't?**
> The "telescoping constructor" problem — when a constructor has many optional parameters, callers must pass nulls for unused ones and can't tell what each position means. Builder uses named methods, is readable, and allows validation at `build()` time.

**Q: Why does each setter return `this`?**
> To enable method chaining. `return this` means each setter returns the builder itself, so you can immediately call the next setter on the result. This is also called a "fluent interface."

**Q: What's the difference between Builder and Factory?**
> Factory creates the object in ONE step and decides WHICH type to create. Builder creates the object in MANY steps and focuses on HOW to construct a complex object. Use Factory when you need to choose among types; use Builder when construction is complex.

**Q: Where is Builder used in real Node.js libraries?**
> Knex.js query builder (`knex('users').where('id', 1).select('name')`), Mongoose `Query` object, Jest `expect().toBe().not.toThrow()`, Supertest for HTTP testing, email template builders.

---

## Summary

```
Builder Pattern
├── What: Build complex objects step by step via method chaining
├── Why: Avoid messy constructors, enable validation, improve readability
├── How: Each setter returns 'this'; .build() validates and creates the object
├── Node.js: Query builders, HTTP request builders, email builders
└── Key: return this in every setter = fluent interface / method chaining
```
