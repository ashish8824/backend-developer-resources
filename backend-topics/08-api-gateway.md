# API Gateway

## What is an API Gateway?

An API Gateway is a server that acts as the single entry point for all client requests to your backend services. Instead of clients calling individual microservices directly, every request goes through the gateway, which routes it to the appropriate service.

```
Without API Gateway:
Client -> User Service  (directly)
Client -> Order Service (directly)
Client -> Payment Service (directly)
Each service manages its own auth, rate limiting, logging...

With API Gateway:
Client -> API Gateway -> User Service
                      -> Order Service
                      -> Payment Service
Gateway handles: auth, rate limiting, logging, SSL, routing for everyone
```

## Why do we need it?

### 1. Single Entry Point

Clients only need to know one address. Internal service topology (IPs, ports, how many instances) is hidden.

### 2. Cross-Cutting Concerns in One Place

Without a gateway, every microservice must implement:

- Authentication / Authorization
- Rate Limiting
- Logging
- SSL Termination
- Request Validation
- CORS handling

This is duplication across dozens of services. The gateway centralizes all of it.

### 3. Protocol Translation

Clients may speak HTTP/REST. Backend services may use gRPC, GraphQL, WebSocket, or AMQP. The gateway translates between protocols so clients don't need to know or care.

### 4. Aggregation / BFF (Backend for Frontend)

A mobile app may need data from 3 different services in one screen. Without a gateway, the app makes 3 separate calls. The gateway can aggregate them into one response.

### 5. Security

The internal network is completely hidden. Services are not exposed to the internet at all — only the gateway is. Even if an attacker knows a service's internal address, they can't reach it.

## What an API Gateway Does

### 1. Request Routing

Map incoming URL paths to backend services:

```
GET  /api/users/**      -> User Service    (http://user-service:8081)
GET  /api/orders/**     -> Order Service   (http://order-service:8082)
POST /api/payments/**   -> Payment Service (http://payment-service:8083)
```

### 2. Authentication & Authorization

Validate JWT tokens centrally. If the token is invalid, reject at the gateway — downstream services never see the request.

```
Request with JWT
  -> Gateway extracts token
  -> Validates signature + expiry
  -> Extracts user ID, roles
  -> Forwards to downstream with X-User-Id, X-User-Roles headers
  -> Downstream services trust these headers (no need to re-validate JWT)
```

### 3. Rate Limiting

Enforce per-client or per-endpoint rate limits before the request reaches backend services.

### 4. SSL Termination

The gateway handles HTTPS decryption. Internal service-to-service communication can use plain HTTP (on a private network), reducing TLS overhead across all services.

```
Client --(HTTPS)--> Gateway --(HTTP)--> Services (private network)
```

### 5. Load Balancing

If multiple instances of a service are running, the gateway distributes requests across them.

### 6. Request / Response Transformation

- Add, remove, or modify headers
- Transform request body (e.g. rename fields for backward compatibility)
- Compress response (gzip)
- Add standard response envelope

### 7. Caching

Cache responses for expensive, frequently-called endpoints (e.g. product catalog, config data) — return cached response without hitting the downstream service.

### 8. Circuit Breaker

If a downstream service is failing, the gateway can trip a circuit breaker — returning a cached/default response or an error immediately without queuing up requests that will just time out.

### 9. Logging & Observability

Every request passes through the gateway — one place to log request/response details, latency, status codes, and emit metrics. Correlate traces with a request ID injected by the gateway.

## API Gateway vs Reverse Proxy vs Load Balancer

| Component | Primary Job | Awareness |
|---|---|---|
| Load Balancer | Distribute traffic across instances of ONE service | Layer 4 (IP/TCP) or Layer 7 (HTTP) |
| Reverse Proxy | Forward requests to one or more backend servers; hide backend from internet | Layer 7 (HTTP) |
| API Gateway | Route requests across MULTIPLE different services + cross-cutting concerns | Layer 7 + business logic |

A reverse proxy (Nginx) can do basic routing and SSL. An API Gateway (Kong, AWS API Gateway, Spring Cloud Gateway) adds auth, rate limiting, transformations, and business-aware routing on top.

## API Gateway Patterns

### Backend for Frontend (BFF)

Different clients (web, mobile, IoT) have different data needs. Instead of one gateway serving all, create a separate gateway per client type:

```
Web Browser    -> Web BFF     -> Services
Mobile App     -> Mobile BFF  -> Services (fewer fields, smaller payloads)
IoT Device     -> IoT BFF     -> Services (binary protocol, minimal data)
```

Each BFF knows exactly what its client needs and aggregates/transforms accordingly.

### Aggregation Gateway

One client request triggers multiple backend calls; the gateway merges the results:

```
GET /dashboard
  -> User Service:    GET /users/123
  -> Order Service:   GET /orders?userId=123
  -> Loyalty Service: GET /points?userId=123

Gateway aggregates all three responses:
{
  "user": {...},
  "recentOrders": [...],
  "loyaltyPoints": 450
}
```

Reduces client round trips from 3 to 1.

## Spring Cloud Gateway (Java Implementation)

Spring Cloud Gateway is the modern replacement for Zuul in the Spring ecosystem. It's built on reactive WebFlux (non-blocking).

### Dependencies (pom.xml)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

### Route Configuration (application.yml)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: http://user-service:8081
          predicates:
            - Path=/api/users/**
          filters:
            - StripPrefix=1           # removes /api from forwarded path
            - AddRequestHeader=X-Gateway-Source, api-gateway

        - id: order-service
          uri: http://order-service:8082
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20
```

### Custom Authentication Filter

```java
@Component
public class JwtAuthFilter implements GlobalFilter, Ordered {

    @Autowired
    private JwtService jwtService;

    private static final List<String> PUBLIC_PATHS =
            List.of("/api/auth/login", "/api/auth/register");

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {

        String path = exchange.getRequest().getURI().getPath();

        // Skip auth for public endpoints
        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        String authHeader = exchange.getRequest()
                .getHeaders().getFirst(HttpHeaders.AUTHORIZATION);

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }

        String token = authHeader.substring(7);

        try {
            Claims claims = jwtService.validateToken(token);

            // Forward user info to downstream services via headers
            ServerHttpRequest mutatedRequest = exchange.getRequest()
                    .mutate()
                    .header("X-User-Id", claims.getSubject())
                    .header("X-User-Role", claims.get("role", String.class))
                    .build();

            return chain.filter(exchange.mutate().request(mutatedRequest).build());

        } catch (Exception e) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
    }

    @Override
    public int getOrder() {
        return -1;  // run before other filters
    }
}
```

### Rate Limiting Filter (Redis-backed)

```java
@Bean
public KeyResolver userKeyResolver() {
    // Rate limit per user ID (extracted from header by auth filter)
    return exchange -> Mono.just(
            exchange.getRequest().getHeaders().getFirst("X-User-Id")
    );
}
```

## Node.js API Gateway (Express-based)

```javascript
const express = require('express');
const httpProxy = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');

const app = express();

// Rate limiting
const limiter = rateLimit({ windowMs: 60000, max: 100 });
app.use(limiter);

// Auth middleware
function authenticate(req, res, next) {
    const token = req.headers.authorization?.split(' ')[1];
    if (!token) return res.status(401).json({ error: 'Unauthorized' });

    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.headers['x-user-id'] = decoded.sub;
        req.headers['x-user-role'] = decoded.role;
        next();
    } catch {
        res.status(401).json({ error: 'Invalid token' });
    }
}

// Routes
app.use('/api/users', authenticate, httpProxy.createProxyMiddleware({
    target: 'http://user-service:8081',
    changeOrigin: true,
    pathRewrite: { '^/api/users': '' }
}));

app.use('/api/orders', authenticate, httpProxy.createProxyMiddleware({
    target: 'http://order-service:8082',
    changeOrigin: true,
    pathRewrite: { '^/api/orders': '' }
}));

app.listen(8080);
```

## Popular API Gateway Solutions

| Gateway | Best For |
|---|---|
| AWS API Gateway | Serverless / AWS-native stacks; managed, scales automatically |
| Kong | High-performance, extensible via plugins, self-hosted or cloud |
| Nginx + Lua | Ultra-high performance, fine-grained control |
| Spring Cloud Gateway | Java/Spring Boot microservices |
| Traefik | Kubernetes-native, auto-discovers services via labels |
| Apigee (Google) | Enterprise API management |

## Production Architecture

```
Internet
    |
  [WAF / DDoS Protection]
    |
  [API Gateway]  -- Auth, Rate Limit, Routing, Logging, SSL
    |
  [Service Mesh / Internal Load Balancer]
    |
  +------------------+------------------+
  |                  |                  |
User Service    Order Service    Payment Service
(3 instances)   (2 instances)    (2 instances)
```

## Interview Questions

**Q1: What is the difference between an API Gateway and a Load Balancer?**

A load balancer distributes traffic across multiple instances of the same service — it doesn't know about different services or business logic. An API Gateway routes requests to different services based on the URL path, and adds cross-cutting concerns like auth, rate limiting, and logging. A gateway typically includes or sits in front of a load balancer.

**Q2: What is SSL termination at the gateway?**

The gateway decrypts HTTPS from the client and forwards plain HTTP to backend services on the internal private network. This means only the gateway needs an SSL certificate and manages TLS; services behind it don't. It reduces TLS overhead across all services and centralizes certificate management.

**Q3: How does authentication work at the API Gateway layer?**

The gateway validates the JWT token (or API key) on every request. On success, it extracts the user ID and roles and forwards them as trusted internal headers (e.g. `X-User-Id`, `X-User-Role`) to downstream services. Downstream services trust these headers without re-validating the token — they're on a private network and only receive requests through the gateway.

**Q4: What is a BFF pattern?**

Backend for Frontend — a dedicated API gateway instance (or layer) per client type (web, mobile, IoT). Each BFF is optimized for its client: the mobile BFF returns fewer fields and smaller payloads; the web BFF may aggregate more data. It avoids building a single gateway that tries to serve every client's needs and ends up serving none well.

**Q5: What happens if the API Gateway goes down?**

The entire system is unreachable — the gateway is a single point of failure. In production, run multiple gateway instances behind a load balancer, in multiple availability zones. Use health checks to automatically route away from unhealthy instances. The gateway itself must be highly available since everything depends on it.

**Q6: Can you aggregate multiple service responses in the API Gateway?**

Yes — the aggregation pattern. The gateway makes parallel calls to multiple services, waits for all responses, merges them, and returns a single response to the client. This reduces the number of network round trips for the client. Spring Cloud Gateway supports this via custom filters; Kong has plugins for it; custom gateways implement it in a fan-out handler.
