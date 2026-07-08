# Module 1 — What Are Microservices?

> **Microservices Masterclass** | Level: Beginner | Track: Node.js Backend Engineering
> Prerequisite: Basic Node.js, Express.js, and REST API knowledge
> Next Module: Module 2 — Monolith vs Microservices

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

Every backend engineer eventually hits a wall. You build an app. It works. Users grow. Features pile up. The codebase becomes a jungle. Deployments take forever. One small bug in the "notifications" code brings down "checkout." Sound familiar?

That wall has a name: **the limits of monolithic architecture**. And the industry's answer to that wall is **microservices**.

This module is the foundation of the entire Microservices Masterclass. Everything you learn later — API gateways, service discovery, Kafka-based events, Kubernetes deployments, distributed transactions — sits on top of the ideas introduced here. If you understand this module deeply, every later module will feel like a natural extension rather than a new topic.

We will not just define microservices. We will build the reasoning that leads to microservices, so that in an interview you can defend *why* a company would choose this architecture, not just recite the definition.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain, in plain English, what a microservice is and is not.
- Describe the historical and business pressures that made microservices necessary.
- Identify the core building blocks of a microservices architecture (service, database, gateway, registry, broker).
- Draw the architecture of a simple microservices system from memory.
- Explain trade-offs: what you gain and what you give up compared to a monolith.
- Recognize microservices anti-patterns before you build them.
- Answer common interview questions about microservices with confidence, including trade-off and scenario-based questions.

---

## 3. Problem Statement

Imagine you are the lone backend engineer for an e-commerce startup. You wrote one Node.js + Express application containing:

- Authentication
- User profiles
- Product catalog
- Orders
- Payments
- Notifications

This single application is deployed as **one process**, backed by **one database**. For the first year, this is perfect — it's simple, fast to build, and easy to reason about.

Then the company grows:

- Traffic to the **product catalog** spikes 50x during a sale, but **payments** traffic stays flat. You can't scale just the catalog — you must scale the *entire* application, wasting resources.
- Ten engineers now work on the same codebase. Two people editing the `orders` module accidentally break the `notifications` module because everything shares the same process and dependencies.
- A memory leak in the "recommendation engine" crashes the entire application — including checkout — because they run in the same process.
- Every deploy requires the *whole* application to be rebuilt, tested, and redeployed, even for a one-line change in the email template.
- You want to rewrite the search feature in a language better suited for it, but you can't — everything is bundled together.

This is the **problem statement** microservices exist to solve: **as systems and teams grow, a single, tightly-coupled codebase becomes a bottleneck for scaling, reliability, and team velocity.**

---

## 4. Why This Concept Exists

Microservices did not appear because someone thought splitting code into folders called "services" was fashionable. They emerged as a *response* to real constraints that monoliths hit at scale:

| Constraint at Scale | Monolith Problem | Microservices Response |
|---|---|---|
| Traffic is uneven across features | Must scale everything together | Scale only the hot service |
| Many teams, one codebase | Merge conflicts, coordination overhead | Each team owns a service independently |
| One bug can crash everything | Shared process = shared failure | Isolated processes = isolated failure |
| Slow deploys | Whole app redeployed for any change | Deploy only the changed service |
| Technology lock-in | One language/framework for everything | Each service can pick the best-fit tech |
| Long-term maintainability | Codebase becomes unmanageable "big ball of mud" | Clear boundaries per business capability |

The core idea: **decompose a large system along business capability boundaries so that each piece can be built, deployed, scaled, and owned independently.**

---

## 5. Historical Background

Understanding *how we got here* helps you speak intelligently about microservices in interviews.

- **Early 2000s** — Most systems were monolithic. Service-Oriented Architecture (SOA) was introduced to enable reuse across large enterprises, typically using heavyweight protocols like SOAP and an Enterprise Service Bus (ESB). SOA solved some coupling issues but introduced heavy governance and centralized bottlenecks.
- **2011** — The term "microservices" was popularized in workshops among software architects (notably discussed at a workshop near Venice), describing an architectural style distinct from SOA: lightweight, independently deployable services communicating over simple protocols like HTTP/REST.
- **2014** — Martin Fowler and James Lewis published an influential article defining and popularizing the microservices architectural style for the broader industry.
- **Mid 2010s onward** — Netflix, Amazon, Uber, and Spotify publicly shared how they broke up monoliths into hundreds/thousands of microservices to handle massive scale and large distributed engineering teams. Docker (2013) and Kubernetes (2014) provided the container and orchestration tooling that made running hundreds of small services operationally feasible.
- **Today** — Microservices are a default consideration (though not always the right choice) for medium-to-large scale systems, especially those with multiple teams.

> **Interview tip:** Many interviewers ask "who invented microservices?" — there's no single inventor. It emerged organically from real-world scaling problems and was later named and formalized by the software architecture community.

---

## 6. Real-World Analogy

**Analogy: A Restaurant Kitchen**

**Monolith = A single chef doing everything.** One chef takes orders, cooks appetizers, mains, desserts, washes dishes, and manages billing — all alone, in one small kitchen. It works fine for a 10-seat café. But scale to a 200-seat restaurant, and this chef becomes the bottleneck. If they get sick, the entire restaurant stops.

**Microservices = A professional kitchen brigade.** There's a grill station, a sauté station, a pastry station, a dishwasher, and a host managing the front. Each station:
- Has its own equipment (its own "database").
- Can be staffed independently (scaled independently).
- Can be replaced or upgraded without shutting down the whole kitchen.
- Follows a shared ticket system to coordinate (this is like an API Gateway / message broker).

If the pastry station is overwhelmed with dessert orders on a busy night, you add more pastry chefs — you don't need to add more grill chefs. That's independent scaling. If the dishwasher machine breaks, the kitchen keeps cooking — that's fault isolation.

---

## 7. Technical Definition

> **Microservices** is an architectural style that structures an application as a collection of small, autonomous services, each built around a specific **business capability**, independently deployable, independently scalable, owning its own data, and communicating with other services through well-defined, lightweight mechanisms — typically synchronous APIs (REST/gRPC) or asynchronous messaging (Kafka/RabbitMQ).

Key properties packed into this definition:

1. **Small and focused** — each service does one business capability well (e.g., "Order Service" only handles orders).
2. **Autonomous** — a team can build, test, and deploy it without coordinating a release with every other team.
3. **Independently deployable** — you can ship a change to the Payment Service without redeploying the User Service.
4. **Owns its own data** — no shared database; each service has a private data store it alone can write to.
5. **Communicates over the network** — services talk via HTTP, gRPC, or asynchronous events, not via in-process function calls or a shared database.
6. **Decentralized governance** — teams can choose their own tech stack, as long as they respect the communication contract (API/schema).

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Service** | A single, independently deployable unit implementing one business capability |
| **Service Instance** | One running copy of a service (you typically run many instances for scale) |
| **Bounded Context** | The explicit boundary within which a domain model is valid (from Domain-Driven Design) |
| **API Gateway** | Single entry point that routes client requests to the correct backend service |
| **Service Registry** | A directory that tracks which service instances are alive and where they are (host/port) |
| **Service Discovery** | The mechanism a service uses to find other services at runtime |
| **Message Broker / Event Bus** | Middleware (e.g., Kafka, RabbitMQ) enabling asynchronous, decoupled communication |
| **Database per Service** | Pattern where each service owns a private database no other service can access directly |
| **Contract** | The agreed-upon shape of an API or event (its schema) between producer and consumer services |
| **Container** | A lightweight, portable OS-level virtualization unit used to package and run a service (e.g., Docker) |
| **Orchestrator** | A system (e.g., Kubernetes) that manages deployment, scaling, and healing of containers |

---

## 9. Internal Working

To understand *how* a microservices system actually operates, walk through what happens conceptually when it's running:

1. Each service is packaged (usually as a Docker container) with everything it needs to run: code, runtime, dependencies.
2. Multiple **instances** of each service run concurrently, often across multiple machines, for scale and fault tolerance.
3. A **load balancer** or the orchestrator's built-in networking distributes incoming requests across these instances.
4. Services register themselves (or are auto-registered by the orchestrator) so others can find them — this is **service discovery**.
5. An **API Gateway** sits at the edge, exposing one unified interface to external clients, and internally routes to the right service.
6. Services communicate via:
   - **Synchronous calls** (REST/gRPC) for request-response interactions that need an immediate answer.
   - **Asynchronous events** (Kafka/RabbitMQ) for fire-and-forget or eventually-consistent workflows.
7. Each service reads/writes only to **its own database**. If Service A needs data owned by Service B, it must ask Service B (via API or by consuming events Service B published) — never by directly querying Service B's database.
8. Failures are expected and handled defensively: retries, timeouts, circuit breakers, and fallbacks prevent one failing service from cascading into a system-wide outage.

---

## 10. Step-by-Step Request Flow

Let's trace a real request: **"Place an Order"** on an e-commerce platform.

```
Step 1: Client (mobile app) sends POST /orders request
Step 2: Request hits the Load Balancer
Step 3: Load Balancer forwards to an API Gateway instance
Step 4: API Gateway authenticates the request (validates JWT)
Step 5: API Gateway routes the request to the Order Service
Step 6: Order Service validates the request payload
Step 7: Order Service calls Product Service (REST) to confirm stock
Step 8: Order Service calls Payment Service (REST) to charge the customer
Step 9: Order Service writes a new Order row to its own database
Step 10: Order Service publishes an "OrderPlaced" event to Kafka
Step 11: Inventory Service consumes "OrderPlaced" and decrements stock
Step 12: Notification Service consumes "OrderPlaced" and sends confirmation email
Step 13: API Gateway returns response to the Client
```

Notice: only Steps 1–9 are on the **synchronous critical path** — the customer waits for these. Steps 11–12 happen **asynchronously**, after the customer already got their confirmation. This separation is central to why microservices scale well: slow, non-critical work is decoupled from the fast, critical path.

---

## 11. Architecture Overview

```
                          Client Applications
                        (Web, Mobile, Partner API)
                                  │
                                  ▼
                           Load Balancer
                                  │
                                  ▼
                            API Gateway
                (Auth, Routing, Rate Limiting, Aggregation)
                                  │
        ┌───────────┬────────────┼────────────┬───────────┐
        ▼           ▼            ▼            ▼           ▼
     User        Product       Order       Payment    Notification
    Service      Service      Service      Service       Service
        │           │            │            │           │
        ▼           ▼            ▼            ▼           ▼
   PostgreSQL    MongoDB    PostgreSQL    PostgreSQL     Redis
                                  │
                                  ▼
                          Kafka Event Bus
                  (OrderPlaced, PaymentFailed, etc.)
```

---

## 12. ASCII Diagrams

### 12.1 Monolith vs Microservices (side-by-side)

```
        MONOLITH                          MICROSERVICES

  +--------------------+          +------+  +------+  +------+
  |     Application    |          | User |  |Order |  |Pay   |
  |--------------------|          |Svc   |  |Svc   |  |Svc   |
  | Auth               |          +------+  +------+  +------+
  | Users              |             |         |         |
  | Products           |          +------+  +------+  +------+
  | Orders             |          |UserDB|  |OrderDB| |PayDB |
  | Payments           |          +------+  +------+  +------+
  | Notifications      |
  +--------------------+
            │
            ▼
        One Database
```

### 12.2 Service Anatomy (what's inside ONE microservice)

```
+-----------------------------------+
|          Order Service            |
|-----------------------------------|
|  API Layer (Express Routes)       |
|  Business Logic (Domain Rules)    |
|  Data Access Layer (Prisma/ORM)   |
|  Event Publisher (KafkaJS)        |
|  Config & Secrets                 |
|  Health Check Endpoint            |
+-----------------------------------+
                 │
                 ▼
         Private Database
        (only this service
         can read/write here)
```

### 12.3 Independent Scaling

```
Normal Traffic:
  Product Service  [■■]        (2 instances)
  Order Service    [■■]        (2 instances)

Sale Event (only Product traffic spikes):
  Product Service  [■■■■■■■■]  (8 instances)
  Order Service    [■■]        (still 2 instances — no waste)
```

---

## 13. Mermaid Flowcharts

### 13.1 High-Level System Flow

```mermaid
flowchart TD
    A[Client App] --> B[Load Balancer]
    B --> C[API Gateway]
    C --> D[User Service]
    C --> E[Product Service]
    C --> F[Order Service]
    C --> G[Payment Service]
    F --> H[(Order DB)]
    D --> I[(User DB)]
    E --> J[(Product DB)]
    G --> K[(Payment DB)]
    F --> L[Kafka Event Bus]
    L --> M[Inventory Service]
    L --> N[Notification Service]
```

### 13.2 Decision Flow: Should You Split a Monolith?

```mermaid
flowchart TD
    Start[Is your app struggling?] --> Q1{Are teams blocking each other on deploys?}
    Q1 -- Yes --> Q2{Do features have very different scaling needs?}
    Q1 -- No --> Stay1[Consider staying monolith]
    Q2 -- Yes --> Q3{Can you define clear business boundaries?}
    Q2 -- No --> Stay2[Modular monolith may be enough]
    Q3 -- Yes --> Go[Adopt Microservices]
    Q3 -- No --> Redesign[Do Domain-Driven Design first]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Place Order — Synchronous + Asynchronous Mix

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant O as Order Service
    participant P as Product Service
    participant Pay as Payment Service
    participant K as Kafka
    participant N as Notification Service

    C->>G: POST /orders
    G->>O: Forward request (with auth context)
    O->>P: Check stock (REST)
    P-->>O: Stock available
    O->>Pay: Charge customer (REST)
    Pay-->>O: Payment success
    O->>O: Save order to Order DB
    O->>K: Publish "OrderPlaced" event
    O-->>G: 201 Created
    G-->>C: Order confirmation
    K-->>N: Consume "OrderPlaced"
    N->>N: Send confirmation email
```

---

## 15. Component Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                      API Gateway Layer                     │
│  - Authentication      - Rate Limiting                     │
│  - Request Routing     - Response Aggregation               │
└───────────────────────────────────────────────────────────┘
        │              │              │              │
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│  User Svc │  │Product Svc│  │ Order Svc │  │Payment Svc│
│  (Node.js)│  │ (Node.js) │  │ (Node.js) │  │ (Node.js) │
└───────────┘  └───────────┘  └───────────┘  └───────────┘
        │              │              │              │
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│PostgreSQL │  │ MongoDB   │  │PostgreSQL │  │PostgreSQL │
└───────────┘  └───────────┘  └───────────┘  └───────────┘
```

---

## 16. Deployment Diagrams

```
                       Kubernetes Cluster
┌───────────────────────────────────────────────────────────┐
│  Namespace: production                                     │
│                                                             │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────────┐ │
│  │ order-svc pod │   │ order-svc pod │   │ order-svc pod │ │
│  │  (replica 1)  │   │  (replica 2)  │   │  (replica 3)  │ │
│  └───────────────┘   └───────────────┘   └───────────────┘ │
│           │                   │                   │        │
│           └─────────────┬─────┴─────────┬─────────┘        │
│                         ▼                                  │
│                  Service (ClusterIP)                       │
│                         │                                   │
│                         ▼                                  │
│                    Ingress / Gateway                        │
└───────────────────────────────────────────────────────────┘
```

Each service is packaged as a Docker image, deployed as a Kubernetes **Deployment** (managing multiple **Pods**/replicas), exposed internally via a **Service** object, and reachable externally through an **Ingress** or gateway.

---

## 17. Database Interaction

The golden rule of microservices: **Database per Service.**

```
Order Service   ──writes/reads──▶  Order DB   (private, exclusive access)
Product Service ──writes/reads──▶  Product DB (private, exclusive access)

Order Service   ──✗ NEVER directly query──▶  Product DB
```

If the Order Service needs product details, it must:
- Call the Product Service's API (`GET /products/:id`), **or**
- Maintain a local, denormalized read-cache built from consuming Product events (`ProductUpdated`) published to Kafka.

This avoids the classic monolith trap of a shared database becoming a hidden, uncontrolled integration point that no one can safely change.

---

## 18. Failure Scenarios

| Scenario | What Happens | Mitigation |
|---|---|---|
| Payment Service is down | Order Service's call to charge fails | Circuit breaker + retry + graceful error to client |
| Product Service is slow | Order Service waits, risking timeout cascades | Set aggressive timeouts + fallback (e.g., cached stock data) |
| Kafka broker is unreachable | "OrderPlaced" event isn't published | Outbox pattern: write event to DB transactionally, publish later via a relay |
| Network partition between services | Requests fail intermittently | Retries with backoff, idempotent APIs |
| One service instance crashes | Some requests fail | Orchestrator (Kubernetes) auto-restarts; load balancer routes around it |

```
Order Service ──(times out)──▶ Payment Service (down)
      │
      ▼
Circuit Breaker OPENS
      │
      ▼
Fallback: return "Payment pending, we'll notify you" instead of hanging
```

---

## 19. Scalability Discussion

Because each service is independently deployable, you scale exactly what needs scaling:

```
Black Friday Traffic Pattern:

Product Browsing  → 20x traffic spike → scale Product Service to 20 instances
Order Placement    → 3x traffic spike  → scale Order Service to 3 instances
User Profile Edits → flat traffic      → keep User Service at 2 instances
```

This is **horizontal scaling at the service level** — something a monolith cannot do, since scaling a monolith means spinning up more copies of the *entire* application, wasting compute on parts that don't need it.

---

## 20. High Availability Considerations

- Run **multiple instances** of every service across multiple availability zones/machines so a single machine failure doesn't cause downtime.
- Use **health checks** so orchestrators can detect and replace unhealthy instances automatically.
- Design for **graceful degradation**: if Recommendation Service is down, show the product page without recommendations rather than failing the whole page.
- Avoid **single points of failure** — even the API Gateway should run as multiple instances behind a load balancer.

---

## 21. CAP Theorem Implications

The CAP theorem states a distributed system can only guarantee two of three properties during a network partition: **Consistency**, **Availability**, **Partition Tolerance**. Since network partitions are a reality in distributed systems, you're really choosing between **Consistency** and **Availability** when a partition happens.

In a microservices world with a database-per-service:
- Data is **eventually consistent** across services (e.g., a "ProductUpdated" event may take a moment to reach the Order Service's local cache).
- Most systems favor **Availability** for user-facing operations (better to show slightly stale product info than show an error page).
- Critical operations (like payment) may favor **Consistency**, accepting some latency or unavailability to guarantee correctness.

```
Partition happens between Order Service and Product Service:

Choose Consistency: Order Service refuses to place order until it confirms
                     fresh stock data (safer, but less available)

Choose Availability: Order Service uses last-known stock data from cache
                     (available, but may oversell in rare cases)
```

---

## 22. Node.js Implementation

Below is a minimal, production-shaped **Product Service** skeleton in Node.js (ES Modules) to make these concepts concrete.

**Folder structure:**

```
product-service/
├── src/
│   ├── config/
│   │   └── env.js
│   ├── routes/
│   │   └── product.routes.js
│   ├── controllers/
│   │   └── product.controller.js
│   ├── services/
│   │   └── product.service.js
│   ├── db/
│   │   └── prisma.js
│   ├── middleware/
│   │   └── errorHandler.js
│   ├── utils/
│   │   └── logger.js
│   └── app.js
├── Dockerfile
├── package.json
└── .env
```

**`src/config/env.js`**

```javascript
// Centralizes all environment variable access so the rest of the
// codebase never calls process.env directly (easier to test & audit).
export const config = {
  port: process.env.PORT || 4000,
  databaseUrl: process.env.DATABASE_URL,
  serviceName: "product-service",
};
```

---

## 23. Express.js Examples

**`src/app.js`**

```javascript
import express from "express";
import productRoutes from "./routes/product.routes.js";
import { errorHandler } from "./middleware/errorHandler.js";
import { logger } from "./utils/logger.js";

const app = express();

app.use(express.json());

// Health check endpoint — required so the orchestrator (Kubernetes)
// knows this instance is alive and ready to receive traffic.
app.get("/health", (req, res) => {
  res.status(200).json({ status: "ok", service: "product-service" });
});

// All product-related routes are mounted under /products
app.use("/products", productRoutes);

// Centralized error handler — ensures unhandled errors return a
// consistent JSON shape instead of crashing the process or leaking stack traces.
app.use(errorHandler);

export default app;
```

**`src/routes/product.routes.js`**

```javascript
import { Router } from "express";
import { getProductById, listProducts } from "../controllers/product.controller.js";

const router = Router();

router.get("/", listProducts);
router.get("/:id", getProductById);

export default router;
```

**`src/controllers/product.controller.js`**

```javascript
import { fetchProductById, fetchAllProducts } from "../services/product.service.js";

// Thin controller: only handles HTTP concerns (parsing, status codes).
// Business logic stays in the service layer.
export async function listProducts(req, res, next) {
  try {
    const products = await fetchAllProducts();
    res.status(200).json(products);
  } catch (err) {
    next(err); // Delegate to centralized error handler
  }
}

export async function getProductById(req, res, next) {
  try {
    const product = await fetchProductById(req.params.id);
    if (!product) {
      return res.status(404).json({ message: "Product not found" });
    }
    res.status(200).json(product);
  } catch (err) {
    next(err);
  }
}
```

**`src/services/product.service.js`**

```javascript
import { prisma } from "../db/prisma.js";

// This service ONLY talks to its own database (product DB).
// It never reaches into another service's database directly.
export async function fetchAllProducts() {
  return prisma.product.findMany();
}

export async function fetchProductById(id) {
  return prisma.product.findUnique({ where: { id } });
}
```

**`src/middleware/errorHandler.js`**

```javascript
import { logger } from "../utils/logger.js";

// Any error thrown/passed via next(err) lands here.
// Keeping this centralized avoids duplicated try/catch noise everywhere.
export function errorHandler(err, req, res, next) {
  logger.error({ message: err.message, stack: err.stack, path: req.path });

  const statusCode = err.statusCode || 500;
  res.status(statusCode).json({
    error: {
      message: statusCode === 500 ? "Internal server error" : err.message,
    },
  });
}
```

---

## 24. Docker Examples

**`Dockerfile`**

```dockerfile
# Use a lightweight, production-appropriate Node.js base image
FROM node:20-alpine

WORKDIR /app

# Copy dependency manifests first to leverage Docker layer caching
COPY package*.json ./
RUN npm ci --omit=dev

# Copy the rest of the application source
COPY . .

# Product Service listens on port 4000
EXPOSE 4000

# Run as a non-root user for security
USER node

CMD ["node", "src/app.js"]
```

**`docker-compose.yml`** (local multi-service dev environment)

```yaml
version: "3.9"
services:
  product-service:
    build: ./product-service
    ports:
      - "4000:4000"
    environment:
      - DATABASE_URL=postgresql://user:pass@product-db:5432/products
    depends_on:
      - product-db

  order-service:
    build: ./order-service
    ports:
      - "4001:4001"
    environment:
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - KAFKA_BROKER=kafka:9092
    depends_on:
      - order-db
      - kafka

  product-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=products
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass

  order-db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=orders
      - POSTGRES_USER=user
      - POSTGRES_PASSWORD=pass

  kafka:
    image: bitnami/kafka:latest
    ports:
      - "9092:9092"
```

This compose file demonstrates the "database per service" pattern directly: `product-db` and `order-db` are **separate containers**, not shared.

---

## 25. Kafka/Redis Integration

This module introduces *why* asynchronous communication matters; deep Kafka mechanics belong to the Kafka Masterclass, but here's the minimal shape relevant to microservices.

**Publishing an event (Order Service, using KafkaJS):**

```javascript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "order-service",
  brokers: [process.env.KAFKA_BROKER],
});

const producer = kafka.producer();

export async function publishOrderPlaced(order) {
  await producer.connect();
  await producer.send({
    topic: "order-events",
    messages: [
      {
        key: order.id,
        value: JSON.stringify({ type: "OrderPlaced", payload: order }),
      },
    ],
  });
}
```

**Consuming the event (Notification Service):**

```javascript
import { Kafka } from "kafkajs";

const kafka = new Kafka({
  clientId: "notification-service",
  brokers: [process.env.KAFKA_BROKER],
});

const consumer = kafka.consumer({ groupId: "notification-group" });

export async function startConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: "order-events", fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      if (event.type === "OrderPlaced") {
        // Send confirmation email — decoupled from the customer's request path
        await sendConfirmationEmail(event.payload);
      }
    },
  });
}
```

**Redis usage example** — caching product data to reduce cross-service calls:

```javascript
import { createClient } from "redis";

const redisClient = createClient({ url: process.env.REDIS_URL });
await redisClient.connect();

export async function getCachedProduct(productId) {
  const cached = await redisClient.get(`product:${productId}`);
  if (cached) return JSON.parse(cached);
  return null;
}

export async function cacheProduct(productId, data) {
  // Cache for 60 seconds — short enough to limit staleness risk
  await redisClient.set(`product:${productId}`, JSON.stringify(data), { EX: 60 });
}
```

---

## 26. Error Handling

In microservices, errors must be handled with the assumption that **the network will fail** and **other services will sometimes be unavailable**.

```javascript
import axios from "axios";

// Basic retry wrapper for calls to other services
export async function callWithRetry(fn, retries = 3, delayMs = 200) {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === retries) throw err;
      await new Promise((resolve) => setTimeout(resolve, delayMs * attempt));
    }
  }
}

// Usage: calling Product Service from Order Service with retry + timeout
export async function checkStock(productId) {
  return callWithRetry(() =>
    axios.get(`${process.env.PRODUCT_SERVICE_URL}/products/${productId}`, {
      timeout: 2000, // fail fast rather than hanging the request
    })
  );
}
```

Key principles:
- **Fail fast** with timeouts instead of hanging indefinitely.
- **Retry** transient failures, but only for idempotent operations.
- **Circuit-break** repeated failures to stop hammering a struggling downstream service.
- **Return meaningful errors** to the client instead of leaking internal stack traces.

---

## 27. Logging & Monitoring

Each service should emit **structured logs** (JSON) tagged with the service name, and correlation IDs so a single request can be traced across services.

```javascript
import pino from "pino";

export const logger = pino({
  base: { service: "product-service" },
  formatters: {
    level(label) {
      return { level: label };
    },
  },
});

// Example usage with a correlation/trace ID propagated from the gateway
logger.info({ traceId: req.headers["x-trace-id"] }, "Fetched product list");
```

In production, logs from all services are aggregated centrally (e.g., ELK stack, Loki), and metrics are collected via Prometheus, visualized in Grafana. Distributed tracing (OpenTelemetry + Jaeger/Zipkin) stitches together the full path of a request across every service it touched — critical because a single "slow request" in microservices could be caused by any one of ten services.

---

## 28. Security Considerations

- **Authenticate once at the edge** (API Gateway validates the JWT), then propagate a trusted identity/context to downstream services — don't make every service re-implement login logic.
- **Service-to-service authentication**: use mutual TLS (mTLS) or short-lived service tokens so services can trust requests from each other, not just from the gateway.
- **Least privilege**: each service's database credentials should only grant access to its own database.
- **Secrets management**: never hardcode credentials; use a secrets manager (Vault, AWS Secrets Manager, Kubernetes Secrets).
- **Network segmentation**: internal services shouldn't be publicly reachable — only the API Gateway should be exposed to the internet.

---

## 29. Performance Optimization

- **Cache aggressively** for data that doesn't change often (product catalog) to avoid unnecessary cross-service calls.
- **Use asynchronous communication** for anything that doesn't need an immediate response, reducing the length of the synchronous critical path.
- **Batch requests** where possible instead of issuing one HTTP call per item (N+1 problem across services is worse than within a single database).
- **Right-size instances**: over-provisioned services waste money; under-provisioned ones bottleneck under load. Use metrics to inform autoscaling thresholds.
- **Connection pooling** for databases and keep-alive HTTP agents for service-to-service calls to avoid connection overhead per request.

---

## 30. Production Best Practices

- Define clear, **versioned API contracts** between services (e.g., `/v1/products`) so you can evolve services independently without breaking consumers.
- Adopt **Database per Service** strictly — resist the temptation to let two services share a database "just this once."
- Automate everything: **CI/CD pipelines** per service so each team can deploy independently and safely.
- Implement **health checks** (`/health`, `/ready`) so orchestrators can manage lifecycle correctly.
- Design for **idempotency** in APIs and event consumers, since retries and duplicate deliveries are a normal part of distributed systems.
- Document service ownership clearly — every service should have a named owning team.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Shared Database** across services | Recreates monolith-style coupling through the back door |
| **Distributed Monolith** (services that must always be deployed together) | You get all the operational complexity of microservices with none of the independence benefits |
| **Chatty Services** (excessive synchronous calls between services for one operation) | High latency, cascading failure risk, tight coupling |
| **No API versioning** | Any change to a service risks breaking every consumer |
| **Nano-services** (splitting things too small, e.g., a separate service per database table) | Excessive operational overhead for little benefit |
| **Ignoring failure handling** | A single slow/failing service can cascade into a full outage |
| **Premature adoption** | Small teams/startups often don't have the scale or org size to justify the operational overhead of microservices |

```
Distributed Monolith (BAD):

Order Service ──always deployed with──▶ Payment Service
      │
      └── They share a library, a schema, and must release together
          = All the network overhead of microservices,
            none of the independence benefits
```

---

## 32. Debugging Tips

- Always propagate a **correlation ID / trace ID** through every request so you can filter logs across services for a single user request.
- Use **distributed tracing** tools to visualize where time is being spent across a multi-service call chain.
- When a request fails, check in order: **Gateway logs → the failing service's logs → its dependency's logs**.
- Reproduce issues locally using `docker-compose` to spin up the minimal set of dependent services.
- Watch for **silent failures** in async event consumers — always log consumption failures and consider a Dead Letter Queue (covered in the Kafka Masterclass).

---

## 33. Interview Questions

### Easy
1. What is a microservice?
2. Name three differences between a monolith and a microservices architecture.
3. What does "Database per Service" mean, and why is it important?
4. What is an API Gateway, and why is it needed in a microservices system?
5. Give a real-world analogy for microservices.

### Medium
6. What are the trade-offs of adopting microservices? When would you *not* recommend them?
7. How do services communicate with each other? Compare synchronous vs asynchronous approaches.
8. What is a "distributed monolith," and how do you avoid creating one?
9. How does independent scaling work in a microservices architecture?
10. Explain the role of service discovery.

### Hard
11. How would you handle a scenario where Service A needs data that Service B owns, without directly querying Service B's database?
12. Explain the CAP theorem implications for a microservices system during a network partition.
13. How do you ensure data consistency across services without distributed transactions (ACID) across databases?
14. Design a failure-handling strategy for a service dependency that becomes slow (not down, just slow).
15. How would you migrate a monolith to microservices incrementally, with minimal risk? (Strangler Fig pattern)

---

## 34. Scenario-Based Questions

1. Your company has 3 backend engineers and a single moderately-trafficked product. A senior engineer proposes splitting into 15 microservices. What do you say?
2. During a flash sale, the Order Service starts timing out because the Payment Service is overwhelmed. What immediate and long-term actions would you take?
3. Two teams both want to "quickly" add a column to a shared database table used by their respective services. How do you respond, and what pattern do you recommend instead?
4. A new engineer proposes that the Notification Service directly reads from the Order Service's database "to keep things simple." How do you explain why this is problematic?
5. You need to evolve the Product API from v1 to v2 with a breaking change, but 5 other services depend on v1. What's your rollout strategy?

---

## 35. Hands-on Exercises

1. Draw the architecture diagram for a "Food Delivery" platform with at least 5 microservices, their databases, and their communication style (sync/async).
2. List 5 business capabilities in a "Library Management System" and define bounded contexts for each.
3. Write the folder structure for a new "Review Service" following the pattern shown in Section 22.
4. Identify which of the anti-patterns in Section 31 apply to a system you've worked on (or a hypothetical one), and describe how you'd fix each.
5. Sketch a sequence diagram for a "Cancel Order" flow that must notify Payment (refund), Inventory (restock), and Notification (email) services.

---

## 36. Mini Project

**Build: A Two-Service System**

Build two independent Node.js/Express microservices:

1. **User Service** — CRUD for users, backed by PostgreSQL (via Prisma).
2. **Order Service** — Create orders that reference a `userId`. Before creating an order, it must call the User Service's REST API to confirm the user exists.

**Requirements:**
- Each service has its own Dockerfile and its own database (via `docker-compose`).
- Order Service must handle the case where User Service is unreachable (return a clear 503 error, don't crash).
- Both services expose a `/health` endpoint.
- Include structured logging with a request correlation ID header (`x-trace-id`).

---

## 37. Advanced Project

**Build: Event-Driven Order Pipeline**

Extend the Mini Project into a 4-service system:

1. **User Service** (PostgreSQL)
2. **Product Service** (MongoDB)
3. **Order Service** (PostgreSQL) — synchronously validates user + product, then publishes an `OrderPlaced` event to Kafka
4. **Notification Service** (stateless) — consumes `OrderPlaced` and logs/simulates sending an email

**Requirements:**
- Use KafkaJS for the event bus.
- Implement retry logic for the synchronous calls from Order Service.
- Implement a simple circuit breaker (you may use the `opossum` npm package) around the Product Service call.
- Add Docker Compose orchestration for all 4 services + Kafka + both databases.
- Bonus: add an API Gateway (using `express-http-proxy` or similar) in front of all services.

---

## 38. Summary

- Microservices decompose an application into small, independently deployable services, each owning a specific business capability and its own data.
- They exist to solve real problems monoliths face at scale: uneven traffic, team coordination bottlenecks, slow deploys, and fragile shared failure domains.
- The architecture trades simplicity for independence — you gain scalability and team autonomy, but you take on distributed systems complexity: network failures, eventual consistency, and operational overhead.
- Communication happens either synchronously (REST/gRPC) for request-response needs, or asynchronously (Kafka/RabbitMQ) for decoupled, eventually-consistent workflows.
- Microservices are not automatically "better" — for small teams and simple products, a well-structured monolith (or modular monolith) is often the smarter choice.

---

## 39. Revision Notes

- Monolith → one deployable unit, one database, tightly coupled.
- Microservices → many deployable units, database-per-service, loosely coupled via network calls.
- Key motivators: independent scaling, independent deployment, team autonomy, fault isolation.
- Core components: Service, API Gateway, Service Registry/Discovery, Message Broker, Database per Service.
- CAP theorem forces a choice between consistency and availability during partitions.
- Common failure modes: cascading failures, chatty services, distributed monolith, shared database.

---

## 40. One-Page Cheat Sheet

```
WHAT:         Small, independently deployable services per business capability
WHY:          Solve scaling, team, deploy, and fault-isolation problems of monoliths
KEY RULE:     Database per Service — never share a DB across services
COMMUNICATION:REST/gRPC (sync) | Kafka/RabbitMQ (async)
ENTRY POINT:  API Gateway (auth, routing, rate limiting)
DISCOVERY:    Service Registry (who is alive, where)
RESILIENCE:   Timeouts + Retries + Circuit Breakers + Fallbacks
CONSISTENCY:  Eventual consistency across services (CAP theorem trade-off)
ANTI-PATTERNS:Shared DB, Distributed Monolith, Chatty Services, Nano-services
WHEN TO SKIP: Small team, simple product, no clear scaling/team pain yet
```

---

**Suggested Next Module:** Module 2 — Monolith vs Microservices (deep dive into migration strategy, comparison tables, and when each architecture wins)
