# Day 4 — Core Built-in Modules: `fs`, `path`, `os`, and `EventEmitter`

> **Why this day matters:** These are the modules you `require()` without even thinking about it in almost every real Node.js backend project. More importantly, `EventEmitter` is not just "a module" — it's the **pattern that Node's entire internals are built on** (HTTP servers, streams, process signals — all of it emits events under the hood). Understanding `EventEmitter` deeply will make several later topics (streams on Day 5, HTTP servers on Day 6) click much faster, because you'll recognize the same pattern everywhere instead of learning each one as something new.

---

## 1. `fs` (File System) Module

### Background: why does this module have 3 different "styles" of the same functions?

Node.js evolved over a decade. The `fs` module was written in the **callback era** (2009-2011). When Promises became standard (2015) and async/await became idiomatic (2017+), Node couldn't just delete the old callback-based functions — too much existing code (yours, npm packages, everything) depended on them. So Node kept all three styles **side by side**, and you need to recognize all three when you see them in a codebase:

| Style | Example | When you'll see it |
|---|---|---|
| **Sync** | `fs.readFileSync()` | Startup scripts, CLI tools, config loading — never in request handlers |
| **Callback (async)** | `fs.readFile(path, cb)` | Older codebases, Node's original async API |
| **Promise-based (async)** | `fs.promises.readFile()` or `require('fs/promises')` | Modern code, used with async/await |

```javascript
const fs = require('fs');
const fsPromises = require('fs/promises'); // modern, promise-based API

// --- 1. SYNCHRONOUS ---
// Blocks the entire thread until the file is fully read. NEVER use this
// inside an Express route handler — it would block ALL other concurrent
// requests for the duration of the disk read (Day 2 event loop concept!).
// Acceptable use: reading a config file ONCE at app startup, before the
// server starts accepting traffic — there's nothing else competing for the thread yet.
const config = fs.readFileSync('./config.json', 'utf8');
console.log(JSON.parse(config));

// --- 2. CALLBACK-BASED (legacy async style) ---
// Non-blocking — the thread moves on immediately, Node calls this callback
// once the OS finishes reading the file (delegated to libuv's thread pool).
fs.readFile('./data.txt', 'utf8', (err, data) => {
  if (err) {
    console.error('Read failed:', err);
    return;
  }
  console.log(data);
});

// --- 3. PROMISE-BASED (modern, preferred in new code) ---
// Same non-blocking behavior as the callback version, but plays nicely
// with async/await and Promise.all, etc.
async function readConfig() {
  try {
    const data = await fsPromises.readFile('./data.txt', 'utf8');
    console.log(data);
  } catch (err) {
    console.error('Read failed:', err);
  }
}
readConfig();
```

### Common `fs` operations you'll actually use in backend work

```javascript
const fsPromises = require('fs/promises');

// Write a file (overwrites if it exists)
await fsPromises.writeFile('./output.txt', 'Hello world', 'utf8');

// Append to a file (e.g., a simple log file)
await fsPromises.appendFile('./app.log', 'New log entry\n', 'utf8');

// Check if a file/folder exists (note: fs.exists is DEPRECATED — 
// the modern pattern is to try the operation and catch the error,
// because checking-then-acting has a race condition: the file could be
// deleted between your check and your actual read)
try {
  await fsPromises.access('./somefile.txt');
  console.log('File exists');
} catch {
  console.log('File does NOT exist');
}

// Read a directory's contents
const files = await fsPromises.readdir('./uploads');
console.log(files); // array of filenames

// Get file metadata (size, created/modified time, is it a directory?)
const stats = await fsPromises.stat('./data.txt');
console.log(stats.size, stats.isDirectory(), stats.mtime);

// Delete a file
await fsPromises.unlink('./tempfile.txt');

// Rename / move a file
await fsPromises.rename('./old-name.txt', './new-name.txt');

// Create a directory (recursive: true also creates parent dirs if missing,
// similar to `mkdir -p` in bash)
await fsPromises.mkdir('./uploads/2026/june', { recursive: true });
```

**Interview-relevant production scenario:** *"You're building a file upload feature. What considerations come into play with `fs`?"*
- Use the **async/promise** API, never sync, since this runs inside a request handler.
- Validate file size/type **before** writing to disk (or use `multer`'s built-in limits — this is usually handled via a library, not raw `fs`, in real projects).
- Consider streaming large files instead of loading them fully into memory (this connects directly to Day 5 — Streams).
- Generate unique filenames (e.g., UUID + original extension) to avoid overwrites/collisions.

---

## 2. `path` Module

### Background: why not just use string concatenation for file paths?

```javascript
// THE TRAP: this looks fine but BREAKS on Windows, because Windows uses
// backslashes (\) as path separators, while Linux/Mac use forward slashes (/).
// Hardcoding '/' makes your code behave differently depending on the OS
// it's deployed on — a real, very common cross-platform bug.
const badPath = __dirname + '/uploads/' + filename;
```

The `path` module abstracts away OS-specific path separator differences and gives you safe, predictable ways to build and manipulate file paths.

```javascript
const path = require('path');

// path.join() — the #1 most-used function from this module.
// Correctly joins path segments using the RIGHT separator for the current OS.
const filePath = path.join(__dirname, 'uploads', filename);
// On Linux/Mac: /home/user/project/uploads/photo.png
// On Windows:   C:\Users\user\project\uploads\photo.png
// Same code, correct result on either OS.

// path.resolve() — similar to join, but always returns an ABSOLUTE path,
// resolving from the CURRENT WORKING DIRECTORY if a relative path is given.
// This is different from join(), which does NOT guarantee an absolute path
// if your first segment isn't already absolute.
const absolutePath = path.resolve('uploads', filename);

// path.basename() — extracts just the filename from a full path
console.log(path.basename('/home/user/uploads/photo.png')); // 'photo.png'

// path.extname() — extracts the file extension
console.log(path.extname('photo.png')); // '.png'

// path.dirname() — extracts just the directory portion
console.log(path.dirname('/home/user/uploads/photo.png')); // '/home/user/uploads'

// path.parse() — breaks a path into ALL its components at once
console.log(path.parse('/home/user/uploads/photo.png'));
// {
//   root: '/', dir: '/home/user/uploads', base: 'photo.png',
//   ext: '.png', name: 'photo'
// }
```

**Interview tip:** if asked "why use `path.join` instead of string concatenation," the answer they want is **cross-platform compatibility** — this is a real bug class that has bitten teams deploying Node apps where dev happens on Windows/Mac but production runs on Linux.

---

## 3. `os` Module

### Background: why does a backend engineer care about the OS module?

You rarely use `os` in everyday CRUD route handlers, but it becomes important for **performance tuning, health-check endpoints, logging system diagnostics, and deciding how many worker processes to spawn** (this connects directly to the `cluster` module in Week 3 — you typically spawn one worker process per CPU core, and `os` is how you find out how many cores exist).

```javascript
const os = require('os');

console.log(os.platform());      // 'linux', 'darwin' (mac), 'win32'
console.log(os.cpus().length);   // number of logical CPU cores — used to decide
                                  // how many worker processes to spawn in cluster mode
console.log(os.totalmem());      // total system RAM in bytes
console.log(os.freemem());       // currently free RAM in bytes
console.log(os.uptime());        // system uptime in seconds
console.log(os.hostname());      // machine's hostname

// REAL USE CASE: a basic health-check / diagnostics endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    uptime: process.uptime(),      // how long THIS Node process has run
    freeMemoryMB: Math.round(os.freemem() / 1024 / 1024),
    cpuCount: os.cpus().length,
  });
});
```

---

## 4. `EventEmitter` — the most important pattern today

### Background: the mental model you need first

This is the part of today that's worth slowing down for, because once it clicks, you'll start seeing it everywhere in Node — `http.Server` extends EventEmitter, `stream.Readable` extends EventEmitter, `process` itself emits events (`process.on('exit', ...)`, remember `process.on('unhandledRejection', ...)` from Day 3? — that's an EventEmitter pattern too!).

**The core idea, explained without jargon first:** imagine a radio station and listeners. The station **broadcasts** ("emits") a signal on a particular channel/frequency ("event name"). Anyone with a radio tuned to that frequency ("listener function registered for that event name") receives it and reacts. The station doesn't know or care who's listening, or how many people are listening, or whether anyone is listening at all — it just broadcasts. This decoupling — the broadcaster doesn't need to know about the listeners — is the entire point of the pattern, and it's called the **Observer pattern** in general software design (EventEmitter is Node's specific implementation of it).

**Why this matters more than "just another API":** it lets you build features where different, **unrelated parts of your app** can react to the same occurrence without being directly wired together in code. Example: when a user registers, you might want to (a) send a welcome email, (b) log an analytics event, (c) add them to a CRM. Without EventEmitter, your registration function would need to directly call all three of those things — tightly coupling registration logic to email/analytics/CRM logic. With EventEmitter, your registration function just emits `'user:registered'` and walks away; each of those three concerns can independently listen for that event and do their own thing, without registration code ever knowing they exist.

```javascript
const EventEmitter = require('events');

// Create an instance (or, more commonly in real apps, EXTEND EventEmitter
// in your own class — shown further below)
const emitter = new EventEmitter();

// .on() registers a LISTENER for a named event. You can register multiple
// listeners for the same event — they all get called, in the order registered.
emitter.on('user:registered', (user) => {
  console.log(`Sending welcome email to ${user.email}`);
});

emitter.on('user:registered', (user) => {
  console.log(`Logging analytics event for user ${user.id}`);
});

// .emit() BROADCASTS the event, synchronously calling every registered
// listener IMMEDIATELY, one after another, passing along any extra arguments.
// NOTE: emit() is synchronous! If a listener throws, it can crash the
// emit() call itself unless caught — this surprises people coming from
// other languages where "events" often imply async dispatch by default.
emitter.emit('user:registered', { id: 1, email: 'test@example.com' });

// .once() — listens for the event only ONE time, then automatically removes itself
emitter.once('server:started', () => {
  console.log('This only logs the FIRST time server:started fires');
});

// .off() / .removeListener() — manually unregister a listener
// (important for preventing MEMORY LEAKS in long-running apps — see below)
function logHandler(data) { console.log(data); }
emitter.on('data', logHandler);
emitter.off('data', logHandler);
```

### Building your own EventEmitter-based class (this is what real production code looks like)

```javascript
const EventEmitter = require('events');

// Extending EventEmitter is THE standard pattern for building any
// "service" or "module" in Node that needs to notify other parts of
// your app about things happening inside it, without tight coupling.
class OrderService extends EventEmitter {
  async createOrder(orderData) {
    // ... actual business logic: save to DB, etc. ...
    const order = { id: 123, ...orderData, status: 'created' };

    // Notify the rest of the app that an order was created.
    // OrderService doesn't know or care who's listening — could be
    // an email service, an inventory service, a webhook dispatcher, etc.
    this.emit('order:created', order);

    return order;
  }

  async cancelOrder(orderId) {
    // ... cancellation logic ...
    this.emit('order:cancelled', { orderId });
  }
}

const orderService = new OrderService();

// Elsewhere in your app — completely separate files/modules can listen:
orderService.on('order:created', (order) => {
  console.log(`Send confirmation email for order ${order.id}`);
});

orderService.on('order:created', (order) => {
  console.log(`Decrement inventory for order ${order.id}`);
});

orderService.on('order:cancelled', ({ orderId }) => {
  console.log(`Restore inventory for cancelled order ${orderId}`);
});

orderService.createOrder({ item: 'Laptop', qty: 1 });
```

### Error handling with EventEmitter — a sharp edge interviewers like to test

```javascript
const emitter = new EventEmitter();

// CRITICAL RULE: if you emit('error', ...) and there is NO listener
// registered for the 'error' event specifically, Node will THROW that
// error and CRASH your process. This is a special-cased behavior unique
// to the 'error' event name — no other event name behaves this way.
emitter.emit('error', new Error('Something broke')); // CRASHES the process if unhandled!

// THE FIX: always register an 'error' listener on any EventEmitter
// (or subclass of it) that might emit errors — this is a very real,
// very common production bug source, especially with streams (Day 5)
// and database connections, which are built on EventEmitter internally.
emitter.on('error', (err) => {
  console.error('Handled gracefully:', err.message);
});
emitter.emit('error', new Error('Something broke')); // now handled safely
```

### Memory leak warning — another real interview topic

```javascript
const emitter = new EventEmitter();

// By default, Node warns you (via console) if you register MORE than
// 10 listeners for the SAME event on the SAME emitter — this is a
// built-in heuristic to catch a common bug: forgetting to remove
// listeners in a loop, or re-registering a listener every time a
// function runs (e.g., inside an Express route handler by mistake),
// which causes listeners to pile up endlessly over the life of the app
// — a genuine memory leak in long-running production servers.
for (let i = 0; i < 15; i++) {
  emitter.on('data', () => {}); 
}
// Node logs: "MaxListenersExceededWarning: Possible EventEmitter memory leak detected"

// You CAN raise this limit if you genuinely need more listeners
// (sometimes legitimate in larger apps), but it's more often a signal
// that listeners aren't being cleaned up properly.
emitter.setMaxListeners(20);
```

---

## 5. How this connects to real backend work (3 YOE framing)

- **"How would you decouple your order creation logic from sending notifications?"** → EventEmitter pattern, exactly as shown above — emit an event, let separate listener modules handle side effects.
- **"You noticed your app's memory usage grows slowly over days of uptime — where would you look?"** → A classic culprit is EventEmitter listeners being added repeatedly without ever being removed (e.g., adding a `.on()` inside a function that runs on every request instead of once at startup).
- **"Why did your app crash with an unhandled error even though you thought you handled errors?"** → Check if it's an `emitter.emit('error', ...)` with no `'error'` listener attached — this is a very specific, very real Node.js gotcha distinct from a typical thrown exception.
- **"How would you build a config loader that reads a JSON file at startup?"** → `fs.readFileSync` is actually CORRECT here (not an anti-pattern) since it runs once before the server starts handling any requests — show you understand sync isn't always wrong, context matters.

---

## 6. Hands-on practice for today

```javascript
// practice-day4.js
const EventEmitter = require('events');
const fs = require('fs/promises');
const path = require('path');

class Logger extends EventEmitter {
  async log(message) {
    const logLine = `[${new Date().toISOString()}] ${message}\n`;
    const logPath = path.join(__dirname, 'app.log');

    try {
      await fs.appendFile(logPath, logLine, 'utf8');
      this.emit('logged', logLine.trim());
    } catch (err) {
      this.emit('error', err);
    }
  }
}

const logger = new Logger();

// ALWAYS register an error listener — remember the crash rule above!
logger.on('error', (err) => {
  console.error('Logger failed:', err.message);
});

logger.on('logged', (line) => {
  console.log('Successfully wrote:', line);
});

async function main() {
  await logger.log('Server started');
  await logger.log('User 42 logged in');
}

main();
```

Run it, check that `app.log` gets created in the same directory with your two log lines, then try intentionally breaking the path (e.g., point it to a folder that doesn't exist) to watch your `'error'` listener catch it gracefully instead of crashing.

---

## 7. Interview Q&A Recap (say these out loud)

1. **Q: Why does `fs` have sync, callback, and promise-based versions of the same functions?**
   A: Node evolved over many years — callbacks were the original (only) async pattern in Node, and sync versions existed for simple scripting use cases. Promise-based versions were added once Promises/async-await became the JS standard. All three are kept for backward compatibility; modern code should default to the promise-based API except for one-time startup scripts.

2. **Q: Why use `path.join()` instead of string concatenation for file paths?**
   A: Cross-platform compatibility — Windows uses backslashes, Linux/Mac use forward slashes. `path.join()` (and `path.resolve()`) handles this automatically regardless of the OS the code runs on.

3. **Q: Explain EventEmitter and the Observer pattern in your own words.**
   A: It lets one part of code "broadcast" that something happened (`emit`) without knowing or caring who's listening, while other independent parts of the app "subscribe" (`on`) to react to it. This decouples the code that triggers an action from the code that reacts to it.

4. **Q: What happens if you emit an 'error' event with no listener attached?**
   A: Node treats this as a special case — it throws the error and crashes the process, unlike any other event name. Always attach an `'error'` listener to any EventEmitter that might emit one.

5. **Q: What does the "MaxListenersExceededWarning" mean and what usually causes it?**
   A: More than 10 listeners were registered for the same event on the same emitter — usually a sign that listeners are being added repeatedly without cleanup (e.g., inside a function that runs many times), which is a real memory leak risk in long-running servers.

6. **Q: Is `emit()` synchronous or asynchronous?**
   A: Synchronous — all registered listeners for an event run immediately, one after another, in the same tick, in the order they were registered.

---

## Tomorrow (Day 5) Preview
We cover **Streams and Buffers** — readable, writable, duplex, and transform streams, plus the concept of **backpressure**. This builds directly on today's `fs` and `EventEmitter` knowledge, since streams are EventEmitters under the hood, and you'll see how Node handles large files (videos, big CSVs) WITHOUT loading them entirely into memory — a very common interview and real-world performance topic.
