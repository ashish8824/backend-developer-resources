# Module 16 — CQRS (Command Query Responsibility Segregation)

> **Microservices Masterclass** | Level: Advanced | Track: Node.js Backend Engineering
> Prerequisite: Module 1–15 (especially Module 14 — Database per Service, Module 15 — Saga Pattern)
> Next Module: Module 17 — Event Sourcing

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

Module 14 introduced Data Duplication: a service maintaining its own local, denormalized copy of another service's data to avoid slow, coupling-heavy synchronous calls. That pattern — write to one model, read from a different, purpose-built model — is actually a specific instance of a much more general and powerful architectural principle: **CQRS (Command Query Responsibility Segregation)**.

Most applications (and most of this masterclass so far) have used the same model for both reading and writing data — the same `Order` object, the same `orders` table, whether you're creating an order or displaying a dashboard of order history. CQRS challenges this assumption directly: **what if the way you need to write data and the way you need to read data are fundamentally different problems, each deserving its own optimized solution?** This module explores when that question's answer is "yes, split them," how to do it safely, and — critically — when CQRS is unnecessary complexity you should avoid.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain the CQRS pattern and how it differs from a traditional single-model CRUD approach.
- Identify the specific symptoms that indicate CQRS would genuinely help a system.
- Design separate command (write) and query (read) models for the same underlying business entity.
- Implement a working CQRS system in Node.js, including keeping the read model in sync via events.
- Understand the eventual consistency implications CQRS introduces between writes and reads.
- Recognize when CQRS is overengineering, and avoid applying it indiscriminately.

---

## 3. Problem Statement

An e-commerce platform's Order Service needs to support two very different workloads on the same underlying "order" data:

- **Writing an order** (placing it) is relatively simple: validate the request, apply a few business rules, insert one row (plus line items) — a straightforward, normalized, transactional write.
- **Reading orders** for a customer dashboard needs to show: order totals, item counts, customer name, current shipping status, and a running summary of "total spent this year" — requiring **joins and aggregations** across data that, if normalized purely for writing, would require expensive queries on every single dashboard load, especially at scale with millions of orders.

If the team optimizes their single `orders` table purely for writing (fully normalized, minimal redundancy), the dashboard becomes slow under real load, requiring complex joins and aggregations computed on-demand, every single time, for every user. If they instead optimize purely for reading (a wide, denormalized, redundant structure making dashboards fast), writing a new order becomes awkward and error-prone, since every write must carefully update multiple redundant fields consistently.

This module solves the underlying issue: **you don't have to choose one structure that serves both needs poorly — you can maintain two separate models, each optimized for its own specific job**, kept in sync via the events pattern from Module 9.

---

## 4. Why This Concept Exists

CQRS exists because **the assumption that one single data model must serve both write and read needs equally well is often false, and forcing it creates unnecessary trade-offs.** Traditional CRUD design treats "the entity" as one thing with one shape — but in reality:

- **Writes** typically care about business rules, validation, and consistency — they benefit from a normalized, transactional structure that prevents data anomalies.
- **Reads** typically care about speed and convenience — they benefit from a denormalized, pre-joined, sometimes pre-aggregated structure that avoids expensive computation at read time.

These are genuinely different concerns with genuinely different optimal solutions. CQRS gives you permission to **stop compromising** between them — you get a write model optimized purely for correctness and a read model optimized purely for query performance, paying the cost of keeping them in sync (usually via events) rather than the cost of a single, compromised model serving both poorly.

---

## 5. Historical Background

- **2000s** — The **Command-Query Separation (CQS)** principle, introduced by Bertrand Meyer in the context of object-oriented programming, established the idea that a method should either be a **command** (performs an action, changes state) or a **query** (returns data, has no side effects) — never both. This was originally a code-level, single-object design principle.
- **Late 2000s** — **Greg Young** extended CQS's idea to the **architectural** level, coining the term **CQRS**: rather than just individual methods being either commands or queries, entire **models** (and often entire data stores) could be separated by responsibility — one for handling writes (commands), one for handling reads (queries).
- **2010s** — As Event-Driven Architecture (Module 9) and Event Sourcing (discussed next in Module 17) matured within the DDD and microservices communities, CQRS became a natural, frequently-paired pattern: an event stream naturally serves as the bridge keeping a write model's changes propagated into one or more purpose-built read models.
- **Present** — CQRS is widely recognized as a powerful but **situational** pattern — commonly emphasized in the community precisely because it's **overused** almost as often as it's rightly used; experienced architects (including Greg Young himself) have repeatedly cautioned against applying it to simple CRUD scenarios where it adds unjustified complexity.

---

## 6. Real-World Analogy

**Analogy: A Restaurant's Order Ticket vs. the Nightly Sales Report**

Think about how a busy restaurant handles two very different needs around the same underlying fact ("what did customers order tonight"):

- **The kitchen order ticket (the Command/Write side)** is simple, immediate, and transactional: "Table 5: 2x Burger, 1x Salad, no onions." It's written once, quickly, by the server taking the order, and it needs to be **correct** above all else — the kitchen must prepare exactly what was ordered.
- **The nightly sales report (the Query/Read side)** is a completely different shape of information, built from the SAME underlying orders but reorganized entirely: "Total revenue: $4,230. Best-selling item: Burger (87 sold). Average table spend: $52." This report is **derived** from the individual order tickets, but pre-aggregated, summarized, and optimized for the restaurant manager to glance at and understand instantly — no one wants the manager to have to manually add up 200 individual paper tickets every night.

The kitchen never writes directly into "the sales report" structure — that would slow down every single order-taking moment with unnecessary aggregation work. Instead, the sales report is built **separately**, derived from the same underlying orders, at its own pace, optimized purely for the reading/reporting need. This is exactly CQRS: one model optimized for the fast, correct act of writing (the order ticket), a completely separate model optimized for the different need of reading/reporting (the sales summary), derived from the same source of truth but shaped entirely differently.

---

## 7. Technical Definition

> **CQRS (Command Query Responsibility Segregation)** is an architectural pattern that separates the model used to **write** data (the Command Model, optimized for validating and applying business rules and changes) from the model used to **read** data (the Query Model, optimized for the specific shape and performance needs of read operations), with the Query Model typically kept up to date asynchronously, derived from changes made via the Command Model.

> A **Command** is a request to change state (e.g., `PlaceOrder`, `UpdateShippingAddress`) — it does not return data, only an acknowledgment of success/failure (echoing the Command vs. Event distinction from Module 9).

> A **Query** is a request to read data (e.g., `GetOrderHistory`) — it does not change any state, and is served entirely by the Query Model, never the Command Model.

> The **Query Model (Read Model)** is often a **denormalized**, purpose-built data structure — potentially stored in a completely different database technology than the Command Model (Polyglot Persistence, Module 14) — optimized specifically for the read patterns it needs to serve.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **CQRS** | Separating the write model (Command) from the read model (Query) |
| **Command Model / Write Model** | The model handling business logic, validation, and state changes — often normalized |
| **Query Model / Read Model** | The model optimized for specific read/query needs — often denormalized |
| **Command** | A request to change state, returning only success/failure |
| **Query** | A request to read data, causing no state change |
| **Command Handler** | The component processing a specific command, applying business logic |
| **Projection** | The process of transforming events/changes from the Command Model into an updated Query Model |
| **Read Model Synchronization** | Keeping the Query Model up to date, typically via consumed events (Module 9) |
| **Single Model CRUD** | The traditional approach where ONE model serves both writes and reads — CQRS's alternative |

---

## 9. Internal Working

Here's how a CQRS system works end-to-end:

1. A client sends a **Command** (e.g., `PlaceOrder`) to the Command side of the service.
2. The **Command Handler** validates the command, applies business rules (potentially using DDD's Aggregates from Module 4), and persists the change to the **Command Model's** database — typically normalized, optimized for correctness and transactional integrity.
3. Upon successfully persisting the change, the service publishes an **event** (e.g., `OrderPlaced`) reflecting what just happened (directly connecting to Module 9's Event-Driven Architecture).
4. A separate **Projection** component (which may live in the same service or a dedicated read-model service) **consumes** this event and updates the **Query Model** — a denormalized structure specifically shaped for a known read need (e.g., a `order_summary_view` table with pre-joined customer name, pre-calculated totals, ready for instant dashboard display).
5. When a client sends a **Query** (e.g., `GetOrderHistory`), it is served **entirely** from the Query Model — never touching the Command Model's database at all, and never re-computing joins/aggregations on the fly.
6. Because the Query Model is updated **asynchronously** (via the projection consuming events), there is a small window of **eventual consistency**: a just-placed order might not appear in the read model for a few milliseconds to seconds, depending on event processing latency.

---

## 10. Step-by-Step Request Flow

**Scenario: Placing an order (Command) and viewing the order history dashboard (Query), in a CQRS system.**

```
COMMAND SIDE (writing):

Step 1:  Client sends PlaceOrder command: { customerId, items }
Step 2:  Command Handler validates the command (stock check via
         Saga, Module 15; business rules via Order Aggregate, Module 4)
Step 3:  Command Handler persists the new order to the COMMAND
         MODEL's database (normalized: orders + order_line_items tables)
Step 4:  Command Handler publishes OrderPlaced event
Step 5:  Command side returns { orderId, status: "PLACED" } to the client
         (NOT the full order details — just acknowledgment, per CQRS's
         command/query distinction)


QUERY SIDE (reading, asynchronously updated):

Step 6:  A Projection consumer picks up the OrderPlaced event
Step 7:  Projection fetches any additional data it needs (e.g., the
         customer's display name, perhaps from a LOCAL cache built
         via Module 14's Data Duplication pattern)
Step 8:  Projection writes a DENORMALIZED row into the QUERY MODEL's
         database: order_summary_view (orderId, customerName, itemCount,
         total, status, placedAt) — pre-joined, pre-computed, ready
         for instant reads
Step 9:  (Some time later) Client requests GetOrderHistory for this customer
Step 10: Query Handler reads DIRECTLY from order_summary_view — a
         single, simple, fast query with NO joins or aggregation
         needed at read time, since the Projection already did that work
```

---

## 11. Architecture Overview

```
                          Client
                    ┌───────┴───────┐
                    ▼               ▼
              COMMAND SIDE      QUERY SIDE
             (writes/business    (reads/dashboards)
              logic)
                    │               │
                    ▼               ▼
         ┌─────────────────┐  ┌─────────────────┐
         │  Command Model      │  │  Query Model         │
         │  Database             │  │  Database              │
         │  (normalized,          │  │  (denormalized,         │
         │   PostgreSQL,           │  │   pre-joined,            │
         │   optimized for          │  │   optimized for           │
         │   correctness)           │  │   read speed - could      │
         │                          │  │   even be a DIFFERENT      │
         │                          │  │   technology, e.g. a        │
         │                          │  │   search index or NoSQL)     │
         └─────────┬───────────┘  └─────────▲───────────┘
                    │                          │
                    ▼                          │
              Publishes Events ─────────▶ Projection
              (OrderPlaced, etc.)         (consumes events,
                                           updates Query Model)
```

---

## 12. ASCII Diagrams

### 12.1 Single Model CRUD vs CQRS

```
SINGLE MODEL CRUD (traditional):

  Write ──▶ ONE Model/DB ──▶ Read
  (same structure serves both; compromises between
   normalization-for-correctness and denormalization-for-speed)


CQRS (separated):

  Write (Command) ──▶ Command Model DB (normalized)
                              │
                        publishes event
                              │
                              ▼
                        Projection
                              │
                              ▼
  Read (Query) ◀────── Query Model DB (denormalized, FAST reads)
```

### 12.2 Command Model vs Query Model Shape

```
COMMAND MODEL (normalized, for correctness):

  orders table:          order_line_items table:
  id | customer_id       order_id | product_id | qty
  ---|------------       ---------|-----------|----
  1  | 42                1        | abc123     | 2
                          1        | def456     | 1


QUERY MODEL (denormalized, for read speed - ONE row has EVERYTHING):

  order_summary_view:
  order_id | customer_name | item_count | total   | status
  ---------|---------------|------------|---------|--------
  1        | "Jane Doe"    | 3          | $49.99  | PLACED

  (NO JOIN needed at read time - everything is pre-computed
   and denormalized into ONE flat row)
```

### 12.3 Multiple Query Models From One Command Model

```
              Command Model (Orders - source of truth)
                              │
                       publishes events
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
     Query Model A:    Query Model B:    Query Model C:
     Customer Order    Admin Analytics    Search Index
     History Dashboard  Dashboard          (Elasticsearch,
     (fast lookups by   (aggregated        supporting fuzzy
      customer)          revenue by         text search across
                         product/day)       order contents)

     ONE command model can feed MULTIPLE, DIFFERENTLY-SHAPED
     query models, each optimized for its own specific need
```

---

## 13. Mermaid Flowcharts

### 13.1 Should You Use CQRS?

```mermaid
flowchart TD
    A[Considering CQRS for this service] --> B{Are read and write patterns genuinely very different?}
    B -- No, simple CRUD is fine --> C[Do NOT use CQRS - unnecessary complexity]
    B -- Yes --> D{Is read performance/scale a REAL, measured problem?}
    D -- No --> C
    D -- Yes --> E{Can the business tolerate eventual consistency between write and read?}
    E -- No --> F[Reconsider - CQRS may not fit this specific need]
    E -- Yes --> G[CQRS is likely a good fit - implement Command/Query separation]
```

### 13.2 Command and Projection Flow

```mermaid
flowchart LR
    A[Command received] --> B[Command Handler validates + applies business logic]
    B --> C[Persist to Command Model DB]
    C --> D[Publish Domain Event]
    D --> E[Projection consumes event]
    E --> F[Update Query Model DB - denormalized]
    F --> G[Query Model ready for fast reads]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Full CQRS Write-Then-Read Flow

```mermaid
sequenceDiagram
    participant C as Client
    participant CH as Command Handler
    participant CDB as Command Model DB
    participant K as Kafka
    participant P as Projection
    participant QDB as Query Model DB
    participant QH as Query Handler

    C->>CH: PlaceOrder command
    CH->>CDB: Persist order (normalized)
    CDB-->>CH: Success
    CH->>K: Publish OrderPlaced event
    CH-->>C: { orderId, status: "PLACED" }
    Note over C: Client has its response - the read model<br/>update below happens independently
    K-->>P: Consume OrderPlaced (async)
    P->>QDB: Write denormalized order_summary_view row
    C->>QH: GetOrderHistory query (later)
    QH->>QDB: Simple, fast, pre-joined read
    QDB-->>QH: Order summary rows
    QH-->>C: Order history response
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                     Order Service (CQRS)                    │
│  ┌───────────────────────┐   ┌───────────────────────┐      │
│  │      COMMAND SIDE          │   │       QUERY SIDE           │      │
│  │  ┌───────────────────┐  │   │  ┌───────────────────┐  │      │
│  │  │ Command Handlers      │  │   │  │ Query Handlers        │  │      │
│  │  │ (PlaceOrder,           │  │   │  │ (GetOrderHistory,      │  │      │
│  │  │  CancelOrder)           │  │   │  │  GetOrderById)          │  │      │
│  │  └─────────┬─────────┘  │   │  └─────────┬─────────┘  │      │
│  │            ▼               │   │            ▲               │      │
│  │  ┌───────────────────┐  │   │  ┌───────────┴───────┐  │      │
│  │  │ Command Model DB      │  │   │  │ Query Model DB          │  │      │
│  │  │ (normalized)            │  │   │  │ (denormalized)           │  │      │
│  │  └─────────┬─────────┘  │   │  └───────────▲───────┘  │      │
│  └────────────┼───────────┘   └───────────────┼──────────┘      │
│               ▼                                 │                   │
│         Publishes Events ──────────▶ Projection Consumer            │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                               │
│  order-command-service pods ──▶ Command DB (PostgreSQL)         │
│         │                                                     │
│  Publishes events to Kafka                                    │
│         │                                                     │
│  order-projection pods (consumer, updates read model)           │
│         │                                                     │
│  order-query-service pods ──▶ Query DB (could be a DIFFERENT     │
│                                 technology - e.g. Elasticsearch    │
│                                 for search, or a read-optimized     │
│                                 PostgreSQL replica/materialized     │
│                                 view)                               │
│                                                               │
│  Command and Query sides can be SEPARATE deployments,           │
│  scaled INDEPENDENTLY based on their very different              │
│  traffic patterns (writes are often much lower volume            │
│  than reads in most systems)                                    │
└───────────────────────────────────────────────────────────┘
```

---

## 17. Database Interaction

CQRS takes Module 14's Data Duplication concept and elevates it to a first-class architectural principle rather than an occasional technique:

```
COMMAND MODEL DATABASE (source of truth, normalized):
  orders (id, customer_id, status, created_at)
  order_line_items (order_id, product_id, quantity, unit_price)

QUERY MODEL DATABASE (derived, denormalized, can be a
DIFFERENT technology entirely):
  order_summary_view (order_id, customer_name, item_count,
                       total, status, placed_at)
  -- ALL fields pre-computed and flattened; a dashboard query
  -- is a SIMPLE "SELECT * FROM order_summary_view WHERE
  -- customer_id = ?" with ZERO joins or aggregation needed

The Query Model is NEVER written to directly by command
handlers - it is EXCLUSIVELY updated by the Projection,
which consumes events from the Command side.
```

---

## 18. Failure Scenarios

| Scenario | CQRS Handling |
|---|---|
| The Projection consumer falls behind (lag) | The Query Model becomes increasingly stale, but the Command Model (source of truth) remains correct and unaffected — this is the accepted eventual consistency trade-off, monitored via consumer lag (Module 9) |
| The Query Model's database is temporarily unavailable | Reads fail/degrade, but WRITES (via the Command Model) can continue completely unaffected, since they're entirely separate databases — a resilience benefit of the separation |
| A bug in the Projection logic corrupts the Query Model | Since the Query Model is fully DERIVED from events, it can typically be **rebuilt from scratch** by replaying the event log (assuming events are retained, as with Kafka) — a significant advantage over a single-model system where a corrupted table has no independent source to rebuild from |
| A client reads the Query Model immediately after a write and doesn't see their own change yet | A common CQRS UX challenge — mitigated by techniques like returning enough data from the Command side's acknowledgment for the client to update its own UI optimistically, without needing to immediately re-query the (still-catching-up) read model |

```
Rebuilding a corrupted Query Model (a genuine CQRS advantage):

  Bug introduced in Projection logic corrupts order_summary_view
           │
           ▼
  Since the Command Model (source of truth) is UNAFFECTED, and
  Kafka retains the event history, you can:
    1. Fix the Projection's bug
    2. Truncate the corrupted Query Model
    3. REPLAY all historical events from Kafka
    4. Query Model is rebuilt correctly from scratch
```

---

## 19. Scalability Discussion

CQRS enables **independent scaling of reads and writes**, which is often exactly what's needed since most systems have vastly different read and write volumes (frequently 10:1, 100:1, or higher read-to-write ratios). The Query Model database can be **scaled, replicated, and even choose an entirely different technology** (e.g., Elasticsearch for full-text search, a read-optimized columnar store for analytics) purely based on read performance needs, without those choices ever needing to compromise the Command Model's design, which can remain focused purely on correctness and transactional integrity for writes.

---

## 20. High Availability Considerations

- Because Command and Query sides are physically separate, an outage in the Query Model's infrastructure does **not** prevent new writes from succeeding — a meaningful resilience benefit (echoing Module 9's producer/consumer decoupling).
- The Projection mechanism itself should be deployed with multiple consumer instances (within a consumer group, Module 9) to avoid it becoming a single point of failure for keeping the Query Model current.
- Since the Query Model can be **rebuilt from the event log** (Section 18), a full Query Model outage/corruption is recoverable without data loss, provided your event broker retains sufficient history — a significant HA/disaster-recovery advantage worth designing for deliberately.

---

## 21. CAP Theorem Implications

CQRS makes the **eventual consistency** trade-off explicit and structural, rather than incidental: the Query Model is, by design and by name, **not** the source of truth, and is expected to lag behind the Command Model by some (ideally small) amount of time. This is a deliberate choice to favor **Availability and read performance** over strict, immediate read-after-write consistency — appropriate for the vast majority of read use cases (dashboards, reports, listings) but requiring careful UX consideration for the specific cases where a user expects to immediately see the effect of their own just-submitted action.

---

## 22. Node.js Implementation

Let's implement a working CQRS system: a Command side (placing orders) and a Query side (order history), connected via an event-driven Projection.

**Folder structure:**
```
order-command-service/
├── src/
│   ├── commands/
│   │   └── placeOrderHandler.js
│   ├── domain/
│   │   └── Order.js  (from Module 4's DDD Aggregate)
│   └── app.js

order-query-service/
├── src/
│   ├── projections/
│   │   └── orderSummaryProjection.js
│   ├── queries/
│   │   └── getOrderHistoryHandler.js
│   └── app.js
```

**`order-command-service/src/commands/placeOrderHandler.js`**
```javascript
import { Order } from "../domain/Order.js";
import { orderRepository } from "../repositories/orderRepository.js";
import { publishEvent } from "../events/eventPublisher.js";

// COMMAND: changes state, returns only acknowledgment (not full data)
export async function handlePlaceOrder({ customerId, items }) {
  const order = new Order(crypto.randomUUID(), customerId);
  for (const item of items) {
    order.addLineItem(item.productId, item.quantity, item.unitPrice);
  }
  order.place(); // enforces business invariants, per Module 4

  await orderRepository.save(order); // writes to the NORMALIZED command model DB

  await publishEvent("order-events", {
    eventType: "OrderPlaced",
    payload: {
      orderId: order.id,
      customerId: order.customerId,
      itemCount: order.lineItems.length,
      total: order.total,
      placedAt: order.placedAt,
    },
  });

  // Return ONLY acknowledgment - the command side does not serve
  // rich query data; that is the QUERY side's job
  return { orderId: order.id, status: "PLACED" };
}
```

**`order-query-service/src/projections/orderSummaryProjection.js`**
```javascript
import { Kafka } from "kafkajs";
import { getCustomerName } from "../clients/userServiceClient.js"; // may use API Composition or local cache, Module 14
import { upsertOrderSummary } from "../repositories/orderSummaryRepository.js";

const kafka = new Kafka({ clientId: "order-query-service", brokers: [process.env.KAFKA_BROKER] });
const consumer = kafka.consumer({ groupId: "order-summary-projection" });

// The PROJECTION: the ONLY thing allowed to write to the Query Model
export async function startProjection() {
  await consumer.connect();
  await consumer.subscribe({ topic: "order-events", fromBeginning: true });

  await consumer.run({
    eachMessage: async ({ message }) => {
      const event = JSON.parse(message.value.toString());
      if (event.eventType !== "OrderPlaced") return;

      const customerName = await getCustomerName(event.payload.customerId);

      // Write the DENORMALIZED, pre-joined row - this is what makes
      // reads fast: everything needed is already flattened into one row
      await upsertOrderSummary({
        orderId: event.payload.orderId,
        customerId: event.payload.customerId,
        customerName,
        itemCount: event.payload.itemCount,
        total: event.payload.total,
        status: "PLACED",
        placedAt: event.payload.placedAt,
      });
    },
  });
}
```

---

## 23. Express.js Examples

**`order-command-service/src/app.js`**
```javascript
import express from "express";
import { handlePlaceOrder } from "./commands/placeOrderHandler.js";

const app = express();
app.use(express.json());

// COMMAND endpoint - a POST, returns ONLY acknowledgment
app.post("/commands/place-order", async (req, res) => {
  try {
    const result = await handlePlaceOrder(req.body);
    res.status(201).json(result); // { orderId, status } - NOT full order data
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.listen(4020, () => console.log("Order Command Service running on port 4020"));
```

**`order-query-service/src/queries/getOrderHistoryHandler.js`**
```javascript
import { db } from "../db/connection.js"; // the QUERY MODEL's own database

// QUERY: read-only, served ENTIRELY from the denormalized read model
export async function getOrderHistory(customerId) {
  const result = await db.query(
    `SELECT order_id, customer_name, item_count, total, status, placed_at
     FROM order_summary_view
     WHERE customer_id = $1
     ORDER BY placed_at DESC`,
    [customerId]
  );
  return result.rows; // a SIMPLE query - no joins, no aggregation needed here
}
```

```javascript
// order-query-service/src/app.js
import express from "express";
import { getOrderHistory } from "./queries/getOrderHistoryHandler.js";
import { startProjection } from "./projections/orderSummaryProjection.js";

const app = express();

// QUERY endpoint - a GET, served entirely from the read model
app.get("/queries/order-history/:customerId", async (req, res) => {
  const history = await getOrderHistory(req.params.customerId);
  res.json(history);
});

app.listen(4021, async () => {
  console.log("Order Query Service running on port 4021");
  await startProjection(); // begin keeping the read model up to date
});
```

---

## 24. Docker Examples

```yaml
version: "3.9"
services:
  order-command-service:
    build: ./order-command-service
    ports: ["4020:4020"]
    environment:
      - DATABASE_URL=postgresql://user:pass@command-db:5432/orders_command
      - KAFKA_BROKER=kafka:9092
    depends_on: [command-db, kafka]

  order-query-service:
    build: ./order-query-service
    ports: ["4021:4021"]
    environment:
      - DATABASE_URL=postgresql://user:pass@query-db:5432/orders_query
      - KAFKA_BROKER=kafka:9092
      - USER_SERVICE_URL=http://user-service:4001
    depends_on: [query-db, kafka, user-service]

  command-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=orders_command]

  query-db:
    image: postgres:16-alpine   # could ALSO be a different technology entirely
    environment: [POSTGRES_DB=orders_query]
```

Notice `command-db` and `query-db` are **completely separate database instances**, even though both currently use PostgreSQL — in a more advanced setup, `query-db` could be swapped for Elasticsearch (for search) or a columnar store (for analytics) without touching the Command side at all.

---

## 25. Kafka/Redis Integration

This entire module's Projection mechanism (Sections 22-23) IS the Kafka integration — CQRS and Event-Driven Architecture (Module 9) are deeply complementary patterns. Redis can serve as an even faster Query Model for simple, high-volume lookups:

```javascript
// An alternative, Redis-backed Query Model for VERY high-read-volume
// scenarios (e.g., a "recent orders" widget checked on every page load)
export async function upsertOrderSummaryRedis(summary) {
  await redis.set(
    `order-summary:${summary.orderId}`,
    JSON.stringify(summary)
  );
  // Also maintain a sorted set for "customer's orders, most recent first"
  await redis.zadd(
    `customer-orders:${summary.customerId}`,
    Date.parse(summary.placedAt),
    summary.orderId
  );
}
```

---

## 26. Error Handling

If the Projection fails to process an event (e.g., `getCustomerName` fails due to `user-service` being temporarily down), it should **retry** rather than silently skip the update, since skipping would leave the Query Model permanently out of sync for that order:

```javascript
export async function handleOrderPlacedProjection(event, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      const customerName = await getCustomerName(event.payload.customerId);
      await upsertOrderSummary({ ...event.payload, customerName });
      return;
    } catch (err) {
      if (attempt === maxRetries) {
        // Route to a Dead Letter Topic (Module 15's philosophy) for
        // later reprocessing, rather than silently losing this update
        await publishToDeadLetterTopic("order-events-dlq", event, err.message);
        return;
      }
      await new Promise((r) => setTimeout(r, 500 * attempt));
    }
  }
}
```

---

## 27. Logging & Monitoring

- Monitor **Projection lag** (Module 9's consumer lag concept) as the primary metric for how "eventually consistent" your Query Model currently is — surface this to stakeholders so they understand the actual, current staleness window.
- Log Command and Query operations **separately**, since they now have genuinely different performance characteristics and failure modes worth tracking independently.
- Track the **ratio of Command to Query traffic** — this data justifies (or challenges) your CQRS investment, and informs how aggressively to scale each side independently.

---

## 28. Security Considerations

- Command endpoints (which change state) require stricter authentication/authorization checks (Module 13) than Query endpoints, which are read-only and typically lower-risk — but don't neglect Query-side authorization entirely (a user should only be able to query their own order history, for instance).
- Since the Query Model may be denormalized and contain data "flattened" from multiple sources, be deliberate about which fields are included — avoid inadvertently duplicating sensitive data (Module 14's security considerations) into a read model with potentially broader or less-audited access than the original source.

---

## 29. Performance Optimization

- Design the Query Model's schema **specifically** around the exact queries it needs to serve — this is the entire point of CQRS, so don't stop short of true denormalization out of habitual normalization instincts from traditional relational design.
- Consider multiple, differently-shaped Query Models derived from the **same** Command Model and event stream (Section 12.3) — each optimized for a specific, distinct read need, rather than trying to make one "universal" read model serve every possible query well.
- Index the Query Model aggressively for its specific known access patterns — since it's not used for writes, you don't need to worry about write-amplification from excessive indexing the way you would on the Command Model.

---

## 30. Production Best Practices

- Apply CQRS **selectively** — to the specific services/entities where read and write patterns are genuinely, measurably different, not as a default architecture for every service in your system.
- Always design the Query Model to be **rebuildable from the event log** (Section 18) — this is one of CQRS's most valuable properties and should be a deliberate design goal, not an afterthought.
- Clearly document, per service, which side (Command or Query) a given endpoint belongs to, and enforce this separation in code structure/naming (as shown in Section 22-23's explicit folder separation).
- Monitor and alert on Projection lag thresholds appropriate to your business's tolerance for read staleness — a customer dashboard might tolerate seconds of lag; a real-time trading system might not.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Applying CQRS to simple CRUD services by default** | Adds significant complexity (separate models, projections, eventual consistency handling) with no corresponding benefit for simple, low-scale read/write patterns |
| **Allowing Command handlers to also serve queries directly** | Blurs the separation, reintroducing the exact compromise CQRS was meant to eliminate |
| **The Query Model being writable by anything other than the Projection** | Creates ambiguity about the source of truth and risks the Query Model diverging from the Command Model's actual history |
| **No mechanism to rebuild the Query Model from events** | Forfeits one of CQRS's most valuable resilience properties; a corrupted read model becomes a genuine data-loss incident instead of a quick rebuild |
| **Ignoring the eventual consistency UX implications** | Users seeing "their order isn't in the history yet" immediately after placing it, with no explanation, creates a confusing, poor user experience |

```
CQRS applied unnecessarily (anti-pattern):

  A simple internal "Team Members" admin CRUD service, with
  low traffic, simple reads (a basic list/detail view), and
  no meaningful performance problem

  Applying full CQRS here means: two databases, a projection
  consumer, eventual consistency handling, more deployment
  complexity - ALL FOR NO ACTUAL BENEFIT, since simple CRUD
  was already serving this need perfectly well
```

---

## 32. Debugging Tips

- If the Query Model shows stale or missing data, check **Projection consumer lag** first (Module 9's debugging techniques apply directly) before assuming a deeper logic bug.
- If Command and Query results seem inconsistent in a way that persists (not just eventual-consistency lag), verify nothing is writing to the Query Model **outside** of the Projection — a common bug is an emergency hotfix or admin tool that bypasses the proper flow.
- When testing, always account for the eventual consistency window — a test that writes a Command and immediately queries the Query Model without waiting/polling for projection completion will be flaky, not actually broken.

---

## 33. Interview Questions

### Easy
1. What is CQRS, and what problem does it solve?
2. What is the difference between a Command and a Query?
3. Why might a Query Model be denormalized while a Command Model is normalized?
4. What is a Projection in the context of CQRS?
5. What does it mean for the Query Model to be "eventually consistent" with the Command Model?

### Medium
6. How would you decide whether a specific service genuinely needs CQRS, versus simple CRUD being sufficient?
7. Explain how CQRS enables independent scaling of reads and writes.
8. Why can the Query Model typically be rebuilt from the event log, and why is this valuable?
9. What UX challenges does CQRS's eventual consistency introduce, and how would you mitigate them?
10. How does CQRS relate to and build upon Module 14's Data Duplication pattern?

### Hard
11. Design a CQRS system with THREE different Query Models derived from the same Command Model, each serving a genuinely different read need.
12. Explain how you would test a CQRS system's correctness given the inherent eventual consistency between Command and Query sides.
13. A Projection bug corrupts the Query Model for a subset of orders. Design a recovery plan using the event log.
14. Discuss the trade-offs of using a completely different database technology for the Query Model versus keeping it in the same technology as the Command Model for simplicity.
15. How would you design read-after-write consistency for a specific, sensitive UX flow (e.g., "confirm your order was placed") within an otherwise eventually-consistent CQRS system?

---

## 34. Scenario-Based Questions

1. Your customer dashboard, built on a single normalized model, has become slow under load due to complex joins and aggregations on every page view. How would you evaluate whether CQRS is the right fix?
2. After implementing CQRS, customers occasionally report "I placed an order but don't see it in my history yet." How do you explain this, and what mitigation would you propose?
3. A junior engineer wants to apply CQRS to every single service in your system "for consistency and best practice." How do you respond?
4. Your Query Model's database experiences an outage, but writes (Command side) continue succeeding. How does this reflect CQRS's architectural benefits, and what would customers experience during this outage?
5. You discover an admin tool has been directly writing to the Query Model's database to "quickly fix" some display data. What risks does this pose, and what would you recommend instead?

---

## 35. Hands-on Exercises

1. For a "Blog Platform" (Posts, Comments), identify whether CQRS would genuinely help, and if so, design the Command Model and at least one denormalized Query Model.
2. Implement the Command side (Section 22-23) for placing an order, including the Order Aggregate's business rule enforcement.
3. Implement the Query side's Projection (Section 22-23), and verify that placing an order eventually results in a correctly denormalized row in the Query Model.
4. Write a test that correctly WAITS for projection completion (rather than immediately querying) to avoid a flaky eventual-consistency-related test failure.
5. Design (on paper) a Query Model rebuild procedure: given a corrupted read model and a retained Kafka event log, outline the exact steps to restore it correctly.

---

## 36. Mini Project

**Build: A Basic CQRS Order System**

1. Build `order-command-service` with a `PlaceOrder` command handler (Section 22), persisting to a normalized Command Model database and publishing `OrderPlaced` events.
2. Build `order-query-service` with a Projection consumer building a denormalized `order_summary_view`, and a `GetOrderHistory` query endpoint serving directly from it.
3. Demonstrate the full flow: place an order via the Command endpoint, then poll the Query endpoint until the (eventually consistent) summary appears.
4. Add Projection lag monitoring/logging, and intentionally introduce artificial delay to observe and log the eventual consistency window in action.

---

## 37. Advanced Project

**Build: Multiple Query Models From One Command Model**

1. Extend the Mini Project: from the SAME `OrderPlaced` event stream, build TWO separate Query Models — (a) a customer-facing `order_summary_view` (as in the Mini Project), and (b) an admin-facing `daily_revenue_view` that aggregates total revenue and order count PER DAY.
2. Implement both Projections as independent consumers (separate consumer groups, Module 9), demonstrating that they can process the same event stream at their own independent pace.
3. Simulate a Query Model corruption: manually corrupt the `order_summary_view` table, then implement and execute a rebuild procedure that truncates it and replays all historical events from Kafka (`fromBeginning: true`) to restore it correctly.
4. Add a third Query Model using a DIFFERENT database technology (e.g., Redis for a simple "recent orders" widget, as in Section 25), demonstrating Polyglot Persistence (Module 14) applied specifically to CQRS's Query side.
5. Write a short document explaining, for each of the three Query Models, why its specific shape and technology choice fits its specific read need.

---

## 38. Summary

- CQRS separates the model used for writing (Command Model, optimized for correctness and business rule enforcement) from the model(s) used for reading (Query Model, optimized for specific read/query performance needs).
- Commands change state and return only acknowledgment; Queries read data and never change state — a direct extension of Module 9's Command vs. Event distinction.
- The Query Model is kept up to date asynchronously via a Projection consuming events published by the Command side, introducing an explicit, deliberate eventual consistency window.
- CQRS enables independent scaling, independent technology choices (Polyglot Persistence), and even multiple differently-shaped Query Models derived from the same underlying Command Model and event stream.
- CQRS should be applied **selectively**, only where read and write patterns are genuinely, measurably different — applying it universally as a default is a well-known anti-pattern that adds unjustified complexity.

---

## 39. Revision Notes

- CQRS: separate Command Model (writes, normalized) from Query Model (reads, denormalized).
- Command: changes state, returns acknowledgment only. Query: reads data, no side effects.
- Projection: consumes events, updates the Query Model — the ONLY thing allowed to write to it.
- Query Model can use a different database technology (Polyglot Persistence) and can be rebuilt from the event log.
- Eventual consistency between Command and Query sides is explicit and must be handled deliberately in UX.
- Apply CQRS selectively — not as a default architecture for every service.

---

## 40. One-Page Cheat Sheet

```
CQRS:                separate Command Model (write) from Query Model (read)
COMMAND:             changes state, returns ONLY acknowledgment
QUERY:               reads data, NO side effects, served ENTIRELY from Query Model
PROJECTION:          consumes events, updates Query Model - the ONLY writer to it
COMMAND MODEL:       normalized, optimized for correctness/business rules
QUERY MODEL:         denormalized, optimized for read speed, CAN use different DB tech

KEY BENEFIT:         independent scaling + independent optimization of reads vs writes
KEY TRADE-OFF:       eventual consistency between write and read - must be handled in UX
KEY RESILIENCE WIN:  Query Model can be REBUILT from the event log if corrupted

GOLDEN RULES:
  - Apply CQRS only where read/write patterns are GENUINELY different - not by default
  - NEVER let anything but the Projection write to the Query Model
  - Design the Query Model to be REBUILDABLE from the event log
  - Monitor Projection lag - it IS your current eventual consistency window
  - Handle read-after-write UX deliberately, don't let it surprise users
```

---

**Suggested Next Module:** Module 17 — Event Sourcing (storing state as a sequence of events rather than current-state snapshots, and how it pairs naturally with CQRS)
