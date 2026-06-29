# Day 8 — Express.js Fundamentals: Routing, Middleware, and the Request Lifecycle

> **Why this day matters:** This is where Week 1 pays off. Every single thing Express does today is something you already manually built on Day 6 and Day 7 — routing, body parsing, response helpers. The difference is Express organizes it into a clean, composable system. If you understand **middleware and the role of `next()`** deeply, you'll be able to debug almost any "my route isn't working" or "my middleware isn't running" production issue, which is a very real, very common day-to-day Express problem.

---

## 1. Background: what problem is Express actually solving?

Remember Day 6/7 — manual routing with a growing chain of `if/else if` statements, and manually parsing `req.url` and the request body stream. That approach has three concrete problems as an app grows:

1. **No clean way to extract URL parameters** (`/users/:id/orders/:orderId`) without writing fragile string-splitting code yourself.
2. **No way to share logic across routes** (e.g., "check if the user is authenticated" needs to run before 10 different routes) without copy-pasting that check into every handler.
3. **No structured way to handle errors centrally** — every route handler needs its own try/catch and its own way of formatting an error response.

Express solves all three with one core idea: **everything is a function that receives `(req, res, next)`, and these functions run in a defined order, each one able to pass control to the next.** Once that one idea clicks, routing, middleware, and error handling all stop looking like separate topics — they're the same mechanism applied in different ways.

---

## 2. Setting Up a Basic Express App

```javascript
const express = require('express');
const app = express(); // creates an Express application instance

// express.json() is BUILT-IN middleware (added to Express core in v4.16+,
// previously required the separate 'body-parser' package). It does
// EXACTLY what you manually wrote on Day 6: listens for 'data'/'end'
// events on the request stream, accumulates the body, and parses it as
// JSON, attaching the result to req.body — but now you get it for free.
app.use(express.json());

// A basic route: app.METHOD(path, handler)
app.get('/', (req, res) => {
  res.send('Hello from Express!'); // res.send() is a convenience wrapper
                                    // around res.end() that also sets
                                    // appropriate headers automatically
                                    // based on what you pass it
});

app.listen(3000, () => {
  console.log('Server running on http://localhost:3000');
});
```

**Interview tip:** if asked "what is Express, in one sentence" — say: *"A minimal, unopinionated web framework built on top of Node's `http` module, providing routing, middleware composition, and convenience methods around the raw request/response objects."* The word "unopinionated" matters — Express deliberately doesn't force a project structure, ORM, or templating engine on you, unlike more opinionated frameworks (e.g., NestJS, Django). This is often asked as "Express vs NestJS" — know that distinction.

---

## 3. Routing — and what Express gives you over raw Node

```javascript
// Route parameters — this is the BIG upgrade over Day 6/7's manual
// string-splitting approach. Express parses ':id' style segments
// automatically and gives them to you as a clean object.
app.get('/users/:id', (req, res) => {
  res.send(`User ID is ${req.params.id}`);
});

// Multiple parameters in one route
app.get('/users/:userId/orders/:orderId', (req, res) => {
  const { userId, orderId } = req.params;
  res.send(`User ${userId}, Order ${orderId}`);
});

// Query strings — automatically parsed into req.query (remember Day 6:
// in raw Node you had to manually use the URL class for this!)
app.get('/search', (req, res) => {
  // GET /search?q=node&page=2
  const { q, page } = req.query;
  res.json({ searchTerm: q, page: page || 1 });
});

// All the standard HTTP verbs are supported as methods on 'app'
app.post('/users', (req, res) => {
  // req.body is available here BECAUSE we registered express.json() above
  const newUser = req.body;
  res.status(201).json(newUser);
});

app.put('/users/:id', (req, res) => {
  res.json({ id: req.params.id, ...req.body });
});

app.delete('/users/:id', (req, res) => {
  res.status(204).end(); // 204 No Content -- standard for successful deletes
                          // with no body to return
});

// app.all() matches EVERY HTTP method for a given path — rarely used for
// real routes, but common for logging/debugging a specific path
app.all('/admin', (req, res, next) => {
  console.log(`${req.method} request to /admin`);
  next();
});
```

### `express.Router()` — organizing routes into modules (real production pattern)

**Background:** in a real app with 50+ routes, putting everything in one file (`app.js`) becomes unmanageable. `express.Router()` lets you group related routes (e.g., all `/users/*` routes) into their own file, then "mount" that group onto the main app.

```javascript
// routes/users.js
const express = require('express');
const router = express.Router(); // a mini, self-contained Express app

router.get('/', (req, res) => {
  res.send('List all users');
});

router.get('/:id', (req, res) => {
  res.send(`Get user ${req.params.id}`);
});

router.post('/', (req, res) => {
  res.status(201).send('Create user');
});

module.exports = router;
```

```javascript
// app.js
const express = require('express');
const userRoutes = require('./routes/users');

const app = express();
app.use(express.json());

// Mounting: every route inside userRoutes is now PREFIXED with '/users'.
// So router.get('/:id') above actually becomes GET /users/:id in the
// real running app. This is the standard way production codebases
// organize routes by resource/domain (users, orders, products, etc.)
app.use('/users', userRoutes);

app.listen(3000);
```

---

## 4. Middleware — the single most important Express concept

### Background: what IS middleware, really?

**The precise definition:** middleware is simply a function with the signature `(req, res, next)` that sits **in between** the incoming request and the final route handler, with the ability to:
- Inspect or modify `req`/`res`
- End the request-response cycle itself (e.g., reject unauthorized requests)
- Pass control forward to the NEXT function in the chain by calling `next()`

**The mental model that makes this click:** think of middleware as a series of **checkpoints** a request must pass through, like airport security lines, before it reaches its final destination (your route handler). Each checkpoint can let the request through (`next()`), reject it entirely (`res.send()`/`res.status().json()` and NOT calling `next()`), or even just observe and log something before letting it continue.

```javascript
const express = require('express');
const app = express();

// A simple logging middleware — runs for EVERY request, on EVERY route,
// because it's registered with app.use() with no path restriction.
app.use((req, res, next) => {
  console.log(`${new Date().toISOString()} - ${req.method} ${req.url}`);
  next(); // CRITICAL: without calling next(), the request hangs forever —
          // it never reaches the actual route handler, and the client's
          // request will eventually time out with no response at all.
});

app.get('/', (req, res) => {
  res.send('Home');
});

app.listen(3000);
```

### The Order of Middleware Registration MATTERS — a real bug source

```javascript
const express = require('express');
const app = express();

// Middleware and routes run in the EXACT order they're registered.
// This is one of the most common sources of confusing bugs for
// developers newer to Express: a middleware registered AFTER a route
// will NEVER run for requests matching that route, because the route
// already sent a response and ended the cycle.

app.get('/test', (req, res) => {
  res.send('Test route');
});

// This middleware is USELESS for the /test route above — it was
// registered AFTER that route already handled and ended the request.
app.use((req, res, next) => {
  console.log('This NEVER logs for GET /test');
  next();
});
```

**The fix — and the standard convention:** register global middleware (logging, auth checks, body parsing) **before** your routes.

```javascript
app.use(loggingMiddleware);   // 1st: runs for every request
app.use(express.json());      // 2nd: parses body for every request
app.use('/users', userRoutes); // 3rd: route-specific logic
app.use(errorHandler);         // LAST: catches errors from everything above
```

### Custom Authentication Middleware (a real-world example you'll actually write at work)

```javascript
// middleware/auth.js
function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1]; // "Bearer <token>"

  if (!token) {
    // Notice: we call res.status().json() and do NOT call next() here.
    // This DELIBERATELY ends the request-response cycle right here —
    // the request never reaches whatever route handler was after this.
    return res.status(401).json({ error: 'No token provided' });
  }

  try {
    const decoded = verifyToken(token); // imagine this validates a JWT (Day 12 topic)
    req.user = decoded; // ATTACHING data to req so LATER middleware/handlers can use it —
                         // this is a core middleware pattern: enriching req as it flows through
    next(); // token is valid, let the request continue to the actual route
  } catch (err) {
    return res.status(401).json({ error: 'Invalid token' });
  }
}

module.exports = requireAuth;
```

```javascript
// app.js
const requireAuth = require('./middleware/auth');

// Applying middleware to a SPECIFIC route only (not globally) — this is
// the second way to register middleware: as an extra argument to a
// specific route, rather than with app.use().
app.get('/profile', requireAuth, (req, res) => {
  // By the time we get here, requireAuth already ran and either:
  // (a) rejected the request and we'd never reach this line, or
  // (b) called next() and attached req.user for us to use
  res.json({ profile: req.user });
});

// You can also chain MULTIPLE middleware functions for one route
app.post('/admin/delete-user/:id', requireAuth, requireAdminRole, (req, res) => {
  res.send('User deleted');
});
```

### Built-in Middleware You'll Use Constantly

```javascript
app.use(express.json());                       // parses JSON request bodies into req.body
app.use(express.urlencoded({ extended: true })); // parses HTML form submissions into req.body
app.use(express.static('public'));               // serves static files (images, CSS, HTML) from a folder
```

### Third-Party Middleware (you'll see these in almost every real production Express app)

```javascript
const helmet = require('helmet');     // sets various security-related HTTP headers
const cors = require('cors');         // handles Cross-Origin Resource Sharing headers
const morgan = require('morgan');     // HTTP request logging (more featureful than a custom logger)

app.use(helmet());
app.use(cors());
app.use(morgan('dev'));
```

---

## 5. Error-Handling Middleware — a special case worth understanding now

**Background:** Express has a special category of middleware specifically for errors — it's identified not by where it's registered, but by its **function signature having 4 parameters instead of 3**: `(err, req, res, next)`. Express detects this signature and treats it differently — it's only invoked when an error is passed to `next(err)`, or when a synchronous error is thrown inside a regular route handler.

```javascript
app.get('/risky', (req, res, next) => {
  try {
    throw new Error('Something broke');
  } catch (err) {
    next(err); // passing an error to next() SKIPS all remaining regular
               // middleware/routes and jumps STRAIGHT to error-handling middleware
  }
});

// Error-handling middleware MUST be registered LAST, after all routes —
// Express identifies it by the 4-argument signature.
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong on our end' });
});
```

We'll go much deeper into centralized error handling patterns on Day 13 — for today, just understand that `next(err)` (with an argument) behaves completely differently from `next()` (with no argument): one continues normally to the next regular middleware, the other jumps straight to error-handling middleware.

---

## 6. How this connects to real backend work (3 YOE framing)

- **"A teammate added new middleware but says it's not running for some routes — what would you check first?"** → Registration order. Middleware registered after a route that already sends a response will never run for that route.
- **"How would you protect 15 different admin routes without repeating auth logic 15 times?"** → `express.Router()` with `router.use(requireAuth)` applied to that entire router, or apply `requireAuth` once via `app.use('/admin', requireAuth, adminRouter)`.
- **"Your route handler isn't getting `req.body` — what are the likely causes?"** → `express.json()` not registered (or registered AFTER the route), or the client didn't send a `Content-Type: application/json` header, or it's registered but the route is mounted on a router file that doesn't inherit middleware applied only to a different router.
- **"What's the difference between `next()` and `next(err)`?"** → `next()` passes control to the next matching middleware/route in normal sequence. `next(err)` skips all of that and jumps directly to error-handling middleware (4-arg signature), used for centralized error handling.

---

## 7. Hands-on practice for today

```javascript
// practice-day8.js
const express = require('express');
const app = express();

app.use(express.json());

// Custom logging middleware
app.use((req, res, next) => {
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.url}`);
  next();
});

// Custom middleware that simulates an auth check via a simple header
function requireApiKey(req, res, next) {
  if (req.headers['x-api-key'] !== 'secret123') {
    return res.status(401).json({ error: 'Invalid or missing API key' });
  }
  next();
}

const users = [{ id: 1, name: 'Alice' }];

app.get('/users', (req, res) => {
  res.json(users);
});

// Protected route — only accessible with the correct header
app.post('/users', requireApiKey, (req, res) => {
  const newUser = { id: users.length + 1, ...req.body };
  users.push(newUser);
  res.status(201).json(newUser);
});

// Intentionally throwing an error to test error-handling middleware
app.get('/crash', (req, res) => {
  throw new Error('Intentional crash for testing');
});

// Error-handling middleware — MUST be last
app.use((err, req, res, next) => {
  console.error('Caught error:', err.message);
  res.status(500).json({ error: 'Internal Server Error' });
});

app.listen(3000, () => console.log('Running on http://localhost:3000'));
```

Test it:
```bash
curl http://localhost:3000/users
curl -X POST http://localhost:3000/users -H "Content-Type: application/json" -d '{"name":"Bob"}'
# ^ should fail with 401, no x-api-key header

curl -X POST http://localhost:3000/users -H "Content-Type: application/json" -H "x-api-key: secret123" -d '{"name":"Bob"}'
# ^ should succeed with 201

curl http://localhost:3000/crash
# ^ should be caught by error-handling middleware, returning 500 cleanly instead of crashing the process
```

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What is middleware in Express, precisely?**
   A: A function with signature `(req, res, next)` that sits between the incoming request and final route handler, able to inspect/modify req or res, end the cycle early, or call `next()` to pass control forward.

2. **Q: Why does middleware order matter?**
   A: Express processes middleware and routes in the exact order they're registered. A middleware registered after a route that already responded will never execute for that route — the request-response cycle already ended.

3. **Q: How does Express identify error-handling middleware?**
   A: By its function signature having 4 parameters `(err, req, res, next)` instead of the usual 3. It's only triggered by `next(err)` or a thrown synchronous error in a route handler, and must be registered after all other routes/middleware.

4. **Q: What's the difference between applying middleware with `app.use()` vs as a route argument like `app.get('/path', middleware, handler)`?**
   A: `app.use()` (with no path, or a path prefix) applies broadly to many/all routes. Passing middleware directly as an argument to a specific route applies it ONLY to that one route (or that one router if applied via `router.use()`).

5. **Q: What does `express.Router()` give you?**
   A: A way to group related routes into a separate, mountable mini-app — keeping route files organized by resource/domain, and letting you apply middleware to an entire group of routes at once via `app.use('/prefix', router)`.

6. **Q: What happens if you forget to call `next()` inside a middleware function?**
   A: The request hangs indefinitely — it never reaches subsequent middleware or the route handler, and the client eventually times out with no response.

---

## Tomorrow (Day 9) Preview
We cover **REST API design principles**: proper status code usage, resource naming conventions, API versioning strategies, pagination, and request validation with libraries like Zod/Joi — the difference between an API that "works" and one that a senior engineer would actually approve in a code review.
