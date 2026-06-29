# Database Partitioning

## What is Database Partitioning?

Partitioning is the technique of dividing a large table into smaller, more manageable pieces (partitions) while keeping it logically as one table. Unlike sharding (which splits data across multiple servers), partitioning splits data within a single database server (or cluster).

```
Without Partitioning:
orders table: 10 billion rows
Query: SELECT * FROM orders WHERE created_at > '2024-01-01'
-> Full table scan: reads all 10 billion rows
-> Extremely slow

With Partitioning (by year):
orders_2022 partition: rows from 2022
orders_2023 partition: rows from 2023
orders_2024 partition: rows from 2024 <- query reads only this partition

Query: SELECT * FROM orders WHERE created_at > '2024-01-01'
-> Partition pruning: only reads orders_2024
-> 90% fewer rows scanned
```

## Partitioning vs Sharding

| | Partitioning | Sharding |
|---|---|---|
| Location | Within one DB server/cluster | Across multiple DB servers |
| Transparency | Fully transparent — app sees one table | Requires routing logic in app/middleware |
| Joins | Work normally across partitions | Complex across shards |
| Transactions | Normal ACID | Distributed transactions needed |
| Complexity | Low | High |
| Solves | Query performance, manageability | Write throughput, storage limits beyond one server |

**When to use:** partition first — it's much simpler. Only shard when partitioning + caching + replication isn't enough.

## Types of Partitioning

### 1. Range Partitioning

Rows are divided based on a range of values in a partitioning key (usually a date or numeric ID).

```sql
-- PostgreSQL range partitioning by date
CREATE TABLE orders (
    id          BIGSERIAL,
    user_id     BIGINT,
    total       DECIMAL(10,2),
    status      VARCHAR(20),
    created_at  TIMESTAMP NOT NULL
) PARTITION BY RANGE (created_at);

-- Create partitions for each year
CREATE TABLE orders_2022 PARTITION OF orders
    FOR VALUES FROM ('2022-01-01') TO ('2023-01-01');

CREATE TABLE orders_2023 PARTITION OF orders
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

-- Create future partition
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

**Query — partition pruning in action:**

```sql
-- Only reads orders_2024 partition
SELECT * FROM orders
WHERE created_at >= '2024-06-01' AND created_at < '2024-07-01';

-- EXPLAIN shows partition pruning
EXPLAIN SELECT * FROM orders WHERE created_at = '2024-06-15';
-- Output: Seq Scan on orders_2024 (not all partitions)
```

**Best for:** time-series data (logs, events, orders, transactions), where most queries filter by date range.

### 2. List Partitioning

Rows are divided based on a list of discrete values.

```sql
-- Partition orders by region
CREATE TABLE orders (
    id      BIGSERIAL,
    region  VARCHAR(20) NOT NULL,
    total   DECIMAL(10,2),
    created_at TIMESTAMP
) PARTITION BY LIST (region);

CREATE TABLE orders_india    PARTITION OF orders FOR VALUES IN ('INDIA');
CREATE TABLE orders_usa      PARTITION OF orders FOR VALUES IN ('USA');
CREATE TABLE orders_europe   PARTITION OF orders FOR VALUES IN ('UK', 'DE', 'FR');
CREATE TABLE orders_rest     PARTITION OF orders FOR VALUES IN ('OTHER');
```

**Best for:** categorically distinct data (region, country, product category, tenant_id in multi-tenant systems).

### 3. Hash Partitioning

Rows are divided by applying a hash function to the partitioning key and distributing modulo N partitions.

```sql
-- Partition users into 8 partitions by user_id hash
CREATE TABLE users (
    id      BIGSERIAL,
    email   VARCHAR(100),
    name    VARCHAR(100)
) PARTITION BY HASH (id);

CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 8, REMAINDER 2);
-- ... up to p7
```

**Best for:** even distribution without natural range/category — avoids hotspots when no date/region key makes sense. Queries without the partition key still scan all partitions (no pruning benefit).

### 4. Composite Partitioning (Sub-partitioning)

Combine two partitioning strategies — partition by range, then sub-partition each by hash or list.

```sql
-- Partition by year, then by region within each year
CREATE TABLE orders_2024 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    PARTITION BY LIST (region);

CREATE TABLE orders_2024_india  PARTITION OF orders_2024 FOR VALUES IN ('INDIA');
CREATE TABLE orders_2024_usa    PARTITION OF orders_2024 FOR VALUES IN ('USA');
```

Queries filtering on both `created_at` AND `region` benefit from double pruning.

## Partition Pruning

The query optimizer inspects the WHERE clause and determines which partitions could contain matching rows. Non-matching partitions are skipped entirely.

```sql
-- Pruning works:
SELECT * FROM orders WHERE created_at = '2024-06-15';       -- only orders_2024
SELECT * FROM orders WHERE region = 'INDIA';                 -- only orders_india
SELECT * FROM orders WHERE created_at = '2024-06-15'
                       AND region = 'INDIA';                 -- orders_2024_india only

-- Pruning does NOT work (function on partition key):
SELECT * FROM orders WHERE YEAR(created_at) = 2024;         -- full scan all partitions
-- Rewrite as:
SELECT * FROM orders WHERE created_at >= '2024-01-01'
                       AND created_at < '2025-01-01';       -- pruning works
```

## Indexes on Partitioned Tables

```sql
-- PostgreSQL: indexes created on parent are applied to all partitions
CREATE INDEX idx_orders_user_id ON orders(user_id);
-- Creates idx on orders_2022, orders_2023, orders_2024 automatically

-- Local index (per partition) vs Global index
-- PostgreSQL uses local indexes by default — more efficient for writes
-- Each partition's index is smaller and fits better in memory
```

## MySQL Partitioning

```sql
-- MySQL range partitioning
CREATE TABLE orders (
    id          BIGINT AUTO_INCREMENT,
    user_id     BIGINT,
    total       DECIMAL(10,2),
    created_at  DATETIME NOT NULL,
    PRIMARY KEY (id, created_at)  -- partition key must be part of PK in MySQL
) ENGINE=InnoDB
PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Check which partition a query uses
EXPLAIN SELECT * FROM orders WHERE created_at = '2024-06-15'\G
-- partitions: p2024
```

## Partition Maintenance

### Adding New Partitions

```sql
-- PostgreSQL: add new partition for 2025
CREATE TABLE orders_2025 PARTITION OF orders
    FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');
```

Use a scheduled job to create next year's partition in advance (before the current year ends).

### Archiving / Dropping Old Partitions

The biggest operational advantage of range partitioning: dropping old data is instant.

```sql
-- Instead of: DELETE FROM orders WHERE created_at < '2020-01-01' (slow on billions of rows!)
-- Just detach and drop the partition:
ALTER TABLE orders DETACH PARTITION orders_2019;  -- instant, no row scanning
DROP TABLE orders_2019;                            -- instant table drop

-- Or archive: detach and keep as standalone table
-- orders_2019 is now a regular table, can be moved to cold storage
```

### Partition Pruning Check (PostgreSQL)

```sql
-- Enable partition pruning (on by default in PostgreSQL 12+)
SET enable_partition_pruning = ON;

-- See which partitions are accessed
EXPLAIN (ANALYZE, VERBOSE) SELECT * FROM orders WHERE created_at = '2024-06-15';
```

## Partitioning in JPA / Spring Boot

JPA works transparently with partitioned tables — the application sees one logical table.

```java
@Entity
@Table(name = "orders")  // maps to partitioned parent table
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private Long userId;
    private BigDecimal total;

    @Column(nullable = false)
    private LocalDateTime createdAt;  // this is the partition key
}

// Spring Data JPA — works normally
public interface OrderRepository extends JpaRepository<Order, Long> {
    // PostgreSQL uses partition pruning automatically for these queries:
    List<Order> findByCreatedAtBetween(LocalDateTime from, LocalDateTime to);
    List<Order> findByUserIdAndCreatedAtAfter(Long userId, LocalDateTime after);
}
```

The partition pruning happens at the database level — JPA doesn't need to be partition-aware.

## Real-World Partitioning Examples

| Company/System | Table | Partition Key | Strategy |
|---|---|---|---|
| Banking | transactions | transaction_date | Range (monthly) |
| E-commerce | orders | created_at | Range (yearly/monthly) |
| Logging | audit_logs | log_date | Range (daily) |
| Multi-tenant SaaS | user_data | tenant_id | List |
| Analytics | events | event_timestamp | Range (daily) |
| Social media | posts | user_id % N | Hash |

## Partitioning Best Practices

### 1. Choose the Right Partition Key

The partition key should appear in most WHERE clauses. If 90% of queries filter by `created_at`, partition by `created_at`. If 90% filter by `tenant_id`, partition by `tenant_id`.

```sql
-- Wrong: partition by a column rarely used in WHERE clauses
-- -> partition pruning never triggers -> no benefit

-- Right: partition by the column in your most common filters
```

### 2. Partition Size

```
Too few partitions: partitions still large, limited pruning benefit
Too many partitions: planner overhead, index management complexity

Rule of thumb:
  Range partitioning: partition by month for tables with 1B+ monthly rows
                      partition by year for smaller volumes
  Hash partitioning: 8-32 partitions for even distribution
```

### 3. Avoid Functions on Partition Key in WHERE

```sql
-- Bad (no pruning):
WHERE MONTH(created_at) = 6 AND YEAR(created_at) = 2024

-- Good (pruning works):
WHERE created_at >= '2024-06-01' AND created_at < '2024-07-01'
```

### 4. Default Partition

```sql
-- Catch-all for values that don't fit any partition
-- (e.g. unexpected dates, unknown regions)
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

Without a default partition, rows that don't match any partition cause an error on INSERT.

## Interview Questions

**Q1: What is the difference between partitioning and sharding?**

Partitioning divides a large table into smaller pieces within the same database server — the application still sees one logical table, no routing logic needed, and normal joins and transactions work. Sharding distributes data across multiple separate database servers, requiring routing logic in the application or middleware, and making joins and transactions across shards complex. Partition first — only shard when a single server's limits are reached.

**Q2: What is partition pruning and why does it matter?**

Partition pruning is the query optimizer's ability to skip partitions that cannot contain rows matching the WHERE clause. If a table is partitioned by year and the query filters `WHERE created_at >= '2024-01-01'`, the optimizer reads only the 2024 partition instead of all partitions. This can reduce the data scanned by 90%+ on large tables, dramatically improving query performance.

**Q3: What is the biggest operational advantage of range partitioning for time-series data?**

Instant data deletion — instead of issuing `DELETE FROM orders WHERE created_at < '2020-01-01'` (which scans and deletes billions of rows, locking the table for hours), you `ALTER TABLE orders DETACH PARTITION orders_2019` and `DROP TABLE orders_2019`. Both operations are near-instantaneous because no rows are individually scanned — the entire partition's storage is deallocated at once.

**Q4: When would you use hash partitioning over range partitioning?**

When there's no natural sequential key (date, ID range) that matches query patterns, and you want even distribution to prevent hotspots. Hash partitioning distributes rows evenly across N partitions by hashing the partition key. Downside: queries without the partition key in WHERE must scan all partitions (no pruning possible).

**Q5: Why must the partition key usually appear in the WHERE clause of queries?**

Because partition pruning only works when the optimizer can determine from the WHERE clause which partitions might contain matching rows. If the WHERE clause doesn't reference the partition key at all, the optimizer must scan all partitions — negating the performance benefit of partitioning entirely.
