# 10 — Strategy Pattern

> **Category:** GoF Behavioral  
> **Difficulty:** ⭐⭐ Beginner-Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Imagine you're navigating in Google Maps. You can choose: **Drive**, **Walk**, **Bike**, or **Transit**. Each gives a different route. The destination is the same; the algorithm to get there changes.

The **Strategy pattern** is exactly this: define a family of algorithms, put each in its own class, and make them interchangeable at runtime. The context (your app) picks which strategy to use without changing its own code.

**In one sentence:** Define a family of algorithms, encapsulate each one, and make them swappable — the algorithm can vary independently from the code that uses it.

---

## Visual Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                      Strategy Pattern                         │
│                                                              │
│   Context (PaymentProcessor)                                 │
│   ┌────────────────────────────┐                             │
│   │  strategy = ???            │  ← swappable at runtime    │
│   │                            │                             │
│   │  processPayment(amount) {  │                             │
│   │    strategy.pay(amount)    │──────────────┐              │
│   │  }                         │              │              │
│   └────────────────────────────┘              ▼              │
│                                                              │
│   Strategies:          ┌─────────────────────────────────┐  │
│                        │  <<interface>> PaymentStrategy  │  │
│   StripeStrategy  ────>│  pay(amount): Promise           │  │
│   RazorpayStrategy ───>│                                 │  │
│   CryptoStrategy  ────>└─────────────────────────────────┘  │
│   WalletStrategy  ────>                                      │
│                                                              │
│   Client picks which strategy to inject.                     │
│   Context never changes. Strategies are interchangeable.     │
└──────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Version 1: Payment Strategy

```javascript
// ── Strategy Interface (duck typing in JS) ──
// Each strategy must have: pay(amount, details), getName()

class StripeStrategy {
  async pay(amount, details) {
    console.log(`💳 [Stripe] Charging ₹${amount} to card ending ${details.cardLast4}`);
    // stripe.charges.create(...)
    return { success: true, transactionId: `stripe_${Date.now()}`, provider: 'stripe' };
  }
  getName() { return 'stripe'; }
}

class RazorpayStrategy {
  async pay(amount, details) {
    console.log(`🟣 [Razorpay] Creating UPI payment of ₹${amount} to ${details.upiId}`);
    // razorpay.orders.create(...)
    return { success: true, transactionId: `rzp_${Date.now()}`, provider: 'razorpay' };
  }
  getName() { return 'razorpay'; }
}

class WalletStrategy {
  #walletBalance;
  constructor(walletBalance) { this.#walletBalance = walletBalance; }

  async pay(amount, details) {
    if (this.#walletBalance < amount) {
      throw new Error(`Insufficient wallet balance. Have: ₹${this.#walletBalance}, Need: ₹${amount}`);
    }
    this.#walletBalance -= amount;
    console.log(`👛 [Wallet] Paid ₹${amount}. Remaining balance: ₹${this.#walletBalance}`);
    return { success: true, transactionId: `wallet_${Date.now()}`, provider: 'wallet' };
  }
  getName() { return 'wallet'; }
}

class CashOnDeliveryStrategy {
  async pay(amount, details) {
    console.log(`💵 [COD] Order marked for cash collection of ₹${amount} at delivery`);
    return { success: true, transactionId: `cod_${Date.now()}`, provider: 'cod', collectOnDelivery: true };
  }
  getName() { return 'cod'; }
}

// ── Context — uses whichever strategy is injected ──

class PaymentProcessor {
  #strategy;

  constructor(strategy) {
    this.#strategy = strategy;
  }

  // Swap strategy at runtime
  setStrategy(strategy) {
    console.log(`[PaymentProcessor] Switching to ${strategy.getName()} strategy`);
    this.#strategy = strategy;
  }

  async processPayment(amount, details) {
    console.log(`\nProcessing ₹${amount} via ${this.#strategy.getName()}...`);

    // Pre-processing (same regardless of strategy)
    if (amount <= 0) throw new Error('Amount must be positive');
    if (amount > 500000) throw new Error('Amount exceeds maximum limit');

    // Delegate to strategy
    const result = await this.#strategy.pay(amount, details);

    // Post-processing (same regardless of strategy)
    if (result.success) {
      console.log(`✅ Payment successful | txn: ${result.transactionId}`);
    }
    return result;
  }
}

// ── Factory to pick the right strategy ──

class PaymentStrategyFactory {
  static create(method, options = {}) {
    const strategies = {
      stripe:    () => new StripeStrategy(),
      razorpay:  () => new RazorpayStrategy(),
      wallet:    () => new WalletStrategy(options.balance || 0),
      cod:       () => new CashOnDeliveryStrategy(),
    };

    const factory = strategies[method];
    if (!factory) throw new Error(`Unknown payment method: ${method}`);
    return factory();
  }
}

// ── Usage ──

// User selects payment method at checkout
const userSelectedMethod = 'razorpay'; // comes from frontend
const strategy = PaymentStrategyFactory.create(userSelectedMethod);
const processor = new PaymentProcessor(strategy);

await processor.processPayment(1999, { upiId: 'alice@okaxis' });
// Processing ₹1999 via razorpay...
// 🟣 [Razorpay] Creating UPI payment of ₹1999 to alice@okaxis
// ✅ Payment successful | txn: rzp_1234567890

// User switches to wallet mid-session:
processor.setStrategy(PaymentStrategyFactory.create('wallet', { balance: 5000 }));
await processor.processPayment(499, {});
// 👛 [Wallet] Paid ₹499. Remaining balance: ₹4501
```

---

### Version 2: Sorting Strategy

```javascript
// Different sort algorithms as strategies
class BubbleSortStrategy {
  sort(arr) {
    const a = [...arr];
    for (let i = 0; i < a.length; i++)
      for (let j = 0; j < a.length - i - 1; j++)
        if (a[j] > a[j+1]) [a[j], a[j+1]] = [a[j+1], a[j]];
    return a;
  }
  getName() { return 'bubble'; }
}

class QuickSortStrategy {
  sort(arr) {
    if (arr.length <= 1) return arr;
    const pivot = arr[Math.floor(arr.length / 2)];
    const left  = arr.filter(x => x < pivot);
    const mid   = arr.filter(x => x === pivot);
    const right = arr.filter(x => x > pivot);
    return [...this.sort(left), ...mid, ...this.sort(right)];
  }
  getName() { return 'quicksort'; }
}

class BuiltInSortStrategy {
  sort(arr) { return [...arr].sort((a, b) => a - b); }
  getName() { return 'built-in'; }
}

class Sorter {
  constructor(strategy = new BuiltInSortStrategy()) {
    this.strategy = strategy;
  }
  setStrategy(s) { this.strategy = s; }
  sort(data) {
    console.log(`Sorting ${data.length} items with ${this.strategy.getName()}`);
    return this.strategy.sort(data);
  }
}

const sorter = new Sorter();
const data = [64, 25, 12, 22, 11];
console.log(sorter.sort(data)); // [11, 12, 22, 25, 64]

sorter.setStrategy(new QuickSortStrategy());
console.log(sorter.sort(data)); // [11, 12, 22, 25, 64] — same result, different algorithm
```

---

### Version 3: Auth Strategy (Passport.js Style)

```javascript
// Authentication strategies
class JWTStrategy {
  async authenticate(req) {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) throw new Error('No JWT token');
    const payload = verifyJWT(token, process.env.JWT_SECRET);
    return { userId: payload.sub, role: payload.role, method: 'jwt' };
  }
}

class ApiKeyStrategy {
  #validKeys = new Map([
    ['key_live_abc123', { userId: 'api_user_1', role: 'api' }],
    ['key_live_xyz789', { userId: 'api_user_2', role: 'api' }],
  ]);

  async authenticate(req) {
    const key = req.headers['x-api-key'];
    if (!key) throw new Error('No API key');
    const user = this.#validKeys.get(key);
    if (!user) throw new Error('Invalid API key');
    return { ...user, method: 'api_key' };
  }
}

class OAuth2Strategy {
  async authenticate(req) {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) throw new Error('No OAuth token');
    // Verify with OAuth provider
    const profile = await verifyOAuthToken(token);
    return { userId: profile.id, email: profile.email, method: 'oauth2' };
  }
}

// Auth middleware using strategy
class AuthMiddleware {
  #strategies = new Map();

  registerStrategy(name, strategy) {
    this.#strategies.set(name, strategy);
  }

  // Returns Express middleware
  authenticate(strategyName) {
    const strategy = this.#strategies.get(strategyName);
    if (!strategy) throw new Error(`Unknown auth strategy: ${strategyName}`);

    return async (req, res, next) => {
      try {
        req.user = await strategy.authenticate(req);
        next();
      } catch (error) {
        res.status(401).json({ error: error.message });
      }
    };
  }
}

const auth = new AuthMiddleware();
auth.registerStrategy('jwt',     new JWTStrategy());
auth.registerStrategy('api-key', new ApiKeyStrategy());
auth.registerStrategy('oauth2',  new OAuth2Strategy());

// Express routes use different auth strategies:
app.get('/api/data',      auth.authenticate('jwt'),     dataHandler);
app.get('/api/webhooks',  auth.authenticate('api-key'), webhookHandler);
app.get('/api/social',    auth.authenticate('oauth2'),  socialHandler);
```

---

## Strategy vs State Pattern

This is a common interview question:

| | Strategy | State |
|--|----------|-------|
| **Purpose** | Choose ALGORITHM | Manage BEHAVIOR based on state |
| **Who changes?** | Client changes strategy | Object changes its own state |
| **Aware of each other?** | Strategies don't know each other | States may transition to other states |
| **Example** | Sort algorithm, payment method | Traffic light, order status |

---

## Interview Q&A

**Q: What is Strategy pattern?**
> Defines a family of algorithms, encapsulates each, and makes them interchangeable. The algorithm can vary independently from clients that use it.

**Q: Strategy vs if/else?**
> if/else mixes all algorithms into one class, violates Open/Closed Principle. Strategy puts each algorithm in its own class — add new ones without touching existing code.

**Q: Real examples in Node.js?**
> Passport.js authentication strategies, sorting/compression algorithms, payment method selection, validation strategies, caching strategies (memory/Redis/disk).

---

---

# 11 — Command Pattern

> **Category:** GoF Behavioral  
> **Difficulty:** ⭐⭐⭐ Intermediate  
> **Node.js relevance:** ⭐⭐⭐⭐

---

## What is it? (The Simple Idea)

Think of a **TV remote control**. Each button is a command. You press "Volume Up" — the remote doesn't know HOW the TV increases volume, it just sends the command. The TV executes it.

Now think about "undo." You pressed Volume Up too many times. Press Volume Down. That's the reverse command.

The **Command pattern** wraps a request into an object. This object can be stored, queued, logged, and undone.

**In one sentence:** Encapsulate a request (operation + its parameters + how to undo it) as an object so you can queue, log, and reverse operations.

---

## Visual Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                       Command Pattern                           │
│                                                                │
│  Invoker              Command Object         Receiver          │
│  (Remote)             ─────────────          (TV)              │
│  ┌─────────┐         ┌────────────────┐    ┌──────────────┐  │
│  │ execute │────────>│ execute()      │───>│ volumeUp()   │  │
│  │ undo    │<────────│ undo()         │<───│ volumeDown() │  │
│  │ history │         │                │    │              │  │
│  └─────────┘         │ + stored state │    └──────────────┘  │
│                       │   for undo     │                       │
│                       └────────────────┘                       │
│                                                                │
│  Benefits:                                                     │
│  ● Undo / Redo stack                                           │
│  ● Queue commands for later execution                          │
│  ● Log all operations (audit trail)                            │
│  ● Retry failed commands                                       │
│  ● Batch commands (transaction)                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Node.js Implementation

### Version 1: Text Editor with Undo/Redo

```javascript
// ── Commands ──

class InsertTextCommand {
  constructor(document, position, text) {
    this.document = document;
    this.position = position;
    this.text     = text;
  }

  execute() {
    this.document.content =
      this.document.content.slice(0, this.position) +
      this.text +
      this.document.content.slice(this.position);
    console.log(`[INSERT] "${this.text}" at position ${this.position}`);
  }

  undo() {
    this.document.content =
      this.document.content.slice(0, this.position) +
      this.document.content.slice(this.position + this.text.length);
    console.log(`[UNDO INSERT] Removed "${this.text}"`);
  }
}

class DeleteTextCommand {
  constructor(document, start, length) {
    this.document = document;
    this.start    = start;
    this.length   = length;
    this.deleted  = null; // saved for undo
  }

  execute() {
    this.deleted  = this.document.content.slice(this.start, this.start + this.length);
    this.document.content =
      this.document.content.slice(0, this.start) +
      this.document.content.slice(this.start + this.length);
    console.log(`[DELETE] Removed "${this.deleted}"`);
  }

  undo() {
    this.document.content =
      this.document.content.slice(0, this.start) +
      this.deleted +
      this.document.content.slice(this.start);
    console.log(`[UNDO DELETE] Restored "${this.deleted}"`);
  }
}

class BoldTextCommand {
  constructor(document, start, length) {
    this.document = document;
    this.start    = start;
    this.length   = length;
  }

  execute() {
    const text = this.document.content.slice(this.start, this.start + this.length);
    this.document.content =
      this.document.content.slice(0, this.start) +
      `**${text}**` +
      this.document.content.slice(this.start + this.length);
    console.log(`[BOLD] Applied bold to "${text}"`);
  }

  undo() {
    // Remove the ** markers we added
    this.document.content =
      this.document.content.slice(0, this.start) +
      this.document.content.slice(this.start + 2, this.start + 2 + this.length) +
      this.document.content.slice(this.start + 2 + this.length + 2);
    console.log(`[UNDO BOLD] Removed bold`);
  }
}

// ── Command History (the Invoker) ──

class DocumentEditor {
  #document;
  #history = [];    // executed commands (for undo)
  #redoStack = [];  // undone commands (for redo)

  constructor(initialContent = '') {
    this.#document = { content: initialContent };
  }

  execute(command) {
    command.execute();
    this.#history.push(command);
    this.#redoStack = []; // clear redo on new action
    return this;
  }

  undo() {
    if (!this.#history.length) {
      console.log('Nothing to undo');
      return this;
    }
    const command = this.#history.pop();
    command.undo();
    this.#redoStack.push(command);
    return this;
  }

  redo() {
    if (!this.#redoStack.length) {
      console.log('Nothing to redo');
      return this;
    }
    const command = this.#redoStack.pop();
    command.execute();
    this.#history.push(command);
    return this;
  }

  getContent() { return this.#document.content; }
  getDocument() { return this.#document; }

  // Get history for audit
  getHistory() {
    return this.#history.map(cmd => cmd.constructor.name);
  }
}

// ── Usage ──

const editor = new DocumentEditor('Hello World');
console.log('Start:', editor.getContent()); // Hello World

editor.execute(new InsertTextCommand(editor.getDocument(), 5, ','));
console.log('After insert:', editor.getContent()); // Hello, World

editor.execute(new DeleteTextCommand(editor.getDocument(), 7, 5));
console.log('After delete:', editor.getContent()); // Hello, 

editor.execute(new InsertTextCommand(editor.getDocument(), 7, 'Node.js'));
console.log('After insert:', editor.getContent()); // Hello, Node.js

editor.undo();
console.log('After undo:', editor.getContent()); // Hello, 

editor.undo();
console.log('After undo:', editor.getContent()); // Hello, World

editor.redo();
console.log('After redo:', editor.getContent()); // Hello, 
```

---

### Version 2: Task Queue (Deferred Command Execution)

```javascript
// Commands that are queued and executed later (job queue pattern)

class SendEmailCommand {
  constructor(to, subject, body) {
    this.to = to; this.subject = subject; this.body = body;
    this.createdAt = new Date();
    this.attempts = 0;
  }

  async execute() {
    this.attempts++;
    console.log(`📧 Sending email to ${this.to}: "${this.subject}" (attempt ${this.attempts})`);
    if (Math.random() < 0.3) throw new Error('Email service temporarily unavailable');
    return { sent: true, to: this.to };
  }

  async undo() {
    console.log(`[Cannot undo sent email to ${this.to}]`);
  }
}

class GenerateReportCommand {
  constructor(reportType, params) {
    this.reportType = reportType; this.params = params;
    this.attempts = 0;
  }

  async execute() {
    this.attempts++;
    console.log(`📊 Generating ${this.reportType} report...`);
    await new Promise(r => setTimeout(r, 100)); // simulate work
    return { reportId: `rpt_${Date.now()}`, type: this.reportType };
  }

  async undo() { console.log(`Deleting report ${this.reportType}`); }
}

class ResizeImageCommand {
  constructor(imageUrl, width, height) {
    this.imageUrl = imageUrl; this.width = width; this.height = height;
    this.attempts = 0;
  }

  async execute() {
    this.attempts++;
    console.log(`🖼️  Resizing ${this.imageUrl} to ${this.width}x${this.height}`);
    return { resizedUrl: `${this.imageUrl}_${this.width}x${this.height}.jpg` };
  }

  async undo() { console.log(`Deleting resized image`); }
}

// ── Command Queue (Job Processor) ──

class CommandQueue {
  #queue = [];
  #processing = false;
  #maxRetries = 3;
  #results = [];

  enqueue(command) {
    this.#queue.push(command);
    console.log(`[QUEUE] Added ${command.constructor.name}. Queue size: ${this.#queue.length}`);
    if (!this.#processing) this.#processNext();
    return this;
  }

  async #processNext() {
    if (!this.#queue.length) { this.#processing = false; return; }

    this.#processing = true;
    const command = this.#queue.shift();

    for (let attempt = 1; attempt <= this.#maxRetries; attempt++) {
      try {
        const result = await command.execute();
        this.#results.push({ command: command.constructor.name, success: true, result });
        break;
      } catch (error) {
        console.warn(`[QUEUE] ${command.constructor.name} failed (attempt ${attempt}): ${error.message}`);
        if (attempt === this.#maxRetries) {
          this.#results.push({ command: command.constructor.name, success: false, error: error.message });
        } else {
          await new Promise(r => setTimeout(r, 1000 * attempt));
        }
      }
    }

    this.#processNext(); // process next command
  }

  getResults() { return this.#results; }
  getPending()  { return this.#queue.length; }
}

// Usage
const queue = new CommandQueue();
queue
  .enqueue(new SendEmailCommand('alice@example.com', 'Welcome!', 'Hello Alice'))
  .enqueue(new GenerateReportCommand('monthly-sales', { month: 'Jan', year: 2024 }))
  .enqueue(new ResizeImageCommand('https://cdn.example.com/photo.jpg', 800, 600))
  .enqueue(new SendEmailCommand('bob@example.com', 'Order Shipped', 'Your order is on the way'));

// Commands execute one by one, with retry on failure
```

---

## Interview Q&A

**Q: What is the Command pattern?**
> Encapsulates a request as an object, containing the action, its parameters, and how to undo it. Enables queuing, logging, retry, and undo/redo.

**Q: When do you use Command?**
> When you need undo/redo (text editors), when you need to queue operations (job queues), when you need transaction support (rollback), or when you want to audit/log all operations.

**Q: How is undo/redo implemented?**
> Maintain two stacks: history (executed) and redoStack (undone). `execute()` pushes to history. `undo()` pops from history, calls `command.undo()`, pushes to redoStack. `redo()` pops from redoStack, calls `command.execute()`, pushes back to history.

**Q: Real examples in Node.js?**
> Job queues (Bull, BullMQ), database migration runners (each migration = a command with up/down), event sourcing (each event is a command), HTTP request queuing.

---

## Summary

```
Strategy Pattern
├── What: Swap algorithms at runtime
├── Why: Avoid giant if/else for algorithm selection
├── How: Interface + concrete strategy classes + context that delegates
└── Examples: Payment methods, auth strategies, sort algorithms

Command Pattern
├── What: Wrap requests as objects with execute() and undo()
├── Why: Undo/redo, queuing, retry, audit trail, transactions
├── How: Command objects go into a history stack; undo pops and reverses
└── Examples: Text editors, job queues, DB migrations, event sourcing
```
