# Transactions & ACID Properties

## What is a Database Transaction?

A transaction is a unit of work that groups one or more database operations (INSERT, UPDATE, DELETE, SELECT) into a single logical operation. Either all operations in the transaction succeed together, or none of them do.

**Classic example — bank transfer:**

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- debit Alice
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- credit Bob
COMMIT;
```

If the server crashes after the debit but before the credit, the transaction rolls back — Alice's money is not lost. Without a transaction, you'd have a partial update and money would vanish.

## ACID Properties

ACID is the set of four guarantees a database transaction must provide to be reliable.

### A — Atomicity

"All or nothing." Every operation in the transaction either commits fully or rolls back completely. There is no partial state.

```
Transfer $500:
  Debit Alice  ✅
  Credit Bob   ❌ (server crashes)

Result: entire transaction rolled back.
Alice's balance: unchanged.
Bob's balance:   unchanged.
No money lost.
```

How it works internally: the DB writes changes to a Write-Ahead Log (WAL) / undo log before applying them. On crash, uncommitted changes are rolled back using the undo log.

### C — Consistency

The database must move from one valid state to another valid state. All defined constraints, rules, and cascades must hold before and after the transaction.

```
Constraint: balance >= 0

Alice has $200.
Transaction: debit $500.

Result: REJECTED — would violate balance >= 0 constraint.
Transaction rolls back.
Database stays in a consistent state.
```

Consistency is partly enforced by the DB (constraints, foreign keys, triggers) and partly by the application (business rules).

### I — Isolation

Concurrent transactions must not interfere with each other. Each transaction should execute as if it were the only one running, even when hundreds run simultaneously.

```
Transaction T1: reads Alice's balance (sees $1000)
Transaction T2: deducts $200 from Alice's balance
Transaction T1: reads Alice's balance again

Without isolation: T1 sees $800 (dirty/non-repeatable read)
With isolation:    T1 sees $1000 consistently throughout
```

Isolation is the most nuanced property — it comes in levels (covered below).

### D — Durability

Once a transaction commits, its changes are permanent — even if the server crashes immediately after.

```
COMMIT; -- returns success
Server crashes 1ms later.
DB restarts.
Transaction data: still there, fully applied.
```

How it works: the DB flushes committed data to disk (via WAL/redo log) before returning success. On restart, the redo log replays committed transactions.

## Isolation Levels

Isolation is expensive — full isolation means locking, which kills concurrency. So SQL standard defines four levels, each making a different tradeoff between correctness and performance.

### Read Phenomena (Problems Isolation Prevents)

**Dirty Read:** reading data written by an uncommitted transaction.

```
T1 writes balance = $0 (not yet committed)
T2 reads balance = $0  (dirty read — T1 might roll back!)
T1 rolls back
T2 made a decision based on data that never existed
```

**Non-Repeatable Read:** reading the same row twice within a transaction and getting different values because another transaction committed between the two reads.

```
T1 reads balance = $1000
T2 updates balance = $800 and commits
T1 reads balance again = $800  (different from first read!)
```

**Phantom Read:** re-executing a range query and getting different rows because another transaction inserted or deleted rows between the two reads.

```
T1: SELECT COUNT(*) FROM orders WHERE status='PENDING' -> 5
T2: INSERT INTO orders (status='PENDING') -> commits
T1: SELECT COUNT(*) FROM orders WHERE status='PENDING' -> 6 (phantom row appeared!)
```

**Lost Update:** two transactions both read a value, both modify it, both write back — one's update overwrites the other's.

```
T1 reads stock = 10, calculates 10-1 = 9
T2 reads stock = 10, calculates 10-1 = 9
T1 writes stock = 9
T2 writes stock = 9  (T1's update is lost! should be 8)
```

### The Four Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| READ UNCOMMITTED | ✅ possible | ✅ possible | ✅ possible | Fastest |
| READ COMMITTED | ❌ prevented | ✅ possible | ✅ possible | Fast |
| REPEATABLE READ | ❌ prevented | ❌ prevented | ✅ possible | Moderate |
| SERIALIZABLE | ❌ prevented | ❌ prevented | ❌ prevented | Slowest |

**READ UNCOMMITTED:** transactions can read uncommitted changes from others. Almost never used — too dangerous. Only useful for approximate analytics where a little inaccuracy is acceptable.

**READ COMMITTED:** the default in PostgreSQL, Oracle, SQL Server. Only committed data is visible. Prevents dirty reads but not non-repeatable reads — the most common real-world choice.

**REPEATABLE READ:** the default in MySQL/InnoDB. Once a transaction reads a row, that row's value is frozen for the transaction's lifetime. Prevents dirty and non-repeatable reads. MySQL's implementation also prevents phantom reads (via gap locks).

**SERIALIZABLE:** full isolation — transactions execute as if they were serial (one after another). Implemented via range locks or MVCC with conflict detection. Safest but most likely to cause contention and deadlocks.

### Setting Isolation Level in Java (JDBC)

```java
Connection conn = dataSource.getConnection();
conn.setTransactionIsolation(Connection.TRANSACTION_REPEATABLE_READ);
conn.setAutoCommit(false);

try {
    // ... queries ...
    conn.commit();
} catch (Exception e) {
    conn.rollback();
} finally {
    conn.close();
}
```

### Setting Isolation Level in Spring Boot

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transferFunds(Long fromId, Long toId, BigDecimal amount) {
    Account from = accountRepo.findById(fromId).orElseThrow();
    Account to   = accountRepo.findById(toId).orElseThrow();

    if (from.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }

    from.setBalance(from.getBalance().subtract(amount));
    to.setBalance(to.getBalance().add(amount));

    accountRepo.save(from);
    accountRepo.save(to);
}
```

## Optimistic vs Pessimistic Locking

Two strategies to handle concurrent modifications to the same data.

### Pessimistic Locking

Lock the row when you read it, hold the lock until the transaction commits. Other transactions trying to read/write the locked row must wait.

```sql
BEGIN;
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;  -- locks the row
-- ... process ...
UPDATE accounts SET balance = balance - 500 WHERE id = 1;
COMMIT;
-- lock released
```

**Pros:** guarantees no concurrent modification — safe for critical operations (payments).
**Cons:** high contention, deadlock risk, poor throughput under heavy concurrent load.

### Optimistic Locking

Don't lock on read. Instead, add a `version` column. On write, check that the version hasn't changed since you read it — if it has, someone else modified the row in between, and you retry.

```sql
-- Read
SELECT id, balance, version FROM accounts WHERE id = 1;
-- balance=1000, version=5

-- Write (check version hasn't changed)
UPDATE accounts
SET balance = 500, version = 6
WHERE id = 1 AND version = 5;
-- affected rows = 0 -> someone else updated, retry
-- affected rows = 1 -> success
```

In Spring Data JPA:

```java
@Entity
public class Account {
    @Id
    private Long id;

    private BigDecimal balance;

    @Version  // JPA manages this automatically
    private Long version;
}
```

JPA automatically adds `AND version = ?` to every UPDATE and throws `OptimisticLockException` if the version check fails.

**Pros:** no locking overhead — high throughput when conflicts are rare.
**Cons:** transactions may need to retry on conflict — bad when conflicts are frequent.

**Rule of thumb:** use optimistic locking when conflicts are rare (most reads, occasional writes); use pessimistic locking when conflicts are frequent (high contention on a few hot rows, financial operations).

## Deadlocks

A deadlock occurs when two transactions each hold a lock the other needs — both wait forever.

```
T1 locks Row A, waits for Row B
T2 locks Row B, waits for Row A
Both wait -> deadlock
```

**DB resolution:** databases detect deadlocks automatically and kill one of the transactions (the "deadlock victim"), which then rolls back so the other can proceed.

**Prevention strategies:**
- Always acquire locks in the same order across all transactions (e.g. always lock the lower user_id first in a transfer)
- Keep transactions short — hold locks for as little time as possible
- Use lower isolation levels where safe (fewer locks needed)
- Use optimistic locking to avoid locks altogether

## Savepoints

Partial rollback within a transaction — roll back to a named checkpoint without aborting the whole transaction.

```sql
BEGIN;
  INSERT INTO orders (id, total) VALUES (1, 500);
  SAVEPOINT after_order;

  INSERT INTO order_items (order_id, product) VALUES (1, 'Widget');
  -- something goes wrong with items
  ROLLBACK TO SAVEPOINT after_order;

  -- order still exists, items rolled back
  INSERT INTO order_items (order_id, product) VALUES (1, 'Gadget');
COMMIT;
```

Useful for complex business flows where a sub-step failure shouldn't abort everything.

## Transaction Propagation in Spring

Spring's `@Transactional` has a `propagation` attribute controlling how transactions interact when one transactional method calls another.

| Propagation | Behavior |
|---|---|
| `REQUIRED` (default) | Join existing transaction if present; create new one if not |
| `REQUIRES_NEW` | Always create a new transaction; suspend the current one |
| `NESTED` | Create a nested transaction (savepoint) within the current one |
| `SUPPORTS` | Join if exists; run non-transactionally if not |
| `NOT_SUPPORTED` | Suspend any current transaction; run non-transactionally |
| `MANDATORY` | Must have an existing transaction; throw if none |
| `NEVER` | Must NOT have a transaction; throw if one exists |

**Common interview trap:**

```java
@Service
public class OrderService {

    @Transactional
    public void placeOrder() {
        saveOrder();
        sendNotification();  // this should NOT roll back the order if it fails
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void sendNotification() {
        // runs in its own transaction — failure here doesn't roll back placeOrder()
    }
}
```

**Another common trap — self-invocation:**

```java
@Service
public class MyService {

    public void doSomething() {
        this.processOrder();  // WRONG — @Transactional won't work here!
    }

    @Transactional
    public void processOrder() { ... }
}
```

Spring's `@Transactional` works via a proxy — calling a method on `this` bypasses the proxy and the transaction annotation has no effect. Fix: inject the bean and call it through the injected reference, or refactor into a separate class.

## Write-Ahead Logging (WAL)

WAL is the mechanism databases use to guarantee both atomicity and durability.

```
1. Before modifying data pages, write the change to the WAL (append-only log on disk)
2. Once WAL is flushed to disk, the change can be applied to data pages in memory
3. COMMIT: WAL entry for commit is written and flushed to disk -> COMMIT returns success
4. On crash: replay WAL from last checkpoint to restore committed changes
            roll back any incomplete transactions using undo log
```

The WAL is append-only — sequential writes, which are very fast on both spinning disk and SSDs. This is why "write to WAL then return success" is fast even though it involves a disk flush.

## Common Mistakes

**1. Transactions too long:**

```java
@Transactional
public void process() {
    fetchDataFromDB();          // lock acquired
    callExternalPaymentAPI();   // 5 second HTTP call while holding DB lock!
    updateDB();
}
```

Never make external HTTP calls, send emails, or do slow I/O inside a transaction. Locks are held for the duration — this kills throughput.

**2. Swallowing exceptions:**

```java
@Transactional
public void save(User user) {
    try {
        repo.save(user);
    } catch (Exception e) {
        log.error("error", e);  // exception swallowed — transaction commits anyway!
    }
}
```

Spring only rolls back for unchecked exceptions by default. Swallowing the exception lets the transaction commit in a bad state. Use `@Transactional(rollbackFor = Exception.class)` to roll back on checked exceptions too.

**3. @Transactional on private methods:**

Spring proxies only intercept public methods. A `@Transactional` annotation on a private method is silently ignored.

## Interview Questions

**Q1: What does Atomicity mean? Give a real example.**

Atomicity means all operations in a transaction succeed together or fail together — no partial state. Example: a bank transfer debiting account A and crediting account B must both succeed or both roll back. If the server crashes between the two, the debit is reversed — money is never lost.

**Q2: What is the difference between Isolation and Consistency?**

Consistency means the DB moves between valid states satisfying all constraints — enforced by rules like foreign keys and CHECK constraints. Isolation means concurrent transactions don't interfere with each other — enforced by locking or MVCC. Consistency is about data rules; isolation is about concurrency.

**Q3: What isolation level does MySQL use by default? What does it prevent?**

MySQL InnoDB defaults to REPEATABLE READ. It prevents dirty reads and non-repeatable reads. MySQL's gap-lock implementation also prevents phantom reads within transactions, which is stronger than the SQL standard requires for this level.

**Q4: What is a Lost Update and how do you prevent it?**

A lost update is when two concurrent transactions both read a value, both modify it, and both write back — one's change overwrites the other's silently. Prevent it with: optimistic locking (version column), pessimistic locking (`SELECT FOR UPDATE`), or atomic DB operations (`UPDATE SET balance = balance - 500` instead of read-modify-write in application code).

**Q5: What is the risk of long-running transactions?**

They hold locks for their entire duration, blocking other transactions that need the same rows. This reduces throughput, increases latency for other operations, and greatly increases the probability of deadlocks. Always keep transactions as short as possible.

**Q6: Why doesn't @Transactional work on self-invocation in Spring?**

Spring implements `@Transactional` via a proxy — when you call `this.method()`, you bypass the proxy and call the real object directly, so Spring never gets a chance to start or manage the transaction. Fix by injecting the bean and calling through the injected reference, or extracting the method into a separate Spring-managed bean.
