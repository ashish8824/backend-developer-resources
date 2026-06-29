# CAP Theorem

## What is the CAP Theorem?

The CAP Theorem (also known as Brewer's Theorem) states that a distributed data system can only guarantee two of the following three properties simultaneously:

- **C — Consistency:** Every read returns the most recent write or an error
- **A — Availability:** Every request receives a response (not necessarily the most recent data)
- **P — Partition Tolerance:** The system continues to operate even when network partitions (communication failures) occur between nodes

**In plain English:**

```
C: All nodes see the same data at the same time
A: The system always responds (even if the response might be stale)
P: The system keeps working even if nodes can't communicate with each other
```

## Why Can't You Have All Three?

In a distributed system, **network partitions are inevitable** — networks fail, packets drop, nodes get isolated. This is not optional: if you have multiple servers communicating over a network, partitions will happen.

When a partition occurs, you must choose:

```
[Node 1] ~~~ PARTITION ~~~ [Node 2]

Node 1 receives a write: UPDATE balance = $500

Option A (Choose Consistency):
  Node 1 refuses to respond until partition heals and Node 2 is updated
  -> Sacrifices Availability (no response during partition)

Option B (Choose Availability):
  Node 1 responds with "balance = $500"
  Node 2 (still partitioned) responds with "balance = $200" (old value)
  -> Sacrifices Consistency (different answers from different nodes)
```

Since P (Partition Tolerance) must be handled, the real choice is:
**CA systems don't exist in distributed systems** — you're always choosing between CP and AP.

## The Three Combinations

### CP — Consistency + Partition Tolerance

During a partition: the system refuses to respond (returns error or timeout) to avoid returning stale data. Prioritizes correctness over availability.

```
Partition occurs:
Request arrives -> Node can't verify data is consistent -> Returns error
"System unavailable" is better than "wrong answer"
```

Examples: HBase, Zookeeper, etcd, MongoDB (by default), Redis (single primary), traditional RDBMS

**Use when:** financial transactions, inventory management, anything where stale data causes real harm (you can't sell an item twice)

### AP — Availability + Partition Tolerance

During a partition: the system keeps responding with whatever data it has (potentially stale). Prioritizes uptime over correctness.

```
Partition occurs:
Request arrives -> Node responds with last known data (may be stale)
"Slightly old answer" is better than "no answer"
```

Examples: Cassandra, DynamoDB, CouchDB, Couchbase, DNS

**Use when:** social media feeds, shopping carts, product catalogs, anything where temporary staleness is acceptable

### CA — Consistency + Availability (No Partition Tolerance)

Only possible on a single server or within a single data center where partition is essentially impossible. Traditional RDBMS (single node PostgreSQL, MySQL) are CA — but as soon as you add replication across networks, you're in distributed territory and must handle partitions.

**Not a practical choice** for distributed systems — if you say "CA", you're really saying "single-node."

## PACELC — The Full Picture

PACELC extends CAP to describe trade-offs even when there's NO partition:

```
P: During a Partition:
   Choose between A (Availability) or C (Consistency)

ELC: Else (no partition):
   Choose between L (Latency) and C (Consistency)
```

Even when everything is healthy, you still trade off:

- **Low latency:** respond immediately with local data (possibly slightly stale)
- **Strong consistency:** wait for all replicas to agree before responding (adds latency)

DynamoDB: PA/EL (available during partition, low latency otherwise — eventual consistency)
Cassandra: PA/EL (same — tune consistency level per query)
Spanner: PC/EC (consistent during partition and else — uses atomic clocks for global consistency at the cost of latency)

## Consistency Models (Beyond CAP)

CAP's "Consistency" is specifically **Linearizability** — the strongest model. There's a spectrum:

### Strong Consistency (Linearizability)

Every read sees the result of the most recent write. Reads and writes appear instantaneous and globally ordered.

```
Write: balance = $500 at T1
Read at T1+1ms: balance = $500 (always, on any node)
```

Cost: coordination overhead — all nodes must agree before responding.

### Sequential Consistency

All operations appear in the same order to all nodes, but not necessarily in real-time order. Weaker than linearizability.

### Causal Consistency

Operations that are causally related are seen in the same order by all nodes. Independent operations may be seen in different orders.

```
Write: "Post created" -> causally followed by -> "Comment added"
All nodes see: Post before Comment (causal order preserved)
But two unrelated posts may appear in different orders on different nodes
```

### Eventual Consistency

Given enough time with no new writes, all replicas will converge to the same value.

```
Write: balance = $500 (on Node 1)
Read immediately from Node 2: balance = $200 (stale — not yet propagated)
Read from Node 2 after 200ms: balance = $500 (converged)
```

The "eventually" is usually milliseconds to seconds. No guarantee of when.

### Read-your-own-writes Consistency

A specific user always sees their own writes immediately (but may see stale data from other users' writes).

```
User A writes: profile_pic = "new.jpg"
User A reads:  profile_pic = "new.jpg"  (guaranteed — sees own write)
User B reads:  profile_pic = "old.jpg"  (may be stale — different user)
```

Very important for UX — users expect their own changes to be immediately visible.

## Tunable Consistency (Cassandra)

Cassandra is AP but allows you to tune consistency per query using quorum reads/writes. With N replicas, a quorum is N/2 + 1.

```
N = 3 replicas (on 3 nodes)
Quorum = 2

QUORUM write: must write to 2 of 3 nodes before success
QUORUM read:  must read from 2 of 3 nodes and return latest

If write quorum + read quorum > N:
  2 + 2 > 3 -> True
  -> Reads always see the latest write (strong consistency despite AP system)
```

Consistency levels in Cassandra:

| Level | Reads/Writes to | Consistency | Availability |
|---|---|---|---|
| ONE | 1 node | Weak | Highest |
| QUORUM | N/2+1 nodes | Strong (with QUORUM write) | Medium |
| ALL | All N nodes | Strongest | Lowest |
| LOCAL_QUORUM | Quorum in local DC | Strong in DC | Medium |

## Database Consistency Guarantees

| Database | Type | CAP | Consistency Default |
|---|---|---|---|
| PostgreSQL (single) | RDBMS | CA | Strong (ACID) |
| PostgreSQL (replicated) | RDBMS | CP | Strong primary, eventual replicas |
| MySQL (primary-replica) | RDBMS | CP | Strong primary, eventual replicas |
| MongoDB | Document | CP | Eventual by default, tunable |
| Cassandra | Wide-column | AP | Eventual, tunable |
| DynamoDB | Key-value | AP | Eventual by default, optional strong |
| Redis (single) | Key-value | CP | Strong |
| Redis Cluster | Key-value | CP | Strong per shard |
| HBase | Wide-column | CP | Strong |
| Zookeeper | Coordination | CP | Strong |
| etcd | Coordination | CP | Strong |

## Real-World Examples

### Choose CP: Payment System

```
User pays for order.
Network partition occurs.

CP behavior: reject write -> "Payment system temporarily unavailable"
AP behavior: accept write on Node 1, Node 2 doesn't know
             -> Risk: double charge, or charge without order fulfillment

Money must be correct -> Choose CP.
```

### Choose AP: Social Media Feed

```
User posts a photo.
Network partition occurs.

AP behavior: photo visible to some users immediately, others see it 5 seconds later
CP behavior: post fails until all nodes sync -> poor UX

Seeing a post 5 seconds late is fine -> Choose AP.
```

### Choose AP: Shopping Cart

```
User adds item to cart.
Network partition occurs.

AP behavior: cart updated on Node 1, Node 2 still shows old cart
             When partition heals: merge (maybe show both versions, or last-write-wins)
CP behavior: "Cart unavailable" -> user can't shop

Temporary cart inconsistency is tolerable -> Choose AP.
(Amazon's Dynamo paper specifically describes this choice for shopping carts)
```

## Interview Questions

**Q1: Explain the CAP theorem in simple terms.**

A distributed system can only guarantee two of: Consistency (all nodes see the same data), Availability (always get a response), and Partition Tolerance (works even when network fails). Since network partitions always happen in distributed systems, you're really choosing between Consistency (correct data, may fail) and Availability (always responds, may return stale data).

**Q2: Why can't you always have all three CAP properties?**

When a network partition occurs, two separated nodes can't communicate to stay in sync. If a write happens on Node 1, Node 2 doesn't know. You must choose: refuse to respond until the partition heals (CP — consistent but not available), or respond with potentially stale data (AP — available but not consistent). There's no way to be both available AND consistent when nodes can't communicate.

**Q3: What is eventual consistency and when is it acceptable?**

Eventual consistency means all replicas will converge to the same value given enough time with no new writes — but different nodes may temporarily return different values. It's acceptable when slight staleness doesn't cause harm: social media feeds, product catalogs, user profiles, shopping carts. It's NOT acceptable for financial ledgers, inventory (stock counts), or anything where stale data causes double-spending or data loss.

**Q4: What is the difference between CP and AP systems? Give examples.**

CP systems (HBase, ZooKeeper, MongoDB, Redis) refuse to respond during a partition to avoid returning inconsistent data — correctness over uptime. AP systems (Cassandra, DynamoDB, CouchDB) continue responding during partitions but may return stale data — uptime over correctness. Choose CP for banking, inventory, coordination. Choose AP for social media, analytics, shopping carts.

**Q5: How does Cassandra achieve strong consistency despite being an AP system?**

Cassandra uses tunable consistency — you choose per-query how many nodes must agree. With `QUORUM` writes (write to N/2+1 nodes) AND `QUORUM` reads (read from N/2+1 nodes), if `write quorum + read quorum > N`, every read is guaranteed to see the latest write. This achieves strong consistency without giving up the AP architecture — but at higher latency cost when quorums are used.
