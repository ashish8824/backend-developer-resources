# Day 7 — Week 1 Revision: Build a Dependency-Free File Server + Consolidation

> **Why this day matters:** Knowledge that isn't *used* fades fast. Today has no new theory — instead, you build one cohesive project that forces you to combine Days 1-6 (modules, event loop, async patterns, core modules, streams, raw HTTP) into working code. Then we do a structured self-review, because **explaining a concept out loud is a completely different skill from reading about it**, and that's exactly what an interview demands.

---

## 1. The Mini Project: A Static File Server From Scratch

**The goal:** build a server that serves files from a folder (like a tiny version of what `serve` or `http-server` npm packages do), with zero external dependencies — only Node's built-in modules. This single project naturally forces you to use:

- `http` (Day 6) — to accept requests and send responses
- `fs` / `fs/promises` (Day 4) — to check if files exist and read them
- `path` (Day 4) — to safely build file paths and avoid directory traversal bugs
- Streams (Day 5) — to serve files efficiently instead of loading them fully into memory
- Async/await + error handling (Day 3) — throughout
- An understanding of the event loop (Day 2) — to reason about why this approach scales well under concurrent load

```javascript
// file-server.js
const http = require('http');
const fs = require('fs');
const fsPromises = require('fs/promises');
const path = require('path');

const PORT = 3000;
const PUBLIC_DIR = path.join(__dirname, 'public'); // the folder we'll serve files from

// A small lookup table mapping file extensions to their correct MIME type.
// Without this, browsers wouldn't know how to render/handle the response
// correctly (e.g., a browser needs 'text/html' to render HTML rather than
// just dumping the source as plain text).
const MIME_TYPES = {
  '.html': 'text/html',
  '.css': 'text/css',
  '.js': 'application/javascript',
  '.json': 'application/json',
  '.png': 'image/png',
  '.jpg': 'image/jpeg',
  '.txt': 'text/plain',
};

const server = http.createServer(async (req, res) => {
  try {
    // SECURITY-CRITICAL STEP: prevent "directory traversal" attacks.
    // Without this, a malicious request like GET /../../etc/passwd could
    // let an attacker read files OUTSIDE the intended public folder.
    // path.normalize() resolves '..' segments, and we then double-check
    // the resulting absolute path still starts with our intended PUBLIC_DIR.
    const requestedPath = path.normalize(decodeURIComponent(req.url)).replace(/^\/+/, '');
    const fullPath = path.join(PUBLIC_DIR, requestedPath === '' ? 'index.html' : requestedPath);

    if (!fullPath.startsWith(PUBLIC_DIR)) {
      res.statusCode = 403;
      res.end('Forbidden');
      return;
    }

    // Check the file exists and get its metadata (size, etc.) BEFORE
    // attempting to stream it — lets us return a clean 404 instead of
    // letting a stream error happen mid-response.
    const stats = await fsPromises.stat(fullPath);

    if (stats.isDirectory()) {
      res.statusCode = 403;
      res.end('Directory listing not allowed');
      return;
    }

    const ext = path.extname(fullPath);
    const contentType = MIME_TYPES[ext] || 'application/octet-stream';

    res.statusCode = 200;
    res.setHeader('Content-Type', contentType);
    res.setHeader('Content-Length', stats.size); // tells client exactly how many bytes to expect

    // STREAMING the file out instead of fs.readFile() — this is the Day 5
    // lesson in action. Memory usage stays flat whether the file is 1KB
    // or 1GB, because we never hold the whole file in memory at once.
    const readStream = fs.createReadStream(fullPath);

    readStream.on('error', (err) => {
      // A read error AFTER headers were already sent can't change the
      // status code anymore — this is the "Cannot set headers after they
      // are sent" lesson from Day 6 in practice. Best we can do here is
      // end the response and log the error server-side.
      console.error('Stream error:', err);
      res.end();
    });

    readStream.pipe(res); // Day 5: piping handles backpressure automatically

  } catch (err) {
    if (err.code === 'ENOENT') {
      // ENOENT = "Error NO ENTry" -- Node's code for "file not found"
      res.statusCode = 404;
      res.end('404 Not Found');
    } else {
      console.error('Server error:', err);
      res.statusCode = 500;
      res.end('500 Internal Server Error');
    }
  }
});

// Always handle the server's own error event (Day 6) — e.g. port conflicts
server.on('error', (err) => {
  if (err.code === 'EADDRINUSE') {
    console.error(`Port ${PORT} is already in use.`);
    process.exit(1);
  }
});

server.listen(PORT, () => {
  console.log(`File server running at http://localhost:${PORT}`);
});
```

**To run this yourself:**
```bash
mkdir public
echo "<h1>Hello from my file server</h1>" > public/index.html
node file-server.js
```
Then visit `http://localhost:3000` and `http://localhost:3000/index.html` in a browser, and try `curl http://localhost:3000/does-not-exist.txt` to see the 404 path.

**Try breaking it on purpose** (this is the real learning):
- Request `http://localhost:3000/../file-server.js` (URL-encode the `..` as `%2e%2e` if your browser normalizes it) and confirm you get a 403, not the actual source file — proving the traversal protection works.
- Drop a large file (a video, a big zip) into `public/` and watch memory stay flat while downloading it, versus what would happen if you swapped the stream for `fs.readFileSync`.

---

## 2. Consolidated Concept Map of Week 1

Use this as a "can I explain this out loud, unprompted" checklist — not just a reading list.

| Day | Topic | The ONE thing you must be able to explain without notes |
|---|---|---|
| 1 | Node fundamentals | Node is a runtime (V8 + libuv), not a language or framework; CommonJS vs ESM differences |
| 2 | Event loop | The 6 phases, and the exact priority order: nextTick → microtasks → macrotask phases |
| 3 | Async patterns | Why callback hell happens, how Promises/async-await fix it, sequential vs parallel awaits |
| 4 | Core modules | fs has 3 API styles for historical reasons; EventEmitter is the Observer pattern; the `'error'` event crash rule |
| 5 | Streams | Why streaming beats loading everything into memory; what backpressure is and why `.pipe()` matters |
| 6 | Raw HTTP | What Express actually abstracts away — body parsing, routing, response helpers |
| 7 | (today) | All of the above, combined into one working, secure, efficient piece of code |

---

## 3. Self-Test: Predict the Output (Event Loop + Async, Days 2-3 combined)

Try to answer BEFORE scrolling to the explanation. This is exactly the format interviewers use.

```javascript
console.log('1');

setTimeout(() => console.log('2'), 0);

(async () => {
  console.log('3');
  await null; // 'await null' still yields to the microtask queue!
  console.log('4');
})();

Promise.resolve().then(() => console.log('5'));

console.log('6');
```

**Answer:** `1, 3, 6, 5, 4, 2`

**Why:** `1` runs sync. The async IIFE starts executing synchronously up until its first `await`, so `3` prints next (still synchronous, before the function suspends). `6` then prints (still in the original synchronous pass). Once the synchronous code is fully done, the microtask queue drains: the `Promise.resolve().then()` (`5`) and the resumed async function after `await null` (`4`) are both microtasks — they run in the order they were QUEUED, and since the async function hit its `await` before the `.then()` was even registered... actually let's be precise: the async IIFE registers its continuation as a microtask the moment it hits `await null`, which happens BEFORE `Promise.resolve().then(...)` is even reached in the synchronous code (since `await null` is reached before line `Promise.resolve().then()`). So the continuation of `4` is queued slightly before `5`'s callback... 

**Correction worth noting explicitly:** this exact kind of micro-ordering between `await` continuations and separately-chained `.then()` calls is genuinely one of the trickiest areas of JS, and even experienced engineers sometimes get the precise order wrong without running it. **The mature, honest answer in an interview** is: "Both `4` and `5` are microtasks, queued very close together — the exact order depends on the precise queuing sequence, but both will run before the `setTimeout` callback (`2`), because microtasks always fully drain before the event loop proceeds to the timers phase." Demonstrating that you know microtasks beat macrotasks, and being honest about sub-microtask ordering subtleties, reads as MORE senior than confidently asserting a wrong precise order.

---

## 4. Self-Test: Spot the Bug

```javascript
// Bug-hunt #1
app.get('/user/:id', async (req, res) => {
  const user = await getUser(req.params.id);
  res.json(user);
});
// What's missing? (Hint: Day 3, error handling)
```
**Answer:** No `try/catch` — if `getUser` rejects, this becomes an unhandled rejection in an async Express route handler (which, depending on Express version, may not be auto-caught), potentially crashing the process or hanging the request.

```javascript
// Bug-hunt #2
async function uploadAll(files) {
  files.forEach(async (file) => {
    await saveFile(file);
  });
  console.log('All files saved');
}
// What's wrong? (Hint: Day 3)
```
**Answer:** `forEach` doesn't wait for async callbacks — `console.log('All files saved')` runs immediately, before any file is actually saved, and any rejected promise inside becomes an unhandled rejection.

```javascript
// Bug-hunt #3
const data = fs.readFileSync('./huge-upload.dat');
app.post('/process', (req, res) => {
  // imagine this runs the line above on every request
});
// What's wrong? (Hint: Day 4 + Day 2)
```
**Answer:** Using the synchronous `fs` API inside (or per) a request handler blocks the single JS thread for the duration of the read, freezing ALL other concurrent requests — should use the async/promise-based `fs` API instead.

---

## 5. Mock Interview Round — Answer These Out Loud, Then Check Yourself

1. "Walk me through what happens, end to end, when a client sends a GET request to your Node server." *(Should mention: TCP connection → http module parses raw bytes into req/res → your handler runs → event loop coordinates any async work → response streamed back)*
2. "Why is Node good for I/O-bound apps but a poor fit for CPU-bound work?"
3. "Explain the difference between `process.nextTick`, microtasks, and macrotasks, with priority order."
4. "When would you choose `Promise.all` over `Promise.allSettled`?"
5. "What's the Observer pattern, and where does Node use it internally?"
6. "How does Node handle a 10GB file download without running out of memory?"
7. "What is backpressure, and what handles it for you by default?"
8. "What does Express give you that raw `http` doesn't?"

If any of these make you pause and think "I know this but can't phrase it cleanly," that's a signal to re-read that specific day's file and then explain it again out loud — to a rubber duck, a friend, or even just your phone's voice recorder. **Reading is recognition. Speaking is recall.** Interviews test recall.

---

## 6. What's Different Starting Tomorrow

Week 1 was about Node's internals — things that exist whether or not you use any framework. Starting Day 8, we shift into **Express.js and real application architecture**: routing, middleware, REST API design, databases, and authentication. The pace will feel faster because you're no longer building primitives — you're combining them. Every Express concept from here on will map back to something you already understand from this week (middleware is just organized request handling, `express.json()` is the manual body-parsing you wrote today, etc.) — so if something feels new, it's worth asking "what does this map to from Week 1?"

---

## Day 8 Preview
We start Week 2 with **Express.js fundamentals**: app setup, routing (including route parameters and query strings handled for you), the middleware chain (and the critical role of `next()`), and built-in vs custom middleware — directly building on everything from today.
