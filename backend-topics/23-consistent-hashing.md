# Consistent Hashing

## What is Consistent Hashing?

Consistent hashing is a technique for distributing data across nodes in a way that minimizes data movement when nodes are added or removed. Unlike simple modulo hashing, adding or removing a node only remaps ~1/N of the keys, not all of them.

## The Problem with Simple Modulo Hashing

```
3 servers, use: server = hash(key) % 3

key "user:123" -> hash = 456789 -> 456789 % 3 = 0 -> Server 0
key "user:456" -> hash = 789012 -> 789012 % 3 = 2 -> Server 2
key "user:789" -> hash = 012345 -> 012345 % 3 = 1 -> Server 1
```

Works fine. But now add a 4th server:

```
server = hash(key) % 4

key "user:123" -> 456789 % 4 = 1 -> Server 1  (was 0 — moved!)
key "user:456" -> 789012 % 4 = 0 -> Server 0  (was 2 — moved!)
key "user:789" -> 012345 % 4 = 1 -> Server 1  (was 1 — same)
```

Nearly every key maps to a different server → massive data migration, all caches invalidated, thundering herd on DB.

**With N→N+1 servers, modulo hashing remaps ~N/(N+1) keys ≈ almost all of them.**

## How Consistent Hashing Works

### The Hash Ring

Map the hash space (0 to 2^32 - 1) onto a conceptual ring. Place both servers and keys on this ring by hashing them.

```
Hash ring (values 0 to 2^32):

            0
           / \
          /   \
Server A      Server B
(hash=90)     (hash=200)
          \   /
           \ /
        Server C
        (hash=310)
```

### Key Assignment

Each key is assigned to the **first server clockwise** from its hash position on the ring.

```
key "user:123" -> hash = 150 -> first server clockwise = Server B (hash=200)
key "user:456" -> hash = 250 -> first server clockwise = Server C (hash=310)
key "user:789" -> hash = 50  -> first server clockwise = Server A (hash=90)
```

### Adding a Server

Add Server D (hash=260):

```
            0
           / \
          /   \
Server A      Server B
(hash=90)     (hash=200)
          \   /
           \ /
   Server D    Server C
   (hash=260)  (hash=310)
```

Only keys between 200 (Server B) and 260 (Server D) need to move — from Server C to Server D. Everything else stays put.

**~1/N keys remapped instead of ~N/(N+1).**

### Removing a Server

Remove Server A (hash=90):

Only keys that were assigned to Server A (between Server C at 310 and Server A at 90) now move to Server B (the next clockwise). All other keys unaffected.

## Virtual Nodes (Vnodes)

With few physical servers, consistent hashing can produce uneven distribution:

```
3 servers: hash values 90, 200, 310
Ranges:
  Server A: 310 -> 90  (range of ~100/360° = 28% of ring)
  Server B: 90 -> 200  (range of ~110/360° = 31% of ring)
  Server C: 200 -> 310 (range of ~110/360° = 31% of ring)
```

Fairly balanced here, but with real data, luck matters. More critically, if Server A is 10x more powerful than B or C, it should hold more data — but it has the same ring segment.

**Solution: Virtual Nodes (Vnodes)**

Each physical server is assigned multiple positions on the ring. Each position is a "virtual node."

```
Server A (150 vnodes): positions at hash(A-0), hash(A-1), ..., hash(A-149) -> spread around ring
Server B (100 vnodes): positions at hash(B-0), ..., hash(B-99)
Server C (50 vnodes):  positions at hash(C-0), ..., hash(C-49)
```

Benefits:

- More vnodes → more even distribution
- Weight servers by capacity: powerful servers get more vnodes
- When a server leaves, its vnodes are distributed among ALL remaining servers, not just one neighbor — smoother load redistribution

Used by: Cassandra (256 vnodes default), DynamoDB, Riak.

## Java Implementation

```java
import java.security.MessageDigest;
import java.util.SortedMap;
import java.util.TreeMap;

public class ConsistentHash<T> {

    private final int virtualNodesPerServer;
    private final SortedMap<Long, T> ring = new TreeMap<>();

    public ConsistentHash(int virtualNodesPerServer) {
        this.virtualNodesPerServer = virtualNodesPerServer;
    }

    public void addServer(T server) {
        for (int i = 0; i < virtualNodesPerServer; i++) {
            long hash = hash(server.toString() + "-vnode-" + i);
            ring.put(hash, server);
        }
    }

    public void removeServer(T server) {
        for (int i = 0; i < virtualNodesPerServer; i++) {
            long hash = hash(server.toString() + "-vnode-" + i);
            ring.remove(hash);
        }
    }

    public T getServer(String key) {
        if (ring.isEmpty()) return null;

        long hash = hash(key);

        // Find first server clockwise from key's position
        SortedMap<Long, T> tailMap = ring.tailMap(hash);

        // If no server after this hash, wrap around to first server on ring
        long serverHash = tailMap.isEmpty() ? ring.firstKey() : tailMap.firstKey();
        return ring.get(serverHash);
    }

    private long hash(String key) {
        try {
            MessageDigest md = MessageDigest.getInstance("MD5");
            byte[] bytes = md.digest(key.getBytes("UTF-8"));
            // Use first 4 bytes as a long
            return ((long)(bytes[3] & 0xFF) << 24)
                 | ((long)(bytes[2] & 0xFF) << 16)
                 | ((long)(bytes[1] & 0xFF) << 8)
                 | ((long)(bytes[0] & 0xFF));
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public int getServerCount() {
        return ring.values().stream().distinct().toList().size();
    }
}
```

### Usage: Distributed Cache Router

```java
public class DistributedCacheRouter {

    private final ConsistentHash<String> hasher;
    private final Map<String, Jedis> serverConnections;

    public DistributedCacheRouter(List<String> cacheServers) {
        this.hasher = new ConsistentHash<>(150);  // 150 vnodes per server
        this.serverConnections = new HashMap<>();

        for (String server : cacheServers) {
            hasher.addServer(server);
            serverConnections.put(server, new Jedis(server));
        }
    }

    public void set(String key, String value) {
        String server = hasher.getServer(key);
        serverConnections.get(server).set(key, value);
    }

    public String get(String key) {
        String server = hasher.getServer(key);
        return serverConnections.get(server).get(key);
    }

    public void addServer(String newServer) {
        hasher.addServer(newServer);
        serverConnections.put(newServer, new Jedis(newServer));
        // Only ~1/N keys now route to new server — minimal migration
    }

    public void removeServer(String server) {
        hasher.removeServer(server);
        serverConnections.remove(server).close();
        // Only that server's keys re-route to neighbors
    }
}
```

## Node.js Implementation

```javascript
const crypto = require("crypto");

class ConsistentHash {
  constructor(virtualNodes = 150) {
    this.virtualNodes = virtualNodes;
    this.ring = new Map(); // hash -> server
    this.sortedHashes = [];
  }

  hash(key) {
    return parseInt(
      crypto.createHash("md5").update(key).digest("hex").substring(0, 8),
      16,
    );
  }

  addServer(server) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const h = this.hash(`${server}-vnode-${i}`);
      this.ring.set(h, server);
      this.sortedHashes.push(h);
    }
    this.sortedHashes.sort((a, b) => a - b);
  }

  removeServer(server) {
    for (let i = 0; i < this.virtualNodes; i++) {
      const h = this.hash(`${server}-vnode-${i}`);
      this.ring.delete(h);
    }
    this.sortedHashes = this.sortedHashes.filter((h) => this.ring.has(h));
  }

  getServer(key) {
    if (this.ring.size === 0) return null;
    const h = this.hash(key);

    // Binary search for first hash >= h
    let lo = 0,
      hi = this.sortedHashes.length - 1;
    while (lo < hi) {
      const mid = Math.floor((lo + hi) / 2);
      if (this.sortedHashes[mid] < h) lo = mid + 1;
      else hi = mid;
    }

    // Wrap around if past the end
    const ringHash =
      this.sortedHashes[lo] >= h ? this.sortedHashes[lo] : this.sortedHashes[0];

    return this.ring.get(ringHash);
  }
}

// Usage
const ch = new ConsistentHash(150);
ch.addServer("cache-1:6379");
ch.addServer("cache-2:6379");
ch.addServer("cache-3:6379");

console.log(ch.getServer("user:123")); // e.g. "cache-2:6379"
console.log(ch.getServer("product:456")); // e.g. "cache-1:6379"

ch.addServer("cache-4:6379"); // only ~25% of keys move
console.log(ch.getServer("user:123")); // may or may not change
```

## Where Consistent Hashing is Used

| System            | Use Case                                                    |
| ----------------- | ----------------------------------------------------------- |
| Cassandra         | Partition data across nodes; vnodes for load balance        |
| DynamoDB          | Internal partitioning (partition key hashed)                |
| Redis Cluster     | 16,384 hash slots across nodes (fixed-slot variant)         |
| Memcached clients | Client-side consistent hashing for cache key routing        |
| Nginx (upstream)  | `hash $request_uri consistent;` directive                   |
| CDN               | Route requests for the same URL to the same edge cache node |
| Load Balancers    | Sticky routing without explicit session tables              |

## Consistent Hashing vs Modulo Hashing vs Range Sharding

|                             | Modulo Hashing | Range Sharding      | Consistent Hashing    |
| --------------------------- | -------------- | ------------------- | --------------------- |
| Distribution                | Uniform        | Depends on data     | Uniform (with vnodes) |
| Keys remapped on add/remove | ~All (N/(N+1)) | ~1/N target range   | ~1/N                  |
| Ordering support            | No             | Yes (range queries) | No                    |
| Hotspot risk                | Low            | Yes (recent data)   | Low (with vnodes)     |
| Complexity                  | Low            | Low                 | Medium                |

## Interview Questions

**Q1: What problem does consistent hashing solve?**

Simple modulo hashing remaps almost all keys when you add or remove a server — causing massive cache invalidation and data migration. Consistent hashing places servers and keys on a hash ring; adding or removing a server only remaps ~1/N keys (only those that were assigned to the affected ring segment). This is critical for cache systems where remapping keys causes cache misses and spikes DB load.

**Q2: How does a key get assigned to a server in consistent hashing?**

Both keys and servers are hashed to positions on a circular ring (0 to 2^32). A key is assigned to the first server clockwise from its position on the ring. If no server exists at a higher hash position, it wraps around to the first server on the ring.

**Q3: What are virtual nodes and why are they important?**

Virtual nodes place each physical server at multiple positions on the ring. This ensures more uniform key distribution (without vnodes, random server positions may create uneven ranges), allows weighting servers differently by capacity (more powerful servers get more vnodes), and ensures that when a server leaves, its load is distributed among all remaining servers rather than piling onto one neighbor.

**Q4: How many keys are remapped when a server is added in consistent hashing?**

Only the keys whose ring segment now falls between the previous server and the new server — roughly 1/N of total keys, where N is the new total number of servers. All other keys are completely unaffected.

**Q5: Where is consistent hashing used in practice?**

Cassandra uses it for partition distribution across nodes with vnodes. DynamoDB uses it internally for partition routing. Redis Cluster uses a variant (16,384 fixed hash slots assigned to nodes). Distributed cache clients (Memcached, custom Redis sharding) use client-side consistent hashing to route keys to cache nodes without a central coordinator.
