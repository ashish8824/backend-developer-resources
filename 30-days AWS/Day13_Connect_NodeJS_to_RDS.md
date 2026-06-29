# Day 13: Connect Node.js App to RDS
### *(Wiring Your Express Backend to a Real Managed Database)*

> **Roadmap reference:** Week 2, Day 13 — "Connect Node.js App to RDS"

---

## Why This Matters

Yesterday you manually connected to RDS using `psql` — a terminal tool. Today you connect your **actual Express application** to that same RDS instance, which is the whole point of having a managed database. By the end of today, your Node.js API will be reading from and writing to a real PostgreSQL database running in AWS — not a local SQLite file, not an in-memory store.

This is the moment the full stack comes together: EC2 (compute) + RDS (database) + S3 (storage from Day 10) — all talking to each other.

---

## 1. What Changes Compared to Local Development?

```
LOCAL DEVELOPMENT                    RDS (AWS)
──────────────────                   ──────────
Host: localhost                  →   Host: <your-rds-endpoint>
Port: 5432                        →   Port: 5432 (same)
Database: local postgres           →   Database: appdb
SSL: usually off                   →   SSL: REQUIRED by default
User/Password: local config         →   Master credentials from Day 12
```

> **Only two real changes:** the hostname, and SSL mode. Everything else — your queries, your schema, your ORM — stays exactly the same. This is why managed databases like RDS are powerful — your code doesn't need to "know" it's talking to AWS.

---

## 2. Two Ways to Connect — Pick One

You have two main choices for the database layer in a Node.js app:

```
Option A: pg (node-postgres)
  → Raw PostgreSQL client, you write SQL directly
  → Lightweight, total control, no abstraction layer
  → Good when: you want to understand exactly what's happening

Option B: Prisma ORM
  → Schema-first ORM with auto-generated queries
  → Type-safe, migrations built-in, great DX
  → Good when: building a real app quickly, familiar from your
               PulseBloom/QueueCare work

TODAY: We'll implement BOTH so you understand each approach.
       In a real project, pick one and stay consistent.
```

---

## 3. Project Setup

```bash
# SSH into your EC2 instance
ssh -i my-first-server-key.pem ubuntu@<your-elastic-ip>

# Create a new project (or extend your Day 6/10 project)
mkdir node-rds-app && cd node-rds-app
npm init -y

# Install dependencies
npm install express dotenv pg
# OR for Prisma approach:
npm install express dotenv @prisma/client
npm install -D prisma
```

### Environment Variables

```bash
nano .env
```

```env
# Database connection
DB_HOST=my-first-rds-db.xxxxxxxxxx.ap-south-1.rds.amazonaws.com
DB_PORT=5432
DB_NAME=appdb
DB_USER=postgres
DB_PASSWORD=your-master-password

# App
PORT=5000
NODE_ENV=production
```

```bash
echo ".env" >> .gitignore
```

---

## 4. Approach A: Using `pg` (node-postgres) — Raw SQL

### db.js — Database Connection Pool

```javascript
// db.js
require('dotenv').config();
const { Pool } = require('pg');

const pool = new Pool({
  host:     process.env.DB_HOST,
  port:     parseInt(process.env.DB_PORT),
  database: process.env.DB_NAME,
  user:     process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  ssl: {
    rejectUnauthorized: false
    // For production with a verified certificate:
    // rejectUnauthorized: true,
    // ca: fs.readFileSync('rds-ca-cert.pem').toString()
  }
});

// Test the connection on startup
pool.connect((err, client, release) => {
  if (err) {
    console.error('❌ Database connection failed:', err.message);
  } else {
    console.log('✅ Connected to RDS PostgreSQL successfully');
    release();
  }
});

module.exports = pool;
```

```
Why a Pool, not a single Client?
  A Pool maintains multiple open connections simultaneously.
  Each incoming HTTP request grabs an available connection
  from the pool, uses it, and returns it — rather than
  opening/closing a new DB connection on every single request
  (which is extremely slow and exhausting for the database).

Pool size default: 10 connections
  → fine for most small-medium apps
```

### index.js — Express API with Database Routes

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const pool = require('./db');

const app = express();
app.use(express.json());
const PORT = process.env.PORT || 5000;

// ─── CREATE TABLE (run once to set up schema) ───────────────
app.post('/setup', async (req, res) => {
  try {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS users (
        id         SERIAL PRIMARY KEY,
        name       VARCHAR(100) NOT NULL,
        email      VARCHAR(100) UNIQUE NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
      )
    `);
    res.json({ message: 'Table created successfully' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ─── CREATE a user ──────────────────────────────────────────
app.post('/users', async (req, res) => {
  const { name, email } = req.body;
  try {
    const result = await pool.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [name, email]
    );
    res.status(201).json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ─── READ all users ─────────────────────────────────────────
app.get('/users', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT * FROM users ORDER BY created_at DESC'
    );
    res.json(result.rows);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ─── READ one user ──────────────────────────────────────────
app.get('/users/:id', async (req, res) => {
  try {
    const result = await pool.query(
      'SELECT * FROM users WHERE id = $1',
      [req.params.id]
    );
    if (result.rows.length === 0) {
      return res.status(404).json({ error: 'User not found' });
    }
    res.json(result.rows[0]);
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// ─── DELETE a user ──────────────────────────────────────────
app.delete('/users/:id', async (req, res) => {
  try {
    await pool.query('DELETE FROM users WHERE id = $1', [req.params.id]);
    res.json({ message: 'User deleted' });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## 5. Approach B: Using Prisma ORM

If you're building something larger or prefer type-safety (as you have with PulseBloom/QueueCare), Prisma is the cleaner choice.

### Initialize Prisma

```bash
npx prisma init
```

This creates a `prisma/` folder and a `.env` with a `DATABASE_URL` variable.

### Update `.env`

```env
DATABASE_URL="postgresql://postgres:your-password@your-rds-endpoint:5432/appdb?sslmode=require"
```

### Define Your Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  name      String
  email     String   @unique
  createdAt DateTime @default(now())
}
```

### Run Migration

```bash
npx prisma migrate dev --name init
```

```
What this does:
  → Generates SQL from your schema
  → Runs it against your RDS database
  → Creates a migrations history table in RDS so
    future schema changes are tracked
```

### Use in Express

```javascript
// index.js (Prisma version)
const { PrismaClient } = require('@prisma/client');
const prisma = new PrismaClient();

app.post('/users', async (req, res) => {
  const { name, email } = req.body;
  const user = await prisma.user.create({ data: { name, email } });
  res.status(201).json(user);
});

app.get('/users', async (req, res) => {
  const users = await prisma.user.findMany({
    orderBy: { createdAt: 'desc' }
  });
  res.json(users);
});
```

> Notice how much cleaner the Prisma version is — no raw SQL strings, fully type-safe, automatically generates queries. The tradeoff is that it adds a build/migration step your team must manage. For a real production project, this tradeoff is almost always worth it.

---

## 6. Testing the API End-to-End

```bash
# Start the app
pm2 start index.js --name node-rds-app

# Test from another terminal (or Postman)

# Create the table
curl -X POST http://localhost:5000/setup

# Create a user
curl -X POST http://localhost:5000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Ashish","email":"ashish@example.com"}'

# Get all users
curl http://localhost:5000/users

# Expected output:
[
  {
    "id": 1,
    "name": "Ashish",
    "email": "ashish@example.com",
    "created_at": "2026-06-22T10:00:00.000Z"
  }
]
```

---

## 7. Common Errors and Fixes

```
Error: "SSL SYSCALL error: EOF detected"
  Fix: Your SSL config is missing or wrong.
       Add ssl: { rejectUnauthorized: false } to the Pool config.
       Or add ?sslmode=require to your DATABASE_URL.

Error: "connect ETIMEDOUT" or "Connection timed out"
  Fix: RDS Security Group inbound rule is missing or wrong.
       Confirm rds-postgres-sg allows port 5432 FROM your EC2's SG.
       Also confirm your RDS instance is not in "stopped" state.

Error: "password authentication failed for user postgres"
  Fix: Wrong password in .env — copy it carefully, it's case-sensitive.

Error: "ENOTFOUND <endpoint>"
  Fix: The RDS endpoint hostname is wrong or mistyped in .env.
       Copy it exactly from RDS → your instance → Connectivity tab.

Error: "relation users does not exist"
  Fix: You haven't hit POST /setup yet, or your Prisma migration
       hasn't been run. The table doesn't exist in the database yet.
```

---

## 8. The Full Stack You've Now Built

```
         Client (browser / Postman / curl)
                        │
                        │ HTTP
                        ▼
              ┌────────────────────┐
              │  Nginx (port 80)     │   ← Day 7
              └────────────────────┘
                        │
                        ▼
              ┌────────────────────┐
              │  PM2 + Express       │   ← Day 7
              │   (port 5000)         │
              │  /users endpoints      │   ← Today
              │  /upload endpoint       │   ← Day 10
              └────────────────────┘
                   │          │
            TCP 5432     S3 SDK calls
                   │          │
                   ▼          ▼
              ┌──────┐   ┌──────────┐
              │  RDS  │   │    S3     │
              │  PG   │   │  Bucket  │
              └──────┘   └──────────┘
```

**This is a real, functioning, cloud-hosted full-stack backend.** EC2 serving HTTP, RDS persisting relational data, S3 storing files — the three core pieces of almost every Node.js backend in production.

---

## 9. Interview Questions (Practice Explaining These Out Loud)

**Q1: What changes in your Node.js database connection when moving from local PostgreSQL to RDS?**
> The hostname (from `localhost` to the RDS endpoint), and SSL must be explicitly enabled since RDS requires encrypted connections by default. Everything else — queries, schema, ORM — stays identical.

**Q2: Why use a connection pool instead of opening a new database connection per request?**
> Opening a TCP connection to a database is expensive — it takes time and consumes database resources. A pool maintains pre-opened connections and reuses them across requests, dramatically improving performance under concurrent traffic.

**Q3: What is `$1`, `$2` in a `pg` query, and why use it?**
> Parameterized query placeholders — values are passed separately from the SQL string, so the database driver handles escaping. This completely prevents SQL injection attacks, unlike string concatenation.

**Q4: What's the main difference between using raw `pg` vs Prisma?**
> `pg` gives full control with raw SQL — no abstraction overhead. Prisma provides a type-safe, schema-driven ORM with auto-generated queries and migrations — more developer-friendly but adds a build step and abstraction layer.

**Q5: Why should `DATABASE_URL` containing a password be in `.env` and not committed to Git?**
> Exposed database credentials give anyone direct access to your entire database — all user data, tables, everything. It's one of the most severe credential leaks possible and must be gitignored without exception.

---

## 10. Hands-On Assignment for Today

1. **Start your RDS instance** if it's stopped (Actions → Start).
2. Set up the project, install `pg` and `dotenv`, configure `.env` with your RDS credentials.
3. Implement and run the `db.js` + `index.js` from Section 4.
4. Hit `POST /setup` to create the table.
5. Test all 4 endpoints: `POST /users`, `GET /users`, `GET /users/:id`, `DELETE /users/:id`.
6. Verify the data actually persisted by connecting directly with `psql` (from Day 12) and running `SELECT * FROM users;` — confirm the same rows appear.
7. **Bonus:** Add a `PUT /users/:id` endpoint to update a user's name.
8. Write a short note covering:
   - Why SSL is required when connecting to RDS
   - What a connection pool does and why it matters
   - The full request flow from Postman all the way to RDS and back

---

## 11. Day 13 Goal — Can You Teach This?

By the end of today, you should be able to explain, without notes:

> *"Connecting Node.js to RDS is almost identical to connecting to a local PostgreSQL — only the hostname and SSL config change. I use a connection pool (via `pg` or Prisma) so the app reuses database connections efficiently across concurrent requests, never hardcode credentials in code, and use parameterized queries to prevent SQL injection. My full backend now has EC2 for compute, RDS for relational data, and S3 for file storage — each service handling exactly what it's best at."*

---

**Next up — Day 14: Backup and Snapshots (Week 2 Final Day before the Mini Project)**
