# Day 10 — MongoDB + Mongoose: Schemas, CRUD, Indexes, Aggregation

> **Why this day matters:** MongoDB is one of the most common databases paired with Node.js (the "M" in the old MEAN/MERN stack acronyms), and Mongoose is the dominant ODM (Object-Document Mapper) for it. At 3 YOE, interviewers expect you to go beyond basic CRUD — they'll probe whether you understand **why indexes matter for performance**, **when MongoDB is the right choice vs a relational DB**, and **how the aggregation pipeline works**, because these are the things that separate "I used MongoDB in a tutorial" from "I've operated MongoDB in production."

---

## 1. Background: what is MongoDB, and why does it look so different from SQL?

**The core idea:** MongoDB is a **document database** (a type of NoSQL database). Instead of storing data in rigid rows and columns (like a SQL table), it stores data as **documents** — JSON-like structures called **BSON** (Binary JSON) — grouped into **collections**. This maps very naturally onto JavaScript objects, which is a big part of why it pairs so well with Node.js: you're often passing data between your app and your database with very little translation needed.

```javascript
// A SQL "users" table would force every row to have the EXACT same columns.
// A MongoDB "users" COLLECTION can have documents with DIFFERENT shapes —
// this is called being "schema-less" at the database level (though, as
// we'll see, Mongoose lets US enforce a schema at the APPLICATION level,
// which most real production apps do anyway for safety and predictability).

// Example documents in a "users" collection:
{ _id: ObjectId("..."), name: "Alice", email: "alice@test.com" }
{ _id: ObjectId("..."), name: "Bob", email: "bob@test.com", phone: "555-1234" }
// Notice Bob has a 'phone' field that Alice doesn't — MongoDB allows this
// natively. A SQL table couldn't do this without a NULL column for everyone.
```

### SQL vs NoSQL — the comparison interviewers actually want to hear

This question ("when would you use MongoDB vs PostgreSQL/MySQL?") comes up constantly at the 3 YOE level. Give a real, nuanced answer — not "NoSQL is more scalable" (too vague and often wrong).

| Factor | Relational (SQL) | MongoDB (NoSQL/Document) |
|---|---|---|
| Schema | Rigid, enforced at the DB level | Flexible; schema enforced at app level (if at all) |
| Relationships | Strong support via foreign keys, JOINs | Weaker native JOIN support (`$lookup` exists but is less efficient); favors embedding related data |
| Transactions | Mature, strong ACID guarantees historically | Supports multi-document ACID transactions (since v4.0), but historically weaker in this area |
| Best fit | Data with clear, stable structure and complex relationships (e.g., financial systems, inventory with many relations) | Data that's naturally document-shaped, evolves frequently, or has variable structure (e.g., content management, catalogs, logging, user profiles with optional fields) |
| Scaling | Traditionally vertical (though modern Postgres/MySQL scale horizontally too with effort) | Designed with horizontal sharding in mind from the start |

**The honest, senior-level answer:** "It depends on the data's shape and access patterns, not on some inherent 'NoSQL is faster' myth. If my data has many relationships that need strong consistency and complex queries across them, I'd lean relational. If my data is naturally document-shaped, doesn't need complex cross-entity joins, and the schema is likely to evolve, MongoDB is a strong fit." Saying this in an interview signals real judgment, not just memorized buzzwords.

---

## 2. Mongoose — why use an ODM at all?

**Background: the problem with the raw MongoDB driver.** You CAN talk to MongoDB directly with the official `mongodb` npm package, but you'd be writing raw, unvalidated queries with no structure enforcement, no built-in validation, and a lot of repetitive boilerplate. Mongoose sits on top of the raw driver and gives you:
- **Schemas** — define expected structure, types, and validation rules at the application level (compensating for MongoDB's lack of enforced schema)
- **Models** — classes representing a collection, with built-in methods for CRUD
- **Middleware (hooks)** — functions that run before/after certain operations (e.g., hashing a password before saving)
- **Query building** — a friendlier, chainable API over raw MongoDB query syntax

```javascript
const mongoose = require('mongoose');

// Connecting to MongoDB
mongoose.connect('mongodb://localhost:27017/myapp')
  .then(() => console.log('Connected to MongoDB'))
  .catch((err) => console.error('Connection error:', err));
```

---

## 3. Defining Schemas and Models

```javascript
const mongoose = require('mongoose');
const { Schema } = mongoose;

// A Schema defines the SHAPE and RULES for documents in a collection.
const userSchema = new Schema({
  name: {
    type: String,
    required: [true, 'Name is required'], // built-in validation with custom error message
    trim: true, // automatically removes leading/trailing whitespace
  },
  email: {
    type: String,
    required: true,
    unique: true, // enforces uniqueness at the DATABASE level (creates a unique index!)
    lowercase: true, // automatically lowercases before saving
  },
  age: {
    type: Number,
    min: 0, // built-in numeric validation
    max: 120,
  },
  role: {
    type: String,
    enum: ['user', 'admin'], // restricts value to ONE of these exact strings
    default: 'user', // applied automatically if not provided
  },
  // A REFERENCE to another collection — this is how MongoDB/Mongoose
  // models relationships, similar to a foreign key in SQL, but the
  // database itself doesn't enforce referential integrity the way
  // SQL foreign keys do — Mongoose just stores the referenced ID.
  createdBy: {
    type: Schema.Types.ObjectId,
    ref: 'User', // tells Mongoose this ID refers to a document in the 'User' model
  },
}, {
  timestamps: true, // automatically adds and manages createdAt/updatedAt fields
});

// A Model is the actual interface you use to query/create documents.
// Mongoose automatically looks for (and creates, if needed) a collection
// named the LOWERCASE, PLURALIZED version of this string — 'users'.
const User = mongoose.model('User', userSchema);

module.exports = User;
```

**Interview tip:** if asked "does MongoDB enforce schemas?" — the precise, correct answer is: *"MongoDB itself is schema-less at the database engine level — it'll happily store documents with different shapes in the same collection. Mongoose adds an application-level schema layer on top, which is what most production Node apps rely on for data consistency and validation, even though the underlying database wouldn't enforce it on its own."*

---

## 4. CRUD Operations

```javascript
const User = require('./models/User');

// --- CREATE ---
async function createUser() {
  const user = new User({ name: 'Alice', email: 'alice@test.com', age: 28 });
  await user.save(); // runs schema validation BEFORE inserting into the DB
  return user;
}

// Or, more commonly in route handlers, the shorthand:
async function createUserShorthand(data) {
  return await User.create(data); // equivalent to `new User(data).save()`
}

// --- READ ---
async function findUsers() {
  const allUsers = await User.find();                      // get everything
  const activeUsers = await User.find({ role: 'admin' });   // filter by field
  const oneUser = await User.findById('64f...');             // find by _id
  const oneByEmail = await User.findOne({ email: 'a@test.com' }); // find first match

  // Chainable query building — Mongoose queries are NOT executed until
  // you await them (or call .exec()) — this lets you build up complex
  // queries piece by piece before running them.
  const filtered = await User
    .find({ age: { $gte: 18 } })  // MongoDB operators: $gte = greater than or equal
    .select('name email')          // only return these fields (projection)
    .sort({ createdAt: -1 })       // -1 = descending, 1 = ascending
    .limit(10);
}

// --- UPDATE ---
async function updateUser(id, updates) {
  // findByIdAndUpdate: finds by ID and applies updates in one call.
  // { new: true } returns the UPDATED document instead of the original.
  // { runValidators: true } makes sure schema validation rules still
  // apply on update (NOT automatic by default — a common gotcha!)
  const updated = await User.findByIdAndUpdate(id, updates, {
    new: true,
    runValidators: true,
  });
  return updated;
}

// --- DELETE ---
async function deleteUser(id) {
  await User.findByIdAndDelete(id);
}
```

**Common gotcha worth knowing for interviews:** by default, `findByIdAndUpdate` and similar update methods **skip schema validation** unless you explicitly pass `{ runValidators: true }`. This surprises a lot of engineers — they assume validation always applies, then discover invalid data slipped through an update operation in production.

---

## 5. Indexes — the part that actually matters for performance

### Background: why does an index matter at all?

**Without an index**, MongoDB performs a **collection scan** — it checks every single document in the collection to find matches for your query, one by one. With a million documents, that's potentially a million comparisons for every query — extremely slow.

**With an index**, MongoDB maintains a separate, sorted data structure (similar conceptually to an index at the back of a book) that lets it jump almost directly to matching documents, without scanning everything. This is the single biggest lever for query performance in any database, SQL or NoSQL.

```javascript
// Mongoose automatically creates an index for any field marked 'unique: true'
// (as we saw with 'email' above) — but for fields you frequently QUERY or
// SORT by, even without a uniqueness requirement, you should add an index
// explicitly.

const orderSchema = new Schema({
  userId: { type: Schema.Types.ObjectId, ref: 'User' },
  status: String,
  createdAt: { type: Date, default: Date.now },
});

// Single-field index — speeds up queries filtering by 'status'
orderSchema.index({ status: 1 }); // 1 = ascending order

// Compound index — speeds up queries that filter/sort by MULTIPLE fields
// TOGETHER. Order matters here: this index is most useful for queries
// filtering by userId FIRST, then status — not the other way around.
orderSchema.index({ userId: 1, status: 1 });

// Checking what indexes actually exist and how a query performs —
// .explain() is the real-world tool engineers use to debug slow queries
async function checkQueryPerformance() {
  const result = await Order.find({ status: 'pending' }).explain('executionStats');
  console.log(result.executionStats.totalDocsExamined); // how many docs were scanned
  console.log(result.executionStats.executionTimeMillis);
}
```

**The tradeoff you must mention if asked "should we just index everything?":** indexes speed up reads but **slow down writes** (every insert/update/delete must also update every relevant index) and **consume additional disk/memory**. The real-world skill is indexing fields that are actually queried/sorted/filtered on frequently — not blindly indexing every field. This nuance ("indexes aren't free") is exactly what separates a thoughtful answer from a memorized one.

---

## 6. The Aggregation Pipeline — MongoDB's answer to complex SQL queries

### Background: why does this exist?

Simple `.find()` queries are great for filtering and sorting, but they can't do things like "group orders by user and sum their totals" or "calculate average order value per month." The **aggregation pipeline** is a sequence of **stages**, where each stage transforms the data and passes it to the next stage — similar conceptually to piping commands together in a Unix shell, or to the `.map().filter().reduce()` chain pattern in JavaScript, but running INSIDE the database for efficiency.

```javascript
// Real example: "For each user, calculate their total spending and number
// of orders, but only include users who've spent more than $500, sorted
// by total spending descending."
const result = await Order.aggregate([
  // STAGE 1: $match — filters documents (like a WHERE clause in SQL).
  // Doing this FIRST is a performance best practice — filter down the
  // dataset as early as possible so later stages process less data.
  { $match: { status: 'completed' } },

  // STAGE 2: $group — groups documents by a field and computes
  // aggregated values per group (like GROUP BY + SUM/COUNT in SQL).
  {
    $group: {
      _id: '$userId',                  // group by this field
      totalSpent: { $sum: '$amount' },  // sum the 'amount' field per group
      orderCount: { $sum: 1 },          // count documents per group
    },
  },

  // STAGE 3: $match (again) — filtering AFTER aggregation, equivalent
  // to a HAVING clause in SQL (filtering on an aggregated value).
  { $match: { totalSpent: { $gt: 500 } } },

  // STAGE 4: $sort — sort the results
  { $sort: { totalSpent: -1 } },

  // STAGE 5: $lookup — MongoDB's version of a SQL JOIN. Pulls in
  // related data from another collection.
  {
    $lookup: {
      from: 'users',           // the collection to join with
      localField: '_id',        // field from THIS stage's documents (the grouped userId)
      foreignField: '_id',      // field on the 'users' collection to match against
      as: 'userDetails',        // name of the new array field holding matched user(s)
    },
  },

  // STAGE 6: $project — reshape the output, choosing exactly which
  // fields to include/exclude/rename in the final result.
  {
    $project: {
      totalSpent: 1,
      orderCount: 1,
      userName: { $arrayElemAt: ['$userDetails.name', 0] }, // pull the name out of the joined array
    },
  },
]);
```

**Interview tip:** if asked to explain the aggregation pipeline conceptually, the cleanest framing is: *"It's a series of stages, each one transforming the output of the previous stage — similar to chaining `.filter().map().reduce()` in JavaScript, but executed inside the database engine for performance, instead of pulling raw data into your application first."*

---

## 7. Mongoose Middleware (Hooks) — a practical real-world pattern

```javascript
const bcrypt = require('bcrypt');

userSchema.pre('save', async function (next) {
  // 'this' refers to the document being saved. This hook runs
  // AUTOMATICALLY before every .save() call — useful for things that
  // should ALWAYS happen, without relying on every route handler to
  // remember to do it manually (reducing the chance of forgetting it
  // in some code path).
  if (this.isModified('password')) { // only re-hash if password actually changed
    this.password = await bcrypt.hash(this.password, 10);
  }
  next();
});

// Real benefit: ANY code path that creates/updates a user via .save()
// automatically gets password hashing, without that logic being
// duplicated across every route or service that touches users.
```

---

## 8. How this connects to real backend work (3 YOE framing)

- **"Our `/orders` endpoint got slow as the collection grew to millions of documents — what would you check first?"** → Whether the query's filter/sort fields have an appropriate index; use `.explain('executionStats')` to confirm whether it's doing a full collection scan.
- **"How would you calculate monthly revenue totals from an orders collection?"** → The aggregation pipeline: `$match` to filter the date range, `$group` by month, `$sum` the amounts.
- **"Why did invalid data get into the database even though your schema has validation rules?"** → Likely an update operation (`findByIdAndUpdate`, `updateOne`) that didn't pass `{ runValidators: true }`, since Mongoose doesn't apply schema validation to updates by default.
- **"When would you choose MongoDB over PostgreSQL for a new project?"** → Give the nuanced SQL-vs-NoSQL framing from section 1 — data shape, relationship complexity, and how often the schema is expected to evolve, not a blanket "NoSQL is more scalable" claim.

---

## 9. Hands-on practice for today

```javascript
// practice-day10.js
const mongoose = require('mongoose');
const { Schema } = mongoose;

mongoose.connect('mongodb://localhost:27017/practiceday10');

const productSchema = new Schema({
  name: { type: String, required: true },
  price: { type: Number, required: true, min: 0 },
  category: { type: String, index: true }, // shorthand way to add a single-field index
  inStock: { type: Boolean, default: true },
}, { timestamps: true });

const Product = mongoose.model('Product', productSchema);

async function run() {
  await Product.deleteMany({}); // clean slate for repeatable testing

  await Product.create([
    { name: 'Laptop', price: 1200, category: 'Electronics' },
    { name: 'Mouse', price: 25, category: 'Electronics' },
    { name: 'Desk', price: 300, category: 'Furniture' },
  ]);

  // Basic query practice
  const electronics = await Product.find({ category: 'Electronics' }).sort({ price: -1 });
  console.log('Electronics, expensive first:', electronics);

  // Aggregation practice: average price per category
  const avgByCategory = await Product.aggregate([
    { $group: { _id: '$category', avgPrice: { $avg: '$price' }, count: { $sum: 1 } } },
    { $sort: { avgPrice: -1 } },
  ]);
  console.log('Average price per category:', avgByCategory);

  await mongoose.disconnect();
}

run();
```

Run this against a local MongoDB instance (or a free MongoDB Atlas cluster if you don't have one installed locally) and confirm both the sorted query and the aggregation result make sense.

---

## 10. Interview Q&A Recap (say these out loud)

1. **Q: Is MongoDB schema-less? Does Mongoose change that?**
   A: MongoDB itself enforces no schema at the database engine level — different documents in the same collection can have different shapes. Mongoose adds an application-level schema with types, validation, and defaults, which is what most production Node apps rely on for consistency.

2. **Q: Why do indexes matter, and is it ever bad to add too many?**
   A: Without an index, MongoDB scans every document in a collection to find matches — slow at scale. Indexes let it jump directly to relevant documents. But indexes slow down writes (every insert/update must also update the index) and use extra storage, so they should be added deliberately on fields that are actually queried/sorted frequently, not everywhere.

3. **Q: Explain the aggregation pipeline in your own words.**
   A: A sequence of stages, each transforming the data and passing the result to the next stage — similar to chaining `.filter().map().reduce()` in JS, but executed inside the database for efficiency. Common stages: `$match` (filter), `$group` (aggregate), `$sort`, `$lookup` (join), `$project` (reshape output).

4. **Q: Does Mongoose validation apply automatically on update operations?**
   A: No — by default, methods like `findByIdAndUpdate` skip schema validation unless you explicitly pass `{ runValidators: true }`. This is a common source of unexpectedly invalid data slipping into production.

5. **Q: When would you choose MongoDB over a relational database?**
   A: When data is naturally document-shaped, doesn't require complex multi-entity joins, and the schema is likely to evolve over time. Relational databases remain a better fit for data with strong, stable relationships requiring complex queries and strict consistency guarantees.

6. **Q: What does `$lookup` do in MongoDB's aggregation pipeline?**
   A: It's MongoDB's equivalent of a SQL JOIN — it merges in related documents from another collection based on a matching field, though it's generally less performant than native SQL JOINs for heavily relational data.

---

## Tomorrow (Day 11) Preview
We cover **SQL databases with Sequelize/Prisma**: relations, joins, transactions, and migrations — giving you the relational-database counterpart to today's MongoDB knowledge, so you can speak confidently about both paradigms and explain tradeoffs between them in a system design interview.
