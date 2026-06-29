# Day 13 — Error Handling, Logging, Config Management, and Security Basics

> **Why this day matters:** These topics rarely show up as a single dedicated interview question, but they show up constantly INSIDE other questions — "how would you debug a production issue," "how do you structure a Node app," "what security concerns do you watch for." This is the "operational maturity" day. Engineers with real production experience handle these instinctively; engineers who've only built tutorials usually haven't thought about them at all. This is one of the easiest days to differentiate yourself in an interview.

---

## 1. Centralized Error Handling — building on Day 8's error middleware

### Background: why is "just try/catch everywhere" not good enough at scale?

Scattering `try/catch` blocks with ad-hoc `res.status(500).json({error: 'something broke'})` everywhere leads to **inconsistent error responses** across your API (different shapes, different status codes for similar problems), **duplicated logic**, and **no single place** to add cross-cutting behavior like logging every error or hiding sensitive details in production. The fix is a combination of **custom error classes** + Day 8's **centralized error-handling middleware**.

### Custom Error Classes — structuring errors as data, not just strings

```javascript
// errors/AppError.js
// A base class for ALL of our application's intentional, expected errors —
// as opposed to truly unexpected bugs (a null reference, a typo, etc.).
// This distinction matters: expected errors (user not found, invalid
// input) should produce clean, predictable client responses. Unexpected
// bugs should be logged loudly and treated as something to FIX, not
// something to gracefully message to the user about specifics.
class AppError extends Error {
  constructor(message, statusCode) {
    super(message); // sets this.message
    this.statusCode = statusCode;
    this.isOperational = true; // flags this as an EXPECTED, handled error type

    // Captures a clean stack trace excluding the constructor call itself —
    // makes debugging easier by not cluttering the trace with this class's
    // own internals.
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends AppError {
  constructor(message = 'Resource not found') {
    super(message, 404);
  }
}

class ValidationError extends AppError {
  constructor(message = 'Invalid input') {
    super(message, 400);
  }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') {
    super(message, 401);
  }
}

class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') {
    super(message, 403);
  }
}

module.exports = { AppError, NotFoundError, ValidationError, UnauthorizedError, ForbiddenError };
```

```javascript
// Using custom errors in route logic — notice how much cleaner and more
// SEMANTIC this reads compared to scattered res.status(xxx).json() calls
const { NotFoundError } = require('./errors/AppError');

app.get('/users/:id', async (req, res, next) => {
  try {
    const user = await User.findByPk(req.params.id);
    if (!user) {
      throw new NotFoundError(`User ${req.params.id} not found`);
    }
    res.json(user);
  } catch (err) {
    next(err); // forward to centralized error-handling middleware (Day 8)
  }
});
```

### The Centralized Error Handler — the one place all errors converge

```javascript
// middleware/errorHandler.js
function errorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;
  const isOperational = err.isOperational || false;

  // ALWAYS log the error server-side, regardless of what we send to the client.
  // In production, this would go to a proper logging system (section 2),
  // not just console.error.
  console.error({
    message: err.message,
    statusCode,
    stack: err.stack,
    path: req.path,
    method: req.method,
  });

  // CRITICAL SECURITY PRINCIPLE: never leak internal error details
  // (stack traces, database error messages, file paths) to the CLIENT
  // in production. An attacker probing your API can learn a lot from
  // verbose error messages (database structure, library versions, etc.)
  const responseMessage = isOperational
    ? err.message                          // safe to show — WE wrote this message intentionally
    : 'An unexpected error occurred';      // generic — hides internals of a real bug

  res.status(statusCode).json({
    success: false,
    error: { message: responseMessage },
  });
}

module.exports = errorHandler;
```

```javascript
// app.js
const errorHandler = require('./middleware/errorHandler');
// ... all your routes ...
app.use(errorHandler); // MUST be registered LAST (Day 8 rule)
```

**Why the `isOperational` flag matters — a real production distinction:** an `AppError` (like `NotFoundError`) is something YOU anticipated and wrote deliberate handling for — it's safe to show its message to the client. A raw, unexpected `TypeError` from a bug is NOT something you want to expose details about — showing `"Cannot read property 'id' of undefined"` to a client both looks unprofessional and can leak information about your code's internals. This distinction — operational/expected errors vs. programmer bugs — is a genuinely important production engineering concept (popularized heavily by Node.js error-handling best practices writing).

---

## 2. Logging — why `console.log` isn't enough in production

### Background: what's actually wrong with console.log at scale?

`console.log` has no concept of **log levels** (you can't easily say "only show me errors and warnings in production, but everything in development"), no **structured format** (hard to search/filter/aggregate millions of log lines if they're just free-form text), and no built-in way to **route logs** to different destinations (a file, a log aggregation service like Datadog/ELK, etc.). Production apps use dedicated logging libraries — `winston` and `pino` are the two most common in Node.js.

```javascript
const winston = require('winston');

// A logger configured with multiple "transports" (destinations) and a
// structured JSON format — this is what makes logs searchable/filterable
// at scale, versus unstructured plain text.
const logger = winston.createLogger({
  level: process.env.NODE_ENV === 'production' ? 'info' : 'debug',
  // ^ in production, skip noisy 'debug' logs; show them locally during dev
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json() // structured, machine-parseable output
  ),
  transports: [
    new winston.transports.Console(),                       // always log to console
    new winston.transports.File({ filename: 'error.log', level: 'error' }), // errors ALSO go to a dedicated file
  ],
});

// Standard log levels, from most to least severe (know this order):
logger.error('Database connection failed', { errorCode: 'DB_CONN_001' });
logger.warn('API rate limit approaching threshold');
logger.info('User 42 logged in successfully');
logger.debug('Cache hit for key: user:42'); // only shows in dev, per the level config above
```

**Why structured logging (passing an object, not just concatenated strings) matters in practice:** with logs like `{ message: 'User logged in', userId: 42, ip: '1.2.3.4' }`, a log aggregation tool can let you query "show me every log entry where `userId: 42`" instantly. With unstructured text like `"User 42 logged in from 1.2.3.4"`, you'd need fragile text-matching/regex to extract the same information at scale.

**Interview tip:** if asked "how do you handle logging in production," mention: log levels (filtering noise in production vs. development), structured/JSON format (searchability), and routing critical logs (errors) to a dedicated, monitored destination — and ideally, integration with an external monitoring/alerting tool (Datadog, Sentry, CloudWatch) so the team gets notified of issues rather than relying on someone manually reading log files.

---

## 3. Environment Configuration — `dotenv` and config management

### Background: why not just hardcode values like database URLs or API keys?

Hardcoding config means: you can't have different settings for development/staging/production without editing code, you risk **committing secrets to version control** (a serious, common real-world security incident — exposed API keys in public GitHub repos are a constant, real problem), and onboarding a new developer requires them to dig through code to find what needs configuring.

```javascript
// .env (NEVER commit this file — add it to .gitignore immediately)
DATABASE_URL=postgres://user:pass@localhost:5432/mydb
JWT_SECRET=a-long-random-string-only-the-server-knows
PORT=3000
NODE_ENV=development
REDIS_URL=redis://localhost:6379
```

```javascript
// app.js
require('dotenv').config(); // loads .env into process.env at startup

const PORT = process.env.PORT || 3000;
const dbUrl = process.env.DATABASE_URL;

// A REAL production safeguard: fail FAST and LOUD at startup if a
// required config value is missing, rather than discovering it later
// through a confusing runtime error deep in some unrelated code path.
const requiredEnvVars = ['DATABASE_URL', 'JWT_SECRET'];
for (const key of requiredEnvVars) {
  if (!process.env[key]) {
    console.error(`Missing required environment variable: ${key}`);
    process.exit(1); // refuse to start the app in a broken config state
  }
}
```

```
# .gitignore
.env
node_modules/
```

**Interview tip:** if asked "how do you manage secrets across different environments (dev/staging/prod)?" — mention: `.env` files for local development (never committed), and in real production deployments, secrets are typically injected via the hosting platform's secret management (AWS Secrets Manager, environment variables set directly in your deployment platform/CI pipeline, Kubernetes Secrets, etc.) rather than ever existing as a file at all in production.

---

## 4. Security Basics: helmet, CORS, and Input Sanitization

### `helmet` — setting security-related HTTP headers automatically

**Background:** browsers respect certain HTTP response headers that instruct them to behave more securely (e.g., preventing your site from being embedded in a malicious iframe, preventing browsers from guessing/sniffing content types in unsafe ways). Manually setting all of these correctly is easy to get wrong or forget; `helmet` sets sensible secure defaults for you in one line.

```javascript
const helmet = require('helmet');
app.use(helmet());
// Sets headers like:
// X-Content-Type-Options: nosniff  -- prevents MIME-type sniffing attacks
// X-Frame-Options: DENY            -- prevents clickjacking via iframes
// Strict-Transport-Security        -- forces HTTPS on supporting browsers
// ...and several others, all with secure, sensible defaults
```

### CORS (Cross-Origin Resource Sharing)

**Background: why does this even need to be configured?** Browsers enforce the **Same-Origin Policy** by default — JavaScript running on `frontend.com` is normally BLOCKED from making requests to `api.backend.com` (a different origin), as a security measure preventing malicious sites from silently making requests to other sites using a victim's logged-in session. CORS is the mechanism by which YOUR server explicitly tells browsers "it's okay, requests from this specific origin are allowed."

```javascript
const cors = require('cors');

// WRONG for production (though common in quick demos): allows literally
// ANY origin to call your API — defeats the purpose of the protection,
// appropriate only for fully public, non-sensitive APIs.
app.use(cors());

// RIGHT for production: explicitly allowlist only the origins that
// SHOULD be allowed to call your API.
app.use(cors({
  origin: ['https://myapp.com', 'https://staging.myapp.com'],
  credentials: true, // allows cookies to be sent cross-origin, if using session-based auth
}));
```

**Interview tip:** a commonly confused point — CORS is a **browser-enforced** security mechanism, not a server-side security mechanism. A tool like `curl` or Postman completely ignores CORS rules (they only matter to browser-based JavaScript). This means CORS protects against malicious *browser-based* cross-origin requests; it does **nothing** to stop a direct, server-to-server, or script-based attack — that's what authentication/authorization (Day 12) is actually for. Conflating CORS with "API security" broadly is a common but incorrect assumption — being able to correct this distinction shows real depth.

### Input Sanitization — protecting against injection-style attacks beyond SQL

```javascript
// NoSQL injection example — a real, MongoDB-specific attack vector that
// surprises people coming from a SQL-injection-only mental model.
// If a client sends: { "email": { "$gt": "" }, "password": { "$gt": "" } }
// as the BODY of a login request, and your code does this naively:
const user = await User.findOne({ email: req.body.email, password: req.body.password });
// MongoDB would interpret { "$gt": "" } as a QUERY OPERATOR (matching
// ANY value greater than an empty string — i.e., matching almost
// anything), potentially letting an attacker log in as the FIRST user
// in the collection without knowing any real credentials!

// THE FIX: explicitly validate/sanitize input types BEFORE using them in
// a query — this is exactly why Day 9's validation-at-the-edge principle
// (e.g., using Zod to enforce email/password must be STRINGS) matters
// for security, not just for clean error messages.
const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(1),
});
// Once validated against this schema, req.body.email is GUARANTEED to be
// a string, not an object containing a MongoDB operator — the injection
// vector is closed simply by enforcing types strictly at the boundary.
```

```javascript
// XSS (Cross-Site Scripting) consideration: if your API ever returns
// user-submitted content that a FRONTEND renders directly as HTML
// (e.g., a comment, a bio field), unescaped content containing
// "<script>...</script>" could execute in other users' browsers.
// As a backend engineer, you're not solely responsible for this (the
// frontend should escape output), but sanitizing/escaping known-risky
// fields server-side, or using a library like 'sanitize-html' for any
// field that might later be rendered as HTML, is a reasonable extra
// layer of defense — defense in depth, not relying on just one layer.
```

---

## 5. How this connects to real backend work (3 YOE framing)

- **"How would you debug an intermittent 500 error in production with no clear repro steps?"** → Structured logging (correlating logs by request ID, checking error logs filtered by severity) and distinguishing operational errors (logged with context, handled gracefully) from unexpected bugs (logged with full stack traces, alerting the team).
- **"A teammate accidentally committed an API key to GitHub — what's the immediate and long-term fix?"** → Immediately rotate/revoke the exposed key (assume it's compromised the moment it's pushed, even if deleted in a later commit — git history retains it). Long-term: `.env` + `.gitignore` discipline, and consider tools like `git-secrets` or pre-commit hooks that scan for accidental secret commits.
- **"Why might your API work fine in Postman but fail when called from your frontend's JavaScript?"** → Classic CORS issue — the browser is blocking the cross-origin request because the server hasn't explicitly allowed that frontend's origin.
- **"How do you decide what level of detail to include in an error response to the client?"** → The operational vs. non-operational error distinction — safe, intentional messages can be shown; raw bugs should return a generic message while being logged with full detail server-side for the engineering team.

---

## 6. Hands-on practice for today

```javascript
// practice-day13.js
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
require('dotenv').config();

class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

const app = express();
app.use(helmet());
app.use(cors({ origin: 'http://localhost:5173' })); // example: a Vite frontend dev server
app.use(express.json());

app.get('/users/:id', (req, res, next) => {
  try {
    const id = parseInt(req.params.id);
    if (id !== 1) {
      throw new AppError(`User ${id} not found`, 404); // an EXPECTED, operational error
    }
    res.json({ id: 1, name: 'Alice' });
  } catch (err) {
    next(err);
  }
});

app.get('/crash-test', (req, res, next) => {
  try {
    const obj = undefined;
    console.log(obj.property); // an UNEXPECTED bug — TypeError
  } catch (err) {
    next(err); // err.isOperational is undefined here, NOT true
  }
});

app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  console.error({ message: err.message, statusCode, stack: err.stack });

  const responseMessage = err.isOperational ? err.message : 'An unexpected error occurred';
  res.status(statusCode).json({ success: false, error: { message: responseMessage } });
});

app.listen(3000, () => console.log('Running on http://localhost:3000'));
```

Test both `/users/999` (clean 404 with a real message) and `/crash-test` (generic 500 message to the client, but a full stack trace logged server-side) — and notice how the client-facing message differs based on whether the error was "expected" or not.

---

## 7. Interview Q&A Recap (say these out loud)

1. **Q: What's the difference between an "operational" error and a programmer bug, and why does the distinction matter?**
   A: Operational errors are expected, anticipated failure cases (resource not found, invalid input) that are safe to communicate clearly to the client. Programmer bugs (unexpected exceptions) shouldn't leak internal details to clients — they should be logged in full detail server-side while the client receives a generic message.

2. **Q: Why use a logging library like winston/pino instead of console.log in production?**
   A: Log levels (filtering noise by environment), structured/JSON output (searchable and filterable at scale by log aggregation tools), and the ability to route different severities to different destinations (e.g., errors to a dedicated file or alerting system).

3. **Q: Why shouldn't you commit a `.env` file to version control?**
   A: It typically contains secrets (database credentials, API keys, JWT signing secrets) — committing it exposes them in git history, which can be found even if later deleted, and is a common real-world cause of security breaches.

4. **Q: Is CORS a server-side security feature?**
   A: It's a browser-ENFORCED mechanism — it only restricts cross-origin requests made by JavaScript running in a browser. Tools like curl or direct server-to-server calls ignore CORS entirely, so it doesn't protect against non-browser-based attacks; actual security still depends on proper authentication/authorization.

5. **Q: How can a NoSQL injection attack happen in a MongoDB-backed login endpoint, and how do you prevent it?**
   A: If query operators like `$gt` are passed as object values in fields expected to be plain strings (e.g., email/password), MongoDB may interpret them as actual query logic rather than literal values, potentially bypassing authentication. Prevention: strictly validate that such fields are actual strings before using them in a query (e.g., with Zod/Joi schema validation).

6. **Q: What does `helmet` actually do?**
   A: Sets a collection of security-related HTTP response headers with sensible defaults (preventing clickjacking, MIME-sniffing, enforcing HTTPS where supported, etc.) that would otherwise need to be configured manually and are easy to forget.

---

## Day 14 Preview — Week 2 Revision Day
We consolidate Days 8-13 (Express, REST design, MongoDB, SQL, authentication, error handling/security) into one combined project: a full CRUD API with authentication, validation, proper error handling, and basic security hardening — a complete, interview-ready mini backend, mirroring what Day 7 did for Week 1.
