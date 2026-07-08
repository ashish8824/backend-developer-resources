# Module 7 — REST Communication

> **Microservices Masterclass** | Level: Intermediate | Track: Node.js Backend Engineering
> Prerequisite: Module 1–6 (especially Module 6 — Communication Between Services)
> Next Module: Module 8 — gRPC Communication

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

Module 6 told you *when* to use synchronous communication. This module teaches you *how* to do it well using **REST over HTTP** — by far the most common synchronous protocol in microservices, and the one you'll use daily as a Node.js backend engineer.

Most engineers already "know REST" from building simple CRUD APIs for a single application. But **service-to-service REST** in a microservices context has different stakes than a typical public-facing CRUD API: you need to handle network failures gracefully (Axios calls between services fail far more often than a database query), design for idempotency (since retries are a fact of life), and think carefully about versioning and timeouts, since a poorly-designed internal REST API can quietly become the most fragile part of your entire system.

This module takes you from "I know what a GET request is" to "I can design and defensively implement a production-grade internal REST client and API that other engineers can depend on."

---

## 2. Learning Objectives

By the end of this module, you will be able to:

- Design a well-structured, RESTful internal API following resource-oriented conventions.
- Implement a resilient HTTP client for service-to-service calls with timeouts and retries.
- Design idempotent REST endpoints safe to retry.
- Apply the Circuit Breaker pattern to protect against cascading failures.
- Version REST APIs without breaking existing consumers.
- Distinguish when REST is the right choice for synchronous communication versus gRPC (covered next in Module 8).

---

## 3. Problem Statement

A team builds `order-service`, which calls `payment-service`'s REST API to charge a customer. In production:

- A network blip causes the request to `payment-service` to time out — but the charge actually succeeded on Payment's side before the response was lost. Order Service, seeing a timeout, retries the exact same request. Did it just **double-charge** the customer?
- `payment-service`'s team renames a field in their API response (`transactionId` → `txnId`) without warning. `order-service` silently breaks because it was reading the old field name.
- Under a traffic spike, `payment-service` becomes slow (not down). `order-service`'s calls to it start piling up, waiting on slow responses, until `order-service` itself runs out of available connections and becomes unresponsive too — a cascading failure that started in one service and took down another.

This module solves each of these specific, common problems: idempotency for safe retries, API versioning for safe evolution, and circuit breakers/timeouts for cascading failure protection.

---

## 4. Why This Concept Exists

REST (Representational State Transfer) exists because HTTP already provides a universally understood, well-supported, simple protocol for request-response communication — and REST is a set of conventions for using HTTP's existing verbs (GET, POST, PUT, DELETE) and status codes meaningfully, rather than inventing a new protocol from scratch. Its ubiquity (every language, every tool, every monitoring system understands HTTP) is precisely why it remains the default choice for synchronous service-to-service communication, despite newer alternatives like gRPC offering performance advantages in some scenarios.

The specific practices this module emphasizes (idempotency, versioning, circuit breakers) exist because **REST alone doesn't solve the hard problems of distributed communication** — REST just gives you the wire format and verbs; you must still explicitly design for the network's unreliability on top of it.

---

## 5. Historical Background

- **2000** — Roy Fielding's PhD dissertation formally described REST as an architectural style, originally as a description of what made the World Wide Web itself scale so successfully — not originally intended as a strict specification for internal APIs, though it was later adopted (and often loosely interpreted) that way by the industry.
- **Mid-2000s** — RESTful APIs became the dominant style for public web APIs (Twitter, Flickr, and others published REST APIs), gradually displacing the more complex XML-based SOAP protocol common in earlier enterprise integrations.
- **2010s** — As microservices adoption grew, REST became the default choice for internal service-to-service communication as well, given its simplicity, tooling support, and the fact that most engineers already knew it from building public APIs.
- **Mid-2010s onward** — As REST usage matured at scale, patterns like the **Circuit Breaker** (popularized by Netflix's Hystrix library), **idempotency keys** (popularized by payment providers like Stripe), and stricter **API versioning strategies** became standard practice, addressing the real-world pain points of REST at scale that this module focuses on.
- **Present** — REST remains the most common synchronous protocol in microservices, though gRPC (Module 8) has grown significantly for internal, performance-sensitive, or strongly-typed communication needs.

---

## 6. Real-World Analogy

**Analogy: Ordering at a Restaurant Counter**

Think of a REST API as a restaurant's ordering counter with a clear menu (the API contract):

- **GET** is like asking "what's on the menu / what's in my order so far?" — you're just looking, nothing changes.
- **POST** is like placing a brand-new order — something new gets created.
- **PUT** is like saying "replace my entire previous order with this new one" — a full update.
- **DELETE** is like canceling your order.
- **Idempotency** is like a hidden but critical rule: if the power flickers and you're not sure the cashier heard your order, and you repeat "one large coffee" again, a well-run restaurant should recognize your original order (perhaps through a receipt number) and NOT charge you twice for the same order — this is exactly the idempotency key pattern.
- **Versioning** is like the restaurant introducing a new menu (v2) while still honoring the old menu (v1) for customers who haven't seen the new one yet, rather than confusingly changing what "a burger" means overnight for everyone already mid-order.
- **Circuit Breaker** is like a smart host who, after noticing the kitchen has failed to fulfill the last 10 orders in a row, stops sending new orders to the kitchen for a few minutes and tells customers "we're experiencing delays" instead of letting the whole restaurant grind to a halt.

---

## 7. Technical Definition

> **REST (Representational State Transfer)** is an architectural style for network communication built on HTTP, where interactions are modeled as operations (GET, POST, PUT, PATCH, DELETE) on named **resources** (nouns, typically represented as URLs like `/orders/123`), using HTTP status codes to communicate outcomes, and (usually) JSON as the payload format.

> An **idempotent** REST operation is one where making the same request multiple times has the same effect as making it once — critical because network retries are unavoidable in distributed systems, and non-idempotent operations risk duplicated side effects (like double-charging a customer) on retry.

> **API Versioning** is the practice of evolving a REST API's contract (adding, changing, or removing fields/endpoints) without breaking existing consumers who haven't yet migrated to the new version.

> A **Circuit Breaker** is a pattern that monitors calls to a downstream service and, after detecting a threshold of failures, temporarily stops sending new requests to that service (the circuit "opens"), allowing it time to recover and preventing the caller from wasting resources on calls likely to fail.

---

## 8. Core Terminology

| Term | Meaning |
|---|---|
| **Resource** | A named entity exposed by the API, typically identified by a URL (e.g., `/orders/123`) |
| **HTTP Verb/Method** | GET (read), POST (create), PUT (full replace), PATCH (partial update), DELETE (remove) |
| **Idempotency Key** | A unique client-generated token attached to a request so the server can detect and safely ignore duplicate retries |
| **Status Code** | A standardized HTTP response code (200 OK, 201 Created, 400 Bad Request, 404 Not Found, 500 Internal Server Error, 503 Service Unavailable) |
| **HATEOAS** | "Hypermedia as the Engine of Application State" — a stricter REST principle where responses include links to related actions/resources (rarely fully implemented in practice, but good to recognize) |
| **API Versioning** | Strategy for evolving an API contract without breaking existing consumers (e.g., `/v1/orders` vs `/v2/orders`) |
| **Circuit Breaker** | Pattern that stops calling a failing downstream service temporarily to prevent cascading failure |
| **Timeout** | Maximum time a caller waits for a response before giving up |
| **Backoff / Retry** | Strategy for retrying a failed request, typically with increasing delay between attempts (exponential backoff) |
| **Content Negotiation** | HTTP mechanism (`Accept`/`Content-Type` headers) for agreeing on the payload format between client and server |

---

## 9. Internal Working

Here's how a well-designed internal REST interaction actually works end-to-end:

1. The calling service constructs a request to a resource URL (e.g., `POST /v1/charges`), including an **idempotency key** header for any operation with side effects.
2. The HTTP client library sends the request with a configured **timeout**, so the caller never waits indefinitely.
3. If the request fails due to a transient network issue, the client applies a **retry with exponential backoff** — but only if the operation is safe to retry (idempotent).
4. A **circuit breaker** wraps the call: if recent calls to this downstream service have been failing above a threshold, the breaker "opens" and fails fast (without even attempting the network call) for a cool-down period, giving the downstream service room to recover.
5. On the server side, the receiving service checks the idempotency key (if present) against a store of recently-processed keys; if it's seen this key before, it returns the **original** response rather than re-executing the operation.
6. The server processes the request, using appropriate HTTP status codes to communicate the outcome clearly (e.g., 201 for successful creation, 409 for a conflict, 422 for invalid business rules).
7. The client interprets the response: 2xx as success, 4xx as a client-side problem (don't blindly retry), 5xx as a possible transient server issue (may be safe to retry, depending on idempotency).

---

## 10. Step-by-Step Request Flow

**Scenario: Order Service charges a customer via Payment Service's REST API, with full resilience.**

```
Step 1:  Order Service generates a unique idempotencyKey for this
         charge attempt (e.g., a UUID tied to the order ID)
Step 2:  Order Service sends POST /v1/charges with the idempotency
         key in a header, wrapped in a circuit breaker, with a 5s timeout
Step 3:  Payment Service receives the request, checks its idempotency
         store: "Have I processed this exact key before?"
Step 4:  (First attempt) No match found — Payment Service processes
         the charge, stores the result against the idempotency key
Step 5:  Payment Service returns 201 Created with the transaction details
Step 6:  Network hiccup causes Order Service to never receive the response,
         and its client times out
Step 7:  Order Service's retry logic fires, resending the EXACT SAME
         request with the SAME idempotency key
Step 8:  Payment Service checks its idempotency store again: "Have I
         processed this key before?" YES — returns the ORIGINAL response
         WITHOUT charging the customer a second time
Step 9:  Order Service receives the (now successful) response and proceeds
```

Step 8 is the critical moment this entire module is designed around: without the idempotency key, Step 7's retry would have caused a second, duplicate charge.

---

## 11. Architecture Overview

```
                        Order Service
                              │
                  ┌───────────┴───────────┐
                  │   Circuit Breaker        │
                  │  (opossum / custom)      │
                  └───────────┬───────────┘
                              │
                  ┌───────────┴───────────┐
                  │  HTTP Client (axios)    │
                  │  - timeout: 5000ms      │
                  │  - retry w/ backoff     │
                  │  - idempotency key      │
                  └───────────┬───────────┘
                              │
                     POST /v1/charges
                              │
                              ▼
                       Payment Service
                  ┌───────────┴───────────┐
                  │ Idempotency Middleware  │
                  │ (checks Redis store)    │
                  └───────────┬───────────┘
                              │
                  ┌───────────┴───────────┐
                  │  Charge Business Logic  │
                  └───────────┬───────────┘
                              ▼
                          Payment DB
```

---

## 12. ASCII Diagrams

### 12.1 REST Resource Modeling

```
GOOD (resource/noun-oriented):

  GET    /orders           -> list orders
  GET    /orders/123       -> get one order
  POST   /orders           -> create a new order
  PUT    /orders/123       -> replace order 123 entirely
  PATCH  /orders/123       -> partially update order 123
  DELETE /orders/123       -> cancel/remove order 123


BAD (verb/RPC-style, not idiomatic REST):

  POST /getOrder?id=123
  POST /createOrder
  POST /updateOrderStatus
  POST /deleteOrderById
```

### 12.2 Circuit Breaker State Machine

```
        ┌──────────┐   failures exceed threshold   ┌──────────┐
        │  CLOSED   │ ─────────────────────────────▶│   OPEN    │
        │ (calls go  │                                │ (calls    │
        │  through)  │                                │  fail     │
        └────┬─────┘                                │  fast)    │
             │                                       └────┬─────┘
             │ success rate recovers                       │
             │ after cool-down period                      │ cool-down
             │                                              │ timer expires
        ┌────▼─────┐   one trial call succeeds      ┌──────▼─────┐
        │  CLOSED   │◀────────────────────────────── │ HALF-OPEN   │
        └──────────┘   (else back to OPEN)           │ (one trial  │
                                                       │  call)      │
                                                       └────────────┘
```

### 12.3 Idempotency Key Flow

```
Client Request 1:  POST /v1/charges  [Idempotency-Key: abc-123]
                          │
                          ▼
              Payment Service: key "abc-123" seen? NO
                          │
                          ▼
              Process charge, store result under "abc-123"
                          │
                          ▼
                  Return 201 Created


Client Request 2 (retry, SAME key):  POST /v1/charges [Idempotency-Key: abc-123]
                          │
                          ▼
              Payment Service: key "abc-123" seen? YES
                          │
                          ▼
              Return the SAME stored 201 Created response
              (charge is NOT processed again)
```

---

## 13. Mermaid Flowcharts

### 13.1 Resilient REST Call Decision Flow

```mermaid
flowchart TD
    A[Prepare REST call] --> B{Is circuit breaker OPEN?}
    B -- Yes --> C[Fail fast - return fallback immediately]
    B -- No --> D[Attach idempotency key if operation has side effects]
    D --> E[Send request with timeout]
    E --> F{Response received?}
    F -- Yes, 2xx --> G[Success - return result]
    F -- Yes, 4xx --> H[Client error - do NOT retry, surface to caller]
    F -- Yes, 5xx or timeout --> I{Is operation idempotent?}
    I -- Yes --> J[Retry with exponential backoff, up to max attempts]
    I -- No --> K[Surface error - do NOT blindly retry]
    J --> F
```

### 13.2 API Versioning Strategy

```mermaid
flowchart LR
    A[New breaking change needed] --> B[Introduce /v2/resource alongside /v1/resource]
    B --> C[Existing consumers keep using /v1 unaffected]
    C --> D[New/updated consumers adopt /v2]
    D --> E[Monitor /v1 usage over time]
    E --> F[Deprecate /v1 with advance notice once usage reaches zero]
```

---

## 14. Mermaid Sequence Diagrams

### 14.1 Idempotent Retry Flow

```mermaid
sequenceDiagram
    participant O as Order Service
    participant CB as Circuit Breaker
    participant P as Payment Service
    participant R as Idempotency Store (Redis)

    O->>CB: chargeCustomer(idempotencyKey=abc-123)
    CB->>P: POST /v1/charges [Idempotency-Key: abc-123]
    P->>R: Has key abc-123 been processed?
    R-->>P: No
    P->>P: Process charge
    P->>R: Store result under abc-123
    P-->>CB: 201 Created
    Note over O,P: Network failure - response lost in transit
    CB-->>O: Timeout error
    O->>CB: Retry chargeCustomer(SAME idempotencyKey=abc-123)
    CB->>P: POST /v1/charges [Idempotency-Key: abc-123]
    P->>R: Has key abc-123 been processed?
    R-->>P: Yes - return stored result
    P-->>CB: 201 Created (original result, no duplicate charge)
    CB-->>O: Success
```

---

## 15. Component Diagrams

```
┌─────────────────────────────────────────────────────────┐
│                    Payment Service                          │
│  ┌───────────────────┐                                      │
│  │  Express Router      │                                    │
│  │  POST /v1/charges    │                                    │
│  └─────────┬───────────┘                                    │
│            ▼                                                 │
│  ┌───────────────────┐                                      │
│  │ Idempotency          │  <- checks/stores keys in Redis     │
│  │ Middleware           │                                    │
│  └─────────┬───────────┘                                    │
│            ▼                                                 │
│  ┌───────────────────┐                                      │
│  │ Charge Controller    │                                    │
│  └─────────┬───────────┘                                    │
│            ▼                                                 │
│  ┌───────────────────┐        ┌──────────────────┐          │
│  │ Charge Service        │──────▶│  Payment DB       │          │
│  └───────────────────┘        └──────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

---

## 16. Deployment Diagrams

```
┌───────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                        │
│                                                               │
│  order-svc pods ──HTTP (via ClusterIP Service)──▶ payment-svc pods │
│         │                                                     │
│  Each order-svc pod runs its own HTTP client with its own     │
│  circuit breaker STATE (per-pod, not shared) — a common        │
│  production nuance: circuit breaker state doesn't automatically│
│  sync across replicas unless you use a shared store            │
│  (e.g., Redis-backed circuit breaker state) or a service mesh   │
└───────────────────────────────────────────────────────────┘
```

---

## 17. Database Interaction

The idempotency mechanism itself requires a small but critical piece of database/cache interaction:

```
Payment Service's Idempotency Store (Redis or a dedicated DB table):

  Key: idempotency-key:abc-123
  Value: { statusCode: 201, body: { transactionId: "txn_789", ... } }
  TTL: 24 hours (long enough to cover realistic retry windows)

  On every incoming request with an Idempotency-Key header:
    1. Check if this key exists in the store
    2. If yes -> return the STORED response immediately, skip business logic
    3. If no  -> execute business logic, THEN store the result under this key
                 (ideally atomically with the business logic's own transaction,
                 to avoid a race where the charge succeeds but the key
                 fails to be stored)
```

---

## 18. Failure Scenarios

| Scenario | Without This Module's Practices | With This Module's Practices |
|---|---|---|
| Response lost after successful processing, client retries | Customer double-charged | Idempotency key returns original result — charged once |
| Downstream service unavailable, caller keeps retrying rapidly | Caller wastes resources hammering a dead service; may worsen the outage | Circuit breaker opens, fails fast, gives downstream time to recover |
| Downstream service renames a response field | Consumer silently breaks or misbehaves | API versioning (`/v1` frozen contract) protects existing consumers |
| Downstream service is slow (not down) | Caller hangs indefinitely, risking its own resource exhaustion | Timeout ensures caller gives up after a bounded time |
| Client retries a non-idempotent operation (e.g., "add item to cart" without a key) | Item could be added multiple times | Design mutation-heavy endpoints to be idempotent, or require an idempotency key for anything with side effects |

---

## 19. Scalability Discussion

REST over HTTP scales well horizontally — stateless REST services can simply add more instances behind a load balancer. The scalability concern specific to this module is **connection and thread/resource exhaustion under slow downstream calls**: without timeouts and circuit breakers, a caller's own capacity can be consumed waiting on a struggling downstream service, meaning the downstream service's slowness effectively reduces the caller's throughput too — this is precisely the cascading failure this module's patterns are designed to prevent. Additionally, idempotency stores (Redis) must themselves scale to handle key lookups at your API's full request volume, so choose appropriate TTLs and consider Redis clustering for very high-throughput services.

---

## 20. High Availability Considerations

- Circuit breakers directly improve system-wide availability by preventing one failing service from consuming the resources (and thus availability) of every service that calls it.
- Idempotency ensures that **retries for availability** (a common HA technique — just try again) don't introduce **new correctness problems** (duplicate charges), letting you retry aggressively and safely.
- Running multiple instances of both the caller and the callee, behind load balancers, remains essential — REST's statelessness makes this straightforward, since any instance can handle any request.

---

## 21. CAP Theorem Implications

A synchronous REST call, by nature, favors **Consistency** — the caller waits for a definitive answer rather than proceeding with a guess. When a circuit breaker "opens" during a partition/outage, the system is explicitly choosing to **fail** (favoring safety/consistency over blindly proceeding) rather than **guess and risk incorrect behavior** — the fallback logic you provide for an open circuit is where you make an explicit, deliberate choice about how to behave during this CAP trade-off moment (e.g., "assume payment failed and ask the customer to retry" vs. "queue the order for later reconciliation").

---

## 22. Node.js Implementation

Let's build a complete, resilient REST client and a corresponding idempotent REST endpoint.

**Folder structure (Order Service — the caller):**
```
order-service/
├── src/
│   ├── clients/
│   │   └── paymentClient.js
│   └── app.js
```

**Folder structure (Payment Service — the callee):**
```
payment-service/
├── src/
│   ├── middleware/
│   │   └── idempotency.js
│   ├── routes/
│   │   └── charge.routes.js
│   ├── controllers/
│   │   └── charge.controller.js
│   └── app.js
```

**`order-service/src/clients/paymentClient.js`**
```javascript
import axios from "axios";
import CircuitBreaker from "opossum";
import { v4 as uuidv4 } from "uuid";

// The actual HTTP call, wrapped by the circuit breaker below
async function sendChargeRequest({ customerId, amount, idempotencyKey }) {
  const response = await axios.post(
    `${process.env.PAYMENT_SERVICE_URL}/v1/charges`,
    { customerId, amount },
    {
      timeout: 5000,
      headers: { "Idempotency-Key": idempotencyKey },
    }
  );
  return response.data;
}

const breakerOptions = {
  timeout: 5000,
  errorThresholdPercentage: 50, // open the circuit if 50% of calls fail
  resetTimeout: 10000,          // after 10s, try again (half-open)
};

const breaker = new CircuitBreaker(sendChargeRequest, breakerOptions);

// Fallback when the circuit is open or the call ultimately fails
breaker.fallback(() => {
  throw new Error("Payment service is currently unavailable. Please try again shortly.");
});

// Retry wrapper — only safe because chargeCustomer is idempotent
// thanks to the idempotencyKey being generated ONCE per logical order attempt
export async function chargeCustomer(customerId, amount, orderId) {
  const idempotencyKey = `charge-${orderId}`; // stable across retries for THIS order
  const maxRetries = 3;

  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await breaker.fire({ customerId, amount, idempotencyKey });
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const backoffMs = 200 * 2 ** attempt; // exponential backoff
      await new Promise((resolve) => setTimeout(resolve, backoffMs));
    }
  }
}
```

---

## 23. Express.js Examples

**`payment-service/src/middleware/idempotency.js`**
```javascript
import { redis } from "../db/redis.js";

// This middleware makes ANY route it wraps safe to retry.
export async function idempotencyMiddleware(req, res, next) {
  const key = req.headers["idempotency-key"];
  if (!key) {
    return next(); // idempotency is optional for read-only/safe endpoints
  }

  const storeKey = `idempotency:${key}`;
  const cached = await redis.get(storeKey);

  if (cached) {
    const { statusCode, body } = JSON.parse(cached);
    return res.status(statusCode).json(body); // return the ORIGINAL response
  }

  // Stash a reference so the controller can store its result after processing
  req.idempotencyStoreKey = storeKey;
  next();
}
```

**`payment-service/src/controllers/charge.controller.js`**
```javascript
import { redis } from "../db/redis.js";
import { processCharge } from "../services/charge.service.js";

export async function createCharge(req, res, next) {
  try {
    const { customerId, amount } = req.body;
    const result = await processCharge(customerId, amount);

    const responseBody = { transactionId: result.id, status: "succeeded" };

    // Store this result against the idempotency key BEFORE responding,
    // so any near-simultaneous retry sees the completed result.
    if (req.idempotencyStoreKey) {
      await redis.set(
        req.idempotencyStoreKey,
        JSON.stringify({ statusCode: 201, body: responseBody }),
        { EX: 60 * 60 * 24 } // 24-hour retention for retry safety
      );
    }

    res.status(201).json(responseBody);
  } catch (err) {
    next(err);
  }
}
```

**`payment-service/src/routes/charge.routes.js`**
```javascript
import { Router } from "express";
import { idempotencyMiddleware } from "../middleware/idempotency.js";
import { createCharge } from "../controllers/charge.controller.js";

const router = Router();

// Versioned route (see Module 7 versioning discussion) + idempotency-protected
router.post("/v1/charges", idempotencyMiddleware, createCharge);

export default router;
```

---

## 24. Docker Examples

```yaml
version: "3.9"
services:
  order-service:
    build: ./order-service
    environment:
      - PAYMENT_SERVICE_URL=http://payment-service:4003
    depends_on: [payment-service]

  payment-service:
    build: ./payment-service
    ports: ["4003:4003"]
    environment:
      - REDIS_URL=redis://idempotency-cache:6379
      - DATABASE_URL=postgresql://user:pass@payment-db:5432/payments
    depends_on: [idempotency-cache, payment-db]

  idempotency-cache:
    image: redis:7-alpine

  payment-db:
    image: postgres:16-alpine
    environment: [POSTGRES_DB=payments]
```

---

## 25. Kafka/Redis Integration

Redis, as shown above, is the natural backing store for the **idempotency key mechanism** — its TTL support is a perfect fit for "remember this key for 24 hours, then let it expire naturally."

While REST itself is synchronous, it's common for the receiving service to **also** publish an asynchronous event after successfully handling a REST request — combining both patterns from Module 6:

```javascript
// After successfully processing the charge via REST, ALSO publish
// an event for other interested services (e.g., Fraud Monitoring)
// that don't need to be in the synchronous critical path.
export async function createCharge(req, res, next) {
  const result = await processCharge(req.body.customerId, req.body.amount);
  await publishChargeProcessedEvent(result); // async, fire-and-forget
  res.status(201).json({ transactionId: result.id });
}
```

---

## 26. Error Handling

Design clear, consistent error response bodies so calling services can handle failures predictably:

```javascript
// Centralized error handler (payment-service)
export function errorHandler(err, req, res, next) {
  logger.error({ err: err.message, path: req.path });

  if (err.name === "ValidationError") {
    return res.status(422).json({ error: { code: "VALIDATION_ERROR", message: err.message } });
  }
  if (err.name === "InsufficientFundsError") {
    return res.status(402).json({ error: { code: "INSUFFICIENT_FUNDS", message: err.message } });
  }

  res.status(500).json({ error: { code: "INTERNAL_ERROR", message: "Something went wrong" } });
}
```

On the caller's side, distinguish retryable from non-retryable errors explicitly:
```javascript
function isRetryable(err) {
  if (!err.response) return true; // network error / timeout — likely transient
  return err.response.status >= 500; // 5xx may be transient; 4xx generally is not
}
```

---

## 27. Logging & Monitoring

- Log every REST call's **status code, latency, and target service** to build dashboards showing per-dependency health (P50/P95/P99 latency, error rate).
- Monitor **circuit breaker state transitions** (closed → open → half-open) as first-class events — an "OPEN" circuit is an urgent signal that a downstream dependency is unhealthy.
- Track **idempotency cache hit rate** — a consistently high hit rate for a given endpoint may indicate an upstream retry storm worth investigating, even though the system is handling it safely.

```javascript
breaker.on("open", () => logger.warn({ service: "payment-service" }, "Circuit breaker OPENED"));
breaker.on("halfOpen", () => logger.info({ service: "payment-service" }, "Circuit breaker HALF-OPEN, testing"));
breaker.on("close", () => logger.info({ service: "payment-service" }, "Circuit breaker CLOSED, recovered"));
```

---

## 28. Security Considerations

- Always use **HTTPS/TLS** for REST calls, even internally, unless a service mesh already provides mTLS transparently.
- Validate and sanitize all incoming request bodies (e.g., using `zod` or `joi`) — never trust that another internal service will always send well-formed data.
- Use short-lived **service-to-service auth tokens** or mTLS certificates rather than long-lived shared API keys for internal REST calls.
- Rate-limit even internal endpoints — a bug in a calling service (e.g., an accidental infinite retry loop) shouldn't be able to overwhelm a downstream service unchecked.

---

## 29. Performance Optimization

- Use **HTTP keep-alive** (connection pooling) in your HTTP client (axios supports this via a custom `httpAgent`) to avoid the overhead of a new TCP/TLS handshake on every request.
- Set **realistic timeouts** based on actual observed P99 latency for the downstream service, not an arbitrary guess — too short causes unnecessary failures; too long delays failure detection.
- Consider **response compression** (gzip) for larger payloads between services, especially over higher-latency networks.
- Cache **GET** responses (which should always be safe to cache, being read-only) where the data doesn't need to be perfectly fresh, reducing repeated identical calls.

```javascript
import http from "http";
import axios from "axios";

const keepAliveAgent = new http.Agent({ keepAlive: true, maxSockets: 50 });
const client = axios.create({ httpAgent: keepAliveAgent, timeout: 5000 });
```

---

## 30. Production Best Practices

- Require an **Idempotency-Key** header for every REST endpoint that has side effects (creates, charges, state transitions) — make this a team-wide API design standard, not an ad-hoc decision per endpoint.
- Version your APIs from day one (`/v1/...`), even if you don't expect to need `/v2` soon — retrofitting versioning onto an unversioned API already in production use is painful.
- Always set a **timeout** on every outgoing HTTP call — treat "no timeout configured" as a code review blocker.
- Wrap every meaningful cross-service REST call in a **circuit breaker**, especially for calls on a critical, high-traffic path.
- Document your REST API contracts formally (OpenAPI/Swagger) so consuming teams have a single source of truth, and consider contract testing to catch breaking changes automatically.

---

## 31. Anti-Patterns and Common Mistakes

| Anti-Pattern | Why It's a Problem |
|---|---|
| **RPC-style REST** (`POST /getOrder`, `POST /updateStatus`) | Ignores REST's resource-oriented conventions, confusing consumers and losing HTTP semantics (caching, idempotency-by-verb) |
| **No timeout on outgoing calls** | A single hung downstream call can exhaust the caller's resources indefinitely |
| **Retrying non-idempotent operations blindly** | Risks duplicated side effects (double charges, duplicate resource creation) |
| **Breaking changes without versioning** | Silently breaks every existing consumer of the API |
| **No circuit breaker on critical-path calls** | One failing downstream service can cascade failure across the whole system |
| **Overly chatty REST APIs** (requiring many round trips for one logical operation) | Increases latency and failure surface unnecessarily — consider a coarser-grained endpoint or GraphQL/BFF pattern |

```
RPC-style REST (anti-pattern):

  POST /getOrderById        <- should be GET /orders/:id
  POST /updateOrderStatus   <- should be PATCH /orders/:id
  POST /deleteOrder         <- should be DELETE /orders/:id

  Problem: loses HTTP's built-in semantics (GET is naturally
  cacheable and safe to retry; POST is not, by convention)
```

---

## 32. Debugging Tips

- When investigating a REST call failure, check the **status code first** — 4xx errors indicate a request problem (fix the caller), 5xx errors indicate a server-side problem (investigate the callee), and no-response/timeout indicates a network or availability problem.
- Use **trace IDs** propagated as headers (`x-trace-id`) across every REST call to reconstruct a full request's cross-service journey in your logs/tracing system.
- If you suspect duplicate processing, check the **idempotency store** directly for the key in question — confirm whether the duplicate request was correctly deduplicated or slipped through (a common bug: idempotency middleware applied inconsistently across routes).
- If a circuit breaker seems "stuck open," check the `resetTimeout` configuration and confirm the downstream service is actually healthy again — the breaker only tests recovery after its cool-down period elapses.

---

## 33. Interview Questions

### Easy
1. What are the main HTTP verbs used in REST, and what does each represent?
2. What is an idempotency key, and why is it needed?
3. What is a Circuit Breaker, and what problem does it solve?
4. Why should you always set a timeout on an outgoing HTTP call?
5. What's the difference between PUT and PATCH?

### Medium
6. Explain how you would design an idempotent "create charge" REST endpoint.
7. What is the difference between a 4xx and a 5xx status code, and how should a caller's retry logic treat each differently?
8. Describe the three states of a circuit breaker and what triggers each transition.
9. Why is API versioning important, and what's a simple strategy for introducing it?
10. Why can retrying a non-idempotent operation be dangerous?

### Hard
11. Design a complete resilient REST client (timeout + retry + circuit breaker + idempotency) for a "reserve inventory" call, and justify each design decision.
12. How would you migrate an existing unversioned REST API to a versioned one (`/v1`) without breaking current production consumers?
13. Explain a scenario where blindly retrying a "safe" GET request could still cause problems, and how you'd mitigate it.
14. How would you design idempotency for an endpoint where the client cannot easily generate a stable idempotency key (e.g., no natural order ID exists yet)?
15. Discuss the trade-offs of a fine-grained REST API (many small endpoints) vs. a coarse-grained one (fewer endpoints returning more combined data) for service-to-service communication.

---

## 34. Scenario-Based Questions

1. Your team discovers that a network blip caused 200 customers to be double-charged over the weekend. Post-incident, what specific change would you implement, and how would you verify it works?
2. A downstream service starts responding slowly (not failing, just slow) during a traffic spike, and your entire platform starts timing out on unrelated features. Diagnose and propose a fix using this module's concepts.
3. Your team wants to change a REST response field name that 5 other internal services depend on. What's your rollout plan?
4. A teammate says "since our internal network is reliable, we don't need timeouts or circuit breakers for internal REST calls." How do you respond?
5. You notice your idempotency cache hit rate for one endpoint has suddenly spiked to 40%. What would you investigate?

---

## 35. Hands-on Exercises

1. Design (on paper) a fully resource-oriented REST API for a "Library Management System" (Books, Members, Loans) using proper HTTP verbs and status codes.
2. Implement the idempotency middleware from Section 23 and write a test proving that two identical requests with the same Idempotency-Key produce the same response without double-processing.
3. Configure and test a circuit breaker (using `opossum`) against a mock service that fails 100% of the time, and observe the breaker transition to OPEN.
4. Write an OpenAPI (Swagger) spec for one endpoint from Exercise 1, including request/response schemas and status codes.
5. Identify one RPC-style REST anti-pattern in code you've seen before (or write a hypothetical example) and refactor it into proper resource-oriented REST.

---

## 36. Mini Project

**Build: A Resilient Payment Charge Endpoint**

1. Build `payment-service` with a `POST /v1/charges` endpoint protected by the idempotency middleware from Section 23, backed by Redis.
2. Build `order-service` with the resilient `paymentClient.js` from Section 22 (timeout + retry + circuit breaker + stable idempotency key).
3. Simulate a network failure (e.g., temporarily block the port or add artificial latency) and verify: (a) the client times out and retries, and (b) the customer is charged exactly once, even after multiple retries.

---

## 37. Advanced Project

**Build: A Fully Versioned, Resilient, Observable REST Integration**

1. Extend the Mini Project: implement both `/v1/charges` and a `/v2/charges` (with an additional optional `metadata` field) in `payment-service`, demonstrating non-breaking versioned evolution.
2. Add structured logging with trace ID propagation across both services.
3. Add circuit breaker event logging (open/half-open/close) and a simple `/metrics` endpoint (or Prometheus integration) exposing call counts, error rates, and current circuit state.
4. Write an integration test suite that: (a) verifies idempotent retries don't double-charge, (b) verifies the circuit breaker opens after repeated failures and recovers after the downstream service comes back, and (c) verifies `/v1` and `/v2` consumers both continue working correctly.
5. Document your API using OpenAPI, including all status codes, the `Idempotency-Key` header requirement, and versioning strategy.

---

## 38. Summary

- REST remains the default choice for synchronous service-to-service communication due to its simplicity and universal tooling support.
- Production-grade internal REST requires more than "know GET/POST/PUT/DELETE" — it requires deliberate design for idempotency (safe retries), timeouts (bounded waiting), circuit breakers (cascading failure prevention), and versioning (safe API evolution).
- Idempotency keys let you retry safely even for operations with side effects (like charging a customer), which is essential since network failures and retries are unavoidable in distributed systems.
- Circuit breakers protect the overall system by failing fast against an unhealthy downstream dependency rather than letting failures cascade.
- Status codes should be used meaningfully (2xx success, 4xx client error — don't blindly retry, 5xx server error — may be safe to retry if idempotent).

---

## 39. Revision Notes

- Resource-oriented REST: nouns as URLs, verbs as HTTP methods, meaningful status codes.
- Idempotency key: client-generated, server-stored, enables safe retries for side-effecting operations.
- Circuit breaker states: Closed (normal) → Open (failing fast) → Half-Open (testing recovery) → Closed or back to Open.
- Timeouts: mandatory on every outgoing call; prevents indefinite blocking and resource exhaustion.
- Versioning: introduce new versions (`/v2`) alongside old ones (`/v1`); never break existing consumers silently.
- Retry only idempotent operations, and only for potentially-transient errors (network failures, 5xx), never blindly for 4xx.

---

## 40. One-Page Cheat Sheet

```
HTTP VERBS:       GET (read) | POST (create) | PUT (replace) | PATCH (partial update) | DELETE (remove)
STATUS CODES:     2xx success | 4xx client error (don't retry) | 5xx server error (maybe retry if idempotent)
IDEMPOTENCY KEY:  client-generated, stable per logical operation, server deduplicates via cache (Redis)
CIRCUIT BREAKER:  Closed -> Open (fail fast) -> Half-Open (test) -> Closed or back to Open
TIMEOUT:          mandatory on every outgoing call - never leave unbounded
RETRY:            exponential backoff, ONLY for idempotent operations + transient errors
VERSIONING:       /v1/, /v2/ side by side - never break existing consumers silently

GOLDEN RULES:
  - Every side-effecting endpoint should support an Idempotency-Key
  - Every outgoing call needs a timeout AND (for critical paths) a circuit breaker
  - Never retry non-idempotent operations without a safety mechanism
  - Version your API from day one; evolve additively, don't break silently
```

---

**Suggested Next Module:** Module 8 — gRPC Communication (Protocol Buffers, unary vs streaming calls, and when gRPC outperforms REST for internal service-to-service communication)
