# Database Indexing

## What is a Database Index?

An index is a separate data structure that the database maintains alongside your table to make lookups faster. It works like the index at the back of a book — instead of scanning every page to find a topic, you jump directly to the right page number.

**Without an index:**

```sql
SELECT * FROM users WHERE email = 'john@example.com';
```

The DB reads every row in the `users` table from top to bottom until it finds a match.
This is called a **Full Table Scan** — O(N) where N is the number of rows.

**With an index on `email`:**

```sql
CREATE INDEX idx_users_email ON users(email);
```

The DB jumps directly to the matching row via the index structure.
This is O(log N) for a B-Tree index.

For a table with 10,000,000 rows:

```
Full scan:  10,000,000 row reads
B-Tree:     ~24 comparisons  (log₂ 10,000,000 ≈ 23.25)
```

## How Indexes Work Internally

### B-Tree Index (Default in MySQL, PostgreSQL)

A B-Tree (Balanced Tree) is the most common index structure. It stores values in sorted order, with each internal node pointing to child nodes, and leaf nodes holding the actual data pointers.

```
                    [50]
                   /    \
             [25]          [75]
            /    \        /    \
          [10]  [30]   [60]  [90]
```

- All leaf nodes are at the same depth (balanced)
- Values are sorted left-to-right
- Supports: equality (`=`), range (`>`, `<`, `BETWEEN`), prefix (`LIKE 'abc%'`), `ORDER BY`, `GROUP BY`

**Lookup:** traverse from root to leaf — O(log N)
**Insert/Update/Delete:** must update the B-Tree — O(log N) but adds write overhead

### Hash Index

Stores a hash of each indexed value, mapping it directly to the row location.

```
hash("john@example.com") -> page 42, slot 7
```

- Supports: equality only (`=`)
- Does NOT support: range queries, sorting, `LIKE`
- O(1) average for equality lookups
- Used internally by: PostgreSQL (explicit hash index), MySQL MEMORY engine, hash join operations

### Bitmap Index

Stores a bit vector per distinct value — each bit represents whether a row has that value.

```
status = 'ACTIVE':   1 0 1 1 0 1 0 0 1
status = 'INACTIVE': 0 1 0 0 1 0 1 1 0
```

Combining filters (`WHERE status = 'ACTIVE' AND gender = 'F'`) becomes a bitwise AND operation — extremely fast.

- Great for: low-cardinality columns (columns with few distinct values — status, gender, boolean flags)
- Bad for: high-cardinality columns (email, user_id) — one bitmap per distinct value becomes huge
- Used in: data warehouses, OLAP systems (Oracle, Redshift)

### Full-Text Index

Specialized index for searching within text. Tokenizes the text and builds an inverted index — mapping each word to the rows that contain it.

```
"quick brown fox" -> tokenize -> ["quick", "brown", "fox"]
"brown": [row 1, row 3, row 7]
```

Supports: `MATCH() AGAINST()` in MySQL, `tsvector` / `@@` in PostgreSQL, full-text search in Elasticsearch.
Does NOT replace a B-Tree for regular WHERE clauses.

## Types of Indexes (by structure/role)

### 1. Primary Index (Clustered Index)

The table data itself is physically stored in the order of the primary key. There is only one clustered index per table because the rows can only be physically sorted one way.

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,   -- clustered index
    user_id BIGINT,
    total DECIMAL
);
```

In InnoDB (MySQL), the primary key is always a clustered index. The leaf nodes of the B-Tree contain the actual row data, not just a pointer to it.

**Range queries on the primary key are very fast** because the data is physically contiguous on disk.

### 2. Secondary Index (Non-Clustered Index)

Any index that is not the clustered index. The leaf nodes store the indexed column value + a pointer back to the primary key (not the full row).

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

```
Lookup: WHERE user_id = 42
-> B-Tree on user_id -> finds primary key values [1001, 1005, 1009]
-> Second lookup in primary index for each PK to get full row
```

This second lookup is called a **double lookup** (or bookmark lookup / key lookup). It's fast for small result sets but expensive if many rows match.

### 3. Unique Index

Enforces uniqueness while also speeding up lookups. A `UNIQUE` constraint automatically creates a unique index.

```sql
CREATE UNIQUE INDEX idx_users_email ON users(email);
```

### 4. Composite Index (Multi-Column Index)

An index on more than one column.

```sql
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

**The Left-Prefix Rule (critical for interviews):**

A composite index on `(A, B, C)` can be used by queries that filter on:

```
A           ✅ — uses index
A, B        ✅ — uses index
A, B, C     ✅ — uses index
B           ❌ — cannot use this index (A is missing)
B, C        ❌ — cannot use this index
C           ❌ — cannot use this index
A, C        ⚠️  — uses index on A only, C is not filtered via the index
```

The index is only usable from the leftmost column forward, in order.

**Column order matters:** put the most selective column first (the one that narrows results the most), then range columns last.

```sql
-- Good: equality column first, range column last
CREATE INDEX idx ON orders(user_id, created_at);
SELECT * FROM orders WHERE user_id = 42 AND created_at > '2024-01-01';

-- Problem: range column first breaks the prefix rule for subsequent columns
CREATE INDEX idx ON orders(created_at, user_id);
-- This index cannot efficiently filter on user_id after a range on created_at
```

### 5. Covering Index

An index that contains all the columns a query needs, so the DB never has to go back to the main table at all.

```sql
-- Query
SELECT user_id, status, total FROM orders WHERE user_id = 42;

-- Covering index: includes all columns the query touches
CREATE INDEX idx_covering ON orders(user_id, status, total);
```

The entire query can be served from the index alone — no trip to the main table. This eliminates the double lookup for secondary indexes and is one of the most powerful performance optimizations available.

### 6. Partial Index

An index that only indexes rows matching a condition. Smaller and faster than a full-column index.

```sql
-- Only index active users — the common query pattern
CREATE INDEX idx_active_users ON users(email)
WHERE status = 'ACTIVE';
```

The index is used only when the query includes that condition:

```sql
SELECT * FROM users WHERE email = 'john@example.com' AND status = 'ACTIVE'; -- uses index
SELECT * FROM users WHERE email = 'john@example.com';                        -- does not use index
```

### 7. Expression / Functional Index

An index on the result of an expression or function — used when your query filters on a computed value.

```sql
-- Query
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

-- Without this, the above query can't use a plain index on email
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
```

## The Write Penalty

Indexes are not free. Every index you add must be maintained on every write:

```
INSERT: add entry to every index on the table
UPDATE: update every affected index on the table
DELETE: remove entry from every index on the table
```

A table with 10 indexes incurs 10 index maintenance operations on every write.

**Rules of thumb:**

- Read-heavy tables (e.g. product catalog, user profiles): index aggressively — reads vastly outnumber writes.
- Write-heavy tables (e.g. event logs, analytics ingestion, audit trail): index sparingly — each extra index slows every insert.
- Always benchmark before adding an index to a high-write table.

## Index Selectivity

Selectivity = the proportion of distinct values in a column.

```
High selectivity: email, user_id, order_id    (almost every value is unique)
Low selectivity:  status, gender, boolean flag (few distinct values)
```

Indexes are most valuable on **high-selectivity columns** because they narrow the result set dramatically. A B-Tree index on `gender` (2 distinct values) is almost useless — the DB may decide a full table scan is cheaper because the index points to half the rows anyway.

Low-cardinality columns are better served by **Bitmap Indexes** (in OLAP systems) or handled via **composite indexes** where high-selectivity columns come first.

## EXPLAIN / Query Plan

Before optimizing, use `EXPLAIN` (MySQL/PostgreSQL) to see how the DB is executing a query.

**MySQL:**

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42 AND status = 'PENDING';
```

Key output columns to look at:

| Column | What it tells you |
|---|---|
| `type` | Join/access type. `ALL` = full scan (bad), `ref` / `range` / `const` = index used (good) |
| `key` | Which index is being used |
| `rows` | Estimated number of rows examined |
| `Extra` | `Using index` = covering index (great); `Using filesort` = in-memory sort (can be expensive); `Using where` = filter applied after index fetch |

**PostgreSQL:**

```sql
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;
```

`EXPLAIN ANALYZE` actually runs the query and shows real execution times alongside estimates.

## Common Index Mistakes

### 1. Over-indexing

Adding an index on every column "just in case." Every index:
- Slows all writes
- Consumes disk space
- Must be maintained by the query planner (more indexes = harder planning decisions)

**Fix:** only add indexes driven by real query patterns.

### 2. Index not used — functions on columns

```sql
-- Index on created_at exists, but this query CANNOT use it
SELECT * FROM orders WHERE YEAR(created_at) = 2024;

-- This CAN use the index
SELECT * FROM orders WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
```

Wrapping an indexed column in a function destroys the ability to use the index.

### 3. Index not used — implicit type cast

```sql
-- user_id is VARCHAR, but you're filtering with an integer
SELECT * FROM users WHERE user_id = 12345;
-- MySQL implicitly casts, which may bypass the index
```

Always match the literal type to the column type.

### 4. Index not used — leading wildcard LIKE

```sql
SELECT * FROM products WHERE name LIKE '%phone%'; -- cannot use B-Tree index
SELECT * FROM products WHERE name LIKE 'phone%';  -- CAN use B-Tree index (prefix match)
```

A leading wildcard (`%word`) requires a full scan. Use a **Full-Text Index** for contains-style searches.

### 5. Low selectivity index

```sql
CREATE INDEX idx_orders_status ON orders(status);
-- status has 3 values: PENDING, COMPLETED, CANCELLED
-- Each status matches ~33% of rows
-- DB may ignore this index and do a full scan instead
```

### 6. Missing index on foreign key

Foreign key columns are frequently used in JOINs. Not indexing them causes full scans of the child table on every join.

```sql
-- orders.user_id references users.id — always index this
CREATE INDEX idx_orders_user_id ON orders(user_id);
```

## Index Strategy by Query Pattern

| Query Pattern | Recommended Index |
|---|---|
| Equality on one column | Single-column B-Tree index on that column |
| Equality + range | Composite index: equality column first, range column last |
| Query returns all columns | Covering index (include all selected columns) |
| Sorting | Index on ORDER BY column(s), in the same direction |
| String prefix match | B-Tree index (works for `LIKE 'prefix%'`) |
| Full-text search (contains) | Full-text index (MySQL FULLTEXT, Postgres `tsvector`) |
| Low-cardinality filter | Partial or composite index; avoid standalone B-Tree on that column |
| Filtered subset only | Partial index with WHERE condition |
| Computed/function on column | Expression/functional index |

## Java + JPA / Hibernate Annotations

```java
// Single-column index
@Entity
@Table(name = "users",
    indexes = @Index(name = "idx_users_email", columnList = "email"))
public class User {
    @Id
    private Long id;
    private String email;
}

// Composite index
@Entity
@Table(name = "orders",
    indexes = {
        @Index(name = "idx_orders_user_status", columnList = "user_id, status"),
        @Index(name = "idx_orders_created",     columnList = "created_at")
    })
public class Order {
    @Id
    private Long id;
    private Long userId;
    private String status;
    private LocalDateTime createdAt;
}

// Unique index
@Column(unique = true)
private String email;
```

These annotations cause Hibernate to emit the correct `CREATE INDEX` DDL when generating or validating the schema.

## Production Architecture Considerations

```
Application
  |
  | (query)
Database Primary (write + read)
  |
  +-- Replica 1 (read) -- same indexes replicated automatically
  +-- Replica 2 (read)
```

- Indexes are replicated automatically to read replicas — you don't need to maintain them separately.
- For very large tables, creating an index online (without locking the table) is critical in production: use `CREATE INDEX CONCURRENTLY` in PostgreSQL, or `ALTER TABLE ... ALGORITHM=INPLACE` in MySQL 5.6+.
- Monitor slow query logs regularly — a missing index often surfaces as a sudden spike in slow queries.

## Interview Questions

**Q1: What is a Full Table Scan and when does it happen?**

A full table scan reads every row in the table to find matching rows. It happens when: no usable index exists for the query's WHERE/JOIN/ORDER BY, the query wraps an indexed column in a function, the index has low selectivity and the optimizer decides scanning is cheaper, or the result set is so large that using the index would cause more I/O than just reading the table sequentially.

**Q2: Explain the Left-Prefix Rule for composite indexes.**

A composite index on `(A, B, C)` is only usable from the leftmost column forward, in order. A query filtering only on `B` or `C` cannot use this index because the index is sorted by `A` first — without knowing `A`, the B-Tree can't find a starting point. The optimizer can use `A` alone, `A+B`, or `A+B+C`, but not `B` alone or `C` alone.

**Q3: What is a Covering Index and why is it powerful?**

A covering index includes all columns referenced in a query (SELECT, WHERE, ORDER BY). This allows the query to be answered entirely from the index without touching the main table — eliminating the double lookup for secondary indexes and reducing I/O to a minimum.

**Q4: How does indexing affect write performance?**

Every INSERT, UPDATE, and DELETE must maintain all indexes on the table. A table with N indexes incurs N additional index update operations per write. On write-heavy tables, over-indexing can cause serious write throughput degradation — always profile before adding indexes.

**Q5: Why doesn't `WHERE YEAR(created_at) = 2024` use an index on `created_at`?**

Because the function `YEAR()` is applied to the column before comparison, the DB can't use the sorted B-Tree structure on `created_at` directly — it would have to evaluate `YEAR()` on every row. Rewrite as `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'` to use the index.

**Q6: When would you choose a Partial Index?**

When a large fraction of your queries only care about a subset of rows — e.g. `WHERE status = 'ACTIVE'` and 99% of users are inactive. A full index on `status` wastes space and time on the 99% you never query. A partial index on `(email) WHERE status = 'ACTIVE'` is tiny, fast, and perfectly targeted.

**Q7: How do you add an index to a large production table without downtime?**

In PostgreSQL: `CREATE INDEX CONCURRENTLY idx_name ON table(column)` — builds the index without holding a table lock, so reads and writes continue. It takes longer than a regular CREATE INDEX but doesn't block production traffic. In MySQL 5.6+ InnoDB: `ALTER TABLE t ADD INDEX ... ALGORITHM=INPLACE, LOCK=NONE` achieves a similar online build.
