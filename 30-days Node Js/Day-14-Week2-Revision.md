# Day 14 — Week 2 Revision: Build a Full CRUD API with Auth

> **Why this day matters:** Same philosophy as Day 7 — concepts fade unless you USE them together. Today you build ONE cohesive project combining Days 8-13: Express routing/middleware, REST conventions, database persistence, JWT authentication with RBAC, centralized error handling, and security basics. This is genuinely close to what a take-home interview assignment or a "build a small feature" live coding round looks like at the 3 YOE level.

---

## 1. The Mini Project: A "Task Management" API

**The goal:** a small but complete backend with users, authentication, and tasks owned by users — using SQLite (via Sequelize) so it runs with zero external database setup, but every pattern shown maps directly onto PostgreSQL/MySQL in a real job.

### Project structure (this organization itself is interview-relevant — "how do you structure a Node project?" is a real question)

```
project/
├── .env
├── app.js
├── config/
│   └── database.js
├── models/
│   ├── User.js
│   └── Task.js
├── middleware/
│   ├── auth.js
│   └── errorHandler.js
├── errors/
│   └── AppError.js
├── routes/
│   ├── authRoutes.js
│   └── taskRoutes.js
└── controllers/
    ├── authController.js
    └── taskController.js
```

**Why this structure (be ready to explain the reasoning, not just the layout):** separating routes (URL → handler mapping) from controllers (actual business logic) from models (data layer) follows a loose **MVC-inspired separation of concerns** — each piece has one clear responsibility, making the codebase easier to navigate and test as it grows. This is a common real-world structure, though exact conventions vary by team.

---

## 2. Building It Piece by Piece

### `errors/AppError.js` (Day 13)

```javascript
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
    this.isOperational = true;
  }
}

class NotFoundError extends AppError {
  constructor(message = 'Resource not found') { super(message, 404); }
}

class ValidationError extends AppError {
  constructor(message = 'Invalid input') { super(message, 400); }
}

class UnauthorizedError extends AppError {
  constructor(message = 'Authentication required') { super(message, 401); }
}

class ForbiddenError extends AppError {
  constructor(message = 'Insufficient permissions') { super(message, 403); }
}

module.exports = { AppError, NotFoundError, ValidationError, UnauthorizedError, ForbiddenError };
```

### `config/database.js` (Day 11)

```javascript
const { Sequelize } = require('sequelize');

// SQLite for easy local running -- same Sequelize API works identically
// against PostgreSQL/MySQL by just changing the connection config
const sequelize = new Sequelize('sqlite:./database.sqlite', { logging: false });

module.exports = sequelize;
```

### `models/User.js` (Day 11 + Day 12)

```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const User = sequelize.define('User', {
  email: { type: DataTypes.STRING, allowNull: false, unique: true, validate: { isEmail: true } },
  password: { type: DataTypes.STRING, allowNull: false }, // stores the HASHED password, never plain text
  role: { type: DataTypes.STRING, defaultValue: 'user' },
});

module.exports = User;
```

### `models/Task.js` (Day 11 relations)

```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');
const User = require('./User');

const Task = sequelize.define('Task', {
  title: { type: DataTypes.STRING, allowNull: false },
  completed: { type: DataTypes.BOOLEAN, defaultValue: false },
});

// One-to-Many: a User has many Tasks. This adds a UserId foreign key
// column to the Task table automatically.
User.hasMany(Task);
Task.belongsTo(User);

module.exports = Task;
```

### `middleware/auth.js` (Day 12)

```javascript
const jwt = require('jsonwebtoken');
const { UnauthorizedError, ForbiddenError } = require('../errors/AppError');

function requireAuth(req, res, next) {
  try {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) throw new UnauthorizedError('No token provided');

    req.user = jwt.verify(token, process.env.JWT_SECRET);
    next();
  } catch (err) {
    next(err instanceof UnauthorizedError ? err : new UnauthorizedError('Invalid or expired token'));
  }
}

function requireRole(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return next(new ForbiddenError());
    }
    next();
  };
}

module.exports = { requireAuth, requireRole };
```

### `middleware/errorHandler.js` (Day 13)

```javascript
function errorHandler(err, req, res, next) {
  const statusCode = err.statusCode || 500;
  const isOperational = err.isOperational || false;

  console.error({ message: err.message, statusCode, stack: err.stack, path: req.path });

  res.status(statusCode).json({
    success: false,
    error: { message: isOperational ? err.message : 'An unexpected error occurred' },
  });
}

module.exports = errorHandler;
```

### `controllers/authController.js` (Day 12 + Day 9 validation)

```javascript
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const { z } = require('zod');
const User = require('../models/User');
const { ValidationError, UnauthorizedError } = require('../errors/AppError');

const credentialsSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});

async function register(req, res, next) {
  try {
    const result = credentialsSchema.safeParse(req.body);
    if (!result.success) throw new ValidationError(result.error.errors[0].message);

    const { email, password } = result.data;
    const existing = await User.findOne({ where: { email } });
    if (existing) throw new ValidationError('Email already registered');

    const hashedPassword = await bcrypt.hash(password, 10);
    const user = await User.create({ email, password: hashedPassword });

    res.status(201).json({ success: true, data: { id: user.id, email: user.email } });
  } catch (err) {
    next(err);
  }
}

async function login(req, res, next) {
  try {
    const result = credentialsSchema.safeParse(req.body);
    if (!result.success) throw new ValidationError(result.error.errors[0].message);

    const { email, password } = result.data;
    const user = await User.findOne({ where: { email } });
    if (!user) throw new UnauthorizedError('Invalid credentials');

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) throw new UnauthorizedError('Invalid credentials');

    const token = jwt.sign({ userId: user.id, role: user.role }, process.env.JWT_SECRET, { expiresIn: '1h' });
    res.json({ success: true, data: { token } });
  } catch (err) {
    next(err);
  }
}

module.exports = { register, login };
```

### `controllers/taskController.js` (Day 9 pagination/validation + Day 11 relations)

```javascript
const { z } = require('zod');
const Task = require('../models/Task');
const { NotFoundError, ValidationError, ForbiddenError } = require('../errors/AppError');

const createTaskSchema = z.object({ title: z.string().min(1) });

async function getTasks(req, res, next) {
  try {
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 10;

    // SECURITY-CRITICAL: only return tasks belonging to THE CURRENTLY
    // AUTHENTICATED user (req.user.userId, attached by requireAuth) —
    // never trust a client-supplied userId for "whose data to return."
    // This is exactly the AuthZ check Day 12 emphasized.
    const tasks = await Task.findAndCountAll({
      where: { UserId: req.user.userId },
      limit,
      offset: (page - 1) * limit,
      order: [['createdAt', 'DESC']],
    });

    res.json({
      success: true,
      data: tasks.rows,
      meta: { page, limit, totalCount: tasks.count },
    });
  } catch (err) {
    next(err);
  }
}

async function createTask(req, res, next) {
  try {
    const result = createTaskSchema.safeParse(req.body);
    if (!result.success) throw new ValidationError(result.error.errors[0].message);

    const task = await Task.create({ ...result.data, UserId: req.user.userId });
    res.status(201).json({ success: true, data: task });
  } catch (err) {
    next(err);
  }
}

async function updateTask(req, res, next) {
  try {
    const task = await Task.findByPk(req.params.id);
    if (!task) throw new NotFoundError('Task not found');

    // OWNERSHIP CHECK -- a task existing isn't enough; it must belong
    // to the requesting user. Without this check, ANY logged-in user
    // could update ANYONE's tasks just by guessing IDs -- a real,
    // common authorization bug (sometimes called "Broken Object Level
    // Authorization," a well-known API security category).
    if (task.UserId !== req.user.userId) throw new ForbiddenError();

    await task.update(req.body);
    res.json({ success: true, data: task });
  } catch (err) {
    next(err);
  }
}

async function deleteTask(req, res, next) {
  try {
    const task = await Task.findByPk(req.params.id);
    if (!task) throw new NotFoundError('Task not found');
    if (task.UserId !== req.user.userId) throw new ForbiddenError();

    await task.destroy();
    res.status(204).end();
  } catch (err) {
    next(err);
  }
}

module.exports = { getTasks, createTask, updateTask, deleteTask };
```

### `routes/authRoutes.js` and `routes/taskRoutes.js` (Day 8 Router pattern)

```javascript
// routes/authRoutes.js
const express = require('express');
const router = express.Router();
const { register, login } = require('../controllers/authController');

router.post('/register', register);
router.post('/login', login);

module.exports = router;
```

```javascript
// routes/taskRoutes.js
const express = require('express');
const router = express.Router();
const { requireAuth } = require('../middleware/auth');
const { getTasks, createTask, updateTask, deleteTask } = require('../controllers/taskController');

// router.use() applies requireAuth to EVERY route defined below it in
// this router -- every task route requires authentication, with one line.
router.use(requireAuth);

router.get('/', getTasks);
router.post('/', createTask);
router.put('/:id', updateTask);
router.delete('/:id', deleteTask);

module.exports = router;
```

### `app.js` (pulling it all together — Day 8 app structure + Day 13 security)

```javascript
require('dotenv').config();
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');

const sequelize = require('./config/database');
const authRoutes = require('./routes/authRoutes');
const taskRoutes = require('./routes/taskRoutes');
const errorHandler = require('./middleware/errorHandler');

const app = express();

app.use(helmet());
app.use(cors());
app.use(express.json());

app.use('/api/v1/auth', authRoutes);   // Day 9 versioning convention
app.use('/api/v1/tasks', taskRoutes);

// Catch-all for unmatched routes -- a clean 404 instead of Express's default HTML error page
app.use((req, res) => {
  res.status(404).json({ success: false, error: { message: 'Route not found' } });
});

app.use(errorHandler); // MUST be last

const PORT = process.env.PORT || 3000;

sequelize.sync().then(() => {
  app.listen(PORT, () => console.log(`Server running on http://localhost:${PORT}`));
});
```

```
# .env
JWT_SECRET=a-long-random-development-secret
PORT=3000
```

---

## 3. Testing the Full Flow

```bash
# Register
curl -X POST http://localhost:3000/api/v1/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@test.com","password":"password123"}'

# Login (copy the returned token)
curl -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@test.com","password":"password123"}'

# Create a task (replace TOKEN below)
curl -X POST http://localhost:3000/api/v1/tasks \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"title":"Finish Node.js roadmap"}'

# List tasks
curl http://localhost:3000/api/v1/tasks -H "Authorization: Bearer TOKEN"

# Try accessing without a token -- should get a clean 401
curl http://localhost:3000/api/v1/tasks
```

**Try the ownership-bug test deliberately:** register a SECOND user, log in as them, get THEIR token, then try to `PUT`/`DELETE` the first user's task ID using the second user's token — confirm you get a 403, not a successful update. This single test proves your authorization logic, not just authentication, actually works — exactly the kind of thing an interviewer asking "walk me through how you'd test this" wants to hear.

---

## 4. Consolidated Concept Map of Week 2

| Day | Topic | The ONE thing you must be able to explain without notes |
|---|---|---|
| 8 | Express fundamentals | Middleware is a checkpoint chain; order of registration matters; `next()` vs `next(err)` |
| 9 | REST API design | 401 vs 403, PUT vs PATCH, offset vs cursor pagination tradeoffs |
| 10 | MongoDB/Mongoose | Schema-less DB + app-level schema; indexes trade write speed for read speed; aggregation pipeline stages |
| 11 | SQL/Sequelize/Prisma | N+1 query problem and its fix; migrations vs sync(); transactions and ACID |
| 12 | Authentication | JWT is signed, not encrypted; sessions (stateful) vs JWT (stateless) tradeoffs; refresh token pattern |
| 13 | Errors/logging/security | Operational vs programmer errors; structured logging; CORS is browser-enforced, not a security boundary |
| 14 | (today) | All of the above, combined into one working, authenticated, authorized CRUD API |

---

## 5. Self-Test: Spot the Bug (Week 2 edition)

```javascript
// Bug-hunt #1
app.use('/admin', adminRoutes);
app.use(requireAuth);
// What's wrong? (Hint: Day 8 middleware order)
```
**Answer:** `requireAuth` is registered AFTER `adminRoutes` is mounted — it will never run for any request to `/admin/*`, since those requests are already being handled by `adminRoutes` before `requireAuth` is even reached.

```javascript
// Bug-hunt #2
async function updateTask(req, res) {
  const task = await Task.findByPk(req.params.id);
  await task.update(req.body); // no ownership check!
  res.json(task);
}
// What's wrong? (Hint: Day 14 ownership pattern, Day 12 AuthZ)
```
**Answer:** Missing an ownership check — any authenticated user could update ANY task by guessing/iterating through IDs, since there's no verification that `task.UserId === req.user.userId`. This is a Broken Object Level Authorization bug, a real, commonly cited API vulnerability category.

```javascript
// Bug-hunt #3
const token = jwt.sign({ userId: user.id, password: user.password }, JWT_SECRET);
// What's wrong? (Hint: Day 12, what JWT payloads actually are)
```
**Answer:** Putting the password (even hashed) inside the JWT payload — the payload is only base64-encoded, NOT encrypted, so anyone holding the token can decode and read it. Never put sensitive data in a JWT payload.

---

## 6. Mock Interview Round — Week 2 Edition

1. "Walk me through your project structure and why you organized it that way."
2. "How does your auth middleware work, end to end, from token creation to verification?"
3. "How would you prevent one user from editing another user's data, even if they know the resource's ID?"
4. "Your task list endpoint is slow for users with thousands of tasks — what would you check, and how would you fix it?"
5. "What happens, step by step, when an error is thrown inside one of your controller functions?"
6. "How would you add support for an 'admin' role that can view ALL users' tasks?"
7. "If you needed to support both MongoDB and PostgreSQL clients for different parts of a larger system, what would change in how you write your data layer?"

For question 6 specifically — this is a great moment to demonstrate RBAC in action: add a `requireRole('admin')` check on a separate admin-only route (e.g., `GET /api/v1/admin/tasks`) that skips the `UserId` filter entirely, rather than complicating the existing user-facing route.

---

## What's Different Starting Tomorrow

Weeks 1-2 covered the foundation every Node backend engineer needs daily. Starting Day 15, we move into the topics that distinguish a 3 YOE engineer from someone earlier in their career: **cluster, worker threads, child processes, Redis, message queues, WebSockets, and testing**. These are the topics where interviewers start probing for real depth, since they typically only come up once you've worked on systems facing actual scale or complexity.

## Day 15 Preview
We start Week 3 with the **`cluster` module and Worker Threads** — how Node.js, despite being "single-threaded," can still take advantage of multi-core machines, and the precise difference between when you'd reach for `cluster`, Worker Threads, or `child_process` (tomorrow's continuation).
