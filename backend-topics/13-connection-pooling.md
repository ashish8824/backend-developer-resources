# Connection Pooling

## What is Connection Pooling?

A connection pool is a cache of pre-opened database connections that are reused across requests, rather than opening and closing a new connection for every database operation.

**Without Connection Pooling:**

```
Request 1 arrives
  -> Open TCP connection to DB       (5-20ms)
  -> Authenticate                    (5-10ms)
  -> Execute query                   (1-5ms)
  -> Close connection
Total: ~25ms just for connection overhead

10,000 requests/sec = 10,000 new connections/sec to DB
DB max connections: 500 -> DB crashes
```

**With Connection Pooling:**

```
App starts -> Pool creates 10 connections upfront

Request 1 arrives
  -> Borrow connection from pool     (~0ms)
  -> Execute query                   (1-5ms)
  -> Return connection to pool
Total: ~5ms

10,000 requests/sec -> all share the same 10-50 connections
```

## Why is Opening a DB Connection Expensive?

A database connection involves:

1. **TCP handshake** — 3-way handshake between app server and DB server (round trip)
2. **TLS/SSL handshake** — if using encrypted connection (multiple round trips)
3. **Authentication** — username/password verification, permission loading
4. **Session initialization** — DB allocates memory for the session (buffers, process/thread)

On a local network this takes ~5–20ms. On a remote/cloud DB, 50–100ms is common. Multiply by thousands of requests per second and this is catastrophic.

## How Connection Pooling Works

```
Application Startup:
Pool creates N connections and keeps them open.

[Conn 1] [Conn 2] [Conn 3] ... [Conn N]
  IDLE     IDLE     IDLE         IDLE

Request arrives:
Pool picks an IDLE connection, marks it IN_USE, hands it to the request.

[Conn 1] [Conn 2] [Conn 3] ... [Conn N]
 IN_USE    IDLE     IDLE         IDLE

Request completes:
Connection is returned to pool (NOT closed), marked IDLE again.

[Conn 1] [Conn 2] [Conn 3] ... [Conn N]
  IDLE     IDLE     IDLE         IDLE
```

If all connections are in use and a new request arrives, the request waits in a queue for a connection to become available (up to `connectionTimeout`).

## Key Configuration Parameters

| Parameter | What it Controls | Typical Value |
|---|---|---|
| `minimumIdle` | Min idle connections kept open always | 5–10 |
| `maximumPoolSize` | Max total connections (idle + active) | 10–50 |
| `connectionTimeout` | Max time to wait for a connection from pool | 30,000ms |
| `idleTimeout` | Time before idle connections are closed | 600,000ms (10 min) |
| `maxLifetime` | Max lifetime of any connection (prevents stale connections) | 1,800,000ms (30 min) |
| `keepaliveTime` | How often to ping idle connections to keep them alive | 60,000ms |
| `validationTimeout` | Timeout for connection validation (alive test) | 5,000ms |

## HikariCP (Java — Industry Standard)

HikariCP is the default and fastest connection pool for Spring Boot. It's bundled with `spring-boot-starter-data-jpa` — no extra dependency needed.

### Configuration (application.yml)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: myuser
    password: mypassword
    driver-class-name: org.postgresql.Driver

    hikari:
      minimum-idle: 5
      maximum-pool-size: 20
      connection-timeout: 30000        # 30s — max wait for a connection
      idle-timeout: 600000             # 10min — close idle connections after
      max-lifetime: 1800000            # 30min — recycle connections
      keepalive-time: 60000            # 1min — ping idle connections
      pool-name: MyAppHikariPool
      connection-test-query: SELECT 1  # validation query (PostgreSQL doesn't need this)
```

### Programmatic Configuration

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        config.setUsername("myuser");
        config.setPassword("mypassword");

        config.setMinimumIdle(5);
        config.setMaximumPoolSize(20);
        config.setConnectionTimeout(30_000);
        config.setIdleTimeout(600_000);
        config.setMaxLifetime(1_800_000);

        // Critical for fast failure detection
        config.setConnectionInitSql("SELECT 1");

        return new HikariDataSource(config);
    }
}
```

### Monitoring HikariCP

```java
@Autowired
private DataSource dataSource;

public PoolStats getPoolStats() {
    HikariDataSource hikari = (HikariDataSource) dataSource;
    HikariPoolMXBean pool = hikari.getHikariPoolMXBean();

    return new PoolStats(
            pool.getTotalConnections(),    // total in pool
            pool.getActiveConnections(),   // currently in use
            pool.getIdleConnections(),     // available
            pool.getThreadsAwaitingConnection()  // waiting for a connection (bad if > 0)
    );
}
```

HikariCP also exposes metrics via Micrometer for Prometheus/Grafana — track `hikaricp_connections_active` and `hikaricp_connections_pending` in production.

## Node.js Connection Pooling (pg — PostgreSQL)

```javascript
const { Pool } = require('pg');

const pool = new Pool({
    host: 'localhost',
    port: 5432,
    database: 'mydb',
    user: 'myuser',
    password: 'mypassword',
    max: 20,                  // max pool size
    min: 5,                   // min idle connections
    idleTimeoutMillis: 30000, // close idle connections after 30s
    connectionTimeoutMillis: 2000,  // throw if can't get connection in 2s
    acquireTimeoutMillis: 30000
});

// Usage — pool.query handles acquire and release automatically
async function getUserById(id) {
    const result = await pool.query(
        'SELECT * FROM users WHERE id = $1',
        [id]
    );
    return result.rows[0];
}

// Manual acquire/release (for transactions)
async function transferFunds(fromId, toId, amount) {
    const client = await pool.connect();  // acquire connection
    try {
        await client.query('BEGIN');
        await client.query(
            'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
            [amount, fromId]
        );
        await client.query(
            'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
            [amount, toId]
        );
        await client.query('COMMIT');
    } catch (err) {
        await client.query('ROLLBACK');
        throw err;
    } finally {
        client.release();  // always return to pool
    }
}

// Monitor pool
pool.on('connect', () => console.log('New client connected to pool'));
pool.on('error', (err) => console.error('Pool client error:', err));
```

## How Many Connections Should the Pool Have?

### The Formula (from HikariCP documentation)

```
pool_size = Tn * (Cm - 1) + 1

Tn = number of threads (concurrent requests) that need DB access
Cm = ceil(response_time_ms / query_time_ms)  -- how many queries per request
```

But in practice, the widely-used rule is:

```
pool_size = (number of CPU cores on DB server * 2) + number of effective spindle disks
```

For an 8-core DB server with SSD:

```
pool_size = 8 * 2 + 1 = 17 -> ~20 is reasonable
```

### Why NOT 500 connections?

Counter-intuitively, a large pool can hurt performance:

```
DB Server: 8 cores
500 active connections -> 500 threads/processes competing for 8 cores
Context switching overhead ↑
Memory per connection (5–10MB for PostgreSQL) -> 500 * 10MB = 5GB just for connection state
Query throughput actually DECREASES
```

This is the "too many connections" problem. Each DB connection consumes real server resources.

**PostgreSQL max_connections** — by default 100. For production, configure `max_connections` carefully and use PgBouncer (a connection pooler that sits in front of PostgreSQL) for very high-concurrency scenarios.

## PgBouncer — External Connection Pooler for PostgreSQL

PostgreSQL creates a process per connection — very resource-intensive. PgBouncer is a lightweight proxy that pools connections at the DB level:

```
App Servers (200 connections to PgBouncer)
      |
  PgBouncer
      |
PostgreSQL (20 actual connections)
```

### PgBouncer Modes

| Mode | Behavior | Use Case |
|---|---|---|
| Session mode | One server connection per client session (held for session lifetime) | Legacy apps |
| Transaction mode | Server connection held only during a transaction, returned to pool after | Most apps (recommended) |
| Statement mode | Server connection returned after each statement | Rare (breaks transactions) |

**Transaction mode** is the most efficient — app servers can have hundreds of connections to PgBouncer, but PgBouncer only needs a small pool of real PostgreSQL connections.

## Connection Leak Detection

A connection leak occurs when code acquires a connection from the pool but never returns it (fails to call `close()` / `release()`). Over time, the pool is exhausted.

### HikariCP Leak Detection

```yaml
spring:
  datasource:
    hikari:
      leak-detection-threshold: 5000  # warn if connection not returned within 5s
```

HikariCP logs a warning with a stack trace showing where the connection was acquired — easy to pinpoint leaks.

### Common Cause of Leaks

```java
// WRONG — connection never released if exception occurs before close()
public void badExample() {
    Connection conn = dataSource.getConnection();
    PreparedStatement stmt = conn.prepareStatement("SELECT 1");
    ResultSet rs = stmt.executeQuery();
    // exception here -> conn never closed -> LEAK
    conn.close();
}

// CORRECT — try-with-resources ensures close() even on exception
public void goodExample() throws SQLException {
    try (Connection conn = dataSource.getConnection();
         PreparedStatement stmt = conn.prepareStatement("SELECT 1");
         ResultSet rs = stmt.executeQuery()) {
        // connection auto-closed when block exits, even on exception
    }
}
```

Always use try-with-resources (Java) or `finally { client.release() }` (Node.js) for connection management.

## Connection Validation

Connections in the pool can become stale — the DB server closes them due to idle timeout, network interruption, or DB restart. If a stale connection is handed to a request, the query fails.

**Strategies:**

1. **Test on borrow** — validate connection before handing to caller (adds small overhead per request)
2. **Test on return** — validate when returned to pool
3. **Background validation** — periodically ping idle connections (`keepaliveTime` in HikariCP)
4. **maxLifetime** — recycle every connection after a max age, preventing long-lived stale connections

HikariCP uses a combination of `maxLifetime` and `keepaliveTime` by default — no explicit `testOnBorrow` needed.

## Connection Pool Exhaustion

When all connections are in use and a new request waits longer than `connectionTimeout`, the request fails with an exception:

```
java.sql.SQLTransientConnectionException: HikariPool - Connection is not available,
request timed out after 30000ms
```

**Causes:**
- Pool size too small for load
- Slow queries holding connections too long
- Connection leaks
- DB server overwhelmed (queries slow down, connections held longer)

**Diagnosis:**

```java
// Check if threads are waiting for connections
pool.getThreadsAwaitingConnection() > 0  // alarm condition

// Check active vs pool size
pool.getActiveConnections() == pool.getMaximumPoolSize()  // pool exhausted
```

**Fixes:**
- Increase pool size (carefully — check DB limits first)
- Optimize slow queries (add indexes, reduce N+1 queries)
- Fix connection leaks
- Add read replicas and route read queries there
- Cache frequently-read data to reduce DB calls entirely

## N+1 Query Problem (Closely Related)

A common cause of pool pressure — loading a list of entities and then making one query per entity instead of a single JOIN:

```java
// N+1 PROBLEM — 1 query for orders + N queries for each order's user
List<Order> orders = orderRepo.findAll();  // 1 query -> 100 orders
for (Order order : orders) {
    User user = userRepo.findById(order.getUserId());  // N=100 queries!
}
// Total: 101 DB queries, 101 connection borrows

// FIX — single JOIN query
List<OrderWithUser> results = orderRepo.findAllWithUser();  // 1 query
// Total: 1 DB query, 1 connection borrow
```

In JPA/Hibernate, use `@EntityGraph` or `JOIN FETCH` to load associations eagerly in a single query.

## Production Architecture

```
App Server 1 [HikariCP pool: 20 connections]
App Server 2 [HikariCP pool: 20 connections]
App Server 3 [HikariCP pool: 20 connections]
      |
  PgBouncer (pools 60 app connections -> 15 actual PG connections)
      |
PostgreSQL Primary (max_connections = 100)
      |
PostgreSQL Replica (for read queries)
```

## Interview Questions

**Q1: Why is opening a new DB connection per request expensive?**

A connection requires a TCP handshake, optional TLS negotiation, DB authentication, and server-side session memory allocation — easily 10–50ms per connection. At thousands of requests per second, this overhead is catastrophic and the DB runs out of connection slots. A pool opens connections once at startup and reuses them.

**Q2: What happens when the connection pool is exhausted?**

New requests wait in a queue for a connection to become free, up to `connectionTimeout`. If no connection becomes available within that time, the request fails with a timeout exception. Fix: increase pool size (within DB limits), optimize slow queries, fix connection leaks, or add read replicas.

**Q3: How do you size a connection pool?**

A common rule: `(DB_server_cores * 2) + effective_disk_spindles`. For an 8-core DB with SSDs: ~20 connections. More connections than cores means cores spend time context-switching instead of executing queries — throughput degrades. Test empirically with realistic load; the right size depends on your query mix.

**Q4: What is a connection leak and how do you detect it?**

A leak occurs when a connection is borrowed from the pool but never returned — usually because `close()` or `release()` is not called (especially when an exception is thrown). Use `try-with-resources` in Java and `finally { client.release() }` in Node.js. Enable HikariCP's `leak-detection-threshold` to log a warning with a stack trace when a connection isn't returned within the configured time.

**Q5: What is PgBouncer and why would you use it?**

PgBouncer is a connection pooler that sits in front of PostgreSQL. PostgreSQL creates a process per connection — very expensive at hundreds of connections. PgBouncer proxies hundreds of app connections down to a small number of real PostgreSQL connections (in transaction mode: connection is held only during a transaction, returned after). It dramatically reduces PostgreSQL memory and process overhead at high concurrency.

**Q6: What is the N+1 query problem and how does it relate to connection pooling?**

The N+1 problem is loading a list of N items and then making N separate queries for associated data (e.g. one query per order's user). This generates N+1 total queries, each borrowing a connection, dramatically increasing pool usage and DB load. Fix with JOIN queries (one connection borrow, one query). N+1 is one of the most common causes of unexpected connection pool exhaustion and slow API responses.
