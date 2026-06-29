# Distributed Lock (Redis)

## What is a Distributed Lock?

A distributed lock is a mechanism to ensure that only one process (across multiple servers) can access a shared resource at a time. In a single-server application, you use a thread lock (`synchronized`, `ReentrantLock`). In a distributed system with multiple servers, those locks are useless — you need a lock that all servers agree on.

```
Without distributed lock:
Server 1: reads stock = 1, decides to sell
Server 2: reads stock = 1, decides to sell
Server 1: sets stock = 0 (sold)
Server 2: sets stock = 0 (sold — oversell! stock went negative)

With distributed lock:
Server 1: acquires lock -> reads stock = 1 -> sells -> sets stock = 0 -> releases lock
Server 2: tries to acquire lock -> BLOCKED until Server 1 releases
Server 2: acquires lock -> reads stock = 0 -> rejects sale -> releases lock
```

## Why Redis for Distributed Locks?

- In-memory: sub-millisecond operations
- Atomic commands: `SET NX PX` is a single atomic operation — no race conditions
- TTL support: lock auto-expires if the holder crashes (prevents deadlock)
- Widely available: most teams already run Redis for caching

## Basic Lock Mechanics

### The Core Redis Command

```
SET lock_key unique_value NX PX 30000
```

- `SET lock_key unique_value` — set the key
- `NX` — only set if the key does NOT already exist (atomic acquire)
- `PX 30000` — expire after 30,000 ms (30 seconds) — safety TTL

Returns `OK` if lock acquired, `nil` if already held.

### Why a Unique Value?

The value must be unique per lock holder (e.g. UUID) so that only the holder can release its own lock.

```
Server 1 acquires lock, value = "uuid-abc"
Server 1 processes (takes 25 seconds — near TTL)
Lock expires (TTL = 30s)
Server 2 acquires the same lock, value = "uuid-xyz"
Server 1 finishes, tries to release lock: DEL lock_key
  -> Without value check: Server 1 deletes Server 2's lock! (wrong)
  -> With value check: Server 1 checks value == "uuid-abc" -> doesn't match -> does NOT delete
```

### Atomic Release with Lua Script

The check-and-delete must be atomic — otherwise there's a race between the check and the delete:

```lua
-- release_lock.lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
else
    return 0
end
```

This runs atomically in Redis — no other command can interleave between the GET and the DEL.

## Java Implementation (Jedis)

```java
import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;

public class RedisDistributedLock {

    private final Jedis jedis;
    private final String lockKey;
    private final String lockValue;
    private final int ttlMs;

    private static final String RELEASE_SCRIPT =
            "if redis.call('GET', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('DEL', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";

    public RedisDistributedLock(Jedis jedis, String lockKey, int ttlMs) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.lockValue = UUID.randomUUID().toString();
        this.ttlMs = ttlMs;
    }

    public boolean tryAcquire() {
        String result = jedis.set(
                lockKey,
                lockValue,
                SetParams.setParams().nx().px(ttlMs)
        );
        return "OK".equals(result);
    }

    public boolean tryAcquireWithRetry(int maxRetries, long retryDelayMs)
            throws InterruptedException {
        for (int i = 0; i < maxRetries; i++) {
            if (tryAcquire()) return true;
            Thread.sleep(retryDelayMs);
        }
        return false;
    }

    public void release() {
        jedis.eval(RELEASE_SCRIPT,
                Collections.singletonList(lockKey),
                Collections.singletonList(lockValue));
    }
}
```

### Usage Pattern

```java
public void sellProduct(String productId) throws InterruptedException {

    RedisDistributedLock lock = new RedisDistributedLock(
            jedis,
            "lock:product:" + productId,
            30_000  // 30 sec TTL
    );

    boolean acquired = lock.tryAcquireWithRetry(3, 200);

    if (!acquired) {
        throw new RuntimeException("Could not acquire lock for product " + productId);
    }

    try {
        // Critical section — only one server runs this at a time
        Product product = productRepo.findById(productId);
        if (product.getStock() <= 0) {
            throw new RuntimeException("Out of stock");
        }
        product.setStock(product.getStock() - 1);
        productRepo.save(product);

    } finally {
        lock.release();  // always release in finally block
    }
}
```

## Spring Boot with Redisson (Production-Grade)

Redisson is a production-grade Redis client with a built-in distributed lock that handles edge cases (watchdog for TTL extension, fair locks, etc.).

### Dependencies

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.25.0</version>
</dependency>
```

### Configuration

```yaml
spring:
  redis:
    host: localhost
    port: 6379
```

### Usage

```java
@Service
public class ProductService {

    @Autowired
    private RedissonClient redisson;

    public void sellProduct(String productId) {

        RLock lock = redisson.getLock("lock:product:" + productId);

        try {
            // Try to acquire, wait up to 5s, auto-release after 30s
            boolean acquired = lock.tryLock(5, 30, TimeUnit.SECONDS);

            if (!acquired) {
                throw new RuntimeException("Could not acquire lock");
            }

            // Critical section
            Product product = productRepo.findById(productId).orElseThrow();
            if (product.getStock() <= 0) throw new RuntimeException("Out of stock");
            product.setStock(product.getStock() - 1);
            productRepo.save(product);

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

**Redisson Watchdog:** if the lock TTL is set to 30 seconds but the operation takes longer, Redisson automatically extends the TTL in the background every 10 seconds (while the lock holder is alive). This prevents the lock from expiring while the holder is still working — without requiring you to set a very long TTL upfront.

## Node.js Implementation

```javascript
const redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');

const client = new redis();

const RELEASE_SCRIPT = `
  if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
  else
    return 0
  end
`;

async function acquireLock(lockKey, ttlMs, retries = 3, retryDelayMs = 200) {
    const lockValue = uuidv4();

    for (let i = 0; i < retries; i++) {
        const result = await client.set(lockKey, lockValue, 'NX', 'PX', ttlMs);
        if (result === 'OK') {
            return lockValue;  // return value so caller can release
        }
        await new Promise(res => setTimeout(res, retryDelayMs));
    }

    return null;  // could not acquire
}

async function releaseLock(lockKey, lockValue) {
    await client.eval(RELEASE_SCRIPT, 1, lockKey, lockValue);
}

// Usage
async function sellProduct(productId) {
    const lockKey = `lock:product:${productId}`;
    const lockValue = await acquireLock(lockKey, 30000);

    if (!lockValue) {
        throw new Error('Could not acquire lock');
    }

    try {
        const product = await productRepo.findById(productId);
        if (product.stock <= 0) throw new Error('Out of stock');
        product.stock -= 1;
        await productRepo.save(product);
    } finally {
        await releaseLock(lockKey, lockValue);
    }
}
```

## Redlock Algorithm (Multi-Node Distributed Lock)

Standard single-Redis lock has a flaw: if the Redis primary fails and failover happens to a replica that hasn't synced the lock yet, two clients could both believe they hold the lock simultaneously.

**Redlock** (designed by Redis's creator) solves this by acquiring the lock on N independent Redis masters (typically 5):

```
1. Get current timestamp T1
2. Try to acquire lock on all 5 Redis nodes (with short timeout per node)
3. Count how many nodes granted the lock
4. If majority (3+) granted AND total elapsed time < TTL:
   Lock is acquired
5. Otherwise:
   Release on all nodes that granted, try again later
```

**Why majority?** At least 3 of 5 nodes must agree. If any single node fails, the majority (3 of 4 remaining) still holds, and at most one client can hold the majority at any time.

**Redlock in Java (Redisson):**

```java
RLock lock1 = redisson1.getLock("lock:product:123");
RLock lock2 = redisson2.getLock("lock:product:123");
RLock lock3 = redisson3.getLock("lock:product:123");

RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
redLock.lock();
try {
    // critical section
} finally {
    redLock.unlock();
}
```

**Redlock is controversial** — Martin Kleppmann argued it has edge cases under clock skew and process pauses. For most use cases (flash sales, idempotency, job deduplication), a single Redis lock is sufficient. Use Redlock only when the cost of two processes both holding the lock simultaneously is very high.

## Common Use Cases

| Use Case | Lock Key Pattern | Reason |
|---|---|---|
| Flash sale / inventory | `lock:product:{id}` | Prevent overselling |
| Payment deduplication | `lock:payment:{orderId}` | Prevent double charge |
| Cron job deduplication | `lock:cron:{jobName}` | Only one instance runs the job |
| Cache re-population | `lock:cache:{key}` | Prevent cache stampede / thundering herd |
| User account operations | `lock:user:{userId}` | Prevent concurrent profile updates |
| Email/SMS sending | `lock:notification:{userId}:{type}` | Prevent duplicate sends |

## TTL Design: How Long Should the Lock Live?

The TTL is your safety net — if the lock holder crashes, the lock auto-expires so others aren't blocked forever.

```
TTL should be > expected operation time (safety margin)
TTL should be < acceptable blocking time for other waiters
```

For a DB update taking ~100ms: TTL = 5–10 seconds is fine.
For a payment flow taking ~2s: TTL = 30 seconds.
For a report generation taking ~60s: use Redisson watchdog (auto-extends TTL) instead of a fixed large TTL.

**Never set TTL = 0 (no expiry)** — if the holder crashes, the lock never releases (deadlock).

## Fencing Tokens (Correctness Under Clock Issues)

Even with a TTL, there's a subtle race:

```
1. Server 1 acquires lock (TTL = 30s)
2. Server 1 gets paused (GC pause, VM migration) for 35 seconds
3. Lock expires
4. Server 2 acquires lock
5. Server 1 resumes and writes to the DB — NOW two servers are writing!
```

**Fencing token** solution: each lock acquisition gets a monotonically increasing token from the lock service. The protected resource (DB, storage) rejects writes with a token number lower than the highest seen.

```
Server 1 acquires lock, token = 33
Server 1 pauses, lock expires, token = 33 discarded by resource
Server 2 acquires lock, token = 34
Server 2 writes, resource records highest_token = 34
Server 1 resumes, tries to write with token = 33
Resource: 33 < 34 -> REJECT
```

This is the strongest correctness guarantee but requires the downstream resource to support fencing.

## Interview Questions

**Q1: Why can't you use a Java `synchronized` block across multiple servers?**

`synchronized` is a JVM-level lock — it only works within a single JVM process. When the same code runs on two different servers (different JVMs), each server has its own lock state and they have no awareness of each other. A distributed lock stores lock state in a shared external store (Redis) that all servers can see.

**Q2: Why must the lock value be unique (a UUID)?**

So that a lock holder can only release its own lock. Without a unique value, Server A could accidentally delete a lock held by Server B (e.g. if Server A's lock expired and Server B re-acquired it, then Server A tries to release it). The check-and-delete Lua script ensures only the holder with the matching value can delete the key.

**Q3: Why does the Redis lock use a TTL?**

As a safety mechanism. If the lock holder crashes or hangs after acquiring the lock, the lock would never be released, blocking all other processes forever (deadlock). The TTL ensures the lock auto-expires after a maximum duration, allowing other processes to acquire it.

**Q4: What is the Redlock algorithm and when do you need it?**

Redlock acquires a lock on a majority of N independent Redis masters. This handles the case where a single Redis master fails and its replica (which may not have the lock key yet) takes over — with Redlock, you'd need to lose the majority of nodes simultaneously for correctness to break. Use it when the cost of two simultaneous lock holders is catastrophic. For most use cases (flash sales, deduplication), a single Redis lock is sufficient.

**Q5: What is the Redisson watchdog?**

A background thread that periodically extends the lock's TTL while the holder is still alive and working. This prevents the lock from expiring mid-operation without requiring you to set an excessively long TTL. If the holder process dies, the watchdog stops, the TTL is not extended, and the lock eventually expires.

**Q6: How do you prevent a cron job from running on multiple servers simultaneously?**

Use a distributed lock with the job name as the key and a TTL slightly longer than the expected job duration. Each server tries to acquire the lock at job start — only one succeeds and runs the job. Others fail to acquire and skip execution. Use Redisson's watchdog if the job duration is variable.
