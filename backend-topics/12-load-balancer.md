# Load Balancer

## What is a Load Balancer?

A Load Balancer is a server (or service) that distributes incoming network traffic across multiple backend servers (instances). It acts as the single point of contact for clients, hiding the fact that multiple servers exist behind it.

```
Without Load Balancer:
Client -> Server 1 (overloaded, crashes)
Client -> Can't connect

With Load Balancer:
Client -> Load Balancer -> Server 1 (30% capacity)
                        -> Server 2 (40% capacity)
                        -> Server 3 (30% capacity)
```

## Why do we need it?

### 1. Scalability

One server can only handle so many requests. A load balancer lets you scale horizontally — add more servers and the load balancer distributes traffic across all of them.

### 2. High Availability

If one server fails, the load balancer detects it (via health checks) and stops sending traffic to it. Other servers absorb the load. No single point of failure at the application layer.

### 3. Zero-Downtime Deployments

Deploy new code to one server at a time. The load balancer routes around the server being updated, so users experience no downtime (rolling deployment).

### 4. Reduced Latency

Load balancers can route requests to the geographically nearest server, or the least loaded server, reducing response time.

### 5. SSL Termination

Handle HTTPS decryption at the load balancer, freeing backend servers from TLS overhead.

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer)

Operates at the TCP/UDP level. Makes routing decisions based on IP address and port number only — doesn't inspect packet contents.

```
TCP connection: 192.168.1.1:443 -> route to Server 2
```

- Faster (no content inspection)
- Cannot make content-based decisions (can't route `/api/users` to one server and `/api/orders` to another)
- Used for: raw TCP load balancing, non-HTTP protocols, ultra-high throughput

### Layer 7 (Application Layer)

Operates at the HTTP/HTTPS level. Can inspect request headers, URL path, cookies, and body to make intelligent routing decisions.

```
GET /api/users  -> Route to User Service cluster
GET /api/orders -> Route to Order Service cluster
POST /upload    -> Route to high-memory upload servers
```

- Smarter routing decisions
- SSL termination
- Can inject/modify headers
- Slightly more overhead than L4
- Used for: almost all modern web applications

## Load Balancing Algorithms

### 1. Round Robin

Distribute requests to servers in sequence: 1, 2, 3, 1, 2, 3, ...

```
Request 1 -> Server 1
Request 2 -> Server 2
Request 3 -> Server 3
Request 4 -> Server 1  (wraps around)
```

- **Pros:** Simple, equal distribution
- **Cons:** Doesn't account for server capacity or current load — a slow server gets as many requests as a fast one
- **Best for:** homogeneous servers with similar capacity and similar request types

### 2. Weighted Round Robin

Servers get different weights based on capacity. Higher-weight servers receive proportionally more requests.

```
Server 1 (weight 3): gets 3 requests
Server 2 (weight 1): gets 1 request
Server 3 (weight 2): gets 2 requests
Repeat.
```

- **Best for:** heterogeneous servers (different CPU/RAM) where some can handle more load

### 3. Least Connections

Route each new request to the server with the fewest active connections.

```
Server 1: 150 connections
Server 2: 80 connections  <- new request goes here
Server 3: 200 connections
```

- **Pros:** Naturally adapts to slow/long connections — servers stuck processing slow requests get fewer new ones
- **Cons:** Slightly more tracking overhead
- **Best for:** long-lived connections (WebSockets, file uploads) where connection count reflects real load

### 4. Least Response Time

Route to the server with both the fewest connections AND the lowest average response time.

```
Server 1: 10 connections, avg 50ms response
Server 2: 8 connections,  avg 200ms response
Server 3: 12 connections, avg 30ms response <- new request goes here
```

- **Pros:** Best for latency-sensitive applications
- **Cons:** Requires active latency measurement
- **Best for:** APIs where response time varies significantly per server

### 5. IP Hash (Sticky Sessions via IP)

Hash the client's IP address to always route the same client to the same server.

```
hash(192.168.1.1) % 3 = 1 -> always Server 1
hash(192.168.1.2) % 3 = 2 -> always Server 2
```

- **Pros:** Session affinity without server-side session sharing — a stateful app works correctly
- **Cons:** If a server goes down, all its clients are re-hashed to other servers (session loss); uneven distribution if many clients share an IP (NAT, corporate proxy)
- **Best for:** legacy stateful apps that can't share session state

### 6. Consistent Hashing

A more sophisticated version of IP hash — uses a ring-based hash so only ~1/N of connections are remapped when a server is added or removed.

### 7. Random

Pick a server at random. Surprisingly effective at scale (by probability, load distributes evenly) and requires no state tracking.

## Algorithm Comparison

| Algorithm | Accounts for Load? | Session Affinity | Best For |
|---|---|---|---|
| Round Robin | No | No | Equal servers, stateless apps |
| Weighted Round Robin | Partially (capacity) | No | Mixed-capacity servers |
| Least Connections | Yes | No | Long-lived connections |
| Least Response Time | Yes | No | Latency-sensitive APIs |
| IP Hash | No | Yes (by IP) | Stateful apps |
| Random | No (statistically) | No | Simple, high scale |

## Health Checks

A load balancer continuously monitors backend servers. If a server fails a health check, it's removed from the pool.

### Passive Health Check

Monitor actual request responses. If a server returns errors or times out, mark it unhealthy.

### Active Health Check

The load balancer periodically sends its own probe requests (HTTP GET `/health`, TCP connection, etc.) to each server, independent of real traffic.

```yaml
# Nginx health check config
upstream backend {
    server app1:8080;
    server app2:8080;
    server app3:8080;

    check interval=5000 rise=2 fall=3 timeout=1000 type=http;
    check_http_send "GET /actuator/health HTTP/1.0\r\n\r\n";
    check_http_expect_alive http_2xx;
}
```

- `interval=5000` — check every 5 seconds
- `rise=2` — mark healthy after 2 consecutive successes
- `fall=3` — mark unhealthy after 3 consecutive failures

## Session Persistence (Sticky Sessions)

Some applications store session state in server memory (not in a shared store like Redis). For these, a client must always be routed to the same server.

### Cookie-Based Persistence

The load balancer injects a cookie identifying which server handled the request. On subsequent requests, it reads the cookie and routes to the same server.

```
Response: Set-Cookie: SERVERID=server2
Next request: Cookie: SERVERID=server2 -> route to server2
```

Preferred over IP hash — works correctly behind NATs, and is more deterministic.

**The real fix:** move session state to Redis and make the app stateless — then you don't need sticky sessions and load balancing is much simpler.

## Types of Load Balancers

### Hardware Load Balancer

Physical appliances (F5 BIG-IP, Citrix ADC). Extremely high performance, expensive. Common in traditional enterprise/finance. Being replaced by software solutions.

### Software Load Balancer

- **Nginx:** widely used, high-performance, also acts as reverse proxy and web server
- **HAProxy:** purpose-built load balancer, extremely high performance, rich health check options
- **Traefik:** Kubernetes-native, auto-discovers services
- **Envoy:** modern L7 proxy, used in service meshes (Istio)

### Cloud Load Balancer

Managed service — no infrastructure to manage:

- AWS: ALB (Application/L7), NLB (Network/L4), GLB (Gateway)
- GCP: Cloud Load Balancing
- Azure: Azure Load Balancer / Application Gateway

## Nginx Load Balancer Configuration

```nginx
upstream order_service {
    least_conn;                    # Least connections algorithm

    server 10.0.0.1:8080 weight=3; # higher weight
    server 10.0.0.2:8080 weight=1;
    server 10.0.0.3:8080 backup;   # only used if others are down
}

server {
    listen 80;
    server_name api.example.com;

    location /api/orders {
        proxy_pass http://order_service;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # Health check
        proxy_connect_timeout 2s;
        proxy_read_timeout 10s;
    }
}
```

## Global Load Balancing (DNS-Based)

For multi-region deployments, DNS-based load balancing routes users to the nearest regional load balancer:

```
User in India   -> DNS -> Mumbai Load Balancer  -> Mumbai Servers
User in Germany -> DNS -> Frankfurt Load Balancer -> Frankfurt Servers
```

Tools: AWS Route 53 (latency-based routing, geolocation routing), Cloudflare, GCP Cloud DNS.

## Load Balancer in Microservices (Client-Side vs Server-Side)

### Server-Side Load Balancing (Traditional)

A dedicated load balancer (hardware or software) sits in front of service instances. Clients talk to the load balancer's single address.

```
Client -> Load Balancer -> Service Instance A
                        -> Service Instance B
```

### Client-Side Load Balancing

The client itself decides which instance to call — using a service registry (Eureka, Consul) to discover available instances and applying a load-balancing algorithm.

```
Service Registry: Order Service = [10.0.0.1:8080, 10.0.0.2:8080, 10.0.0.3:8080]

Client -> fetches list from registry -> picks 10.0.0.2:8080 (round robin) -> calls directly
```

Spring Cloud uses **Spring Cloud LoadBalancer** (replaced the now-deprecated Ribbon) for client-side load balancing in microservices.

```java
// Spring Cloud LoadBalancer — automatic with @LoadBalanced
@Bean
@LoadBalanced
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Uses service name — load balancer resolves to an instance
ResponseEntity<Order> response = restTemplate.getForEntity(
        "http://order-service/api/orders/123", Order.class);
```

## Production Architecture

```
Internet
    |
DNS (Route 53 / Cloudflare) -- geographic routing
    |
Global Load Balancer (Layer 4)
    |
Regional Load Balancer (Layer 7 — Nginx / ALB)
    |
  +---+---+
  |       |
App 1  App 2  App 3  (auto-scaled instances)
    |
Redis (shared session / cache)
    |
Database Cluster
```

## Interview Questions

**Q1: What is the difference between Layer 4 and Layer 7 load balancing?**

Layer 4 (transport) makes routing decisions based on IP and TCP port only — fast, but can't inspect request content. Layer 7 (application) reads HTTP headers, URL paths, and cookies to make content-aware routing decisions (e.g. route `/api/users` to one cluster and `/api/orders` to another). Almost all modern web load balancers operate at L7.

**Q2: What is the difference between Round Robin and Least Connections?**

Round Robin distributes requests sequentially — equal count but ignores actual server load. Least Connections routes to the server with the fewest active connections — better when requests have variable duration (some take 10ms, others 5s) since slower servers accumulate connections and naturally get fewer new ones.

**Q3: What are sticky sessions and when would you use them?**

Sticky sessions ensure a client always routes to the same backend server. Used when session state is stored in server memory (not in a shared store). The load balancer sets a cookie identifying the server, and reads it on subsequent requests. The correct long-term fix is to make the app stateless by storing sessions in Redis — then sticky sessions aren't needed at all.

**Q4: How does a load balancer detect that a backend server is unhealthy?**

Via health checks — the load balancer periodically sends HTTP requests (e.g. GET `/health`) to each server and monitors responses. If a server fails a configurable number of consecutive checks, it's marked unhealthy and removed from rotation. When it passes checks again, it's added back. This enables automatic failover without manual intervention.

**Q5: What is the difference between a Load Balancer and an API Gateway?**

A load balancer distributes traffic across multiple instances of the same service for availability and throughput. An API Gateway routes requests to different services based on the URL, and adds cross-cutting concerns like authentication, rate limiting, and logging. They're complementary — a typical architecture has an API Gateway routing to different services, with a load balancer behind each service distributing across its instances.

**Q6: What is client-side load balancing?**

The client itself discovers available service instances (via a service registry like Eureka or Consul) and applies a load-balancing algorithm to choose which instance to call directly. Eliminates the dedicated load balancer hop. Used in microservices architectures with Spring Cloud LoadBalancer or similar. Tradeoff: each client needs load-balancing logic and service-discovery access.
