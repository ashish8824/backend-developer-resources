# Module 2 — Monolith vs Microservices

> **Microservices Masterclass** | Level: Beginner | Track: Node.js Backend Engineering
> Prerequisite: Module 1 — What Are Microservices?
> Next Module: Module 3 — Microservice Architecture

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

Module 1 established *what* microservices are. Before you go further, you need to deeply understand the architecture you're moving *away from*: the **monolith**. Every architecture decision is a trade-off, and you can't argue for microservices in an interview — or in a real design meeting — if you can't fairly represent what a monolith gets right.

This module is a head-to-head comparison. We'll look at monoliths and microservices across every dimension that matters in production: scaling, team structure, deployment, database design, fault tolerance, and cost. We'll also cover something most tutorials skip: the **modular monolith**, a middle ground that is often the *correct* choice for small-to-medium teams.

By the end, you should be able to walk into a system design interview and confidently answer: "Would you use a monolith or microservices for this system, and why?"

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Define a monolith precisely, beyond "one big codebase."
- Compare monoliths and microservices across scaling, deployment, team structure, and data ownership.
- Explain the "modular monolith" as an intermediate architecture and when to choose it.
- Identify the signals that indicate a monolith should be decomposed.
- Describe a realistic, incremental migration strategy (Strangler Fig pattern) from monolith to microservices.
- Avoid the common trap of migrating to microservices for the wrong reasons.

---

## 3. Problem Statement

Teams frequently make one of two costly mistakes:

1. **Premature microservices** — A 3-person startup with a single moderately-used product adopts 12 microservices on day one. They now spend more time managing Kubernetes YAML, service discovery, and inter-service network failures than building product features. Velocity *drops*, not increases.
2. **Overstaying in a monolith** — A 200-engineer company with dozens of teams still ships one giant deployable. Every release requires coordinating with 40 other teams. A single failing test in an unrelated module blocks everyone's deploy. Ownership is unclear, and the codebase has become what's often called a "big ball of mud."

The problem this module solves: **how do you correctly decide which architecture fits your current situation** — not the architecture that's trendy, but the one that matches your team size, traffic patterns, and organizational maturity?

---

## 4. Why This Concept Exists

The monolith-vs-microservices comparison exists because **architecture decisions are context-dependent, not universally "correct."** Both styles were built to solve specific problems:

- **Monoliths** exist because, for a huge number of systems, having everything in one codebase and one deployable unit is genuinely the fastest, simplest, most productive way to build software — especially early on.
- **Microservices** exist because, past a certain scale of traffic, team size, and organizational complexity, a monolith's constraints (shared deploys, shared failure domain, shared scaling) become a *bigger* cost than the operational overhead of running many services.

Understanding **why** each exists prevents the common mistake of treating microservices as strictly "the more advanced/better" option. They are not superior — they are a different set of trade-offs.

---

## 5. Historical Background

- **1990s–2000s**: Nearly all web applications were monoliths — LAMP stacks (Linux, Apache, MySQL, PHP), early Java EE applications, and Ruby on Rails apps were single deployable units. This was simply how software was built; there wasn't yet a well-known alternative for typical web-scale problems.
- **Mid-2000s**: As companies like Amazon and eBay grew massive, their monoliths began hitting real operational limits — some of Amazon's early internal restructuring efforts around service-oriented boundaries (often referenced anecdotally in "two-pizza team" discussions) are cited as early precursors to what became microservices thinking.
- **2011–2014**: Netflix famously migrated from a data-center-based monolith to a highly distributed, cloud-native microservices architecture on AWS after a major outage caused by a single database corruption issue, which pushed them toward isolating failure domains.
- **2015+**: The rise of Docker and Kubernetes made "many small deployable services" operationally tractable at a level that wasn't practical before containerization — previously, running dozens of separately deployed services meant dozens of manually managed servers.
- **Present**: The **modular monolith** has re-emerged as a respected middle-ground pattern, popularized by voices in the software architecture community pushing back against "microservices by default" thinking, especially for startups and small teams.

> **Interview tip:** If asked "Netflix uses microservices, should we?" — a strong answer highlights that Netflix's scale (100M+ users, thousands of engineers) created problems most companies never encounter, and architecture should match your *actual* scale, not an aspirational one.

---

## 6. Real-World Analogy

**Analogy: Building a House vs Building a Neighborhood**

**Monolith = One large house.** All utilities — plumbing, electrical, heating — are part of one connected structure, built and maintained by one contractor. It's fast to build, and if you want to add a room, you can do it relatively easily since everything is already connected and understood as one system. But if the foundation has a problem, the whole house is at risk, and renovating the kitchen might require briefly shutting off the water for the entire house.

**Microservices = A neighborhood of independent houses**, each with its own utilities, run by separate owners. If one house has an electrical fire, it doesn't affect the house next door. Each owner can renovate their house on their own schedule without needing permission from every neighbor. But now you need shared infrastructure — roads, a postal system, zoning rules — to make the neighborhood function as a coherent whole. That coordination layer (the "roads and postal system") is the equivalent of your API Gateway, service discovery, and shared conventions in microservices.

**Modular Monolith = One large house with well-built internal walls and separate utility shutoffs per room.** You get most of the isolation benefits (you can shut off water to just the kitchen) without the overhead of maintaining 10 separate properties.

---

## 7. Technical Definition

> A **monolith** is a software architecture where the entire application — all business capabilities, from authentication to payments to notifications — is built, tested, deployed, and scaled as a **single deployable unit**, typically backed by a **single shared database**.

> **Microservices** architecture, in contrast, decomposes the application into multiple **independently deployable units**, each owning its own data store, communicating over the network.

> A **modular monolith** is a single deployable unit (like a monolith) but internally organized into strict, well-defined modules with clear boundaries and minimal coupling — often a deliberate stepping stone toward microservices, or a permanent, pragmatic end-state for teams that don't need full service independence.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Monolith** | Single deployable unit containing all application logic |
| **Modular Monolith** | Single deployable unit, but internally split into loosely-coupled modules |
| **Big Ball of Mud** | An anti-pattern monolith with no clear internal structure or boundaries |
| **Coupling** | The degree to which one module/service depends on the internals of another |
| **Cohesion** | The degree to which the responsibilities within one module/service are related |
| **Strangler Fig Pattern** | An incremental migration strategy that gradually replaces monolith functionality with services |
| **Shared Database Anti-pattern** | Multiple services reading/writing the same database, recreating monolith-style coupling |
| **Vertical Slice** | A feature implemented end-to-end (UI to DB) within a single bounded context, used as a natural extraction unit |
| **Deployment Coupling** | When unrelated features must be deployed together because they live in the same deployable unit |

---

## 9. Internal Working

**How a monolith works internally:**

1. All modules (Auth, Users, Orders, Payments...) live in one codebase and compile/run as a single process (or a small number of identical replica processes).
2. All modules typically share one database connection pool and one schema (or at least one database instance).
3. A single build pipeline compiles, tests, and packages the entire application together.
4. Scaling means running more identical copies of the *entire* application behind a load balancer — you cannot scale just the "Orders" part.
5. Any code change, however small, goes through the *same* full test suite and deployment pipeline as every other change.

**How the same system works as microservices:**

1. Each module becomes its own codebase, its own deployable, with its own database.
2. Each has its own build/test/deploy pipeline — changes to Payments don't require re-testing Orders.
3. Scaling is per-service: you scale Orders instances without touching Payments instances.
4. Communication that used to be an in-process function call now happens over the network (REST/gRPC/events), introducing latency and the possibility of network failure that didn't exist before.

---

## 10. Step-by-Step Request Flow

**Same feature — "Update user profile" — in both architectures:**

### Monolith Flow
```
Step 1: Client sends PUT /users/123
Step 2: Request hits the single Application process
Step 3: Application's User module handles validation
Step 4: Application writes directly to the shared database (users table)
Step 5: Application returns response
```
No network hop between "modules" — it's all one process, one function call away.

### Microservices Flow
```
Step 1: Client sends PUT /users/123
Step 2: Request hits API Gateway
Step 3: Gateway authenticates and routes to User Service
Step 4: User Service validates and writes to its own User DB
Step 5: User Service publishes a "UserUpdated" event to Kafka
Step 6: Other services (e.g., Order Service, which caches user display name)
        consume "UserUpdated" and update their local read-cache
Step 7: User Service returns response to Gateway
Step 8: Gateway returns response to Client
```
Notice steps 5–6 don't exist in the monolith at all — in a monolith, every module reads the shared `users` table directly and always sees fresh data instantly. In microservices, this convenience is traded for independence, at the cost of needing eventual consistency mechanisms.

---

## 11. Architecture Overview

```
                     MONOLITH                                  MICROSERVICES

              ┌────────────────────┐                    ┌────────────────────┐
              │   Load Balancer     │                    │   Load Balancer     │
              └─────────┬──────────┘                    └─────────┬──────────┘
                        │                                          │
              ┌─────────▼──────────┐                    ┌─────────▼──────────┐
              │   App Instance 1    │                    │    API Gateway      │
              │  Auth|User|Order|   │                    └───┬────┬────┬─────┘
              │  Payment|Notify     │                        │    │    │
              └─────────┬──────────┘                    ┌────▼┐ ┌─▼──┐┌▼───┐
              ┌─────────▼──────────┐                    │User │ │Ord ││Pay │
              │   App Instance 2    │                    │Svc  │ │Svc ││Svc │
              │  (identical copy)   │                    └────┬┘ └─┬──┘└┬───┘
              └─────────┬──────────┘                        │    │    │
                        │                                ┌───▼┐┌──▼┐┌──▼┐
              ┌─────────▼──────────┐                     │DB1 ││DB2││DB3│
              │   Shared Database   │                     └────┘└───┘└───┘
              └────────────────────┘
```

---

## 12. ASCII Diagrams

### 12.1 Scaling Comparison

```
MONOLITH SCALING (all-or-nothing):

  Instance 1: [Auth][User][Order][Payment][Notify]
  Instance 2: [Auth][User][Order][Payment][Notify]
  Instance 3: [Auth][User][Order][Payment][Notify]

  -> To handle more Order traffic, you must replicate
     EVERYTHING again, wasting CPU/memory on Auth,
     User, Payment, Notify capacity you didn't need.


MICROSERVICES SCALING (targeted):

  Order Service:   [■][■][■][■][■][■]   (6 instances - hot path)
  Auth Service:    [■][■]               (2 instances - light load)
  Payment Service: [■][■][■]            (3 instances)
  Notify Service:  [■]                  (1 instance)
```

### 12.2 Deployment Coupling

```
MONOLITH DEPLOY:

  Change: fix typo in email template (Notify module)
      │
      ▼
  Full build → Full test suite → Full deploy
      │
      ▼
  ENTIRE application redeployed (Auth, Orders, Payments included)
  Risk: any bug anywhere can block/break this single release


MICROSERVICES DEPLOY:

  Change: fix typo in email template (Notification Service)
      │
      ▼
  Build only Notification Service → Test only Notification Service
      │
      ▼
  Deploy ONLY Notification Service
  Risk: contained to Notification Service alone
```

### 12.3 The Modular Monolith (middle ground)

```
+---------------------------------------------------+
|              Single Deployable Process              |
|-----------------------------------------------------|
|  +---------+  +---------+  +---------+  +---------+ |
|  |  Auth   |  |  Users  |  | Orders  |  |Payments | |
|  | Module  |  | Module  |  | Module  |  | Module  | |
|  +---------+  +---------+  +---------+  +---------+ |
|      (communicate via internal interfaces only,     |
|       never reach into each other's internal state)  |
+---------------------------------------------------+
                          │
                          ▼
              Shared Database (but separate
              schemas per module, enforced by
              code convention/tooling)
```

---

## 13. Mermaid Flowcharts

### 13.1 Decision Tree: Monolith, Modular Monolith, or Microservices?

```mermaid
flowchart TD
    A[Start: New or growing system] --> B{Team size?}
    B -- "1-8 engineers" --> C[Monolith or Modular Monolith]
    B -- "9-30 engineers" --> D{Clear domain boundaries exist?}
    B -- "30+ engineers, multiple teams" --> E{Deployment coordination is a bottleneck?}
    D -- Yes --> F[Modular Monolith, prepare for extraction]
    D -- No --> G[Invest in Domain-Driven Design first]
    E -- Yes --> H[Adopt Microservices incrementally]
    E -- No --> F
```

### 13.2 Strangler Fig Migration Flow

```mermaid
flowchart LR
    A[Legacy Monolith] --> B[Add API Gateway in front of Monolith]
    B --> C[Extract first service: e.g. Notifications]
    C --> D[Gateway routes /notifications to new service]
    D --> E[Gateway routes everything else to Monolith]
    E --> F[Repeat: extract next module]
    F --> G[Monolith shrinks over time]
    G --> H[Eventually: Monolith retired or minimal]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Same Feature, Monolith vs Microservices

```mermaid
sequenceDiagram
    participant C as Client
    participant M as Monolith (single process)
    participant DB as Shared DB

    C->>M: PUT /users/123 (update profile)
    M->>M: Call UserModule.update() (in-process function call)
    M->>DB: UPDATE users SET ...
    DB-->>M: OK
    M-->>C: 200 OK
```

```mermaid
sequenceDiagram
    participant C as Client
    participant G as API Gateway
    participant U as User Service
    participant UDB as User DB
    participant K as Kafka
    participant O as Order Service (cache consumer)

    C->>G: PUT /users/123
    G->>U: Forward request
    U->>UDB: UPDATE users SET ...
    UDB-->>U: OK
    U->>K: Publish "UserUpdated" event
    U-->>G: 200 OK
    G-->>C: 200 OK
    K-->>O: Consume "UserUpdated" (async, after response already sent)
    O->>O: Update local cached display name
```

---

## 15. Component Diagrams

```
MONOLITH COMPONENTS                    MICROSERVICES COMPONENTS

┌───────────────────────┐              ┌─────────┐ ┌─────────┐ ┌─────────┐
│   Single Application   │              │  User   │ │  Order  │ │ Payment │
│  ┌─────┐ ┌─────┐       │              │ Service │ │ Service │ │ Service │
│  │Auth │ │User │  ...  │              └────┬────┘ └────┬────┘ └────┬────┘
│  └─────┘ └─────┘       │              ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
└───────────┬────────────┘              │ User DB │ │Order DB │ │ Pay DB  │
            │                           └─────────┘ └─────────┘ └─────────┘
    ┌───────▼────────┐                       ▲            ▲           ▲
    │ Shared Database │                       └────────────┴───────────┘
    └────────────────┘                          API Gateway + Service Mesh
```

---

## 16. Deployment Diagrams

```
MONOLITH DEPLOYMENT (single pipeline):

  Git Push → CI Build (whole app) → CI Test (whole app) → 
  Single Artifact → Deploy to N identical servers behind LB


MICROSERVICES DEPLOYMENT (independent pipelines):

  Git Push (order-service repo) → CI Build (order-service only) →
  CI Test (order-service only) → Container Image →
  Deploy to order-service's Kubernetes Deployment only

  Git Push (payment-service repo) → completely separate pipeline →
  Deploy to payment-service's Kubernetes Deployment only
  (unaffected by order-service's release)
```

---

## 17. Database Interaction

This is the single most important structural difference between the two architectures.

```
MONOLITH:

  Auth Module ───┐
  User Module ───┼──▶  ONE Shared Database  (all modules can query all tables)
  Order Module ──┤
  Payment Module ─┘

  Pro: Easy joins across "Orders" and "Users" tables in one SQL query
  Con: Any module can accidentally depend on another module's internal
       table structure, creating hidden coupling


MICROSERVICES:

  User Service ──▶ User DB (private)
  Order Service ─▶ Order DB (private)
  Payment Service ▶ Payment DB (private)

  Pro: Each service can evolve its schema without breaking others
  Con: No cross-service SQL joins — must use API calls or event-driven
       local caches to combine data from multiple services
```

A common migration mistake: teams extract services but leave them pointing at the **same shared database**. This creates a **distributed monolith** — you now pay for network overhead and operational complexity, but still have the tight coupling of a shared schema.

---

## 18. Failure Scenarios

| Scenario | Monolith Behavior | Microservices Behavior |
|---|---|---|
| Bug causes unhandled exception in Notifications code | Can crash the entire process, taking down Orders/Payments too | Only the Notification Service crashes; Orders/Payments unaffected |
| Database connection pool exhausted | Entire application degrades (all modules share the pool) | Only services connected to that specific database degrade |
| One module has a memory leak | Entire process eventually OOMs and restarts, dropping in-flight requests for ALL features | Only that service's instances restart; others keep serving |
| Deploying a bad change | Entire application is at risk (rollback affects everything) | Only the deployed service needs rollback; others remain stable |

```
Monolith Cascading Failure:

  Notification code throws unhandled error
        │
        ▼
  Process crashes
        │
        ▼
  Orders, Payments, Auth — ALL down (same process)


Microservices Isolated Failure:

  Notification Service crashes
        │
        ▼
  Orders, Payments, Auth — UNAFFECTED (separate processes)
```

---

## 19. Scalability Discussion

| Dimension | Monolith | Microservices |
|---|---|---|
| Scaling granularity | Whole application only | Per-service |
| Resource efficiency at scale | Lower (over-provisioning of cold modules) | Higher (scale only what's hot) |
| Scaling simplicity | Very simple (just add more identical instances) | More complex (autoscaling rules per service) |
| Cost at small scale | Lower (one deployment, one set of infra) | Higher (infra overhead per service: health checks, monitoring, etc.) |
| Cost at large scale | Can become higher due to waste from all-or-nothing scaling | Lower per unit of "useful" scaling |

**Key insight for interviews:** Microservices are not "more scalable" in an absolute sense — a monolith can scale extremely far (Stack Overflow ran a monolith serving billions of requests for years on a small number of powerful servers). Microservices provide *more efficient, more granular* scaling, and importantly, better organizational scaling (many teams working independently) — not necessarily more raw request-handling capability.

---

## 20. High Availability Considerations

- **Monolith**: HA is achieved by running multiple identical instances behind a load balancer. It's simple, but a systemic bug affects all instances equally since they're identical.
- **Microservices**: HA requires managing availability per service, but a failure in one service can be isolated, so the system as a whole can remain substantially available even if one capability degrades (e.g., "Recommendations" down, but checkout still works).
- Both require multi-instance deployment, health checks, and (ideally) multi-zone/region deployment for true resilience — architecture style alone doesn't guarantee HA; it must be deliberately designed.

---

## 21. CAP Theorem Implications

- **Monolith with one shared database**: Typically operates comfortably within strong consistency, since there's usually a single database instance (or a primary with read replicas) as the source of truth. CAP trade-offs mostly show up only in the database's own replication behavior, not in the application layer.
- **Microservices with database-per-service**: CAP trade-offs become an *application-level* concern. When Service A needs data owned by Service B and the network between them is partitioned, the application must explicitly choose: fail the request (favor Consistency) or proceed with stale/cached data (favor Availability). This decision, invisible in a monolith, becomes an explicit design choice you must make for every cross-service interaction in microservices.

```
Monolith:                             Microservices:
  App ──▶ One DB                        Order Svc ──▶ Order DB
  (CAP handled inside DB engine)        Order Svc ──(network)──▶ Product Svc ──▶ Product DB
                                        (CAP trade-off now visible to YOU, the developer)
```

---

## 22. Node.js Implementation

To make the comparison concrete, here's the **same feature** — "Get order with user details" — implemented both ways.

**Monolith folder structure:**
```
monolith-app/
├── src/
│   ├── modules/
│   │   ├── users/
│   │   │   ├── user.controller.js
│   │   │   ├── user.service.js
│   │   │   └── user.model.js
│   │   ├── orders/
│   │   │   ├── order.controller.js
│   │   │   ├── order.service.js
│   │   │   └── order.model.js
│   ├── db/
│   │   └── connection.js   <-- ONE shared DB connection
│   └── app.js
```

**Monolith implementation (`orders/order.service.js`):**
```javascript
import { db } from "../../db/connection.js";
import { getUserById } from "../users/user.service.js"; // direct in-process import!

// In a monolith, calling another module is just a function call —
// no network hop, no serialization, no failure handling needed.
export async function getOrderWithUser(orderId) {
  const order = await db.query("SELECT * FROM orders WHERE id = $1", [orderId]);
  const user = await getUserById(order.userId); // in-process call, always available
  return { ...order, user };
}
```

**Microservices folder structure:**
```
order-service/
├── src/
│   ├── clients/
│   │   └── userServiceClient.js   <-- HTTP client, handles network failure
│   ├── services/
│   │   └── order.service.js
│   └── app.js

user-service/   (completely separate repo/deployment)
├── src/
│   └── ...
```

**Microservices implementation (`order-service/src/clients/userServiceClient.js`):**
```javascript
import axios from "axios";

// Unlike the monolith, this is a NETWORK call — it can time out, fail,
// or the User Service could be temporarily unavailable. We must handle that.
export async function getUserById(userId) {
  try {
    const response = await axios.get(
      `${process.env.USER_SERVICE_URL}/users/${userId}`,
      { timeout: 2000 }
    );
    return response.data;
  } catch (err) {
    // Graceful degradation: return a minimal placeholder instead of failing entirely
    return { id: userId, name: "Unknown User", degraded: true };
  }
}
```

**Microservices implementation (`order-service/src/services/order.service.js`):**
```javascript
import { db } from "../db/connection.js"; // Order Service's OWN database
import { getUserById } from "../clients/userServiceClient.js";

export async function getOrderWithUser(orderId) {
  const order = await db.query("SELECT * FROM orders WHERE id = $1", [orderId]);
  const user = await getUserById(order.userId); // network call, may degrade gracefully
  return { ...order, user };
}
```

Notice how the microservices version *must* account for partial failure — the exact complexity trade-off this module has been describing throughout.

---

## 23. Express.js Examples

**Monolith `app.js`** — everything mounted in one Express app:

```javascript
import express from "express";
import userRoutes from "./modules/users/user.routes.js";
import orderRoutes from "./modules/orders/order.routes.js";
import paymentRoutes from "./modules/payments/payment.routes.js";

const app = express();
app.use(express.json());

// All modules mounted on the SAME process, SAME port
app.use("/users", userRoutes);
app.use("/orders", orderRoutes);
app.use("/payments", paymentRoutes);

app.listen(3000, () => console.log("Monolith running on port 3000"));
```

**Microservices — three separate `app.js` files, each its own process/port:**

```javascript
// user-service/src/app.js
import express from "express";
import userRoutes from "./routes/user.routes.js";

const app = express();
app.use(express.json());
app.use("/users", userRoutes);
app.listen(4001, () => console.log("User Service running on port 4001"));
```

```javascript
// order-service/src/app.js
import express from "express";
import orderRoutes from "./routes/order.routes.js";

const app = express();
app.use(express.json());
app.use("/orders", orderRoutes);
app.listen(4002, () => console.log("Order Service running on port 4002"));
```

Each service is a completely independent Node.js process with its own lifecycle, its own crash domain, and its own deploy pipeline.

---

## 24. Docker Examples

**Monolith — one container, one image:**

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 3000
CMD ["node", "src/app.js"]
```

```yaml
# docker-compose.yml (monolith)
version: "3.9"
services:
  monolith:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/appdb
    depends_on:
      - db
  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_DB=appdb
```

**Microservices — multiple containers, multiple images:**

```yaml
# docker-compose.yml (microservices)
version: "3.9"
services:
  user-service:
    build: ./user-service
    ports: ["4001:4001"]
    environment:
      - DATABASE_URL=postgresql://user:pass@user-db:5432/users
    depends_on: [user-db]

  order-service:
    build: ./order-service
    ports: ["4002:4002"]
    environment:
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:4001
    depends_on: [order-db, user-service]

  user-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=users]

  order-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=orders]
```

Notice `USER_SERVICE_URL=http://user-service:4001` — the Order Service reaches the User Service **over the Docker network**, not through an in-process function call. This single line encapsulates the entire architectural shift.

---

## 25. Kafka/Redis Integration

**Monolith**: Kafka/Redis are optional — many monoliths never need them, since in-process function calls and a shared database already provide consistency and communication.

**Microservices**: Kafka (or another broker) often becomes necessary to keep services in sync **without** synchronous coupling. Example: keeping a local read-cache of user names in the Order Service:

```javascript
// order-service: consume UserUpdated events to keep a local cache fresh,
// avoiding a synchronous call to User Service on every single order read
import { Kafka } from "kafkajs";
import { redisClient } from "../db/redis.js";

const kafka = new Kafka({ clientId: "order-service", brokers: [process.env.KAFKA_BROKER] });
const consumer = kafka.consumer({ groupId: "order-service-user-cache" });

export async function startUserCacheConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: "user-events", fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      if (event.type === "UserUpdated") {
        // Cache just the fields Order Service actually needs (denormalized)
        await redisClient.set(
          `user-cache:${event.payload.id}`,
          JSON.stringify({ name: event.payload.name }),
          { EX: 3600 }
        );
      }
    },
  });
}
```

This pattern — **denormalizing just enough data locally via events** — replaces the SQL join a monolith would do naturally in one query.

---

## 26. Error Handling

**Monolith**: Errors between modules are typically just JavaScript exceptions — `try/catch` around an in-process function call is usually sufficient.

```javascript
// Monolith: simple in-process error handling
try {
  const user = await getUserById(order.userId); // in-process, no network
} catch (err) {
  // Only fails if there's an actual application bug, not a network issue
  throw new Error("Could not fetch user data");
}
```

**Microservices**: Must account for network-specific failure modes — timeouts, connection refused, partial responses, service unavailability — none of which exist in a monolith's in-process calls.

```javascript
// Microservices: must handle network-specific failure modes
import axios from "axios";

export async function getUserByIdSafe(userId) {
  try {
    const res = await axios.get(`${process.env.USER_SERVICE_URL}/users/${userId}`, {
      timeout: 2000,
    });
    return res.data;
  } catch (err) {
    if (err.code === "ECONNABORTED") {
      // Timeout — User Service is too slow
      return { id: userId, name: "Unavailable", degraded: true };
    }
    if (err.response?.status === 404) {
      // User genuinely doesn't exist
      throw new Error("User not found");
    }
    // Network error, DNS failure, connection refused, etc.
    return { id: userId, name: "Unavailable", degraded: true };
  }
}
```

---

## 27. Logging & Monitoring

- **Monolith**: One log stream (or a small number of identical instance logs) is usually enough to understand a request's full journey — everything happened in one process.
- **Microservices**: A single user request may touch 5+ services. You **must** implement a correlation/trace ID propagated through every HTTP call and event, or debugging becomes nearly impossible.

```javascript
// Middleware to propagate a trace ID across service boundaries (microservices)
export function traceIdMiddleware(req, res, next) {
  req.traceId = req.headers["x-trace-id"] || crypto.randomUUID();
  res.setHeader("x-trace-id", req.traceId);
  next();
}

// When calling another service, always forward the trace ID
axios.get(url, { headers: { "x-trace-id": req.traceId } });
```

In a monolith, this pattern is rarely needed since the entire request lifecycle is visible within one process's logs.

---

## 28. Security Considerations

| Concern | Monolith | Microservices |
|---|---|---|
| Attack surface | Single perimeter to secure | Multiple internal services — each is a potential internal attack surface |
| Authentication | Handled once, shared session/context in-process | Must be validated at the edge (Gateway) and identity propagated to each service |
| Internal traffic | N/A (all in-process) | Needs securing (mTLS, service tokens) since it now travels over the network |
| Secrets management | One config/secrets store | Secrets must be managed and rotated per service |

---

## 29. Performance Optimization

- **Monolith**: In-process function calls are extremely fast (nanoseconds/microseconds) — no serialization or network latency. Database joins across "modules" are a single efficient SQL query.
- **Microservices**: Every cross-service call incurs network latency (often 1–50ms+ even on a fast internal network) plus serialization overhead (JSON encode/decode). This is why **chatty** microservices architectures (many small calls to fulfill one operation) can be *slower* than an equivalent monolith unless you deliberately batch, cache, or make communication asynchronous.

```
Monolith:      getOrder() -> in-process getUser() -> ~0.01ms total
Microservices: getOrder() -> HTTP call to User Service -> ~5-30ms per call
```

---

## 30. Production Best Practices

- Don't choose microservices just because "everyone else does" — validate against your actual team size, deployment pain, and scaling needs (see Section 19 decision framework).
- If unsure, **start with a modular monolith**. Enforce module boundaries in code (e.g., via linting rules preventing cross-module imports of internals) so extraction into real services later is mechanical, not a rewrite.
- When you do extract a service, use the **Strangler Fig pattern**: route a thin slice of traffic to the new service via the gateway while the monolith still exists, and only fully cut over once the new service is proven in production.
- Track **deployment frequency** and **cross-team blocking incidents** as leading indicators that a monolith is becoming a bottleneck.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Description |
|---|---|
| **Big Ball of Mud** | A monolith with no internal module boundaries — everything calls everything, impossible to safely change |
| **Premature Decomposition** | Splitting into microservices before you understand your domain boundaries, leading to constant cross-service refactors |
| **Distributed Monolith** | Microservices that share a database or must always deploy together — worst of both worlds |
| **Resume-Driven Architecture** | Choosing microservices because it's trendy/impressive, not because it solves an actual problem you have |
| **"Big Bang" Migration** | Attempting to rewrite the entire monolith into microservices at once, instead of incrementally via Strangler Fig |

```
Big Ball of Mud (Monolith gone wrong):

  UserModule ──calls──▶ OrderModule ──calls──▶ PaymentModule
       ▲                                              │
       └──────────────────calls──────────────────────┘

  Circular dependencies everywhere. No one can change
  one module without breaking three others.
```

---

## 32. Debugging Tips

- **Monolith debugging**: Attach a debugger to the single process; step through the in-process call chain directly. Stack traces show the full call path.
- **Microservices debugging**: You cannot "step through" a request across process boundaries with a normal debugger. Rely on:
  - Correlation/trace IDs across logs.
  - Distributed tracing (OpenTelemetry/Jaeger) to visualize the full multi-service call chain and see where latency or errors occurred.
  - Reproducing the failing chain locally with `docker-compose up` for just the involved services.

---

## 33. Interview Questions

### Easy
1. What is the core structural difference between a monolith and microservices?
2. What is a "modular monolith," and why might a team choose it?
3. Name two advantages of a monolith over microservices.
4. Name two advantages of microservices over a monolith.
5. What does "deployment coupling" mean?

### Medium
6. Why can a monolith sometimes outperform microservices in raw latency for the same feature?
7. What signals indicate a team should start extracting services from a monolith?
8. Explain the "Strangler Fig" migration pattern.
9. Why is "shared database" considered an anti-pattern once you've adopted microservices?
10. How does the CAP theorem's relevance change when moving from a monolith to microservices?

### Hard
11. A company with 10 engineers has been "in microservices hell" for a year — deployments got slower, not faster, after their migration. Diagnose likely causes.
12. Design an incremental migration plan to extract the "Notifications" capability from a monolith into its own service, with zero downtime.
13. How would you decide the right service boundaries when decomposing a monolith? What role does Domain-Driven Design play?
14. What operational capabilities (tooling, monitoring, CI/CD maturity) should a team have *before* attempting a microservices migration?
15. Explain why "the number of services" is a poor metric for architectural maturity.

---

## 34. Scenario-Based Questions

1. Your CTO wants to migrate the entire monolith to 20 microservices over the next quarter, all at once. What risks would you raise, and what alternative would you propose?
2. After splitting into microservices, deployments feel *slower* because every feature still spans 4 services that must be released together. What does this indicate, and how do you fix it?
3. A 5-person startup asks whether they should start with microservices "to be ready to scale." How do you advise them?
4. Your monolith's `orders` module and `payments` module have become so intertwined that no one can change one without breaking the other. Is a full microservices migration required to fix this, or is there a cheaper first step?
5. You're asked to identify the first module to extract from a monolith into a microservice. What criteria would you use to choose it?

---

## 35. Hands-on Exercises

1. Take a hypothetical monolithic "Blog Platform" (Users, Posts, Comments, Notifications) and design it first as a modular monolith (draw module boundaries), then as microservices (draw service boundaries + databases).
2. Write down 5 concrete signals from Section 30 you would monitor to decide when to extract a service from a monolith.
3. For the "Update user profile" example in Section 10, write the exact code differences (in-process call vs HTTP call) side-by-side.
4. Design a Strangler Fig migration plan (in steps) for extracting an "Invoices" module from a monolith into a microservice.
5. List 3 real reasons a small team might rationally prefer a monolith even at moderate scale.

---

## 36. Mini Project

**Build: The Same Feature, Twice**

1. Build a small monolith in Express with two modules: `users` and `posts`, sharing one PostgreSQL database, where `posts` module fetches author details via a direct in-process function call to the `users` module.
2. Refactor it into two separate services (`user-service`, `post-service`), each with its own database, where `post-service` fetches author details via an HTTP call to `user-service`.
3. Write a short comparison doc noting: lines of code changed, new failure modes introduced, and latency difference you'd expect.

---

## 37. Advanced Project

**Build: A Strangler Fig Migration Simulation**

1. Start with a 3-module monolith: `users`, `orders`, `notifications`, all in one Express app with a shared database.
2. Introduce a lightweight API Gateway (e.g., using `express-http-proxy`) in front of the monolith, routing all traffic to it.
3. Extract `notifications` into its own service with its own database. Update the Gateway to route `/notifications/*` to the new service and everything else to the monolith.
4. Implement an event (`OrderPlaced`) published by the monolith to Kafka, consumed by the new Notification Service, replacing the old in-process function call.
5. Document each step as a "migration log entry," describing exactly what changed and what new risk was introduced at each step.

---

## 38. Summary

- A monolith is one deployable unit with (typically) one shared database; microservices are many independently deployable units, each owning its own data.
- Monoliths win on simplicity, raw performance for in-process calls, and low operational overhead — ideal for small teams and early-stage products.
- Microservices win on independent scaling, deployment isolation, and team autonomy — but introduce network failure modes, eventual consistency, and significant operational complexity.
- The **modular monolith** is a legitimate, often-overlooked middle ground: monolith-level simplicity with microservices-style internal boundaries.
- Migrations should be incremental (Strangler Fig), not a risky "big bang" rewrite.
- The right choice depends on team size, domain clarity, and actual (not aspirational) scaling needs — not on industry trends.

---

## 39. Revision Notes

- Monolith: 1 deployable, 1 DB, in-process calls, simple ops, shared fate.
- Microservices: N deployables, N DBs, network calls, complex ops, isolated fate.
- Modular Monolith: 1 deployable, strict internal module boundaries, easier future extraction.
- Distributed Monolith (anti-pattern): looks like microservices, behaves like a tightly-coupled monolith.
- Strangler Fig: incrementally route traffic from monolith to new services via a gateway, extracting piece by piece.
- Decision drivers: team size, deployment pain, scaling unevenness, domain clarity.

---

## 40. One-Page Cheat Sheet

```
MONOLITH:            1 deployable | 1 shared DB | in-process calls | simple ops
MICROSERVICES:        N deployables | DB-per-service | network calls | complex ops
MODULAR MONOLITH:     1 deployable | strict internal boundaries | easy future split
DISTRIBUTED MONOLITH: many services, still coupled (shared DB / must-deploy-together) = BAD

CHOOSE MONOLITH WHEN:      small team, unclear domain, early stage, low traffic unevenness
CHOOSE MICROSERVICES WHEN: multiple teams blocked on each other, uneven scaling needs,
                           clear domain boundaries, mature CI/CD & ops tooling

MIGRATION STRATEGY:  Strangler Fig — gateway + incremental extraction, never big-bang rewrite
KEY RISK TO AVOID:   Shared database across "microservices" = distributed monolith
```

---

**Suggested Next Module:** Module 3 — Microservice Architecture (deep dive into core building blocks: services, registries, gateways, brokers, and how they fit together)
