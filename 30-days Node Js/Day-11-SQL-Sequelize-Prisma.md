# Day 11 — SQL Databases: Sequelize & Prisma — Relations, Transactions, Migrations

> **Why this day matters:** Most production backend roles at the 3 YOE level work with a relational database (PostgreSQL or MySQL) at least as often as MongoDB, sometimes more. Today gives you the relational counterpart to Day 10 — and critically, it covers **transactions** and **migrations**, two topics that come up constantly in interviews because they reveal whether you've actually operated a production database, not just queried one in a tutorial.

---

## 1. Background: ORMs vs raw SQL — why use either Sequelize or Prisma at all?

**The problem with raw SQL strings in application code:**

```javascript
// Writing raw SQL directly has two real problems:
const result = await db.query(`SELECT * FROM users WHERE email = '${email}'`);
// PROBLEM 1: SQL INJECTION — if 'email' comes from user input and contains
// something like "x'; DROP TABLE users; --", this string gets executed
// AS PART of your SQL query, potentially destroying data. This is one of
// the most well-known, longest-standing security vulnerabilities in
// software, and string-concatenated SQL is exactly how it happens.

// PROBLEM 2: no type safety, no autocomplete, manual mapping of raw rows
// back into JS objects, and queries scattered as strings throughout your
// codebase, making refactors (e.g., renaming a column) error-prone.
```

**An ORM (Object-Relational Mapper)** — Sequelize is the older, more established choice; **Prisma** is the newer, more modern choice — solves both problems: it generates parameterized queries safely under the hood (preventing injection), and lets you work with JS/TS objects and methods instead of raw query strings.

**Interview-relevant nuance:** know that ORMs add a layer of abstraction with real tradeoffs — they can generate inefficient queries for complex cases (the classic "N+1 query problem," covered below) and sometimes you genuinely need to drop down to raw SQL for performance-critical or highly complex queries. Both Sequelize and Prisma support raw queries as an escape hatch for exactly this reason. A senior-leaning answer acknowledges this rather than presenting ORMs as strictly superior to raw SQL in every case.

---

## 2. Sequelize — Setup and Models

```javascript
const { Sequelize, DataTypes } = require('sequelize');

// Connecting to a PostgreSQL database (works similarly for MySQL, just
// change the 'dialect')
const sequelize = new Sequelize('postgres://user:pass@localhost:5432/mydb', {
  dialect: 'postgres',
  logging: false, // set to console.log during debugging to see generated SQL
});

// Defining a Model — this maps directly to a TABLE, similar conceptually
// to a Mongoose Schema/Model from Day 10, but built around SQL's strict
// typed columns instead of flexible documents.
const User = sequelize.define('User', {
  name: {
    type: DataTypes.STRING,
    allowNull: false,
  },
  email: {
    type: DataTypes.STRING,
    allowNull: false,
    unique: true,
    validate: { isEmail: true }, // built-in validators, similar to Mongoose's
  },
  age: {
    type: DataTypes.INTEGER,
    validate: { min: 0 },
  },
}, {
  tableName: 'users', // explicit table name (otherwise Sequelize pluralizes the model name)
  timestamps: true,    // adds createdAt/updatedAt automatically
});

// Syncing the model to the actual database — CREATES the table if it
// doesn't exist. NOTE: in real production apps, you do NOT rely on
// sync() for schema changes — you use MIGRATIONS instead (section 5).
// sync() is mainly for quick prototyping/local dev.
await sequelize.sync();
```

---

## 3. CRUD Operations with Sequelize

```javascript
// --- CREATE ---
const user = await User.create({ name: 'Alice', email: 'alice@test.com', age: 28 });

// --- READ ---
const allUsers = await User.findAll();
const oneUser = await User.findByPk(1); // find by Primary Key
const filtered = await User.findAll({
  where: { age: { [Sequelize.Op.gte]: 18 } }, // SQL: WHERE age >= 18
  order: [['createdAt', 'DESC']],
  limit: 10,
  attributes: ['name', 'email'], // SELECT only these columns (projection)
});

// --- UPDATE ---
await User.update(
  { age: 29 },
  { where: { id: 1 } }
);
// or, on an already-fetched instance:
user.age = 29;
await user.save(); // triggers validation, just like Mongoose's .save()

// --- DELETE ---
await User.destroy({ where: { id: 1 } });
```

---

## 4. Relations — where SQL's real strength shows

### Background: why are relations handled so differently from MongoDB?

In MongoDB (Day 10), you either **embed** related data directly inside a document, or **reference** it via an ID and manually `$lookup` it. In SQL, relationships are a **first-class, structurally enforced concept** via **foreign keys** — the database itself guarantees referential integrity (you genuinely cannot insert an order with a `userId` that doesn't exist in the `users` table, if the foreign key constraint is set up correctly — MongoDB has no equivalent built-in guarantee).

```javascript
// One-to-Many: a User has many Orders, an Order belongs to one User
const Order = sequelize.define('Order', {
  total: DataTypes.DECIMAL(10, 2),
  status: DataTypes.STRING,
});

User.hasMany(Order);       // adds a userId foreign key column to the Order table
Order.belongsTo(User);     // lets you access order.getUser() / include User when querying orders

// Many-to-Many: Students and Courses — a student can enroll in many
// courses, and a course can have many students. This requires a
// JOIN/junction table in between (called 'Enrollment' here).
const Student = sequelize.define('Student', { name: DataTypes.STRING });
const Course = sequelize.define('Course', { title: DataTypes.STRING });

Student.belongsToMany(Course, { through: 'Enrollment' });
Course.belongsToMany(Student, { through: 'Enrollment' });
```

### Querying with JOINs (Sequelize calls this "eager loading" via `include`)

```javascript
// Fetch a user AND all their orders in ONE query (a JOIN under the hood)
const userWithOrders = await User.findOne({
  where: { id: 1 },
  include: Order, // this is the JOIN — without it, you'd only get the user
});

console.log(userWithOrders.Orders); // an array of related order records
```

### The N+1 Query Problem — a critical performance topic asked at the 3 YOE level

**Background: what is it, and how does it sneak into real codebases?**

```javascript
// THE BUG: fetching all users (1 query), then looping through them and
// fetching each one's orders SEPARATELY (N more queries) — for 100 users,
// that's 1 + 100 = 101 total queries, when it could have been just 1 or 2.
const users = await User.findAll();
for (const u of users) {
  const orders = await Order.findAll({ where: { userId: u.id } }); // <-- N+1 problem!
  console.log(u.name, orders.length);
}

// THE FIX: eager-load the relationship in the ORIGINAL query using
// 'include' — Sequelize generates ONE efficient JOIN query (or sometimes
// a couple of well-batched queries) instead of N separate round trips.
const usersWithOrders = await User.findAll({ include: Order });
for (const u of usersWithOrders) {
  console.log(u.name, u.Orders.length); // no extra query needed here!
}
```

**Why this matters so much in interviews:** N+1 is one of the most common, real, production-breaking performance bugs in ORM-based applications. Every extra DB round trip adds network latency; 100 sequential queries instead of 1 can turn a 50ms response into a multi-second one. If an interviewer asks "tell me about a performance issue you've debugged," N+1 detection and fixing is a perfect, very common real-world story to have ready.

---

## 5. Migrations — the part that signals real production experience

### Background: why not just use `sequelize.sync()` everywhere?

`sync()` is fine for quick local prototyping, but it's dangerous in real production environments for a few concrete reasons: it can't safely express things like "rename this column without losing its data," it doesn't give you a history of schema changes over time, and multiple developers/environments (dev, staging, production) need a reliable, repeatable, ORDERED way to apply the SAME schema changes everywhere — which is exactly what migrations provide.

**What a migration is, precisely:** a file describing ONE specific schema change (e.g., "add a `phone` column to the users table"), with an `up` function (apply the change) and a `down` function (undo the change, for rollbacks). Migrations are run in order, and the database keeps track of which ones have already been applied, so running migrations again only applies NEW ones.

```javascript
// migrations/20260101000000-add-phone-to-users.js
module.exports = {
  async up(queryInterface, Sequelize) {
    await queryInterface.addColumn('users', 'phone', {
      type: Sequelize.STRING,
      allowNull: true,
    });
  },

  async down(queryInterface, Sequelize) {
    // The 'down' function REVERSES the 'up' function — critical for
    // safely rolling back a bad deploy in production.
    await queryInterface.removeColumn('users', 'phone');
  },
};
```

```bash
# Common Sequelize CLI commands (you'll genuinely run these at a real job)
npx sequelize-cli migration:generate --name add-phone-to-users
npx sequelize-cli db:migrate          # applies all pending migrations
npx sequelize-cli db:migrate:undo     # rolls back the MOST RECENT migration
```

**Interview tip:** if asked "how do you handle database schema changes in production safely?" — describe migrations explicitly: write a migration file, test it against a staging environment first, run it as part of your deploy pipeline (often automated in CI/CD — connects to Day 27), and always write a working `down` function so a bad migration can be rolled back quickly if something goes wrong in production.

---

## 6. Transactions — a heavily-tested concept

### Background: what problem do transactions solve?

Imagine a "transfer money" operation: subtract $100 from User A's balance, add $100 to User B's balance. These are **two separate database writes**. What happens if the first write succeeds but the server crashes (or the second write fails) before the second one completes? User A loses $100 that vanishes into nowhere — a real, serious data integrity bug.

**A transaction groups multiple operations into one atomic unit: either ALL of them succeed together, or NONE of them are applied at all** (it's automatically rolled back if any step fails). This is the "A" (Atomicity) in **ACID** — a term you should be able to define on the spot.

### ACID — define this clearly if asked

- **Atomicity** — all operations in a transaction succeed, or none do.
- **Consistency** — a transaction can't leave the database in an invalid state (violating constraints).
- **Isolation** — concurrent transactions don't interfere with each other's intermediate states.
- **Durability** — once committed, a transaction's changes survive even a crash immediately after.

```javascript
async function transferMoney(fromUserId, toUserId, amount) {
  // sequelize.transaction() gives you a 't' object you pass into EVERY
  // query that should be part of this atomic group.
  const t = await sequelize.transaction();

  try {
    const fromUser = await User.findByPk(fromUserId, { transaction: t, lock: true });
    const toUser = await User.findByPk(toUserId, { transaction: t, lock: true });

    if (fromUser.balance < amount) {
      throw new Error('Insufficient funds');
    }

    fromUser.balance -= amount;
    toUser.balance += amount;

    await fromUser.save({ transaction: t });
    await toUser.save({ transaction: t });

    // If we reach here, BOTH writes succeeded — commit makes them
    // permanent and visible to other connections.
    await t.commit();
  } catch (err) {
    // ANYTHING going wrong above — an error thrown, a failed query —
    // lands here, and rollback() UNDOES every change made under 't',
    // as if none of it ever happened.
    await t.rollback();
    throw err;
  }
}
```

**Interview tip:** the phrase *"lock: true"* above touches on **row-level locking** — worth mentioning if probed further: it prevents another concurrent transaction from reading/modifying the same row until this transaction finishes, preventing race conditions (e.g., two simultaneous transfers from the same account both passing the balance check before either one actually deducts money — a real double-spend bug class).

---

## 7. Prisma — the modern alternative (briefly, since principles carry over)

**Background: how is Prisma different from Sequelize?** Prisma uses a **schema file** (`schema.prisma`) as the single source of truth for your data model, auto-generates a fully type-safe client based on that schema, and treats migrations as a more integrated, first-class workflow rather than something bolted on via a separate CLI tool. Many newer Node.js/TypeScript projects default to Prisma today because of its strong type safety and developer experience.

```prisma
// schema.prisma
model User {
  id     Int     @id @default(autoincrement())
  name   String
  email  String  @unique
  orders Order[] // one-to-many relation
}

model Order {
  id     Int    @id @default(autoincrement())
  total  Float
  userId Int
  user   User   @relation(fields: [userId], references: [id])
}
```

```javascript
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

// Same relational concepts as Sequelize, different syntax
const userWithOrders = await prisma.user.findUnique({
  where: { id: 1 },
  include: { orders: true }, // eager loading, same idea as Sequelize's 'include'
});

// Transactions in Prisma
await prisma.$transaction([
  prisma.user.update({ where: { id: 1 }, data: { balance: { decrement: 100 } } }),
  prisma.user.update({ where: { id: 2 }, data: { balance: { increment: 100 } } }),
]);
```

```bash
npx prisma migrate dev --name add-phone-to-users  # creates AND applies a migration in dev
npx prisma migrate deploy                          # applies pending migrations in production
```

**Interview tip:** if asked "Sequelize or Prisma, which would you choose?" — a strong answer acknowledges both are valid: Sequelize is more mature with a longer track record and more flexibility for complex/legacy use cases; Prisma offers stronger type safety and a smoother developer experience, especially in TypeScript projects, but is comparatively newer. Mentioning you're comfortable with both (since the underlying relational concepts — relations, transactions, migrations — are identical) is a strong, honest answer.

---

## 8. How this connects to real backend work (3 YOE framing)

- **"Your API response time degraded specifically on a page that lists users with their recent orders — what would you investigate?"** → The N+1 query problem — check whether the ORM is eager-loading relations (`include`) or silently issuing a separate query per row in a loop.
- **"How do you handle changing the database schema across dev, staging, and production safely?"** → Migrations — versioned, ordered, reversible schema change files run consistently across all environments, ideally as part of an automated deploy pipeline.
- **"Walk me through how you'd implement a 'transfer funds between accounts' feature safely."** → A database transaction wrapping both the debit and credit operations, with proper locking to prevent race conditions, and a rollback path if any step fails.
- **"Why might you choose raw SQL over your ORM for a specific query?"** → Complex reporting queries, heavy aggregations, or anything where the ORM's generated SQL would be measurably less efficient than a hand-tuned query — both Sequelize and Prisma support raw queries as an escape hatch for exactly this.

---

## 9. Hands-on practice for today

```javascript
// practice-day11.js (Sequelize, SQLite for easy local testing — no setup needed)
const { Sequelize, DataTypes } = require('sequelize');
const sequelize = new Sequelize('sqlite::memory:', { logging: false });

const User = sequelize.define('User', {
  name: DataTypes.STRING,
  balance: { type: DataTypes.INTEGER, defaultValue: 0 },
});

const Order = sequelize.define('Order', {
  total: DataTypes.INTEGER,
});

User.hasMany(Order);
Order.belongsTo(User);

async function run() {
  await sequelize.sync();

  const alice = await User.create({ name: 'Alice', balance: 500 });
  const bob = await User.create({ name: 'Bob', balance: 100 });

  await Order.create({ total: 50, UserId: alice.id });
  await Order.create({ total: 30, UserId: alice.id });

  // Practice eager loading (avoiding N+1)
  const usersWithOrders = await User.findAll({ include: Order });
  usersWithOrders.forEach((u) => {
    console.log(u.name, 'has', u.Orders.length, 'orders');
  });

  // Practice a transaction: transfer 100 from Alice to Bob
  const t = await sequelize.transaction();
  try {
    alice.balance -= 100;
    bob.balance += 100;
    await alice.save({ transaction: t });
    await bob.save({ transaction: t });
    await t.commit();
    console.log('Transfer succeeded');
  } catch (err) {
    await t.rollback();
    console.error('Transfer failed, rolled back:', err.message);
  }

  const refreshedAlice = await User.findByPk(alice.id);
  const refreshedBob = await User.findByPk(bob.id);
  console.log('Alice balance:', refreshedAlice.balance); // 400
  console.log('Bob balance:', refreshedBob.balance);     // 200
}

run();
```

Run this with `node practice-day11.js` (after `npm install sequelize sqlite3`) — no real database server needed since it uses an in-memory SQLite instance, making it easy to experiment with relations and transactions immediately.

---

## 10. Interview Q&A Recap (say these out loud)

1. **Q: Why use an ORM instead of writing raw SQL strings directly?**
   A: Primarily SQL injection prevention (ORMs generate parameterized queries safely), plus type safety, autocomplete, and centralized query logic instead of scattered raw strings — though raw SQL is still appropriate for complex/performance-critical queries the ORM can't express efficiently.

2. **Q: What's the N+1 query problem and how do you fix it?**
   A: Fetching a list of records (1 query), then looping through them and issuing a separate query per record to fetch related data (N more queries) — instead of eager-loading the relation in the original query via `include` (Sequelize) or similar, which generates one efficient JOIN-based query.

3. **Q: What's the difference between `sequelize.sync()` and migrations, and why do production apps use migrations?**
   A: `sync()` auto-creates tables based on current model definitions but can't safely express incremental changes (like renaming a column without data loss) and has no change history. Migrations are versioned, ordered, reversible files describing one schema change at a time, applied consistently and safely across all environments.

4. **Q: Explain ACID, specifically Atomicity, in the context of a database transaction.**
   A: Atomicity means all operations within a transaction succeed together, or none of them take effect at all — if any step fails, the entire transaction is rolled back as if nothing happened, preventing partial/inconsistent writes (e.g., money debited from one account but never credited to another).

5. **Q: How does MongoDB's approach to relationships differ from SQL's?**
   A: SQL enforces relationships structurally via foreign keys, with the database itself guaranteeing referential integrity. MongoDB has no native equivalent — relationships are either embedded directly in documents or referenced by ID and manually joined via `$lookup`, with no built-in integrity guarantee.

6. **Q: When would you reach for raw SQL instead of your ORM?**
   A: Complex reporting/analytics queries, heavy aggregations, or any case where the ORM's generated query would be measurably less efficient than a hand-written one — both Sequelize and Prisma support raw queries as an escape hatch for exactly this.

---

## Tomorrow (Day 12) Preview
We cover **Authentication**: JWT (how it actually works internally, not just `jwt.sign()`), sessions vs token-based auth, password hashing with bcrypt, OAuth basics, and role-based access control (RBAC) — one of the most universally asked topics in any backend interview, regardless of database choice.
