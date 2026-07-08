# Module 3 — Microservice Architecture

> **Microservices Masterclass** | Level: Beginner–Intermediate | Track: Node.js Backend Engineering
> Prerequisite: Module 1 — What Are Microservices? | Module 2 — Monolith vs Microservices
> Next Module: Module 4 — Domain-Driven Design (DDD) Basics

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Learning Objectives](#2-learning-objectives)
3. [Problem Statement](#3-problem-statement)
4. [Why This Concept Exists](#4-why-this-concept-exists)
5. [Historical Background](#5-historical-background)
6. [Real-World Analogy](#6-real-world-analogy)
7. [Technical Definition](#7-technical-definition)
8. [Core Terminology](#8-core-terminology)
9. [Internal Working](#9-internal-working)
10. [Step-by-Step Request Flow](#10-step-by-step-request-flow)
11. [Architecture Overview](#11-architecture-overview)
12. [ASCII Diagrams](#12-ascii-diagrams)
13. [Mermaid Flowcharts](#13-mermaid-flowcharts)
14. [Mermaid Sequence Diagrams](#14-mermaid-sequence-diagrams)
15. [Component Diagrams](#15-component-diagrams)
16. [Deployment Diagrams](#16-deployment-diagrams)
17. [Database Interaction](#17-database-interaction)
18. [Failure Scenarios](#18-failure-scenarios)
19. [Scalability Discussion](#19-scalability-discussion)
20. [High Availability Considerations](#20-high-availability-considerations)
21. [CAP Theorem Implications](#21-cap-theorem-implications)
22. [Node.js Implementation](#22-nodejs-implementation)
23. [Express.js Examples](#23-expressjs-examples)
24. [Docker Examples](#24-docker-examples)
25. [Kafka/Redis Integration](#25-kafkaredis-integration)
26. [Error Handling](#26-error-handling)
27. [Logging & Monitoring](#27-logging--monitoring)
28. [Security Considerations](#28-security-considerations)
29. [Performance Optimization](#29-performance-optimization)
30. [Production Best Practices](#30-production-best-practices)
31. [Anti-Patterns and Common Mistakes](#31-anti-patterns-and-common-mistakes)
32. [Debugging Tips](#32-debugging-tips)
33. [Interview Questions](#33-interview-questions)
34. [Scenario-Based Questions](#34-scenario-based-questions)
35. [Hands-on Exercises](#35-hands-on-exercises)
36. [Mini Project](#36-mini-project)
37. [Advanced Project](#37-advanced-project)
38. [Summary](#38-summary)
39. [Revision Notes](#39-revision-notes)
40. [One-Page Cheat Sheet](#40-one-page-cheat-sheet)

---

## 1. Introduction

Modules 1 and 2 gave you the *why* — why microservices exist, and when they're the right (or wrong) choice. This module gives you the *what*: the actual building blocks that make up a real microservices system.

If Module 1 was "what is a microservice" and Module 2 was "monolith vs microservices," think of Module 3 as **the parts list**. Just like you can't build a car without understanding the engine, transmission, chassis, and electronics as distinct systems that work together, you can't design microservices without understanding services, containers, service registries, API gateways, event brokers, databases, caches, and load balancers as distinct components with specific jobs.

By the end of this module, you'll be able to look at *any* microservices architecture diagram — Netflix's, Uber's, or your own company's — and correctly identify every component and explain what role it plays.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Name and define every core building block of a microservices architecture.
- Explain the specific responsibility of each component (service, registry, gateway, broker, cache, load balancer).
- Draw a complete microservices architecture diagram from scratch, correctly placing each component.
- Explain how these components interact during a real request.
- Understand containerization's role in making this architecture operationally feasible.
- Identify which components are mandatory vs optional depending on system scale.

---

## 3. Problem Statement

A junior engineer is told: "We're building this as microservices." They create five Express apps in five folders, each with its own `package.json`. They deploy each one to a separate server. Now:

- Clients don't know which URL/port to call for which feature — should the frontend call `service-a.company.com:4001` for auth, `:4002` for orders, `:4003` for payments? This is unmanageable and leaks internal architecture to clients.
- When Order Service needs to call Payment Service, it hardcodes `http://192.168.1.15:4003` — but that IP changes every time Payment Service is redeployed or scaled.
- When Order Service traffic spikes, there's no way to automatically add more Order Service instances and route traffic to them.
- Slow, heavy background work (sending emails, generating reports) blocks the main request-response cycle because there's no message broker to hand it off to.
- There's no single place to enforce authentication, so every single service re-implements JWT validation — and someone eventually gets it wrong.

This module explains the components that solve exactly these problems: **API Gateway**, **Service Registry/Discovery**, **Load Balancer**, **Message Broker**, **Cache**, and **Container/Orchestrator** — the essential architecture around the services themselves.

---

## 4. Why This Concept Exists

Simply "splitting code into multiple services" is not a microservices architecture — it's just several small monoliths that don't know how to find or protect themselves. The supporting components in this module exist because **distributing your application creates new problems that a monolith never had**, and each component is a purpose-built solution to one of those new problems:

| New Problem Introduced by Distribution | Component That Solves It |
|---|---|
| Clients need one place to call, not N services | API Gateway |
| Services need to find each other at runtime (IPs change) | Service Registry & Discovery |
| Traffic needs to be spread across multiple instances | Load Balancer |
| Slow/non-critical work shouldn't block the request | Message Broker / Event Bus |
| Repeated cross-service reads are expensive | Cache (e.g., Redis) |
| Services need to be portable & consistently deployed | Containers (Docker) |
| Many containers need to be scheduled, healed, scaled | Orchestrator (Kubernetes) |

---

## 5. Historical Background

- **Pre-2013**: Companies ran services on manually provisioned VMs. Service discovery was often just a hardcoded config file or DNS entries manually updated by ops teams — brittle and slow to change.
- **2013**: Docker was released, standardizing how applications are packaged with their dependencies into portable, consistent containers — solving the "works on my machine" problem and making it dramatically easier to run many small services consistently.
- **2013–2014**: Netflix open-sourced parts of its internal platform (Eureka for service discovery, Zuul as an API Gateway, Hystrix for circuit breaking, Ribbon for client-side load balancing) — collectively known as "Netflix OSS" — which became reference implementations the whole industry studied.
- **2014**: Kubernetes was announced by Google, based on their internal Borg system, providing built-in service discovery, load balancing, and self-healing for containerized workloads — eventually becoming the de facto standard orchestrator.
- **Mid-2010s onward**: API Gateways (Kong, Amazon API Gateway, later cloud-native ingress controllers) and message brokers (Kafka, originally built at LinkedIn and open-sourced in 2011) became standard components in virtually every serious microservices stack.

> **Interview tip:** If asked to name real-world implementations of these components, you can mention: Kong/AWS API Gateway (gateway), Kubernetes/Consul/Eureka (discovery), Kafka/RabbitMQ (brokers), Redis (cache), Kubernetes/Docker Swarm (orchestration).

---

## 6. Real-World Analogy

**Analogy: An Airport**

- **API Gateway = The main terminal entrance.** Every passenger (client request) enters through one place. Security (authentication) happens here, and signage (routing) directs you to the right gate (service) — you never wander the tarmac looking for your specific airline's hangar.
- **Service Registry = The airport's live flight information board.** It knows, in real time, which gate each flight is currently at — because gates change. Services similarly register their current location (host/port) so others can find them even as instances come and go.
- **Load Balancer = Multiple check-in counters for the same airline.** If one line gets long, you're directed to another counter offering the same service, so no single counter gets overwhelmed.
- **Message Broker = The airport's baggage handling system.** You drop your bag at check-in (publish an event) and walk away — you don't personally carry your bag to the plane (synchronous call). The system handles getting it there asynchronously, and you don't wait around for confirmation before proceeding.
- **Cache = The fast-track security line.** For frequent flyers (repeated resource requests) there's a faster path that avoids redoing the full process every time.
- **Containers = Standardized shipping containers for cargo.** Whether the cargo (application) is delicate electronics or bulk goods, once it's packed into the standard container, any crane, ship, or truck (any server/host) can move it the same way.

---

## 7. Technical Definition

> **Microservices Architecture** refers to the full set of building blocks — beyond just the services themselves — required to make a distributed system function reliably and securely as a whole: an **API Gateway** for unified entry, **Service Discovery** for dynamic location resolution, a **Load Balancer** for traffic distribution, a **Message Broker** for asynchronous communication, a **Cache** for reducing redundant cross-service reads, and a **Container Orchestrator** for deployment, scaling, and self-healing.

Each component has a narrow, well-defined responsibility (Single Responsibility Principle, applied at the infrastructure level, not just the code level).

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Service** | An independently deployable unit implementing one business capability |
| **Container** | A lightweight, portable package containing a service's code, runtime, and dependencies |
| **Service Registry** | A directory tracking live service instances and their network locations |
| **Service Discovery** | The mechanism services use to look up other services' current locations |
| **API Gateway** | The single entry point for external clients; handles routing, auth, rate limiting |
| **Load Balancer** | Distributes incoming traffic across multiple instances of a service |
| **Message Broker / Event Bus** | Middleware enabling asynchronous, decoupled communication (e.g., Kafka, RabbitMQ) |
| **Cache** | An in-memory data store (e.g., Redis) that reduces repeated expensive reads |
| **Reverse Proxy** | A server that forwards client requests to backend services (often part of the Gateway) |
| **Container Orchestrator** | A system (e.g., Kubernetes) that schedules, scales, and heals containers |
| **Sidecar** | A helper container deployed alongside a service container (e.g., for logging, proxying, or service mesh functionality) |
| **Service Mesh** | Infrastructure layer (e.g., Istio, Linkerd) handling service-to-service communication concerns (retries, mTLS, observability) transparently, often via sidecars |

---

## 9. Internal Working

Here's how these components work together, conceptually, end to end:

1. Every service is packaged into a **container image** (Docker) containing its code and dependencies.
2. The **orchestrator** (Kubernetes) schedules multiple **instances (pods)** of each service across a cluster of machines.
3. Each running instance **registers itself** (automatically, via the orchestrator's built-in service discovery, or manually via a tool like Consul/Eureka) so its current location is known.
4. External requests arrive at the **API Gateway**, which:
   - Terminates TLS.
   - Authenticates the request (validates a JWT, for example).
   - Applies rate limiting.
   - Looks up (via service discovery) which service should handle this route.
   - Forwards the request to a **Load Balancer**, which picks one healthy instance among many.
5. That service instance processes the request, possibly:
   - Calling other services synchronously (again through discovery + load balancing).
   - Reading/writing to its own private database.
   - Reading from a **cache** (Redis) before hitting the database, when appropriate.
   - Publishing an event to the **message broker** (Kafka) for asynchronous downstream processing.
6. The response flows back through the Gateway to the client.
7. Meanwhile, other services **consume** the published event independently and asynchronously, decoupled from the original request's timeline.

---

## 10. Step-by-Step Request Flow

**Scenario: "View Order Details" — a request that touches nearly every component.**

```
Step 1:  Client sends GET /orders/456 to api.company.com
Step 2:  Request hits the Load Balancer in front of the API Gateway cluster
Step 3:  Load Balancer forwards to one healthy API Gateway instance
Step 4:  API Gateway validates the JWT (authentication)
Step 5:  API Gateway consults Service Discovery: "Where is Order Service?"
Step 6:  Service Discovery returns a list of healthy Order Service instances
Step 7:  Gateway's internal Load Balancer picks one instance (round-robin/least-connections)
Step 8:  Request forwarded to that Order Service instance
Step 9:  Order Service checks Redis cache for order 456 — cache miss
Step 10: Order Service queries its own Order DB for order 456
Step 11: Order Service needs the customer's name — calls User Service
Step 12: Order Service's call to User Service ALSO goes through
         Service Discovery + Load Balancing (same pattern, one level deeper)
Step 13: Order Service caches the combined result in Redis (TTL: 60s)
Step 14: Order Service returns the response to the Gateway
Step 15: Gateway returns the response to the Client
```

Every hop in Steps 5–7 and 11–12 exists *only* because we distributed the system — in a monolith, this entire flow would be a single in-process function call chain.

---

## 11. Architecture Overview

```
                              Internet
                                 │
                                 ▼
                          DNS / CDN Layer
                                 │
                                 ▼
                       ┌─────────────────┐
                       │  Load Balancer   │
                       └────────┬────────┘
                                │
                    ┌───────────┴───────────┐
                    ▼                       ▼
             API Gateway Instance    API Gateway Instance
                    │                       │
                    └───────────┬───────────┘
                                │
                     (Service Discovery lookup)
                                │
        ┌───────────┬──────────┼──────────┬───────────┐
        ▼           ▼          ▼          ▼           ▼
     User Svc    Order Svc  Payment Svc  Product Svc  Notify Svc
     (2 pods)    (5 pods)   (3 pods)     (4 pods)     (1 pod)
        │           │          │          │           │
        ▼           ▼          ▼          ▼           ▼
     User DB     Order DB   Payment DB  Product DB   (stateless)
                    │
                    ▼
              ┌───────────┐        ┌───────┐
              │Kafka Broker│◀──────▶│ Redis │
              └───────────┘        └───────┘
```

---

## 12. ASCII Diagrams

### 12.1 The Full Component Stack

```
┌───────────────────────────────────────────────────────┐
│                       API Gateway                       │
│      Auth | Rate Limiting | Routing | Aggregation        │
└───────────────────────────────────────────────────────┘
                          │
┌───────────────────────────────────────────────────────┐
│                    Service Discovery                     │
│         "Where is Order Service right now?"               │
└───────────────────────────────────────────────────────┘
                          │
┌───────────────────────────────────────────────────────┐
│                     Load Balancer                        │
│      Distributes across N healthy service instances       │
└───────────────────────────────────────────────────────┘
                          │
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Service  │  │ Service  │  │ Service  │   <- Instances (containers/pods)
│ Instance │  │ Instance │  │ Instance │
└──────────┘  └──────────┘  └──────────┘
      │              │              │
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Private  │  │  Cache   │  │  Message  │
│ Database │  │ (Redis)  │  │  Broker   │
└──────────┘  └──────────┘  └──────────┘
```

### 12.2 Service Discovery Pattern (Registration + Lookup)

```
   Order Service Instance starts up
              │
              ▼
   Registers itself: "I am order-service,
   located at 10.0.1.5:4002, status: healthy"
              │
              ▼
        Service Registry
     (e.g., Kubernetes DNS, Consul, Eureka)
              ▲
              │
   "Where is order-service?" ── Payment Service asks
              │
              ▼
   Registry returns: [10.0.1.5:4002, 10.0.1.9:4002]
```

### 12.3 Sync vs Async Communication Paths

```
SYNCHRONOUS (needs immediate answer):

  Order Service ──HTTP GET──▶ Product Service ──▶ returns stock immediately


ASYNCHRONOUS (fire-and-forget, eventually consistent):

  Order Service ──publish──▶ Kafka Topic "order-events"
                                    │
                     ┌──────────────┼──────────────┐
                     ▼              ▼              ▼
              Inventory Svc   Notification Svc   Analytics Svc
              (consumes when ready, independently)
```

---

## 13. Mermaid Flowcharts

### 13.1 Component Responsibility Map

```mermaid
flowchart TD
    Client[Client] --> LB[Load Balancer]
    LB --> GW[API Gateway]
    GW -->|Authenticate| Auth[Auth Check]
    GW -->|Lookup location| SD[Service Discovery]
    SD --> GW
    GW -->|Route + Balance| SVC[Target Service Instance]
    SVC -->|Read/Write| DB[(Private Database)]
    SVC -->|Fast reads| Cache[(Redis Cache)]
    SVC -->|Publish event| MB[Message Broker]
    MB --> Consumers[Downstream Consumer Services]
```

### 13.2 Container to Production Flow

```mermaid
flowchart LR
    A[Source Code] --> B[Docker Build]
    B --> C[Container Image]
    C --> D[Push to Container Registry]
    D --> E[Orchestrator Pulls Image]
    E --> F[Schedules Pods Across Cluster]
    F --> G[Registers with Service Discovery]
    G --> H[Load Balancer Routes Traffic]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Full Component Interaction for One Request

```mermaid
sequenceDiagram
    participant C as Client
    participant LB as Load Balancer
    participant GW as API Gateway
    participant SD as Service Discovery
    participant O as Order Service
    participant R as Redis Cache
    participant D as Order DB
    participant U as User Service
    participant K as Kafka

    C->>LB: GET /orders/456
    LB->>GW: Forward request
    GW->>GW: Validate JWT
    GW->>SD: Where is Order Service?
    SD-->>GW: [instance list]
    GW->>O: Forward request (load-balanced pick)
    O->>R: Check cache for order:456
    R-->>O: Cache miss
    O->>D: Query order 456
    D-->>O: Order data
    O->>U: Get customer name (via discovery + LB)
    U-->>O: Customer data
    O->>R: Store combined result (TTL 60s)
    O->>K: Publish "OrderViewed" analytics event
    O-->>GW: Response
    GW-->>C: 200 OK
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                     API Gateway Layer                      │
│  Kong / AWS API Gateway / NGINX / Custom Express Gateway   │
└─────────────────────────────────────────────────────────┘
                │
┌─────────────────────────────────────────────────────────┐
│              Service Discovery & Registry                  │
│      Kubernetes DNS / Consul / Eureka / etcd                │
└─────────────────────────────────────────────────────────┘
                │
┌───────────────┬───────────────┬───────────────┬─────────┐
│  User Service  │ Order Service │Payment Service│  ...    │
└───────────────┴───────────────┴───────────────┴─────────┘
                │
┌─────────────────────────────────────────────────────────┐
│         Message Broker (Kafka) + Cache (Redis)              │
└─────────────────────────────────────────────────────────┘
                │
┌─────────────────────────────────────────────────────────┐
│    Container Orchestration (Kubernetes / Docker Swarm)      │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
                     Kubernetes Cluster
┌─────────────────────────────────────────────────────────┐
│  Namespace: production                                     │
│                                                             │
│  Ingress Controller (acts as API Gateway entry)             │
│         │                                                  │
│  ┌──────▼──────┐    ┌─────────────┐    ┌─────────────┐    │
│  │ order-svc   │    │ payment-svc │    │ user-svc    │    │
│  │ Deployment  │    │ Deployment  │    │ Deployment  │    │
│  │ (5 replicas)│    │ (3 replicas)│    │ (2 replicas)│    │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    │
│         │                  │                  │            │
│  ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐    │
│  │ order-svc   │    │ payment-svc │    │ user-svc    │    │
│  │ Service     │    │ Service     │    │ Service     │    │
│  │ (ClusterIP) │    │ (ClusterIP) │    │ (ClusterIP) │    │
│  └─────────────┘    └─────────────┘    └─────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Kafka StatefulSet   |   Redis Deployment             │  │
│  └─────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

Kubernetes's built-in **Service** object provides both service discovery (a stable DNS name like `order-svc.production.svc.cluster.local`) and load balancing (across all healthy pods behind it) — meaning in Kubernetes, you often get two of these architectural components "for free."

---

## 17. Database Interaction

Components interact with data differently depending on their role:

```
Service Instances  ──▶  Their OWN private database (writes + reads)

Cache (Redis)       ──▶  NOT a source of truth; a fast, disposable copy
                         of frequently-read data, always rebuildable
                         from the database or from events

Message Broker      ──▶  Does NOT store business data long-term;
(Kafka)                  stores a durable, ordered LOG of events for
                         a configured retention period, enabling
                         replay and multiple independent consumers
```

A critical mental model: **the cache can vanish and be rebuilt; the broker's topic can be replayed; but the service's own database is the only true source of truth for its data.**

---

## 18. Failure Scenarios

| Component Down | Immediate Impact | Mitigation |
|---|---|---|
| API Gateway instance | Traffic reroutes to healthy Gateway instances | Run Gateway as multiple replicas behind its own load balancer |
| Service Discovery | Services may use stale cached location data temporarily | Client-side caching of discovery results with short TTL; discovery systems (Kubernetes, Consul) are themselves highly available by design |
| Load Balancer | Traffic can't reach that pool of instances | Run redundant load balancers; cloud LBs are typically managed HA services |
| Message Broker (Kafka) down | New events can't be published; async workflows pause | Producers should buffer/retry; broker itself is typically deployed as a replicated cluster for HA |
| Cache (Redis) down | Increased latency and DB load (cache misses for everything) | Design services to function correctly (just slower) without cache — never make cache a hard dependency |
| One service instance | Reduced capacity for that service | Orchestrator restarts it automatically; load balancer routes around it meanwhile |

```
Redis goes down:

  Order Service tries cache.get() → connection refused
        │
        ▼
  Falls back to querying the database directly
        │
        ▼
  Slower, but still CORRECT and AVAILABLE
  (This is why cache must always be treated as optional, never mandatory)
```

---

## 19. Scalability Discussion

Each component scales differently:

- **Services**: Scale horizontally by adding more container instances/pods — the orchestrator handles this via autoscaling rules (e.g., based on CPU or custom metrics like queue depth).
- **API Gateway**: Scales horizontally like any stateless service; it holds no persistent state itself.
- **Service Discovery**: Usually backed by a distributed, replicated store (etcd for Kubernetes, Raft-based consensus for Consul) that scales and remains consistent across the cluster.
- **Load Balancer**: Cloud load balancers scale automatically; self-hosted ones (NGINX, HAProxy) need their own redundancy plan.
- **Message Broker (Kafka)**: Scales by adding partitions (for parallelism) and brokers (for capacity) — covered in depth in the Kafka Masterclass.
- **Cache (Redis)**: Scales via clustering/sharding (Redis Cluster) or read replicas for read-heavy workloads.

---

## 20. High Availability Considerations

- No single component in this stack should be a single point of failure — every layer (Gateway, Discovery, Load Balancer, Broker, Cache, Services) should run as **multiple replicas**, ideally spread across availability zones.
- **Health checks** are the backbone of HA: the orchestrator and load balancer must be able to detect an unhealthy instance and stop routing to it (and eventually replace it) automatically.
- **Graceful degradation**: design services to keep functioning (perhaps with reduced features) even if the cache or broker is temporarily unavailable, rather than failing outright.

---

## 21. CAP Theorem Implications

- **Service Discovery systems** (etcd, Consul) are themselves distributed systems and make their own CAP trade-offs — etcd, for example, favors **Consistency** over Availability during a partition (it's built on the Raft consensus algorithm), which is why Kubernetes control-plane operations can pause during certain network issues rather than serve stale data.
- **Message Brokers** like Kafka favor **Availability and Partition tolerance**, offering configurable consistency guarantees (e.g., `acks=all` for stronger durability guarantees at the cost of latency) — a topic covered in depth in the Kafka Masterclass.
- **Caches** (Redis) are explicitly **not** the consistency source of truth — using a cache is an intentional choice to favor availability/speed over perfect consistency, accepting that cached data may be briefly stale.

---

## 22. Node.js Implementation

Let's build a minimal, working **API Gateway** in Express that performs authentication, service discovery (via environment-based config for simplicity), and request forwarding — the three core Gateway responsibilities.

**Folder structure:**
```
api-gateway/
├── src/
│   ├── config/
│   │   └── services.js       <-- simple static "service discovery"
│   ├── middleware/
│   │   ├── auth.js
│   │   └── rateLimiter.js
│   ├── proxy/
│   │   └── proxyRequest.js
│   └── app.js
├── Dockerfile
└── package.json
```

**`src/config/services.js`**
```javascript
// In production, this would be replaced by real service discovery
// (Kubernetes DNS, Consul, etc.) instead of a static map.
export const serviceRegistry = {
  users: process.env.USER_SERVICE_URL || "http://user-service:4001",
  orders: process.env.ORDER_SERVICE_URL || "http://order-service:4002",
  payments: process.env.PAYMENT_SERVICE_URL || "http://payment-service:4003",
};
```

---

## 23. Express.js Examples

**`src/middleware/auth.js`**
```javascript
import jwt from "jsonwebtoken";

// Authentication happens ONCE, here, at the Gateway —
// downstream services trust the Gateway and don't re-implement this.
export function authenticate(req, res, next) {
  const authHeader = req.headers.authorization;
  if (!authHeader) {
    return res.status(401).json({ error: "Missing authorization header" });
  }

  const token = authHeader.split(" ")[1];
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded; // attach identity for downstream use
    next();
  } catch (err) {
    return res.status(401).json({ error: "Invalid or expired token" });
  }
}
```

**`src/middleware/rateLimiter.js`**
```javascript
import rateLimit from "express-rate-limit";

// Protects downstream services from being overwhelmed by a single client
export const rateLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 100,             // 100 requests per minute per IP
  message: { error: "Too many requests, please try again later." },
});
```

**`src/proxy/proxyRequest.js`**
```javascript
import axios from "axios";

// Forwards the incoming request to the resolved backend service,
// propagating identity and a trace ID for observability.
export async function proxyRequest(targetBaseUrl, req, res) {
  try {
    const response = await axios({
      method: req.method,
      url: `${targetBaseUrl}${req.originalUrl.replace(/^\/api\/[^/]+/, "")}`,
      data: req.body,
      headers: {
        "x-user-id": req.user?.id,
        "x-trace-id": req.traceId,
      },
      timeout: 3000,
    });
    res.status(response.status).json(response.data);
  } catch (err) {
    if (err.response) {
      return res.status(err.response.status).json(err.response.data);
    }
    res.status(503).json({ error: "Upstream service unavailable" });
  }
}
```

**`src/app.js`**
```javascript
import express from "express";
import { authenticate } from "./middleware/auth.js";
import { rateLimiter } from "./middleware/rateLimiter.js";
import { proxyRequest } from "./proxy/proxyRequest.js";
import { serviceRegistry } from "./config/services.js";

const app = express();
app.use(express.json());
app.use(rateLimiter);

// Attach a trace ID to every request for cross-service log correlation
app.use((req, res, next) => {
  req.traceId = req.headers["x-trace-id"] || crypto.randomUUID();
  next();
});

// Public route — no auth required
app.get("/health", (req, res) => res.json({ status: "ok" }));

// All routes below require authentication
app.use(authenticate);

app.use("/api/users", (req, res) => proxyRequest(serviceRegistry.users, req, res));
app.use("/api/orders", (req, res) => proxyRequest(serviceRegistry.orders, req, res));
app.use("/api/payments", (req, res) => proxyRequest(serviceRegistry.payments, req, res));

app.listen(3000, () => console.log("API Gateway running on port 3000"));
```

---

## 24. Docker Examples

**`Dockerfile`** (API Gateway):
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
USER node
CMD ["node", "src/app.js"]
```

**`docker-compose.yml`** (full stack, showing every component together):
```yaml
version: "3.9"
services:
  api-gateway:
    build: ./api-gateway
    ports: ["3000:3000"]
    environment:
      - JWT_SECRET=supersecret
      - USER_SERVICE_URL=http://user-service:4001
      - ORDER_SERVICE_URL=http://order-service:4002
    depends_on: [user-service, order-service]

  user-service:
    build: ./user-service
    environment:
      - DATABASE_URL=postgresql://user:pass@user-db:5432/users
    depends_on: [user-db]

  order-service:
    build: ./order-service
    environment:
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - REDIS_URL=redis://cache:6379
      - KAFKA_BROKER=kafka:9092
    depends_on: [order-db, cache, kafka]

  user-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=users]

  order-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=orders]

  cache:
    image: redis:7-alpine

  kafka:
    image: bitnami/kafka:latest
    ports: ["9092:9092"]
```

Notice this single file demonstrates nearly every component from this module: an API Gateway, multiple services, databases-per-service, a cache, and a message broker — all wired together.

---

## 25. Kafka/Redis Integration

**Redis as a cache-aside pattern (Order Service):**
```javascript
import { createClient } from "redis";

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

export async function getOrder(orderId) {
  const cacheKey = `order:${orderId}`;
  const cached = await redis.get(cacheKey);
  if (cached) return JSON.parse(cached);

  const order = await db.query("SELECT * FROM orders WHERE id = $1", [orderId]);
  await redis.set(cacheKey, JSON.stringify(order), { EX: 60 }); // 60s TTL
  return order;
}
```

**Kafka as the asynchronous backbone (Order Service publishing, Analytics Service consuming):**
```javascript
// Publisher (Order Service)
await producer.send({
  topic: "order-events",
  messages: [{ key: order.id, value: JSON.stringify({ type: "OrderPlaced", payload: order }) }],
});
```
```javascript
// Consumer (Analytics Service) — runs independently, doesn't block the order flow
await consumer.run({
  eachMessage: async ({ message }) => {
    const event = JSON.parse(message.value.toString());
    await recordAnalyticsEvent(event); // slow work, safely decoupled
  },
});
```

---

## 26. Error Handling

Every network hop introduced by these components needs explicit failure handling:

```javascript
// Gateway -> Service call: timeout + graceful upstream error mapping
try {
  const response = await axios.get(url, { timeout: 3000 });
  return response.data;
} catch (err) {
  if (err.code === "ECONNABORTED") {
    return res.status(504).json({ error: "Upstream service timed out" });
  }
  if (!err.response) {
    return res.status(503).json({ error: "Upstream service unreachable" });
  }
  return res.status(err.response.status).json(err.response.data);
}
```

For the **cache**, always treat failures as non-fatal:
```javascript
async function safeCacheGet(key) {
  try {
    return await redis.get(key);
  } catch (err) {
    logger.warn({ err }, "Cache unavailable, falling back to DB");
    return null; // triggers a normal DB read instead of crashing
  }
}
```

---

## 27. Logging & Monitoring

- The **API Gateway** is the ideal place to generate a **trace ID** for every incoming request, since it's the single entry point.
- Every downstream service should **log the trace ID** on every log line so a single request's full journey can be reconstructed across services.
- **Service Discovery health** and **Load Balancer distribution metrics** should be monitored (e.g., via Prometheus) to catch uneven load or discovery failures early.
- **Kafka consumer lag** (how far behind consumers are from the latest published messages) is a critical metric — growing lag indicates a struggling downstream consumer.

```javascript
// Structured log line including trace ID, at every service
logger.info({ traceId: req.traceId, route: req.path }, "Handled request");
```

---

## 28. Security Considerations

- **Terminate authentication at the Gateway** — don't expose backend services directly to the internet; only the Gateway (and Load Balancer in front of it) should be publicly reachable.
- Use **mTLS or service tokens** for service-to-service calls behind the Gateway, so a compromised internal network can't be trivially exploited.
- **Rate limit at the Gateway** to protect all downstream services uniformly, rather than each service implementing its own inconsistent rate limiting.
- Keep the **Service Registry** internal-only — exposing your architecture's internal topology publicly is an information leak that aids attackers.
- Rotate secrets (JWT signing keys, DB credentials, Kafka credentials) regularly, and never hardcode them in source.

---

## 29. Performance Optimization

- Use **connection pooling and HTTP keep-alive** for Gateway-to-service calls to avoid the overhead of establishing a new TCP connection per request.
- **Cache aggressively** for read-heavy, infrequently-changing data — but always with a sensible TTL and a safe fallback to the source of truth.
- Prefer **asynchronous (event-based) communication** wherever the caller doesn't need an immediate answer, shortening the synchronous critical path.
- Co-locate services and their caches/brokers in the same data center/region to minimize network latency.
- Monitor and tune **Load Balancer algorithms** (round-robin vs least-connections vs latency-based) based on your actual traffic patterns.

---

## 30. Production Best Practices

- Treat the **API Gateway configuration** (routes, rate limits, auth rules) as code, version-controlled and reviewed like any other change.
- Use your orchestrator's **built-in service discovery and load balancing** (e.g., Kubernetes Services) rather than reinventing it, unless you have a specific reason not to.
- Always define **health check endpoints** (`/health`, `/ready`) for every service so the orchestrator can manage lifecycle correctly.
- Set **resource limits and autoscaling rules** per service based on its actual observed load profile, not a one-size-fits-all default.
- Document which components are **hard dependencies** (must be up for the system to function) vs **soft dependencies** (cache, some async consumers) that can degrade gracefully.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **No API Gateway** (clients call services directly) | Leaks internal architecture, duplicates auth logic everywhere, no central place for rate limiting |
| **Hardcoded service URLs/IPs** | Breaks the moment an instance restarts or scales; defeats the purpose of dynamic discovery |
| **Treating cache as a source of truth** | Data loss risk; caches can and do get evicted or restarted |
| **Synchronous chains for non-critical work** | Slows down the critical path unnecessarily; should be asynchronous via the broker |
| **Single instance of Gateway/Broker/Cache** (no redundancy) | Creates a single point of failure for the entire system |
| **Gateway doing too much business logic** | Turns the Gateway into a hidden monolith; it should route and enforce cross-cutting concerns, not implement domain logic |

```
Anti-pattern: Client calling services directly (no Gateway)

  Mobile App ──▶ user-service:4001
  Mobile App ──▶ order-service:4002
  Mobile App ──▶ payment-service:4003

  Problem: client must know internal ports/hosts, no central auth,
  no way to add rate limiting or a new cross-cutting concern
  without changing every client
```

---

## 32. Debugging Tips

- When a request fails, check components **in request order**: Load Balancer logs → Gateway logs → Service Discovery resolution → target Service logs → its dependencies (DB, cache, broker).
- Use the **trace ID** to filter logs across every component involved in a single request.
- For discovery issues, verify the registry directly (e.g., `kubectl get endpoints <service-name>`) to confirm expected instances are registered and healthy.
- For cache-related bugs, always test the "cache is empty/down" path explicitly — many bugs only appear on cache miss or cache failure.
- For broker-related issues, check **consumer lag** first — a stuck or crashed consumer is the most common cause of "events aren't being processed."

---

## 33. Interview Questions

### Easy
1. What is the role of an API Gateway in a microservices architecture?
2. What problem does Service Discovery solve?
3. Why is a Load Balancer necessary even if you already have an API Gateway?
4. What's the difference between a cache and a database in this architecture?
5. Why are containers commonly used to package microservices?

### Medium
6. Explain the difference between client-side and server-side service discovery.
7. Why should authentication happen at the Gateway rather than in each individual service?
8. What happens to a request if the Message Broker is temporarily down? How should producers handle this?
9. Why must a cache always be treated as "optional" from a correctness standpoint?
10. What role does a Container Orchestrator play beyond just "running containers"?

### Hard
11. Design the failure-handling strategy for an API Gateway when a downstream service times out versus when it's completely unreachable.
12. How does Kubernetes provide both service discovery and load balancing "for free," and what are its limitations compared to a dedicated service mesh?
13. Explain how you'd design a system where the API Gateway itself must be highly available with zero single point of failure.
14. When would you introduce a Service Mesh (e.g., Istio) on top of Kubernetes, and what problems does it solve beyond basic Kubernetes Services?
15. Explain the trade-offs of putting response aggregation logic (calling multiple services and combining results) in the API Gateway versus a dedicated Backend-for-Frontend (BFF) service.

---

## 34. Scenario-Based Questions

1. Your API Gateway is a single Node.js process with no redundancy, and it just crashed, taking down the entire platform. What immediate and long-term fixes would you propose?
2. A new engineer hardcodes `http://10.0.1.5:4002` for Order Service in the Payment Service's code, "since it's simpler." What issues will this cause, and what should they do instead?
3. Your team notices that adding Redis caching didn't reduce database load as expected — investigation shows a very short TTL and a cache key that includes a rapidly-changing timestamp. Diagnose and fix.
4. Consumer lag on your "order-events" Kafka topic has been steadily growing for the last hour. Walk through your debugging steps.
5. Leadership wants to add rate limiting per API key, per feature, and audit logging for all traffic — where should you implement this, and why?

---

## 35. Hands-on Exercises

1. Draw the full component stack (Section 15) for a "Ride Sharing" platform, labeling each component with its concrete real-world technology choice (e.g., Kong, Kafka, Redis, Kubernetes).
2. Implement a minimal Express-based API Gateway that authenticates requests and proxies to two mock backend services.
3. Write a Redis cache-aside function with a safe fallback if Redis is unreachable, similar to Section 26.
4. Design a load-balancing strategy (round-robin vs least-connections vs latency-based) for a service with highly variable per-request processing time, and justify your choice.
5. List every component in this module that could become a single point of failure if run as only one instance, and describe how you'd make each highly available.

---

## 36. Mini Project

**Build: A Minimal Gateway + Two Services + Discovery via Environment Config**

1. Build `user-service` and `order-service` as separate Express apps, each with its own SQLite/PostgreSQL database.
2. Build a simple `api-gateway` (as shown in Section 23) that authenticates requests via JWT and proxies to both services using a static service registry config.
3. Add a `/health` endpoint to every service and have the Gateway check it before forwarding (basic client-side health awareness).
4. Containerize all three with Docker and wire them together with `docker-compose`.

---

## 37. Advanced Project

**Build: Full Component Stack with Kafka, Redis, and Kubernetes-style Discovery**

1. Extend the Mini Project into 4 services: `user-service`, `order-service`, `payment-service`, `notification-service`.
2. Add Redis caching in `order-service` for order reads (cache-aside pattern, with safe fallback).
3. Add Kafka: `order-service` publishes `OrderPlaced`; `notification-service` and a new `analytics-service` both consume it independently.
4. Deploy the entire stack locally using `docker-compose`, then write equivalent Kubernetes manifests (Deployments + Services) for at least two of the services, demonstrating how Kubernetes Services provide discovery and load balancing natively.
5. Simulate failure: stop Redis and confirm `order-service` still functions correctly (just slower). Stop Kafka and confirm `order-service` doesn't crash when publishing fails (implement a retry/buffer strategy).

---

## 38. Summary

- A microservices architecture is more than "many small services" — it requires supporting components: API Gateway, Service Discovery, Load Balancer, Message Broker, Cache, and Container Orchestrator.
- Each component solves a specific problem introduced by distributing your application, mirroring problems a monolith never had (finding services, balancing load, decoupling slow work, avoiding repeated expensive reads).
- Containers (Docker) and orchestrators (Kubernetes) make running many small services operationally feasible; Kubernetes provides built-in service discovery and load balancing.
- Caches and message brokers must always be treated as **non-authoritative and optional** — the service's own database remains the single source of truth.
- Every component in this stack should be deployed redundantly; none should be a single point of failure in a production system.

---

## 39. Revision Notes

- API Gateway: single entry point — auth, routing, rate limiting.
- Service Discovery: how services find each other's current location dynamically.
- Load Balancer: distributes traffic across healthy instances.
- Message Broker: asynchronous, decoupled communication (Kafka, RabbitMQ).
- Cache: fast, disposable, non-authoritative — always have a fallback.
- Containers + Orchestrator: package and run many services consistently and resiliently (Docker + Kubernetes).
- Kubernetes gives you discovery + load balancing "for free" via its Service object.

---

## 40. One-Page Cheat Sheet

```
API GATEWAY:        single entry point | auth | routing | rate limiting
SERVICE DISCOVERY:   dynamic lookup of service instance locations
LOAD BALANCER:       spreads traffic across healthy instances
MESSAGE BROKER:      async, decoupled, durable event log (Kafka/RabbitMQ)
CACHE:               fast, disposable, NEVER the source of truth (Redis)
CONTAINER:           portable packaging of code + runtime + deps (Docker)
ORCHESTRATOR:        schedules/scales/heals containers (Kubernetes)

GOLDEN RULES:
  - Clients never call services directly — always through the Gateway
  - Never hardcode service locations — always use discovery
  - Cache and broker failures must degrade gracefully, never crash the service
  - Every component runs as MULTIPLE replicas — no single point of failure
  - Kubernetes Services = discovery + load balancing, often built-in
```

---

**Suggested Next Module:** Module 4 — Domain-Driven Design (DDD) Basics (learn how to correctly define service boundaries using bounded contexts, aggregates, and ubiquitous language)
