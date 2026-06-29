# Database Replication

## What is Database Replication?

Database replication is the process of automatically copying and synchronizing data from one database server (primary) to one or more other servers (replicas/secondaries). Replicas hold a copy of the same data and can serve read queries.

```
Without Replication:
All reads AND writes -> Single DB server
-> Server becomes bottleneck
-> Single point of failure

With Replication:
All writes -> Primary (source of truth)
All reads  -> Replica 1, Replica 2, Replica 3
-> Reads scaled out
-> Primary failure -> promote a replica
```

## Why Replication?

### 1. Read Scalability

Most applications are read-heavy (read:write = 80:20 or 90:10). Adding replicas distributes read load.

```
10,000 reads/sec, 1,000 writes/sec
Without replicas: single DB handles everything
With 3 replicas: each replica handles ~3,333 reads/sec, primary handles only writes
```

### 2. High Availability

If the primary fails, a replica is promoted to primary. Downtime is seconds to minutes instead of hours.

```
Primary dies at 2 AM
Replica 1 automatically promoted to primary
App reconnects to new primary
Total downtime: ~30 seconds
```

### 3. Disaster Recovery

Replicas in different data centers protect against catastrophic failures (fire, flood, power outage).

### 4. Analytical Queries (OLAP Offloading)

Heavy analytical queries (full table scans, aggregations) can run on a dedicated replica without affecting the production primary.

```
Reporting service -> Read Replica (dedicated for analytics)
User-facing API   -> Primary + Standard Replicas
Heavy analytics never touches the production primary
```

### 5. Geographic Distribution

Replicas in different regions serve local reads with lower latency.

```
Europe users -> European replica (10ms)
Asia users   -> Asian replica (10ms)
vs. all users hitting US primary (200ms)
```

## Replication Types

### 1. Primary-Replica (Master-Slave) — Most Common

One primary (handles writes), one or more replicas (serve reads, asynchronously replicated from primary).

```
Client writes -> Primary -> WAL/binlog
                         -> Replica 1 (applies changes)
                         -> Replica 2 (applies changes)
                         -> Replica 3 (applies changes)

Client reads  -> Any Replica
```

### 2. Primary-Primary (Multi-Master / Active-Active)

Multiple nodes accept writes. Changes replicate bidirectionally.

```
Write -> Primary A -> replicates to -> Primary B
Write -> Primary B -> replicates to -> Primary A
```

Pros: write scalability, no single point of failure for writes.
Cons: conflict resolution when the same row is modified on both primaries simultaneously — complex to handle correctly. Rare in practice; usually used in geographically distributed setups with careful conflict avoidance.

### 3. Synchronous vs Asynchronous Replication

**Asynchronous (Default):**

```
Client -> Primary (writes, commits, returns OK to client)
       -> Replica (receives and applies changes shortly after, with some delay)
```

Primary doesn't wait for replicas to confirm. Fast — primary isn't blocked by network latency to replicas. Risk: if primary crashes between commit and replication, replica may lose some data (replication lag = potential data loss).

**Synchronous:**

```
Client -> Primary (writes, waits...)
       -> Replica (confirms receipt)
       -> Primary (commits, returns OK to client)
```

Primary waits for at least one replica to confirm before returning success. No data loss on primary failure. Cost: every write is as slow as the network round trip to the replica.

**Semi-Synchronous (MySQL default in HA setups):**

Primary waits for at least one replica to confirm receipt (not necessarily apply). Balances data safety and performance.

## Replication Lag

Replication lag is the delay between a write being committed on the primary and that change being visible on a replica.

```
Primary: INSERT INTO orders (id, status) VALUES (123, 'PLACED')  at T=0
Replica: Sees this change at T=50ms (replication lag = 50ms)

If user reads immediately after write:
App reads from replica at T=10ms -> order doesn't exist yet!
User sees: "Order not found" immediately after placing it
```

Replication lag is normally milliseconds but can spike to seconds or minutes under:
- High write load on primary
- Network congestion
- Slow replica (disk, CPU)
- Large transactions (a single 10GB transaction must fully apply before the replica processes the next one)

### Handling Replication Lag in Application Code

**1. Read-after-write consistency — Read your own writes:**

After a write, route the user's subsequent reads to the primary (or wait for replica to catch up).

```java
// After placing an order, read the order from primary
public Order placeOrder(OrderRequest req) {
    Order order = orderRepo.save(new Order(req));  // write to primary

    // Read back from PRIMARY (not replica) to avoid replication lag
    return primaryOrderRepo.findById(order.getId()).orElseThrow();
}
```

**2. Route critical reads to primary:**

```java
@Transactional(readOnly = false)  // goes to primary
public Order getOrderAfterPayment(Long orderId) {
    return orderRepo.findById(orderId).orElseThrow();
}
```

**3. Sticky routing — route a user to the same replica for a session:**

If user reads from Replica 1, keep routing them there — same replica has consistent view.

**4. Monotonic reads:**

Track the last-seen replication position and only read from a replica that has caught up to at least that point.

## Routing Reads and Writes

### Spring Boot with Multiple Data Sources

```java
public enum DataSourceType {
    PRIMARY, REPLICA
}

@Component
public class DataSourceContextHolder {
    private static final ThreadLocal<DataSourceType> context = new ThreadLocal<>();

    public static void setDataSourceType(DataSourceType type) {
        context.set(type);
    }

    public static DataSourceType getDataSourceType() {
        return context.getOrDefault(DataSourceType.PRIMARY);
    }

    public static void clear() {
        context.remove();
    }
}

@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("primaryDataSource") DataSource primary,
            @Qualifier("replicaDataSource") DataSource replica) {

        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return DataSourceContextHolder.getDataSourceType();
            }
        };

        Map<Object, Object> sources = new HashMap<>();
        sources.put(DataSourceType.PRIMARY, primary);
        sources.put(DataSourceType.REPLICA, replica);
        routing.setTargetDataSources(sources);
        routing.setDefaultTargetDataSource(primary);
        return routing;
    }
}

// Annotation-based routing
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface ReadOnly {}

@Aspect
@Component
public class ReadOnlyDataSourceAspect {

    @Around("@annotation(ReadOnly)")
    public Object routeToReplica(ProceedingJoinPoint joinPoint) throws Throwable {
        DataSourceContextHolder.setDataSourceType(DataSourceType.REPLICA);
        try {
            return joinPoint.proceed();
        } finally {
            DataSourceContextHolder.clear();
        }
    }
}

// Usage
@Service
public class OrderService {

    @ReadOnly   // -> goes to replica
    public List<Order> getAllOrders() {
        return orderRepo.findAll();
    }

    // No annotation -> goes to primary
    public Order createOrder(OrderRequest req) {
        return orderRepo.save(new Order(req));
    }
}
```

### Spring `@Transactional(readOnly = true)`

Spring Boot (with certain JPA/connection pool setups) can automatically route `readOnly = true` transactions to replicas:

```java
@Transactional(readOnly = true)  // hint: can route to replica
public List<Order> getOrders() { ... }

@Transactional  // write -> primary
public Order createOrder(OrderRequest req) { ... }
```

This requires configuring a routing `DataSource` as shown above.

## Node.js — Multiple DB Connections

```javascript
const { Pool } = require('pg');

const primaryPool = new Pool({ host: 'primary-db', ... });
const replicaPool = new Pool({ host: 'replica-db-1', ... });

// Route by operation type
async function query(sql, params, write = false) {
    const pool = write ? primaryPool : replicaPool;
    return pool.query(sql, params);
}

// Usage
async function getOrders() {
    return query('SELECT * FROM orders', [], false);  // replica
}

async function createOrder(order) {
    return query(
        'INSERT INTO orders (user_id, total) VALUES ($1, $2) RETURNING *',
        [order.userId, order.total],
        true  // primary
    );
}
```

## PostgreSQL Replication Setup (Conceptual)

```
-- Primary (postgresql.conf)
wal_level = replica
max_wal_senders = 5
wal_keep_size = 1GB

-- Primary (pg_hba.conf) — allow replica to connect
host replication replicator replica_ip/32 md5

-- Replica setup
pg_basebackup -h primary_ip -U replicator -D /var/lib/postgresql/data -P -Xs -R
# -R creates standby.signal and recovery config automatically

-- Check replication status on primary
SELECT client_addr, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       (sent_lsn - replay_lsn) AS replication_lag
FROM pg_stat_replication;
```

## MySQL Replication Setup (Conceptual)

```sql
-- Primary (my.cnf)
server-id = 1
log-bin = mysql-bin
binlog_format = ROW    -- most reliable; records actual row changes

-- Replica (my.cnf)
server-id = 2

-- Replica SQL
CHANGE MASTER TO
  MASTER_HOST='primary_ip',
  MASTER_USER='replicator',
  MASTER_PASSWORD='password',
  MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=154;
START SLAVE;

-- Check replication status
SHOW SLAVE STATUS\G
-- Check: Seconds_Behind_Master (replication lag)
--        Slave_IO_Running: Yes
--        Slave_SQL_Running: Yes
```

## Failover

### Manual Failover

```
1. Primary dies
2. DBA promotes best replica: pg_ctl promote / STOP SLAVE on replica
3. Update app config / DNS to point to new primary
4. Other replicas reconfigure to replicate from new primary
5. Old primary (if recoverable) rejoins as a replica
```

### Automatic Failover

Tools:

- **PostgreSQL:** Patroni (etcd/ZooKeeper-backed), Repmgr
- **MySQL:** MHA (Master High Availability), Orchestrator, MySQL Group Replication
- **Cloud:** AWS RDS Multi-AZ (automatic failover in ~30-60 seconds), Google Cloud SQL HA

```
Patroni Architecture:
Primary + 2 Replicas + etcd (distributed consensus store)
  |
  If primary fails:
  etcd detects failure
  Patroni elects new primary (the replica most caught up)
  HAProxy/PgBouncer updated to point to new primary
  Total failover time: ~15-30 seconds
```

## Replication vs Sharding

| | Replication | Sharding |
|---|---|---|
| What it does | Copies the same data to multiple nodes | Splits different data across nodes |
| Solves | Read scalability, HA, DR | Write scalability, storage limits |
| All nodes have | All the data (full copy) | A subset of the data |
| Write bottleneck | Remains (all writes go to primary) | Eliminated (writes distributed) |
| Complexity | Low-Medium | High |
| When to use | First step after single-DB limits | After replication + caching aren't enough |

## Interview Questions

**Q1: What is the difference between synchronous and asynchronous replication?**

Asynchronous: primary commits and responds to the client before waiting for replicas to confirm. Fast, but if the primary crashes before replication completes, some writes may be lost. Synchronous: primary waits for at least one replica to confirm receipt before committing. No data loss on failover, but every write is limited by network latency to the replica.

**Q2: What is replication lag and how do you handle it?**

Replication lag is the delay between a write on the primary and that change being visible on a replica. Applications that read from replicas immediately after writing may get stale data. Handle it by: reading from the primary immediately after writes (read-your-writes), routing critical reads to the primary, using sticky session routing to the same replica, or implementing monotonic read guarantees.

**Q3: What is the difference between replication and sharding?**

Replication copies the full dataset to multiple nodes — solves read scalability and high availability but doesn't help with write throughput or storage limits (every node stores everything). Sharding splits the dataset across nodes — each node holds a different subset, solving write throughput and storage, but doesn't directly help with read scalability per shard. Production systems use both: each shard has its own replicas.

**Q4: What happens during a failover?**

When the primary fails, a replica is promoted to become the new primary. In manual failover, a DBA triggers promotion and updates connection configs. In automatic failover (Patroni, AWS RDS Multi-AZ), a cluster management tool detects failure via health checks, elects the most up-to-date replica as primary, and updates the service endpoint — typically taking 15–60 seconds.

**Q5: How do you route read and write traffic to different DB nodes in Spring Boot?**

Implement an `AbstractRoutingDataSource` that checks a `ThreadLocal` context to select either the primary or replica `DataSource`. Use an AOP aspect to set the context based on `@Transactional(readOnly=true)` or a custom `@ReadOnly` annotation. Write operations go to primary; read operations route to a replica automatically.
