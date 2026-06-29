# Service Discovery

## What is Service Discovery?

Service Discovery is the mechanism by which microservices automatically find and communicate with each other — without hardcoding IP addresses or hostnames. In a dynamic cloud environment where service instances start, stop, and scale constantly, service discovery provides a live registry of available services.

```
Without Service Discovery:
Order Service calls Payment Service
config: payment.url=http://192.168.1.10:8083

Payment Service restarts on a new IP: 192.168.1.15
Order Service still has old IP -> Connection refused!
Manual config update required -> downtime

With Service Discovery:
Payment Service registers: { name: "payment-service", host: "192.168.1.15", port: 8083 }
Order Service asks registry: "Where is payment-service?"
Registry returns: "192.168.1.15:8083"
Order Service calls the right address -> works automatically
```

## Why Service Discovery?

### 1. Dynamic Environments

In Kubernetes/Docker, containers restart with new IPs constantly. No human can keep up.

### 2. Horizontal Scaling

```
payment-service: 3 instances
  192.168.1.10:8083
  192.168.1.11:8083
  192.168.1.12:8083

Order Service needs to know all 3 to load-balance.
Without discovery: update config manually when instances change.
With discovery: registry always has current list.
```

### 3. Health-Based Routing

Discovery services remove unhealthy instances from the registry automatically — callers never route to a dead instance.

### 4. Zero-Config Deployment

New service instances self-register — no manual DNS updates, no config file changes required.

## Types of Service Discovery

### Client-Side Discovery

The client (caller) queries the service registry directly, gets a list of instances, and applies its own load balancing to choose which one to call.

```
Order Service -> Eureka Registry: "Where are payment-service instances?"
              <- [192.168.1.10:8083, 192.168.1.11:8083, 192.168.1.12:8083]
Order Service -> applies Round Robin -> calls 192.168.1.11:8083
```

**Tools:** Netflix Eureka + Spring Cloud LoadBalancer (Ribbon was deprecated)

**Pros:** client controls load balancing strategy; no proxy hop.
**Cons:** every client needs discovery logic; different tech stacks need separate client libraries.

### Server-Side Discovery

The client calls a load balancer or API gateway, which queries the registry and routes the request. Client is unaware of discovery.

```
Order Service -> Load Balancer / API Gateway: "POST payment-service/charge"
LB -> Service Registry: "Where is payment-service?"
   <- [192.168.1.10:8083, ...]
LB -> picks instance -> forwards request
```

**Tools:** AWS ALB + ECS, Kubernetes Service + kube-proxy, Consul + Nginx/Envoy

**Pros:** client needs no discovery logic; works for any language/tech stack.
**Cons:** extra network hop through the load balancer.

## Key Components

### Service Registry

A database that stores the current location (host, port, metadata) of all service instances. Services register on startup and deregister on shutdown. Health checks periodically verify instances are alive.

### Service Registration

**Self-registration:** Each service registers itself with the registry on startup and deregisters on shutdown.

```java
// Spring Boot with Eureka: auto-registers on startup
// @EnableEurekaClient or spring.cloud.discovery.enabled=true
```

**Third-party registration:** An external system (like a Kubernetes controller) registers services based on infrastructure events.

```yaml
# Kubernetes: Service object handles discovery automatically
# kube-proxy routes traffic to healthy pods; DNS resolves service names
```

## Eureka (Spring Cloud — Client-Side Discovery)

### Eureka Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
```

```yaml
# application.yml (Eureka Server)
server:
  port: 8761

eureka:
  client:
    register-with-eureka: false   # server doesn't register with itself
    fetch-registry: false
  server:
    wait-time-in-ms-when-sync-empty: 0
    eviction-interval-timer-in-ms: 5000  # check for dead instances every 5s
```

Visit `http://localhost:8761` — Eureka dashboard shows registered services.

### Eureka Client (Service Registration)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
# application.yml (Payment Service)
spring:
  application:
    name: payment-service   # this is the service name used in discovery

eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 10  # heartbeat every 10s
    lease-expiration-duration-in-seconds: 30  # remove if no heartbeat for 30s
    instance-id: ${spring.application.name}:${random.value}
    metadata-map:
      version: "1.0"
      environment: "production"
```

The service auto-registers with Eureka on startup and deregisters on shutdown (via shutdown hook).

### Service-to-Service Call with Discovery

```java
// Order Service calling Payment Service by name (not IP)

// Option 1: @LoadBalanced RestTemplate
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced  // Spring Cloud intercepts and resolves service names
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class PaymentClient {

    @Autowired
    private RestTemplate restTemplate;  // @LoadBalanced

    public PaymentResult charge(String orderId, BigDecimal amount) {
        // "payment-service" is resolved via Eureka, load balanced automatically
        return restTemplate.postForObject(
                "http://payment-service/api/payments/charge",
                new ChargeRequest(orderId, amount),
                PaymentResult.class
        );
    }
}

// Option 2: WebClient (reactive, preferred for Spring Boot 3+)
@Configuration
public class WebClientConfig {

    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
}

@Service
public class PaymentClient {

    @Autowired
    private WebClient.Builder webClientBuilder;

    public Mono<PaymentResult> charge(String orderId, BigDecimal amount) {
        return webClientBuilder.build()
                .post()
                .uri("http://payment-service/api/payments/charge")
                .bodyValue(new ChargeRequest(orderId, amount))
                .retrieve()
                .bodyToMono(PaymentResult.class);
    }
}

// Option 3: OpenFeign (declarative REST client)
@FeignClient(name = "payment-service")
public interface PaymentServiceClient {

    @PostMapping("/api/payments/charge")
    PaymentResult charge(@RequestBody ChargeRequest request);

    @GetMapping("/api/payments/{id}")
    PaymentResult getPayment(@PathVariable String id);
}

// Usage
@Service
public class OrderService {

    @Autowired
    private PaymentServiceClient paymentClient;  // Spring creates proxy automatically

    public Order placeOrder(OrderRequest req) {
        Order order = orderRepo.save(new Order(req));
        PaymentResult result = paymentClient.charge(
                new ChargeRequest(order.getId(), order.getTotal()));
        // ...
    }
}
```

## Consul (Multi-Language Service Discovery)

Consul is more feature-rich than Eureka: supports multiple data centers, health checks, key-value store, and works with any tech stack (not just Java).

```yaml
# application.yml with Consul
spring:
  cloud:
    consul:
      host: consul
      port: 8500
      discovery:
        service-name: payment-service
        health-check-path: /actuator/health
        health-check-interval: 10s
        tags:
          - version=1.0
          - env=production
```

Consul health checks:

```yaml
# Consul-registered health check
{
  "ID": "payment-service-1",
  "Name": "payment-service",
  "Address": "192.168.1.10",
  "Port": 8083,
  "Check": {
    "HTTP": "http://192.168.1.10:8083/actuator/health",
    "Interval": "10s",
    "Timeout": "5s",
    "DeregisterCriticalServiceAfter": "30s"  // auto-deregister if failing for 30s
  }
}
```

## Kubernetes Service Discovery

In Kubernetes, service discovery is built-in:

```yaml
# payment-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: payment-service
  template:
    spec:
      containers:
        - name: payment-service
          image: myapp/payment-service:1.0
          ports:
            - containerPort: 8083

---
# Kubernetes Service (DNS-based discovery)
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
  ports:
    - port: 80
      targetPort: 8083
  type: ClusterIP
```

In Kubernetes:
```
order-service calls: http://payment-service/api/payments
kube-proxy resolves: payment-service -> one of the 3 pod IPs
DNS: payment-service.default.svc.cluster.local
```

No Eureka needed in Kubernetes — the platform handles discovery natively.

## Health Checks

Service registries use health checks to remove dead instances:

```java
// Spring Boot Actuator — standard health endpoint
// GET /actuator/health -> { "status": "UP" }
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics
  endpoint:
    health:
      show-details: always

# Custom health indicator
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    @Autowired
    private DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            conn.isValid(1);
            return Health.up()
                    .withDetail("database", "connected")
                    .build();
        } catch (SQLException e) {
            return Health.down()
                    .withDetail("database", "disconnected")
                    .withException(e)
                    .build();
        }
    }
}
```

## Service Discovery vs DNS

| | Service Discovery (Eureka/Consul) | DNS |
|---|---|---|
| Health-based routing | Yes — unhealthy instances removed | No — DNS returns stale records |
| TTL (staleness) | Real-time (seconds) | DNS TTL (minutes to hours) |
| Metadata | Rich (version, env, zone) | None (just IP) |
| Multi-instance | Returns list + LB | Round-robin DNS (limited) |
| Complexity | Higher | Low |
| Best for | Microservices with dynamic scaling | Simple, stable services |

## Interview Questions

**Q1: What is the difference between client-side and server-side service discovery?**

Client-side: the calling service queries the registry directly and load-balances itself (Eureka + Spring Cloud LoadBalancer). Server-side: a load balancer or proxy queries the registry and routes requests transparently — the calling service just calls the load balancer's fixed address (Kubernetes Service, AWS ALB). Client-side gives the caller control over load balancing; server-side is language-agnostic and simpler for clients.

**Q2: How does Eureka handle service instance failures?**

Each service instance sends a heartbeat to Eureka every 30 seconds (configurable). If Eureka doesn't receive a heartbeat within `lease-expiration-duration-in-seconds` (default 90s), it marks the instance as down and removes it from the registry. Callers querying the registry after that will not receive the dead instance's address.

**Q3: Why is Kubernetes service discovery preferred over Eureka in cloud-native deployments?**

Kubernetes has built-in service discovery via its Service resource and DNS. Services are resolved by name (`http://payment-service`) without any client library. Kubernetes health checks (liveness/readiness probes) automatically route traffic away from unhealthy pods. No separate Eureka cluster to maintain, no client dependency on spring-cloud-netflix libraries, and it works for any language.

**Q4: What is the role of health checks in service discovery?**

Health checks ensure the registry only contains healthy, reachable instances. When a service fails (crash, DB down, OOM), its health check starts failing. The registry detects this and removes the instance. Callers don't route to it until health is restored. Without health checks, the registry contains stale entries and callers get connection errors.

**Q5: How does Spring Cloud LoadBalancer use Eureka?**

When a `@LoadBalanced` RestTemplate or WebClient makes a call to `http://service-name/endpoint`, Spring Cloud LoadBalancer intercepts the request, queries Eureka for all healthy instances of `service-name`, applies a load-balancing algorithm (round-robin by default), selects an instance, resolves the host/IP, and sends the actual HTTP request. The application code only uses service names — never raw IPs.
