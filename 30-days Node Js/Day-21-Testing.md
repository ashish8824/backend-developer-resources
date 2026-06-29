# Day 21 — Testing: Jest, Unit vs Integration Tests, Mocking, Supertest

> **Why this day matters:** At 3 YOE, "do you write tests?" isn't really the question — it's assumed you do. The REAL question is whether you understand **what kind of test is appropriate for what situation**, **how to isolate a unit from its dependencies** (mocking), and **how to test an API endpoint properly**. Weak answers here ("I write tests sometimes") signal junior-level testing discipline. Today builds the vocabulary and patterns that signal real production testing experience.

---

## 1. Background: why do we even categorize tests into different types?

Not all tests serve the same purpose, and treating them as interchangeable leads to slow, brittle, or insufficient test suites. The classic hierarchy (often visualized as a "testing pyramid"):

```
        ▲
       / \      E2E (End-to-End) -- few, slow, test the whole system
      /───\     through a real browser/client, highest confidence but
     /     \    most expensive to write/maintain/run
    /───────\
   /         \  Integration -- moderate number, test how multiple
  /───────────\ pieces work TOGETHER (e.g., a route handler + real DB)
 /             \
/───────────────\ Unit -- MANY, fast, test ONE piece of logic in
                   isolation, with dependencies mocked/faked
```

**Why this shape (more unit tests, fewer E2E tests) — the actual reasoning, not just a diagram to memorize:** unit tests are fast (milliseconds) and pinpoint exactly what broke when they fail, so you want MANY of them covering detailed logic branches. E2E tests are slow (seconds/minutes) and broad (a failure could be caused by almost anything), so you want FEW of them, reserved for confirming critical user flows actually work end-to-end. Integration tests sit in between — more realistic than unit tests, faster and more isolated than full E2E.

---

## 2. Unit Tests — testing ONE thing in isolation

### Background: what makes a test a "unit" test specifically?

A unit test verifies a single function or module's logic, **with all of its external dependencies replaced by fakes** (mocks/stubs) — so a failure can ONLY be caused by a bug in the unit itself, not by a database being down, a network call failing, or some unrelated module's bug.

```javascript
// math.js -- the "unit" we're testing
function calculateDiscount(price, discountPercent) {
  if (discountPercent < 0 || discountPercent > 100) {
    throw new Error('Discount percent must be between 0 and 100');
  }
  return price - (price * discountPercent) / 100;
}

module.exports = { calculateDiscount };
```

```javascript
// math.test.js -- using Jest (the most common Node.js testing framework today)
const { calculateDiscount } = require('./math');

// 'describe' groups related tests together -- purely organizational
describe('calculateDiscount', () => {
  // 'test' (or its alias 'it') defines ONE specific test case
  test('applies a 10% discount correctly', () => {
    // 'expect().toBe()' is an ASSERTION -- it checks the actual result
    // against the expected result, and the test fails if they don't match
    expect(calculateDiscount(100, 10)).toBe(90);
  });

  test('applies a 0% discount (no change)', () => {
    expect(calculateDiscount(100, 0)).toBe(100);
  });

  test('applies a 100% discount (free)', () => {
    expect(calculateDiscount(100, 100)).toBe(0);
  });

  // Testing the ERROR-HANDLING path is just as important as testing
  // the "happy path" -- this is a common gap in weaker test suites
  test('throws an error for negative discount percent', () => {
    expect(() => calculateDiscount(100, -10)).toThrow('Discount percent must be between 0 and 100');
  });

  test('throws an error for discount percent over 100', () => {
    expect(() => calculateDiscount(100, 150)).toThrow();
  });
});
```

```bash
npx jest math.test.js   # run a specific test file
npx jest                # run the entire test suite
npx jest --watch        # re-run tests automatically as files change -- useful during active development
npx jest --coverage     # generates a report of what % of your code is actually exercised by tests
```

### Common Jest Matchers (the assertion methods) worth knowing

```javascript
expect(value).toBe(5);                    // strict equality (===), for primitives
expect(obj).toEqual({ a: 1, b: 2 });       // deep equality, for objects/arrays
expect(value).toBeNull();
expect(value).toBeUndefined();
expect(value).toBeTruthy();
expect(array).toContain('item');
expect(fn).toThrow();
expect(asyncFn()).resolves.toBe('value');  // for testing resolved Promises
expect(asyncFn()).rejects.toThrow();       // for testing rejected Promises
```

---

## 3. Mocking — the technique that makes true unit testing possible

### Background: why can't you just test the real database/API call directly?

If your function calls a real database or a real external API, your "unit" test isn't really testing a unit anymore — it's an integration test in disguise, AND it becomes slow, flaky (network issues, rate limits), and dependent on external state (what if the test database has different data tomorrow?). **Mocking replaces the real dependency with a fake, controllable stand-in**, so you can test YOUR code's logic in complete isolation, deciding exactly what the "dependency" returns for each test case.

```javascript
// userService.js -- depends on an external module (the database layer)
const User = require('./models/User');

async function getUserDisplayName(userId) {
  const user = await User.findByPk(userId);
  if (!user) return 'Unknown User';
  return user.nickname || user.name;
}

module.exports = { getUserDisplayName };
```

```javascript
// userService.test.js
const { getUserDisplayName } = require('./userService');
const User = require('./models/User');

// jest.mock() REPLACES the entire module with an automatically-generated
// fake version -- every exported function becomes a "mock function" that
// you control the behavior of, instead of running real database code.
jest.mock('./models/User');

describe('getUserDisplayName', () => {
  // Clearing mocks between tests prevents one test's mock setup from
  // accidentally leaking into and affecting another test
  afterEach(() => {
    jest.clearAllMocks();
  });

  test('returns nickname if it exists', async () => {
    // Configuring exactly what the MOCKED findByPk should return for
    // THIS specific test -- we're fully in control, no real DB involved
    User.findByPk.mockResolvedValue({ name: 'Alice Smith', nickname: 'Ali' });

    const result = await getUserDisplayName(1);
    expect(result).toBe('Ali');
  });

  test('falls back to full name if no nickname exists', async () => {
    User.findByPk.mockResolvedValue({ name: 'Bob Jones', nickname: null });

    const result = await getUserDisplayName(2);
    expect(result).toBe('Bob Jones');
  });

  test('returns "Unknown User" if user does not exist', async () => {
    User.findByPk.mockResolvedValue(null);

    const result = await getUserDisplayName(999);
    expect(result).toBe('Unknown User');
  });

  test('can verify the mock was called with the correct argument', async () => {
    User.findByPk.mockResolvedValue({ name: 'Carol' });

    await getUserDisplayName(42);

    // Beyond checking the RETURN value, you can also verify HOW a
    // dependency was called -- useful for confirming your code passes
    // the right arguments downstream, without needing the real dependency
    expect(User.findByPk).toHaveBeenCalledWith(42);
    expect(User.findByPk).toHaveBeenCalledTimes(1);
  });
});
```

**Mocks vs Stubs vs Spies — a terminology distinction interviewers sometimes probe:**
- **Stub** — a fake that returns a predetermined value, with no real logic (`User.findByPk.mockResolvedValue(...)` above is functioning as a stub).
- **Mock** — similar to a stub, but specifically used when you also want to ASSERT on how it was called (`toHaveBeenCalledWith`).
- **Spy** — wraps a REAL function, letting it still execute normally WHILE ALSO recording how it was called — useful when you want the real behavior but also want to verify call details (`jest.spyOn`).

```javascript
const utils = require('./utils');
const spy = jest.spyOn(utils, 'formatDate'); // the REAL formatDate still runs
formatSomething(); // some code that internally calls utils.formatDate
expect(spy).toHaveBeenCalled(); // but we can still verify it was called
```

---

## 4. Integration Tests — testing multiple pieces working together

### Background: what's actually different here, compared to a unit test?

An integration test deliberately does NOT mock everything — it tests how real pieces interact (e.g., your route handler + middleware + an ACTUAL test database), closer to how the system behaves in production, at the cost of being slower and slightly less isolated when something fails (a failure could be in the route logic OR the database query OR the middleware).

```javascript
// A common, practical pattern: use a REAL but ISOLATED test database
// (often an in-memory SQLite instance, exactly like Day 11's practice
// file) rather than mocking the entire data layer -- this verifies
// your actual SQL/Sequelize queries work correctly, not just that your
// JS logic CALLS the right mocked functions.
const { Sequelize, DataTypes } = require('sequelize');

let sequelize;
let User;

// beforeAll runs ONCE before all tests in this file -- good for
// expensive setup you don't want to repeat per-test
beforeAll(async () => {
  sequelize = new Sequelize('sqlite::memory:', { logging: false });
  User = sequelize.define('User', { name: DataTypes.STRING, email: DataTypes.STRING });
  await sequelize.sync();
});

// beforeEach runs before EVERY individual test -- good for resetting
// state so tests don't interfere with each other
beforeEach(async () => {
  await User.destroy({ where: {}, truncate: true }); // clean slate for every test
});

afterAll(async () => {
  await sequelize.close();
});

describe('User model integration', () => {
  test('creates and retrieves a user from a real (in-memory) database', async () => {
    await User.create({ name: 'Alice', email: 'alice@test.com' });

    const found = await User.findOne({ where: { email: 'alice@test.com' } });
    expect(found.name).toBe('Alice');
  });

  test('enforces unique constraints correctly', async () => {
    // This test would need a 'unique: true' validator set up on email
    // to actually demonstrate Sequelize's real constraint enforcement
    // -- the point is, this is testing REAL database behavior, not a mock
  });
});
```

**Interview tip:** if asked "do you mock the database in your tests," the nuanced, correct answer is: *"It depends on the test's purpose. For unit tests of pure business logic, I mock the database layer to isolate the logic itself. For integration tests verifying the data layer actually behaves correctly (constraints, relations, query correctness), I use a real but isolated test database — often in-memory SQLite or a dedicated test database/schema — rather than mocking, since the whole point is verifying the REAL interaction."*

---

## 5. Testing Express Routes with Supertest

### Background: why is testing an HTTP route different from testing a plain function?

A route handler needs an actual HTTP request to trigger it — you can't just call `app.get('/users')` like a normal function. **Supertest** lets you make real HTTP requests directly against your Express app IN MEMORY (no need to actually start a listening server on a real port), and assert on the response.

```javascript
// app.js (a small Express app for testing)
const express = require('express');
const app = express();
app.use(express.json());

const users = [{ id: 1, name: 'Alice' }];

app.get('/users/:id', (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ error: 'User not found' });
  res.json(user);
});

app.post('/users', (req, res) => {
  if (!req.body.name) return res.status(400).json({ error: 'Name is required' });
  const newUser = { id: users.length + 1, name: req.body.name };
  users.push(newUser);
  res.status(201).json(newUser);
});

module.exports = app; // exporting WITHOUT calling .listen() -- Supertest handles that internally
```

```javascript
// app.test.js
const request = require('supertest');
const app = require('./app');

describe('GET /users/:id', () => {
  test('returns a user if found', async () => {
    const response = await request(app).get('/users/1');

    expect(response.status).toBe(200);
    expect(response.body).toEqual({ id: 1, name: 'Alice' });
  });

  test('returns 404 if user is not found', async () => {
    const response = await request(app).get('/users/999');

    expect(response.status).toBe(404);
    expect(response.body.error).toBe('User not found');
  });
});

describe('POST /users', () => {
  test('creates a new user with valid input', async () => {
    const response = await request(app)
      .post('/users')
      .send({ name: 'Bob' }) // sends a JSON body
      .set('Content-Type', 'application/json');

    expect(response.status).toBe(201);
    expect(response.body.name).toBe('Bob');
  });

  test('returns 400 if name is missing', async () => {
    const response = await request(app).post('/users').send({});

    expect(response.status).toBe(400);
    expect(response.body.error).toBe('Name is required');
  });
});
```

**Testing protected/authenticated routes (combining Day 12's JWT knowledge):**

```javascript
describe('GET /profile (protected route)', () => {
  test('returns 401 without a token', async () => {
    const response = await request(app).get('/profile');
    expect(response.status).toBe(401);
  });

  test('returns the profile with a valid token', async () => {
    const token = jwt.sign({ userId: 1, role: 'user' }, process.env.JWT_SECRET);

    const response = await request(app)
      .get('/profile')
      .set('Authorization', `Bearer ${token}`); // attaching the auth header, just like a real client would

    expect(response.status).toBe(200);
  });
});
```

---

## 6. Jest vs Mocha — briefly, since both come up by name

**Background:** Mocha is an older, more minimal test runner — it provides the test structure (`describe`/`it`) but deliberately leaves assertions and mocking to SEPARATE libraries (commonly `chai` for assertions, `sinon` for mocks/spies). **Jest** is a more "batteries-included" framework — it bundles test running, assertions, and mocking all together in one package, which is why Jest has become the more common default choice for new Node.js projects today, since it requires less setup/configuration to get a fully working test suite.

```javascript
// Mocha + Chai + Sinon equivalent of a Jest test, for comparison
const { expect } = require('chai');
const sinon = require('sinon');

describe('calculateDiscount', () => {
  it('applies a 10% discount correctly', () => {
    expect(calculateDiscount(100, 10)).to.equal(90); // chai's syntax differs slightly from Jest's
  });
});
```

**Interview tip:** if asked "Jest or Mocha, which do you prefer," a fine honest answer is that Jest's all-in-one nature reduces setup friction and is what you'd default to for a new project, but Mocha's more modular approach (mixing and matching your preferred assertion/mocking libraries) remains common in many existing/legacy codebases, and the underlying testing CONCEPTS (describe/it, assertions, mocking) transfer directly between them.

---

## 7. How this connects to real backend work (3 YOE framing)

- **"How would you test a function that calls a third-party payment API, without actually charging a real card during your test run?"** → Mock the payment API client/module entirely, controlling exactly what it returns (success, specific error types) for each test case, keeping the test fast and not dependent on a real external service.
- **"Your test suite passes locally but fails in CI — what are common causes?"** → Tests depending on shared/leftover state between runs (missing proper `beforeEach` cleanup), environment differences (missing env vars in CI), or tests that aren't properly isolated from each other (one test's side effects bleeding into another).
- **"How would you decide whether something needs a unit test or an integration test?"** → Pure business logic with no external dependencies (calculations, validation rules, data transformations) → unit test with mocks. Behavior that depends on real interaction between pieces (does this Sequelize query actually return the right rows, does this middleware chain actually reject unauthorized requests) → integration test.
- **"How do you test a protected API route that requires authentication?"** → Generate a valid test JWT directly (using the same secret/logic as your app) and attach it via the `Authorization` header in your Supertest request, exactly as a real authenticated client would.

---

## 8. Hands-on practice for today

```javascript
// practice-day21.test.js
const request = require('supertest');
const express = require('express');

function createApp() {
  const app = express();
  app.use(express.json());

  const tasks = [];

  app.post('/tasks', (req, res) => {
    if (!req.body.title) return res.status(400).json({ error: 'Title is required' });
    const task = { id: tasks.length + 1, title: req.body.title, completed: false };
    tasks.push(task);
    res.status(201).json(task);
  });

  app.get('/tasks', (req, res) => res.json(tasks));

  app.patch('/tasks/:id/complete', (req, res) => {
    const task = tasks.find((t) => t.id === parseInt(req.params.id));
    if (!task) return res.status(404).json({ error: 'Task not found' });
    task.completed = true;
    res.json(task);
  });

  return app;
}

describe('Task API', () => {
  let app;

  beforeEach(() => {
    app = createApp(); // fresh app + fresh in-memory tasks for EVERY test, avoiding cross-test pollution
  });

  test('creates a task', async () => {
    const res = await request(app).post('/tasks').send({ title: 'Learn testing' });
    expect(res.status).toBe(201);
    expect(res.body.title).toBe('Learn testing');
    expect(res.body.completed).toBe(false);
  });

  test('rejects a task with no title', async () => {
    const res = await request(app).post('/tasks').send({});
    expect(res.status).toBe(400);
  });

  test('marks a task as complete', async () => {
    const created = await request(app).post('/tasks').send({ title: 'Test me' });
    const res = await request(app).patch(`/tasks/${created.body.id}/complete`);
    expect(res.body.completed).toBe(true);
  });

  test('returns 404 when completing a non-existent task', async () => {
    const res = await request(app).patch('/tasks/999/complete');
    expect(res.status).toBe(404);
  });
});
```

Run with `npx jest practice-day21.test.js` (after `npm install --save-dev jest supertest`) and confirm all 4 tests pass — then deliberately break the app logic (e.g., remove the title validation) and watch the corresponding test correctly catch the regression.

---

## 9. Interview Q&A Recap (say these out loud)

1. **Q: What's the difference between a unit test and an integration test?**
   A: A unit test isolates ONE piece of logic, mocking all external dependencies, so failures point precisely to a bug in that unit. An integration test verifies how multiple real pieces work together (e.g., a route handler with an actual test database), trading some isolation/speed for more realistic coverage of how components interact.

2. **Q: Why mock a database call in a unit test instead of just using a real test database?**
   A: Mocking keeps the test fast, deterministic, and fully isolated — failures can only come from the unit's own logic, not from network issues, data setup problems, or external state. Real database interaction belongs in integration tests, where verifying that actual interaction is the whole point.

3. **Q: What's the difference between a stub, a mock, and a spy?**
   A: A stub returns a predetermined value with no real logic. A mock is similar but typically also used to assert HOW it was called. A spy wraps a real function, letting it execute normally while still recording call details for later assertions.

4. **Q: How does Supertest let you test an Express route without starting a real server on a port?**
   A: It takes your exported Express `app` instance directly and simulates HTTP requests against it in memory, asserting on the resulting response object, without needing an actual listening network socket.

5. **Q: Why is testing the error/failure path of a function just as important as testing the success path?**
   A: Bugs often hide in edge cases and error handling rather than the main happy path — a function that handles valid input correctly but mishandles invalid input, missing dependencies, or thrown errors can still cause real production incidents; thorough tests cover both deliberately.

6. **Q: What's the difference between Jest and Mocha at a conceptual level?**
   A: Jest is an all-in-one framework bundling test running, assertions, and mocking together. Mocha is a more minimal test runner that's typically paired with separate libraries (like Chai for assertions, Sinon for mocks) — the underlying testing concepts are the same, but Jest requires less setup out of the box.

---

## Day 22 Preview — Week 4 Begins: Rate Limiting Algorithms Deep Dive
Week 3 is complete — you've now covered cluster/worker threads/child processes, Redis (caching, pub/sub, sessions, rate limiting), message queues, WebSockets, and testing. Starting Day 22, Week 4 shifts into **system design and DevOps**: a focused deep dive specifically on rate limiting algorithms (building further on Day 18's foundation with a cleaner, interview-ready comparison), followed by load balancing, caching strategies at scale, microservices, Docker, CI/CD, and performance profiling — closing the gap between "I can build an API" and "I understand how production systems are actually run."
