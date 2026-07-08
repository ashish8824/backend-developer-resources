# Module 15 — Distributed Transactions (Saga Pattern)

> **Microservices Masterclass** | Level: Advanced | Track: Node.js Backend Engineering
> Prerequisite: Module 1–14 (especially Module 9 — Event-Driven Architecture, Module 14 — Database per Service)
> Next Module: Module 16 — CQRS

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

Module 14 established that each service owns its own database, and that data crossing service boundaries becomes eventually consistent. This raises a hard question that every serious microservices system must eventually confront: **what happens when a single business operation needs to update data in multiple services, and one of those updates fails partway through?**

In a monolith, this was solved trivially — a single database transaction (`BEGIN`, multiple writes, `COMMIT` or `ROLLBACK`) guarantees all-or-nothing atomicity, backed by ACID guarantees. In microservices, there is **no single database transaction that spans multiple services** — you cannot `BEGIN TRANSACTION` across Order DB and Payment DB and Inventory DB simultaneously. This module introduces the **Saga pattern**, the standard, battle-tested answer to this exact problem: how to achieve business-level consistency across multiple services without a distributed ACID transaction.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain why traditional ACID transactions don't work across microservice boundaries.
- Define the Saga pattern and explain how it achieves eventual, business-level consistency.
- Distinguish Choreography-based Sagas from Orchestration-based Sagas, and know when to use each.
- Design compensating transactions to undo the effects of already-completed steps when a later step fails.
- Implement a working Saga (both choreographed and orchestrated) in Node.js.
- Recognize Saga-specific failure modes and anti-patterns, including non-idempotent compensations.

---

## 3. Problem Statement

An e-commerce "Place Order" operation needs to: (1) reserve inventory, (2) charge the customer's payment method, and (3) create a shipping record — three separate services, three separate databases. Consider what happens if step 2 (payment) fails **after** step 1 (inventory reservation) has already succeeded:

- If nothing is done, the customer's inventory has been reserved (and is now unavailable to other customers) for an order that was never actually paid for — a real, damaging bug affecting real inventory availability.
- There is no way to wrap "reserve inventory" (in Inventory DB) and "charge payment" (in Payment DB) inside a single database transaction, since they're different databases, possibly different database technologies entirely (Module 14's Polyglot Persistence).
- The team needs a way to explicitly **undo** the inventory reservation if the payment step fails — but "undo" isn't a database rollback here; it's a deliberate, explicit business operation (e.g., "release the reserved inventory") that Inventory Service must support and expose.

This module solves exactly this: how to design a multi-step, multi-service business operation so that a failure partway through leaves the system in a correct, consistent (if not identical-to-success) state, using explicit compensating actions rather than an impossible distributed transaction rollback.

---

## 4. Why This Concept Exists

The Saga pattern exists because **traditional distributed transaction protocols (like Two-Phase Commit, or 2PC) don't fit well with microservices' core goals of independent scalability and availability.** 2PC requires all participating databases to lock their resources and wait for a central coordinator to decide whether to commit or abort — this creates tight coupling, blocking behavior, and a single point of coordination failure that directly contradicts the availability and independence benefits microservices are meant to provide (echoing the CAP theorem discussions throughout this masterclass: 2PC strongly favors Consistency at a severe cost to Availability and service independence). Sagas provide an alternative: **break the transaction into a sequence of local transactions, each within one service's own database, with explicit compensating actions to undo completed steps if a later step fails** — trading strict, immediate atomicity for eventual, business-level consistency that's actually compatible with independently-deployable, independently-available services.

---

## 5. Historical Background

- **1987** — The Saga pattern was originally described in a database systems research paper by Hector Garcia-Molina and Kenneth Salem, addressing **long-lived transactions** within a single database system that were impractical to lock for their full duration — the original context predates microservices entirely.
- **2000s** — Two-Phase Commit (2PC) and other distributed transaction protocols (like XA transactions) were used in some SOA and distributed systems, but were widely observed to create tight coupling, poor scalability, and fragility under partial failures — lessons the microservices community absorbed directly.
- **2010s** — As microservices matured, the original Saga concept was **rediscovered and adapted** specifically for the multi-service transaction problem, popularized extensively by Chris Richardson (author of "Microservices Patterns") and others as the standard alternative to distributed ACID transactions in this context.
- **Mid-2010s onward** — Two distinct implementation styles crystallized: **Choreography** (services react to each other's events independently, no central coordinator — a natural extension of Module 9's Event-Driven Architecture) and **Orchestration** (a dedicated coordinator service explicitly directs each step) — both now widely used, chosen based on the complexity of the specific business process.

---

## 6. Real-World Analogy

**Analogy: Booking a Multi-Part Vacation Package (Flight + Hotel + Rental Car)**

Imagine booking a vacation package involving three completely independent companies: an airline, a hotel, and a car rental agency. There's no single "vacation transaction system" that can atomically book all three or none at all — each company has its own separate booking system, entirely outside your control.

If you successfully book the flight and the hotel, but the rental car agency has no cars available, you can't "roll back" the flight and hotel bookings the way a database transaction would — instead, you must **explicitly cancel** the flight booking and **explicitly cancel** the hotel booking (assuming their systems support cancellation, which is the **compensating action** for each). Each cancellation is its own separate operation, with its own rules (perhaps the airline charges a cancellation fee — a real-world example of a compensation not perfectly "undoing" the original action, just addressing its business consequences).

- If you personally coordinate all three bookings yourself, checking each one and manually deciding when to cancel — that's **Orchestration** (a central coordinator, you, directing each step).
- If instead each company automatically watched for "trip cancelled" notifications from a shared trip-planning platform and independently cancelled their own booking — that's **Choreography** (each participant reacts independently, no central coordinator).

---

## 7. Technical Definition

> A **Saga** is a sequence of local transactions, each executed within a single service (and its own database), where each local transaction publishes an event or triggers the next step, and where **compensating transactions** are defined to semantically undo the effects of previously-completed steps if a later step in the sequence fails.

> A **Compensating Transaction** is an explicit, service-provided operation that reverses the business effect of a previously-completed local transaction — it is **not** a database rollback (which isn't possible across service boundaries), but a deliberate, forward-moving business operation (e.g., "release reserved inventory," "refund payment," "cancel shipment").

> **Choreography-based Saga**: each service, upon completing its local transaction, publishes an event; other services independently subscribe to and react to these events (including triggering their own compensations), with **no central coordinator** directing the overall flow.

> **Orchestration-based Saga**: a dedicated **Saga Orchestrator** component explicitly issues commands to each participating service in sequence, tracks the overall Saga's state, and explicitly triggers compensating actions on prior steps if a later step fails.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Saga** | A sequence of local transactions across services with defined compensations for failure |
| **Local Transaction** | A transaction scoped to a single service's own database — normal ACID guarantees apply here |
| **Compensating Transaction** | An explicit operation that semantically undoes a previously-completed step |
| **Choreography** | Services react to each other's events independently; no central coordinator |
| **Orchestration** | A dedicated coordinator explicitly directs each step and manages overall Saga state |
| **Saga Orchestrator** | The component (often itself a service) responsible for directing an orchestrated Saga |
| **Pivot Transaction** | The point in a Saga after which the operation is guaranteed to eventually succeed (compensations only happen before this point, retries happen after) |
| **Semantic Lock** | A business-level "reservation" (e.g., marking inventory as "reserved" rather than fully "deducted") used to avoid conflicting concurrent operations during a Saga's in-progress window |
| **Idempotent Compensation** | A compensating action safe to execute more than once without unintended side effects (essential given at-least-once event delivery, Module 9) |

---

## 9. Internal Working

Here's how a Saga works end-to-end, contrasting both implementation styles for the same "Place Order" scenario.

**Choreography-based Saga:**
1. `order-service` creates an order in a `PENDING` state (its own local transaction) and publishes `OrderCreated`.
2. `inventory-service` consumes `OrderCreated`, attempts to reserve stock (its own local transaction), and publishes either `StockReserved` or `StockReservationFailed`.
3. `payment-service` consumes `StockReserved`, attempts to charge the customer (its own local transaction), and publishes either `PaymentProcessed` or `PaymentFailed`.
4. If `PaymentFailed` is published, `inventory-service` (independently subscribing to this event) triggers its own **compensating transaction**: releasing the previously-reserved stock.
5. `order-service` (also independently subscribing to relevant events) updates the order's status to `CONFIRMED` (on full success) or `CANCELLED` (on any failure requiring compensation).
6. No single component "knows" the full Saga's state at any one point — each service simply reacts to the events relevant to it.

**Orchestration-based Saga:**
1. A dedicated **Order Saga Orchestrator** receives the "place order" request and explicitly tracks the Saga's state.
2. The Orchestrator sends a `ReserveStock` **command** to `inventory-service`, and waits for a response.
3. On success, the Orchestrator sends a `ChargePayment` command to `payment-service`, and waits for a response.
4. If `ChargePayment` fails, the Orchestrator explicitly sends a `ReleaseStock` **compensating command** back to `inventory-service`.
5. The Orchestrator updates the order's final status and is the **single source of truth** for "where is this Saga right now" at every point in the process.

---

## 10. Step-by-Step Request Flow

**Scenario: Choreography-based Saga for "Place Order," including a payment failure and compensation.**

```
Step 1:  order-service creates order (status: PENDING), publishes OrderCreated
Step 2:  inventory-service consumes OrderCreated, reserves stock (local
         transaction), publishes StockReserved
Step 3:  payment-service consumes StockReserved, attempts to charge
         the customer — the charge FAILS (e.g., insufficient funds)
Step 4:  payment-service publishes PaymentFailed
Step 5:  inventory-service consumes PaymentFailed, triggers its
         COMPENSATING transaction: releases the previously reserved
         stock (local transaction), publishes StockReleased
Step 6:  order-service consumes PaymentFailed, updates the order's
         status to CANCELLED (local transaction)
Step 7:  (Optionally) notification-service consumes PaymentFailed,
         sends the customer an email explaining the order was cancelled

RESULT: no distributed transaction rollback occurred — instead, each
service performed its OWN local transaction, and the FAILURE triggered
an explicit, forward-moving compensating action (releasing stock),
leaving the overall system in a correct, consistent BUSINESS state
(order cancelled, stock available again) even though no single
atomic operation spanned all 3 services
```

---

## 11. Architecture Overview

```
CHOREOGRAPHY-BASED SAGA:

  order-service ──OrderCreated──▶ Kafka ──▶ inventory-service
                                              │
                                     StockReserved / StockReservationFailed
                                              │
                                              ▼
                                          Kafka ──▶ payment-service
                                                       │
                                          PaymentProcessed / PaymentFailed
                                                       │
                                                       ▼
                              Kafka ──▶ inventory-service (compensates if failed)
                              Kafka ──▶ order-service (updates final status)


ORCHESTRATION-BASED SAGA:

                        Order Saga Orchestrator
                     (tracks state, issues commands)
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
       inventory-service  payment-service   order-service
       (ReserveStock /     (ChargePayment /   (finalize order
        ReleaseStock)       - )                status)
```

---

## 12. ASCII Diagrams

### 12.1 Successful Saga vs Saga With Compensation

```
SUCCESSFUL SAGA (happy path):

  Reserve Stock ──▶ Charge Payment ──▶ Create Shipment ──▶ SAGA COMPLETE
     (success)         (success)          (success)


SAGA WITH COMPENSATION (payment fails):

  Reserve Stock ──▶ Charge Payment ──▶ FAILS
     (success)        (FAILS)
        │
        ▼
  COMPENSATE: Release Stock ──▶ SAGA COMPLETE (compensated, order cancelled)
```

### 12.2 Choreography vs Orchestration

```
CHOREOGRAPHY (no central coordinator):

  Service A ──event──▶ Service B ──event──▶ Service C
       ▲                                          │
       └──────────compensating event, if needed────┘

  Pro: fully decoupled, no single point of failure
  Con: hard to see/track the OVERALL Saga's state at a glance;
       business logic ("what happens next") is scattered across
       multiple services' event handlers


ORCHESTRATION (central coordinator):

              Saga Orchestrator
           (explicitly tracks state)
          ┌────────┼────────┐
          ▼        ▼        ▼
      Service A  Service B  Service C
      (commands, responses - explicit, sequential)

  Pro: Saga's state and logic are centralized and easy to reason about
  Con: Orchestrator becomes a new critical component; must be
       highly available
```

### 12.3 Semantic Lock (Reserve, Not Deduct)

```
WITHOUT a semantic lock (risky):

  Step 1: Immediately DEDUCT stock (100 -> 98)
  Step 2: Payment FAILS
  Step 3: Must "add back" the stock (98 -> 100) — but what if
          ANOTHER order deducted stock in between? Race condition risk!


WITH a semantic lock (safer):

  Step 1: RESERVE stock (100 available, 2 RESERVED — not yet deducted)
  Step 2: Payment FAILS
  Step 3: RELEASE the reservation (2 reserved -> 0 reserved,
          100 still available) — no ambiguity, no race condition
          with other concurrent orders
```

---

## 13. Mermaid Flowcharts

### 13.1 Choreography vs Orchestration Decision

```mermaid
flowchart TD
    A[Designing a multi-service business transaction] --> B{How many steps/services are involved?}
    B -- "2-3 simple steps" --> C{Is the failure/compensation logic simple?}
    C -- Yes --> D[Choreography - simpler, fully decoupled]
    C -- No --> E[Consider Orchestration for clearer state tracking]
    B -- "4+ steps, complex branching" --> F[Orchestration - centralized, easier to reason about and monitor]
```

### 13.2 Saga Execution With Compensation Flow

```mermaid
flowchart TD
    A[Start Saga: Reserve Stock] --> B{Success?}
    B -- Yes --> C[Charge Payment]
    B -- No --> Z[Saga Failed - no compensation needed, nothing succeeded yet]
    C --> D{Success?}
    D -- Yes --> E[Create Shipment]
    D -- No --> F[Compensate: Release Stock]
    F --> Z
    E --> G{Success?}
    G -- Yes --> H[Saga Complete - Success]
    G -- No --> I[Compensate: Refund Payment]
    I --> F
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Orchestration-Based Saga With Compensation

```mermaid
sequenceDiagram
    participant O as Order Saga Orchestrator
    participant I as Inventory Service
    participant P as Payment Service

    O->>I: Command: ReserveStock
    I-->>O: Success: StockReserved
    O->>P: Command: ChargePayment
    P-->>O: Failure: PaymentFailed
    Note over O: Payment failed - must compensate the ALREADY completed step
    O->>I: Compensating Command: ReleaseStock
    I-->>O: Success: StockReleased
    O->>O: Mark Saga as FAILED (compensated), update order status
```

### 14.2 Choreography-Based Saga With Compensation

```mermaid
sequenceDiagram
    participant OS as Order Service
    participant K as Kafka
    participant IS as Inventory Service
    participant PS as Payment Service

    OS->>K: Publish OrderCreated
    K-->>IS: Consume OrderCreated
    IS->>IS: Reserve stock (local transaction)
    IS->>K: Publish StockReserved
    K-->>PS: Consume StockReserved
    PS->>PS: Attempt charge - FAILS
    PS->>K: Publish PaymentFailed
    K-->>IS: Consume PaymentFailed
    IS->>IS: Release stock (COMPENSATING transaction)
    IS->>K: Publish StockReleased
    K-->>OS: Consume PaymentFailed
    OS->>OS: Update order status to CANCELLED
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│              Order Saga Orchestrator (if used)               │
│  ┌───────────────────┐                                      │
│  │  Saga State Store      │  <- tracks: which steps completed,│
│  │  (its own database)     │     which compensations may be    │
│  │                          │     needed, current Saga status   │
│  └─────────┬───────────┘                                    │
│            ▼                                                 │
│  ┌───────────────────┐                                      │
│  │  Step Coordinator      │  <- issues commands, handles        │
│  │  Logic                   │     responses, triggers            │
│  │                          │     compensations on failure        │
│  └───────────────────┘                                      │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                               │
│  order-saga-orchestrator pods (if using Orchestration)         │
│  ──▶ own database (Saga state) + calls to Inventory/Payment    │
│                                                               │
│  OR, for Choreography:                                        │
│                                                               │
│  order-service, inventory-service, payment-service pods         │
│  ──▶ each independently consumes/produces to Kafka topics        │
│      (order-events, inventory-events, payment-events)            │
│      NO dedicated orchestrator deployment needed                  │
└───────────────────────────────────────────────────────────┘
```

---

## 17. Database Interaction

Each step of a Saga is a **local transaction**, fully respecting Module 14's Database per Service principle — the Saga pattern is specifically the technique for coordinating across these independently-owned databases without violating that principle:

```
inventory-service's LOCAL transaction (reserve stock):
  BEGIN;
  UPDATE products SET reserved_quantity = reserved_quantity + 2 WHERE id = 'abc';
  INSERT INTO reservations (order_id, product_id, quantity) VALUES (...);
  COMMIT;

  This is a NORMAL ACID transaction, but SCOPED ENTIRELY to
  inventory-service's own database — no cross-service transaction exists

payment-service's LOCAL transaction (charge, and if it fails, nothing to commit):
  BEGIN;
  -- attempt charge via external payment processor
  -- on failure: ROLLBACK (a normal, LOCAL rollback within payment-service alone)
  COMMIT; -- only if the charge succeeded
```

The Saga's compensation for a failed payment is **inventory-service's own separate local transaction** (releasing the reservation) — never a cross-database rollback, which is impossible.

---

## 18. Failure Scenarios

| Scenario | Saga Handling |
|---|---|
| A step fails before any prior steps completed | No compensation needed — nothing has happened yet that requires undoing |
| A step fails after prior steps completed | Trigger compensating transactions for each already-completed prior step, in reverse order |
| A compensating transaction itself fails | This is a serious failure mode requiring retry logic, alerting, and potentially manual intervention — compensations should be designed to be as reliable and simple as possible, and ideally idempotent |
| The Saga Orchestrator crashes mid-Saga (Orchestration style) | The Orchestrator must persist its Saga state durably (its own database) so it can resume tracking and completing/compensating the Saga correctly after restarting |
| Duplicate event delivery causes a compensation to run twice (Choreography style) | Compensating transactions MUST be idempotent (Module 9's principle applied here) — e.g., "release stock" should be safe to call twice without releasing more stock than was actually reserved |

```
Compensation itself failing (a serious edge case):

  Payment fails -> attempt to release reserved stock -> 
  inventory-service is TEMPORARILY DOWN
           │
           ▼
  The compensating action FAILS to execute
           │
           ▼
  Mitigation: retry the compensation with backoff (Module 7's
  retry patterns); if retries are exhausted, alert a human and/or
  route to a manual reconciliation process — this is one of the
  few places in a well-designed system where manual intervention
  may genuinely be the appropriate fallback
```

---

## 19. Scalability Discussion

Sagas, by design, avoid the blocking, lock-holding behavior of distributed transaction protocols like 2PC — each local transaction completes and commits quickly within its own service, without holding locks across service/network boundaries while waiting for other services to respond. This makes Sagas dramatically more scalable under load than any approach requiring cross-service locking, at the cost of the business needing to tolerate a temporary window of inconsistency (e.g., stock briefly "reserved" for an order that ultimately fails) during the Saga's execution.

---

## 20. High Availability Considerations

- Choreography-based Sagas inherit the high availability characteristics of Event-Driven Architecture (Module 9) — no single component's availability gates the entire Saga's ability to proceed (though a specific step's processing does wait on its own relevant consumer being healthy).
- Orchestration-based Sagas introduce the Orchestrator as a new, critical component — it must be deployed with multiple replicas and durable, persisted state (so a crash mid-Saga doesn't lose track of in-progress transactions) to avoid becoming a single point of failure for the entire business process.
- Regardless of style, compensating transactions should be designed with **retry logic** (Module 7) since the service needing to execute a compensation might itself be temporarily unavailable when the compensation is first attempted.

---

## 21. CAP Theorem Implications

The Saga pattern is a direct, practical embodiment of choosing **Availability and Partition Tolerance** over the strict, immediate **Consistency** that a true distributed ACID transaction (2PC) would provide — the system accepts a temporary window where different services' data is not perfectly consistent with each other (e.g., stock is "reserved" but the overall order hasn't yet been confirmed or cancelled), in exchange for avoiding the blocking, availability-reducing behavior of 2PC-style coordination. This is, in effect, the most advanced and most consequential application of the eventual consistency theme running throughout this entire masterclass (Modules 4, 6, 9, 14) — Sagas are what makes that theoretical trade-off concretely workable for genuinely complex, multi-step business processes.

---

## 22. Node.js Implementation

Let's implement an Orchestration-based Saga for "Place Order," including a compensating transaction.

**Folder structure:**
```
order-saga-orchestrator/
├── src/
│   ├── sagas/
│   │   └── placeOrderSaga.js
│   ├── db/
│   │   └── sagaStateRepository.js
│   └── clients/
│       ├── inventoryClient.js
│       └── paymentClient.js
```

**`src/db/sagaStateRepository.js`** — durable Saga state (essential for orchestrator resilience, Section 18)
```javascript
import { db } from "./connection.js";

export async function createSagaState(sagaId, orderId) {
  await db.query(
    `INSERT INTO saga_state (saga_id, order_id, status, current_step)
     VALUES ($1, $2, 'STARTED', 'RESERVE_STOCK')`,
    [sagaId, orderId]
  );
}

export async function updateSagaStep(sagaId, step, status) {
  await db.query(
    `UPDATE saga_state SET current_step = $2, status = $3 WHERE saga_id = $1`,
    [sagaId, step, status]
  );
}
```

**`src/clients/inventoryClient.js`**
```javascript
import axios from "axios";

export async function reserveStock(orderId, items) {
  const response = await axios.post(
    `${process.env.INVENTORY_SERVICE_URL}/reservations`,
    { orderId, items },
    { timeout: 3000, headers: { "Idempotency-Key": `reserve-${orderId}` } } // Module 7's idempotency
  );
  return response.data;
}

// The COMPENSATING transaction — a deliberate, explicit business
// operation, NOT a database rollback
export async function releaseStock(orderId) {
  await axios.post(
    `${process.env.INVENTORY_SERVICE_URL}/reservations/${orderId}/release`,
    {},
    { timeout: 3000, headers: { "Idempotency-Key": `release-${orderId}` } } // idempotent - safe to retry/duplicate
  );
}
```

**`src/sagas/placeOrderSaga.js`** — the orchestrator's core logic
```javascript
import { reserveStock, releaseStock } from "../clients/inventoryClient.js";
import { chargePayment, refundPayment } from "../clients/paymentClient.js";
import { createSagaState, updateSagaStep } from "../db/sagaStateRepository.js";

export async function runPlaceOrderSaga(orderId, items, customerId, amount) {
  const sagaId = crypto.randomUUID();
  await createSagaState(sagaId, orderId);

  // Step 1: Reserve Stock
  try {
    await reserveStock(orderId, items);
    await updateSagaStep(sagaId, "STOCK_RESERVED", "IN_PROGRESS");
  } catch (err) {
    await updateSagaStep(sagaId, "RESERVE_STOCK", "FAILED");
    throw new Error("Order failed: could not reserve stock");
    // No compensation needed - nothing succeeded yet
  }

  // Step 2: Charge Payment
  try {
    await chargePayment(customerId, amount, orderId);
    await updateSagaStep(sagaId, "PAYMENT_CHARGED", "IN_PROGRESS");
  } catch (err) {
    // COMPENSATE the already-completed Step 1
    await releaseStock(orderId);
    await updateSagaStep(sagaId, "COMPENSATED", "FAILED");
    throw new Error("Order failed: payment declined, stock reservation released");
  }

  // Step 3: Finalize (in a real system, this might involve Shipping too)
  await updateSagaStep(sagaId, "COMPLETED", "SUCCESS");
  return { sagaId, status: "SUCCESS" };
}
```

---

## 23. Express.js Examples

```javascript
// order-saga-orchestrator/src/app.js
import express from "express";
import { runPlaceOrderSaga } from "./sagas/placeOrderSaga.js";

const app = express();
app.use(express.json());

app.post("/orders", async (req, res) => {
  const { orderId, items, customerId, amount } = req.body;

  try {
    const result = await runPlaceOrderSaga(orderId, items, customerId, amount);
    res.status(201).json(result);
  } catch (err) {
    // The Saga either failed with no compensation needed, or failed
    // AND successfully compensated — either way, this is a clean,
    // handled business outcome, not an unhandled system error
    res.status(400).json({ error: err.message });
  }
});

app.listen(4010, () => console.log("Order Saga Orchestrator running on port 4010"));
```

**Choreography-style equivalent — the compensating consumer in `inventory-service`:**
```javascript
// inventory-service: reacts independently to a PaymentFailed event,
// with NO orchestrator directing it — this IS the choreography style
import { releaseStock } from "../services/inventoryService.js";

export async function handlePaymentFailed(event) {
  // MUST be idempotent - this event might be delivered more than once
  await releaseStock(event.payload.orderId);
}
```

---

## 24. Docker Examples

```yaml
version: "3.9"
services:
  order-saga-orchestrator:
    build: ./order-saga-orchestrator
    ports: ["4010:4010"]
    environment:
      - DATABASE_URL=postgresql://user:pass@saga-db:5432/sagas
      - INVENTORY_SERVICE_URL=http://inventory-service:4004
      - PAYMENT_SERVICE_URL=http://payment-service:4003
    depends_on: [saga-db, inventory-service, payment-service]

  inventory-service:
    build: ./inventory-service
    ports: ["4004:4004"]

  payment-service:
    build: ./payment-service
    ports: ["4003:4003"]

  saga-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=sagas]
```

---

## 25. Kafka/Redis Integration

For a **Choreography-based Saga**, Kafka is the entire backbone — every step and every compensation flows through published/consumed events, exactly as demonstrated throughout Module 9. For an **Orchestration-based Saga**, the Orchestrator might still use Kafka to publish a final `OrderSagaCompleted` or `OrderSagaFailed` event for other, unrelated services to consume asynchronously (e.g., Analytics), even though the core Saga steps themselves are driven by direct commands/calls rather than events:

```javascript
// After the orchestrator completes (success or compensated failure),
// publish a final event for OTHER interested services
await kafkaProducer.send({
  topic: "order-saga-events",
  messages: [{
    key: orderId,
    value: JSON.stringify({ type: "OrderSagaCompleted", orderId, status: "SUCCESS" }),
  }],
});
```

---

## 26. Error Handling

Compensating transactions themselves need robust error handling, since they're executing **during** an already-degraded situation (something already failed):

```javascript
async function compensateWithRetry(compensationFn, maxAttempts = 5) {
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      await compensationFn();
      return; // compensation succeeded
    } catch (err) {
      if (attempt === maxAttempts) {
        // Exhausted retries - this requires alerting/manual intervention,
        // one of the few legitimate cases for a human-in-the-loop fallback
        await alertOpsTeam("Saga compensation failed after max retries", err);
        return;
      }
      await new Promise((r) => setTimeout(r, 1000 * attempt));
    }
  }
}

// Usage in the Saga:
await compensateWithRetry(() => releaseStock(orderId));
```

---

## 27. Logging & Monitoring

- Log **every Saga step transition** (started, succeeded, failed, compensating, compensated) with the Saga ID, enabling full reconstruction of any Saga's history for debugging or auditing.
- Monitor **Saga failure/compensation rates** as a key business health metric — a rising rate of payment-triggered compensations, for instance, might indicate a payment processor issue worth investigating separately.
- Alert explicitly on **compensation failures** (Section 26) — these represent situations where the system may be left in a genuinely inconsistent state requiring human attention.

```javascript
logger.info({ sagaId, step: "RESERVE_STOCK", status: "SUCCESS", orderId }, "Saga step completed");
logger.error({ sagaId, step: "RELEASE_STOCK", status: "COMPENSATION_FAILED", orderId }, "Saga compensation failed");
```

---

## 28. Security Considerations

- Compensating transaction endpoints (e.g., `releaseStock`, `refundPayment`) are sensitive operations — they should be authenticated as legitimate internal service-to-service calls (Module 13), not accessible to arbitrary callers, since triggering an unauthorized compensation could itself be exploited (e.g., maliciously releasing a competitor's reserved stock).
- Idempotency keys (used throughout this module's compensation calls) also serve a security purpose: preventing a replayed or duplicated malicious request from triggering unintended repeated compensations.

---

## 29. Performance Optimization

- Keep each **local transaction** within a Saga step as fast and simple as possible — Sagas work best when each step commits quickly, minimizing the window during which the overall business process is in an intermediate, not-yet-finalized state.
- For Orchestration-based Sagas, consider running independent, non-dependent steps in **parallel** where the business logic allows (e.g., if reserving stock and validating a shipping address don't depend on each other, do both simultaneously) rather than always executing strictly sequentially.
- Persist Saga state efficiently (Section 22) — this write happens on every step transition, so ensure it doesn't become a bottleneck under high Saga throughput.

---

## 30. Production Best Practices

- Choose **Choreography** for simple Sagas (2-3 steps, straightforward compensation logic) where full decoupling is valuable, and **Orchestration** for complex Sagas (4+ steps, complex branching, a need for clear, centralized visibility into "where is this business process right now") — document this decision per Saga, as recommended in Module 6's sync/async framework.
- Design **every** compensating transaction to be **idempotent** from day one — assume it will be called more than once due to retries or duplicate event delivery (Module 9).
- Persist Orchestrator state durably and design for orchestrator restarts to correctly resume in-progress Sagas — never keep Saga state only in memory.
- Build dashboards/tooling to inspect the state of any given Saga by its ID — this becomes essential operational tooling once your system runs many concurrent Sagas.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Attempting a real distributed transaction (2PC) across microservices** | Creates tight coupling, blocking behavior, and availability risk that directly contradicts microservices' core goals |
| **Non-idempotent compensating transactions** | Given at-least-once event delivery / retry semantics, a non-idempotent compensation risks double-releasing stock, double-refunding payment, etc. |
| **No persisted Saga state (Orchestration style)** | An orchestrator crash mid-Saga loses track of in-progress transactions, potentially leaving the system in an unrecoverable, inconsistent state |
| **Treating a Saga's local transactions as if they were still one big atomic unit** | Ignores the reality that OTHER operations can observe intermediate, in-progress states (e.g., "reserved but not yet confirmed" stock) — design for this visibility deliberately (semantic locks, Section 12.3) |
| **Overly complex Sagas with many steps and unclear compensation logic** | Becomes difficult to reason about, test, and debug — consider whether the business process could be simplified or split |

```
Non-idempotent compensation (anti-pattern):

  function releaseStock(orderId) {
    stock.available += reservation.quantity;  // BAD: if called TWICE
                                                // (due to a retry or
                                                // duplicate event),
                                                // stock is incorrectly
                                                // increased TWICE
  }

  FIX: check if a reservation for this orderId has ALREADY been
  released before incrementing again - make the operation
  idempotent by checking and updating reservation STATUS,
  not blindly incrementing a counter
```

---

## 32. Debugging Tips

- Use the **Saga ID** as your primary correlation key across all logs — every step, compensation, and final outcome for a given business operation should be traceable via this single ID.
- If a Saga appears "stuck," check whether it's an Orchestration-style Saga whose Orchestrator crashed without properly persisting/resuming state, or a Choreography-style Saga where a consumer is experiencing lag or has silently failed to process a critical event (Module 9's debugging techniques apply directly here).
- If you observe stock or balances that seem "off," check for a Saga that failed to compensate correctly — this is the most common root cause of subtle data inconsistency bugs in Saga-based systems.

---

## 33. Interview Questions

### Easy
1. What problem does the Saga pattern solve?
2. What is a compensating transaction?
3. What is the difference between Choreography-based and Orchestration-based Sagas?
4. Why can't you use a traditional ACID transaction across multiple microservices?
5. Why must compensating transactions be idempotent?

### Medium
6. Explain the trade-offs between Choreography and Orchestration for a 5-step business process.
7. What is a "semantic lock" in the context of Sagas, and why is "reserve" often preferable to "immediately deduct"?
8. How would you design compensation for a step that has no natural "undo" (e.g., an email that's already been sent)?
9. Why must an Orchestrator persist its Saga state durably rather than keeping it only in memory?
10. How does the Saga pattern relate to the CAP theorem trade-offs discussed throughout this masterclass?

### Hard
11. Design a full Saga (with compensations) for a "Book a Multi-City Trip" process involving Flight, Hotel, and Car Rental services.
12. What happens if a compensating transaction itself repeatedly fails, and how would you design a system to handle this gracefully?
13. Compare the observability/debugging experience of a Choreography-based Saga versus an Orchestration-based Saga, and how you'd instrument each for production visibility.
14. Design a Saga where two steps can safely execute in parallel, and explain how you'd handle a failure in one while the other is still in progress.
15. Discuss how you would migrate an existing, informally-handled multi-service transaction (with ad-hoc, inconsistent error handling) into a properly-designed Saga.

---

## 34. Scenario-Based Questions

1. Your team's "Place Order" flow occasionally leaves inventory "stuck" as reserved for orders that failed payment, and no one investigated why. Using this module's concepts, what would you check first?
2. Leadership asks whether your multi-service checkout flow can be made "fully ACID" like the old monolith was. How do you explain why this isn't practical, and what you're doing instead?
3. A Saga Orchestrator crashes mid-transaction during a deployment, and several in-progress orders are left in an ambiguous state. What does this reveal about your Saga implementation, and how would you fix it?
4. Your team is deciding between Choreography and Orchestration for a NEW 6-step "Employee Onboarding" business process spanning HR, IT, Payroll, and Facilities services. Which would you recommend, and why?
5. You discover that a compensating "refund payment" call has been triggered twice for the same failed order, resulting in a double refund. Diagnose the root cause and the fix.

---

## 35. Hands-on Exercises

1. Design (on paper) a Saga for a "Book a Hotel Room" process (Reserve Room, Charge Deposit, Send Confirmation), including compensations for each possible failure point.
2. Implement the Orchestration-based Saga from Section 22-23, including the compensating `releaseStock` call, and test both the happy path and the payment-failure path.
3. Implement the Choreography-based equivalent (Section 23's `handlePaymentFailed` consumer), and compare the amount and location of code required versus the Orchestration approach.
4. Write an idempotency check for the `releaseStock` compensation, ensuring it's safe to call multiple times for the same order.
5. Sketch the Saga state machine (all possible states and transitions) for a 4-step business process of your choosing.

---

## 36. Mini Project

**Build: An Orchestration-Based Saga With One Compensation**

1. Build `inventory-service` and `payment-service` as separate services, each with a simple in-memory or database-backed state.
2. Build `order-saga-orchestrator` implementing the Saga from Section 22, calling `reserveStock` then `chargePayment`, with `releaseStock` as the compensation if payment fails.
3. Persist Saga state in a database table (Section 22), and demonstrate that restarting the orchestrator mid-Saga (simulated) doesn't lose track of an in-progress transaction.
4. Test both paths: a successful Saga (both steps succeed) and a compensated Saga (payment fails, stock is released).

---

## 37. Advanced Project

**Build: A Full Choreography-Based Saga With Multiple Compensations**

1. Build 4 services: `order-service`, `inventory-service`, `payment-service`, `shipping-service`, all communicating via Kafka events (no central orchestrator).
2. Implement the full happy path: `OrderCreated` → `StockReserved` → `PaymentProcessed` → `ShipmentCreated` → order marked `CONFIRMED`.
3. Implement TWO failure/compensation scenarios: (a) payment fails after stock is reserved — compensate by releasing stock; (b) shipment creation fails after payment succeeds — compensate by BOTH refunding payment AND releasing stock (a multi-step compensation cascade).
4. Ensure every compensating consumer is idempotent, and write a test proving a duplicated compensation event doesn't cause incorrect behavior (e.g., double-refunding).
5. Build a simple "Saga Tracker" read model (a service or endpoint that consumes all relevant events and maintains a queryable view of "current status of Saga for order X") to address the Choreography style's inherent lack of centralized state visibility, and reflect on how this compares to the Orchestration approach's built-in state tracking.

---

## 38. Summary

- The Saga pattern coordinates a multi-step business transaction across multiple services' independently-owned databases (Module 14), without requiring a traditional distributed ACID transaction, which is impractical and availability-reducing in a microservices context.
- Each step of a Saga is a local transaction, fully ACID within its own service's database; cross-service consistency is achieved through explicit, forward-moving compensating transactions rather than rollbacks.
- Choreography-based Sagas use independent, event-reactive services with no central coordinator; Orchestration-based Sagas use a dedicated coordinator explicitly directing and tracking each step — the choice depends on the process's complexity and your need for centralized visibility.
- Compensating transactions must be idempotent, since at-least-once delivery and retries mean they may be executed more than once.
- Sagas represent the most advanced, practical application of the eventual consistency trade-off (favoring Availability over strict Consistency) that has run throughout this entire masterclass.

---

## 39. Revision Notes

- Saga: sequence of local transactions + compensating transactions, no distributed ACID transaction.
- Compensating transaction: explicit, forward-moving business operation to undo a completed step's effects.
- Choreography: event-reactive, no central coordinator, fully decoupled but harder to observe overall state.
- Orchestration: central coordinator directs/tracks each step, easier to observe but introduces a new critical component.
- Compensations MUST be idempotent (at-least-once delivery is the norm).
- Semantic locks (e.g., "reserve" not "deduct") avoid race conditions during a Saga's in-progress window.
- Sagas favor Availability over strict Consistency — an explicit, deliberate CAP theorem trade-off.

---

## 40. One-Page Cheat Sheet

```
SAGA:                    sequence of local transactions + compensations, no distributed ACID
LOCAL TRANSACTION:        normal ACID transaction, scoped to ONE service's own database
COMPENSATING TRANSACTION: explicit operation to undo a completed step's business effect
CHOREOGRAPHY:             event-reactive, no coordinator, fully decoupled
ORCHESTRATION:            central coordinator explicitly directs + tracks each step
SEMANTIC LOCK:            "reserve" not "deduct" - avoids race conditions mid-Saga

CHOOSE CHOREOGRAPHY WHEN:  simple Saga (2-3 steps), full decoupling valued
CHOOSE ORCHESTRATION WHEN: complex Saga (4+ steps), need centralized state visibility

GOLDEN RULES:
  - NEVER attempt a real distributed ACID transaction across services
  - EVERY compensating transaction MUST be idempotent
  - Persist Orchestrator state durably - never keep it only in memory
  - Use semantic locks (reserve/release) instead of immediate deduction
  - Alert on compensation FAILURES - this is a genuine "needs a human" scenario
```

---

**Suggested Next Module:** Module 16 — CQRS (Command Query Responsibility Segregation — separating read and write models for scalability and flexibility, building directly on this module's Saga and Module 14's read-model concepts)
