# Module 17: Production Best Practices

**Level:** Advanced

This module is a practical checklist — the accumulated lessons that separate "Redis running on my laptop" from "Redis running safely in production."

---

## 1. Always Set a Password (Authentication)

By default, older Redis setups have **no password at all** — anyone who can reach the port can read/write/delete everything. Never expose Redis to the public internet without authentication.

```
# In redis.conf
requirepass YourStrongPasswordHere
```

```js
const redis = new Redis({
  host: "127.0.0.1",
  port: 6379,
  password: process.env.REDIS_PASSWORD, // never hardcode secrets in source code!
});
```

Modern Redis (6+) also supports **ACLs** (Access Control Lists), letting you create specific users with limited permissions (e.g., a user that can only read, not delete):

```bash
# Create a read-only user
ACL SETUSER readonly_app on >somepassword ~cache:* +get +mget
```

---

## 2. Never Expose Redis Directly to the Internet

Redis is designed for trusted internal networks — it has minimal built-in protection against network-level attacks. Always:

- Run Redis behind a firewall / private network (VPC).
- Bind Redis to specific internal IPs, not `0.0.0.0` (all interfaces), unless firewalled properly.
- Use TLS if connecting across an untrusted network (most managed cloud Redis providers support this).

```
# In redis.conf — restrict which IP Redis listens on
bind 127.0.0.1 10.0.0.5
```

---

## 3. Set `maxmemory` and a Sensible Eviction Policy (Recap of Module 7)

```
maxmemory 512mb
maxmemory-policy allkeys-lru
```

An unbounded Redis instance is a ticking time bomb in production — always cap it explicitly.

---

## 4. Avoid Dangerous Commands in Production

| Command | Why it's risky | Safer alternative |
|---|---|---|
| `KEYS *` | Blocks the single Redis thread while scanning everything | `SCAN` (incremental, non-blocking) |
| `FLUSHALL` / `FLUSHDB` | Instantly wipes all data | Restrict via ACLs; never run casually |
| Very large `MGET`/`HGETALL` on huge structures | Can create latency spikes | Paginate; avoid oversized single keys |

**Renaming/disabling dangerous commands** is a common production hardening step:

```
# In redis.conf — effectively disable FLUSHALL in production
rename-command FLUSHALL ""
```

---

## 5. Use Pipelining for Bulk Operations (Recap of Module 5)

```js
// ❌ Slow: 1,000 separate network round-trips
for (const item of items) {
  await redis.set(`item:${item.id}`, JSON.stringify(item));
}

// ✅ Fast: batched into far fewer round-trips
const pipeline = redis.pipeline();
for (const item of items) {
  pipeline.set(`item:${item.id}`, JSON.stringify(item));
}
await pipeline.exec();
```

---

## 6. Monitor Key Metrics

Set up dashboards/alerts (via tools like Redis's own `INFO` command, Prometheus + Grafana, Datadog, or your cloud provider's built-in monitoring) for:

- **`used_memory`** — are you approaching your `maxmemory` limit?
- **`connected_clients`** — sudden spikes can indicate a connection leak (e.g., forgetting to reuse a single client, from Module 9).
- **`evicted_keys`** — are you evicting so much that your cache hit rate is suffering?
- **`keyspace_hits` / `keyspace_misses`** — your actual cache hit ratio; low hit rates mean your caching strategy may need tuning.
- **Latency** (`--latency` flag on `redis-cli`, or `SLOWLOG`) — catch slow commands before they become a real problem.

```bash
# Check current slow commands (anything over a configurable threshold, default 10ms)
SLOWLOG GET 10

# Live latency monitoring from the CLI
redis-cli --latency
```

---

## 7. Handle Connection Failures Gracefully (Recap of Module 9)

Never let a Redis outage take down your whole application. Always:

- Wrap Redis calls in `try/catch`.
- Fall back to your database (or a sensible default) on failure.
- Use `retryStrategy` in `ioredis` to auto-reconnect.
- Set reasonable command timeouts so a hung Redis connection doesn't hang your entire request.

```js
const redis = new Redis({
  host: "127.0.0.1",
  port: 6379,
  connectTimeout: 5000,       // fail fast if Redis is unreachable, instead of hanging forever
  maxRetriesPerRequest: 2,    // don't retry a single command forever if Redis is struggling
});
```

---

## 8. Use Environment-Specific Configuration

```js
// config/redisClient.js
const redis = new Redis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD,
  tls: process.env.REDIS_TLS === "true" ? {} : undefined, // enable TLS in production if needed
});
```

Never hardcode connection details — use `.env` files locally (with a library like `dotenv`) and your hosting platform's secret management in production.

---

## 9. Plan for Persistence Deliberately (Recap of Module 8)

Don't just accept Redis's defaults blindly — deliberately choose:

- RDB only? AOF only? Both? Neither?
- Based on: how expensive is losing this specific data if Redis restarts unexpectedly?

---

## 10. Test Your Failover / Disaster Recovery Plan

Don't just assume replication + Sentinel will save you — actually simulate a master failure in a staging environment and confirm:

- A replica gets promoted correctly.
- Your application reconnects to the new master automatically.
- No data corruption or split-brain scenarios occur.

---

## 11. A Practical Production Checklist

- [ ] Password/ACL authentication enabled
- [ ] Not exposed directly to the public internet
- [ ] `maxmemory` and `maxmemory-policy` explicitly configured
- [ ] Dangerous commands (`FLUSHALL`, etc.) restricted or renamed
- [ ] Persistence strategy (RDB/AOF/both/neither) deliberately chosen
- [ ] Monitoring & alerting set up (memory, hit rate, latency, slow log)
- [ ] Application code handles Redis outages gracefully (fallback to DB)
- [ ] Connection reused/pooled, not created per-request
- [ ] Replication + Sentinel (or managed cloud equivalent) for high availability
- [ ] Failover tested in staging, not just assumed to work

---

## 12. Interview Question: "What steps would you take before putting Redis into production?"

**Answer:** Walk through the checklist above conversationally: secure it (auth, network isolation), bound it (maxmemory + eviction policy), harden it (restrict dangerous commands), plan for durability (persistence choice matched to data criticality), make the application resilient to Redis outages (graceful fallback), and set up observability (monitoring key metrics, slow log) — then validate the whole setup, including failover, before trusting it with real traffic.

---

## 13. Summary

Production-readiness for Redis comes down to five pillars: **Security** (auth, network isolation), **Stability** (memory limits, eviction policy), **Resilience** (graceful degradation, replication/failover), **Observability** (monitoring, slow log), and **Discipline** (avoiding dangerous commands, deliberate persistence choices).

---

## 14. Homework

1. Go through the checklist above against your local Redis setup — how many boxes can you already check?
2. Run `SLOWLOG GET 10` after running a deliberately slow command (like `KEYS *` on a large dataset) and observe the output.
3. Write a short paragraph (for yourself) describing your chosen persistence strategy for a hypothetical e-commerce cart system, and why.

---

**Next up:** [Module 18 — Interview Questions](./18-interview-questions.md)
