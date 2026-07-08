# Module 14 — Database per Service

> **Microservices Masterclass** | Level: Intermediate | Track: Node.js Backend Engineering
> Prerequisite: Module 1–13 (especially Module 4 — DDD Basics, Module 9 — Event-Driven Architecture)
> Next Module: Module 15 — Distributed Transactions (Saga Pattern)

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

Throughout this masterclass, "each service owns its own database" has been stated as a foundational rule — introduced in Module 1, reinforced in Module 3, and treated as a hard boundary in Module 5. This module is where we finally stop citing the rule and start rigorously examining it: **why** is this rule so important that it's repeated constantly, **what** specific problems does breaking it cause, and **how** do you actually get the data you need when it lives in someone else's database?

This last question is the crux of the module. If Order Service can't query Product Service's database directly, how does it show a product name on an order confirmation? This module gives you the concrete techniques — polyglot persistence, data duplication via events, and API composition — that make Database per Service practical rather than merely theoretically correct.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Articulate precisely why a shared database undermines microservices' core benefits.
- Explain and apply Polyglot Persistence — choosing the right database technology per service's actual needs.
- Design data duplication strategies (via events) to avoid synchronous cross-service queries for frequently-needed data.
- Distinguish API Composition from data duplication as approaches for combining data across services.
- Implement a working example of a service maintaining its own denormalized read model derived from another service's events.
- Recognize the specific anti-patterns that emerge when the Database per Service rule is violated, even partially.

---

## 3. Problem Statement

A team building an e-commerce platform initially shares one PostgreSQL database across `order-service`, `product-service`, and `inventory-service` "to make joins easier." Over time:

- A schema migration needed by `inventory-service` (adding a new column, changing a type) requires careful coordination with `order-service` and `product-service` teams, since all three read/write overlapping tables — recreating the exact deployment-coupling problem microservices were meant to eliminate (Module 2).
- `product-service`'s team wants to switch their catalog storage to a document database (better suited for their flexible, varied product attributes) but can't, because `order-service` and `inventory-service` still directly query the same relational tables.
- A slow, poorly-optimized query written by `inventory-service`'s team (perhaps during a big data migration) locks tables and degrades performance for `order-service`'s checkout flow — a completely unrelated business capability, coupled only by an accident of shared infrastructure.
- No one can confidently say which service "owns" a given table anymore — several services write to the same `products` table for different reasons, and a bug in one is nearly impossible to trace to its root cause.

This module addresses the two real questions this scenario raises: first, **why** exactly is shared-database access this damaging (beyond "microservices best practice says so"), and second, once truly separated, **how** do services get the cross-cutting data they need?

---

## 4. Why This Concept Exists

Database per Service exists because a **shared database is the single most common and most damaging way to accidentally recreate a monolith while believing you've built microservices**. A database schema is, in practice, an **implicit API** — every table, column, and index is a contract that any service querying it directly depends on, whether or not anyone intended it that way. Unlike an explicit REST or event contract (Modules 7, 9), a shared database schema has:

- **No versioning mechanism** — you can't have a "v1 products table" and a "v2 products table" running side by side the way you can with API versions.
- **No clear ownership** — anyone with database credentials can read or write any table, with no way to enforce "only inventory-service may write to this column."
- **No decoupled evolution** — a schema change for one service's benefit can silently break another service's assumptions, often without any error until data corruption or a subtle bug appears much later.

Database per Service directly enables the **independent deployability** and **team autonomy** benefits promised throughout this masterclass (Modules 1, 2, 5) — those benefits are not achievable, even in principle, with a shared database, no matter how well-organized the services calling into it appear to be.

---

## 5. Historical Background

- **Pre-microservices era** — Large enterprise applications commonly used one shared, "canonical" database (or a small number of them) as the source of truth for the entire organization, with different applications/modules reading and writing to it — a natural extension of monolithic architecture's data model.
- **Early 2010s** — As Netflix, Amazon, and others pioneered microservices at scale, "database per service" emerged as one of the most emphasized, non-negotiable principles distinguishing genuine microservices from merely "distributed monoliths" (a term and concern discussed extensively in Modules 2 and 5).
- **2010s** — The rise of **Polyglot Persistence** (a term popularized in this period, notably by Martin Fowler and colleagues) reflected a growing recognition that different services have genuinely different data needs — a social graph is naturally suited to a graph database, a product catalog with varying attributes suits a document database, and financial transaction records suit a relational database with strong ACID guarantees — and a shared, one-size-fits-all database actively prevents each service from choosing its best-fit technology.
- **Mid-2010s onward** — As Event-Driven Architecture (Module 9) matured, patterns for **data duplication via events** (services maintaining their own local, denormalized copies of data owned by other services) became a standard, well-understood technique for avoiding both shared databases AND excessive synchronous cross-service queries.

---

## 6. Real-World Analogy

**Analogy: Separate Bank Accounts vs. One Shared Family Bank Account**

Imagine three adult siblings who each run their own independent business, but their parents insist they all share **one family bank account** for "simplicity."

- Any sibling can see (and potentially misinterpret) any other sibling's transactions, creating confusion about whose money is whose.
- If one sibling's business needs a specific type of account (a foreign currency account for international trade, say), they can't have it — they're stuck with whatever the shared account supports.
- If one sibling makes a large, risky withdrawal for their own business, it affects the balance and available funds for the other two siblings' completely unrelated businesses.
- Auditing "which business is actually profitable" becomes a tangled mess, since transactions aren't clearly separated by purpose or owner.

The solution is obvious: **each sibling gets their own bank account**, suited to their own business's specific needs (a foreign currency account, a simple checking account, a high-yield savings account), and when money genuinely needs to move between them (e.g., one sibling temporarily lending money to another), it happens via an **explicit transfer** — a deliberate, visible, auditable transaction — not by them all reaching into one shared pot. This explicit transfer is the analogy for both API composition (asking directly) and event-driven duplication (a standing arrangement to automatically receive relevant updates) covered later in this module.

---

## 7. Technical Definition

> **Database per Service** is the principle that each microservice exclusively owns its own private data store, which no other service may access directly — all cross-service data access must happen through the owning service's explicit API or through events it publishes.

> **Polyglot Persistence** is the practice of choosing a different database technology for each service based on that service's specific data access patterns and requirements, rather than forcing every service to use the same database technology organization-wide.

> **Data Duplication (via events)** is the pattern where a service maintains its own local, denormalized copy of data it frequently needs but doesn't own, kept up to date by consuming Domain/Integration Events (Module 9) published by the owning service — trading storage duplication and eventual consistency for reduced runtime coupling.

> **API Composition** is the alternative pattern where, instead of duplicating data locally, a service (or a dedicated aggregator, like the API Gateway from Module 10) queries the owning service's API directly, in real time, whenever the data is needed.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Database per Service** | Each service exclusively owns its own private data store |
| **Polyglot Persistence** | Choosing different database technologies per service based on need |
| **Data Duplication** | Maintaining a local, denormalized copy of another service's data, kept in sync via events |
| **API Composition** | Querying the owning service's API in real time instead of duplicating its data |
| **Materialized View / Read Model** | A locally-stored, pre-computed, denormalized view built from events, optimized for a specific service's read needs |
| **Source of Truth** | The one service that owns and can authoritatively write a given piece of data |
| **Eventual Consistency** | The locally-duplicated data may briefly lag behind the source of truth, converging over time |
| **Denormalization** | Storing data in a flattened, redundant form optimized for read speed, rather than normalized to avoid redundancy |
| **Shared Database Anti-pattern** | Multiple services directly reading/writing the same database, recreating monolith-style coupling |

---

## 9. Internal Working

Here's how a service correctly obtains data it needs but doesn't own, using both of this module's core techniques:

**Technique 1 — API Composition (synchronous, real-time, Module 7-style):**
1. `order-service` needs the current price of a product to display on an order confirmation.
2. Since price changes are relatively infrequent and `order-service` needs the *current* value at the exact moment of order placement, it makes a **synchronous REST/gRPC call** to `product-service`'s API.
3. `product-service` remains the sole owner and sole writer of price data; `order-service` never stores this value long-term, just uses it in the moment.

**Technique 2 — Data Duplication via Events (asynchronous, eventually consistent, Module 9-style):**
1. `order-service` frequently needs to display a customer's name alongside every order in a dashboard — calling `user-service` synchronously for every single dashboard row would be slow and create a hard dependency.
2. Instead, `order-service` **subscribes to `UserUpdated` events** published by `user-service` (Module 9) and maintains its own small, local, denormalized table: `order_service_customer_names (userId, displayName)`.
3. `order-service` reads from this **local copy** for its dashboard needs — fast, no cross-service network call required, and no dependency on `user-service`'s availability at read time.
4. `user-service` remains the sole **source of truth** — `order-service`'s copy is explicitly understood to be a **derived, eventually-consistent read model**, never authoritative, and never directly written to except by consuming events.

The decision between these two techniques mirrors Module 6's sync-vs-async framework: use API Composition when you need the absolute latest value and can tolerate the synchronous dependency; use Data Duplication when you need fast, frequent access and can tolerate slight staleness.

---

## 10. Step-by-Step Request Flow

**Scenario: Rendering an Order History dashboard requiring data from 3 different services' databases.**

```
Step 1:  User requests GET /orders/history from order-service
Step 2:  order-service queries its OWN database for the list of
         orders (order ID, date, total, status) — its own source of truth
Step 3:  For each order, the dashboard needs the customer's display
         name. order-service does NOT call user-service synchronously
         for every row — instead, it JOINS against its OWN LOCAL
         denormalized copy: order_service_customer_names
         (populated ahead of time via consumed UserUpdated events)
Step 4:  For each order, the dashboard ALSO needs the CURRENT,
         real-time shipping status. Since this changes frequently
         and order-service wants the absolute latest value,
         it makes a SYNCHRONOUS call (API Composition) to
         shipping-service for just the handful of orders being displayed
Step 5:  order-service combines: its own data (Step 2) + its local
         denormalized copy (Step 3) + the live API composition
         result (Step 4) into one unified response
Step 6:  Response returned to the user — NO service directly queried
         another service's database at ANY point in this entire flow
```

---

## 11. Architecture Overview

```
   user-service              order-service            shipping-service
   (owns User data)          (owns Order data)          (owns Shipment data)
        │                          │                          │
   ┌─────────┐              ┌─────────────┐              ┌─────────┐
   │ User DB  │              │  Order DB    │              │Shipment │
   │(Postgres)│              │ (Postgres)   │              │   DB    │
   └─────────┘              │              │              │(Mongo)  │
        │                    │  + LOCAL     │                   │
        │  UserUpdated        │  denormalized │                  │
        │  events (Kafka)     │  customer_    │                  │
        └───────────────────▶│  names table  │                  │
                              │  (data         │                  │
                              │   duplication) │                  │
                              └──────┬───────┘                  │
                                     │                            │
                                     │  API Composition            │
                                     │  (live REST call for        │
                                     │   current shipping status)   │
                                     └──────────────────────────────┘
```

---

## 12. ASCII Diagrams

### 12.1 Shared Database (Anti-Pattern) vs Database per Service

```
SHARED DATABASE (anti-pattern):

  order-service ────┐
  product-service ──┼──▶  ONE Shared Database
  inventory-service ─┘     (schema changes require
                            cross-team coordination;
                            any service can read/write
                            ANY table)


DATABASE PER SERVICE (correct):

  order-service ────▶ Order DB (PostgreSQL)
  product-service ──▶ Product DB (MongoDB - flexible attributes)
  inventory-service ▶ Inventory DB (PostgreSQL - strong consistency
                       for stock counts)

  Each service evolves its OWN schema independently;
  cross-service data access ONLY via API or events
```

### 12.2 Polyglot Persistence Decision

```
  Service Data Need                    Best-Fit Database Technology

  Product catalog (flexible,           Document DB (MongoDB)
  varying attributes per category)

  Financial transactions               Relational DB (PostgreSQL) -
  (strong consistency, ACID needed)    strong consistency guarantees

  Social graph / recommendations       Graph DB (Neo4j) - naturally
  (friend-of-friend queries)           suited to relationship queries

  Session/cache data                   Key-Value Store (Redis) -
  (fast, simple, ephemeral)            speed over query complexity

  Analytics/event history               Column-oriented / time-series DB
  (large volume, aggregate queries)     (e.g., ClickHouse) - optimized
                                         for large-scale aggregation
```

### 12.3 API Composition vs Data Duplication

```
API COMPOSITION (real-time, synchronous):

  order-service ──REST call, EVERY time──▶ shipping-service
  Pro: always current data
  Con: synchronous dependency, added latency per request


DATA DUPLICATION (eventually consistent, asynchronous):

  user-service ──publishes UserUpdated events──▶ Kafka
                                                    │
                                          order-service consumes,
                                          maintains LOCAL COPY
                                                    │
  order-service ──reads LOCAL COPY, no network call──▶ (fast!)
  Pro: fast, no runtime dependency on user-service
  Con: local copy may be briefly stale
```

---

## 13. Mermaid Flowcharts

### 13.1 Choosing API Composition vs Data Duplication

```mermaid
flowchart TD
    A[Service needs data it doesn't own] --> B{Is absolute latest value critical?}
    B -- Yes --> C{Is this a low-frequency need - e.g. one-off check?}
    C -- Yes --> D[Use API Composition - synchronous call]
    C -- No, high frequency --> E[Reconsider: maybe cache briefly, or accept slight staleness]
    B -- No, some staleness tolerable --> F[Use Data Duplication via events]
    F --> G[Consume events, maintain local denormalized read model]
```

### 13.2 Polyglot Persistence Selection

```mermaid
flowchart TD
    A[New service being designed] --> B{What are its data access patterns?}
    B -->|Complex relationships, ACID needs| C[Relational DB - PostgreSQL/MySQL]
    B -->|Flexible, varying schema per record| D[Document DB - MongoDB]
    B -->|Fast key-based lookups, caching| E[Key-Value Store - Redis]
    B -->|Relationship/graph traversal queries| F[Graph DB - Neo4j]
    B -->|High-volume time-series/analytics| G[Column-oriented DB - ClickHouse/Cassandra]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Data Duplication via Events (Building a Local Read Model)

```mermaid
sequenceDiagram
    participant U as User Service
    participant K as Kafka
    participant O as Order Service
    participant ODB as Order Service's LOCAL customer_names table

    U->>U: Customer updates their display name
    U->>K: Publish UserUpdated event
    K-->>O: Deliver event (async, whenever ready)
    O->>ODB: UPDATE local customer_names table
    Note over O,ODB: order-service now has an up-to-date<br/>LOCAL copy, without ever querying user-service
```

### 14.2 API Composition (Real-Time Cross-Service Query)

```mermaid
sequenceDiagram
    participant C as Client
    participant O as Order Service
    participant S as Shipping Service

    C->>O: GET /orders/456
    O->>O: Fetch order from OWN database
    O->>S: GET /shipments?orderId=456 (API Composition - real time)
    S-->>O: Current shipment status
    O-->>C: Combined response { order, shipmentStatus }
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                     Order Service                          │
│  ┌───────────────────┐   ┌───────────────────┐             │
│  │  Order DB              │   │  Local Read Model     │             │
│  │  (PostgreSQL,           │   │  (customer_names,      │             │
│  │   SOURCE OF TRUTH        │   │   product_titles -      │             │
│  │   for orders)            │   │   DERIVED, NOT          │             │
│  │                          │   │   authoritative)         │             │
│  └───────────────────┘   └─────────┬─────────┘             │
│                                      ▲                       │
│                            consumed events                   │
│                            (UserUpdated,                     │
│                             ProductRenamed)                  │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                               │
│  order-service pods ──▶ Order DB (PostgreSQL StatefulSet)      │
│  product-service pods ──▶ Product DB (MongoDB StatefulSet)      │
│  inventory-service pods ──▶ Inventory DB (PostgreSQL, SEPARATE  │
│                              instance from Order DB)             │
│                                                               │
│  Each database is a SEPARATE deployable/manageable unit —      │
│  can be scaled, backed up, and upgraded on its OWN schedule,    │
│  independent of any other service's database                   │
└───────────────────────────────────────────────────────────┘
```

---

## 17. Database Interaction

This entire module IS about database interaction, but to make the core rule maximally explicit:

```
ALLOWED:

  order-service  ──reads/writes──▶  Order DB (its OWN database)
  order-service  ──API call──▶      product-service (never Product DB directly)
  order-service  ──consumes events──▶ Kafka topics (never queries another
                                        service's database to "catch up")


NEVER ALLOWED, under ANY circumstances:

  order-service  ──directly queries──▶  Product DB
  inventory-service ──directly writes──▶ Order DB
  ANY service  ──directly accesses──▶  ANY other service's database
```

This is, without exaggeration, the single most important rule in the entire masterclass for preserving the benefits promised in Module 1.

---

## 18. Failure Scenarios

| Scenario | Handling Given Database per Service |
|---|---|
| `product-service`'s database is temporarily down | `order-service`'s LOCAL denormalized product data (if using data duplication) remains available and usable; only genuinely real-time API Composition calls would be affected |
| A locally-duplicated read model becomes stale (event processing lag) | Acceptable, by design, for data duplication — the trade-off explicitly accepted; critical, real-time-required data should use API Composition instead |
| `order-service` needs data from a service that has never published relevant events | Requires either adding a new event type to that service (a coordination point, but a one-time, explicit contract change) or falling back to API Composition |
| A shared database (in a poorly-migrated legacy system) causes one service's slow query to lock tables needed by another | This is precisely the failure mode Database per Service eliminates entirely — each service's database performance issues are now fully isolated |

```
Product Service DB temporarily down:

  order-service needs a product title for a NEW order confirmation
  page (via API Composition, since it wants the CURRENT title)
           │
           ▼
  product-service unreachable — order-service's API call fails
           │
           ▼
  Graceful degradation: show "Product details temporarily
  unavailable" rather than failing the entire order confirmation
  (echoing Module 10's aggregation resilience patterns)

  CONTRAST: if order-service had instead used DATA DUPLICATION
  for product titles (reasonable, since titles rarely change),
  this exact scenario would have had ZERO impact — the local
  copy would still be readable even with product-service down
```

---

## 19. Scalability Discussion

Database per Service directly enables **independent scaling of both compute AND data storage** — `product-service`'s MongoDB cluster can be scaled/sharded based purely on catalog read/write patterns, completely independent of `order-service`'s PostgreSQL scaling needs, which are driven by entirely different traffic patterns (Module 3's independent-scaling benefit, now extended fully down into the data layer). Data duplication via events further improves scalability by removing synchronous, potentially-bottlenecking cross-service database queries from a service's critical read path entirely, at the cost of additional storage (duplicated data) and event-processing infrastructure.

---

## 20. High Availability Considerations

- Each service's database can be made highly available (replication, backups, failover) **independently**, tailored to that specific service's criticality and traffic profile — a low-traffic internal admin tool's database doesn't need the same HA investment as the primary Orders database.
- Data duplication improves overall system availability for read-heavy needs by removing a live, synchronous dependency on another service's availability (Section 18's example).
- API Composition, by contrast, does introduce a synchronous dependency — for genuinely critical, real-time-required data, ensure the owning service itself is highly available (Module 3, Module 7's circuit breaker patterns), since API Composition's availability is only as good as the owning service's.

---

## 21. CAP Theorem Implications

Database per Service, combined with data duplication via events, is the concrete data-layer implementation of the eventual consistency trade-off discussed conceptually in Modules 4, 6, and 9: **each service's own database can maintain strong consistency internally** (a single database, single transaction), while **data duplicated from other services is explicitly, deliberately eventually consistent** — the CAP trade-off is made once, clearly, at the point of choosing data duplication over API Composition for a given need, rather than being an accidental, unexamined property of the system.

---

## 22. Node.js Implementation

Let's implement both API Composition and Data Duplication for the same underlying need — displaying a customer's name alongside an order — so you can see the concrete code difference.

**Folder structure:**
```
order-service/
├── src/
│   ├── clients/
│   │   └── userServiceClient.js       <- API Composition approach
│   ├── readmodels/
│   │   ├── customerNameConsumer.js    <- Data Duplication approach
│   │   └── customerNameRepository.js
│   └── application/
│       └── OrderHistoryService.js
```

**API Composition approach (`clients/userServiceClient.js`):**
```javascript
import axios from "axios";

// Fetches the CURRENT customer name directly from user-service,
// every time it's needed — always fresh, but a synchronous dependency
export async function getCustomerName(userId) {
  const response = await axios.get(
    `${process.env.USER_SERVICE_URL}/users/${userId}`,
    { timeout: 2000 }
  );
  return response.data.displayName;
}
```

**Data Duplication approach (`readmodels/customerNameConsumer.js`):**
```javascript
import { Kafka } from "kafkajs";
import { upsertCustomerName } from "./customerNameRepository.js";

const kafka = new Kafka({ clientId: "order-service", brokers: [process.env.KAFKA_BROKER] });
const consumer = kafka.consumer({ groupId: "order-service-customer-names" });

// Maintains a LOCAL, denormalized copy of customer names, kept
// eventually consistent via events — order-service NEVER queries
// user-service's database directly, and doesn't even need a
// synchronous API call at READ time
export async function startCustomerNameConsumer() {
  await consumer.connect();
  await consumer.subscribe({ topic: "user-events", fromBeginning: true });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      if (event.eventType === "UserUpdated" || event.eventType === "UserRegistered") {
        await upsertCustomerName(event.payload.userId, event.payload.displayName);
      }
    },
  });
}
```

**`readmodels/customerNameRepository.js`**
```javascript
import { db } from "../db/connection.js"; // order-service's OWN database

// This table lives in order-service's OWN database — it is a LOCAL,
// DERIVED copy, explicitly understood to NOT be the source of truth
export async function upsertCustomerName(userId, displayName) {
  await db.query(
    `INSERT INTO order_service_customer_names (user_id, display_name)
     VALUES ($1, $2)
     ON CONFLICT (user_id) DO UPDATE SET display_name = $2`,
    [userId, displayName]
  );
}

export async function getLocalCustomerName(userId) {
  const result = await db.query(
    "SELECT display_name FROM order_service_customer_names WHERE user_id = $1",
    [userId]
  );
  return result.rows[0]?.display_name ?? "Unknown Customer";
}
```

---

## 23. Express.js Examples

**`application/OrderHistoryService.js`** — combining both approaches deliberately, based on data needs
```javascript
import { getOrdersForCustomer } from "../repositories/orderRepository.js";
import { getLocalCustomerName } from "../readmodels/customerNameRepository.js"; // data duplication
import { getCurrentShipmentStatus } from "../clients/shippingServiceClient.js"; // API composition

export async function getOrderHistory(customerId) {
  const orders = await getOrdersForCustomer(customerId); // order-service's OWN data

  // Fast, local read — no network call, uses the eventually-consistent
  // local copy maintained via Data Duplication
  const customerName = await getLocalCustomerName(customerId);

  // Real-time API Composition ONLY for the small number of orders
  // being displayed, since shipment status changes frequently and
  // the dashboard should show the CURRENT status
  const ordersWithShipmentStatus = await Promise.all(
    orders.map(async (order) => ({
      ...order,
      shipmentStatus: await getCurrentShipmentStatus(order.id).catch(() => "Unavailable"),
    }))
  );

  return { customerName, orders: ordersWithShipmentStatus };
}
```

```javascript
// order-service/src/app.js
import express from "express";
import { getOrderHistory } from "./application/OrderHistoryService.js";
import { startCustomerNameConsumer } from "./readmodels/customerNameConsumer.js";

const app = express();

app.get("/orders/history/:customerId", async (req, res) => {
  const history = await getOrderHistory(req.params.customerId);
  res.json(history);
});

app.listen(4002, async () => {
  console.log("Order Service running on port 4002");
  await startCustomerNameConsumer(); // begin building the local read model
});
```

---

## 24. Docker Examples

```yaml
version: "3.9"
services:
  order-service:
    build: ./order-service
    ports: ["4002:4002"]
    environment:
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - USER_SERVICE_URL=http://user-service:4001
      - KAFKA_BROKER=kafka:9092
    depends_on: [order-db, kafka]

  user-service:
    build: ./user-service
    environment:
      - DATABASE_URL=postgresql://user:pass@user-db:5432/users  # SEPARATE database
    depends_on: [user-db]

  product-service:
    build: ./product-service
    environment:
      - DATABASE_URL=mongodb://product-db:27017/products  # POLYGLOT: different DB technology!
    depends_on: [product-db]

  order-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=orders]

  user-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=users]

  product-db:
    image: mongo:7  # DIFFERENT technology from order-db/user-db - Polyglot Persistence in action
```

Notice `product-db` uses **MongoDB** while `order-db` and `user-db` use **PostgreSQL** — a concrete demonstration of Polyglot Persistence, made possible only because each service exclusively owns its own database and no other service needs to understand or query it directly.

---

## 25. Kafka/Redis Integration

This entire module has demonstrated Kafka's central role in enabling Data Duplication (Sections 22-23). Redis can serve as a lighter-weight alternative to a full local database table for read models that don't need complex querying:

```javascript
// A simpler Data Duplication implementation using Redis instead of
// a dedicated local table, suitable for simple key-based lookups
import { redis } from "../db/redis.js";

export async function handleUserUpdated(event) {
  await redis.set(
    `customer-name:${event.payload.userId}`,
    event.payload.displayName
    // No TTL — this is a LONG-LIVED local copy, continuously kept
    // fresh by ongoing events, not a short-lived cache
  );
}

export async function getLocalCustomerName(userId) {
  return (await redis.get(`customer-name:${userId}`)) ?? "Unknown Customer";
}
```

---

## 26. Error Handling

For **Data Duplication**, handle the case where the local copy doesn't yet have an entry (e.g., the consumer hasn't processed the relevant event yet, or an edge case in event ordering):

```javascript
export async function getLocalCustomerName(userId) {
  const name = await getFromLocalStore(userId);
  if (!name) {
    // Fallback strategy: either return a sensible placeholder, or
    // (for a CRITICAL need) fall back to a real-time API Composition
    // call as a one-time exception — a deliberate hybrid approach
    return await getCustomerNameViaApiComposition(userId).catch(() => "Unknown Customer");
  }
  return name;
}
```

For **API Composition**, always handle the owning service being unavailable gracefully (as covered extensively in Modules 7 and 10), rather than allowing one dependency's outage to break every consumer of the composed data.

---

## 27. Logging & Monitoring

- Monitor **event consumer lag** for any Data Duplication read model (Module 9) — growing lag directly translates to growing staleness in the local copy, which stakeholders should understand is an accepted trade-off, but still needs visibility.
- Log **API Composition call latency and failure rates** per downstream dependency, since these directly impact the composing service's own response times and reliability.
- Track how many services maintain a local duplicated copy of a given upstream service's data — an unusually high number might indicate that upstream service's data model needs to be re-examined, or that a shared, well-designed event schema would reduce duplicated effort.

---

## 28. Security Considerations

- Each service's database credentials should be scoped **exclusively** to that service — no shared "god" database user with access to every service's database, which would undermine the security isolation Database per Service is meant to provide.
- Locally duplicated data (read models) should respect the **same** data sensitivity classifications as the source — don't duplicate sensitive fields (e.g., full payment details) into a read model unless genuinely necessary and equally well-protected.
- Auditing data access becomes cleaner with Database per Service: each service's database access logs clearly show exactly which service accessed its own data, with no ambiguity from shared, cross-service database connections.

---

## 29. Performance Optimization

- Choose **Data Duplication** for any cross-service data need that's read frequently but changes relatively infrequently (e.g., customer names, product titles) — the performance win (no network call per read) is substantial at scale.
- Choose **API Composition** for data that's read infrequently relative to how often it changes, or where staleness is genuinely unacceptable (e.g., real-time inventory stock levels immediately before checkout).
- For Polyglot Persistence, select each service's database technology based on **actual measured access patterns** (heavy writes vs. heavy reads, simple key lookups vs. complex queries, need for ACID transactions vs. flexible schema) rather than defaulting to "whatever we already use everywhere."

---

## 30. Production Best Practices

- Treat the decision between API Composition and Data Duplication as a **deliberate, documented architectural choice per data need**, not an ad-hoc decision made differently by every engineer — establish team-wide guidelines (mirroring Module 6's sync/async decision framework).
- Clearly document, per service, which local tables represent **owned, authoritative data** versus **duplicated, derived read models** — a simple naming convention (e.g., prefixing derived tables, as in `order_service_customer_names`) helps prevent confusion and accidental treatment of derived data as a source of truth.
- Regularly audit for any accidental shared-database access that may have crept in (e.g., during a rushed migration or an emergency fix) — this is the single most valuable Database per Service audit you can perform periodically.
- Embrace Polyglot Persistence deliberately where it provides real benefit, but avoid introducing a new database technology purely for novelty — every additional technology adds operational overhead (backups, monitoring, on-call expertise) that must be justified.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Shared database "just for this one join"** | The exact anti-pattern this entire module exists to prevent — recreates monolith-style coupling immediately |
| **Treating a Data Duplication read model as authoritative** | Leads to data integrity confusion — if two "sources" disagree, which one is correct? Only the OWNING service's database should ever be treated as authoritative |
| **Choosing Polyglot Persistence for novelty, not need** | Adds operational complexity (new backup strategy, new monitoring, new team expertise needed) without a genuine corresponding benefit |
| **No consumer for a published event, "just in case it's needed later"** | Maintaining unused Data Duplication infrastructure adds unnecessary storage and complexity; build it when there's an actual, current need |
| **Excessive, unstructured Data Duplication across many services** | Can lead to redundant, hard-to-audit copies of the same data scattered everywhere — establish clear conventions and periodic reviews |

```
Treating a read model as authoritative (anti-pattern):

  order-service's LOCAL customer_names table shows "John Smith"

  An engineer, unaware this is a DERIVED copy, "corrects" a typo
  directly in order-service's local table

  Problem: the NEXT UserUpdated event from user-service will
  OVERWRITE this "correction" — because user-service, NOT
  order-service's local copy, is the actual source of truth.
  The fix must happen in user-service, not in the derived copy.
```

---

## 32. Debugging Tips

- If a Data Duplication read model shows stale or incorrect data, check **consumer lag first** (Module 9's debugging approach), then verify the source service's actual current data to confirm which one is "correct" (hint: the owning service always is).
- If you discover two services can somehow see the same underlying data changing in sync without any event or API call between them, **immediately investigate for an accidental shared database connection** — this is a serious architectural violation, not a minor bug.
- When choosing a database technology for a new service, profile its **actual** expected read/write patterns and query complexity before committing — Polyglot Persistence should be an informed choice, not a guess.

---

## 33. Interview Questions

### Easy
1. What does "Database per Service" mean, and why is it a core microservices principle?
2. What is Polyglot Persistence?
3. What is the difference between API Composition and Data Duplication for accessing another service's data?
4. Why is a shared database considered an anti-pattern in microservices?
5. What does it mean for locally duplicated data to be "eventually consistent"?

### Medium
6. How would you decide between API Composition and Data Duplication for a specific cross-service data need?
7. Explain why a shared database schema is effectively an "implicit API," and why that's problematic.
8. What are the security benefits of Database per Service beyond just architectural cleanliness?
9. How does Database per Service enable independent scaling of both services AND their data stores?
10. Why must a Data Duplication read model never be treated as authoritative, even if it's more convenient to edit directly?

### Hard
11. Design the data ownership and access strategy for an e-commerce platform with Order, Product, Inventory, and Review services, specifying which cross-service data needs should use API Composition versus Data Duplication.
12. How would you migrate an existing shared-database system to Database per Service incrementally, without a risky "big bang" cutover?
13. Discuss the trade-offs of choosing a different database technology (Polyglot Persistence) for every service versus standardizing on one technology organization-wide for operational simplicity.
14. Explain how you would detect, in a large existing system, any services that are accidentally sharing a database or querying another service's tables directly.
15. Design a strategy for keeping multiple services' Data Duplication read models consistent with each other when they all derive from the same upstream event stream but may process events at different rates.

---

## 34. Scenario-Based Questions

1. Your team discovers that `analytics-service` has been directly querying `order-service`'s production database "temporarily" for the past 8 months, for a dashboard. What risks does this pose, and what's your remediation plan?
2. A new engineer suggests adding a foreign key constraint between `orders` (in Order DB) and `products` (in Product DB) "to ensure data integrity." Why is this not possible/appropriate in a Database per Service architecture, and what would you suggest instead?
3. Your `order-service`'s local, duplicated `product_titles` read model has been showing stale data for 2 hours. Walk through your debugging process.
4. Leadership wants to switch `product-service`'s database from PostgreSQL to a document database for better catalog flexibility. What concerns (if any) does this raise for other services, given proper Database per Service adherence?
5. Two services both need frequent access to "current inventory stock levels" — one for display purposes (slight staleness acceptable) and one for checkout validation (must be current). How would you architect data access differently for each?

---

## 35. Hands-on Exercises

1. For a "Food Delivery" platform (Restaurant, Menu, Order, Delivery services), decide and justify a Polyglot Persistence strategy — which database technology fits each service's likely data access patterns?
2. Implement the Data Duplication pattern from Section 22-23 (a local `customer_names` table kept in sync via consumed events).
3. Implement the API Composition pattern from Section 22-23 (a real-time call to fetch current shipment status).
4. Identify, in a hypothetical (or real) system, one place where API Composition is currently used but Data Duplication would likely perform better (or vice versa), and justify switching.
5. Write a short audit checklist you'd use to verify no service in a given system is accidentally violating Database per Service.

---

## 36. Mini Project

**Build: Two Services With Both Composition and Duplication Patterns**

1. Build `user-service` (PostgreSQL) publishing `UserUpdated` events to Kafka whenever a display name changes.
2. Build `order-service` (a SEPARATE PostgreSQL database) that (a) maintains a local `customer_names` table via a Kafka consumer (Data Duplication), and (b) also has a client that can call `user-service`'s API directly for a "get full profile" endpoint (API Composition), demonstrating both patterns coexisting for different needs.
3. Demonstrate: stopping `user-service` entirely, and confirming `order-service`'s dashboard (using the local read model) still works, while a hypothetical "full profile" feature (using API Composition) correctly fails gracefully.

---

## 37. Advanced Project

**Build: A Full Polyglot Persistence System With Both Data Access Patterns**

1. Build 3 services: `user-service` (PostgreSQL), `product-service` (MongoDB — Polyglot Persistence), and `order-service` (PostgreSQL).
2. `order-service` maintains local, denormalized read models for BOTH customer names (from `user-service`'s events) and product titles (from `product-service`'s events), demonstrating Data Duplication across two different upstream services with two different underlying database technologies.
3. Implement a real-time API Composition call from `order-service` to a new `inventory-service` for checking current stock immediately before order confirmation (where staleness is NOT acceptable).
4. Simulate and handle a failure: stop `product-service` and confirm `order-service`'s existing orders still display correctly (using the local read model), while a NEW product lookup (if any exists via API Composition elsewhere in your system) fails gracefully.
5. Write an architecture decision document justifying, for each of the 3 cross-service data needs in this project, why you chose Data Duplication or API Composition specifically.

---

## 38. Summary

- Database per Service is the principle that each microservice exclusively owns its data store, with no other service permitted direct access — a shared database recreates monolithic coupling regardless of how well-separated the application code appears to be.
- Polyglot Persistence lets each service choose the database technology genuinely best suited to its own data access patterns, a benefit only possible because of strict data ownership.
- Cross-service data needs are met through API Composition (real-time, synchronous, always current) or Data Duplication (asynchronous, eventually consistent, fast local reads) — a deliberate choice per data need, mirroring Module 6's sync/async decision framework.
- Data Duplication read models must always be treated as derived and non-authoritative; the owning service remains the sole source of truth.
- This module's rule directly enables the independent scaling, deployment, and team-ownership benefits promised throughout the entire masterclass — violating it, even partially, undermines those benefits significantly.

---

## 39. Revision Notes

- Database per Service: each service exclusively owns its data; no shared databases, ever.
- Polyglot Persistence: choose the right database technology per service's actual needs.
- API Composition: real-time, synchronous query to the owning service — always current, adds a dependency.
- Data Duplication: local, denormalized copy kept in sync via events — fast, decoupled, eventually consistent.
- Read models (duplicated data) are NEVER authoritative — the owning service is always the source of truth.
- A shared database schema is an unversioned, ownerless "implicit API" — the core reason it's so damaging.

---

## 40. One-Page Cheat Sheet

```
DATABASE PER SERVICE:   each service exclusively owns its own data store
POLYGLOT PERSISTENCE:   different DB tech per service, chosen by actual need
API COMPOSITION:        real-time synchronous query -> always current, adds dependency
DATA DUPLICATION:       local copy via events -> fast, decoupled, eventually consistent

DECISION RULE: staleness unacceptable + infrequent need -> API Composition
               staleness acceptable + frequent need      -> Data Duplication

GOLDEN RULES:
  - NEVER let one service directly query another service's database
  - The OWNING service is ALWAYS the source of truth, even for duplicated copies
  - Choose DB technology per service's real access patterns, not organizational default
  - Document, per cross-service data need, WHY you chose composition vs duplication
  - Regularly audit for accidental shared-database access creeping into the system
```

---

**Suggested Next Module:** Module 15 — Distributed Transactions (Saga Pattern) (handling multi-service transactions without ACID guarantees, using Choreography and Orchestration-based Sagas with compensating actions)
