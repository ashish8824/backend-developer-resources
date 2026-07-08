# Module 16: Redis Cluster & Replication

**Level:** Advanced

So far we've talked about Redis as if it's a single server. In production, that single server is both a **single point of failure** and a **scaling ceiling**. This module covers how Redis solves both problems.

---

## 1. The Two Separate Problems

1. **"What if my Redis server crashes?"** → solved by **Replication** (High Availability).
2. **"What if my dataset is too big / too much traffic for one server?"** → solved by **Clustering** (Horizontal Scaling / Sharding).

These are related but distinct concepts — let's tackle them one at a time.

---

## 2. Replication: Master-Replica Setup

**The idea:** Run one "master" (primary) Redis instance that handles all writes, and one or more "replica" instances that continuously copy everything the master does. If the master goes down, a replica can be promoted to take over.

```
                Writes
                  │
                  ▼
             ┌─────────┐
             │ Master  │
             └────┬────┘
        (continuously replicates data)
                  │
       ┌──────────┼──────────┐
       ▼                     ▼
 ┌───────────┐        ┌───────────┐
 │ Replica 1 │        │ Replica 2 │
 └───────────┘        └───────────┘
   (read-only)          (read-only)
```

**Why replicas matter:**

- **Read scaling:** Send read-heavy traffic (`GET`) to replicas, while the master focuses on writes — spreading load across multiple servers.
- **High availability:** If the master crashes, a replica can be promoted to become the new master, minimizing downtime.
- **Backups:** You can safely run backup/analysis operations against a replica without impacting the master's performance.

**Basic setup (conceptual, via `redis.conf` on the replica):**

```
# On the replica server's config:
replicaof <master-ip> <master-port>

# Example:
replicaof 192.168.1.10 6379
```

Or live, via the CLI:

```bash
# Run this ON the replica, pointing it at the master
REPLICAOF 192.168.1.10 6379

# To stop being a replica and become independent again
REPLICAOF NO ONE
```

**Connecting from Node.js with read/write splitting (conceptual):**

```js
const masterClient = new Redis({ host: "192.168.1.10", port: 6379 }); // writes go here
const replicaClient = new Redis({ host: "192.168.1.11", port: 6379 }); // reads go here

async function getUser(id) {
  return replicaClient.get(`user:${id}`); // read from replica — reduces load on master
}

async function saveUser(id, data) {
  return masterClient.set(`user:${id}`, JSON.stringify(data)); // writes MUST go to master
}
```

---

## 3. Automatic Failover: Redis Sentinel

Manually promoting a replica when the master dies is slow and error-prone. **Redis Sentinel** is a separate system that monitors your master/replica setup and **automatically promotes a replica to master** if it detects the current master has failed — with no manual intervention needed.

```
   Sentinel 1 ──┐
   Sentinel 2 ──┼── continuously monitor ──▶ Master + Replicas
   Sentinel 3 ──┘

If Master goes down:
   Sentinels "vote" (majority agreement) → promote a Replica → new Master
   Sentinels also notify connected clients about the new master's address
```

`ioredis` has built-in Sentinel support:

```js
const redis = new Redis({
  sentinels: [
    { host: "sentinel1", port: 26379 },
    { host: "sentinel2", port: 26379 },
    { host: "sentinel3", port: 26379 },
  ],
  name: "mymaster", // the name Sentinel uses to identify the master group
});

// ioredis automatically asks the Sentinels "who's the current master?"
// and reconnects automatically if a failover happens — you don't manage this manually.
```

You typically run **at least 3 Sentinel processes** (an odd number) so they can reach a majority "quorum" vote and avoid ambiguous split-decisions about who the real master is.

---

## 4. Clustering: Sharding Data Across Multiple Nodes

Replication solves *availability*, but every replica still holds a **full copy** of the *entire* dataset — it doesn't help if your dataset itself is simply too big to fit on one machine, or if write throughput on a single master becomes the bottleneck.

**Redis Cluster** solves this by **sharding**: splitting your keyspace across multiple master nodes, where each node is responsible for only a *portion* of the total data.

```
                 Total keyspace: 16,384 "hash slots"
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   Node A            Node B             Node C
 (slots 0-5460)   (slots 5461-10922) (slots 10923-16383)
```

Redis decides which node a key belongs to using a hash function on the key name (`CRC16(key) % 16384`), which deterministically maps every possible key to one of the 16,384 "hash slots," which are in turn distributed across your cluster nodes.

```
Key "user:15"  ──▶ hash("user:15") % 16384 = 3421 ──▶ lives on Node A (which owns slots 0-5460)
Key "user:999" ──▶ hash("user:999") % 16384 = 12000 ──▶ lives on Node C (which owns slots 10923-16383)
```

Each node in a real cluster would also typically have its own replica(s) for high availability — combining clustering (scale) with replication (safety) is the standard production pattern.

```
Node A (master) ──▶ Node A-replica
Node B (master) ──▶ Node B-replica
Node C (master) ──▶ Node C-replica
```

**Connecting to a cluster from Node.js:**

```js
const Redis = require("ioredis");

const cluster = new Redis.Cluster([
  { host: "10.0.0.1", port: 6379 },
  { host: "10.0.0.2", port: 6379 },
  { host: "10.0.0.3", port: 6379 },
]);

// You use it EXACTLY like a normal client — ioredis automatically
// figures out which node a key belongs to and routes the command there for you.
await cluster.set("user:15", JSON.stringify({ name: "Ashish" }));
const user = await cluster.get("user:15");
```

---

## 5. An Important Cluster Limitation: Multi-Key Operations

Because data is spread across different physical nodes, operations involving **multiple keys** only work if all those keys happen to live on the **same node** (i.e., map to the same hash slot).

```js
// ❌ This may FAIL in a cluster if "user:15" and "user:99" happen to live on different nodes
await cluster.mget("user:15", "user:99");
```

**Solution: Hash Tags** — force related keys onto the same node by including a common `{tag}` portion in the key name. Redis only hashes the part inside the `{}` braces when deciding which node a key belongs to.

```js
// Both keys share the SAME tag "user:15", so Redis guarantees they land on the same node,
// making multi-key operations between them safe.
await cluster.mset("user:15:{user:15}", "...", "user:15:profile:{user:15}", "...");
```

This is a subtle but important gotcha to know about if you ever design a real Redis Cluster deployment.

---

## 6. Replication vs Clustering — Side-by-Side

| | Replication | Clustering |
|---|---|---|
| Solves | High availability (crash recovery) | Horizontal scaling (data too big / too much load for one node) |
| Each node has | A full copy of ALL data | Only a PORTION (shard) of the data |
| Typical setup | 1 master + N replicas | Multiple masters, each usually with their own replica(s) |
| Failover | Sentinel promotes a replica | Cluster has built-in failure detection between nodes |

Most serious production deployments use **both together**: a cluster of multiple shards, where each shard is itself a small master+replica group.

---

## 7. Interview Question: "What's the difference between Redis replication and Redis Cluster?"

**Answer:** Replication is about *redundancy* — every replica holds a complete copy of the same data, primarily for read scaling and failover safety. Redis Cluster is about *sharding* — splitting the total dataset across multiple independent master nodes so no single machine has to hold or serve all the data, which is what lets Redis scale horizontally beyond the memory/throughput limits of one server. In production, both are typically combined: a cluster of shards, each shard internally protected by its own replica(s).

---

## 8. Summary

- **Replication** (master + replicas) provides read scaling and failover safety — every replica has a full copy of the data.
- **Redis Sentinel** automates failover, promoting a replica to master automatically if the current master fails.
- **Redis Cluster** shards data across multiple nodes using hash slots, enabling true horizontal scaling.
- Multi-key operations only work reliably across keys on the same node — use hash tags (`{tag}`) to keep related keys co-located.
- Production systems typically combine clustering (for scale) with replication (for safety) within each shard.

---

## 9. Homework

1. In your own words, explain why replication alone doesn't help if your dataset is too big for one server's RAM.
2. Explain what a "hash slot" is and why Redis Cluster uses 16,384 of them specifically (hint: it's a deliberate, practical design trade-off — look up why if you're curious!).
3. Describe a scenario where you'd need Sentinel even if you're not using Redis Cluster at all.

---

**Next up:** [Module 17 — Production Best Practices](./17-production-best-practices.md)
