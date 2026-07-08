# Module 4 — Domain-Driven Design (DDD) Basics

> **Microservices Masterclass** | Level: Beginner–Intermediate | Track: Node.js Backend Engineering
> Prerequisite: Module 1–3
> Next Module: Module 5 — Service Boundaries

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

Here's a question most tutorials skip: **how do you actually decide where one microservice ends and another begins?** Modules 1–3 assumed clean boundaries like "User Service," "Order Service," "Payment Service" — but in the real world, business logic doesn't come pre-divided into neat boxes. Deciding those boundaries badly is the single most common reason microservices migrations fail.

**Domain-Driven Design (DDD)** is the discipline that gives you a principled, repeatable way to find those boundaries — not by guessing, not by mirroring your database tables, but by deeply understanding the *business itself*. This module introduces the core DDD concepts you need before you can responsibly design service boundaries in Module 5.

DDD is a big topic with entire books written about it (Eric Evans's original 2003 book, and Vaughn Vernon's "Implementing Domain-Driven Design" are the classics). This module distills the 20% of DDD that gives you 80% of the value for microservices boundary design.

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Explain what a "domain" is and why it's the starting point for architecture decisions.
- Define and identify a **Bounded Context** in a real system.
- Distinguish between **Entities**, **Value Objects**, and **Aggregates**.
- Explain the role of a **Repository** and a **Domain Service**.
- Use **Ubiquitous Language** to align code with how the business actually talks.
- Apply these concepts to sketch bounded contexts for a real business domain.
- Avoid the common mistake of designing microservices around database tables instead of business capabilities.

---

## 3. Problem Statement

A team building an e-commerce platform starts designing microservices by looking at their database schema: `users`, `products`, `orders`, `payments`, `reviews`, `shipments` tables. They create one microservice per table. Six months later:

- The word "Product" means something different to the Catalog team (name, images, description) than to the Inventory team (SKU, stock count, warehouse location) than to the Pricing team (cost, margin, discounts) — but they're all crammed into one "Product Service" fighting over what a "Product" even *is*.
- A single "Order" object is edited by five different services for five different reasons (placing, shipping, refunding, reviewing, invoicing), leading to constant merge conflicts in that service's codebase and unclear ownership.
- Every schema change to the "Product" table requires cross-team sign-off from Catalog, Inventory, and Pricing teams simultaneously, recreating the exact deployment-coupling problem microservices were supposed to solve.

The problem: **splitting by database table (or by "noun") is not the same as splitting by business capability.** DDD's bounded contexts solve this by asking not "what tables exist?" but "where does one team's model of the world stop making sense and another's begin?"

---

## 4. Why This Concept Exists

DDD exists because **the hardest part of building good software isn't writing code — it's correctly understanding the business domain the code represents.** Eric Evans, who coined the term, observed that most software failures come not from bad code but from a **model mismatch**: the software's internal model of "Order" or "Customer" doesn't match how the business actually thinks about those concepts, causing constant translation errors, bugs, and miscommunication between business stakeholders and engineers.

For microservices specifically, DDD exists because it gives you an **objective methodology** for drawing service boundaries — instead of splitting by database table, org chart, or gut feeling, you split along **bounded contexts**: places where a specific model of a business concept is self-consistent and complete.

---

## 5. Historical Background

- **2003** — Eric Evans published *"Domain-Driven Design: Tackling Complexity in the Heart of Software,"* introducing the vocabulary (Entity, Value Object, Aggregate, Bounded Context, Ubiquitous Language) still used today. At the time, this was aimed at large, complex enterprise systems generally — not specifically microservices, which didn't yet exist as a named architectural style.
- **2011–2014** — As microservices emerged as an architectural style, practitioners recognized that DDD's "Bounded Context" concept was almost a perfect match for "where should one microservice's responsibility end and another's begin?" This connection is now considered one of the most important ideas in microservices design.
- **2013** — Vaughn Vernon published *"Implementing Domain-Driven Design,"* making DDD's ideas more approachable and directly connecting them to service-oriented and microservices architectures.
- **Present** — "Bounded Context" is now one of the most commonly referenced DDD terms in microservices job interviews and architecture discussions, often considered a prerequisite vocabulary for any serious system design conversation involving service boundaries.

> **Interview tip:** If asked "how do you decide microservice boundaries," a strong answer references Domain-Driven Design and Bounded Contexts specifically — this signals you understand *why* boundaries exist, not just that they exist.

---

## 6. Real-World Analogy

**Analogy: The Word "Book" Means Different Things to Different People**

Imagine a company that both **publishes** and **sells** books.

- To the **Publishing department**, a "Book" is a manuscript with an author, editing status, chapters, and a publication date. Its stock count or sale price is irrelevant to them.
- To the **Warehouse/Inventory department**, a "Book" is a physical SKU with a barcode, a shelf location, a quantity in stock, and a weight for shipping calculations. The manuscript content is irrelevant to them.
- To the **Sales/Storefront department**, a "Book" is a product listing with a price, a description, images, and customer reviews. The chapter content and warehouse shelf location are irrelevant to them.

All three departments say "Book" — but they mean **three different, self-consistent models**. Trying to build ONE "Book" class/table/service that satisfies all three departments perfectly leads to a bloated, contradictory mess where every team keeps adding fields the others don't need and breaking each other's assumptions.

**A Bounded Context says: it's okay — even correct — for "Book" to mean three different things in three different contexts**, as long as each context's model is internally consistent and there's a clear, explicit translation when contexts need to talk to each other (e.g., Sales asking Inventory "is this SKU in stock?").

---

## 7. Technical Definition

> A **Domain** is the subject area a piece of software is built for — the real-world business problem it exists to solve (e.g., "e-commerce," "ride-sharing," "banking").

> A **Bounded Context** is an explicit boundary (usually corresponding to a microservice) within which a particular **domain model** is defined and consistent — terms, rules, and objects inside this boundary have one unambiguous meaning, even if the same term means something different in another Bounded Context.

> An **Entity** is a domain object defined by its **identity**, not its attributes — two Orders with identical items are still different Orders if they have different IDs; identity persists even as attributes change over time.

> A **Value Object** is a domain object defined entirely by its **attributes**, with no distinct identity — two `Money` objects representing "$50 USD" are interchangeable; there's no meaningful sense in which they're "different $50s."

> An **Aggregate** is a cluster of Entities and Value Objects treated as a single unit for data changes, with one designated **Aggregate Root** acting as the only entry point for modifications, ensuring the whole cluster stays internally consistent.

> A **Repository** is an abstraction that provides the illusion of an in-memory collection of Aggregates, hiding the actual persistence/database details from the domain logic.

> A **Domain Service** holds domain logic that doesn't naturally belong to any single Entity or Value Object (e.g., "transfer funds between two accounts" doesn't belong to just one Account).

> **Ubiquitous Language** is a shared vocabulary, used consistently by both business stakeholders and engineers, in conversation, documentation, AND in the code itself (class names, function names) — eliminating the translation gap between "what the business says" and "what the code says."

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Domain** | The overall business problem space (e.g., "logistics," "payments") |
| **Subdomain** | A specific slice of the domain (e.g., within "e-commerce": Catalog, Ordering, Shipping) |
| **Core Subdomain** | The subdomain that provides your competitive advantage — deserves the most design investment |
| **Supporting Subdomain** | Necessary but not differentiating (e.g., internal notifications) |
| **Generic Subdomain** | Common, solved problems better bought than built (e.g., authentication, payments processing via Stripe) |
| **Bounded Context** | An explicit boundary where one domain model is consistent and unambiguous |
| **Context Map** | A diagram showing all Bounded Contexts and how they relate/communicate |
| **Entity** | Identity-based domain object (persists identity across attribute changes) |
| **Value Object** | Attribute-based domain object (no identity, interchangeable if attributes match) |
| **Aggregate / Aggregate Root** | A consistency boundary for a cluster of related objects, with one root controlling all changes |
| **Repository** | Abstraction for persisting/retrieving Aggregates without exposing storage details |
| **Domain Service** | Domain logic that spans multiple Entities/Value Objects |
| **Ubiquitous Language** | Shared, consistent vocabulary between business and code |

---

## 9. Internal Working

Here's how DDD concepts map onto actual code and system design, step by step:

1. Business stakeholders and engineers hold **domain modeling sessions** (often using techniques like Event Storming) to explore the real business processes and vocabulary.
2. From these sessions, distinct **subdomains** emerge (e.g., "Order Management," "Inventory," "Shipping," "Customer Identity").
3. Each subdomain is evaluated: is it **Core** (differentiating, build carefully), **Supporting** (necessary, build adequately), or **Generic** (buy/use existing solutions)?
4. Each subdomain is mapped to a **Bounded Context** — a boundary in which a specific model is authoritative. In a microservices system, a Bounded Context typically becomes one microservice (or a small, cohesive group of services).
5. Inside each Bounded Context, the team models the domain using **Entities**, **Value Objects**, and **Aggregates**, all named using the **Ubiquitous Language** the business actually uses.
6. **Repositories** are implemented to persist and retrieve Aggregates, keeping the domain model itself free of database-specific code (an important separation for testability and clarity).
7. A **Context Map** documents how Bounded Contexts relate to each other (e.g., "Order Context is a customer of Inventory Context, calling its API to check stock").

---

## 10. Step-by-Step Request Flow

**Scenario: Placing an order, viewed through a DDD lens.**

```
Step 1:  Client sends "Place Order" command to Order Bounded Context
Step 2:  Order Context's Application Service receives the command
Step 3:  Application Service loads the "Order" Aggregate Root via its Repository
         (or constructs a new one, since this is a new order)
Step 4:  Order Aggregate Root validates business invariants internally
         (e.g., "an order must have at least one line item")
Step 5:  Order Context needs to confirm stock — this concept ("stock")
         belongs to the Inventory Bounded Context, not Order's own model
Step 6:  Order Context calls Inventory Context's API (crossing the
         Bounded Context boundary explicitly, via a defined contract)
Step 7:  Inventory Context responds with stock confirmation, translated
         into terms Order Context understands (available: true/false)
Step 8:  Order Aggregate Root finalizes the order, ensuring all its
         internal Entities/Value Objects (LineItems, ShippingAddress) are consistent
Step 9:  Order Repository persists the entire Aggregate as one atomic unit
Step 10: A Domain Event ("OrderPlaced") is raised, to be published
         for other Bounded Contexts (Shipping, Notification) to react to
```

Notice Step 6: crossing a Bounded Context boundary is always **explicit and deliberate** — never an implicit shared-database read. This is the DDD principle that directly enables clean microservice boundaries.

---

## 11. Architecture Overview

```
                    E-Commerce Domain
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
  Order Management    Inventory Context    Shipping Context
  Bounded Context      Bounded Context      Bounded Context
  (Core Subdomain)    (Supporting Subdomain) (Supporting Subdomain)
        │                   │                   │
   "Order" model        "Product" model      "Shipment" model
   means: line items,   means: SKU, stock    means: carrier,
   totals, status        count, warehouse     tracking number
        │                   │                   │
        └─────────Context Map / API Contracts────┘
              (explicit translation at boundaries)
```

Each Bounded Context above typically maps to one (or a small cohesive set of) microservice(s) — this is the direct bridge from DDD theory to Module 3's architecture components.

---

## 12. ASCII Diagrams

### 12.1 Entity vs Value Object

```
ENTITY (identity matters):

  Order #1001 { items: [Book x2], total: $40 }
  Order #1002 { items: [Book x2], total: $40 }

  -> Even with IDENTICAL attributes, these are TWO DIFFERENT orders
     because they have different identities (#1001 vs #1002)


VALUE OBJECT (attributes matter, no identity):

  Money { amount: 40, currency: "USD" }
  Money { amount: 40, currency: "USD" }

  -> These are the SAME value — completely interchangeable,
     there's no meaningful "identity" distinguishing them
```

### 12.2 Aggregate Boundary

```
                     Order Aggregate
        ┌─────────────────────────────────────┐
        │        Order (Aggregate Root)         │
        │  ┌───────────┐   ┌───────────┐        │
        │  │ LineItem  │   │ LineItem  │  ...   │
        │  │ (Entity)  │   │ (Entity)  │        │
        │  └───────────┘   └───────────┘        │
        │  ┌────────────────────────┐            │
        │  │ ShippingAddress        │            │
        │  │ (Value Object)         │            │
        │  └────────────────────────┘            │
        └─────────────────────────────────────┘
                          │
        ALL changes to LineItems or ShippingAddress
        MUST go through the Order Aggregate Root —
        never modified directly from outside
```

### 12.3 Context Map (relationships between Bounded Contexts)

```
Order Context ──(Customer/Supplier)──▶ Inventory Context
   (Order depends on Inventory's stock API; Inventory
    team must consider Order's needs when changing their API)

Order Context ──(Publishes Events)──▶ Shipping Context
   (Order publishes "OrderPlaced"; Shipping independently
    reacts, no direct coupling to Order's internal model)

Order Context ──(Anti-Corruption Layer)──▶ Legacy Billing System
   (Order translates the legacy system's confusing model
    into its own clean model at the boundary, isolating
    legacy mess from the new codebase)
```

---

## 13. Mermaid Flowcharts

### 13.1 From Domain to Microservice

```mermaid
flowchart TD
    A[Business Domain: E-Commerce] --> B[Identify Subdomains via Event Storming]
    B --> C{Classify each Subdomain}
    C -->|Core| D[Order Management]
    C -->|Supporting| E[Inventory]
    C -->|Generic| F[Payments - use Stripe]
    D --> G[Define Bounded Context: Order]
    E --> H[Define Bounded Context: Inventory]
    G --> I[Model Aggregates: Order, LineItem]
    H --> J[Model Aggregates: Product, StockLevel]
    I --> K[Order Microservice]
    J --> L[Inventory Microservice]
```

### 13.2 Deciding Core vs Supporting vs Generic

```mermaid
flowchart TD
    Start[New Subdomain identified] --> Q1{Does it directly create competitive advantage?}
    Q1 -- Yes --> Core[Core Subdomain: invest heavily, build in-house]
    Q1 -- No --> Q2{Is it necessary but not differentiating?}
    Q2 -- Yes --> Support[Supporting Subdomain: build adequately]
    Q2 -- No --> Q3{Is it a common, well-solved problem?}
    Q3 -- Yes --> Generic[Generic Subdomain: buy/use existing solution]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Crossing a Bounded Context Boundary

```mermaid
sequenceDiagram
    participant C as Client
    participant OC as Order Context (App Service)
    participant OA as Order Aggregate
    participant IC as Inventory Context (API)
    participant OR as Order Repository

    C->>OC: Place Order command
    OC->>OA: Load/construct Order Aggregate
    OA->>OA: Validate invariants (line items, addresses)
    OC->>IC: Check stock (crossing Bounded Context boundary)
    IC-->>OC: Stock confirmed
    OC->>OA: Finalize order
    OC->>OR: Save Aggregate (atomic persistence)
    OR-->>OC: Saved
    OC-->>C: Order placed confirmation
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│              Order Bounded Context (Microservice)         │
│                                                             │
│  ┌───────────────────┐                                     │
│  │ Application Service│  <- orchestrates use cases          │
│  └─────────┬─────────┘                                     │
│            ▼                                               │
│  ┌───────────────────┐                                     │
│  │  Order Aggregate    │  <- Entities + Value Objects        │
│  │  (domain model)     │     + business invariants           │
│  └─────────┬─────────┘                                     │
│            ▼                                               │
│  ┌───────────────────┐                                     │
│  │  Order Repository   │  <- persistence abstraction          │
│  └─────────┬─────────┘                                     │
│            ▼                                               │
│  ┌───────────────────┐                                     │
│  │     Order DB         │                                    │
│  └───────────────────┘                                     │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
Each Bounded Context typically becomes its own deployable service:

┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐
│  order-context      │   │ inventory-context   │   │ shipping-context    │
│  (order-service pod) │   │ (inventory-service  │   │ (shipping-service   │
│                      │   │  pod)               │   │  pod)               │
└───────────────────┘   └───────────────────┘   └───────────────────┘
          │                       │                       │
   ┌──────▼──────┐        ┌──────▼──────┐        ┌──────▼──────┐
   │  Order DB    │        │ Inventory DB │        │ Shipping DB  │
   └─────────────┘        └─────────────┘        └─────────────┘
```

This is the direct payoff of DDD for microservices: a well-identified Bounded Context maps cleanly onto a deployable service boundary with its own database — exactly the pattern from Modules 1–3, now backed by a principled reason for *where* the lines are drawn.

---

## 17. Database Interaction

DDD strongly informs how a Bounded Context's database should be structured:

```
Order Bounded Context's Database:

  orders table          <- maps to Order Aggregate Root
  order_line_items table <- maps to LineItem Entities (child of Order)
  (shipping_address is often embedded as JSON, since it's a Value Object,
   not a separate identity-bearing row)

  RULE: The database schema should reflect Aggregate boundaries —
  an Aggregate is typically saved/loaded as ONE atomic transaction,
  never partially, to preserve its internal consistency invariants.
```

A common mistake: designing the database schema first (normalized, table-per-noun) and then trying to retrofit Aggregates onto it. DDD says the **domain model comes first**; the schema serves the Aggregate boundaries, not the other way around.

---

## 18. Failure Scenarios

| Scenario | DDD-Informed Handling |
|---|---|
| A change to an Order must update 3 line items and the total atomically | Because they're all inside the *same Aggregate*, this is one transaction against one database — consistency is guaranteed by design |
| Order Context needs Inventory data mid-transaction, and Inventory is down | Since Inventory is a *different* Bounded Context (different service, different DB), this is inherently an eventual-consistency / async boundary — the Order Context must have a defined fallback (reject the order, or proceed optimistically and reconcile later) |
| Two Bounded Contexts define the term "Customer" incompatibly | This is expected and fine — DDD explicitly allows this, as long as there's an explicit translation (Anti-Corruption Layer) whenever the two contexts must communicate |
| A misidentified Aggregate boundary causes changes to ripple across contexts constantly | This signals the Bounded Context boundary was drawn incorrectly and should be revisited — a well-drawn boundary should minimize cross-context chatter |

---

## 19. Scalability Discussion

DDD doesn't directly address technical scalability (that's Module 3's territory), but it directly enables it: because each Bounded Context is an independent model with its own Aggregates and its own persistence, each maps cleanly to an independently deployable, independently scalable microservice. Poorly-drawn boundaries (e.g., splitting an Aggregate across two services) force distributed transactions and tight coupling, which actively *hurts* scalability — so good DDD boundary decisions are a prerequisite for good microservices scalability, not just a nice-to-have.

---

## 20. High Availability Considerations

Because each Bounded Context owns its Aggregates and its data exclusively, one context can go down without corrupting another context's data — the Aggregate's consistency boundary is also, conveniently, a *failure* boundary. This is one of DDD's under-appreciated benefits: correctly-drawn Bounded Contexts naturally produce good fault-isolation properties in the resulting microservices architecture.

---

## 21. CAP Theorem Implications

Within a single Bounded Context (a single Aggregate, a single database), you can typically enforce strong consistency easily — it's one transaction against one store. **Across** Bounded Contexts, you are inherently in eventually-consistent, CAP-theorem territory, since crossing a context boundary means crossing a network boundary to a separately-deployed service and database. This is why DDD explicitly recommends using **Domain Events** for most cross-context communication: it embraces eventual consistency as the natural, correct way to synchronize independent Bounded Contexts, rather than fighting it with fragile distributed transactions.

---

## 22. Node.js Implementation

Let's implement a small but real **Order Aggregate** following DDD principles in Node.js.

**Folder structure:**
```
order-service/
├── src/
│   ├── domain/
│   │   ├── Order.js           <- Aggregate Root
│   │   ├── LineItem.js        <- Entity (child of Order)
│   │   └── Money.js           <- Value Object
│   ├── repositories/
│   │   └── OrderRepository.js
│   ├── application/
│   │   └── PlaceOrderService.js  <- Application Service (use case)
│   └── app.js
```

**`src/domain/Money.js`** (Value Object)
```javascript
// A Value Object: defined ENTIRELY by its attributes, no identity.
// Two Money instances with the same amount/currency are interchangeable.
export class Money {
  constructor(amount, currency = "USD") {
    if (amount < 0) throw new Error("Money amount cannot be negative");
    this.amount = amount;
    this.currency = currency;
    Object.freeze(this); // Value Objects are immutable by convention
  }

  add(other) {
    if (other.currency !== this.currency) {
      throw new Error("Cannot add Money of different currencies");
    }
    return new Money(this.amount + other.amount, this.currency);
  }

  equals(other) {
    return this.amount === other.amount && this.currency === other.currency;
  }
}
```

**`src/domain/LineItem.js`** (Entity, child of the Order Aggregate)
```javascript
// An Entity: has an identity (id), even though its attributes (quantity)
// may change over its lifetime while remaining "the same" LineItem.
export class LineItem {
  constructor(id, productId, quantity, unitPrice) {
    this.id = id;
    this.productId = productId;
    this.quantity = quantity;
    this.unitPrice = unitPrice; // a Money Value Object
  }

  get subtotal() {
    return new Money(this.unitPrice.amount * this.quantity, this.unitPrice.currency);
  }
}
```

**`src/domain/Order.js`** (Aggregate Root — the ONLY entry point for changes)
```javascript
import { Money } from "./Money.js";
import { LineItem } from "./LineItem.js";

// The Aggregate Root enforces ALL business invariants for the cluster.
// External code NEVER modifies LineItems directly — only through Order's methods.
export class Order {
  constructor(id, customerId) {
    this.id = id;
    this.customerId = customerId;
    this.lineItems = [];
    this.status = "DRAFT";
  }

  addLineItem(productId, quantity, unitPrice) {
    if (this.status !== "DRAFT") {
      throw new Error("Cannot modify an order that is already placed");
    }
    const lineItemId = `${this.id}-item-${this.lineItems.length + 1}`;
    this.lineItems.push(new LineItem(lineItemId, productId, quantity, unitPrice));
  }

  get total() {
    return this.lineItems.reduce(
      (sum, item) => sum.add(item.subtotal),
      new Money(0)
    );
  }

  place() {
    // Business invariant: an order must have at least one line item
    if (this.lineItems.length === 0) {
      throw new Error("Cannot place an order with no line items");
    }
    this.status = "PLACED";
    this.placedAt = new Date();
  }
}
```

---

## 23. Express.js Examples

**`src/application/PlaceOrderService.js`** (Application Service — orchestrates the use case)
```javascript
import { Order } from "../domain/Order.js";
import { Money } from "../domain/Money.js";
import { orderRepository } from "../repositories/OrderRepository.js";
import { checkStock } from "../clients/inventoryClient.js"; // crosses Bounded Context

export async function placeOrder({ customerId, items }) {
  const order = new Order(crypto.randomUUID(), customerId);

  for (const item of items) {
    // Crossing into the Inventory Bounded Context explicitly, via its API —
    // never by querying Inventory's database directly.
    const inStock = await checkStock(item.productId, item.quantity);
    if (!inStock) {
      throw new Error(`Product ${item.productId} is out of stock`);
    }
    order.addLineItem(item.productId, item.quantity, new Money(item.unitPrice));
  }

  order.place(); // enforces the Aggregate's own invariants
  await orderRepository.save(order); // atomic persistence of the whole Aggregate

  return order;
}
```

**`src/app.js`**
```javascript
import express from "express";
import { placeOrder } from "./application/PlaceOrderService.js";

const app = express();
app.use(express.json());

app.post("/orders", async (req, res) => {
  try {
    const order = await placeOrder(req.body);
    res.status(201).json(order);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.listen(4002, () => console.log("Order Service running on port 4002"));
```

Notice the **Ubiquitous Language** in action: the code says `placeOrder`, `addLineItem`, `checkStock` — the exact verbs a business stakeholder would use describing this process, not generic CRUD terms like `create`, `update`, `insert`.

---

## 24. Docker Examples

```dockerfile
# Dockerfile — Order Service (a single Bounded Context's deployable)
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
EXPOSE 4002
USER node
CMD ["node", "src/app.js"]
```

```yaml
# docker-compose.yml — each Bounded Context is its own container + database
version: "3.9"
services:
  order-service:
    build: ./order-service
    ports: ["4002:4002"]
    environment:
      - DATABASE_URL=postgresql://user:pass@order-db:5432/orders
      - INVENTORY_SERVICE_URL=http://inventory-service:4004
    depends_on: [order-db, inventory-service]

  inventory-service:
    build: ./inventory-service
    ports: ["4004:4004"]
    environment:
      - DATABASE_URL=postgresql://user:pass@inventory-db:5432/inventory
    depends_on: [inventory-db]

  order-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=orders]

  inventory-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=inventory]
```

---

## 25. Kafka/Redis Integration

**Publishing a Domain Event when the Order Aggregate transitions state:**
```javascript
// After order.place() succeeds and is persisted, publish a Domain Event
// so OTHER Bounded Contexts (Shipping, Notification) can react independently.
import { producer } from "../kafka/producer.js";

export async function publishOrderPlacedEvent(order) {
  await producer.send({
    topic: "order-events",
    messages: [
      {
        key: order.id,
        value: JSON.stringify({
          type: "OrderPlaced",
          orderId: order.id,
          customerId: order.customerId,
          total: order.total, // serialized Value Object
          occurredAt: new Date().toISOString(),
        }),
      },
    ],
  });
}
```

This is DDD's **Domain Event** concept made concrete: the event name (`OrderPlaced`) uses the Ubiquitous Language, and it represents a meaningful business occurrence, not just a technical "row updated" notification.

**Redis as a read-model cache for a different Bounded Context** (e.g., Shipping Context caching a denormalized view of Order data it needs, translated into its own terms):
```javascript
// Inside Shipping Context — note it stores ONLY what shipping cares about,
// translated into ITS OWN model, not a raw copy of Order's internal structure.
await redis.set(
  `shippable-order:${event.orderId}`,
  JSON.stringify({ orderId: event.orderId, destinationAddress: event.shippingAddress }),
  { EX: 3600 }
);
```

---

## 26. Error Handling

Domain invariant violations should raise clear, domain-specific errors — not generic database errors:

```javascript
// Good: a domain-specific error that speaks the Ubiquitous Language
export class OrderAlreadyPlacedError extends Error {
  constructor(orderId) {
    super(`Order ${orderId} has already been placed and cannot be modified`);
    this.name = "OrderAlreadyPlacedError";
  }
}

// Inside Order.addLineItem():
if (this.status !== "DRAFT") {
  throw new OrderAlreadyPlacedError(this.id);
}
```

Handling errors crossing a Bounded Context boundary (e.g., Inventory Context unreachable):
```javascript
export async function checkStock(productId, quantity) {
  try {
    const res = await axios.get(`${process.env.INVENTORY_SERVICE_URL}/stock/${productId}`, {
      timeout: 2000,
    });
    return res.data.available >= quantity;
  } catch (err) {
    // Business decision, not just a technical one: do we fail the order,
    // or proceed optimistically? DDD encourages making this an EXPLICIT
    // domain policy decision, not an accidental side effect of a try/catch.
    throw new Error("Unable to verify stock at this time — please try again");
  }
}
```

---

## 27. Logging & Monitoring

Log using the **Ubiquitous Language**, not generic technical terms — this makes logs far more useful to both engineers and business stakeholders debugging an issue together:

```javascript
// Good: speaks the business language
logger.info({ orderId: order.id, customerId: order.customerId }, "Order placed successfully");

// Less useful: purely technical, loses business meaning
logger.info({ table: "orders", operation: "INSERT" }, "Row inserted");
```

Monitor **domain-meaningful metrics** per Bounded Context (e.g., "orders placed per minute," "stock-check failures") rather than only generic technical metrics (CPU, memory) — this connects observability directly to business health.

---

## 28. Security Considerations

- Bounded Context boundaries are also good **authorization boundaries** — e.g., only the Order Context should be able to transition an Order's status; even an admin tool should go through Order Context's API, not bypass it by writing to the database directly.
- Keep **Aggregate invariants enforced in the domain layer**, not just at the API layer — this prevents invariant violations even from internal scripts, migrations, or other services that might call into the domain logic directly (e.g., via a shared library or a message handler).

---

## 29. Performance Optimization

- Load and save **whole Aggregates** as a unit — avoid designs that require loading a huge Aggregate just to read one small field; if you find yourself doing this often, it may signal the Aggregate boundary is too large and should be reconsidered.
- Use **read models / denormalized views** derived from Domain Events for high-read, cross-context data needs (e.g., a "Shipping Dashboard" that needs Order + Customer + Product data combined) instead of forcing synchronous joins across Bounded Context boundaries on every read.

---

## 30. Production Best Practices

- Involve actual **domain experts** (not just engineers) in defining Bounded Contexts — DDD is fundamentally a collaborative modeling discipline, not a purely technical exercise.
- Keep a living **Context Map** document, updated as the system evolves, so new engineers can quickly understand how Bounded Contexts relate.
- Revisit Bounded Context boundaries periodically — a boundary that made sense at 5 engineers may need to be reconsidered at 50 engineers; DDD boundaries are not meant to be permanent, unchangeable decisions.
- Use an **Anti-Corruption Layer** whenever integrating with a legacy system or third-party API whose model doesn't match your domain's Ubiquitous Language — translate at the boundary, don't let legacy concepts leak into your clean model.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **Anemic Domain Model** | Entities/Aggregates with only getters/setters and no real business logic — all logic lives elsewhere (services), defeating the purpose of DDD's rich domain modeling |
| **Designing services around database tables** | Produces boundaries that don't match real business capabilities, causing constant cross-service coupling |
| **One giant Aggregate for the whole domain** | Makes every change a full-system lock/transaction; kills scalability and concurrency |
| **Sharing one model across multiple Bounded Contexts** | Forces every context to agree on every field forever — recreates monolith-style coupling |
| **Skipping Ubiquitous Language** | Code uses generic CRUD terms while business uses domain terms, causing constant miscommunication and translation bugs |

```
Anemic Domain Model (anti-pattern):

  class Order {
    id; customerId; lineItems; status;  // just data, no behavior
  }

  // ALL logic lives in some external "OrderService" instead:
  function placeOrder(order) {
    if (order.lineItems.length === 0) throw new Error(...);
    order.status = "PLACED"; // invariant enforcement scattered everywhere,
                             // easy to forget to check in some OTHER function
  }
```

---

## 32. Debugging Tips

- If a bug involves an Aggregate ending up in an "impossible" state, check whether something modified its internals **directly** instead of going through the Aggregate Root's methods — this is the most common source of invariant violations.
- If two teams keep needing to coordinate changes to "the same" concept, it's a signal their Bounded Context boundary might be misdrawn, or that a proper Anti-Corruption Layer / API contract is missing between them.
- When cross-context data looks stale or inconsistent, check the **event consumption** path — since cross-context sync is asynchronous, verify the consumer is actually processing events (not stuck) before assuming a logic bug.

---

## 33. Interview Questions

### Easy
1. What is a Bounded Context, and why does it matter for microservices?
2. What is the difference between an Entity and a Value Object?
3. What is Ubiquitous Language, and why is it important?
4. What is an Aggregate Root, and what role does it play?
5. Give an example of a Core, a Supporting, and a Generic subdomain for an e-commerce business.

### Medium
6. Why shouldn't you design microservice boundaries directly from your database schema?
7. Explain why the same business term (e.g., "Customer") can mean different things in different Bounded Contexts, and why that's acceptable.
8. What is a Repository, and how does it help keep domain logic clean?
9. Why do Aggregates typically map to a single database transaction?
10. What is an Anti-Corruption Layer, and when would you use one?

### Hard
11. How would you identify Bounded Context boundaries for a completely new domain you've never worked in before?
12. Explain the trade-off between a large Aggregate (fewer round trips, more lock contention) and a small Aggregate (more round trips, less contention).
13. How do Domain Events relate to eventual consistency between Bounded Contexts?
14. Design the Context Map for a food delivery platform (Restaurant, Order, Delivery, Payment contexts) including the relationship type between each pair.
15. How does an Anemic Domain Model undermine the benefits of DDD, even if the Bounded Contexts themselves are well-defined?

---

## 34. Scenario-Based Questions

1. Your team has one "Customer" table shared by Marketing, Support, and Billing services, and each keeps adding fields the others don't need. How would you apply DDD to fix this?
2. A senior engineer insists every microservice should map 1:1 to a database table. How do you push back using DDD concepts?
3. You're modeling a "Ride" for a ride-sharing platform. The Pricing team, the Matching (driver-assignment) team, and the Trip-Tracking team all need "Ride" data. How would you define Bounded Contexts here?
4. Two services both modify the same "Account" balance directly in a shared database, occasionally causing race conditions. How would DDD's Aggregate concept prevent this?
5. Your company is integrating with a 15-year-old legacy inventory system with confusing field names and business rules. How would you introduce this system's data into your new, clean domain model?

---

## 35. Hands-on Exercises

1. For a "Hotel Booking" platform, identify at least 4 subdomains and classify each as Core, Supporting, or Generic.
2. Model a `Booking` Aggregate in JavaScript (similar to the `Order` example) with at least one business invariant enforced inside the Aggregate Root.
3. Write the Ubiquitous Language glossary (10 terms) for a "Library Management System" domain, as both business stakeholders and engineers would use them.
4. Draw a Context Map for a "Video Streaming" platform (Content Catalog, Subscription/Billing, Playback/Streaming, Recommendations contexts).
5. Identify one Anemic Domain Model in code you've written before (or a hypothetical example) and rewrite it to move logic into the domain object itself.

---

## 36. Mini Project

**Build: An Order Aggregate with Enforced Invariants**

1. Implement the `Order`, `LineItem`, and `Money` classes from Section 22, in a real Node.js project.
2. Write unit tests proving that:
   - You cannot place an order with zero line items.
   - You cannot modify an order after it has been placed.
   - Two `Money` Value Objects with the same amount/currency are considered equal.
3. Wrap the Aggregate in a simple Express API (`POST /orders`) using an Application Service, as shown in Section 23.

---

## 37. Advanced Project

**Build: Two Bounded Contexts With an Explicit Boundary**

1. Build `order-service` (Order Bounded Context) and `inventory-service` (Inventory Bounded Context) as fully separate Node.js services with separate databases.
2. In `order-service`, implement the `Order` Aggregate as in the Mini Project.
3. In `inventory-service`, implement a `Product` Aggregate with its own invariants (e.g., "stock cannot go negative").
4. Implement the boundary crossing: `order-service` calls `inventory-service`'s REST API to check stock — never touching Inventory's database directly.
5. Add a Domain Event: when an order is placed, publish `OrderPlaced` to Kafka; have `inventory-service` consume it and decrement stock, enforcing its own "stock cannot go negative" invariant independently.
6. Write a short Context Map document describing the relationship between these two Bounded Contexts (e.g., "Order is a Customer of Inventory's API; Inventory is a Customer of Order's Domain Events").

---

## 38. Summary

- DDD gives you a principled way to find microservice boundaries by identifying **Bounded Contexts** — places where one specific domain model is self-consistent and complete.
- **Entities** have identity; **Value Objects** don't; **Aggregates** group related objects under one root that enforces all consistency rules.
- **Ubiquitous Language** keeps code, conversation, and documentation aligned with how the business actually thinks and talks.
- Bounded Contexts map naturally to microservices, each with its own Aggregates, its own database, and explicit, deliberate communication (APIs or Domain Events) at the boundaries — never a shared database.
- Splitting services by database table or org chart, instead of by Bounded Context, is a leading cause of poorly-designed microservices architectures.

---

## 39. Revision Notes

- Domain → Subdomains (Core/Supporting/Generic) → Bounded Contexts → Microservices.
- Entity: identity matters. Value Object: only attributes matter, immutable, interchangeable.
- Aggregate Root: sole entry point for changes; enforces invariants; maps to one DB transaction.
- Repository: hides persistence details behind a collection-like interface.
- Ubiquitous Language: same words in conversation, docs, and code.
- Anti-Corruption Layer: translates messy external/legacy models at the boundary.
- Cross-context communication = APIs (sync) or Domain Events (async) — never shared databases.

---

## 40. One-Page Cheat Sheet

```
DOMAIN:              the overall business problem space
SUBDOMAIN:           Core (differentiator) | Supporting | Generic (buy, don't build)
BOUNDED CONTEXT:     boundary where one model is consistent -> maps to 1 microservice
ENTITY:              has identity, attributes can change, identity persists
VALUE OBJECT:        no identity, immutable, defined fully by attributes
AGGREGATE ROOT:      sole entry point for changes, enforces invariants, = 1 transaction
REPOSITORY:          hides persistence, exposes collection-like interface
DOMAIN SERVICE:      logic that spans multiple entities/value objects
UBIQUITOUS LANGUAGE: same vocabulary in business talk, docs, AND code
CONTEXT MAP:         diagram of how Bounded Contexts relate/communicate

GOLDEN RULE: Split services by BOUNDED CONTEXT (business capability),
             never by database table or org chart alone.
```

---

**Suggested Next Module:** Module 5 — Service Boundaries (applying DDD concepts to make concrete, defensible decisions about where to split services, including anti-patterns and migration strategy)
