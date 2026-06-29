# Day 1 — Node.js & TypeScript Foundations
### NestJS Interview Prep | 3-Year Backend Engineer Level

> **Goal for today:** Understand the engine that runs your NestJS app — Node.js internals, async patterns, and TypeScript features that NestJS uses heavily. An interviewer WILL ask these. You must explain WHY, not just WHAT.

---

## Table of Contents

1. [What is Node.js — Really?](#1-what-is-nodejs--really)
2. [The Event Loop — The Heart of Node.js](#2-the-event-loop--the-heart-of-nodejs)
3. [Callbacks — The Original Async Pattern](#3-callbacks--the-original-async-pattern)
4. [Promises — Solving Callback Hell](#4-promises--solving-callback-hell)
5. [Async/Await — Promises in Disguise](#5-asyncawait--promises-in-disguise)
6. [TypeScript — Why NestJS Forces It](#6-typescript--why-nestjs-forces-it)
7. [Types & Interfaces](#7-types--interfaces)
8. [Generics — Reusable Type-Safe Code](#8-generics--reusable-type-safe-code)
9. [Decorators — The Magic Behind NestJS](#9-decorators--the-magic-behind-nestjs)
10. [ES Modules vs CommonJS](#10-es-modules-vs-commonjs)
11. [tsconfig.json — TypeScript Compiler Settings](#11-tsconfigjson--typescript-compiler-settings)
12. [NPM & package.json — Project Anatomy](#12-npm--packagejson--project-anatomy)
13. [Interview Questions & Answers](#13-interview-questions--answers)

---

## 1. What is Node.js — Really?

### Simple explanation
Node.js is a **runtime environment** that lets you run JavaScript outside the browser — on your server, your machine, or a Docker container.

### Technical explanation
Before Node.js (2009), JavaScript only ran inside web browsers. Ryan Dahl took Google Chrome's **V8 JavaScript engine** (the part of Chrome that executes JS) and embedded it into a C++ program called Node.js. He also added:

- **libuv** — a C library that gives Node.js access to the operating system's async I/O (reading files, making network calls, etc.)
- **Core modules** — `http`, `fs`, `path`, `crypto`, `stream`, etc.

This means Node.js is NOT just JavaScript. Under the hood it is:

```
Your JS Code
    ↓
V8 Engine (compiles JS to machine code)
    ↓
Node.js C++ Bindings
    ↓
libuv (handles OS-level async I/O, thread pool)
    ↓
Operating System (Linux/macOS/Windows)
```

### Why does this matter for interviews?
When an interviewer asks "why is Node.js fast?", the answer is NOT "because JavaScript is fast." The answer is:

> "Node.js is fast for I/O-bound tasks because it uses a **non-blocking, event-driven architecture** powered by libuv. Instead of waiting for a database query or HTTP response, Node.js registers a callback and continues processing other requests. The OS notifies Node.js when the I/O is complete."

### What Node.js is good at vs bad at

| Good at | Bad at |
|---|---|
| REST APIs with many concurrent users | CPU-heavy tasks (video encoding, ML) |
| Real-time apps (chat, live updates) | Heavy number crunching |
| Microservices | Tasks needing true parallelism |
| I/O-intensive operations | Blocking computation on main thread |

### Node.js single-threaded — the truth
Node.js runs your JavaScript on **one thread** (the main thread / event loop thread). But libuv uses a **thread pool** (default 4 threads) for operations like file I/O, DNS lookups, crypto. So Node.js is single-threaded for JS execution but multi-threaded under the hood for I/O.

```
Main Thread (Your JS)          libuv Thread Pool (C++)
─────────────────────          ──────────────────────────
Runs event loop                Thread 1: Reading file A
Executes callbacks             Thread 2: DNS lookup
Runs your NestJS code          Thread 3: crypto.pbkdf2()
                               Thread 4: File write
```

---

## 2. The Event Loop — The Heart of Node.js

### Simple explanation
The event loop is a loop that constantly checks: "Is there any work to do?" If there is, it does it. If not, it waits.

### Technical explanation — the actual phases

The event loop has **6 phases** that run in order, over and over:

```
   ┌──────────────────────────────┐
┌─>│           timers             │  ← setTimeout, setInterval callbacks
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │     pending callbacks        │  ← I/O callbacks deferred to next loop
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │       idle, prepare          │  ← internal use only
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │           poll               │  ← retrieve new I/O events (MOST IMPORTANT)
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
│  │           check              │  ← setImmediate callbacks
│  └──────────────┬───────────────┘
│  ┌──────────────▼───────────────┐
└──│      close callbacks         │  ← e.g., socket.on('close')
   └──────────────────────────────┘
```

Between EVERY phase, Node.js drains two special queues:
- `process.nextTick()` — runs before anything else in the next iteration
- `Promise` microtasks — `.then()` / `.catch()` / `await` continuations

### Priority order (most important to least)

```
process.nextTick()   ← runs first, before everything
Promise microtasks   ← .then(), await continuations
setTimeout(fn, 0)    ← after microtasks
setImmediate()       ← in check phase
```

### Code example — event loop in action

```javascript
// Run this in Node.js to understand the execution order

console.log('1: Script starts');  // Runs synchronously first

setTimeout(() => {
  console.log('5: setTimeout (timers phase)');
}, 0);  // Even with 0ms, this waits until later

setImmediate(() => {
  console.log('6: setImmediate (check phase)');
});

Promise.resolve().then(() => {
  console.log('3: Promise .then() — microtask queue');
});

process.nextTick(() => {
  console.log('2: process.nextTick — runs before everything else');
});

console.log('4: Script ends (still synchronous)');

/*
OUTPUT ORDER:
1: Script starts
4: Script ends (still synchronous)
2: process.nextTick — runs before everything else
3: Promise .then() — microtask queue
5: setTimeout (timers phase)
6: setImmediate (check phase)

WHY THIS ORDER?
- 1 and 4 are synchronous — run immediately on call stack
- 2: nextTick has HIGHEST priority, drains before any event loop phase
- 3: Promise microtasks drain next
- 5: setTimeout fires in timers phase
- 6: setImmediate fires in check phase (after poll)
*/
```

### The poll phase — where Node.js spends most of its time

The poll phase is where Node.js **waits** for I/O events. When you make a database call:

```javascript
// This is what conceptually happens:
db.query('SELECT * FROM users', (err, result) => {
  // This callback sits in the poll phase waiting list
  // Node.js is NOT blocked here — it can handle other requests
  // When DB responds, OS notifies libuv → callback goes to poll queue
  console.log(result);
});
```

### Why blocking the event loop is catastrophic

```javascript
// ❌ NEVER DO THIS in a Node.js server
app.get('/users', (req, res) => {
  // This BLOCKS the ENTIRE event loop for 5 seconds
  // During this time, ALL other requests are frozen
  const start = Date.now();
  while (Date.now() - start < 5000) {}  // Busy-wait — criminal in Node.js
  res.json({ users: [] });
});

// ✅ CORRECT — non-blocking async operation
app.get('/users', async (req, res) => {
  // Node.js registers this DB call and moves on to other requests
  // When DB responds, execution resumes here
  const users = await db.query('SELECT * FROM users');
  res.json({ users });
});
```

### Interview question this answers
**"What happens if you run a CPU-intensive operation in Node.js?"**

> "Since Node.js is single-threaded, a CPU-intensive synchronous operation blocks the event loop. This means no other requests can be processed until the operation completes. Solutions include: offloading to a Worker Thread, using a child process, or moving CPU work to a separate microservice."

---

## 3. Callbacks — The Original Async Pattern

### What and why
Before Promises existed (pre-2015), the ONLY way to handle async operations in Node.js was callbacks — functions passed as arguments that get called when the async work is done.

```javascript
// The Node.js callback convention: Error-First Callback (also called "errback")
// RULE: First argument is ALWAYS the error, second is the result
// This is a Node.js STANDARD — not optional

const fs = require('fs');

// Reading a file — the classic Node.js callback pattern
fs.readFile('/path/to/file.txt', 'utf8', (err, data) => {
  // err: null if success, Error object if something went wrong
  // data: the file content if success, undefined if error
  
  if (err) {
    // Always check error FIRST
    console.error('Failed to read file:', err.message);
    return;  // ← IMPORTANT: return early to prevent "success" code running
  }
  
  // Only here if there's no error
  console.log('File content:', data);
});

console.log('This runs BEFORE the file is read');
// The file reading is async — Node.js doesn't wait here
```

### Callback Hell — the problem callbacks created

```javascript
// Real-world scenario: Get user → get their orders → get order items
// With nested callbacks, this becomes unmaintainable:

getUserById(userId, (err, user) => {
  if (err) return handleError(err);
  
  getOrdersByUserId(user.id, (err, orders) => {
    if (err) return handleError(err);
    
    getItemsByOrderId(orders[0].id, (err, items) => {
      if (err) return handleError(err);
      
      getProductDetails(items[0].productId, (err, product) => {
        if (err) return handleError(err);
        
        // We're 4 levels deep — this is "callback hell" / "pyramid of doom"
        // Problems:
        // 1. Hard to read and reason about
        // 2. Error handling duplicated at every level
        // 3. Difficult to run things in parallel
        // 4. Almost impossible to test properly
        console.log(product);
      });
    });
  });
});
```

### Why you still need to understand callbacks in 2024
- Node.js core modules (`fs`, `http`, `crypto`) still use callbacks
- Many npm packages use callbacks internally
- `util.promisify()` converts callback functions to Promises
- Understanding callbacks helps debug async issues in production

```javascript
// util.promisify — converting callback-based to Promise-based
const { promisify } = require('util');
const fs = require('fs');

// fs.readFile uses callbacks — let's convert it
const readFileAsync = promisify(fs.readFile);

// Now we can use it with async/await
async function readConfig() {
  try {
    const data = await readFileAsync('./config.json', 'utf8');
    return JSON.parse(data);
  } catch (err) {
    console.error('Config read failed:', err);
  }
}
```

---

## 4. Promises — Solving Callback Hell

### What is a Promise?
A Promise is an object that represents the **eventual result** of an async operation. It's either:
- **Pending** — the async work is still running
- **Fulfilled** — the async work completed successfully (has a value)
- **Rejected** — the async work failed (has an error/reason)

```
       Promise
          │
    ┌─────┴─────┐
  Pending    Settled (final state)
              │
       ┌──────┴──────┐
   Fulfilled      Rejected
   (success)       (error)
```

### Creating and using Promises

```javascript
// Creating a Promise from scratch
const myPromise = new Promise((resolve, reject) => {
  // This function runs IMMEDIATELY (synchronously)
  
  const success = Math.random() > 0.5;  // 50% chance
  
  if (success) {
    resolve('Operation succeeded!');  // Fulfills the promise
  } else {
    reject(new Error('Operation failed!'));  // Rejects the promise
  }
});

// Consuming the Promise
myPromise
  .then((result) => {
    // Runs when promise fulfills
    console.log('Success:', result);
  })
  .catch((error) => {
    // Runs when promise rejects
    console.error('Error:', error.message);
  })
  .finally(() => {
    // Runs ALWAYS — success or failure
    // Use for cleanup (closing DB connections, hiding loading spinners)
    console.log('Done — cleanup here');
  });
```

### Promise chaining — the real power

```javascript
// The SAME scenario as callback hell — but clean and readable
// Each .then() returns a new Promise, allowing chaining

getUserById(userId)
  .then((user) => getOrdersByUserId(user.id))    // Returns a new Promise
  .then((orders) => getItemsByOrderId(orders[0].id))  // Chained
  .then((items) => getProductDetails(items[0].productId))  // Chained
  .then((product) => {
    console.log(product);  // Final result
  })
  .catch((error) => {
    // ONE error handler catches errors from ANY step above
    // This is the key advantage over callbacks
    console.error('Something failed:', error);
  });
```

### Promise combinators — running things in parallel

```javascript
// Promise.all — run multiple promises in PARALLEL, wait for ALL
// If ANY rejects → entire Promise.all rejects

const [users, products, orders] = await Promise.all([
  db.getUsers(),      // All three start at the SAME TIME
  db.getProducts(),   // Not waiting for each other
  db.getOrders(),     // Total time ≈ slowest of the three
]);

// Promise.allSettled — like Promise.all but NEVER rejects
// You get results AND errors for each promise
const results = await Promise.allSettled([
  fetchFromServiceA(),
  fetchFromServiceB(),  // If this fails, you still get result from A
]);

results.forEach((result) => {
  if (result.status === 'fulfilled') {
    console.log('Success:', result.value);
  } else {
    console.log('Failed:', result.reason);
  }
});

// Promise.race — resolves/rejects as soon as the FIRST promise settles
// Use case: implement timeout
const withTimeout = (promise, ms) => Promise.race([
  promise,
  new Promise((_, reject) =>
    setTimeout(() => reject(new Error('Timeout')), ms)
  ),
]);

// Promise.any — resolves with the FIRST fulfilled (ignores rejections)
// Use case: try multiple mirrors, use whichever responds first
const data = await Promise.any([
  fetchFromRegion('us-east'),
  fetchFromRegion('eu-west'),
  fetchFromRegion('ap-south'),
]);
```

---

## 5. Async/Await — Promises in Disguise

### What is async/await?
`async/await` is **syntactic sugar** over Promises. It makes async code look and behave like synchronous code. Under the hood, it compiles to Promise chains.

**CRITICAL INTERVIEW POINT:** `async/await` does NOT change how Node.js event loop works. It's just cleaner syntax. An `await` suspends the current function but does NOT block the event loop — other code can still run.

```javascript
// async keyword: marks a function as async
// - The function ALWAYS returns a Promise, even if you return a plain value
// - You can use await inside it

async function fetchUserData(userId) {
  // await keyword: pauses THIS function, not the whole program
  // - Can only be used inside async functions
  // - Unwraps the Promise — gives you the resolved value directly
  // - If the Promise rejects, it throws an error (catchable with try/catch)
  
  try {
    // Without await this would return a Promise<User>, not a User
    const user = await db.findUser(userId);     // Pauses here until DB responds
    const orders = await db.findOrders(user.id); // Pauses here until DB responds
    
    return { user, orders };  // async function wraps this in Promise.resolve()
  } catch (error) {
    // Catches both: rejected Promises AND thrown errors
    throw new Error(`Failed to fetch user data: ${error.message}`);
  }
}

// Calling an async function — it returns a Promise
const result = await fetchUserData(123);
// OR
fetchUserData(123).then(result => console.log(result));
```

### The compiled output — what TypeScript actually generates

```typescript
// What you write:
async function getUser(id: number): Promise<User> {
  const user = await userRepository.findOne(id);
  return user;
}

// What TypeScript compiles to (conceptually):
function getUser(id) {
  return __awaiter(this, void 0, void 0, function* () {
    const user = yield userRepository.findOne(id);
    return user;
  });
}
// It uses generators (function*) internally — that's the real mechanism
```

### Common async/await mistakes

```javascript
// ❌ MISTAKE 1: Unnecessary sequential await
// These two DB calls have no dependency — don't make them sequential!
async function badExample() {
  const users = await db.getUsers();    // Wait for this...
  const orders = await db.getOrders();  // THEN wait for this
  // Total time = users_time + orders_time
}

// ✅ FIX: Run independent async operations in parallel
async function goodExample() {
  const [users, orders] = await Promise.all([
    db.getUsers(),    // Both start at the same time
    db.getOrders(),   // Total time = max(users_time, orders_time)
  ]);
}

// ❌ MISTAKE 2: Swallowing errors silently
async function badErrorHandling() {
  const data = await fetchData().catch(() => null);  // null on error — but caller doesn't know
  data.items.forEach(/* ... */);  // 💥 TypeError: Cannot read property 'items' of null
}

// ✅ FIX: Either handle the error properly or let it propagate
async function goodErrorHandling() {
  const data = await fetchData();  // Let it throw if it fails — caller can handle it
  return data.items;
}

// ❌ MISTAKE 3: await in a loop (serial execution)
async function badLoop(ids) {
  for (const id of ids) {
    await processItem(id);  // Each item waits for the previous one
  }
}

// ✅ FIX: Process all items in parallel
async function goodLoop(ids) {
  await Promise.all(ids.map(id => processItem(id)));
}
// OR: Process in batches to avoid overwhelming the DB
async function batchedLoop(ids, batchSize = 10) {
  for (let i = 0; i < ids.length; i += batchSize) {
    const batch = ids.slice(i, i + batchSize);
    await Promise.all(batch.map(id => processItem(id)));
  }
}
```

### async/await in NestJS — what you'll see daily

```typescript
// In NestJS, almost every service method is async
// Here's a typical pattern you'll write 100 times:

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  // async service method — returns a Promise
  async findOne(id: number): Promise<User> {
    // await unwraps the Promise from repository.findOne()
    const user = await this.userRepository.findOne({ where: { id } });
    
    if (!user) {
      // NestJS has built-in HTTP exceptions
      throw new NotFoundException(`User #${id} not found`);
    }
    
    return user;  // NestJS controller awaits this automatically
  }

  async create(createUserDto: CreateUserDto): Promise<User> {
    // TypeORM operations are async
    const user = this.userRepository.create(createUserDto);
    return await this.userRepository.save(user);
  }
}
```

---

## 6. TypeScript — Why NestJS Forces It

### What is TypeScript?
TypeScript is a **superset of JavaScript** created by Microsoft. It adds a **type system** on top of JavaScript and compiles back to plain JavaScript.

```
TypeScript (.ts files)
        ↓  tsc (TypeScript Compiler)
JavaScript (.js files)
        ↓  Node.js / V8
Machine Code
```

### Why NestJS is built entirely in TypeScript

NestJS's entire architecture depends on TypeScript features that don't exist in plain JavaScript:

1. **Decorators** (`@Controller()`, `@Injectable()`, `@Get()`) — require TypeScript
2. **Metadata reflection** (`Reflect.metadata`) — used for dependency injection
3. **Type inference** — NestJS reads parameter types to inject dependencies
4. **Interfaces** — define contracts between layers
5. **Generics** — Repository<User>, Observable<T>, etc.

```typescript
// NestJS reads the TYPE of constructor parameters to inject dependencies
// This is ONLY possible because TypeScript emits type metadata
@Injectable()
export class UsersService {
  constructor(
    // NestJS sees "Repository<User>" at runtime via Reflect.metadata
    // It knows to inject a TypeORM repository configured for User entity
    private readonly userRepository: Repository<User>,
    
    // NestJS sees "MailService" and injects the registered provider
    private readonly mailService: MailService,
  ) {}
}
// Without TypeScript, this dependency injection system cannot work
```

---

## 7. Types & Interfaces

### Primitive types

```typescript
// TypeScript's basic types
let name: string = 'John';
let age: number = 25;
let isActive: boolean = true;
let data: null = null;
let value: undefined = undefined;
let anything: any = 'whatever';  // ← Avoid this! Defeats the purpose of TypeScript

// Arrays
let names: string[] = ['Alice', 'Bob'];
let ids: Array<number> = [1, 2, 3];  // Generic syntax — same thing

// Tuple — fixed length, fixed types at each position
let point: [number, number] = [10, 20];
let entry: [string, number] = ['age', 25];

// Union types — can be one of multiple types
let id: string | number;
id = 'abc-123';  // ✅
id = 456;        // ✅
id = true;       // ❌ TypeScript error

// Literal types — only specific values allowed
let direction: 'north' | 'south' | 'east' | 'west';
direction = 'north';  // ✅
direction = 'up';     // ❌ TypeScript error

// Type aliases — give a name to a type
type UserId = string | number;
type Direction = 'north' | 'south' | 'east' | 'west';
type Status = 'active' | 'inactive' | 'suspended';
```

### Interfaces — defining object shapes

```typescript
// Interface defines the SHAPE an object must have
// It's a CONTRACT — "any object claiming to be a User must have these properties"

interface User {
  id: number;
  name: string;
  email: string;
  age?: number;          // ? makes it optional
  readonly createdAt: Date;  // readonly — can't be changed after creation
}

// Using the interface
const user: User = {
  id: 1,
  name: 'Alice',
  email: 'alice@example.com',
  // age is optional, so we can omit it
  createdAt: new Date(),
};

// user.createdAt = new Date();  // ❌ Error: cannot assign to readonly property

// Interfaces can extend other interfaces (inheritance)
interface AdminUser extends User {
  role: 'admin' | 'super-admin';
  permissions: string[];
}

// Interfaces can describe functions too
interface Comparator<T> {
  compare(a: T, b: T): number;  // Returns negative, 0, or positive
}

// Interface vs Type Alias — the real differences
// Interface: can be extended (merged), better for object shapes
// Type: can use union/intersection, better for complex type operations

// Interface merging — you can declare the same interface twice
interface Request {
  userId: number;
}
interface Request {
  sessionId: string;
}
// Both merge into: { userId: number; sessionId: string }
// This is how @types/express works — adds custom properties to Express's Request

// Type alias CANNOT be merged — redeclaring is an error
```

### In NestJS context

```typescript
// Interfaces are used everywhere in NestJS

// 1. Service interfaces — define contracts
interface IUsersService {
  findAll(): Promise<User[]>;
  findOne(id: number): Promise<User>;
  create(dto: CreateUserDto): Promise<User>;
  update(id: number, dto: UpdateUserDto): Promise<User>;
  remove(id: number): Promise<void>;
}

// 2. Repository interfaces (TypeORM implements these)
// 3. Configuration interfaces
interface DatabaseConfig {
  host: string;
  port: number;
  username: string;
  password: string;
  database: string;
}

// 4. Response shapes
interface ApiResponse<T> {
  data: T;
  message: string;
  statusCode: number;
  timestamp: string;
}
```

---

## 8. Generics — Reusable Type-Safe Code

### What are generics?
Generics let you write code that works with **any type** while still being **type-safe**. Think of them as type parameters — like function parameters but for types.

### Simple explanation
Without generics, you'd write `getUser()`, `getProduct()`, `getOrder()` separately. With generics, you write `getOne<T>()` once and it works for any type.

```typescript
// The problem generics solve:

// Without generics — you'd need separate functions for each type
function getFirstUser(items: User[]): User {
  return items[0];
}
function getFirstProduct(items: Product[]): Product {
  return items[0];
}
// Repeated code!

// With generics — ONE function works for any type
// T is a "type variable" — it's a placeholder that gets filled in when called
function getFirst<T>(items: T[]): T {
  return items[0];
}

// TypeScript INFERS the type automatically
const user = getFirst(users);      // TypeScript knows this is User
const product = getFirst(products); // TypeScript knows this is Product
const name = getFirst(['a', 'b']);  // TypeScript knows this is string
```

### Generic constraints — limiting what T can be

```typescript
// Without constraint — T can be ANYTHING (even number, which has no .id)
function findById<T>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);  // ❌ Error: T might not have .id
}

// With constraint — T must have at least an 'id' property
interface HasId {
  id: number;
}

function findById<T extends HasId>(items: T[], id: number): T | undefined {
  return items.find(item => item.id === id);  // ✅ TypeScript knows T has .id
}

// Works with any type that has an id:
findById(users, 1);    // User has id ✅
findById(products, 5); // Product has id ✅
```

### Generics in NestJS — you'll see these every day

```typescript
// 1. Repository<T> — TypeORM's generic repository
// T is the entity type — Repository<User> works with User entities
@InjectRepository(User)
private userRepository: Repository<User>;

// The Repository class is defined like:
class Repository<T> {
  find(options?: FindManyOptions<T>): Promise<T[]>;
  findOne(options: FindOneOptions<T>): Promise<T | null>;
  save(entity: T): Promise<T>;
  // ... all typed with T
}

// 2. Generic API response wrapper
interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  lastPage: number;
}

async function getPaginated<T>(
  repository: Repository<T>,
  page: number,
  limit: number,
): Promise<PaginatedResponse<T>> {
  const [data, total] = await repository.findAndCount({
    skip: (page - 1) * limit,
    take: limit,
  });
  
  return {
    data,
    total,
    page,
    lastPage: Math.ceil(total / limit),
  };
}

// 3. Observable<T> — used in NestJS microservices (RxJS)
// 4. Promise<T> — every async operation
// 5. Type<T> — NestJS uses this for class references
```

---

## 9. Decorators — The Magic Behind NestJS

### What are decorators?
Decorators are **functions that wrap other functions, classes, or properties** to add behavior without modifying the original code. They look like `@Something` and are placed above what they decorate.

This is the **most important TypeScript feature for NestJS**. Every single NestJS feature uses decorators.

### How decorators actually work

```typescript
// A decorator is just a FUNCTION that receives what it decorates

// Class Decorator: receives the constructor function
function Injectable(constructor: Function) {
  // NestJS registers this class as a provider in its DI container
  Reflect.defineMetadata('injectable', true, constructor);
  console.log(`${constructor.name} is now injectable`);
}

// Method Decorator: receives target, method name, and property descriptor
function Get(path: string) {
  return function(target: any, methodName: string, descriptor: PropertyDescriptor) {
    // NestJS registers this method as a GET handler for the path
    Reflect.defineMetadata('method', 'GET', target, methodName);
    Reflect.defineMetadata('path', path, target, methodName);
  };
}

// Using these decorators:
@Injectable()  // Calls Injectable(UsersService)
class UsersService {
  @Get('/users')  // Calls Get('/users')(UsersService.prototype, 'findAll', descriptor)
  findAll() {
    return [];
  }
}
```

### Types of decorators and NestJS examples

```typescript
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 1. CLASS DECORATORS — annotate the whole class
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@Controller('users')    // Marks this as a controller, sets route prefix
@Injectable()           // Marks this as a provider (injectable dependency)
@Module({})             // Marks this as a NestJS module
@Entity()               // TypeORM — marks this as a database table


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 2. METHOD DECORATORS — annotate methods
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@Get(':id')             // Register as HTTP GET handler
@Post()                 // Register as HTTP POST handler
@UseGuards(AuthGuard)   // Apply guard to this method
@UsePipes(ValidationPipe)  // Apply pipe to this method


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 3. PARAMETER DECORATORS — extract data from request
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@Param('id')            // Extract route parameter :id
@Body()                 // Extract request body
@Query('page')          // Extract query string ?page=1
@Headers('authorization')  // Extract header value
@Req()                  // Entire request object
@Res()                  // Entire response object (use sparingly)


// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
// 4. PROPERTY DECORATORS — annotate class properties
// ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

@Column()               // TypeORM — maps to a DB column
@PrimaryGeneratedColumn()  // TypeORM — auto-increment primary key
@IsEmail()              // class-validator — validate email format
@IsNotEmpty()           // class-validator — must not be empty
```

### Writing a custom decorator (this impresses interviewers)

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

// Custom parameter decorator to extract the current user from request
// After AuthGuard runs, it attaches the user to request.user
// This decorator makes extracting it clean and reusable

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    // ctx gives us access to the underlying HTTP request
    const request = ctx.switchToHttp().getRequest();
    
    const user = request.user;  // Set by JWT auth guard
    
    // If data is passed (e.g., @CurrentUser('email')), return that field
    // If not, return the whole user object
    return data ? user?.[data] : user;
  },
);

// Usage in controller:
@Get('profile')
@UseGuards(JwtAuthGuard)
getProfile(@CurrentUser() user: User) {
  // user is automatically extracted from request.user
  return user;
}

@Get('email')
@UseGuards(JwtAuthGuard)
getEmail(@CurrentUser('email') email: string) {
  // Only the email field is extracted
  return { email };
}
```

### Reflect.metadata — how NestJS reads decorators at runtime

```typescript
// NestJS uses the Reflect Metadata API (a polyfill for a proposed JS standard)
// Decorators attach metadata to classes/methods
// NestJS reads this metadata to wire everything together

// Example: NestJS reads parameter types for dependency injection
import 'reflect-metadata';  // This import is in main.ts — required for NestJS

@Injectable()
class UsersService {
  constructor(private emailService: EmailService) {
    // At compile time, TypeScript emits:
    // Reflect.defineMetadata('design:paramtypes', [EmailService], UsersService)
    
    // At runtime, NestJS reads:
    // Reflect.getMetadata('design:paramtypes', UsersService)
    // → [EmailService]
    // And it knows to inject an instance of EmailService
  }
}

// This is why you need "emitDecoratorMetadata": true in tsconfig.json
// Without it, TypeScript doesn't emit parameter type information
// And NestJS's dependency injection breaks
```

---

## 10. ES Modules vs CommonJS

### The two module systems in Node.js

```
CommonJS (CJS)          ES Modules (ESM)
─────────────           ──────────────────
require()               import
module.exports          export / export default
.js files               .mjs files or "type":"module"
Loads synchronously     Can load asynchronously
Node.js original        JavaScript standard (ES2015+)
```

### CommonJS — Node.js's original system

```javascript
// CommonJS — how Node.js has worked since day 1

// Exporting
const PI = 3.14159;
function add(a, b) { return a + b; }
class Calculator { /* ... */ }

module.exports = { PI, add, Calculator };
// OR shorthand:
module.exports.greet = (name) => `Hello, ${name}`;

// Importing
const { PI, add } = require('./math');
const fs = require('fs');           // Built-in module
const express = require('express'); // npm package

// require() characteristics:
// 1. Synchronous — blocks until module loads
// 2. Can be called anywhere in code (inside if statements, functions)
// 3. Returns cached module on repeat calls (modules are singletons)
// 4. Dynamic — can use variables: require(`./${name}`)
```

### ES Modules — the JavaScript standard

```javascript
// ES Modules — the modern JavaScript way

// Named exports
export const PI = 3.14159;
export function add(a, b) { return a + b; }
export class Calculator { /* ... */ }

// Default export (one per file)
export default class App { /* ... */ }

// Importing named exports
import { PI, add } from './math.js';  // Extension required in ESM!

// Importing default export
import App from './app.js';

// Import everything as namespace
import * as MathUtils from './math.js';
MathUtils.add(1, 2);

// Dynamic import — returns a Promise (code splitting)
const module = await import('./heavy-module.js');

// import() characteristics:
// 1. Top-level only (static analysis)
// 2. Must be at file level, not inside functions (unless dynamic import())
// 3. Imports are live bindings — reflect changes to exports
```

### What TypeScript and NestJS use

```typescript
// NestJS uses TypeScript which compiles to CommonJS by default
// BUT your source code uses ES Module syntax (import/export)
// TypeScript compiles your import/export → require()/module.exports

// In your NestJS code (TypeScript source):
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';

@Injectable()
export class UsersService {
  // ...
}

// What TypeScript compiles to (JavaScript output):
"use strict";
const common_1 = require("@nestjs/common");
const typeorm_1 = require("@nestjs/typeorm");
const typeorm_2 = require("typeorm");
const user_entity_1 = require("./user.entity");

// So at runtime, NestJS runs on CommonJS
// But you write in ES Module syntax thanks to TypeScript
```

---

## 11. tsconfig.json — TypeScript Compiler Settings

### The full NestJS tsconfig explained

```json
{
  "compilerOptions": {
    // ─── Target & Module ───────────────────────────────────────
    
    "module": "commonjs",
    // What module system to compile TO
    // "commonjs" → Node.js compatible (require/module.exports)
    // NestJS uses commonjs at runtime even though you write ESM syntax
    
    "target": "ES2021",
    // What JavaScript version to compile TO
    // ES2021 supports: async/await, optional chaining (?.), nullish coalescing (??)
    // Higher target = more modern JS features kept as-is (smaller output)
    
    "lib": ["ES2021"],
    // TypeScript's built-in type definitions to include
    // ES2021 includes: Promise, Array methods, Object methods, etc.
    
    // ─── Decorator Support ────────────────────────────────────
    
    "experimentalDecorators": true,
    // REQUIRED for NestJS — enables @Decorator syntax
    // Without this: "Decorators are not valid here" error
    
    "emitDecoratorMetadata": true,
    // REQUIRED for NestJS dependency injection
    // Makes TypeScript emit type information about constructor parameters
    // NestJS reads this to know WHAT to inject into services
    // Without this: NestJS DI (dependency injection) breaks completely
    
    // ─── Strictness ───────────────────────────────────────────
    
    "strict": true,
    // Enables ALL strict type checking options at once:
    // - strictNullChecks: null/undefined are not assignable to other types
    // - strictPropertyInitialization: class properties must be initialized
    // - noImplicitAny: variables must have explicit types
    // - strictFunctionTypes: function parameters checked contravariantly
    
    "strictNullChecks": true,
    // null and undefined are separate types
    // Without this: string includes null/undefined (dangerous!)
    // With this: you MUST handle the null case explicitly
    
    "noImplicitAny": true,
    // Variables must have explicit types — no sneaky 'any'
    // Forces you to type everything properly
    
    // ─── Output ───────────────────────────────────────────────
    
    "outDir": "./dist",
    // Where compiled JavaScript files go
    // 'nest build' compiles your .ts → ./dist/*.js
    
    "rootDir": "./src",
    // Root of TypeScript source files
    // All .ts files in src/ get compiled
    
    "sourceMap": true,
    // Generate .js.map files
    // Maps compiled JS back to original TS for debugging
    // Error stack traces show TypeScript line numbers, not compiled JS
    
    // ─── Path Resolution ──────────────────────────────────────
    
    "baseUrl": "./",
    // Base directory for resolving non-relative imports
    
    "paths": {
      "@app/*": ["src/app/*"],
      "@common/*": ["src/common/*"]
    },
    // Alias paths — instead of '../../../common/utils'
    // you can write '@common/utils'
    // MUST also configure in webpack or tsconfig-paths at runtime
    
    // ─── Other Important Options ──────────────────────────────
    
    "esModuleInterop": true,
    // Allows default imports from CommonJS modules
    // Without this: import express from 'express' fails
    // With this: both import styles work
    
    "skipLibCheck": true,
    // Skip type checking of .d.ts declaration files in node_modules
    // Speeds up compilation significantly (some packages have type errors)
    
    "incremental": true,
    // Save compilation state to .tsbuildinfo
    // Subsequent compilations only recompile changed files
    // Speeds up 'nest build' during development
    
    "allowJs": false,
    // Don't compile .js files alongside .ts
    // NestJS projects are TypeScript-only
    
    "declaration": true,
    // Generate .d.ts type declaration files
    // Required if you're building a library for others to import
    
    "removeComments": true
    // Strip comments from output .js files
    // Smaller bundle size in production
  },
  "include": ["src/**/*"],     // Compile all files in src/
  "exclude": ["node_modules", "dist", "**/*spec.ts"]  // Skip these
}
```

---

## 12. NPM & package.json — Project Anatomy

### NestJS package.json explained

```json
{
  "name": "my-nestjs-app",
  "version": "1.0.0",
  "description": "My NestJS backend API",
  
  "scripts": {
    // NestJS CLI commands
    "build": "nest build",
    // Compiles TypeScript → JavaScript in ./dist
    
    "start": "nest start",
    // Start the app (compiles first)
    
    "start:dev": "nest start --watch",
    // Development mode — watches for file changes, auto-restarts
    // Uses ts-node for hot reload without full compilation
    
    "start:debug": "nest start --debug --watch",
    // Debug mode — exposes debugger port 9229
    // Connect Chrome DevTools or VS Code debugger
    
    "start:prod": "node dist/main",
    // Production start — runs pre-compiled JavaScript
    // NEVER use 'nest start' in production (compiles on every start)
    
    "test": "jest",
    // Run all unit tests
    
    "test:watch": "jest --watch",
    // Run tests in watch mode during development
    
    "test:cov": "jest --coverage",
    // Run tests with code coverage report
    
    "test:e2e": "jest --config ./test/jest-e2e.json",
    // Run end-to-end tests (tests actual HTTP endpoints)
    
    "lint": "eslint \"{src,apps,libs,test}/**/*.ts\" --fix"
    // Run ESLint and auto-fix fixable issues
  },
  
  "dependencies": {
    // Runtime dependencies — included in production
    
    "@nestjs/common": "^10.0.0",
    // Core NestJS package — all decorators, pipes, guards, etc.
    
    "@nestjs/core": "^10.0.0",
    // NestJS runtime — application factory, module compiler
    
    "@nestjs/platform-express": "^10.0.0",
    // HTTP adapter using Express.js
    // Alternative: @nestjs/platform-fastify (faster, less ecosystem)
    
    "reflect-metadata": "^0.1.13",
    // Polyfill for Reflect.metadata API
    // REQUIRED — enables decorator metadata
    // Must be imported first in main.ts: import 'reflect-metadata'
    
    "rxjs": "^7.8.0"
    // Reactive Extensions — NestJS uses Observables internally
    // Used heavily in microservices and WebSockets
  },
  
  "devDependencies": {
    // Development only — NOT in production bundle
    
    "@nestjs/cli": "^10.0.0",
    // The 'nest' CLI command — 'nest new', 'nest generate', 'nest build'
    
    "@nestjs/schematics": "^10.0.0",
    // Code generation templates for 'nest generate'
    
    "@nestjs/testing": "^10.0.0",
    // Testing utilities — TestingModule for unit/integration tests
    
    "@types/express": "^4.17.0",
    // TypeScript type definitions for Express
    // Not Express itself — just the types for TypeScript
    
    "@types/node": "^20.0.0",
    // TypeScript types for Node.js built-ins (fs, path, http, etc.)
    
    "typescript": "^5.0.0",
    // The TypeScript compiler itself
    
    "ts-jest": "^29.0.0",
    // Jest transformer — lets Jest understand TypeScript files
    
    "jest": "^29.0.0",
    // The testing framework
    
    "eslint": "^8.0.0",
    // Code linter
    
    "prettier": "^3.0.0"
    // Code formatter
  },
  
  "jest": {
    // Jest configuration inline in package.json
    "moduleFileExtensions": ["js", "json", "ts"],
    "rootDir": "src",
    "testRegex": ".*\\.spec\\.ts$",  // Files ending in .spec.ts are tests
    "transform": {
      "^.+\\.(t|j)s$": "ts-jest"    // Use ts-jest to compile TypeScript
    },
    "coverageDirectory": "../coverage",
    "testEnvironment": "node"
  }
}
```

### Essential NestJS CLI commands

```bash
# Create a new NestJS project
nest new my-app

# Generate components — saves time, creates files + updates module
nest generate module users          # Creates users/users.module.ts
nest generate controller users      # Creates users/users.controller.ts + spec
nest generate service users         # Creates users/users.service.ts + spec
nest generate resource users        # Creates ALL of the above + DTO files

# Short form
nest g mo users    # module
nest g co users    # controller
nest g s users     # service
nest g res users   # resource (full CRUD scaffold)

# Build for production
nest build

# Check NestJS version
nest --version

# Install a package and its types
npm install @nestjs/jwt
npm install @types/passport-jwt --save-dev
```

---

## 13. Interview Questions & Answers

These are real questions asked in Node.js backend engineer interviews.

---

**Q1: What is the event loop and how does it work?**

> "The event loop is Node.js's mechanism for handling asynchronous operations on a single thread. It continuously checks if there are tasks to execute. It processes several phases in order: timers (setTimeout/setInterval), pending I/O callbacks, poll (waiting for new I/O), check (setImmediate), and close callbacks. Between phases, it drains the microtask queues — first `process.nextTick()`, then Promise callbacks. This allows Node.js to handle thousands of concurrent connections without creating new threads for each one."

---

**Q2: What is the difference between process.nextTick() and setImmediate()?**

> "`process.nextTick()` fires before the event loop continues to the next phase — it runs after the current operation completes but before any I/O. `setImmediate()` fires in the check phase, after I/O events are processed. So `process.nextTick()` always runs before `setImmediate()`, even if both are called in the same context. Use `process.nextTick()` for operations that must run before any I/O, and `setImmediate()` for operations that should run after I/O but not delay it."

---

**Q3: What is the difference between interface and type in TypeScript?**

> "Both define type shapes, but with key differences. Interfaces can be extended using `extends` and support declaration merging — two interface declarations with the same name merge into one. Type aliases support union types (`string | number`), intersection types, and mapped types. In practice: use `interface` for object shapes and class contracts where you might need extension or merging. Use `type` for unions, function signatures, primitives, and complex computed types. In NestJS, interfaces define service contracts and DTOs, while types are used for utility types and unions."

---

**Q4: What is async/await and how does it relate to Promises?**

> "async/await is syntactic sugar over Promises, introduced in ES2017. An `async` function always returns a Promise. The `await` keyword pauses execution of the async function until a Promise resolves, then returns the resolved value. Under the hood, TypeScript compiles async/await to generator functions and Promise chains. Critically, `await` does NOT block the Node.js event loop — it suspends the current function and allows other code to run. The function resumes when the awaited Promise resolves."

---

**Q5: Why is blocking the event loop dangerous?**

> "Node.js runs JavaScript on a single thread. If you perform a CPU-intensive synchronous operation, it occupies that single thread and the event loop cannot process any other requests — the entire application freezes for all concurrent users. For example, synchronously computing a large Fibonacci number, parsing a huge JSON file with `JSON.parse()` on a very large string, or using `bcrypt` with a high cost factor synchronously — all block the loop. Solutions include: Worker Threads for CPU work, `setImmediate()` to break up long tasks, offloading to separate processes, or using async versions of all operations."

---

**Q6: What is emitDecoratorMetadata and why does NestJS require it?**

> "It's a TypeScript compiler option that causes TypeScript to emit runtime metadata about class constructors and their parameter types using the `Reflect.metadata` API. NestJS's dependency injection system reads this metadata at runtime to determine what to inject into service constructors. Without it, when NestJS creates an instance of `UsersService`, it can't determine what `Repository<User>` or `EmailService` means — those type annotations don't exist at runtime. With `emitDecoratorMetadata: true`, TypeScript generates `Reflect.defineMetadata('design:paramtypes', [Repository, EmailService], UsersService)`, and NestJS reads this to perform injection."

---

**Q7: What is the difference between Promise.all and Promise.allSettled?**

> "`Promise.all` takes an array of Promises and returns a single Promise that resolves when ALL input Promises resolve, with an array of all resolved values. If ANY Promise rejects, `Promise.all` immediately rejects — you lose all results. `Promise.allSettled` also waits for all Promises but NEVER rejects. It resolves with an array of objects, each with a `status` of either 'fulfilled' (with `value`) or 'rejected' (with `reason'`). Use `Promise.all` when all operations must succeed for your logic to work. Use `Promise.allSettled` when you want all results regardless of individual failures — like fetching from multiple data sources where partial data is still useful."

---

**Q8: What are decorators in TypeScript and how do they work internally?**

> "Decorators are special functions prefixed with `@` that can be attached to classes, methods, properties, or parameters. They receive the decorated target as arguments and can modify its behavior or attach metadata. Internally, `@Injectable()` calls `Injectable(TargetClass)` which uses `Reflect.defineMetadata` to mark the class as injectable. NestJS then reads this metadata during module compilation to build its dependency injection container. Decorators execute at class definition time (when the module loads), not when an instance is created."

---

## Quick Reference — Day 1 Cheat Sheet

```
Event Loop Phases:    timers → pending callbacks → poll → check → close
Priority Order:       nextTick > Promise microtasks > setTimeout > setImmediate
Callback pattern:     (err, result) — always error first
Promise states:       pending → fulfilled | rejected (immutable once settled)
async always returns: a Promise, even if you return a plain value
await suspends:       the function, NOT the event loop
TypeScript compiles:  import/export → require/module.exports (CommonJS)

tsconfig must-haves for NestJS:
  ✓ experimentalDecorators: true   (enables @Decorator syntax)
  ✓ emitDecoratorMetadata: true    (enables DI — most critical)
  ✓ strict: true                   (type safety)
  ✓ target: ES2021                 (modern JS features)

Decorator types:      Class / Method / Property / Parameter
Generics:            function<T>() — T is a type variable, filled at call time
interface vs type:   interface extends & merges; type supports unions & computed
```

---

*Day 1 complete. Tomorrow: NestJS Core Architecture — Modules, Controllers, Providers, and how dependency injection actually works.*
