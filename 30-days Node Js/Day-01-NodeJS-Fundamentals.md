# Day 1 — Node.js Fundamentals (Runtime, V8, REPL, Modules)

> Goal for today: understand **what Node.js actually is**, **how it runs your code**, and **how modules work (CommonJS vs ES Modules)** — well enough to explain it confidently to an interviewer.

---

## 1. What is Node.js?

**What it is:**
Node.js is **not a language** — JavaScript is the language. Node.js is a **runtime environment** that lets you run JavaScript **outside the browser** (e.g., on a server, your laptop, a Raspberry Pi).

It is built on:
- **V8 engine** (Google's JavaScript engine, also used in Chrome) — compiles JS to machine code.
- **libuv** — a C library that gives Node its **event loop**, **async I/O**, and **thread pool**.
- **Node bindings** — C++ code that connects JS to system-level features (file system, networking, etc.) that V8 alone cannot do.

**Why it matters (interview angle):**
Interviewers ask "What is Node.js?" to check if you understand it's a *runtime*, not a *framework* or *language*. A common wrong answer is "Node.js is a JavaScript framework" — that's incorrect and will hurt you immediately.

**Why Node.js is popular for backend:**
- Non-blocking, event-driven I/O → handles many concurrent connections with a single thread efficiently (great for I/O-heavy apps: APIs, chat apps, streaming).
- Same language (JS) on frontend and backend.
- Huge npm ecosystem.

**Where it's NOT ideal:**
- CPU-intensive tasks (heavy computation, image processing, video encoding) — because Node is single-threaded for JS execution, a long CPU-bound task blocks the event loop. (We solve this later with Worker Threads / child processes — Week 3.)

---

## 2. Node.js Architecture (High-Level)

```
Your JS Code
     │
     ▼
 V8 Engine  ───► Compiles & executes JS
     │
     ▼
Node.js APIs (fs, http, etc.) ───► Bindings (C++) 
     │
     ▼
   libuv ───► Event Loop + Thread Pool ───► OS (disk, network, etc.)
```

- **Single thread** runs your JS code (the "main thread").
- **libuv's thread pool** (default size 4) handles blocking operations like file I/O, DNS lookups, and some crypto operations **in the background**, so the main thread isn't blocked.
- When the background work finishes, the **event loop** picks up the result and runs your callback.

**Interview tip:** If asked "Is Node.js single-threaded?" — the correct nuanced answer is:
> "JavaScript execution in Node.js is single-threaded, but Node itself is not single-threaded — it uses a thread pool (via libuv) internally for certain blocking operations like file system access, DNS, and some crypto functions."

---

## 3. REPL (Read-Eval-Print-Loop)

**What it is:** An interactive shell to run JS line by line. Start it by typing `node` in your terminal with no file name.

**Why it matters:** Quick testing of snippets, debugging small logic, checking Node version behavior — useful in day-to-day dev work, and interviewers sometimes ask what REPL stands for.

```bash
$ node
> const x = 10;
undefined
> x * 2
20
> .exit
```

- **Read** — reads user input, parses it into JS data structure
- **Eval** — evaluates the parsed input
- **Print** — prints the result
- **Loop** — loops back to read again

---

## 4. npm and npx

**npm (Node Package Manager):**
- Installs, manages, and publishes JS packages.
- `package.json` — manifest file: project metadata, dependencies, scripts.
- `package-lock.json` — locks exact dependency versions for reproducible installs across machines.

**npx:**
- Executes a package **without installing it globally**. Example: `npx create-react-app myapp` downloads and runs the package temporarily.

```bash
npm init -y                  # create package.json with defaults
npm install express          # install & save to dependencies
npm install --save-dev jest  # install & save to devDependencies (not needed in production)
npm install                  # installs everything listed in package.json
npm run start                # runs the "start" script defined in package.json
```

**Interview tip:** Know the difference between `dependencies` and `devDependencies`, and what `npm ci` does (clean install strictly from package-lock.json — used in CI/CD pipelines for reproducibility, faster and stricter than `npm install`).

---

## 5. CommonJS vs ES Modules — THE topic interviewers love

This is one of the **most commonly asked Node.js topics** at 2-3 years experience level. Know it cold.

### CommonJS (CJS) — Node's original module system

```javascript
// math.js
// 'module.exports' is how CommonJS exposes things from this file to others
function add(a, b) {
  return a + b;
}

function subtract(a, b) {
  return a - b;
}

// Exporting multiple functions as an object
module.exports = { add, subtract };
```

```javascript
// app.js
// 'require' is CommonJS's way of importing. It's SYNCHRONOUS — 
// Node reads and executes the file immediately, blocking until done.
const { add, subtract } = require('./math.js');

console.log(add(5, 3)); // 8
```

**Key facts about CommonJS:**
- `require()` is **synchronous** and **cached** — calling `require('./math.js')` multiple times returns the same cached module object (the file runs only once).
- Module wrapping: Node secretly wraps every CJS file in a function like this:
  ```javascript
  (function(exports, require, module, __filename, __dirname) {
    // your code here
  });
  ```
  This is **why** `require`, `module`, `exports`, `__filename`, `__dirname` are available in every file without importing them — they're injected as function parameters by Node.

### ES Modules (ESM) — the modern JavaScript standard

```javascript
// math.mjs  (or math.js with "type": "module" in package.json)
// 'export' is the ESM way of exposing values. Can be named or default.
export function add(a, b) {
  return a + b;
}

export function subtract(a, b) {
  return a - b;
}
```

```javascript
// app.mjs
// 'import' is ESM's way of importing — it's ASYNCHRONOUS under the hood
// (the module graph is resolved before execution begins).
import { add, subtract } from './math.mjs';

console.log(add(5, 3)); // 8
```

**How to enable ESM in Node:**
- Use `.mjs` file extension, OR
- Add `"type": "module"` in `package.json` (then `.js` files are treated as ESM)

### Key Differences (this is the table interviewers expect you to recite)

| Feature | CommonJS | ES Modules |
|---|---|---|
| Syntax | `require()` / `module.exports` | `import` / `export` |
| Loading | Synchronous | Asynchronous (resolved at parse time) |
| `this` at top level | refers to `module.exports` | `undefined` |
| `__dirname` / `__filename` | Available by default | NOT available (use `import.meta.url` instead) |
| Tree-shaking (dead code elimination) | Not supported well | Supported (static structure helps bundlers) |
| Dynamic import | `require()` anywhere, conditionally | `import()` (returns a Promise) for dynamic imports |
| File extension | `.js` (default) | `.mjs` or `.js` with `"type": "module"` |

**Why does this matter in real projects?**
Most production Node backends (especially 2-3 yrs old codebases) still use CommonJS, but ESM adoption is growing because it's the JS standard (used in browsers too) and supports static analysis better. You should be comfortable reading and writing both.

**Tricky interview question:** *"Can you `require()` an ES Module, or `import` a CommonJS module?"*
> CommonJS can dynamically `import()` an ESM module (since `import()` returns a Promise).
> ESM can `import` a CommonJS module's default export pretty seamlessly, but named exports interop can be inconsistent depending on how the CJS module set `module.exports`.

---

## 6. `__dirname`, `__filename`, `process`, `global`

```javascript
// These exist only in CommonJS by default
console.log(__filename); // full path to the current file
console.log(__dirname);  // full path to the current directory

// 'process' is a GLOBAL object available in BOTH CJS and ESM
// It gives info about and control over the current Node.js process
console.log(process.argv);     // command-line arguments passed to the script
console.log(process.env);      // environment variables (e.g., process.env.NODE_ENV)
console.log(process.platform); // 'win32', 'linux', 'darwin', etc.
console.log(process.version);  // Node version running this script

// process.exit(code) — manually exits the process
// 0 = success, non-zero = some kind of failure (used in scripts/CI)
process.exit(0);
```

**Interview tip:** `process.env` is heavily used in real jobs for config (`NODE_ENV`, `PORT`, `DB_URL`, secrets). Mention that hardcoding secrets is bad practice — use environment variables + a `.env` file (with the `dotenv` package) instead, and never commit `.env` to git.

---

## 7. Mini Practice Task for Today

Create two files to lock in CJS vs ESM:

**`utils.js` (CommonJS):**
```javascript
// A simple utility module using CommonJS
function greet(name) {
  return `Hello, ${name}!`;
}

module.exports = { greet };
```

**`index.js` (CommonJS):**
```javascript
const { greet } = require('./utils.js');

console.log(greet('Node learner')); // Hello, Node learner!
console.log(__dirname);             // prints current folder path
console.log(process.version);       // prints installed Node version
```

Run it:
```bash
node index.js
```

Then convert the same thing to ESM (`utils.mjs` + `index.mjs` using `export`/`import`) and run with `node index.mjs` to feel the difference yourself.

---

## 8. Interview Q&A Recap (explain these out loud to yourself)

1. **Q: What is Node.js?**
   A: A JavaScript runtime built on Chrome's V8 engine, using libuv for non-blocking, event-driven I/O — lets us run JS on the server.

2. **Q: Is Node.js single-threaded?**
   A: JS execution is single-threaded (one main thread, one call stack), but Node uses a libuv thread pool internally for certain blocking operations, so it's not "purely" single-threaded at the system level.

3. **Q: Difference between CommonJS and ES Modules?**
   A: (Recite the table above — sync vs async loading, `require`/`module.exports` vs `import`/`export`, `__dirname` availability, tree-shaking support.)

4. **Q: What's the difference between `dependencies` and `devDependencies`?**
   A: `dependencies` are needed at runtime in production; `devDependencies` are only needed during development/testing (e.g., testing libraries, linters) and aren't installed when running `npm install --production`.

5. **Q: Why is Node good for I/O-heavy apps but not CPU-heavy apps?**
   A: Non-blocking I/O lets one thread juggle many concurrent connections efficiently. But a CPU-heavy synchronous task occupies the single JS thread, blocking the event loop and freezing all other requests until it finishes.

---

## Tomorrow (Day 2) Preview
We go deep into the **Event Loop** — phases, microtasks vs macrotasks, `setTimeout` vs `setImmediate` vs `process.nextTick`. This is the single most-asked Node.js topic in interviews, so we'll spend a full day on it with diagrams and code traces.
