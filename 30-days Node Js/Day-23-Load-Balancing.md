# Day 23 — Load Balancing: Algorithms, Nginx, Health Checks, Sticky Sessions

> **Why this day matters:** Day 15 scaled across CPU cores on ONE machine (`cluster`). Today scales across MULTIPLE machines entirely — the next level up, and a near-guaranteed system design interview topic for any backend role. Understanding load balancing properly also retroactively makes sense of several earlier "why does this break across multiple instances" problems from this course (sessions, WebSockets, rate limiting) — today gives you the missing piece: the actual component that's distributing traffic across those instances in the first place.

---

## 1. Background: horizontal vs vertical scaling

Before load balancing makes sense, you need the underlying scaling decision it serves.

**Vertical scaling ("scale up"):** make ONE server more powerful — more CPU, more RAM, faster disk. Simple conceptually (no architectural changes needed), but has a hard ceiling (there's a maximum size of machine you can rent/buy) and a single point of failure (if that one powerful machine goes down, everything is down).

**Horizontal scaling ("scale out"):** run MULTIPLE smaller servers, each handling a portion of the total traffic. No hard ceiling (keep adding more machines as needed), and inherently more fault-tolerant (one machine failing doesn't take down the whole system) — but it requires something to DISTRIBUTE traffic across those multiple machines. **That "something" is the load balancer**, and it requires the stateless design principles from Day 9 (no single server should be the only one that "remembers" a particular client) to actually work well.

```
                          ┌─────────────┐
                    ┌────>│  Server A    │
                    │     └─────────────┘
┌──────────┐   ┌────┴────┐
│  Clients  │──>│   Load   │     ┌─────────────┐
└──────────┘   │ Balancer │────>│  Server B    │
                └────┬────┘     └─────────────┘
                    │
                    └────>┌─────────────┐
                          │  Server C    │
                          └─────────────┘
```

**Interview tip:** if asked "horizontal vs vertical scaling, which is better," the correct nuanced answer is that they're not mutually exclusive — most real systems use SOME vertical scaling (reasonably sized instances) combined with horizontal scaling (multiple of them) for the best balance of cost, resilience, and ceiling. Pure vertical scaling alone is rarely the long-term answer for a system expecting real growth.

---

## 2. Load Balancing Algorithms

### Round Robin — the simplest, default approach

**The idea:** distribute requests to servers in strict rotating order — Server A, then B, then C, then back to A, repeating indefinitely.

```nginx
upstream backend_servers {
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
    server 10.0.0.3:3000;
}
```

**Weakness:** treats all servers as equally capable and assumes all requests are roughly equal "cost" — if one request happens to be much more expensive (a large report generation) while others are cheap, round robin doesn't account for that; a server could end up disproportionately loaded just from bad luck in request distribution.

### Weighted Round Robin

**The idea:** same rotation concept, but servers with a higher "weight" receive proportionally more requests — useful when your servers AREN'T identical (e.g., one machine has double the CPU/RAM of the others).

```nginx
upstream backend_servers {
    server 10.0.0.1:3000 weight=3; # receives 3x the traffic of the others
    server 10.0.0.2:3000 weight=1;
    server 10.0.0.3:3000 weight=1;
}
```

### Least Connections

**The idea:** send each new request to whichever server CURRENTLY has the fewest active connections — actively adapts to real-time load, rather than blindly rotating regardless of how busy each server actually is.

**Why this matters in practice:** if requests vary significantly in how long they take to process (some quick, some slow), round robin can accidentally pile up slow requests on one server while another sits comparatively idle. Least connections actively avoids this by routing based on REAL current load, not assumed equal load.

```nginx
upstream backend_servers {
    least_conn;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
```

### IP Hash

**The idea:** compute a hash of the CLIENT'S IP address, and use that hash to consistently route the SAME client to the SAME server every time (as long as the server pool doesn't change).

**Why this matters — this is the algorithm that solves Day 20's WebSocket scaling foreshadowing:** if a client always lands on the same server, you get **session affinity** ("sticky sessions") for free, without needing a shared external store for everything — useful specifically for stateful connections (WebSockets) or in-memory session data, though Redis-backed shared state (Day 17-18) remains the more robust, universal solution.

```nginx
upstream backend_servers {
    ip_hash;
    server 10.0.0.1:3000;
    server 10.0.0.2:3000;
}
```

**The real weakness of IP hash, worth naming if probed:** if many clients share one IP (common behind a corporate NAT or large ISP, as mentioned in Day 22), they'll ALL be routed to the SAME server, potentially overloading it disproportionately while others remain underused — an uneven distribution problem that round robin and least connections don't have.

### The Comparison Table

| Algorithm | Distributes based on | Best for | Weakness |
|---|---|---|---|
| Round Robin | Strict rotation | Simple, roughly equal servers and request costs | Ignores real-time load and request cost variance |
| Weighted Round Robin | Rotation + assigned weight | Non-identical server hardware | Weights must be manually tuned/maintained |
| Least Connections | Current active connection count | Variable request durations/costs | Slightly more overhead to track in real-time |
| IP Hash | Client IP | Stateful connections needing session affinity (WebSockets) | Uneven load if many clients share an IP (NAT) |

---

## 3. Nginx as a Reverse Proxy / Load Balancer

### Background: what's a "reverse proxy," and how is it different from a regular ("forward") proxy?

A regular (forward) proxy sits in front of CLIENTS, making requests to the internet ON THEIR BEHALF (hiding the client's identity from servers). A **reverse proxy** sits in front of SERVERS, receiving requests FROM clients and forwarding them to one of potentially several backend servers (hiding the backend servers' details from clients, and adding capabilities like load balancing, SSL termination, and caching at this single entry point).

```nginx
# /etc/nginx/nginx.conf (or a site-specific config file)

upstream backend_servers {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
    # Note: this assumes 3 separate Node.js processes already running
    # locally on different ports -- e.g., via PM2's cluster mode (Day 15),
    # OR 3 entirely separate machines in a real production deployment.
}

server {
    listen 80;
    server_name myapi.com;

    location / {
        proxy_pass http://backend_servers; # forwards requests to the upstream group above

        # These headers preserve information about the ORIGINAL client
        # request that would otherwise be lost once Nginx forwards the
        # request -- without them, your Node app would see EVERY
        # request as coming from Nginx's own internal IP, not the
        # real client's IP (relevant for logging, and for IP-based
        # rate limiting from Day 22!).
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```javascript
// IMPORTANT real-world consequence: your Express app needs to
// explicitly TRUST these forwarded headers to get the real client IP
// (e.g., for rate limiting by IP, Day 22) -- otherwise req.ip would
// just show Nginx's address for every single request.
app.set('trust proxy', true); // tells Express to read X-Forwarded-For correctly
```

**Interview tip:** this `trust proxy` setting is a genuinely common, real "why is my rate limiter / IP logging broken in production" bug — every request appearing to come from the same IP (the load balancer/reverse proxy's address) is a strong signal this setting is missing.

### Why use Nginx specifically, rather than relying solely on Node's `cluster` module?

Recall Day 15's distinction: `cluster` distributes across CORES on ONE machine. Nginx (or a cloud load balancer like AWS ALB) distributes across MULTIPLE MACHINES entirely. In a real production deployment, you'd typically use BOTH together: Nginx (or a cloud LB) distributing traffic across several server machines, each of which runs a clustered Node app (via PM2 or the `cluster` module) using all of ITS OWN cores. This is a two-layer scaling strategy worth being able to draw out and explain.

```
                     ┌──────────────┐
                     │    Nginx /    │
Clients ────────────>│  Cloud Load   │
                     │   Balancer    │
                     └──────┬───────┘
              ┌─────────────┼─────────────┐
              ▼              ▼              ▼
       ┌───────────┐  ┌───────────┐  ┌───────────┐
       │ Machine A  │  │ Machine B  │  │ Machine C  │
       │ (cluster:  │  │ (cluster:  │  │ (cluster:  │
       │ 4 workers) │  │ 4 workers) │  │ 4 workers) │
       └───────────┘  └───────────┘  └───────────┘
       Scaling ACROSS machines (Nginx/cloud LB)
       Scaling ACROSS cores WITHIN each machine (cluster/PM2)
```

---

## 4. Health Checks — keeping the load balancer honest

### Background: what happens without this?

If a backend server crashes, hangs, or starts returning errors for every request, but the load balancer doesn't KNOW that, it will keep sending it a share of traffic anyway — those requests fail for end users, even though OTHER healthy servers could have handled them just fine. **Health checks** let the load balancer periodically verify each backend server is actually responsive and healthy, automatically removing unhealthy servers from rotation until they recover.

```javascript
// A standard, simple health-check endpoint in your Node app -- the
// load balancer (or an external monitoring tool) hits this periodically
app.get('/health', async (req, res) => {
  try {
    // A REAL health check should verify CRITICAL dependencies are
    // actually reachable, not just that the Node process itself is
    // running -- a process can be "up" while its database connection
    // is silently broken, which is a much more useful thing to detect.
    await sequelize.authenticate(); // Day 11's Sequelize connection check
    await redisClient.ping();        // Day 17's Redis connection check

    res.status(200).json({ status: 'healthy', uptime: process.uptime() });
  } catch (err) {
    res.status(503).json({ status: 'unhealthy', error: err.message });
  }
});
```

```nginx
upstream backend_servers {
    server 127.0.0.1:3001 max_fails=3 fail_timeout=30s;
    server 127.0.0.1:3002 max_fails=3 fail_timeout=30s;
    # max_fails: how many consecutive failures before marking a server
    # "down" and temporarily removing it from rotation
    # fail_timeout: how long to wait before trying that server again
}
```

**Interview tip:** when asked "what should a health check actually verify," the strong answer distinguishes between a **liveness check** ("is the process running at all?") and a **readiness check** ("is the process actually ABLE to serve real traffic right now — are its critical dependencies, like the database, reachable?") — this exact liveness/readiness distinction is also the standard terminology used in Kubernetes (foreshadowing Day 26), so knowing it by name transfers directly.

---

## 5. Sticky Sessions — completing Day 20's WebSocket cliffhanger

### Background: when do you actually NEED this?

Recall Day 20: a WebSocket connection is pinned to whichever server handled its initial connection — if a load balancer routes a client's SUBSEQUENT requests to a DIFFERENT server (as round robin naturally would), that breaks the persistent connection model entirely. **Sticky sessions** configure the load balancer to consistently route the SAME client to the SAME backend server for the DURATION of their session, rather than rotating freely.

```nginx
# Cookie-based sticky sessions (one common approach) -- Nginx sets a
# cookie identifying which backend server a client was routed to, and
# uses that cookie to route ALL of that client's SUBSEQUENT requests
# back to the SAME server consistently.
upstream backend_servers {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    sticky cookie srv_id expires=1h domain=.myapi.com path=/;
}
```

**The honest tradeoff to articulate if asked "should you use sticky sessions":** sticky sessions reintroduce a form of statefulness into your otherwise horizontally-scalable architecture — if that ONE pinned server goes down, that client's session/connection is lost (or at minimum disrupted) even though OTHER servers are perfectly healthy. This is exactly why the Day 20 Redis Adapter approach (broadcasting via Redis Pub/Sub across ALL instances, rather than relying on sticky pinning alone) is often considered the MORE ROBUST solution for WebSockets specifically — sticky sessions help, but don't fully eliminate the underlying fragility of pinning a stateful connection to one machine.

---

## 6. How this connects to real backend work (3 YOE framing)

- **"Your API logs show every single request coming from the same IP address, even though you have thousands of distinct users" → what's wrong?"** → Missing `app.set('trust proxy', true)`, or the load balancer/Nginx config isn't forwarding `X-Forwarded-For` correctly — every request appears to originate from the load balancer's own address.
- **"How would you design a deployment that can survive losing an entire server machine without any downtime?"** → Horizontal scaling with multiple machines behind a load balancer with health checks — the load balancer automatically stops routing to the failed machine, and remaining healthy machines absorb the traffic.
- **"Why might round robin load balancing lead to one server being noticeably more loaded than others, even with equal-spec machines?"** → Variance in individual request cost/duration — round robin assumes equal cost per request; least connections adapts to actual real-time load instead.
- **"What's the tradeoff of using sticky sessions for a stateful feature like WebSockets, versus a Redis-backed shared-state approach?"** → Sticky sessions are simpler to set up but reintroduce a single point of fragility per client (losing that one server disrupts their session); a Redis-backed approach (Pub/Sub relaying across instances) is more resilient but requires additional infrastructure and careful design.

---

## 7. Hands-on practice for today

This topic is infrastructure-configuration-heavy rather than code-heavy, so today's practice is conceptual + a runnable local simulation:

```javascript
// practice-day23-server.js
// Run this 3 times with different PORT env vars to simulate 3 backend
// instances locally, then put Nginx (or even a simple custom round-robin
// proxy, shown below) in front of them.
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3001;

app.get('/health', (req, res) => res.json({ status: 'healthy', port: PORT }));
app.get('/', (req, res) => res.send(`Handled by instance on port ${PORT}`));

app.listen(PORT, () => console.log(`Instance running on port ${PORT}`));
```

```bash
PORT=3001 node practice-day23-server.js &
PORT=3002 node practice-day23-server.js &
PORT=3003 node practice-day23-server.js &
```

```javascript
// practice-day23-simple-lb.js -- a TOY round-robin load balancer, purely
// to make the CONCEPT tangible (real production work would use Nginx or
// a cloud load balancer, not hand-rolled code like this)
const http = require('http');
const httpProxy = require('http-proxy'); // npm install http-proxy

const servers = ['http://localhost:3001', 'http://localhost:3002', 'http://localhost:3003'];
let currentIndex = 0;
const proxy = httpProxy.createProxyServer();

const server = http.createServer((req, res) => {
  const target = servers[currentIndex];
  currentIndex = (currentIndex + 1) % servers.length; // round-robin rotation
  proxy.web(req, res, { target });
});

server.listen(8080, () => console.log('Load balancer running on http://localhost:8080'));
```

Hit `http://localhost:8080` repeatedly and confirm the response cycles through ports 3001, 3002, 3003 in order — a direct, hands-on demonstration of round robin distribution.

---

## 8. Interview Q&A Recap (say these out loud)

1. **Q: What's the difference between horizontal and vertical scaling, and how do they typically combine in real systems?**
   A: Vertical scaling makes one server more powerful; horizontal scaling adds more servers. Most real production systems combine both — reasonably sized individual machines (some vertical scaling), multiplied across several instances (horizontal scaling) behind a load balancer, for the best balance of cost, ceiling, and fault tolerance.

2. **Q: Compare round robin and least connections load balancing.**
   A: Round robin distributes requests in strict rotation, assuming roughly equal request cost and server capability. Least connections routes each new request to whichever server currently has the fewest active connections, adapting to real-time load — better suited when request durations/costs vary significantly.

3. **Q: What is a reverse proxy, and how does it differ from a forward proxy?**
   A: A reverse proxy sits in front of backend servers, receiving client requests and forwarding them to one of several backend instances, hiding backend details from clients (and enabling load balancing, SSL termination, etc. at one entry point). A forward proxy sits in front of clients, making requests on their behalf to hide the client from the destination server.

4. **Q: Why might `req.ip` show the wrong value in a load-balanced Node.js deployment, and how do you fix it?**
   A: The load balancer/reverse proxy's own IP is seen instead of the real client's, unless the proxy forwards the original IP via headers like `X-Forwarded-For` AND the Express app is explicitly configured to trust and read those headers via `app.set('trust proxy', true)`.

5. **Q: What's the difference between a liveness check and a readiness check?**
   A: A liveness check verifies the process is running at all. A readiness check verifies the process can actually serve real traffic right now, including that critical dependencies (database, cache, etc.) are reachable — a process can be "alive" but not "ready."

6. **Q: What's the tradeoff of using sticky sessions for WebSocket scaling, versus a Redis Pub/Sub-based broadcasting approach (Day 20)?**
   A: Sticky sessions are simpler but reintroduce fragility — losing the one pinned server disrupts that client's session even if other servers are healthy. A Redis-backed cross-instance broadcasting approach is more resilient (any instance can relay messages to any client) at the cost of additional infrastructure complexity.

---

## Tomorrow (Day 24) Preview
We cover **caching strategies at scale**: CDNs, cache-control HTTP headers, multi-layer caching architectures (browser → CDN → application cache → database), and cache invalidation patterns beyond what Day 17 covered for Redis specifically — taking a broader, system-wide view of where caching fits across an entire request's journey.
