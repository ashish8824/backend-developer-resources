# Database Normalization & Isolation Levels

## PART 1 — DATABASE NORMALIZATION

## What is Normalization?

Normalization is the process of organizing a relational database schema to reduce data redundancy and improve data integrity. It involves decomposing large, messy tables into smaller, well-structured tables and defining relationships between them.

```
Unnormalized (messy):
| order_id | customer_name | customer_email | product1    | product2    | city    |
|----------|---------------|----------------|-------------|-------------|---------|
| 1        | Alice         | alice@x.com    | Phone       | Headphones  | Mumbai  |
| 2        | Alice         | alice@x.com    | Laptop      | NULL        | Mumbai  |
| 3        | Bob           | bob@x.com      | Tablet      | NULL        | Delhi   |

Problems:
- Alice's email stored twice — if she changes it, must update multiple rows
- product1, product2 columns — what if someone orders 5 products?
- Deleting order 3 loses Bob's info entirely
```

## Why Normalize?

### 1. Eliminate Redundancy

Same data stored in multiple places wastes storage and creates inconsistency risk.

### 2. Data Integrity

If a customer's email is stored in one place, updating it updates everywhere — no inconsistency.

### 3. Easier Updates

Change one row, not dozens.

### 4. Anomaly Prevention

- **Update Anomaly:** updating Alice's email in one row but missing another row
- **Insert Anomaly:** can't insert a new customer without also placing an order
- **Deletion Anomaly:** deleting the last order of a customer deletes the customer too

## Normal Forms

Each Normal Form (NF) builds on the previous. You must satisfy 1NF before 2NF, 2NF before 3NF, etc.

### 1NF — First Normal Form

**Rules:**
1. Each column contains atomic (indivisible) values — no arrays, lists, or comma-separated values
2. Each column contains values of a single type
3. Each row is unique (has a primary key)
4. No repeating column groups

**Violation:**

```
| order_id | products              |   <- not atomic (list in one cell)
|----------|-----------------------|
| 1        | Phone, Headphones     |

| order_id | product1 | product2 |  <- repeating groups
|----------|----------|----------|
| 1        | Phone    | Headphones |
```

**1NF Fix:**

```
| order_id | product    |
|----------|------------|
| 1        | Phone      |
| 1        | Headphones |
| 2        | Laptop     |
```

Each row has one atomic value per column, and (order_id, product) together form the primary key.

### 2NF — Second Normal Form

**Rules:**
1. Must be in 1NF
2. Every non-key attribute must depend on the WHOLE primary key, not just part of it (eliminates partial dependencies)

This only matters when the primary key is composite (multiple columns).

**Violation:**

```
Primary key: (order_id, product_id)

| order_id | product_id | product_name | customer_name | quantity |
|----------|------------|--------------|---------------|----------|
| 1        | 101        | Phone        | Alice         | 1        |
| 1        | 102        | Headphones   | Alice         | 2        |
| 2        | 101        | Phone        | Bob           | 1        |

product_name depends only on product_id (not on the full key)  <- partial dependency!
customer_name depends only on order_id (not on the full key)   <- partial dependency!
```

**2NF Fix:** split into separate tables

```
orders table:
| order_id | customer_name |
|----------|---------------|
| 1        | Alice         |
| 2        | Bob           |

products table:
| product_id | product_name |
|------------|--------------|
| 101        | Phone        |
| 102        | Headphones   |

order_items table:
| order_id | product_id | quantity |
|----------|------------|----------|
| 1        | 101        | 1        |
| 1        | 102        | 2        |
| 2        | 101        | 1        |
```

Now every non-key attribute depends on the full primary key of its table.

### 3NF — Third Normal Form

**Rules:**
1. Must be in 2NF
2. No transitive dependencies — non-key attribute must NOT depend on another non-key attribute

**Violation:**

```
| employee_id | department_id | department_name | manager_name |
|-------------|---------------|-----------------|--------------|
| 1           | 10            | Engineering     | Carol        |
| 2           | 10            | Engineering     | Carol        |
| 3           | 20            | Marketing       | Dave         |

employee_id -> department_id (direct)
department_id -> department_name (transitive! department_name depends on department_id, not employee_id)
department_id -> manager_name (transitive!)
```

**3NF Fix:**

```
employees table:
| employee_id | department_id |
|-------------|---------------|
| 1           | 10            |
| 2           | 10            |
| 3           | 20            |

departments table:
| department_id | department_name | manager_name |
|---------------|-----------------|--------------|
| 10            | Engineering     | Carol        |
| 20            | Marketing       | Dave         |
```

Now department_name and manager_name live in departments where they belong — changing a manager name updates one row.

### BCNF — Boyce-Codd Normal Form

**Rules:**
1. Must be in 3NF
2. For every functional dependency X → Y, X must be a superkey

BCNF is stronger than 3NF and handles rare edge cases with multiple overlapping candidate keys. For most practical applications, 3NF is sufficient.

### 4NF and 5NF

Deal with multi-valued dependencies and join dependencies — rarely discussed in backend interviews. Knowing up to BCNF is sufficient for most roles.

## Normal Form Quick Reference

| Normal Form | Eliminates |
|---|---|
| 1NF | Repeating groups, non-atomic values |
| 2NF | Partial dependencies on composite key |
| 3NF | Transitive dependencies (non-key → non-key) |
| BCNF | Remaining anomalies from overlapping candidate keys |

## Practical Example: E-Commerce Schema

**Unnormalized:**

```sql
CREATE TABLE orders_bad (
    order_id        INT,
    customer_name   VARCHAR(100),
    customer_email  VARCHAR(100),
    customer_city   VARCHAR(50),
    product_names   VARCHAR(500),  -- "Phone,Headphones" in one cell (not atomic)
    total           DECIMAL
);
```

**Normalized to 3NF:**

```sql
CREATE TABLE customers (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(100) NOT NULL,
    email   VARCHAR(100) UNIQUE NOT NULL,
    city    VARCHAR(50)
);

CREATE TABLE products (
    id      BIGSERIAL PRIMARY KEY,
    name    VARCHAR(100) NOT NULL,
    price   DECIMAL(10,2) NOT NULL,
    stock   INT NOT NULL DEFAULT 0
);

CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT REFERENCES customers(id),
    status      VARCHAR(20) DEFAULT 'PLACED',
    created_at  TIMESTAMP DEFAULT NOW(),
    total       DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id    BIGINT REFERENCES orders(id),
    product_id  BIGINT REFERENCES products(id),
    quantity    INT NOT NULL,
    unit_price  DECIMAL(10,2) NOT NULL,  -- price at time of order (can change later)
    PRIMARY KEY (order_id, product_id)
);
```

## Denormalization — When to Break the Rules

Sometimes you intentionally denormalize (add redundancy back) for performance:

```
Normalized: to get an order summary, JOIN orders + order_items + products + customers
  -> 4-table JOIN on millions of rows — expensive

Denormalized: orders table includes customer_name, customer_email directly
  -> One-table read for order summary — fast
```

### When to Denormalize

- **Read-heavy analytical queries** (OLAP) — reporting, dashboards
- **Cached/precomputed values** — store `order_item_count` on the order row instead of `COUNT(*)` every time
- **Hot-path performance** — a critical API endpoint that runs 10,000 times/sec
- **Event store / audit log** — denormalized snapshot of state at a point in time

### Denormalization Strategies

```sql
-- Store computed totals
ALTER TABLE orders ADD COLUMN item_count INT;  -- updated on insert/delete of order_items

-- Store frequently joined data
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);  -- copied from customers on order creation

-- Materialized view (DB manages the denormalized copy)
CREATE MATERIALIZED VIEW order_summary AS
SELECT o.id, o.status, c.name AS customer_name, c.email,
       COUNT(oi.product_id) AS item_count, o.total
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.status, c.name, c.email, o.total;

-- Refresh periodically
REFRESH MATERIALIZED VIEW order_summary;
```

---

## PART 2 — ISOLATION LEVELS (Deep Dive)

## Why Isolation Levels Exist

Full isolation (Serializable) means every transaction sees a perfectly consistent snapshot — but it requires locking rows and even ranges, severely limiting concurrent throughput. Lower isolation levels reduce locking overhead at the cost of allowing certain read anomalies.

## Read Phenomena (Recap)

```
DIRTY READ:
T1 writes row X = "new" (not yet committed)
T2 reads row X = "new"   <- reads uncommitted data
T1 rolls back -> "new" never existed
T2 made decisions on data that never actually committed

NON-REPEATABLE READ:
T1 reads row X = 100
T2 updates row X = 200 and commits
T1 reads row X = 200  <- different value in same transaction!

PHANTOM READ:
T1: SELECT COUNT(*) WHERE status='PENDING' -> 5
T2: INSERT a new PENDING row, commits
T1: SELECT COUNT(*) WHERE status='PENDING' -> 6  <- phantom row appeared!

LOST UPDATE:
T1 reads stock = 10, computes 10-1=9
T2 reads stock = 10, computes 10-1=9
T1 writes stock = 9
T2 writes stock = 9   <- T1's update is lost! Should be 8
```

## The Four Isolation Levels

### READ UNCOMMITTED

Transactions can read uncommitted (dirty) data from other transactions.

```sql
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;
-- May return a value that another transaction will roll back!
COMMIT;
```

- Allows: dirty reads, non-repeatable reads, phantom reads
- Performance: highest (almost no locking)
- Use case: approximate analytics where accuracy doesn't matter (e.g. rough row counts on huge tables)
- Practical use: very rare — almost always wrong choice

### READ COMMITTED (PostgreSQL default, Oracle default)

Only reads committed data. A new read within the same transaction may see newly committed changes.

```sql
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- sees $1000
-- Another transaction commits: balance = $800
SELECT balance FROM accounts WHERE id = 1;  -- now sees $800 (non-repeatable read!)
COMMIT;
```

- Prevents: dirty reads
- Allows: non-repeatable reads, phantom reads
- Performance: good — shared locks released after each read
- Use case: most web applications where a transaction's reads don't need to be 100% stable

### REPEATABLE READ (MySQL/InnoDB default)

Once a transaction reads a row, that row's value is frozen for the transaction's lifetime (uses MVCC snapshot). Other transactions' committed changes to that row are invisible.

```sql
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN;
SELECT balance FROM accounts WHERE id = 1;  -- sees $1000
-- Another transaction commits: balance = $800
SELECT balance FROM accounts WHERE id = 1;  -- still sees $1000 (stable read!)
COMMIT;
```

- Prevents: dirty reads, non-repeatable reads
- Allows: phantom reads (in theory — MySQL InnoDB prevents them with gap locks in practice)
- Performance: moderate — snapshot taken at transaction start
- Use case: financial summaries, reports that need consistent data within the transaction

### SERIALIZABLE

The strictest level. Transactions execute as if they were serial (one after another). Range locks or MVCC conflict detection prevent phantom reads.

```sql
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN;
SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- 5
-- If another transaction tries to INSERT a PENDING order here, it blocks (or fails)
-- until this transaction commits
SELECT COUNT(*) FROM orders WHERE status = 'PENDING';  -- still 5 (no phantoms)
COMMIT;
```

- Prevents: all phenomena (dirty reads, non-repeatable reads, phantom reads)
- Performance: lowest — highest lock contention, deadlock risk
- Use case: financial transactions, anything where any anomaly causes data corruption

## MVCC (Multi-Version Concurrency Control)

Most modern databases (PostgreSQL, MySQL InnoDB) use MVCC instead of read locks for higher concurrency.

```
Instead of locking a row for reading:
  Each row has multiple versions (with timestamps/transaction IDs)
  Readers see the version that was committed before their transaction started
  Writers create a new version

T1 starts at timestamp 100
T2 writes row X = "new" and commits at timestamp 110

T1 reads row X: sees the version from before timestamp 100 (old value)
T3 starts at timestamp 120: sees T2's version (timestamp 110 < 120)
```

Benefits:
- Readers don't block writers
- Writers don't block readers
- Much higher concurrency than pure locking

Cost: storing multiple row versions increases storage and requires periodic cleanup (VACUUM in PostgreSQL, purge threads in MySQL).

## Implementing Isolation in Java (JDBC + Spring)

### JDBC

```java
Connection conn = dataSource.getConnection();
conn.setAutoCommit(false);
conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);

try {
    // ... queries
    conn.commit();
} catch (Exception e) {
    conn.rollback();
} finally {
    conn.close();
}
```

### Spring @Transactional

```java
// Default isolation (inherits DB default — usually READ_COMMITTED)
@Transactional
public void processOrder(Long orderId) { ... }

// Explicit isolation level
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void generateReport(LocalDate from, LocalDate to) {
    // All reads within this method see a consistent snapshot
    List<Order> orders = orderRepo.findByDateRange(from, to);
    BigDecimal total = calculateTotal(orders);  // same orders, no phantoms
    reportRepo.save(new Report(total));
}

@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    // No anomalies at all — safest for financial operations
    Account from = accountRepo.findById(fromId).orElseThrow();
    Account to = accountRepo.findById(toId).orElseThrow();

    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }

    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));

    accountRepo.save(from);
    accountRepo.save(to);
}
```

## Isolation Level + Locking Strategies

### Shared Lock (Read Lock, S-Lock)

Multiple transactions can hold a shared lock on the same row simultaneously. A write lock blocks until all shared locks are released.

```sql
-- Explicit shared lock (PostgreSQL)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;
```

### Exclusive Lock (Write Lock, X-Lock)

Only one transaction can hold an exclusive lock. Blocks all other readers and writers.

```sql
-- Explicit exclusive lock — used for pessimistic locking
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;
-- Row is locked until transaction commits or rolls back
```

### Gap Lock (MySQL InnoDB, REPEATABLE READ)

Locks the gap between index values to prevent phantom inserts.

```sql
-- MySQL: SELECT with range creates a gap lock
SELECT * FROM orders WHERE id BETWEEN 10 AND 20 FOR UPDATE;
-- Locks the gap — no new rows with id 11-19 can be inserted by other transactions
```

### Lock Escalation

Row-level locks are most granular (least contention). If a transaction locks too many rows, the DB may escalate to a table lock — faster to manage but blocks everyone else.

## Choosing the Right Isolation Level

| Scenario | Recommended Level | Reason |
|---|---|---|
| Reading a user's profile | READ COMMITTED | Stale by 100ms is fine, high throughput needed |
| Generating a monthly report | REPEATABLE READ | Report must be internally consistent (same data throughout) |
| Bank transfer | SERIALIZABLE or REPEATABLE READ + pessimistic lock | Zero tolerance for lost updates |
| Flash sale inventory deduction | REPEATABLE READ + SELECT FOR UPDATE | Prevent overselling via pessimistic locking |
| Approximate analytics | READ UNCOMMITTED | Maximum throughput, accuracy not critical |
| Booking a seat (no double booking) | SERIALIZABLE or unique constraint + retry | Seat must be booked exactly once |

## Real World Isolation Level Defaults

| Database | Default Isolation Level |
|---|---|
| PostgreSQL | READ COMMITTED |
| MySQL (InnoDB) | REPEATABLE READ |
| Oracle | READ COMMITTED |
| SQL Server | READ COMMITTED |
| SQLite | SERIALIZABLE |

## Common Mistakes

### 1. Using SERIALIZABLE everywhere

```java
// Wrong — kills throughput unnecessarily
@Transactional(isolation = Isolation.SERIALIZABLE)
public List<Product> getAllProducts() {  // just a read, no anomalies possible!
    return productRepo.findAll();
}
```

Use the lowest isolation level that is correct for the operation.

### 2. Not knowing your DB's default

```java
// No isolation specified — uses DB default
// PostgreSQL: READ_COMMITTED
// MySQL: REPEATABLE_READ
@Transactional
public void process() { ... }
```

Know what your DB defaults to — it matters for whether you need to override it.

### 3. Forgetting that READ COMMITTED allows non-repeatable reads

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void calculateAndSaveTotal(Long orderId) {
    BigDecimal sum = orderItemRepo.sumByOrderId(orderId);  // $100
    // Another transaction discounts an item here, commits
    BigDecimal tax = taxService.calculate(orderId);  // based on $90 (different snapshot!)
    orderRepo.saveTotal(orderId, sum.add(tax));  // $100 + tax on $90 = inconsistent!
}
// Fix: use REPEATABLE READ for operations that read the same data multiple times
```

## Interview Questions

**Q1: What are the three read anomalies, and which isolation levels prevent each?**

Dirty Read (reading uncommitted data): prevented by READ COMMITTED and above. Non-Repeatable Read (reading same row twice gets different values): prevented by REPEATABLE READ and above. Phantom Read (re-running a range query returns different rows): prevented only by SERIALIZABLE (and MySQL's REPEATABLE READ with gap locks as a bonus).

**Q2: What is the difference between 2NF and 3NF?**

2NF eliminates partial dependencies — every non-key attribute must depend on the entire composite primary key, not just part of it. 3NF eliminates transitive dependencies — a non-key attribute must not depend on another non-key attribute (A → B → C where A is the key, B and C are non-key attributes is a 3NF violation because C should live in a separate table). 2NF is about the key; 3NF is about non-key attributes depending on each other.

**Q3: What is MVCC and how does it enable higher concurrency?**

Multi-Version Concurrency Control maintains multiple versions of each row, timestamped by transaction. Readers see a consistent snapshot from when their transaction started — without acquiring read locks. Writers create new versions rather than overwriting. Since readers don't lock rows, writers can proceed concurrently. This means readers never block writers and writers never block readers — much higher concurrency than traditional lock-based isolation.

**Q4: When would you denormalize a database and what are the risks?**

Denormalize when read performance is critical and join overhead is a measurable bottleneck — e.g. analytical queries, high-traffic read endpoints, reporting. Risks: data redundancy means the same data must be updated in multiple places (update anomalies), inconsistency if updates are missed, and increased storage. Mitigate with triggers, application-layer consistency enforcement, or materialized views.

**Q5: What is a phantom read and how does SERIALIZABLE prevent it?**

A phantom read occurs when a transaction re-executes a range query and finds different rows — because another transaction inserted or deleted rows matching the range in between. SERIALIZABLE prevents this with range/predicate locks (or optimistic conflict detection in MVCC systems like PostgreSQL's SSI) — if another transaction tries to insert a row that would affect an ongoing range query, it blocks or fails. MySQL's REPEATABLE READ also prevents phantoms via gap locks on InnoDB indexes.

**Q6: What is the difference between READ COMMITTED and REPEATABLE READ?**

READ COMMITTED: each statement within a transaction sees the latest committed data at the time of that statement — different statements in the same transaction may see different snapshots. REPEATABLE READ: the entire transaction sees a single consistent snapshot taken at transaction start — re-reading the same row always returns the same value regardless of commits by other transactions during the transaction. Use READ COMMITTED for simple operations; REPEATABLE READ when a transaction reads the same data multiple times and needs consistency across those reads.
