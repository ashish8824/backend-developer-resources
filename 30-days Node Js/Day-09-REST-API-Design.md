# Day 9 — REST API Design: Status Codes, Naming, Versioning, Pagination, Validation

> **Why this day matters:** Anyone can make an Express route that "works." What separates a 3-year engineer from a junior is whether their API **follows conventions other engineers expect**, **fails predictably**, and **scales gracefully** as data grows. This is exactly what code reviews catch, and exactly what system-design-flavored interview questions probe. Today is less about new syntax and more about **judgment** — which matters a lot at the 3 YOE level, where interviewers expect opinions, not just facts.

---

## 1. Background: what does "REST" actually mean, and why do conventions matter?

**REST (Representational State Transfer)** is an architectural style, not a strict protocol or library — there's no `require('rest')`. It's a set of conventions for designing HTTP APIs around **resources** (nouns: users, orders, products) manipulated through standard HTTP methods (verbs: GET, POST, PUT, DELETE), using **status codes** to communicate outcomes, and being **stateless** (each request contains everything needed to process it — the server doesn't rely on memory of previous requests, which is what makes APIs horizontally scalable across multiple server instances, a concept you'll need again on Day 23 with load balancing).

**Why conventions matter practically, not just academically:** if every engineer on your team invents their own status codes and URL patterns, every new endpoint requires reading documentation from scratch, every frontend developer has to ask "wait, does this return 200 or 201 on success?", and every API consumer (including future you) wastes time guessing instead of relying on shared conventions. Following REST conventions is a way of **reducing cognitive load across an entire team and its API consumers** — that's the real "why."

---

## 2. HTTP Status Codes — know these cold, they get asked directly and often

### The Five Categories

| Range | Category | Meaning |
|---|---|---|
| 1xx | Informational | Rarely used directly in app code |
| 2xx | Success | The request was handled successfully |
| 3xx | Redirection | The client should look elsewhere / cache is still valid |
| 4xx | Client Error | The CLIENT did something wrong (bad input, no auth, etc.) |
| 5xx | Server Error | The SERVER failed to handle a valid request |

### The Specific Codes You'll Use Constantly

```javascript
// 200 OK — generic success, most commonly used for GET requests
res.status(200).json(users);

// 201 Created — successfully created a NEW resource (typically after POST)
// Convention: also include a 'Location' header pointing to the new resource
res.status(201)
  .location(`/users/${newUser.id}`)
  .json(newUser);

// 204 No Content — success, but there's nothing to return in the body
// (very common for DELETE operations)
res.status(204).end(); // NOTE: do NOT call .json() here — 204 means literally no body

// 400 Bad Request — the client sent malformed/invalid data
// (e.g., failed validation, missing required fields)
res.status(400).json({ error: 'Email is required' });

// 401 Unauthorized — the client is NOT authenticated at all
// (no token, expired token, invalid credentials)
res.status(401).json({ error: 'Authentication required' });

// 403 Forbidden — the client IS authenticated, but doesn't have
// permission to do this specific action (a very commonly confused
// pair with 401 — see the explicit comparison below)
res.status(403).json({ error: 'You do not have permission to do this' });

// 404 Not Found — the requested resource doesn't exist
res.status(404).json({ error: 'User not found' });

// 409 Conflict — the request conflicts with the current state of the
// resource (e.g., trying to register an email that already exists)
res.status(409).json({ error: 'Email already registered' });

// 422 Unprocessable Entity — syntactically valid request, but semantically
// invalid (e.g., valid JSON, but a date field is in the past when it
// must be in the future) — some teams use 400 for this instead; know
// both exist and that conventions vary by team
res.status(422).json({ error: 'Start date must be in the future' });

// 429 Too Many Requests — rate limit exceeded (directly connects to
// Day 22's rate limiting deep dive)
res.status(429).json({ error: 'Too many requests, please try again later' });

// 500 Internal Server Error — something broke on YOUR side, unexpectedly
res.status(500).json({ error: 'Something went wrong' });

// 503 Service Unavailable — the server is temporarily unable to handle
// requests (e.g., during a deploy, or if a critical dependency like the
// database is down) — often paired with a 'Retry-After' header
res.status(503).json({ error: 'Service temporarily unavailable' });
```

### 401 vs 403 — THE pair that gets mixed up constantly, even by working engineers

**The precise distinction (memorize this exact framing):**
> **401 Unauthorized** means "I don't know who you are" (authentication failure — missing or invalid credentials).
> **403 Forbidden** means "I know who you are, but you're not allowed to do this" (authorization failure — valid credentials, insufficient permissions).

```javascript
function requireAuth(req, res, next) {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ error: 'No token provided' }); // we don't know who you are
  }
  req.user = verifyToken(token);
  next();
}

function requireAdmin(req, res, next) {
  if (req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Admins only' }); // we know who you are, but you can't do this
  }
  next();
}
```

---

## 3. Resource Naming Conventions

```javascript
// GOOD — nouns, plural, lowercase, hierarchical relationships clear
GET    /users                  // list all users
GET    /users/123               // get a specific user
POST   /users                  // create a new user
PUT    /users/123               // replace user 123 entirely
PATCH  /users/123               // partially update user 123
DELETE /users/123               // delete user 123
GET    /users/123/orders        // list orders belonging to user 123
GET    /users/123/orders/456    // get a specific order belonging to user 123

// BAD — verbs in the URL (the HTTP METHOD is already the verb!), 
// inconsistent casing, singular/plural mixed
GET    /getAllUsers
POST   /createNewUser
GET    /User/123
POST   /users/123/deleteOrder
```

**Why verbs in URLs are wrong (the "why," not just "don't do it"):** the HTTP method ALREADY communicates the action (GET = read, POST = create, DELETE = remove). Adding a verb into the URL too (`/createNewUser`) is redundant and breaks the convention that URLs represent **resources** (nouns), while methods represent **actions** (verbs) on those resources. It also makes routes harder to predict — if every action gets its own custom URL pattern, there's no consistent rule for guessing the URL of a new endpoint.

### PUT vs PATCH — another frequently confused pair

```javascript
// PUT — replaces the ENTIRE resource. The client should send the
// COMPLETE representation; any fields not included are typically
// expected to be reset/removed (this is the strict REST definition,
// though some real-world APIs are lenient about it)
app.put('/users/:id', (req, res) => {
  // req.body should contain ALL fields: { name, email, age, ... }
});

// PATCH — partially updates the resource. The client sends ONLY the
// fields that should change; everything else stays as-is.
app.patch('/users/:id', (req, res) => {
  // req.body might just be: { email: 'new@email.com' }
});
```

**Interview tip:** if asked "when would you use PUT vs PATCH," the precise answer is about **completeness of the payload** — PUT implies replacing the whole resource, PATCH implies a partial update. In practice, many real-world APIs use PATCH almost everywhere for updates since most update operations are partial, but knowing the distinction (and that PUT is technically supposed to be idempotent and complete) shows you understand the spec, not just common usage.

---

## 4. API Versioning — why and how

**Background: why would an API ever need versions?** Once external clients (mobile apps, third-party integrations, your own frontend deployed separately) depend on your API's response shape, you **cannot freely change that shape** without breaking them. If you need to make a breaking change (renaming a field, changing a response structure, removing a field), you need a way to keep the OLD behavior available for existing clients while introducing the NEW behavior for clients ready to use it. Versioning is how you do that.

```javascript
// STRATEGY 1: URL path versioning (the MOST common in real-world APIs —
// easy to see at a glance, easy to route, cache-friendly)
app.use('/api/v1/users', usersV1Router);
app.use('/api/v2/users', usersV2Router);

// STRATEGY 2: Header-based versioning (cleaner URLs, but less discoverable —
// you can't just look at a URL to know the version; requires reading docs)
app.get('/api/users', (req, res) => {
  const version = req.headers['api-version'] || 'v1';
  if (version === 'v2') {
    // ... v2 logic
  } else {
    // ... v1 logic
  }
});

// STRATEGY 3: Query parameter versioning (less common, considered less
// clean since query params are usually meant for filtering/searching,
// not core routing decisions)
// GET /api/users?version=2
```

**What most real companies actually do:** URL path versioning (`/api/v1/...`) is overwhelmingly the most common in production because it's simple, explicit, and works well with API gateways, caching layers, and load balancers that route based on URL path (a concept that'll connect to Day 23/25).

---

## 5. Pagination — a topic that ALWAYS comes up once data grows

**Background: why can't you just return `GET /users` with everything?** If a table has 2 million rows, returning all of them in one response would be slow, use enormous memory and bandwidth, and likely time out. Pagination breaks large result sets into manageable pages.

### Offset-based pagination (the simpler, more common approach)

```javascript
app.get('/users', async (req, res) => {
  // Convert query string values (always strings) into numbers, with
  // sensible defaults if the client doesn't specify them
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const offset = (page - 1) * limit; // how many records to SKIP

  // (Pseudo-DB-query — real implementation depends on your DB layer,
  // covered properly on Day 10/11)
  const users = await db.query(
    'SELECT * FROM users ORDER BY id LIMIT ? OFFSET ?',
    [limit, offset]
  );
  const totalCount = await db.query('SELECT COUNT(*) FROM users');

  res.json({
    data: users,
    pagination: {
      page,
      limit,
      totalCount,
      totalPages: Math.ceil(totalCount / limit),
    },
  });
});

// GET /users?page=2&limit=20
```

**The weakness of offset pagination (an interview-worthy nuance):** for very large tables, `OFFSET 1000000` still requires the database to scan through and discard a million rows before returning results — this gets SLOWER as the offset grows, even though you're only asking for 20 rows. This is a real, measurable performance problem at scale.

### Cursor-based pagination (the scalable alternative — used by Twitter/X, Stripe, GitHub APIs)

```javascript
// Instead of "skip N rows," we remember WHERE we left off using a unique,
// sortable value (commonly the ID, or a timestamp) from the last item
// of the previous page — and ask the DB for "everything after that point."
app.get('/users', async (req, res) => {
  const limit = parseInt(req.query.limit) || 20;
  const cursor = req.query.cursor; // the ID of the last item from the previous page

  const query = cursor
    ? 'SELECT * FROM users WHERE id > ? ORDER BY id LIMIT ?'
    : 'SELECT * FROM users ORDER BY id LIMIT ?';
  const params = cursor ? [cursor, limit] : [limit];

  const users = await db.query(query, params);
  const nextCursor = users.length > 0 ? users[users.length - 1].id : null;

  res.json({
    data: users,
    nextCursor, // client sends this back as ?cursor=<value> for the next page
  });
});

// GET /users?limit=20             (first page)
// GET /users?limit=20&cursor=120  (next page, starting after id 120)
```

**Why cursor-based scales better:** `WHERE id > 120` can use an index efficiently regardless of how deep into the dataset you are — it doesn't need to scan and discard preceding rows the way `OFFSET` does. The tradeoff: you lose the ability to jump directly to "page 47" — cursor pagination only supports moving forward/backward sequentially, not random access to an arbitrary page number.

---

## 6. Request Validation — why "just check if it exists" isn't enough

**Background: why validate at all if you'll get a DB error anyway?** Two reasons: (1) you want to reject bad input **fast, with a clear, specific error message**, before it ever touches your database or business logic — DB errors are often cryptic and expose internal details you don't want clients to see; (2) some bad input (e.g., a negative "age," a malformed email NOT caught by a DB column type) would otherwise pass through silently and corrupt your data, since the database often doesn't enforce the SAME rules your business logic needs.

```javascript
// Manual validation (what you'd write WITHOUT a library — educational,
// but error-prone and repetitive at scale)
app.post('/users', (req, res) => {
  const { name, email, age } = req.body;

  if (!name || typeof name !== 'string') {
    return res.status(400).json({ error: 'Name is required and must be a string' });
  }
  if (!email || !/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)) {
    return res.status(400).json({ error: 'A valid email is required' });
  }
  if (age !== undefined && (typeof age !== 'number' || age < 0)) {
    return res.status(400).json({ error: 'Age must be a non-negative number' });
  }

  // ... proceed with creating the user
});
```

### Using Zod (a modern, widely-adopted validation library) — what production code actually looks like

```javascript
const { z } = require('zod');

// Define a SCHEMA describing exactly what valid input looks like.
// This is declarative — you describe the SHAPE of valid data once,
// instead of writing imperative if-checks for every field.
const createUserSchema = z.object({
  name: z.string().min(1, 'Name is required'),
  email: z.string().email('Must be a valid email address'),
  age: z.number().int().nonnegative().optional(),
});

// A reusable validation middleware — this pattern lets you apply
// schema validation to ANY route with one line, instead of repeating
// manual checks everywhere.
function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      // .flatten() turns Zod's error details into a clean, readable object
      return res.status(400).json({ error: result.error.flatten() });
    }
    req.body = result.data; // use the PARSED (and type-coerced) data going forward
    next();
  };
}

app.post('/users', validate(createUserSchema), (req, res) => {
  // by the time we're here, req.body is GUARANTEED to match the schema
  res.status(201).json(req.body);
});
```

**Interview tip:** mentioning Zod or Joi by name, and explaining the "validate at the edge before business logic" principle, signals real production experience — this is a very commonly asked practical question: *"How do you validate incoming requests in your APIs?"*

---

## 7. Consistent Response Envelope — a smaller but real convention

Many production APIs wrap responses in a consistent shape, rather than returning raw arrays/objects inconsistently across endpoints:

```javascript
// A consistent success shape
res.status(200).json({
  success: true,
  data: users,
  meta: { page: 1, totalCount: 50 },
});

// A consistent error shape
res.status(400).json({
  success: false,
  error: {
    code: 'VALIDATION_ERROR',
    message: 'Email is required',
  },
});
```

This isn't a strict REST rule, but it's a common, defensible convention worth knowing and being able to discuss tradeoffs on (consistency and easier frontend handling vs. slightly more verbose payloads).

---

## 8. How this connects to real backend work (3 YOE framing)

- **"A frontend dev says your DELETE endpoint returns weird data they don't need" → what status code/response should DELETE actually return?"** → 204 No Content, with no body — there's nothing meaningful to return after a successful deletion.
- **"Your `/users` endpoint started timing out as the users table grew to millions of rows — what's happening, and how would you fix it?"** → Likely offset-based pagination with a growing OFFSET causing the DB to scan and discard more rows over time; switch to cursor-based pagination using an indexed column.
- **"How would you introduce a breaking change to an API that mobile apps already depend on?"** → Introduce a new version (`/api/v2/...`), keep `/api/v1/...` running unchanged for existing clients, and deprecate v1 on a timeline communicated to API consumers.
- **"Why validate input on the server if the frontend already validates it?"** → Frontend validation is for UX only — it can always be bypassed (direct API calls via curl/Postman, malicious clients, bugs in frontend code). Server-side validation is the actual security/data-integrity boundary.

---

## 9. Hands-on practice for today

```javascript
// practice-day9.js
const express = require('express');
const { z } = require('zod');
const app = express();
app.use(express.json());

let users = [
  { id: 1, name: 'Alice', email: 'alice@test.com' },
  { id: 2, name: 'Bob', email: 'bob@test.com' },
];

const createUserSchema = z.object({
  name: z.string().min(1),
  email: z.string().email(),
});

function validate(schema) {
  return (req, res, next) => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      return res.status(400).json({ success: false, error: result.error.flatten() });
    }
    req.body = result.data;
    next();
  };
}

// Paginated list endpoint
app.get('/users', (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 10;
  const start = (page - 1) * limit;
  const paginated = users.slice(start, start + limit);

  res.status(200).json({
    success: true,
    data: paginated,
    meta: { page, limit, totalCount: users.length },
  });
});

app.post('/users', validate(createUserSchema), (req, res) => {
  const newUser = { id: users.length + 1, ...req.body };
  users.push(newUser);
  res.status(201).location(`/users/${newUser.id}`).json({ success: true, data: newUser });
});

app.delete('/users/:id', (req, res) => {
  const exists = users.some((u) => u.id === parseInt(req.params.id));
  if (!exists) {
    return res.status(404).json({ success: false, error: { message: 'User not found' } });
  }
  users = users.filter((u) => u.id !== parseInt(req.params.id));
  res.status(204).end();
});

app.listen(3000, () => console.log('Running on http://localhost:3000'));
```

Test the validation failure path and confirm you get a clean 400 with field-level error details — that's the behavior a senior engineer would expect from any production-grade endpoint.

---

## 10. Interview Q&A Recap (say these out loud)

1. **Q: Explain the difference between 401 and 403.**
   A: 401 means the server doesn't know who's making the request (missing/invalid authentication). 403 means the server knows who's making the request, but that identity doesn't have permission for this specific action.

2. **Q: Why shouldn't URLs contain verbs like `/getUsers` or `/createUser`?**
   A: The HTTP method already conveys the action (GET, POST, etc.) — URLs should represent resources (nouns). Mixing verbs into URLs is redundant and breaks predictable, consistent API design.

3. **Q: What's the difference between PUT and PATCH?**
   A: PUT is meant to replace the entire resource with the provided payload. PATCH applies a partial update, changing only the fields included in the request.

4. **Q: Why does offset-based pagination get slower as you go deeper into a dataset, and what's the alternative?**
   A: `OFFSET` requires the database to scan and discard all preceding rows before returning the requested page, which gets more expensive as the offset grows. Cursor-based pagination uses an indexed column (like ID or timestamp) to jump directly to "everything after this point," which stays fast regardless of depth, at the cost of losing random page-number access.

5. **Q: Why validate requests on the server if the frontend already does it?**
   A: Frontend validation can always be bypassed by direct API calls, so it's a UX nicety, not a security boundary. Server-side validation is the actual enforcement point protecting data integrity and business logic.

6. **Q: What status code should a successful DELETE return, and why?**
   A: 204 No Content — the deletion succeeded and there's no meaningful body to return, so the response should have an empty body with that status code.

---

## Tomorrow (Day 10) Preview
We move into **MongoDB + Mongoose**: schema design, CRUD operations, indexes (and why they matter for query performance), and the aggregation pipeline — your first real persistence layer in this course, building directly on today's API design principles (you'll be returning real paginated, validated data from an actual database).
