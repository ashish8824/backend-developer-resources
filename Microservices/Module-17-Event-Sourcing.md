# Module 17 — Event Sourcing

> **Microservices Masterclass** | Level: Advanced | Track: Node.js Backend Engineering
> Prerequisite: Module 1–16 (especially Module 9 — Event-Driven Architecture, Module 16 — CQRS)
> Next Module: Module 18 — Resilience Patterns

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

Every application you've built so far — in this masterclass and probably in your career — stores **current state**: a row in a database representing "here is what this order looks like right now." When the order's status changes from `PLACED` to `SHIPPED`, you `UPDATE` that row, and the fact that it was ever `PLACED` (and exactly when, and why) is simply **overwritten and lost**, unless you separately built an audit log for it.

**Event Sourcing** inverts this completely: instead of storing current state, you store the **full sequence of events** that led to that state — `OrderPlaced`, `PaymentProcessed`, `OrderShipped` — as the **only** source of truth, and current state becomes something you **compute** by replaying those events, not something you store directly at all. This module explores this genuinely different way of thinking about persistence, when it's a powerful fit (audit-heavy domains, complex state histories, natural pairing with CQRS from Module 16), and when it's a significant, unjustified complexity burden.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain Event Sourcing and how it fundamentally differs from traditional state-based persistence.
- Design an event-sourced Aggregate, including event replay to reconstruct current state.
- Implement snapshotting to optimize replay performance for long event histories.
- Explain how Event Sourcing pairs naturally with CQRS (Module 16) and Event-Driven Architecture (Module 9).
- Handle event schema evolution in an event-sourced system, where old events can never be discarded.
- Recognize when Event Sourcing's benefits (full audit history, temporal queries, natural event integration) justify its very real complexity costs.

---

## 3. Problem Statement

A financial services company needs to track every account balance change with **perfect historical accuracy** — not just "the current balance is $500," but "exactly what sequence of deposits, withdrawals, and fees produced this balance, in what order, at what timestamps, and why." Using traditional state-based storage:

- The `accounts` table stores only the current balance — a single `UPDATE` overwrites the previous value, and the **history of how it got there is gone**, unless a separate `audit_log` table is maintained painstakingly in parallel (and kept perfectly in sync, itself a real engineering burden).
- If a regulator or an internal investigation later asks "what was this account's exact balance at 3:47pm last Tuesday, and why," a state-based system has no natural way to answer this — you'd need to have anticipated this exact need and built point-in-time historical tracking separately.
- If a bug in the balance calculation logic is discovered months later, there's no way to "replay" the account's history with corrected logic to see what the balance *should* have been — the original sequence of operations was never preserved as first-class data, only their cumulative, already-applied effect.

Event Sourcing solves this directly: since the **event log itself is the source of truth**, you can always answer "what happened, in what order, and why," reconstruct state at **any point in time** by replaying events up to that point, and even recompute state with corrected logic by replaying the same historical events through updated calculation code.

---

## 4. Why This Concept Exists

Event Sourcing exists because **traditional state-based persistence discards information that is sometimes genuinely valuable, and that loss is irreversible** — once you `UPDATE` a row, the previous value (and the fact that it ever existed) is gone unless you separately, manually preserved it. For domains where the **history of how you arrived at the current state** matters as much as (or more than) the current state itself — financial ledgers, audit-heavy regulated industries, collaborative editing systems, inventory systems needing precise "why did stock change" traceability — Event Sourcing makes this history a **first-class, permanent, and complete** part of your data model, rather than an afterthought bolted on via a separate, often incomplete audit log.

---

## 5. Historical Background

- **Long-standing accounting practice** — Event Sourcing's core idea (recording every transaction as an immutable entry, never overwriting history) mirrors traditional **double-entry bookkeeping**, a practice centuries old in accounting — ledgers have always recorded a sequence of transactions rather than just a current balance, for exactly the auditability reasons this module discusses.
- **2005** — **Martin Fowler** wrote an influential early article specifically describing the "Event Sourcing" pattern in software architecture terms, connecting this accounting-like idea directly to application state management.
- **Late 2000s** — **Greg Young** (also central to CQRS's history, Module 16) extensively promoted and refined Event Sourcing as a software architecture pattern, frequently pairing it conceptually with CQRS — since an event-sourced Command Model naturally produces the exact event stream a CQRS Projection needs to build Query Models.
- **2010s** — As distributed systems and microservices matured, dedicated **Event Store** databases (e.g., **EventStoreDB**, purpose-built for this exact pattern) emerged, alongside general-purpose event brokers like Kafka being used as a practical (if not purpose-built) event log for this pattern.
- **Present** — Event Sourcing remains a specialized, powerful pattern used deliberately in domains with strong audit/history requirements (finance, healthcare, some e-commerce order-management systems) rather than a general-purpose default — its adoption has stayed intentionally narrower than CQRS's, reflecting the community's widely-shared caution about its real complexity costs.

---

## 6. Real-World Analogy

**Analogy: A Bank Ledger vs. an ATM's "Current Balance" Display**

The ATM screen shows you one number: "Current Balance: $500." That's **state-based** thinking — a single current value, with no visible history of how it got there.

But behind that screen, the **bank's actual ledger** doesn't work that way at all — it's an **append-only sequence** of every single transaction: "Deposit $1,000 (Jan 1)," "Withdrawal $200 (Jan 3)," "Fee $10 (Jan 5)," "Deposit $50 (Jan 10)," and so on. The bank **never** deletes or overwrites a past transaction to "simplify" the record — every entry is permanent. Your current balance ($500 + $1000 - $200 - $10 + $50 - previous entries... ) is simply the **computed result of replaying every transaction from the beginning** (or, in practice, from the last checkpoint/statement).

This is exactly Event Sourcing: the **ledger of events** (deposits, withdrawals, fees) is the actual source of truth — permanent, append-only, complete — while the "current balance" you see on the ATM screen is merely a **derived, computed view**, exactly analogous to a CQRS Query Model (Module 16) built by replaying/projecting the event history.

---

## 7. Technical Definition

> **Event Sourcing** is a persistence pattern where an entity's state is not stored directly; instead, every state-changing action is recorded as an immutable **event** in an **append-only event log**, and the entity's current state is derived by **replaying** (folding/reducing over) its complete sequence of events from the beginning (or from a snapshot, Section 8).

> An **Event Store** is a specialized (or repurposed) database optimized for storing an append-only, ordered sequence of events per entity (often called a **stream**), supporting efficient appending of new events and efficient retrieval of an entity's full event history for replay.

> A **Snapshot** is a periodically-saved, pre-computed current-state representation of an entity at a specific point in its event history, used to avoid replaying an entity's **entire** event history from the very beginning every single time its current state is needed — instead, you replay only the events **since** the last snapshot.

> **Event Replay** is the process of reconstructing an entity's current (or historical, at any past point in time) state by applying each of its events, in order, to an initial empty state, using the same business logic (a "reducer" function) that originally produced that state when each event was first created.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Event Sourcing** | Storing state as a sequence of events, not as directly-stored current state |
| **Event Store** | A database optimized for storing and retrieving append-only event streams |
| **Stream** | The complete, ordered sequence of events for one specific entity (e.g., one specific Order) |
| **Event Replay** | Reconstructing state by applying events in order, via a reducer function |
| **Reducer / Apply Function** | The business logic function that takes (current state, next event) and returns new state |
| **Snapshot** | A periodically-saved current-state checkpoint, avoiding full replay from the very beginning |
| **Append-Only** | Events are only ever added, never modified or deleted, once written |
| **Temporal Query** | A query for an entity's state **as of** some past point in time — a natural Event Sourcing capability |
| **Event Schema Evolution** | Since old events can never be discarded/rewritten, new event versions must be handled alongside old ones indefinitely |
| **Upcasting** | Transforming an old-version event into the shape a newer version of the reducer logic expects, at replay time |

---

## 9. Internal Working

Here's how Event Sourcing works end-to-end for an Order Aggregate (directly extending Module 4's DDD Aggregate concept):

1. Instead of `order-service` storing a current-state `orders` table row, it maintains an **Event Store**: for each Order (identified by its ID), a **stream** of every event that has ever happened to it — `OrderCreated`, `LineItemAdded`, `OrderPlaced`, `OrderShipped`, potentially `OrderCancelled`.
2. When a **Command** arrives (e.g., "add a line item to this order"), the Command Handler must first determine the order's **current state** — it does this by **loading the entire event stream** for that order and **replaying** it through a reducer function, folding each event into the growing current state.
3. With the current state now known, the Command Handler applies the business logic/validation (e.g., "can't modify an already-placed order" — Module 4's invariant enforcement) and, if valid, generates a **new event** (`LineItemAdded`) representing this change.
4. This new event is **appended** to the Order's event stream in the Event Store — never modifying or deleting any prior event, only ever adding a new one.
5. To avoid replaying an ever-growing, potentially very long event history from the beginning on every single command (a real performance concern), the system periodically saves a **Snapshot** — a pre-computed current state at a specific event sequence number — so future replays only need to process events **since** that snapshot.
6. Because these events are, in essence, exactly the Domain Events from Module 9, they can be **published externally** as Integration Events, feeding CQRS Query Model Projections (Module 16) directly and naturally — Event Sourcing and CQRS pair together almost seamlessly, since the event-sourced Command Model already produces exactly what a CQRS Projection needs.

---

## 10. Step-by-Step Request Flow

**Scenario: Adding a line item to an existing, event-sourced Order.**

```
Step 1:  Command arrives: AddLineItem { orderId: "123", productId: "abc", qty: 2 }

Step 2:  Command Handler loads the LATEST SNAPSHOT for order "123"
         (e.g., a snapshot at event sequence #40, representing state
         as of that point) - avoiding a full replay from event #1

Step 3:  Command Handler loads all events AFTER the snapshot
         (e.g., events #41 through #47) from the Event Store

Step 4:  Command Handler REPLAYS these remaining events on top of
         the snapshot, arriving at the CURRENT actual state of order "123"

Step 5:  Command Handler validates the AddLineItem command against
         this current state (e.g., "is the order still in DRAFT status?")

Step 6:  If valid, a NEW event is created: LineItemAdded { orderId,
         productId, qty, sequenceNumber: 48 }

Step 7:  This new event is APPENDED to order "123"'s stream in the
         Event Store - event #47 and everything before it remain
         UNTOUCHED, permanently

Step 8:  The new event is ALSO published to Kafka (Module 9) as an
         Integration Event, for any interested CQRS Projections
         (Module 16) or other services to consume

Step 9:  (Periodically, e.g., every 50 events) a NEW SNAPSHOT is
         saved, so future replays for this order start from THIS
         more recent checkpoint instead of the older one
```

---

## 11. Architecture Overview

```
                     Command: AddLineItem
                              │
                              ▼
                     Order Command Handler
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        Load Snapshot    Load Events      Replay to get
        (if any)         SINCE snapshot    CURRENT state
              │               │               │
              └───────────────┴───────────────┘
                              │
                    Validate command against
                       current state
                              │
                              ▼
                    Generate NEW Event
                    (LineItemAdded)
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
        APPEND to        Publish to        (Periodically)
        Event Store       Kafka             save new Snapshot
        (permanent,      (for CQRS
         append-only)     Projections,
                           Module 9/16)
```

---

## 12. ASCII Diagrams

### 12.1 State-Based vs Event-Sourced Persistence

```
STATE-BASED (traditional):

  orders table:
  id  | status  | total
  123 | SHIPPED | $49.99

  History LOST: we don't know it was ever PLACED, or PAID,
  or exactly WHEN each transition happened, unless a
  SEPARATE audit log was manually maintained


EVENT-SOURCED:

  order_events stream for order 123:
  seq | event_type      | payload                    | timestamp
  1   | OrderCreated     | { customerId: 42 }          | 10:00:00
  2   | LineItemAdded    | { productId: abc, qty: 2 }   | 10:00:05
  3   | OrderPlaced      | { total: 49.99 }             | 10:01:00
  4   | PaymentProcessed | { transactionId: xyz }        | 10:01:30
  5   | OrderShipped     | { carrier: "UPS" }             | 14:22:00

  CURRENT STATE = replay ALL 5 events = { status: SHIPPED, total: 49.99, ... }
  FULL HISTORY = permanently preserved, queryable at any point in time
```

### 12.2 Snapshot Optimization

```
WITHOUT snapshots (replay from the very beginning, every time):

  Event 1 → Event 2 → ... → Event 5000 → CURRENT STATE
  (must replay ALL 5000 events every single time state is needed
   - SLOW for long-lived, frequently-modified entities)


WITH snapshots (replay only since the last checkpoint):

  Snapshot @ Event 4900 (pre-computed state as of this point)
       │
  Event 4901 → Event 4902 → ... → Event 5000 → CURRENT STATE
  (only replay the last 100 events on top of the snapshot -
   MUCH faster)
```

### 12.3 Event Sourcing + CQRS Together

```
  Command ──▶ Event-Sourced Command Model (Event Store)
                              │
                    Events are ALREADY the natural
                    output of this model
                              │
                              ▼
                    Kafka (Integration Events)
                              │
                    ┌─────────┴─────────┐
                    ▼                   ▼
            CQRS Projection A    CQRS Projection B
            (Order Summary       (Admin Analytics
             Dashboard)           Dashboard)

  Event Sourcing and CQRS fit together almost perfectly:
  the Command side's NATURAL output (events) is EXACTLY
  what CQRS Projections need as input
```

---

## 13. Mermaid Flowcharts

### 13.1 Command Processing With Event Sourcing

```mermaid
flowchart TD
    A[Command received] --> B[Load latest Snapshot for this entity]
    B --> C[Load events since the Snapshot]
    C --> D[Replay events on top of Snapshot to get CURRENT state]
    D --> E[Validate command against current state]
    E --> F{Valid?}
    F -- No --> G[Reject command]
    F -- Yes --> H[Generate new Event]
    H --> I[Append event to Event Store]
    I --> J[Publish event to Kafka for CQRS/other consumers]
    J --> K{Snapshot threshold reached?}
    K -- Yes --> L[Save new Snapshot]
    K -- No --> M[Done]
    L --> M
```

### 13.2 Should You Use Event Sourcing?

```mermaid
flowchart TD
    A[Considering Event Sourcing] --> B{Does the domain genuinely need full audit history?}
    B -- No --> C[Do NOT use Event Sourcing - traditional state-based persistence is simpler]
    B -- Yes --> D{Do you need temporal queries - state as of a past point in time?}
    D -- No --> E{Is the audit need satisfiable with a simpler audit log table?}
    E -- Yes --> C
    E -- No --> F[Event Sourcing may be justified]
    D -- Yes --> F
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Full Event-Sourced Command Flow With Snapshot

```mermaid
sequenceDiagram
    participant C as Client
    participant H as Command Handler
    participant ES as Event Store
    participant K as Kafka

    C->>H: AddLineItem command
    H->>ES: Load latest Snapshot (seq #40)
    ES-->>H: Snapshot state
    H->>ES: Load events since seq #40
    ES-->>H: Events #41-47
    H->>H: Replay events on Snapshot -> current state
    H->>H: Validate command against current state
    H->>ES: Append new event (LineItemAdded, seq #48)
    ES-->>H: Appended successfully
    H->>K: Publish LineItemAdded event
    H-->>C: Acknowledgment
    Note over H,ES: Every 50 events, a new Snapshot is saved<br/>to keep future replays fast
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                  Order Command Service                       │
│  ┌───────────────────┐                                      │
│  │  Command Handlers      │  <- receive commands, load state,  │
│  │                          │     validate, generate events      │
│  └─────────┬───────────┘                                    │
│            ▼                                                 │
│  ┌───────────────────┐   ┌───────────────────┐              │
│  │  Reducer / Apply       │   │  Snapshot Manager      │              │
│  │  Function                │   │  (periodic saves,        │              │
│  │  (event -> state          │   │   loads for replay)       │              │
│  │   folding logic)           │   └───────────────────┘              │
│  └─────────┬───────────┘                                    │
│            ▼                                                 │
│  ┌───────────────────────────────────────────────┐          │
│  │              Event Store (append-only)              │          │
│  │  (EventStoreDB, or Kafka with compaction, or a         │          │
│  │   dedicated events table in PostgreSQL)                │          │
│  └───────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                               │
│  order-command-service pods ──▶ Event Store                    │
│                                  (EventStoreDB StatefulSet,      │
│                                   OR a PostgreSQL append-only     │
│                                   events table, OR Kafka with     │
│                                   log compaction for snapshots)    │
│         │                                                     │
│  Publishes events ──▶ Kafka (for CQRS Projections, Module 16)   │
│         │                                                     │
│  order-query-service pods ──▶ Query Model DB (denormalized,      │
│                                 built via Projections)             │
└───────────────────────────────────────────────────────────┘
```

---

## 17. Database Interaction

The Event Store's schema is fundamentally different from a traditional state-based table:

```
TRADITIONAL orders table (state-based):
  id | customer_id | status  | total  | updated_at
  1  | 42          | SHIPPED | 49.99  | 2026-01-05 14:22:00
  (ONE row per entity, OVERWRITTEN on every change)


EVENT-SOURCED order_events table:
  stream_id | sequence_number | event_type       | payload (JSON)      | occurred_at
  1         | 1               | OrderCreated      | {...}                | 10:00:00
  1         | 2               | LineItemAdded     | {...}                | 10:00:05
  1         | 3               | OrderPlaced       | {...}                | 10:01:00
  1         | 4               | PaymentProcessed  | {...}                | 10:01:30
  1         | 5               | OrderShipped      | {...}                | 14:22:00
  (MANY rows per entity, NEVER overwritten, only APPENDED)

  order_snapshots table:
  stream_id | sequence_number | state (JSON)         | created_at
  1         | 4               | { status: PAID,... }   | 10:01:35
  (periodic checkpoints to avoid full replay from seq #1)
```

---

## 18. Failure Scenarios

| Scenario | Event Sourcing Handling |
|---|---|
| A bug in the reducer/apply function produces incorrect current state | Since the EVENTS themselves are unaffected (only the reducer logic was buggy), you can FIX the reducer and REPLAY all events to get corrected state — a significant advantage over state-based systems where the original inputs that produced a bad state may be lost |
| Two commands for the same entity arrive concurrently | Requires optimistic concurrency control: each append checks the expected sequence number, rejecting a conflicting append if another event was appended first (similar in spirit to optimistic locking) |
| The Event Store becomes very large for a long-lived, frequently-modified entity | Snapshots (Section 9) mitigate replay performance, but very long streams may still need periodic archival/compaction strategies for extremely high-volume entities |
| An old event's schema no longer matches the current reducer's expectations | Requires "upcasting" (Section 8) — transforming old-shaped events into the new shape at replay time, since old events themselves can NEVER be rewritten or discarded |

```
Optimistic concurrency control (a REQUIRED safeguard):

  Command Handler A loads order "123" at sequence #47
  Command Handler B ALSO loads order "123" at sequence #47
  (both running concurrently, unaware of each other)

  Handler A appends event #48 SUCCESSFULLY
  Handler B attempts to append its OWN event, ALSO claiming
  to be #48 -> REJECTED (expected sequence mismatch)
  Handler B must RELOAD the current state (now including A's
  event #48) and retry its operation against the CORRECT
  current state
```

---

## 19. Scalability Discussion

Event Sourcing's append-only write pattern is generally very scalable for writes (appending is a simple, fast operation with no complex locking of existing data), and Snapshots keep read/replay performance manageable even for long-lived entities. However, Event Sourcing introduces genuine complexity costs that must be weighed against these benefits: every command requires a (snapshot + recent events) load-and-replay cycle rather than a simple, single-row read, meaning naive implementations without proper snapshotting can become a real performance liability for high-frequency, long-lived entities.

---

## 20. High Availability Considerations

- The Event Store itself is now a highly critical piece of infrastructure — since it IS the source of truth (not just a cache or derived copy), it must be deployed with strong durability and replication guarantees (whatever underlying technology you choose: EventStoreDB clustering, Kafka's own replication, or a replicated PostgreSQL instance).
- Because CQRS Query Models (Module 16) can be entirely rebuilt from the event stream, the **Query side's** availability requirements can be somewhat relaxed relative to the Event Store's — a Query Model outage is recoverable without data loss, precisely because the events themselves remain safe.

---

## 21. CAP Theorem Implications

The Event Store, as the system's actual source of truth, generally must favor **Consistency** for its own internal append operations (you cannot tolerate two conflicting, un-reconciled versions of "what happened to this order" — Section 18's optimistic concurrency control exists specifically to enforce this). Meanwhile, any CQRS Query Models built from this event stream continue to favor **Availability** with eventual consistency, exactly as discussed in Module 16 — Event Sourcing doesn't change CQRS's Query-side trade-offs, it simply makes the Command-side event stream a cleaner, more natural source for those Query-side projections to consume.

---

## 22. Node.js Implementation

Let's implement a simplified event-sourced Order Aggregate with snapshotting, using PostgreSQL as a straightforward (if not purpose-built) Event Store.

**Folder structure:**
```
order-command-service/
├── src/
│   ├── eventstore/
│   │   ├── eventStore.js
│   │   └── snapshotStore.js
│   ├── domain/
│   │   └── orderReducer.js
│   └── commands/
│       └── addLineItemHandler.js
```

**`src/eventstore/eventStore.js`**
```javascript
import { db } from "../db/connection.js";

// Append a new event - with OPTIMISTIC CONCURRENCY CONTROL to
// prevent two concurrent commands from silently overwriting each other
export async function appendEvent(streamId, expectedSequence, eventType, payload) {
  const result = await db.query(
    `INSERT INTO order_events (stream_id, sequence_number, event_type, payload, occurred_at)
     SELECT $1, $2, $3, $4, NOW()
     WHERE NOT EXISTS (
       SELECT 1 FROM order_events WHERE stream_id = $1 AND sequence_number = $2
     )
     RETURNING sequence_number`,
    [streamId, expectedSequence, eventType, JSON.stringify(payload)]
  );

  if (result.rowCount === 0) {
    throw new Error("CONCURRENCY_CONFLICT: another command modified this order first");
  }
  return result.rows[0].sequence_number;
}

export async function loadEventsSince(streamId, sinceSequence) {
  const result = await db.query(
    `SELECT sequence_number, event_type, payload FROM order_events
     WHERE stream_id = $1 AND sequence_number > $2
     ORDER BY sequence_number ASC`,
    [streamId, sinceSequence]
  );
  return result.rows;
}
```

**`src/eventstore/snapshotStore.js`**
```javascript
import { db } from "../db/connection.js";

const SNAPSHOT_INTERVAL = 20; // save a new snapshot every 20 events

export async function loadLatestSnapshot(streamId) {
  const result = await db.query(
    `SELECT sequence_number, state FROM order_snapshots
     WHERE stream_id = $1 ORDER BY sequence_number DESC LIMIT 1`,
    [streamId]
  );
  return result.rows[0] ?? { sequence_number: 0, state: null };
}

export async function maybeSaveSnapshot(streamId, sequenceNumber, state) {
  if (sequenceNumber % SNAPSHOT_INTERVAL === 0) {
    await db.query(
      `INSERT INTO order_snapshots (stream_id, sequence_number, state) VALUES ($1, $2, $3)`,
      [streamId, sequenceNumber, JSON.stringify(state)]
    );
  }
}
```

**`src/domain/orderReducer.js`** — the pure function that folds events into state
```javascript
// This REDUCER is the heart of Event Sourcing: given the current state
// and the NEXT event, compute the new state. Applying this function
// repeatedly over a full event history reconstructs current state.
export function applyEvent(state, event) {
  switch (event.event_type) {
    case "OrderCreated":
      return { status: "DRAFT", customerId: event.payload.customerId, lineItems: [] };
    case "LineItemAdded":
      return { ...state, lineItems: [...state.lineItems, event.payload] };
    case "OrderPlaced":
      return { ...state, status: "PLACED", total: event.payload.total };
    case "OrderShipped":
      return { ...state, status: "SHIPPED" };
    default:
      return state; // unknown event types are ignored, not fatal (forward compatibility)
  }
}

export function replayEvents(initialState, events) {
  return events.reduce(applyEvent, initialState);
}
```

---

## 23. Express.js Examples

**`src/commands/addLineItemHandler.js`**
```javascript
import { appendEvent, loadEventsSince } from "../eventstore/eventStore.js";
import { loadLatestSnapshot, maybeSaveSnapshot } from "../eventstore/snapshotStore.js";
import { replayEvents } from "../domain/orderReducer.js";
import { publishEvent } from "../events/eventPublisher.js";

export async function handleAddLineItem(orderId, { productId, quantity, unitPrice }) {
  // Load snapshot + events since -> reconstruct CURRENT state
  const snapshot = await loadLatestSnapshot(orderId);
  const recentEvents = await loadEventsSince(orderId, snapshot.sequence_number);
  const currentState = replayEvents(snapshot.state ?? {}, recentEvents);

  // Business rule validation against CURRENT (replayed) state
  if (currentState.status !== "DRAFT") {
    throw new Error("Cannot add items to an order that is already placed");
  }

  const nextSequence = snapshot.sequence_number + recentEvents.length + 1;
  const eventPayload = { productId, quantity, unitPrice };

  await appendEvent(orderId, nextSequence, "LineItemAdded", eventPayload);

  const newState = replayEvents(currentState, [{ event_type: "LineItemAdded", payload: eventPayload }]);
  await maybeSaveSnapshot(orderId, nextSequence, newState);

  await publishEvent("order-events", {
    eventType: "LineItemAdded",
    payload: { orderId, ...eventPayload },
  });

  return { orderId, sequence: nextSequence };
}
```

```javascript
// order-command-service/src/app.js
import express from "express";
import { handleAddLineItem } from "./commands/addLineItemHandler.js";

const app = express();
app.use(express.json());

app.post("/orders/:orderId/line-items", async (req, res) => {
  try {
    const result = await handleAddLineItem(req.params.orderId, req.body);
    res.status(201).json(result);
  } catch (err) {
    if (err.message.startsWith("CONCURRENCY_CONFLICT")) {
      return res.status(409).json({ error: err.message }); // 409 Conflict is the correct HTTP status
    }
    res.status(400).json({ error: err.message });
  }
});

app.listen(4020, () => console.log("Order Command Service running on port 4020"));
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
      - DATABASE_URL=postgresql://user:pass@eventstore-db:5432/orders_events
      - KAFKA_BROKER=kafka:9092
    depends_on: [eventstore-db, kafka]

  eventstore-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=orders_events]
    # In production, consider a purpose-built option like EventStoreDB
    # for stronger built-in append-only/optimistic-concurrency semantics
```

---

## 25. Kafka/Redis Integration

Every event appended to the Event Store (Section 22-23) is also published to Kafka — this is the natural, near-automatic bridge to CQRS's Projections (Module 16), since an event-sourced Command Model's output **is already** exactly the event stream a Projection needs:

```javascript
// The SAME LineItemAdded event that was appended to the Event Store
// is ALSO published to Kafka - feeding CQRS Query Model Projections
// with ZERO additional translation work required
await publishEvent("order-events", {
  eventType: "LineItemAdded",
  payload: { orderId, productId, quantity, unitPrice },
});
```

Redis can be used to cache **frequently-accessed current state** (the replayed result) for very hot entities, avoiding repeated replay for read-only checks, while the Event Store remains the authoritative source:

```javascript
export async function getCachedCurrentState(orderId) {
  const cached = await redis.get(`order-state:${orderId}`);
  if (cached) return JSON.parse(cached);
  // fall back to snapshot + replay (Section 22), then cache briefly
}
```

---

## 26. Error Handling

Concurrency conflicts (Section 18) must be handled with a **retry-with-reload** strategy, since the command's validation was based on now-outdated state:

```javascript
export async function handleAddLineItemWithRetry(orderId, input, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await handleAddLineItem(orderId, input);
    } catch (err) {
      if (err.message.startsWith("CONCURRENCY_CONFLICT") && attempt < maxRetries) {
        continue; // retry - will reload the (now more current) state and re-validate
      }
      throw err;
    }
  }
}
```

---

## 27. Logging & Monitoring

- Log every **appended event** (stream ID, sequence number, event type) for full auditability — this log, combined with the Event Store itself, provides comprehensive traceability.
- Monitor **replay performance** (time taken to load snapshot + replay recent events) per command — a growing replay time for a specific entity signals it may need more frequent snapshotting.
- Track **concurrency conflict rate** — a high rate for a specific entity might indicate genuine contention (multiple users/processes frequently modifying the same entity) requiring a different concurrency strategy or business process redesign.

---

## 28. Security Considerations

- Since the Event Store retains a **permanent, complete history**, be deliberate about what sensitive data enters event payloads — unlike a state-based table where old values are naturally overwritten and gone, an event-sourced system's history is, by design, forever (raising data retention/privacy considerations, e.g., under GDPR's "right to be forgotten," which can be genuinely challenging to reconcile with a truly immutable event log, and often requires deliberate technical strategies like encrypting personally-identifiable payloads with per-user keys that can later be destroyed).
- Access to the Event Store itself should be tightly controlled, since it contains the complete, authoritative history of every sensitive business operation.

---

## 29. Performance Optimization

- **Always implement snapshotting** for any entity expected to accumulate more than a small number of events over its lifetime — replaying an unbounded, ever-growing event list on every command is a significant, avoidable performance risk.
- Tune your **snapshot interval** (Section 22's `SNAPSHOT_INTERVAL`) based on actual measured replay performance versus snapshot storage overhead — too frequent wastes storage; too infrequent risks slow replays.
- Consider a purpose-built Event Store technology (e.g., EventStoreDB) for very high-throughput event-sourced systems, since general-purpose relational databases may not be optimally structured for extremely high-volume append/read-stream patterns at scale.

---

## 30. Production Best Practices

- Apply Event Sourcing **deliberately and selectively** — to specific entities/services with genuine audit, temporal-query, or event-integration needs — not as a default persistence strategy across your entire system.
- Design your reducer functions to **gracefully ignore unknown event types** (Section 22's `default` case) — this provides essential forward compatibility as your event schema evolves over time.
- Plan for **event schema evolution** deliberately from day one: since old events can never be rewritten, your reducer logic must be able to handle multiple historical event shapes indefinitely (via upcasting, Section 8, when needed).
- Combine Event Sourcing with CQRS (Module 16) as a natural pairing — the natural event output of an event-sourced Command Model is precisely CQRS's Projection input.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Applying Event Sourcing to every entity by default** | Adds significant complexity (event stores, snapshotting, reducers, schema evolution) with no benefit for entities that don't genuinely need audit history or temporal queries |
| **No snapshotting for long-lived, frequently-modified entities** | Causes replay performance to degrade unboundedly as event history grows |
| **Mutating or deleting past events** | Violates the fundamental append-only guarantee that makes Event Sourcing's audit and replay guarantees trustworthy in the first place |
| **No optimistic concurrency control on event appends** | Risks silently losing one of two concurrent commands' effects, corrupting the entity's history |
| **Ignoring event schema evolution planning** | Leads to reducer logic that breaks or behaves incorrectly when encountering old-shaped events from before a schema change |

```
Mutating past events (a fundamental violation):

  "Let's just fix that OrderPlaced event from last month - the
  total was calculated with a bug, let's UPDATE it to the correct value"

  Problem: this DESTROYS the very audit trail and replay
  guarantee that Event Sourcing exists to provide. If the past
  calculation was WRONG, the fix belongs in a NEW event (e.g.,
  "OrderTotalCorrected") or in updated REDUCER logic replayed
  going forward - never in silently altering historical events
```

---

## 32. Debugging Tips

- To debug an entity's current state, **replay its full event history manually** (or via a debugging tool) and inspect the state after each individual event — this gives you a step-by-step view of exactly how the current state was derived, a debugging capability state-based systems simply don't offer.
- If replay performance is slow, check snapshot frequency and the number of events being replayed since the last snapshot — this is almost always the root cause of Event Sourcing performance issues.
- If a concurrency conflict occurs unexpectedly often, investigate whether the business process itself has genuine, frequent contention on the same entity, which may warrant a business-level redesign rather than just a technical retry fix.

---

## 33. Interview Questions

### Easy
1. What is Event Sourcing, and how does it differ from traditional state-based persistence?
2. What is a Snapshot, and why is it needed?
3. What is Event Replay?
4. Why can events in an Event Store never be modified or deleted once written?
5. How does Event Sourcing relate to CQRS?

### Medium
6. How would you reconstruct an entity's state as of a specific point in time using Event Sourcing?
7. Explain optimistic concurrency control in the context of appending events, and why it's necessary.
8. Why must reducer functions be designed to gracefully handle unknown event types?
9. What are the performance implications of NOT implementing snapshotting for a long-lived entity?
10. What genuine business/domain characteristics suggest Event Sourcing is a good fit?

### Hard
11. Design an event-sourced Order Aggregate, including its reducer function and a snapshotting strategy, for a system expecting up to 10,000 events per entity over its lifetime.
12. How would you handle event schema evolution when an old event's shape no longer matches your current reducer's expectations (upcasting)?
13. Discuss the tension between Event Sourcing's "permanent, complete history" principle and data privacy regulations requiring the ability to delete specific user data (e.g., GDPR's "right to be forgotten").
14. Explain how you would debug a production incident where an entity's computed current state appears incorrect, using Event Sourcing's replay capability.
15. Compare the operational complexity of Event Sourcing combined with CQRS versus a simpler CQRS implementation using traditional state-based Command Model storage.

---

## 34. Scenario-Based Questions

1. Your finance team needs to answer "what was this account's balance at any point in the last 2 years, and exactly why" for a regulatory audit. Your current state-based system can't answer this. How would you evaluate whether Event Sourcing is the right fix?
2. A bug in your order total calculation logic is discovered, affecting orders processed over the last month. Using Event Sourcing, how would you determine the CORRECT historical totals?
3. Your event-sourced Order entity has accumulated 50,000 events over 3 years (a long-running corporate account), and command processing has become noticeably slow. Diagnose and propose a fix.
4. A regulatory requirement now demands you be able to permanently delete a specific customer's data upon request, but your Event Store's events reference that customer throughout their permanent history. How would you reconcile this with Event Sourcing's append-only principle?
5. Two customer service representatives simultaneously modify the same customer's order, and one của their changes appears to have been silently lost. Using this module's concepts, diagnose the likely cause.

---

## 35. Hands-on Exercises

1. Implement the `orderReducer.js` from Section 22 with at least 4 event types, and write a test replaying a sequence of events to verify the correct final state.
2. Implement the snapshot mechanism (Section 22), and verify that loading state via (snapshot + recent events) produces the SAME result as replaying the entire event history from the beginning.
3. Implement optimistic concurrency control (Section 22-23) and write a test simulating two concurrent commands, verifying one succeeds and one is correctly rejected.
4. Design (on paper) an upcasting strategy for a hypothetical event schema change (e.g., `OrderPlaced`'s `total` field changing from a plain number to a `{ amount, currency }` object).
5. Write a short comparison document: for a "Hotel Booking" system, would Event Sourcing genuinely help, or would a simpler audit-log-augmented state-based system suffice? Justify your answer.

---

## 36. Mini Project

**Build: An Event-Sourced Order Aggregate With Snapshotting**

1. Implement the Event Store (Section 22) using PostgreSQL, with `order_events` and `order_snapshots` tables.
2. Implement the reducer function (Section 22) supporting `OrderCreated`, `LineItemAdded`, `OrderPlaced`, and `OrderShipped` events.
3. Implement the `AddLineItem` command handler (Section 23) with snapshot-aware replay and optimistic concurrency control.
4. Demonstrate: create an order, add several line items, place it, and verify you can reconstruct its exact current state via replay, matching what you'd expect from the sequence of commands issued.

---

## 37. Advanced Project

**Build: Full Event Sourcing + CQRS Integration With Temporal Query Support**

1. Extend the Mini Project's event-sourced Order Aggregate with a `CancelOrder` command/event and appropriate business rule enforcement (e.g., cannot cancel an already-shipped order).
2. Publish every appended event to Kafka, and build a CQRS Query Model Projection (Module 16) consuming these events to maintain a denormalized `order_summary_view`.
3. Implement a **temporal query** endpoint: `GET /orders/:orderId/state-as-of/:timestamp`, which replays only the events that occurred before the given timestamp, reconstructing the order's state exactly as it was at that historical moment.
4. Simulate a reducer bug fix: intentionally introduce a bug in a calculation within the reducer, generate some events with the buggy logic, then fix the reducer and replay the full event history to show the CORRECTED state, demonstrating this unique Event Sourcing capability.
5. Write a design document addressing the GDPR "right to be forgotten" tension (Section 28) for this system, proposing a concrete technical strategy (e.g., per-user payload encryption with destructible keys) that satisfies both Event Sourcing's append-only principle and the deletion requirement.

---

## 38. Summary

- Event Sourcing stores an entity's state as a complete, immutable sequence of events, rather than directly storing current state — current state is a computed, replayed result, not a stored fact.
- Snapshots are essential for performance, avoiding full event-history replay from the beginning on every command for long-lived entities.
- Event Sourcing pairs naturally with CQRS (Module 16), since an event-sourced Command Model's natural output is exactly the event stream CQRS Projections need.
- Optimistic concurrency control is required to safely handle concurrent commands modifying the same entity.
- Event Sourcing provides powerful capabilities (full audit history, temporal queries, replay-with-corrected-logic) but introduces real complexity (schema evolution, snapshotting, GDPR tension) that must be justified by genuine domain needs, not applied as a default.

---

## 39. Revision Notes

- Event Sourcing: state = replayed sequence of events, not directly stored.
- Snapshot: periodic checkpoint to avoid full replay from the beginning.
- Reducer/Apply function: (state, event) -> new state; the core replay logic.
- Optimistic concurrency control: required to safely handle concurrent appends to the same stream.
- Event Sourcing + CQRS: a natural, near-seamless pairing.
- Events are NEVER modified/deleted once appended — fixes go into new events or updated reducer logic, replayed forward.
- Real complexity costs: schema evolution (upcasting), snapshotting, and data privacy tension (GDPR) must be deliberately planned for.

---

## 40. One-Page Cheat Sheet

```
EVENT SOURCING:       state = replay of a permanent, append-only event sequence
EVENT STORE:          specialized DB for storing/retrieving event streams
SNAPSHOT:             periodic state checkpoint - avoids full replay from event #1
REDUCER:              (state, event) -> new state - the core replay logic
OPTIMISTIC CONCURRENCY: expected sequence number check on append - prevents lost updates
TEMPORAL QUERY:        replay events UP TO a past point in time - a unique ES capability

PAIRS NATURALLY WITH: CQRS (Module 16) - events ARE the Projection input

GOLDEN RULES:
  - NEVER modify or delete a past event once appended - fixes go FORWARD, not backward
  - ALWAYS implement snapshotting for long-lived, frequently-modified entities
  - ALWAYS use optimistic concurrency control on event appends
  - Design reducers to gracefully ignore unknown/future event types
  - Apply Event Sourcing SELECTIVELY - only where audit/temporal/event-integration needs justify its real complexity
```

---

**Suggested Next Module:** Module 18 — Resilience Patterns (Retry, Timeout, Bulkhead, Circuit Breaker, Rate Limiting, and Fallback strategies for building fault-tolerant microservices)
